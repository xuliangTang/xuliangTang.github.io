---
title: "使用 kubebuilder 开发 Operator"
date: 2023-02-12
draft: false
Categories: [kubernetes]
---

## Operator 模式

Kubernetes 是一个高度可扩展的"系统"，比如常见的自定义资源，控制器，准入控制及调度器进行扩展开发等。Operator 是一种包装、运行和管理 K8S 应用的一种方式。它涵盖了 CRD（CustomResourceDeftination） + AdmissionWebhook + Controller ，并以 Deployment 的形式部署到 K8S 中

- **CRD** 用来定义声明式API（yaml），程序会通过该定义一直让最小调度单元（POD）趋向该状态
- **AdmissionWebhook** 用来拦截请求做 mutate（修改）提交的声明（yaml）和 validate（校验）声明式的字段
- **Controller** 主要的控制器，监视资源的 创建 / 更新 / 删除 事件，并触发 `Reconcile` 函数作为响应。整个调整过程被称作 Reconcile Loop（协调一致的循环），其实就是让  POD 趋向 CRD 定义所需的状态

### Operator 的工作方式

Operator 通过扩展 Kubernetes 控制平面和 API 进行工作。Operator 将一个 endpoint（称为自定义资源 CR）添加到 Kubernetes API 中，该 endpoint 还包含一个监控和维护新类型资源的控制平面组件。整个操作原理如下图所示：

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302102322563.png)

当 Operator 接收任何信息时，它将采取行动将 Kubernetes 集群或外部系统调整到所需的状态，作为其在自定义 controller 中的和解循环（reconciliation loop）的一部分

## Resource、ResourceType 和 Controller

Kubernetes发展到今天，其本质已经显现：

- Kubernetes 就是一个 “数据库”（数据实际持久存储在etcd中）
- 其 API 就是 “sql语句”
- API 设计采用基于 resource 的 Restful 风格，resource type 是 API 的端点（endpoint）
- 每一类 resource（即 Resource Type）是一张 “表”，Resource Type 的 spec 对应 “表结构” 信息（schema）
- 每张 “表” 里的一行记录就是一个 resource，即该表对应的 Resource Type 的一个实例（instance）
- Kubernetes 这个 “数据库” 内置了很多 “表”，比如 Pod、Deployment、DaemonSet、ReplicaSet 等

下面是一个Kubernetes API与resource关系的示意图：

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302102332099.png)

我们看到 resource type 有两类，一类的 namespace 相关的（namespace-scoped），另外一类则是 namespace 无关，即 cluster 范围（cluster-scoped）的

我们知道 Kubernetes 并非真的只是一个 “数据库”，它是服务编排和容器调度的平台标准，它的基本调度单元是Pod（也是一个resource type），即一组容器的集合。那么 Pod 又是如何被创建、更新和删除的呢？这就离不开控制器（controller）了。**每一类 resource type 都有自己对应的控制器（controller）**。以 pod 这个 resource type 为例，它的 controller 为 ReplicasSet 的实例。控制器的运行逻辑如下图所示：

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302102334551.png)

控制器一旦启动，将尝试获得 resource 的当前状态（current state），并与存储在 k8s 中的 resource 的期望状态（desired state，即 spec）做比对，如果不一致，controller 就会调用相应 API 进行调整，尽力使得 current state 与期望状态达成一致。这个达成一致的过程被称为**协调（reconciliation）**

根据前面我们对 resource type 理解，定义 CRD 相当于建立新 “表”（resource type），一旦 CRD 建立，k8s 会为我们自动生成对应 CRD 的 API endpoint，我们就可以通过 yaml 或 API 来操作这个 “表”。我们可以向 “表” 中 “插入” 数据，即基于 CRD 创建 Custom Resource（CR），这就好比我们创建 Deployment 实例，向 Deployment “表” 中插入数据一样

