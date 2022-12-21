---
title: "K8s list-watch 机制和 Informer 模块"
date: 2022-12-18T18:29:49+08:00
draft: false
Categories: [k8s-go]
---

在 Kubernetes 中，有5个主要的组件，分别是 master 节点上的 kube-api-server、kube-controller-manager 和 kube-scheduler，node 节点上的 kubelet 和kube-proxy 。这其中 kube-apiserver 是对外和对内提供资源的声明式 API 的组件，其它4个组件都需要和它交互。为了保证消息的实时性，有两种方式：

- 客户端组件 (kubelet, scheduler, controller-manager 等) 轮询 apiserver
- apiserver 通知客户端

为了降低 kube-apiserver 的压力，有一个非常关键的机制就是 list-watch。list-watch 本质上也是 client 端监听 k8s 资源变化并作出相应处理的生产者消费者框架

list-watach 机制需要满足以下需求：

1. 实时性 (即数据变化时，相关组件越快感知越好)
2. 保证消息的顺序性 (即消息要按发生先后顺序送达目的组件。很难想象在Pod创建消息前收到该Pod删除消息时组件应该怎么处理)
3. 保证消息不丢失或者有可靠的重新获取机制 (比如 kubelet 和 kube-apiserver 间网络闪断，需要保证网络恢复后kubelet可以收到网络闪断期间产生的消息)



## list-watch 机制

list-watch 由两部分组成，分别是 list 和 watch。list 非常好理解，就是调用资源的 list API 罗列资源 ，基于 HTTP 短链接实现，watch 则是调用资源的 watch API 监听资源变更事件，基于 HTTP 长链接实现

etcd 存储集群的数据信息，apiserver 作为统一入口，任何对数据的操作都必须经过 apiserver。客户端通过 list-watch 监听 apiserver 中资源的 create, update 和 delete 事件，并针对事件类型调用相应的事件处理函数

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202212181854816.jpeg)



## informer 机制

k8s 的 informer 模块封装 list-watch API，用户只需要指定资源，编写事件处理函数，AddFunc, UpdateFunc 和 DeleteFunc 等。如下图所示，informer 首先通过 list API 罗列资源，然后调用 watch API 监听资源的变更事件，并将结果放入到一个 FIFO 队列，队列的另一头有协程从中取出事件，并调用对应的注册函数处理事件。Informer 还维护了一个只读的 Map Store 缓存，主要为了提升查询的效率，降低 apiserver 的负载

![informer实现原理](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202212181902721.png)



## 在 client-go 中的应用

client-go 使用 k8s.io/client-go/tools/cache 包里的 informer 对象进行 list-watch 机制的封装

最粗暴的解释：

1. 初始化时，调 List API 获得全量 list，缓存起来(本地缓存)，这样就不需要每次请求都去请求 ApiServer
2. 调用 Watch API 去 watch 资源，发生变更后会通过一定机制维护缓存

```go
type DepHandler struct{}
func (this *DepHandler) OnAdd(obj interface{}) {}
func (this *DepHandler) OnUpdate(oldObj, newObj interface{}) {
	if dep, ok := newObj.(*v1.Deployment); ok {
		fmt.Println(dep.Name)
	}
}
func (this *DepHandler) OnDelete(obj interface{}) {}

func main() {
	_, c := cache.NewInformer(
		// 监听 default 命名空间中 deployment 的变化
		cache.NewListWatchFromClient(K8sClient.AppsV1().RESTClient(),
			"deployments", "default", fields.Everything()),
		&v1.Deployment{},
		0,				// 重新同步时间
		&DepHandler{},	// 实现类
	)
	c.Run(wait.NeverStop)
	select {}
}
```

### SharedInformerFactory

sharedInformerFactory 用来构造各种 Informer 的工厂对象，它可以共享多个 informer 资源

```go
informerFactory := informers.NewSharedInformerFactory(K8sClient, 0)

// 构建一个 deployment informer
depInformer := informerFactory.Apps().V1().Deployments()
depInformer.Informer().AddEventHandler(&DepHandler{})

informerFactory.Start(wait.NeverStop)
select {}
```



## 示例

### 监听 deployment

