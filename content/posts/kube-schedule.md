---
title: "调度器 kube-schedule"
date: 2022-12-13
draft: false
Categories: [kubernetes]
---

Kube-scheduler 是 Kubernetes 集群默认的调度器，并且是控制面中一个核心组件。scheduler 通过 kubernetes 的监测（Watch）机制来发现集群中新创建且尚未被调度到 Node 上的 Pod。 scheduler 会将发现的每一个未调度的 Pod 调度到一个合适的 Node 上来运行。 scheduler会依据下文的调度原则来做出调度选择。

![kube-scheduler](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2393007-20220220103712446-1739360732.png)

对于新创建的 pod 或其他未调度的 pod来讲，kube-scheduler 选择一个最佳节点供它们运行。但是，Pod 中的每个容器对资源的要求都不同，每个 Pod 也有不同的要求。因此，需要根据具体的调度要求对现有节点进行过滤。

在Kubernetes集群中，满足 Pod 调度要求的节点称为可行节点 （feasible nodes *FN*） 。如果没有合适的节点，则 pod 将保持未调度状态，直到调度程序能够放置它。也就是说，当我们创建 Pod 时，如果长期处于 Pending 状态，这个时候应该看你的集群调度器是否因为某些问题没有合适的节点了

调度器为 Pod 找到 FN 后，然后运行一组函数对 FN 进行评分，并在 FN 中找到得分最高的节点来运行 Pod。

调度策略在决策时需要考虑的因素包括个人和集体资源需求、硬件/软件/策略约束 （constraints）、亲和性 (affinity) 和反亲和性（ anti-affinity ）规范、数据局部性、工作负载间干扰等。

基本调度流程：

- 发布 Pod
- ControllerManager 会把 Pod 加入待调度队列
- kube-scheduler 决定调度到哪个 node，然后写入 etcd
- 被选中节点中的 kubelet 开始工作（pull image、启动 Pod）



## 如何为 Pod 选择节点？

kube-scheduler 给一个 Pod 做调度选择时包含两个步骤：

- 过滤 (Filtering)
- 打分 (Scoring)

过滤也被称为预选 （Predicates），该步骤会找到可调度的节点集，然后通过是否满足特定资源的请求，例如通过 PodFitsResources 过滤器检查候选节点是否有足够的资源来满足 Pod 资源的请求。这个步骤完成后会得到一个包含合适的节点的列表（通常为多个），如果列表为空，则Pod不可调度。

打分也被称为优选（Priorities），在该步骤中，会对上一个步骤的输出进行打分，Scheduer 通过打分的规则为每个通过 Filtering 步骤的节点计算出一个分数。

完成上述两个步骤之后，kube-scheduler 会将Pod分配给分数最高的 Node，如果存在多个相同分数的节点，会随机选择一个。

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/kube-scheduler-filter%20(1)%20(2).jpeg)

## 将 Pod 指派给节点

你可以约束一个 Pod 以便限制其只能在特定的节点上运行，或优先在特定的节点上运行。有几种方法可以实现这点，推荐的方法都是用标签选择算符来进行选择。 通常这样的约束不是必须的，因为调度器将自动进行合理的放置，但在某些情况下，你可能需要进一步控制 Pod 被部署到哪个节点。

给节点 lain1 添加一个标签 disktype=ssd

```bash
$ kubectl label nodes lain1 disktype=ssd

# 删除标签
$ kubectl label nodes lain1 disktype-
```

### nodeSelector

设置你希望目标节点所具有的节点标签。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd	# 该Pod将被调度到有disktype=ssd标签的节点
```

### 节点亲和性 nodeAffinity

节点亲和性有两种：

- **requiredDuringSchedulingIgnoredDuringExecution**：调度器只有在规则被满足的时候才能执行调度。此功能类似于
- **preferredDuringSchedulingIgnoredDuringExecution**：调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。

#### 强制的节点亲和性调度

下面的 pod 只会调度到具有 disktype=ssd 标签的节点上

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

#### 首选的节点亲和性调度

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1	# 权重
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

### 节点污点和容忍度

节点亲和性使 Pod 被吸引到一类特定的节点，污点 (Taint) 则相反，它使节点能够排斥一类特定的 Pod。污点有三种类型：

- **NoSchedule**：不会将 Pod 调度到该节点
- **PreferNoSchedule**：尽量避免将 Pod 调度到该节点上
- **NoExecute**：任何不能忍受这个污点的 Pod 都会马上被驱逐

一些内置的污点：

- node.kubernetes.io/not-ready：节点未准备好。这相当于节点状况 Ready 的值为 False
- node.kubernetes.io/unreachable：节点控制器访问不到节点. 这相当于节点状况 Ready 的值为 Unknown
- node.kubernetes.io/memory-pressure：节点存在内存压力
- node.kubernetes.io/disk-pressure：节点存在磁盘压力
- node.kubernetes.io/pid-pressure: 节点的 PID 压力
- node.kubernetes.io/network-unavailable：节点网络不可用
- node.kubernetes.io/unschedulable: 节点不可调度
- node.cloudprovider.kubernetes.io/uninitialized：如果 kubelet 启动时指定了一个“外部”云平台驱动， 它将给当前节点添加一个污点将其标志为不可用。在 cloud-controller-manager 的一个控制器初始化这个节点后，kubelet 将删除这个污点

容忍度 (Toleration) 是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。 容忍度允许调度但并不保证调度：作为其功能的一部分， 调度器也会评估其他参数。

```bash
# 查看节点的污点
$ kubectl describe node lain1 | grep Taints

# 给节点打一个污点
$ kubectl taint nodes lain1 key1=value1:NoSchedule

# 删除污点
$ kubectl taint node lain1 key1:NoSchedule-
```

使用容忍

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "key1"		# 对应污点key
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
```

### Pod 亲和性

Pod 间亲和性与反亲和性使你可以基于已经在节点上运行的 Pod 的标签来约束 Pod 可以调度到的节点，而不是基于节点上的标签。

> Pod 亲和性和反亲和性都需要相当的计算量，因此会在大规模集群中显著降低调度速度。 不建议在包含数百个节点的集群中使用这类设置

与节点亲和性类似，Pod 的亲和性与反亲和性也有两种类型：

- **requiredDuringSchedulingIgnoredDuringExecution**
- **preferredDuringSchedulingIgnoredDuringExecution**

实例资源清单

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx1
spec:
  selector:
    matchLabels:
      app: ngx1
  replicas: 1
  template:
    metadata:
      labels:
        app: ngx1
    spec:
      nodeName: lain1
      containers:
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
```

下面这个 Pod 必须调度到具有 disktype 标签的节点上，并且集群中至少有一个位于该可用区的节点上运行着带有 app=ngx1 标签的 Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx2
spec:
  selector:
    matchLabels:
      app: ngx2
  replicas: 1
  template:
    metadata:
      labels:
        app: ngx2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - ngx1
              topologyKey: disktype
      containers:
        - name: ngx2
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
```