和原生内置的 resource type 一样，光有存储对象状态的 CR 还不够，原生 resource type 有对应 controller 负责协调（reconciliation）实例的创建、伸缩与删除，CR 也需要这样的 “协调者”，即我们也需要定义一个 controller 来负责监听 CR 状态并管理 CR 创建、伸缩、删除以及保持期望状态（spec）与当前状态（current state）的一致。这个 controller 不再是面向原生 Resource type 的实例，而是**面向 CRD 的实例 CR 的 controller**

有了自定义的操作对象类型（CRD），有了面向操作对象类型实例的 controller，我们将其打包为一个概念：“Operator模式”，operator 模式中的 controller 也被称为 operator，它是在集群中对 CR 进行维护操作的主体

## 使用 kubebuilder 开发 operator

Kubebuilder 是一个基于 CRD 来构建 Kubernetes API 的框架，可以使用 CRD 来构建 API、Controller 和 Admission Webhook。参考文档：[The Kubebuilder Book](https://book.kubebuilder.io/quick-start.html)

**1. 安装 kubebuilder**

Github 地址：[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)，将 kubebuilder 拷贝到你的 PATH 环境变量中的某个路径下即可

**2. 创建项目**

```bash
kubebuilder init --domain virtuallain.com
```

**3. 创建API**

这里我们就来建立自己的 CRD，即自定义的 resource type，也就是 API 的 endpoint

```bash
kubebuilder create api --group myapp --version v1 --kind Redis
```

此刻，整个工程的目录布局如下：

```bash
tree -F .
.
├── Dockerfile
├── Makefile
├── PROJECT
├── README.md
├── api/
│   └── v1/
│       ├── groupversion_info.go
│       ├── redis_types.go
│       └── zz_generated.deepcopy.go
├── bin/
│   ├── controller-gen*
├── config/
│   ├── crd/
│   │   ├── bases/
│   │   │   └── myapp.virtuallain.com_redis.yaml
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches/
│   │       ├── cainjection_in_redis.yaml
│   │       └── webhook_in_redis.yaml
│   ├── default/
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager/
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus/
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac/
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── redis_editor_role.yaml
│   │   ├── redis_viewer_role.yaml
│   │   ├── role.yaml
│   │   ├── role_binding.yaml
│   │   └── service_account.yaml
│   └── samples/
│       └── myapp_v1_redis.yaml
├── controllers/
│   ├── redis_controller.go
│   └── suite_test.go
├── go.mod
├── go.sum
├── hack/
│   └── boilerplate.go.txt
├── main.go
```

**4. 为 CRD spec 添加字段**

CRD 与 api 中的 redis_types.go 文件是同步的，我们只需修改这个文件即可。在 RedisSpec 结构中增加 Port 和 Replicas 字段

```go
// RedisSpec defines the desired state of Redis
type RedisSpec struct {
    //+kubebuilder:validation:Minimum:=81
    //+kubebuilder:validation:Maximum=30000
    Port int `json:"port,omitempty"`
    
    //+kubebuilder:validation:Minimum:=1
    //+kubebuilder:validation:Maximum=100
    Replicas int `json:"replicas,omitempty"`
}
```

一旦定义完 CRD，我们就可以将其安装/卸载到 k8s 中

```bash
make install
make uninstall
```

查看

```bash
kubectl get crd | grep redis
```

**5. 实现 controller 的 Reconcile(协调) 逻辑**

kubebuilder 为我们搭好了 controller 的代码架子，我们只需要在 controllers/redis_controller.go 中实现RedisReconciler 的 Reconcile 方法即可。下面是 Reconcile 的一个简易流程图

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302110019743.png)

```go
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    redis := &myappv1.Redis{}
    if err := r.Get(ctx, req.NamespacedName, redis); err != nil {
        return ctrl.Result{RequeueAfter: time.Second * 5}, err
    }
    
    fmt.Println("得到对象", redis.Spec)
}
```

**6. 本地运行控制器**

```bash
make run
```

通过 kubectl 创建该 CR

```yaml
apiVersion: myapp.virtuallain.com/v1
kind: Redis
metadata:
  name: myredis
  namespace: default
spec:
  port: 2000
```

观察 controller 的日志，可以看到当 CR 被创建后，controller 监听到相关事件

**7. 构建 controller image**

