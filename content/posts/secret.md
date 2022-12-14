---
title: "Secret"
date: 2022-12-10
draft: false
Categories: [kubernetes]
---

Secret 是一种包含少量敏感信息例如密码、OAuth 令牌和 SSH 密钥。 将这些信息放在 secret 中比放在 Pod 的定义或者 容器镜像中来说更加安全和灵活。

Kubernetes 提供若干种内置的类型，用于一些常见的使用场景。 针对这些类型，Kubernetes 所执行的合法性检查操作以及对其所实施的限制各不相同。

| 内置类型                              | 用法                                     |
| ------------------------------------- | ---------------------------------------- |
| `Opaque`                              | 用户定义的任意数据                       |
| `kubernetes.io/service-account-token` | 服务账号令牌                             |
| `kubernetes.io/dockercfg`             | `~/.dockercfg` 文件的序列化形式          |
| `kubernetes.io/dockerconfigjson`      | `~/.docker/config.json` 文件的序列化形式 |
| `kubernetes.io/basic-auth`            | 用于基本身份认证的凭据                   |
| `kubernetes.io/ssh-auth`              | 用于 SSH 身份认证的凭据                  |
| `kubernetes.io/tls`                   | 用于 TLS 客户端或者服务器端的数据        |
| `bootstrap.kubernetes.io/token`       | 启动引导令牌数据                         |



## 基本用法

### 创建

首先将字符串转换为 base64

```bash
$ echo -n 'admin' | base64
$ echo -n '1f2d1e2e67df' | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

如果希望使用非 base64 编码的字符串直接放入 Secret 中，应当使用 ``stringData`` 字段

```yaml
stringData:
  config.yaml: |
    apiUrl: "https://my.api.com/api/v1"
    username: "admin"
    password: "1f2d1e2e67df"
```

### 解码

输出 yaml 内容

```bash
$ kubectl get secret mysecret -o yaml
$ echo -n 'YWRtaW4=' | base64 -d
```

使用 JSONPath 模板输出特定字段，文档：[JSONPath 支持 | Kubernetes](https://kubernetes.io/zh-cn/docs/reference/kubectl/jsonpath/)

```bash
$ kubectl get secret mysecret -o jsonpath={.data.username} | base64 -d
```

### 以环境变量的方式使用

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
      containers:
      - name: ngx
        image: nginx:1.18-alpine
        imagePullPolicy: IfNotPresent
        env:
          - name: SECRET_USERNAME
            valueFrom:
              secretKeyRef:
                name: mysecret
                key: username
```

### 挂载到文件

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
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
          - name: foo
            mountPath: /etc/foo
            readOnly: true
      volumes:
      - name: foo
        secret:
          secretName: mysecret
          items:
          - key: username
            path: my-username
```

如果省略 items 节点，会映射 secret 所有的 key



## 手工配置 basic-auth 认证

### 生成密码文件

可以使用 htpasswd 或者 openssl passwd 命令生成

#### 安装

```bash
$ sudo yum -y install httpd-tools
```

#### 生成认证密码

创建一个用户名和密码的认证到auth文件中

```bash
$ htpasswd -c auth txl
```

### 导入 Secret

```bash
$ kubectl create secret generic basic-auth --from-file=auth
```

### 创建 nginx Configmap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ngx
data:
  default: |
    server {
        listen       80;
        server_name  localhost;
        location / {
           auth_basic "test auth";
           auth_basic_user_file /etc/nginx/basicauth;	# 指向生成的密码文件
           root   /usr/share/nginx/html;
           index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /usr/share/nginx/html;
        }
    }
```

### 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx
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
      containers:
      - name: ngx
        image: nginx:1.18-alpine
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: nginx-default
          mountPath: /etc/nginx/conf.d/default.conf		# 覆盖nginx默认配置
          subPath: default
        - name: basic-auth
          mountPath: /etc/nginx/basicauth
          subPath: auth
      volumes:
        - name: nginx-default
          configMap:
            name: ngx
            defaultMode: 0655
        - name: basic-auth
          secret:
            secretName: basic-auth
            defaultMode: 0655
```

### 访问

```bash
$ kubectl get pods -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE   READINESS GATES
ngx-5597699bdf-ntbjx   1/1     Running   0          31m   10.244.3.29   lain2   <none>           <none>

$ curl --basic -u txl:123 http://10.244.3.29
```



## 从私有仓库拉取镜像

使用 Docker Hub 镜像仓库

### 登录 Docker 镜像仓库

```bash
$ docker login --username=<用户名>

# 发布
$ docker push <镜像名>
```

### 创建 DockerHub Secret

```bash
$ kubectl create secret docker-registry regcred \
    --docker-server=https://index.docker.io/v1/ \
    --docker-username=<用户名> \
    --docker-password=<密码> \
    --docker-email=<邮箱地址>
```

解码

```bash
$ kubectl get secret docker-registry -o jsonpath={.data.*} | base64 -d
```

### 使用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myalpine
spec:
  selector:
    matchLabels:
      app: myalpine
  replicas: 1
  template:
    metadata:
      labels:
        app: myalpine
    spec:
      imagePullSecrets:
        - name: docker-registry
      containers:
        - name: alpine
          image: lains3/alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","echo this is alpine && sleep 36000"]
```
