---
title: "Pod 水平自动扩缩 — HPA"
date: 2022-12-12
draft: false
Categories: [kubernetes]
---

Horizontal Pod Autoscaling（Pod 水平自动伸缩），简称HPA。它可以基于 CPU 利用率或其他指标自动扩缩 ReplicationController、Deployment 和 ReplicaSet 中的 Pod 数量，它不适用于无法扩缩的对象，比如 DaemonSet。除了 CPU 利用率，也可以基于其他应程序提供的自定义度量指标来执行自动扩缩

文档：[Pod 水平自动扩缩 | Kubernetes](https://kubernetes.io/zh-cn/docs/tasks/run-application/horizontal-pod-autoscale/)

![image-20200928160203334](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/20201004174900.png)

我们可以简单的通过 `kubectl autoscale` 命令来创建一个 HPA 资源对象，HPA Controller 默认 30s 轮询一次（可通过 kube-controller-manager 的 `--horizontal-pod-autoscaler-sync-period` 参数进行设置)，查询指定的资源中的 Pod 资源使用率，并且与创建时设定的值和指标做对比，从而实现自动伸缩的功能。



## Metrics Server

Metrics Server 可以通过标准的 Kubernetes Summary API 把监控数据暴露出来，有了 Metrics Server 之后，就可以采集节点和 Pod 的内存、磁盘、CPU 和网络的使用率等。Metrics API URI 为 ``/apis/metrics.k8s.io/ ``

### 安装

可以通过官方仓库的资源清单安装： [Github](https://github.com/kubernetes-sigs/metrics-server) 

部署之前，需要修改 k8s.gcr.io/metrics-server/metrics-server 镜像的地址

```yaml
# image: k8s.gcr.io/metrics-server/metrics-server:v0.4.1
image: bitnami/metrics-server:0.4.1
```

等待部署完成后，可以查看 pod 日志是否正常

```bash
$ kubectl get pods -n kube-system -l k8s-app=metrics-server
NAME                              READY   STATUS    RESTARTS   AGE
metrics-server-7d8467779f-vgtzb   1/1     Running   0          18m

$ kubectl logs -f metrics-server-7d8467779f-vgtzb -n kube-system
```

### 查看

```bash
$ kubectl top nodes
NAME    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
lain1   149m         3%     3526Mi          45%       
lain2   208m         10%    2786Mi          75%

$ kubectl top pod etcd-lain1 -n kube-system
NAME         CPU(cores)   MEMORY(bytes)   
etcd-lain1   20m          249Mi
```



## 使用

HPA 的 API 有三个版本，在当前稳定版本 autoscaling/v1 中只支持基于 CPU 指标的缩放。在 Beta 版本 autoscaling/v2beta2，引入了基于内存和自定义指标的缩放。

```bash
$ kubectl api-versions | grep autoscal
autoscaling/v1			# 只支持通过cpu伸缩
autoscaling/v2beta1		# 支持通过cpu、内存和自定义数据来进行伸缩
autoscaling/v2beta2
```

我们部署一个测试 api，执行一些 CUP 密集型计算，然后利用 HAP 来进行自动伸缩容

```go
test := map[string]string{
	"str": "requests来设置各容器需要的最小资源",
}
r := gin.New()
r.GET("/", func(context *gin.Context) {
	ret := 0
	for i := 0; i <= 1000000; i++ {
		t := map[string]string{}
		b, _ := json.Marshal(test)
		_ = json.Unmarshal(b, t)
		ret++
	}
	context.JSON(200, gin.H{"message": ret})
})
r.Run(":8080")
```

资源清单如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web1
spec:
  selector:
    matchLabels:
      app: myweb
  replicas: 1
  template:
    metadata:
      labels:
        app: myweb
    spec:
      nodeName: lain1
      containers:
        - name: web1test
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/stress"]
          volumeMounts:
            - name: app
              mountPath: /app
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "400m" 	# 1物理核=1000个微核(millicores) 1000m=1CPU
              memory: "512Mi"
          ports:
            - containerPort: 8080
      volumes:
        - name: app
          hostPath:
            path: /home/txl/goapi
---
apiVersion: v1
kind: Service
metadata:
  name: web1
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: myweb
```

- `requests` 节点用来设置各容器需要的最小资源
- `limits` 节点用于限制运行时容器占用的资源

### 创建

现在创建一个 HPA 资源对象，可以使用命令创建

```bash
$ kubectl autoscale deployment web1 --min=1 --max=5 --cpu-percent=20

$ kubectl get hpa
NAME   REFERENCE         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
web1   Deployment/web1   0%/20%    1         5         1          12m
```

此命令创建了一个关联资源 web1 的 HPA，最小的 Pod 副本数为1，最大为5。HPA 会根据设定的 cpu 使用率（20%）动态的增加或者减少 Pod 数量。

也可以使用 yaml 来创建

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: web1hpa
  namespace: default
spec:
  minReplicas: 1
  maxReplicas: 5
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web1
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # 使用率
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 50
```

你还可以指定资源度量指标使用绝对数值，而不是百分比，你需要将 target.type 从 Utilization 替换成 AverageValue，同时设置 target.averageValue 而非 target.averageUtilization 的值

```yaml
 metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: AverageValue
          averageValue: 230m   # 使用量
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400m
```

### 压测

```bash
$ sudo yum -y install httpd-tools

$ ab -n 10000 -c 10  http://web1/
```

可以看到，HPA 已经开始工作，副本数量已经从原来的1变成了4个

```bash
$ kubectl get hpa
NAME   REFERENCE         TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
web1   Deployment/web1   192%/20%   1         5         4          107s
```

查看 HPA 资源工作过程

```bash
$ kubectl describe hpa web1
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  20m   horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  20m   horizontal-pod-autoscaler  New size: 5; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  14m   horizontal-pod-autoscaler  New size: 1; reason: All metrics below target
```
