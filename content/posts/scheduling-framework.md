---
title: "自定义 POD 调度 Scheduling Framework"
date: 2023-02-26
draft: false
Categories: [kubernetes]
math: true
---

kube-scheduler 是 kubernetes 的核心组件之一，主要负责整个集群资源的调度功能，根据特定的调度算法和策略，将 Pod 调度到最优的工作节点上面去，从而更加合理、更加充分的利用集群的资源

Kubernetes v1.15 版本中引入了可插拔架构的调度框架，调库框架向现有的调度器中添加了一组插件化的 API，该 API 在保持调度程序“核心”简单且易于维护的同时，使得大部分的调度功能以插件的形式存在，参考文档：[调度框架](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/scheduling-framework/)

## 调度框架

调度框架定义了一组扩展点，用户可以实现扩展点定义的接口来定义自己的调度逻辑（我们称之为**扩展**），并将扩展注册到扩展点上，调度框架在执行调度工作流时，遇到对应的扩展点时，将调用用户注册的扩展。调度框架在预留扩展点时，都是有特定的目的，有些扩展点上的扩展可以改变调度程序的决策方法，有些扩展点上的扩展只是发送一个通知

我们知道每当调度一个 Pod 时，都会按照两个过程来执行：**调度过程**和**绑定过程**

调度过程为 Pod 选择一个合适的节点，绑定过程则将调度过程的决策应用到集群中（也就是在被选定的节点上运行 Pod），将调度过程和绑定过程合在一起，称之为**调度上下文（scheduling context）**。需要注意的是调度过程是`同步`运行的（同一时间点只为一个 Pod 进行调度），绑定过程可异步运行（同一时间点可并发为多个 Pod 执行绑定）

调度过程和绑定过程遇到如下情况时会中途退出：

- 调度程序认为当前没有该 Pod 的可选节点
- 内部错误

这个时候，该 Pod 将被放回到 **待调度队列**，并等待下次重试

## 扩展点

下图展示了调度框架中的调度上下文及其中的扩展点，一个扩展可以注册多个扩展点，以便可以执行更复杂的有状态的任务，参考：[调度器配置](https://kubernetes.io/zh-cn/docs/reference/scheduling/config/)

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302261944744.png)

1. `QueueSort` 扩展用于对 Pod 的待调度队列进行排序，以决定先调度哪个 Pod，`QueueSort` 扩展本质上只需要实现一个方法 `Less(Pod1, Pod2)` 用于比较两个 Pod 谁更优先获得调度即可，同一时间点只能有一个 `QueueSort` 插件生效
2. `Pre-filter` 扩展用于对 Pod 的信息进行预处理，或者检查一些集群或 Pod 必须满足的前提条件，如果 `pre-filter` 返回了 error，则调度过程终止
3. `Filter` 扩展用于排除那些不能运行该 Pod 的节点，对于每一个节点，调度器将按顺序执行 `filter` 扩展；如果任何一个 `filter` 将节点标记为不可选，则余下的 `filter` 扩展将不会被执行。调度器可以同时对多个节点执行 `filter` 扩展
4. `Post-filter` 是一个通知类型的扩展点，调用该扩展的参数是 `filter` 阶段结束后被筛选为**可选节点**的节点列表，可以在扩展中使用这些信息更新内部状态，或者产生日志或 metrics 信息
5. `PreScore` 扩展用于执行 “前置评分（pre-scoring）” 工作，即生成一个可共享状态供 Score 插件使用。 如果 PreScore 插件返回错误，则调度周期将终止
6. `Score` 评分插件用于对通过过滤阶段的节点进行排名。调度器为每个节点调用每个评分插件。 将有一个定义明确的整数范围，代表最小和最大分数。 在标准化评分阶段之后，调度器将根据配置的插件权重 合并所有插件的节点分数
7. `NormalizeScore` 扩展在调度器对节点进行最终排序之前修改每个节点的评分结果，注册到该扩展点的扩展在被调用时，将获得同一个插件中的 `scoring` 扩展的评分结果作为参数，调度框架每执行一次调度，都将调用所有插件中的一个 `normalize scoring` 扩展一次
8. `Reserve` 是一个通知性质的扩展点，有状态的插件可以使用该扩展点来获得节点上为 Pod 预留的资源，该事件发生在调度器将 Pod 绑定到节点之前，目的是避免调度器在等待 Pod 与节点绑定的过程中调度新的 Pod 到节点上时，发生实际使用资源超出可用资源的情况。（因为绑定 Pod 到节点上是异步发生的）。这是调度过程的最后一个步骤，Pod 进入 reserved 状态以后，要么在绑定失败时触发 Unreserve 扩展，要么在绑定成功时，由 Post-bind 扩展结束绑定过程
9. `Permit` 扩展在每个 Pod 调度周期的最后调用，用于阻止或者延迟 Pod 与节点的绑定。Permit 扩展可以做下面三件事中的一项：
   - approve（批准）：当所有的 permit 扩展都 approve 了 Pod 与节点的绑定，调度器将继续执行绑定过程
   - deny（拒绝）：如果任何一个 permit 扩展 deny 了 Pod 与节点的绑定，Pod 将被放回到待调度队列，此时将触发 `Unreserve` 扩展
   - wait（等待）：如果一个 permit 扩展返回了 wait，则 Pod 将保持在 permit 阶段，同时该 Pod 的绑定周期启动时即直接阻塞直到得到批准，如果超时事件发生，wait 状态变成 deny，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