通过前文我们知道，这个 controller 其实就是运行在 k8s 中的一个 deployment 下的 pod，我们需要构建其 image 并通过 deployment 部署到 k8s 中。kubebuilder 创建的 operator 工程中包含了 Makefile，通过 `make docker-build` 即可构建 controller image，由于默认 GOPROXY 在国内无法访问，需要先修改 Dockerfile

```dockerfile
ENV GOPROXY=https://goproxy.io
```

然后执行

```bash
make docker build
```

构建成功后，执行 `make docker-push` 将 image 推送到镜像仓库中

**8. 部署 controller**

之前我们已经通过 `make install` 将 CRD 安装到k8s中了，接下来再把 controller 部署到 k8s 上，我们的 operator 就算部署完毕了

```bash
make deploy IMG=<some-registry>/<project-name>:tag
```

使用 `make undeploy` 可以完整卸载 operator 相关 resource

> 因为一些特殊原因，一些镜像如 gcr.io/kubebuilder/kube-rbac-proxy 镜像可能拉取不下来，可以找到替代的镜像如 bitnami/kube-rbac-proxy 手动更改 tag

### 实现业务逻辑

- 提交 CR 时自动创建 pod
- 动态伸缩副本
- 删除时清理关联 pod

```go
func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	redis := &myappv1.Redis{}
	if err := r.Get(ctx, req.NamespacedName, redis); err != nil {
		return ctrl.Result{}, nil
	}
    
    // 资源存在finalizers，删除时会增加DeletionTimestamp字段，只有当finalizers清空时才会真的删除
	if !redis.DeletionTimestamp.IsZero() {
		return ctrl.Result{}, helper.ClearRedis(r.Client, ctx, redis)	// 清理资源
	}

	edited := false
	podNames := helper.GetRedisPodNames(redis)
	diff := len(redis.Finalizers) - len(podNames) // 实际副本数量 - 期望副本数量
	if diff > 0 {                                 // 收缩副本
		if err := helper.RmIfSurplus(r.Client, ctx, redis, podNames); err != nil {
			return ctrl.Result{}, err
		}
		edited = true
	}

	// 创建副本
	for _, podName := range podNames {
		createPod, err := helper.CreateRedis(r.Client, ctx, redis, podName, r.Scheme)
		if err != nil {
			return ctrl.Result{}, err
		}

		// pod创建成功且redis资源的finalizers列表不包含新创建的pod名称
		if createPod != "" && !controllerutil.ContainsFinalizer(redis, createPod) {
			redis.Finalizers = append(redis.Finalizers, createPod) // 追加pod
			edited = true
		}
	}

	if edited {
		err := r.Client.Update(ctx, redis) // 更新redis的Finalizers列表
		return ctrl.Result{}, err
	}

	return ctrl.Result{}, nil
}

// GetRedisPodNames 根据副本数生成pod名称
func GetRedisPodNames(redis *v1.Redis) []string {
	podNames := make([]string, redis.Spec.Replicas)

	for i := 0; i < redis.Spec.Replicas; i++ {
		podNames[i] = fmt.Sprintf("%s-%d", redis.Name, i)
	}

	return podNames
}

// ExistPod 判断pod是否存在
func ExistPod(client client.Client, ctx context.Context, podName string, redis *v1.Redis) bool {
	err := client.Get(ctx, types.NamespacedName{Namespace: redis.Namespace, Name: podName}, &corev1.Pod{})

	return err == nil
}

// CreateRedis 创建redis
func CreateRedis(client client.Client, ctx context.Context, redis *v1.Redis, podName string, schema *runtime.Scheme) (string, error) {
	if ExistPod(client, ctx, podName, redis) {
		return "", nil
	}

	pod := &corev1.Pod{}
	pod.Namespace = redis.Namespace
	pod.Name = podName
	pod.Spec.Containers = []corev1.Container{
		{
			Name:            podName,
			Image:           "redis:5-alpine",
			ImagePullPolicy: corev1.PullIfNotPresent,
			Ports: []corev1.ContainerPort{
				{
					ContainerPort: int32(redis.Spec.Port),
				},
			},
		},
	}

	// 设置pod的ownReference为redis资源，删除redis资源时会删除关联的pod
	if err := controllerutil.SetOwnerReference(redis, pod, schema); err != nil {
		return "", err
	}

	return podName, client.Create(ctx, pod)
}

// RmIfSurplus 收缩副本
func RmIfSurplus(client client.Client, ctx context.Context, redis *v1.Redis, podNames []string) error {
	rmPods := redis.Finalizers[len(podNames):]
	for _, rmPod := range rmPods {
		err := client.Delete(ctx, &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      rmPod,
				Namespace: redis.Namespace,
			},
		})

		if err != nil {
			return err
		}
	}

	redis.Finalizers = podNames // 更新finalizers
	return nil
}

// ClearRedis 清理关联pod
func ClearRedis(client client.Client, ctx context.Context, redis *v1.Redis) error {
	/*
	// pod设置了ownReference后，就可以不用手动清理pod了
	podList := redis.Finalizers

	for _, podName := range podList {
		pod := &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      podName,
				Namespace: redis.Namespace,
			},
		}
		if err := client.Delete(ctx, pod); err != nil {
			return err
		}
	}*/

	redis.Finalizers = []string{}	// 将finalizers设置为空，否则无法删除该CR
	return client.Update(ctx, redis)
}
```

