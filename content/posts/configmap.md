---
title: "ConfigMap"
date: 2022-12-09
draft: false
Categories: [kubernetes]
---

ConfigMap 是一种 API 对象，用来将非机密性的数据保存到键值对中。使用时， Pod 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。

ConfigMap 将你的环境配置信息和容器镜像解耦，便于应用配置的修改。

使用场景：

1. 容器 entrypoint 的命令行参数

2. 容器的环境变量
3. 映射成文件
4. 编写代码在 Pod 中运行，使用 Kubernetes API 来读取 ConfigMap

## 使用

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mycm
data:
  host: "0.0.0.0"
  port: "9999"
  user.properties: |
    user.name=txl
    user.age=18
```

查看

```bash
$ kubectl get cm -n default
NAME   DATA   AGE
mycm   3      45m
```

### 在环境变量中使用

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
            - name: HOST
              valueFrom:
                configMapKeyRef:
                  name: mycm  # ConfigMap 名称
                  key: host   # 需要取值的键
```

### 映射成文件

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
          - name: cmconfig
            mountPath: "/config"
            readOnly: true
      volumes:
      - name: cmconfig
        configMap:
          name: mycm  # ConfigMap 名称
          items:      # 来自 ConfigMap 的一组键，将被创建为文件
          - key: "user.properties"
            path: "userinfo.conf"
```

如果省略 ``items`` 节点，会映射 ConfigMap 全部的 key

```bash
$ cd /config && ls -l
total 0
lrwxrwxrwx    1 root     root            11 Dec  6 14:00 host -> ..data/host
lrwxrwxrwx    1 root     root            11 Dec  6 14:00 port -> ..data/port
lrwxrwxrwx    1 root     root            22 Dec  6 14:00 user.properties -> ..data/user.properties
```

#### 使用 subPath

可用于指定所引用的卷内的子路径，而不是其根路径。

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
            - name: HOST
              valueFrom:
                configMapKeyRef:
                  name: mycm
                  key: host
          volumeMounts:
          - name: cmconfig
            mountPath: /config/userinfo.conf
            subPath: user.properties
      volumes:
      - name: cmconfig
        configMap:
          defaultMode: 0655
          name: mycm
```



## 使用 client-go 调用

GitHub: [kubernetes/client-go: Go client for Kubernetes](https://github.com/kubernetes/client-go/)

### 集群外调用

使用 api 代理

```bash
$ kubectl proxy --address='0.0.0.0' --accept-hosts='^*$' --port=8009
```

获取 ConfigMap

```go
func getClient() *kubernetes.Clientset {
	config := &rest.Config{
		Host: "http://ip:8009",
	}

	c, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalln(err)
	}

	return c
}

func main() {
	client := getClient()
	cm, err := client.CoreV1().ConfigMaps("default").Get(context.Background(), "mycm", v1.GetOptions{})
	if err != nil {
		log.Fatalln(err)
	}

	fmt.Println(cm.Data)
}
```

执行结果

```
map[host:0.0.0.0 port:9999 user.properties:user.name=txl
user.age=18
]
```

### 集群内调用

#### 创建 ServiceAccouont

拥有 default 空间内对 ConfigMap 的查看权限

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-cm
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: clusterrole-cm
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: clusterrolebinding-cm
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: clusterrole-cm
subjects:
  - kind: ServiceAccount
    name: sa-cm
    namespace: default
```

#### 调用 API

- token 路径：``/var/run/secrets/kubernetes.io/serviceaccount/token``
- api server 地址：``https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_PORT_443_TCP_PORT``
- 证书路径：``/var/run/secrets/kubernetes.io/serviceaccount/ca.crt``

```go
var apiServer string
var token string

func init() {
    apiServer = fmt.Sprintf("https://%s:%s",
        os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_PORT_443_TCP_PORT"))
    f, err := os.Open("/var/run/secrets/kubernetes.io/serviceaccount/token")
    if err != nil {
        log.Fatal(err)
    }
    b, _ := io.ReadAll(f)
    token = string(b)
}

func getClient() *kubernetes.Clientset {
    config := &rest.Config{
        Host:            apiServer,
        BearerToken:     token,
        TLSClientConfig: rest.TLSClientConfig{CAFile:  	"/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"},
    }

    c, err := kubernetes.NewForConfig(config)
    if err != nil {
        log.Fatalln(err)
    }

    return c
}

func main() {
    client := getClient()
    cm, err := client.CoreV1().ConfigMaps("default").Get(context.Background(), "mycm", v1.GetOptions{})
    if err != nil {
        log.Fatalln(err)
    }

    fmt.Println(cm.Data)
    select {}
}
```

交叉编译

```bash
set GOOS=linux
set GOARCH=amd64
go build -o cmtest main.go
```

#### 创建 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cmtest
spec:
  selector:
    matchLabels:
      app: cmtest
  replicas: 1
  template:
    metadata:
      labels:
        app: cmtest
    spec:
      serviceAccount: sa-cm		# 指定 ServiceAccount
      nodeName: lain1			# 指定 node
      containers:
        - name: cmtest
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/cmtest"]
          volumeMounts:
            - name: app
              mountPath: /app
      volumes:
        - name: app
          hostPath:
            path: /home/txl/goapi
            type: Directory
```

查看

```bash
$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
cmtest-96b96c458-szxhk   1/1     Running   0          2m46s

$ kubectl logs cmtest-96b96c458-szxhk
map[host:0.0.0.0 port:9999 user.properties:user.name=txl
user.age=18
]
```

### 监控 ConfigMap 变化

```go
type CmHandler struct{}
func(this *CmHandler) OnAdd(obj interface{}){}
func(this *CmHandler) OnUpdate(oldObj, newObj interface{}){
    if newObj.(*v1.ConfigMap).Name=="mycm"{
        log.Println("mycm发生了变化")
    }
}
func(this *CmHandler) OnDelete(obj interface{}){}

func main() {
    fact:=informers.NewSharedInformerFactory(getClient(), 0)

    cmInformer:=fact.Core().V1().ConfigMaps()
    cmInformer.Informer().AddEventHandler(&CmHandler{})

    fact.Start(wait.NeverStop)
    select {}
}
```

