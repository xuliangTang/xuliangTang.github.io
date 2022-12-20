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