---
title: "K8s 认证和授权"
date: 2022-12-03
draft: false
Categories: [kubernetes]
---

文档：[使用 RBAC 鉴权 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/)

## 用户

- UserAccount（普通用户）：集群外部访问时使用的用户账号，最常见的就是 ``kubectl`` 命令就是作为 kubernetes-admin 用户来执行，k8s本身不记录这些账号
- ServiceAccount（服务账户）：它们被绑定到特定的名字空间，服务账号与一组以 Secret 保存的凭据相关，这些凭据会被挂载到 Pod 中，从而允许集群内的进程访问 Kubernetes API

## 用户认证

![image-20221203231927703](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/image-20221203231927703.png)

### A. 使用 X509 客户证书

#### 生成证书

安装 OpenSSL

```bash
$ sudo yum install openssl openssl-devel
```

生成一个名称为txl的普通用户的客户端证书

```bash
$ mkdir ua/txl
$ cd ua/txl

# 生成客户端私钥
$ openssl genrsa -out client.key 2048

# 根据私钥生成csr, 指定用户名txl
$ openssl req -new -key client.key -out client.csr -subj "/CN=txl"

# 根据k8s的CA证书生成客户端证书
$ sudo openssl x509 -req -in client.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out client.crt -days 365
```

#### 证书反解

获取证书设置的CN(Common name)

```bash
$ openssl x509 -noout -subject -in client.crt
```

#### 使用证书初步请求API

```bash
$ kubectl get endpoints
NAME         ENDPOINTS            AGE
kubernetes   192.168.0.111:6443   4d5h

$ curl --cert ./client.crt --key ./client.key --cacert /etc/kubernetes/pki/ca.crt -s https://192.168.0.111:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.0.111:6443"
    }
  ]
}
```

可以使用 ``--insecure`` 代替 `` --cacert /etc/kubernetes/pki/ca.crt`` 忽略服务端证书验证

#### 证书加入 kube config

把 client.crt 加入到 ~/.kube/config

```bash
$ kubectl config --kubeconfig=/home/txl/.kube/config set-credentials txl --client-certificate=/home/txl/ua/txl/client.crt --client-key=/home/txl/ua/txl/client.key
```

创建一个名为 user_context 的 context

```bash
$ kubectl config --kubeconfig=/home/txl/.kube/config set-context user_context --cluster=kubernetes --user=txl
```

切换当前上下文为 user_context

```bash
$ kubectl config use-context user_context

# 查看
$ kubectl config current-context

# 重新切回默认管理员
$ kubectl config use-context kubernetes-admin@kubernetes
```



### B. 使用静态令牌文件(Token)

token 和证书只能配一个

#### 生成 Token

```bash
$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

#### 加入 kube config

```bash
$ kubectl config set-credentials txl --token=fdb72d94a1c2c2cfbf82341d1f98c68c
```

#### 修改 api-server 启动参数

```bash
$ sudo vi /etc/kubernetes/pki/token_auth
# 加入
fdb72d94a1c2c2cfbf82341d1f98c68c,txl,1001

$ sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# 加入
--token-auth-file=/etc/kubernetes/pki/token_auth
```

#### 查看

```bash
$ curl -H "Authorization: Bearer fdb72d94a1c2c2cfbf82341d1f98c68c" https://192.168.0.111:6443/api/v1/namespaces/default/pods --insecure
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "1586027"
  },
  "items": []
}
```



## 资源

查看所有资源

```bash
$ kubectl api-resources -o wide
```

其中 VERBS 列展示了该资源对应的操作，比如 role 

- create 创建
- delete 删除
- deletecollection 批量删除
- get 获取
- list 列表
- patch 合并变更
- update 更新
- watch 监听



## Role 和 RoleBinding

- Role（角色）：包含一组代表相关权限的规则，用于授予对单个命名空间的资源访问
- RoleBinding（角色绑定）：将角色中定义的权限赋予一个或者一组用户

### 创建 Role

下面是一个位于default的role，拥有对pod的读访问权限

```bash
$ vi role_mypod.yaml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: mypod
rules:
- apiGroups: ["*"]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

执行创建

```bash
$ kubectl apply -f role_mypod.yaml

# 查看default空间所有role
$ kubectl get role -n default
```

删除

```bash
$ kubectl delete role mypod -n default
```

### 绑定 RoleBinding

创建一个名为mypodbinding的 rolebinding，关联创建的用户txl和创建的角色mypod

