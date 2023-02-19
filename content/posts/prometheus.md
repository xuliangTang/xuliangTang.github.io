---
title: "Prometheus 入门"
date: 2023-02-19
draft: false
Categories: [kubernetes]
---

为了提升服务的稳定性，我们需要不断的收集数据，分析数据，监控数据，进而优化能优化的点，Prometheus 在这方面就为我们提供了很好的监控方案。Prometheus 是一个开源的监控和报警系统，它将我们关心的指标值通过 PULL 的方式获取并存储为时间序列数据。如果单从它的收集功能来讲，我们也可以通过 mysql、redis 等方式实现。然而，这些数据是在每时每刻产生的，其庞大的规模需要我们好好的考虑其存储方式。另外，这些监控数据大多数时候是跟统计相关的，比如数据与时间的分布情况等

由于 Prometheus 的关注重点在于指标值以及时间点这两个因素，所以外部程序对它的接入成本非常的低。这种易用性可以让我们对数据进行多维度、多角度的观察和分析，使得监控的效果更加具体化，例如内存消耗、网络利用率、请求连接数等

除此之外，Prometheus 还具备了操作简单、可拓展的数据收集和强大的查询语言等特性，这些特性能帮助我们在问题出现的时候，快速告警并定位错误。所以现在很多微服务基础设施都会选择接入 Prometheus，像 k8s、云原生等

## 整体架构

Prometheus 为了保证它的拓展性、可靠性，在除了提供核心的 server 外还提供了很多生态组件，为了不增加理解的复杂度，我们先从上帝视角，看看它的核心 Prometheus server

![image-20230217221855601](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302172219125.png)

可以看到，Prometheus server 的处理链路很清晰，其实就是数据收集-数据存储-数据查询。当然，一个完善的系统肯定会衍生出许多组件来支撑它的特性。所以我们会看到，在 Prometheus 架构里还存在着其他的组件，例如：

- Pushgateway：为监控节点提供 Push 功能，再由 Prometheus server 到 Pushgateway 集中 Pull 数据
- Targets Discover：根据服务发现获取监控节点的地址
- PromQL：针对指标数据查询的语言，类似 SQL
- Alertmanager：根据配置规则以及指标分析，提供告警服务

最后，Prometheus 的整体架构如下

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302172220283.jpeg)

## 部署

### kube-state-metrics