10. `Pre-bind` 扩展用于在 Pod 绑定之前执行某些逻辑。例如，pre-bind 扩展可以将一个基于网络的数据卷挂载到节点上，以便 Pod 可以使用。如果任何一个 `pre-bind` 扩展返回错误，Pod 将被放回到待调度队列，此时将触发 Unreserve 扩展
11. `Bind` 扩展用于将 Pod 绑定到节点上：
    - 只有所有的 pre-bind 扩展都成功执行了，bind 扩展才会执行
    - 调度框架按照 bind 扩展注册的顺序逐个调用 bind 扩展
    - 具体某个 bind 扩展可以选择处理或者不处理该 Pod
    - 如果某个 bind 扩展处理了该 Pod 与节点的绑定，余下的 bind 扩展将被忽略
    - 如果失败，则执行 Unreverse 扩展点将预先消费的资源释放掉(如 PVC 和 PV)，并将 Pod 从调度队列中删除
12. `Post-bind` 是一个通知性质的扩展，是整个调度的最后一步：
    - Post-bind 扩展在 Pod 成功绑定到节点上之后被动调用
    - Post-bind 扩展是绑定过程的最后一个步骤，可以用来执行资源清理的动作
13. `Unreserve` 是一个通知性质的扩展，如果为 Pod 预留了资源，Pod 又在被绑定过程中被拒绝绑定，则 unreserve 扩展将被调用。Unreserve 扩展应该释放已经为 Pod 预留的节点上的计算资源。在一个插件中，reserve 扩展和 unreserve 扩展应该成对出现