```go
// 全局对象，存储所有deployments
var DepMapImpl *DeploymentMap

func init() {
	DepMapImpl = &DeploymentMap{Data: new(sync.Map)}
}

type DeploymentMap struct {
  Data *sync.Map	// key:namespace value:[]*v1.Deployments
}

// 添加
func (this *DeploymentMap) Add(deployment *v1.Deployment) {
	if depList, ok := this.Data.Load(deployment.Namespace); ok {
		depList = append(depList.([]*v1.Deployment), deployment)
		this.Data.Store(deployment.Namespace, depList)
	} else {
		this.Data.Store(deployment.Namespace, []*v1.Deployment{deployment})
	}
}

// 获取列表
func (this *DeploymentMap) ListByNs(namespace string) ([]*v1.Deployment, error) {
	if depList, ok := this.Data.Load(namespace); ok {
		return depList.([]*v1.Deployment), nil
	}

	return nil, fmt.Errorf("record not found")
}

// 更新
func (this *DeploymentMap) Update(deployment *v1.Deployment) error {
	if depList, ok := this.Data.Load(deployment.Namespace); ok {
		depList := depList.([]*v1.Deployment)
		for i, dep := range depList {
			if dep.Name == deployment.Name {
				depList[i] = deployment
				break
			}
		}
		return nil
	}

	return fmt.Errorf("deployment [%s] not found", deployment.Name)
}

// 删除
func (this *DeploymentMap) Delete(deployment *v1.Deployment) {
	if depList, ok := this.Data.Load(deployment.Namespace); ok {
		depList := depList.([]*v1.Deployment)
		for i, dep := range depList {
			if dep.Name == deployment.Name {
				newDepList := append(depList[:i], depList[i+1:]...)
				this.Data.Store(deployment.Namespace, newDepList)
				break
			}
		}
	}
}

// informer实现
type DepHandler struct{}
func (this *DepHandler) OnAdd(obj interface{}) {
	DepMapImpl.Add(obj.(*v1.Deployment))
}
func (this *DepHandler) OnUpdate(oldObj, newObj interface{}) {
	err := DepMapImpl.Update(newObj.(*v1.Deployment))
	if err != nil {
		log.Println(err)
	}
}
func (this *DepHandler) OnDelete(obj interface{}) {
	DepMapImpl.Delete(obj.(*v1.Deployment))
}

// 执行监听
func InitDeployments() {
	informerFactory := informers.NewSharedInformerFactory(K8sClient, 0)

	depInformer := informerFactory.Apps().V1().Deployments()
	depInformer.Informer().AddEventHandler(&DepHandler{})

	informerFactory.Start(wait.NeverStop)
}
```

### 获取 deployment 的关联 pod

之前做过利用 Deployment 的 MatchLabels 去匹配 pod 的 labels 的方式。这次我们利用 ReplicaSet 的标签去匹配 Pod，这种方式可以区分当多个 Deployment 的 Pod 设置为相同标签的场景

当创建完 Deployment 后，k8s 会创建对应的 ReplicaSet，它会根据 template 里的内容进行 hash，然后自动设置一个标签 `pod-template-hash`，且与它管理的所有 Pod 标签相对应

```
Labels: app=xnginx
        pod-template-hash=767447889d
```

我们只需要通过 Deployment 获取它的 ReplicaSet，再拿 labels 去匹配 Pod

- **第一步**：监听 Deployment、ReplicaSet 和 Pod，分别实现对应的 informer 方法，将数据缓存到本地

- **第二步**：通过 Deployment 获取对应的 ReplicaSet，拿到 labels

关键代码：

```go
// 从本地缓存中取出所有的rs
rsList, err := RSMapImpl.ListByNs(namespace)

// 获取 labels
labels, err := GetListWatchRsLabelByDeployment(deployment, rsList)
```

```go
// list-watch方式 根据deployment获取当前ReplicaSet的标签
func GetListWatchRsLabelByDeployment(deployment *v1.Deployment, rsList []*v1.ReplicaSet) (map[string]string, error) {
	for _, rs := range rsList {
		if IsCurrentRsByDeployment(rs, deployment) {
			selector, err := metaV1.LabelSelectorAsMap(rs.Spec.Selector)
			if err != nil {
				return nil, err
			}
			return selector, nil
		}
	}

	return nil, nil
}

// 判断rs是否对应当前deployment
func IsCurrentRsByDeployment(set *v1.ReplicaSet, deployment *v1.Deployment) bool {
	if set.ObjectMeta.Annotations["deployment.kubernetes.io/revision"] != deployment.ObjectMeta.Annotations["deployment.kubernetes.io/revision"] {
		return false
	}

	for _, rf := range set.OwnerReferences {
		if rf.Kind == "Deployment" && rf.Name == deployment.Name {
			return true
		}
	}

	return false
}
```

- **第三步**：通过 labels 去匹配 pods

关键代码：

```go
// 根据标签获取Pod列表
func (this *PodMap) ListByLabels(ns string, labels map[string]string) ([]*v1.Pod, error) {
	ret := make([]*v1.Pod, 0)
	if podList, ok := this.Data.Load(ns); ok {
		podList := podList.([]*v1.Pod)
		for _, p := range podList {
			// 判断标签完全匹配
			if reflect.DeepEqual(p.Labels, labels) {
				ret = append(ret, p)
			}
		}

		return ret, nil
	}

	return nil, fmt.Errorf("pods not found")
}
```

### 获取 Pod 状态和 Event

Pod 状态信息包含：

- **阶段**：Pod 的 status 字段是一个 PodStatus 对象，其中包含一个 phase 字段