[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 是官方提供的 k8s 内部各个组件的状态指标，通过监听 API Server 生成有关资源对象（如 Deployment、Node、Pod）的状态指标（如副本数有几个、pod 状态、重启了几次）

参考配置：[kube-state-metrics/examples/standard](https://github.com/kubernetes/kube-state-metrics/tree/main/examples/standard)

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 2.8.0
  name: kube-state-metrics
  namespace: kube-system
spec:
  type: NodePort	# 这里改成了NodePort
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
    nodePort: 32280
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
```

### node_exporter

[node-exporter](https://github.com/prometheus/node_exporter) 用来监控节点 CPU、内存、磁盘、I/O 等信息

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: prometheus
  labels:
    name: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
      - name: node-exporter
        image: bitnami/node-exporter:1.2.2
        ports:
        - containerPort: 9100
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        securityContext:
          privileged: true
        args:
        - --path.procfs
        - /host/proc
        - --path.sysfs
        - /host/sys
        - --collector.filesystem.ignored-mount-points
        - '"^/(sys|proc|dev|host|etc)($|/)"'
        volumeMounts:
        - name: dev
          mountPath: /host/dev
        - name: proc
          mountPath: /host/proc
        - name: sys
          mountPath: /host/sys
        - name: rootfs
          mountPath: /rootfs
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      volumes:
        - name: proc
          hostPath:
            path: /proc # CPU信息、内存信息、内核信息等
        - name: dev
          hostPath:
            path: /dev	# 存放与设备（包括外设）有关的文件，如打印机、USB、串口/并口等
        - name: sys
          hostPath:
            path: /sys # 硬件设备的驱动程序信息
        - name: rootfs
          hostPath:
            path: /
```

- HostPID：控制 Pod 中容器是否可以共享宿主上的进程 ID 空间 
- HostIPC：控制 Pod 容器是否可共享宿主上的 IPC  (进程通信)
- hostNetwork: true：Pod 允许使用宿主机网络
- privileged: true：容器以特权方式允许（为了能够访问宿主机所有设备）

### prometheus

**1. 配置文件**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 5s	# 默认抓取间隔
    scrape_configs:
    - job_name: 'prometheus-state-metrics'
      static_configs:
      - targets: ['192.168.0.111:32280']
    - job_name: 'node-exporter'
      static_configs:
      - targets: ['192.168.0.111:9100', '192.168.0.105:9100']
```

**2. deployment**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    name: prometheus
spec:
  selector:
    matchLabels:
      app: prometheus
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      nodeName: lain1
      # serviceAccountName: myprometheus
      containers:
        - name: prometheus
          image: prom/prometheus:v2.30.0
          command:
            - prometheus
          args:
            - "--config.file=/config/prometheus.yml"
            - "--web.enable-lifecycle"
          volumeMounts:
            - name: config
              mountPath: /config
          ports:
            - containerPort: 9090
      volumes:
        - name: config
          configMap:
            name: prometheus
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: prometheus
  labels:
    app: prometheus
spec:
  type: ClusterIP
  ports:
    - port: 9090
      targetPort: 9090
  selector:
    app: prometheus
```

**3. 更新**

```bash
kubectl get svc -n prometheus
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
prometheus   ClusterIP   10.107.82.59   <none>        9090/TCP   4d6h

curl -X POST http://10.107.82.59:9090/-/reload
```



## 服务自动发现

Prometheus 添加被监控端有两种方式：

- 静态配置：手动配置
- 服务发现：动态发现需要监控的实例，服务发现支持来源有 consul_sd_configs、file_sd_configs、kubernetes_sd_configs 等

**1. 创建 RBAC 资源**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myprometheus
  namespace: prometheus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: myprometheus-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
    - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: myprometheus-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: myprometheus-clusterrole
subjects:
- kind: ServiceAccount
  name: myprometheus
  namespace: prometheus
```

**2. deployment 指定 serviceAccount**

```yaml
serviceAccountName: myprometheus
```

**2. 添加配置**

```yaml
global:
  scrape_interval: 5s
scrape_configs:
- job_name: 'prometheus-state-metrics'
  static_configs:
  - targets: ['192.168.0.111:32280']
- job_name: 'k8s-node'
  metrics_path: /metrics
  kubernetes_sd_configs:
  - api_server: https://kubernetes.default.svc
    role: node
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      # insecure_skip_verify: true
  relabel_configs:	# 在抓取目标之前动态重写目标的标签集
  - source_labels: [__address__]	# 源标签从现有标签中选择值
    regex: '(.*):10250'	# 与提取的值匹配的正则表达式
    replacement: '${1}:9100' # 替换值
    target_label: __address__ # 被替换的标签
    action: replace	# 匹配执行的操作(replace 、keep、drop、labelmap、labeldrop)
```

### Pod 监控

cAdvisor 负责单节点内部的容器和节点资源使用统计，已经集成在 Kubelet 内部

```bash
# 查看节点lain1统计
curl https://192.168.0.111:6443/api/v1/nodes/lain1/proxy/metrics/cadvisor \
--header "Authorization: Bearer $TOKEN" --cacert ca.crt
```

添加配置

```yaml
- job_name: 'kubelet'
  scheme: https
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  tls_config:
     ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  kubernetes_sd_configs:
  - api_server: https://kubernetes.default.svc
    role: node
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  relabel_configs:
  - target_label: __address__
    replacement: kubernetes.default.svc
  - source_labels: [__meta_kubernetes_node_name]
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```



## 基本查询

[PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) (Prometheus Query Language) ，Prometheus的数据查询 DSL 语言 

有四种类型：

- 即时向量（instant vector）：包含每个时间序列的单个样本的一组时间序列，共享相同的时间戳
- 范围向量（Range vector）：包含每个时间序列随时间变化的数据点的一组时间序列
- 标量（Scalar）：一个简单的数字浮点值
- 字符串（String）：一个简单的字符串值（目前未被使用）

### 即时向量

url 请求参数：

- query=<string> : PromQL 表达式
- time=<rfc3339 | unix_timestamp> : 用于指定用于计算 PromQL的时间戳。可选参数，默认情况下使用当前系统时间

示例：

`http://10.107.82.59:9090/api/v1/query?query=container_memory_usage_bytes{namespace="default",pod=~"nginx.*"}`

### 区间向量

区间向量表达式和瞬时向量表达式之间的差异在于需要定义时间范围，通过时间范围选择器 [] 进行定义（s秒 m分钟 h小时 d天 w周 y年）。如：xxx[5m] 查询过去5分钟的数列；xxx{} offset 5m 查询5分钟之前的偏移量

URL 请求参数：

- query=<string> : PromQL 表达式
- start=<rfc3339 | unix_timestamp> : 起始时间戳
- end=<rfc3339 | unix_timestamp> : 结束时间戳
- step=<duration | float> : 查询时间步长，时间区间内每 step 秒执行一次

### 聚合操作符

Prometheus 支持以下内置聚合运算符，这些运算符可以是用于聚合单个即时向量的元素，从而产生新的 具有聚合值的较少元素的向量：

- sum（求和）
- min（最小值）
- max（最大值）
- avg（平均值）
- stddev（标准差）
- stdvar（方差）
- count（元素个数）
- count_values（等于某值的元素个数）
- bottomk（最小的 k 个元素）
- topk（最大的 k 个元素）
- quantile（分位数）

### 函数

参考：[查询函数](https://prometheus.io/docs/prometheus/latest/querying/functions/)，如：rate(xxx[5m]) 查询指标5分钟内，平均每秒数据



## 自定义指标

客户端库：[prometheus/client_golang](https://github.com/prometheus/client_golang)

四个基本指标类型（时序数据）：

- Counter（计数器）：表示一个单调递增的指标数据，请求次数、错误数量等
- Gauge（计量器）：代表一个可以任意变化的指标数据，可增可减。场景有：协程数量、CPU、Memory 、业务队列的数量等
- Histogram（累积直方图）：主要是样本观测数据,在一段时间范围内对数据进行采样。如请求持续时间或响应大小等，这点往往可以配合链路追踪系统（如 jaeger）
- Summary（摘要统计）：和直方图类似，也是样本观测，但是它提供了样本值的分位数、所有样本值的大小总和、样本总量

以下是一个计数器自定义指标示例：

**1. 代码**

```go
func init() {
	prometheus.MustRegister(prodsVisit)
}

var prodsVisit = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "lain_prods_visit",
	},
	[]string{"prod_id"},
)

func main() {
	r := gin.New()
	r.GET("/prods/visit", func(c *gin.Context) {
		pid_str := c.Query("pid")
		_, err := strconv.Atoi(pid_str)
		if err != nil {
			c.JSON(400, gin.H{"message": "error pid"})
		}
		prodsVisit.With(prometheus.Labels{
			"prod_id": pid_str,
		}).Inc()
		c.JSON(200, gin.H{"message": "OK"})
	})
	r.GET("/metrics", gin.WrapH(promhttp.Handler()))
	r.Run(":8080")
}
```

**2. 部署**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prodmetrics
  namespace: default
spec:
  selector:
    matchLabels:
      app: prodmetrics
  replicas: 1
  template:
    metadata:
      labels:
        app: prodmetrics
    spec:
      nodeName: lain1
      containers:
        - name: prodmetrics
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          workingDir: /app
          command: ["./prodmetrics"]
          volumeMounts:
            - name: app
              mountPath: /app
          ports:
            - containerPort: 8080
      volumes:
        - name: app
          hostPath:
             path: /home/txl/prometheus/prodmetrics
---
apiVersion: v1
kind: Service
metadata:
  name: prodmetrics
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: prodmetrics
```

**3. prometheus 增加配置**

```yaml
- job_name: 'lain-prodmetrics'
  static_configs:
  - targets: ['prodmetrics']	# service名
```

**4. 重新加载配置**

```bash
curl -X POST prometheus.prometheus.svc.cluster.local:9090/-/reload
```

### 自动发现

**1. service 加入注解**

```yaml
metadata:
  name: prodmetrics
  namespace: default
  annotations:
    scrape: "true"
```

**2. 修改配置**

```yaml
- job_name: 'lain-prodmetrics-auto'
  metrics_path: /metrics
  kubernetes_sd_configs:
  - api_server: https://kubernetes.default.svc
    role: service	# 发现类型为service
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_annotation_scrape]
    regex: true
    action: keep	# 丢弃没有匹配到scrape=true的service
```

**3. 重新加载配置**



## Prometheus Adapter

kubernetes 主要通过两类 API 来获取资源使用指标：

- resource metrics API： 核心组件提供监控指标，如容器 CPU 、内存。 经典的实现就是 metres-server（Group 是 metrics.k8s.io）
-  custom metrics API：自定义指标。比较常用的就是 prometheus-adapter，它也支持第一种 API（Group 是 custom.metrics.k8s.io）

kubernetes apiserver 用于将 kubernetes 的功能通过 restapi 的方式暴露出去，给其他组件使用。对于非核心的功能，k8s 提供对应的 API，具体的实现由第三方软件完成，使用者在使用扩展 apiserver 时，只需要将其注册到 kube-aggregator（kubernetes apiserver 的功能），aggregator 就会将对于这个 api 的请求转发到这个扩展 apiserver 上。Prometheus Adapter 就是一种对 Prometheus adapter 的实现

> adapter 是一个聚合 API，它负责把 prometheus 采集的数据爬虫爬回来，放到缓存中，这样就可以通过 k8s api 的方式来查 prometheus 的数据

### 部署

**1. 创建资源**

参考 [示例 yamls](/prometheusAdapter/prometheus-adapter.zip)

**2. 生成 Secret**

进入 k8s CA 证书所在目录（kubeadm 安装默认在 master 节点的 /etc/kubernetes/pki）

```bash
openssl genrsa -out serving.key 2048
openssl req -new -key serving.key -out serving.csr -subj "/CN=serving"
openssl x509 -req -in serving.csr -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -out serving.crt -days 3650

kubectl create secret generic cm-adapter-serving-certs --from-file=serving.crt=./serving.crt --from-file=serving.key -n custom-metrics
```

验证

```bash
kubectl get apiservice | grep custom-metrics
```

**3. 查看**

如获取 default 命名空间下所有 pod 的 cpu  和内存使用情况

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/cpu_usage"

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/memory_usage_bytes"
```

### 创建自定义指标

下面是一个获取所有 prods（上方创建的 lain_prods_visit 指标）1分钟内平均每秒请求数的总和的示例：

**1. 增加 prometheus 配置**

通过抓取的方式移植到服务发现的标签中

```yaml
- job_name: 'lain-prodmetrics-auto'
  metrics_path: /metrics
  kubernetes_sd_configs:
    - api_server: https://kubernetes.default.svc
      role: service
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  relabel_configs:
    - source_labels: [__meta_kubernetes_service_annotation_scrape]
      regex: true
      action: keep
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: namespace	# 移植namespace
    - source_labels: [__meta_kubernetes_service_name]
      action: replace
      target_label: svcname	# 移植service_name
```

**2. 增加 adapter 配置**

custom-metrics-config-map.yaml：

```yaml
- seriesQuery: '{__name__=~"^lain_.*",namespace!=""}'
  seriesFilters: []
  resources:
    overrides:
      namespace:
        resource: namespace
      svcname:
        resource: service
  name:
    matches: ^lain_(.*)$
    as: ""
  metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[1m])) by (<<.GroupBy>>)
```

- seriesQuery：用于确定需要查询的指标集合
- seriesFilters：用于过滤指标
  - is: <regex>：匹配包含该正则表达式的 metrics
  - isNot: <regex>：匹配不包含该正则表达式的 metrics
- resources：把指标的标签和 k8s 的资源类型（必须是真实资源）关联。通过 seriesQuery 查询到的只是指标，如果要查询某个 pod 的指标，肯定要将它的名称和所在的名称空间作为指标的标签进行查询。有两种添加标签的方式，一种是`overrides`，另一种是 `template`
- name：用来给指标重命名，as 默认值为 $1
- metricsQuery：使用 Go 模板语法将 URL 请求转变为 Prometheus 的请求，常见写法：`sum(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)`
  - Series：metric 名称（如：lain_prods_visit）
  - LabelMatchers：附加的标签，是以逗号分割的 objects（如特定命名空间和 resource），因此我们要在之前使用 resources 进行关联
  - GroupBy：以逗号分割的 label 的集合（当前表示 LabelMatchers 中的 group-resource label），同样需要使用 resources 进行关联

**3. 查询**

通过 metricsQuery 进行查询

```bash
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/prodmetrics/prods_visit"
```

相当于执行了

```
sum(rate(lain_prods_visit[1m])) by(svcname)
```