#### 1. 使用命令

```bash
$ kubectl create rolebinding mypodbinding -n default --role mypod --user txl
```

#### 2. 使用 yaml

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: null
  name: mypodrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mypod
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: txl
- kind: ServiceAccount
  name: txl
```

查看

```bash
$ kubectl get rolebinding -n default
NAME               ROLE
mypodrolebinding   Role/mypod
```

删除

```bash
$ kubectl delete rolebinding mypodbinding -n default
```



## ClusterRole 和 ClusterRoleBinding

ClusterRole 同样可以用于授予 Role 能够授予的权限。 因为 ClusterRole 属于集群范围，所以它也可以为以下资源授予访问权限：

- 集群范围资源（如节点 Node）
- 非资源端点（如 `/healthz`）
- 跨名字空间访问的名字空间作用域的资源（如 Pod）

clusterRole不限定命名空间，绑定既可以使用 RoleBinding，也可以使用 ClusterRoleBinding

### 创建 ClusterRole

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mypod-cluster
rules:
- apiGroups: ["*"]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

查看

```bash
$ kubectl get clusterrole
```

### 绑定 RoleBinding

需要指定命名空间

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: mypodrolebinding-cluster
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mypod-cluster
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: txl
```

查看

```bash
$ kubectl get rolebinding -n kube-system
NAME                                                ROLE
mypodrolebinding-cluster                            ClusterRole/mypod-cluster
```

### 绑定 ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mypod-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: mypod-cluster
subjects:
 - apiGroup: rbac.authorization.k8s.io
   kind: User
   name: txl
 - kind: ServiceAccount
   name: txl
   namespace: default
```



## ServiceAccount

UserAccount 是可以跨 namespace 的，而 ServiceAccount 只能局限在自己所属的 namespace 中，每个 namespace 都会有一个默认的 default 账号 。

### 创建 SA

```bash
$ kubectl create sa mysa
```

也可以导出到 yaml

```bash
kubectl create sa mysa -o yaml --dry-run=client > mysa.yaml
```

查看

```bash
$ kubectl get sa -n default
NAME      SECRETS   AGE
default   1         4d7h
mysa      1         5m21s

# 查看令牌
$ kubectl describe sa mysa
Name:                mysa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   mysa-token-5bnbm
Tokens:              mysa-token-5bnbm
Events:              <none>

$ kubectl describe secret mysa-token-5bnbm
```

### 绑定 ClusterRoleBinding

使用命令的方式

```bash
$ kubectl create clusterrolebinding mysa-clusterrolebinding --clusterrole=mypod-cluster --serviceaccount=default:mysa
```

查看

```bash
$ kubectl get clusterrolebinding
NAME                                                   ROLE
mypod-clusterrolebinding                               ClusterRole/mypod-cluster
mysa-clusterrolebinding                                ClusterRole/mypod-cluster
```

### 外部访问 API

#### 安装 jq

轻量级的 json 处理命令。可以对 json 数据进行分片、过滤、映射、转换和格式化输出

```bash
$ sudo yum install jq -y
```

#### 获取 SA Token

保存到临时变量 mysatoken 中

```bash
$ mysatoken=$(kubectl get secret  $(kubectl get sa  mysa -o json | jq -Mr '.secrets[0].name') -o json | jq -Mr '.data.token' | base64 -d)

# 查看token
$ echo $mysatoken
```

请求

```bash
$ curl -H "Authorization: Bearer $mysatoken" --insecure https://192.168.0.111:6443/api/v1/namespaces/default/pods 
```

### 在 Pod 里访问 API

创建一个测试 pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: mysa  # 指定SA，否则使用的default SA 
      containers:
        - name: nginxtest
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

查看 pod

```bash
$ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
myngx-74748c5956-5rfrs   1/1     Running   0          4m21s
```

进入容器

```bash
$ kubectl exec -it myngx-74748c5956-5rfrs -- sh
```

设置临时变量

```bash
# SA token
$ TOKEN=`cat /var/run/secrets/kubernetes.io/serviceaccount/token`

# api server地址
$ APISERVER="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT"
```

请求（跳过服务器证书检查）

```bash
$ curl --header "Authorization: Bearer $TOKEN" --insecure -s $APISERVER/api/v1/namespaces/default/pods 
```

使用证书请求

``` bash
$ curl --header "Authorization: Bearer $TOKEN" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt $APISERVER/api/v1/namespaces/default/pods 
```