- 自动重建被手动删除的 pod

```go
// 监听删除pod
func (r *RedisReconciler) rmPodHandler(event event.DeleteEvent, limitingInterface workqueue.RateLimitingInterface) {
	for _, ref := range event.Object.GetOwnerReferences() {
		if ref.Kind == "Redis" && ref.APIVersion == "myapp.virtuallain.com/v1" {
			// 重建pod
			limitingInterface.Add(reconcile.Request{ // 重新入列 触发reconcile
				NamespacedName: types.NamespacedName{
					Namespace: event.Object.GetNamespace(),
					Name:      ref.Name,
				},
			})
		}
	}
}

// SetupWithManager sets up the controller with the Manager.
func (r *RedisReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&myappv1.Redis{}).
		Watches(&source.Kind{	// 增加watch
			Type: &corev1.Pod{},
		}, handler.Funcs{DeleteFunc: r.rmPodHandler}).
		Complete(r)
}
```

- 记录 Event 事件

```go
// controllers/redis_controller.go

type RedisReconciler struct {
        client.Client
        Scheme      *runtime.Scheme
        EventRecord record.EventRecorder
}

func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// ...
	r.EventRecord.Event(redis, corev1.EventTypeNormal, "Upgrade", "副本收缩")
}
```

```go
// main.go

if err = (&controllers.RedisReconciler{
	Client:        mgr.GetClient(),
	Scheme:        mgr.GetScheme(),
	EventRecorder: mgr.GetEventRecorderFor("Redis"),
}).SetupWithManager(mgr); err != nil {
	setupLog.Error(err, "unable to create controller", "controller", "Redis")
	os.Exit(1)
}
```

- 展示资源状态

```go
// api/v1/redis_types.go

type RedisStatus struct {
	Replicas int `json:"replicas"`
}

// 下面两个注释对应的是 kubectl get Redis 看到的状态  
//+kubebuilder:printcolumn:JSONPath=".status.replicas",name=replicas,type=integer
//+kubebuilder:printcolumn:JSONPath=".metadata.creationTimestamp",name=AGE,type=date
type Redis struct {
	// ...
}
```

```go
// controllers/redis_controller.go

func (r *RedisReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// ...
	redis.Status.Replicas = len(redis.Finalizers)
	if err := r.Status().Update(ctx, redis) {	// 更新status
		return ctrl.Result{}, err
	}
}
```

### 集成测试

（不推荐使用）参考文档：[Writing tests](https://book.kubebuilder.io/cronjob-tutorial/writing-tests.html)、[配置 EnvTest](https://book.kubebuilder.io/reference/envtest.html)、[Ginkgo](https://ke-chain.github.io/ginkgodoc/#入门-第一个测试)
