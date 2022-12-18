---
title: "K8s client-go"
date: 2022-12-18
draft: false
Categories: [k8s-go]
---

client-go 是负责与 Kubernetes APIServer 服务进行交互的客户端库，利用 Client-Go 与 Kubernetes APIServer 进行的交互访问，来对 Kubernetes 中的各类资源对象进行管理操作，包括内置的资源对象及 CRD

## client-go 客户端

Client-Go 共提供了 4 种与 Kubernetes APIServer 交互的客户端

- RESTClient：最基础的客户端，主要是对 HTTP 请求进行了封装，支持 Json 和 Protobuf 格式的数据。
- DiscoveryClient：发现客户端，负责发现 APIServer 支持的资源组、资源版本和资源信息的。
- ClientSet：负责操作 Kubernetes 内置的资源对象，例如：Pod、Service等。
- DynamicClient：动态客户端，可以对任意的 Kubernetes 资源对象进行通用操作，包括 CRD。

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202212181811849.png)

## 基本使用

参考 [API文档](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/) 的 group 和 apiVersion 等信息

### 创建 admin ServiceAccount

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: admin
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
```

查看 token

```bash
$ kubectl describe sa admin -n kube-system
Name:                admin
Namespace:           kube-system
Tokens:              admin-token-nzxlb

$ kubectl describe secret admin-token-nzxlb -n kube-system
```

### 建立连接

先使用反代的方式，在 master 节点执行

```bash
$ kubectl proxy --address='0.0.0.0' --accept-hosts='^*$' --port=8009
```

安装客户端库 (版本要对应 [Github](https://github.com/kubernetes/client-go))，连接 API Server

```go
var K8sClient *kubernetes.Clientset

func init() {
	config := &rest.Config{
		Host:        "ip:8009",
		BearerToken: "",
	}
	client, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalln(err)
	}

	K8sClient = client
}
```

### 示例：获取资源列表

```go
ctx := context.Background()

// 查询 kube-system 命名空间下的 service
svs, _ := K8sClient.CoreV1().Services("kube-system").List(ctx, v1.ListOptions{})

// 查询 kube-system 命名空间下的 deployment
deps, _ := K8sClient.AppsV1().Deployments("kube-system").List(ctx, v1.ListOptions{})
```

### 示例：获取资源详情

```go
ctx := context.Background()
// 获取名称为 ngx 的 deployment 资源
dep, _ := K8sClient.AppsV1().Deployments(namespace).Get(ctx, "ngx", metav1.GetOptions{})
dep.Name		// 名称
dep.Namespace	// 命名空间
dep.Status.Replicas		// 副本数量
dep.CreationTimestamp	// 创建时间
dep.Spec.Template.Spec.Containers[0].Image	// 第一个镜像名称
```

### 示例：获取 deployment 关联的所有 pod

```go
type PodModel struct {
	Name      string	// pod名称
	NodeName  string	// 节点
	Images    string	// 镜像名称
	CreatedAt string	// 创建时间
}

// 拼接labels字符串
func GetLabels(labels map[string]string) string {
	var labelStr strings.Builder

	for k, v := range labels {
		if labelStr.Len() != 0 {
			labelStr.WriteString(",")
		}
		labelStr.WriteString(fmt.Sprintf("%s=%s", k, v))
	}

	return labelStr.String()
}

// 根据deployment获取关联的pods集合
func GetPodsByDep(namespace string, dep *v1.Deployment) (pods []*PodModel) {
	ctx := context.Background()
	// 通过LabelSelector去匹配对应的pods
	listOpt := metav1.ListOptions{
		LabelSelector: GetLabels(dep.Spec.Selector.MatchLabels)，
    }

	podList, _ := K8sClient.CoreV1().Pods(namespace).List(ctx, listOpt)

	pods = make([]*PodModel, len(podList.Items))

	for i, pod := range podList.Items {
		pods[i] = &PodModel{
			Name:      pod.Name,
			NodeName:  pod.Spec.NodeName,
			Images:    GetPodImages(pod.Spec.Containers),
			CreatedAt: pod.CreationTimestamp.Format("2006-01-02 15:04:05"),
		}
	}

	return
}

dep, _ := K8sClient.AppsV1().Deployments(namespace).Get(context.Background(), "ngx", metav1.GetOptions{})
Pods = GetPodsByDep("default", dep)
```

### 示例：修改 deployment 副本数量

```go
// 获取 deployment 副本数量
scale, _ := K8sClient.AppsV1().Deployments("default").GetScale(ctx, "ngx", v1.GetOptions{})
// 修改副本数量
scale.Spec.Replicas++
K8sClient.AppsV1().Deployments("default").UpdateScale(ctx, "ngx", scale, v1.UpdateOptions{})
```

### 示例：创建资源

根据 yaml 创建一个 nginx deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
  namespace: default
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
        - name: nginxtest
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

```go
ngxDep := &appV1.Deployment{}

// 读取yaml内容
b, _ := os.ReadFile("nginx.yaml")
ngxJson, _ := yaml.ToJSON(b)
json.Unmarshal(ngxJson, ngxDep)

dep, _ := K8sClient.AppsV1().Deployments("default").Create(context.Background(), ngxDep, v1.CreateOptions{})
```