官方实现了一些扩展点插件：[kubernetes/pkg/scheduler/framework/plugins](https://github.com/kubernetes/kubernetes/tree/master/pkg/scheduler/framework/plugins)

如果我们要实现自己的插件，必须向调度框架注册插件并完成配置，另外还必须实现扩展点接口，参考项目：[kubernetes-sigs/scheduler-plugins](https://github.com/kubernetes-sigs/scheduler-plugins)

扩展的调用顺序如下：

- 如果某个扩展点没有配置对应的扩展，调度框架将使用默认插件中的扩展
- 如果为某个扩展点配置且激活了扩展，则调度框架将先调用默认插件的扩展，再调用配置中的扩展
- 默认插件的扩展始终被最先调用，然后按照 `KubeSchedulerConfiguration` 中扩展的激活 `enabled` 顺序逐个调用扩展点的扩展
- 可以先禁用默认插件的扩展，然后在 `enabled` 列表中的某个位置激活默认插件的扩展，这种做法可以改变默认插件的扩展被调用时的顺序

## 基本示例

下面是一个实现 **PreFilter** 扩展点示例，用来限制命名空间内 POD 最大调度数量

```go
// go.mod

require (
	k8s.io/api v0.22.3
	k8s.io/apimachinery v0.22.3
	k8s.io/component-base v0.22.3
	k8s.io/klog/v2 v2.9.0
	k8s.io/kube-scheduler v0.22.3 // indirect
	k8s.io/kubernetes v1.22.3
)

replace (
	k8s.io/api => k8s.io/api v0.22.3
	k8s.io/apiextensions-apiserver => k8s.io/apiextensions-apiserver v0.22.3
	k8s.io/apimachinery => k8s.io/apimachinery v0.22.3
	k8s.io/apiserver => k8s.io/apiserver v0.22.3
	k8s.io/cli-runtime => k8s.io/cli-runtime v0.22.3
	k8s.io/client-go => k8s.io/client-go v0.22.3
	k8s.io/cloud-provider => k8s.io/cloud-provider v0.22.3
	k8s.io/cluster-bootstrap => k8s.io/cluster-bootstrap v0.22.3
	k8s.io/code-generator => k8s.io/code-generator v0.22.3
	k8s.io/component-base => k8s.io/component-base v0.22.3
	k8s.io/component-helpers => k8s.io/component-helpers v0.22.3
	k8s.io/controller-manager => k8s.io/controller-manager v0.22.3
	k8s.io/cri-api => k8s.io/cri-api v0.22.3
	k8s.io/csi-translation-lib => k8s.io/csi-translation-lib v0.22.3
	k8s.io/kube-aggregator => k8s.io/kube-aggregator v0.22.3
	k8s.io/kube-controller-manager => k8s.io/kube-controller-manager v0.22.3
	k8s.io/kube-proxy => k8s.io/kube-proxy v0.22.3
	k8s.io/kube-scheduler => k8s.io/kube-scheduler v0.22.3
	k8s.io/kubectl => k8s.io/kubectl v0.22.3
	k8s.io/kubelet => k8s.io/kubelet v0.22.3
	k8s.io/kubernetes => k8s.io/kubernetes v1.22.3
	k8s.io/legacy-cloud-providers => k8s.io/legacy-cloud-providers v0.22.3
	k8s.io/metrics => k8s.io/metrics v0.22.3
	k8s.io/mount-utils => k8s.io/mount-utils v0.22.3
	k8s.io/pod-security-admission => k8s.io/pod-security-admission v0.22.3
	k8s.io/sample-apiserver => k8s.io/sample-apiserver v0.22.3
)
```

```go
// main.go

import (
	"fmt"
	"k8s.io/component-base/logs"
	"k8s.io/kubernetes/cmd/kube-scheduler/app"
	"os"
	"pod-scheduler/lib"
)

func main() {
	command := app.NewSchedulerCommand(
		app.WithPlugin(lib.TestSchedulingName, lib.NewTestScheduler),
	)
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		_, _ = fmt.Fprintf(os.Stdout, "%v\n", err)
		os.Exit(1)
	}
}
```

```go
// lib/test-scheduling.go

import (
	"context"
	"fmt"
	coreV1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/client-go/informers"
	"k8s.io/klog/v2"
	"k8s.io/kubernetes/pkg/scheduler/framework"
	frameworkRuntime "k8s.io/kubernetes/pkg/scheduler/framework/runtime"
)

const (
	TestSchedulingName = "TestScheduling"
)

type TestScheduler struct {
	fact informers.SharedInformerFactory
	args *Args
}

type Args struct {
	MaxPods int `json:"maxPods,omitempty"`	// 最大调度数量
}

func (s *TestScheduler) AddPod(ctx context.Context, state *framework.CycleState, podToSchedule *coreV1.Pod, podInfoToAdd *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
	return nil
}

func (s *TestScheduler) RemovePod(ctx context.Context, state *framework.CycleState, podToSchedule *coreV1.Pod, podInfoToRemove *framework.PodInfo, nodeInfo *framework.NodeInfo) *framework.Status {
	return nil
}

func (s *TestScheduler) PreFilter(ctx context.Context, state *framework.CycleState, p *coreV1.Pod) *framework.Status {
	// 业务逻辑实现
	klog.V(3).Infof("当前被preFilter的POD名称是：%s\n", p.Name)
	pods, err := s.fact.Core().V1().Pods().Lister().Pods(p.Namespace).List(labels.Everything())
	if err != nil {
		return framework.NewStatus(framework.Error, err.Error())
	}

	if s.args.MaxPods > 0 && len(pods) > s.args.MaxPods {
		return framework.NewStatus(framework.Unschedulable, fmt.Sprintf("POD数量超过上限%d", s.args.MaxPods))
	}

	return framework.NewStatus(framework.Success)
}

func (s *TestScheduler) PreFilterExtensions() framework.PreFilterExtensions {
	return s
}

func (*TestScheduler) Name() string {
	return TestSchedulingName
}

var _ framework.PreFilterPlugin = &TestScheduler{}

func NewTestScheduler(configuration runtime.Object, handle framework.Handle) (framework.Plugin, error) {
	args := &Args{}
	// 得到调度插件自定义参数，反编码为struct
	if err := frameworkRuntime.DecodeInto(configuration, args); err != nil {
		return nil, err
	}
	return &TestScheduler{
		fact: handle.SharedInformerFactory(),
		args: args,
	}, nil
}
```

然后可以当成普通的应用用一个 Deployment 控制器来部署即可，由于我们需要去获取集群中的一些资源对象，所以当然需要申请 RBAC 权限，然后同样通过 `--config` 参数来配置我们的调度器

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: test-scheduling-clusterrole
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
      - events
    verbs:
      - create
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - delete
      - get
      - list
      - watch
      - update
  - apiGroups:
      - ""
    resources:
      - bindings
      - pods/binding
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - pods/status
    verbs:
      - patch
      - update
  - apiGroups:
      - ""
    resources:
      - replicationcontrollers
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
      - extensions
    resources:
      - replicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - persistentvolumeclaims
      - persistentvolumes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - namespaces
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "storage.k8s.io"
    resources: ['*']
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "coordination.k8s.io"
    resources:
      - leases
    verbs:
      - create
      - get
      - list
      - update
  - apiGroups:
      - "events.k8s.io"
    resources:
      - events
    verbs:
      - create
      - patch
      - update

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-scheduling-sa
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: test-scheduling-clusterrolebinding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: test-scheduling-clusterrole
subjects:
  - kind: ServiceAccount
    name: test-scheduling-sa
    namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-scheduling-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1beta2
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: false
    profiles:
      - schedulerName: test-scheduling
        plugins:
          preFilter:
            enabled:
            - name: "TestScheduling"
        pluginConfig:
        - name: TestScheduling
          args:
            maxPods: 17		# 调度插件自定义参数
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-scheduling
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-scheduling
  template:
    metadata:
      labels:
        app: test-scheduling
    spec:
      nodeName: lain1
      serviceAccount: test-scheduling-sa
      containers:
        - name: tests-cheduling
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["/app/test-scheduling"]
          args:
            - --config=/etc/kubernetes/config.yaml
            - --v=3
          volumeMounts:
            - name: config
              mountPath: /etc/kubernetes
            - name: app
              mountPath: /app
      volumes:
        - name: config
          configMap:
            name: test-scheduling-config
        - name: app
          hostPath:
            path: /home/txl/pod-scheduler
```

部署一个测试 pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testngx
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: testngx
  template:
    metadata:
      labels:
        app: testngx
    spec:
      schedulerName: test-scheduling	# 使用指定scheduler
      containers:
        - image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          name: testngx
          ports:
            - containerPort: 80
```

### Filter 过滤节点

不允许调度到存在 noScheduling=true 标签的节点上

```go
// lib/test-scheduling.go

var _ framework.FilterPlugin = &TestScheduler{}

func (s *TestScheduler) Filter(ctx context.Context, state *framework.CycleState, pod *coreV1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
	for k, v := range nodeInfo.Node().Labels {
		if k == "noScheduling" && v == "true" {
			return framework.NewStatus(framework.Unschedulable, "该节点不可调度")
		}
	}
	return framework.NewStatus(framework.Success)
}
```

加入 filter 调度配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-scheduling-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1beta2
    kind: KubeSchedulerConfiguration
    leaderElection:
      leaderElect: false
    profiles:
      - schedulerName: test-scheduling
        plugins:
          preFilter:
            enabled:
            - name: "TestScheduling"
          filter:
            enabled:
            - name: "TestScheduling"
        pluginConfig:
        - name: TestScheduling
          args:
            maxPods: 17
```

### PreScore 预打分

该阶段能得到 Filter 阶段结束后的 Node 列表，在这里我们可以做些预处理并保存到 CycleState（主要负责调度流程中”一些数据”的保存， framework 中所有的插件均可进行数据增加和修改， 线程安全）给后续插件进行流转

```go
// lib/test-scheduling.go

var _ framework.PreScorePlugin = &TestScheduler{}

type NodeMemory struct {
	data map[string]float64 // 内存空闲
}

func (n *NodeMemory) Clone() framework.StateData {	// 实现StateData接口
	return &NodeMemory{data: n.data}
}

func (s *TestScheduler) PreScore(ctx context.Context, state *framework.CycleState, pod *coreV1.Pod, nodes []*coreV1.Node) *framework.Status {
	memory := &NodeMemory{data: map[string]float64{}}
	memory.data["node1"] = 0.3
	memory.data["node2"] = 0.4
	state.Write("nodeMemory", memory)
	klog.Info("预打分阶段：保存数据成功")
	return framework.NewStatus(framework.Success)
}
```

加入 filter 调度配置

```yaml
preScore:
  enabled:
    - name: "TestScheduling"
```

然后就可以在后续扩展点通过 `getNodeMemory, err := state.Read("nodeMemory")` 获取值

### Score 打分
最终的分数需要在 [MinNodeScore，MaxNodeScore]（[0-100]）之间，为了防止分数超出，需要做归一化（Min-Max Normalization ）处理，这是最常见的 min-max 归一化公式
$$
X_{nom} = \frac{X - X_{min}}{X_{max} - X_{min}}
$$
也称为离差标准化，是对原始数据的线性变换，使结果值映射到  [min- max]之间

```go
// lib/test-scheduling.go

var _ framework.ScorePlugin = &TestScheduler{}

func (s *TestScheduler) Score(ctx context.Context, state *framework.CycleState, p *coreV1.Pod, nodeName string) (int64, *framework.Status) {
	if nodeName == "lain2" {
		return 30, framework.NewStatus(framework.Success)
	}
	return 20, framework.NewStatus(framework.Success)
}

func (s *TestScheduler) ScoreExtensions() framework.ScoreExtensions {
	return s
}

func (s *TestScheduler) NormalizeScore(ctx context.Context, state *framework.CycleState, p *coreV1.Pod, scores framework.NodeScoreList) *framework.Status {
	var min, max int64 = 0, 0
	// 求出最小分数和最大分数区间
	for _, score := range scores {
		if score.Score < min {
			min = score.Score
		}
		if score.Score > max {
			max = score.Score
		}
	}
	if max == min {
		min = min - 1
	}

	for i, score := range scores {	// 每个节点的分数归一化处理
		scores[i].Score = (score.Score - min) * framework.MaxNodeScore / (max - min)
		klog.Infof("节点: %v, Score: %v   Pod:  %v", scores[i].Name, scores[i].Score, p.GetName())
	}
	return framework.NewStatus(framework.Success, "")
}
```

加入 filter 调度配置

```yaml
score:
  enabled:
    - name: "TestScheduling"
```

### Permit 阶段

判断前置 POD 如果不存在，则等待10s，超时后后重新入列

```go
// lib/test-scheduling.go

var _ framework.PermitPlugin = &TestScheduler{}

func (s *TestScheduler) Permit(ctx context.Context, state *framework.CycleState, p *coreV1.Pod, nodeName string) (*framework.Status, time.Duration) {
	_, err := s.fact.Core().V1().Pods().Lister().Pods("default").Get("prepose_pod")
	if err != nil {
		klog.Info("Permit：等待10秒")
		return framework.NewStatus(framework.Wait), time.Second * 10
	} else {
		klog.Info("Permit：通过")
		return framework.NewStatus(framework.Success), 0
	}
}
```

加入 filter 调度配置

```yaml
permit:
  enabled:
    - name: "TestScheduling"
```