| 取值                | 描述                                                         |
| :------------------ | :----------------------------------------------------------- |
| `Pending`（悬决）   | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器尚未创建亦未运行。此阶段包括等待 Pod 被调度的时间和通过网络下载镜像的时间。 |
| `Running`（运行中） | Pod 已经绑定到了某个节点，Pod 中所有的容器都已被创建。至少有一个容器仍在运行，或者正处于启动或重启状态。 |
| `Succeeded`（成功） | Pod 中的所有容器都已成功终止，并且不会再重启。               |
| `Failed`（失败）    | Pod 中的所有容器都已终止，并且至少有一个容器是因为失败终止。也就是说，容器以非 0 状态退出或者被系统终止。 |
| `Unknown`（未知）   | 因为某些原因无法取得 Pod 的状态。这种情况通常是因为与 Pod 所在主机通信失败。 |

- **状况**：PodStatus 对象包含一个 PodConditions 数组

| 字段名称             | 描述                                                         |
| :------------------- | :----------------------------------------------------------- |
| `type`               | Pod 状况的名称                                               |
| `status`             | 表明该状况是否适用，可能的取值有 "`True`"、"`False`" 或 "`Unknown`" |
| `lastProbeTime`      | 上次探测 Pod 状况时的时间戳                                  |
| `lastTransitionTime` | Pod 上次从一种状态转换到另一种状态时的时间戳                 |
| `reason`             | 机器可读的、驼峰编码（UpperCamelCase）的文字，表述上次状况变化的原因 |
| `message`            | 人类可读的消息，给出上次状态转换的详细信息                   |

>`PodScheduled`：Pod 已经被调度到某节点
>
>`PodHasNetwork`：Pod 沙箱被成功创建并且配置了网络（Alpha 特性，必须被显式启用）
>
>`ContainersReady`：Pod 中所有容器都已就绪
>
>`Initialized`：所有的 Init 容器都已成功完成
>
>`Ready`：Pod 可以为请求提供服务，并且应该被添加到对应服务的负载均衡池中

- **事件对象**：为用户提供了洞察集群内发生的事情的能力。为了避免主节点磁盘空间被填满，将强制执行保留策略：事件在最后一次发生的一小时后将会被删除

关键代码：

```go
// EventMapImpl 全局对象，存储所有Event
var EventMapImpl *EventMap

func init() {
	EventMapImpl = &EventMap{Data: new(sync.Map)}
}

type EventMap struct {
	Data *sync.Map // key:namespace_kind_name value: *v1.Event
}

func (this *EventMap) GetKey(event *v1.Event) string {
	key := fmt.Sprintf("%s_%s_%s", event.Namespace, event.InvolvedObject.Kind, event.InvolvedObject.Name)
	return key
}

// Add 添加
func (this *EventMap) Add(event *v1.Event) {
	EventMapImpl.Data.Store(this.GetKey(event), event)
}

// Delete 删除
func (this *EventMap) Delete(event *v1.Event) {
	EventMapImpl.Data.Delete(this.GetKey(event))
}

// 获取最新一条event message
func (this *EventMap) GetMessage(ns string, kind string, name string) string {
	key := fmt.Sprintf("%s_%s_%s", ns, kind, name)
	if v, ok := this.Data.Load(key); ok {
		return v.(*v1.Event).Message
	}

	return ""
}

// EventHandler informer实现
type EventHandler struct{}

func (this *EventHandler) OnAdd(obj interface{}) {
	EventMapImpl.Add(obj.(*v1.Event))
}
func (this *EventHandler) OnUpdate(oldObj, newObj interface{}) {
	EventMapImpl.Add(newObj.(*v1.Event))
}
func (this *EventHandler) OnDelete(obj interface{}) {
	EventMapImpl.Delete(obj.(*v1.Event))
}
```

```go
// 评估Pod是否就绪
func GetPodIsReady(pod *coreV1.Pod) bool {
	for _, condition := range pod.Status.Conditions {
		if condition.Type == "ContainersReady" && condition.Status != "True" {
			return false
		}
	}
	
	for _, rg := range pod.Spec.ReadinessGates {
		for _, condition := range pod.Status.Conditions {
			if condition.Type == rg.ConditionType && condition.Status != "True" {
				return false
			}
		}
	}

	return true
}

// 获取pods DTO 把原生的 pod 对象转换为自己的实体对象
func GetPodsByLabels(ns string, labels []map[string]string) (pods []*model.PodModel) {
	podList, err := PodMapImpl.ListByLabels(ns, labels)
	lib.CheckError(err)

	pods = make([]*model.PodModel, len(podList))

	for i, pod := range podList {
		pods[i] = &model.PodModel{
			Name:      pod.Name,
			NodeName:  pod.Spec.NodeName,
			Images:    GetPodImages(pod.Spec.Containers),
			Phase:     string(pod.Status.Phase),
			IsReady:   GetPodIsReady(pod),
			Message:   EventMapImpl.GetMessage(pod.Namespace, "Pod", pod.Name),
			CreatedAt: pod.CreationTimestamp.Format("2006-01-02 15:04:05"),
		}
	}

	return
}
```

