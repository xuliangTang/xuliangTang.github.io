---
title: "原生 CI/CD 框架 Tekton"
date: 2023-03-31
draft: false
Categories: [kubernetes]
---

[Tekton](https://tekton.dev/) 是一款功能非常强大而灵活的 CI/CD 开源的云原生框架。Tekton 的前身是 Knative 项目的 build-pipeline 项目，这个项目是为了给 build 模块增加 pipeline 的功能，但是随着不同的功能加入到 Knative build 模块中，build 模块越来越变得像一个通用的 CI/CD 系统，于是，索性将 build-pipeline 剥离出 Knative，就变成了现在的 Tekton，而 Tekton 也从此致力于提供全功能、标准化的云原生 CI/CD 解决方案

## 部署

### Pipeline

可以通过 GitHub 仓库  [tektoncd/pipeline](https://github.com/tektoncd/pipeline) 中的 release.yaml 文件进行安装

由于官方使用的镜像是 gcr 的镜像，所以正常情况下我们是获取不到的，需要替换镜像地址，参考：[示例 yaml](/tekton/tekton-deploy.yaml)

```yaml
# image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.27.3@sha256:364e0fde644f39a8e2c622d545bc792831270e2fb2076a97363b5cc38446138c
image: tektondev/pipeline-controller:v0.27.3

# image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.27.3@sha256:54cb380aaa4f611eef996d982fd6052a793b54cd6092b12a46bb8afd16124aca
image:  tektondev/pipeline-webhook:v0.27.3
```

### Trigger

可以通过 GitHub 仓库 [tektoncd/triggers](https://github.com/tektoncd/triggers/) 中的 release.yaml 和 interceptors.yaml 安装

同样需要替换镜像地址，参考：[示例 release.yaml](/tekton/trigger.yaml) 和 [示例 interceptors.yaml](/tekton/trigger-interceptor.yaml)

### 安装客户端工具 tkn

从 GitHub 仓库 [tektoncd/cli](https://github.com/tektoncd/cli) 下载



## 资源对象

- **Task**：表示执行命令的一系列步骤，task 里可以定义一系列的 steps，例如编译代码、构建镜像、推送镜像等，每个 step 实际由一个 Pod 执行
- **TaskRun**：task 只是定义了一个模版，taskRun 才真正代表了一次实际的运行，当然你也可以自己手动创建一个 taskRun，taskRun 创建出来之后，就会自动触发 task 描述的构建任务
- **Pipeline**：一组任务，表示一个或多个 task、PipelineResource 以及各种定义参数的集合
- **PipelineRun**：类似 task 和 taskRun 的关系，pipelineRun 也表示某一次实际运行的 pipeline，下发一个 pipelineRun CRD 实例到 Kubernetes 后，同样也会触发一次 pipeline 的构建
- **PipelineResource**：表示 pipeline 输入资源，比如 github 上的源码，或者 pipeline 输出资源，例如一个容器镜像或者构建生成的 jar 包等

Demo：

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test
spec:
  steps:
    - name: list
      image: alpine:3.12
      command:
        - ls
---
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: test-run
spec:
  taskRef:
    name: test
---
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: test-pipeline
spec:
  tasks:
    - name: testtask
      taskRef:
        name: test
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: test-pipelinerun
  namespace: default
spec:
  pipelineRef:
    name: test-pipeline
```



## 示例：打包镜像

从 GitHub 私有仓库拉取代码后，打包 docker 镜像并推送到阿里云私有镜像仓库

**定义 git 和 image 资源**

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-github
  namespace: default
spec:
  type: git
  params:
    - name:  url
      value: git@github.com:xxx/xxx.git
    - name: revision
      value: master
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-image
spec:
  type: image
  params:
    - name: url
      value: registry.cn-hangzhou.aliyuncs.com/xxx/xxx:v1
```

**创建用于拉取 github 的密钥**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-auth
  namespace: default
  annotations:
    tekton.dev/git-0: github.com
type: kubernetes.io/ssh-auth
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
```

**创建登录阿里云镜像仓库的密钥**

```bash
kubectl create secret docker-registry aliyun-registry \
--docker-server=registry.cn-hangzhou.aliyuncs.com \
--docker-username= \
--docker-password=
```

**创建 ServiceAccount**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitsa
  namespace: default
secrets:
  - name: github-auth
  - name: aliyun-registry
```

**定义 Task**

使用谷歌开源的一款用来构建容器镜像的工具：Kaniko

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: git-task
  namespace: default
spec:
  resources:
    inputs:
      - name: test
        type: git
    outputs:
      - name: myimage
        type: image
  steps:
    - name: build-push
      image: aiotceo/kaniko-executor:v1.6.0
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=/workspace/flow/Dockerfile
        - --context=/workspace/flow
        - --destination=$(resources.outputs.myimage.url)
```

**定义 Pipeline 和 PipelineRun**

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: git-pipeline
spec:
  resources:
   - name: source-repo
     type: git
   - name: source-image
     type: image
  tasks:
    - name: task1
      taskRef:
        name: git-task
      resources:
        inputs:
          - name: test	# 关联Task中ipput定义的git名称
            resource: source-repo
        outputs:
          - name: myimage	# 关联Task中output定义的image名称
            resource: source-image
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: gitrun
spec:
  serviceAccountName: gitsa
  resources:
    - name: source-repo
      resourceRef:
        name: test-github	# 关联github PipelineResource名称
    - name: source-image
      resourceRef:
        name: test-image	# 关联阿里云镜像仓库PipelineResource名称
  pipelineRef:
    name: git-pipeline
```



## 使用 Trigger 自动触发流水线

Trigger 可以实现通过外部事件来触发对应的任务，比如有代码提交时触发自动构建以及部署任务。它可以从各种来源的事件中检测并提取需要信息，然后根据这些信息来创建 TaskRun 和 PipelineRun，还可以将提取出来的信息传递给它们以满足不同的运行要求

gitlab、github 的 webhook 就是一种最常用的外部事件，通过 Trigger 组件就监听这部分事件从而实现在提交代码后自动运行某些任务

### 核心模块

- **EventListener**：事件监听器，是外部事件的入口 ，通常需要通过 HTTP 方式暴露，以便于外部事件推送，比如配置 Gitlab 的Webhook
- **Trigger**：指定当 EventListener 检测到事件发生时会发生什么，它会定义 TriggerBinding、TriggerTemplate 以及可选的 Interceptor
- **TriggerTemplate**：用于模板化资源，根据传入的参数实例化 Tekton 对象资源，比如 TaskRun、PipelineRun 等
- **TriggerBinding**：用于捕获事件中的字段并将其存储为参数，然后会将参数传递给 TriggerTemplate
- **ClusterTriggerBinding**：和 TriggerBinding 相似，用于提取事件字段，不过它是集群级别的对象
- **Interceptors**：拦截器，在 TriggerBinding 之前运行，用于负载过滤、验证、转换等处理，只有通过拦截器的数据才会传递给TriggerBinding

### 示例

**创建 RBAC**

参考：[triggers/eventlisteners.md](https://github.com/tektoncd/triggers/blob/main/docs/eventlisteners.md)

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: triggers-role
rules:
  - apiGroups: ["triggers.tekton.dev"]
    resources: ["eventlisteners","triggers", "triggerbindings", "triggertemplates","clustertriggerbindings","clusterinterceptors"]
    verbs: ["get","list","watch","create"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets", "serviceaccounts"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["tekton.dev"]
    resources: ["pipelineruns", "pipelineresources", "taskruns","task"]
    verbs: ["create","get","list","watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gitsa-triggers
subjects:
  - kind: ServiceAccount
    namespace: default
    name: gitsa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: triggers-role
```

**创建 binding 对象**

参考：[triggers/triggerbindings.md](https://github.com/tektoncd/triggers/blob/main/docs/triggerbindings.md#structure-of-a-triggerbinding)

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: git-binding
spec:
  params:
    - name: gitrevision
      value: master
    - name: gitrepositoryurl
      value: $(body.repository.url)
```

**创建 TriggerTemplate 对象**

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: git-template
spec:
  # 和binding里的name对应，下面需要引用
  params:
    - name: gitrevision
    - name: gitrepositoryurl
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: TaskRun  # 定义 TaskRun 模板
      metadata:
        generateName: trigger-run-  # 前缀
      spec:
        serviceAccountName: gitsa
        taskSpec:
          resources:
            inputs:
              - name: source
                type: git
          steps:
            - name: ls
              image: alpine:3.12
              script: |
                #! /bin/sh
                ls /workspace/source
        resources:
          inputs:
          - name: source
            resourceSpec:
              type: git
              params:
                - name: revision
                  value: $(tt.params.gitrevision)
                - name: url
                  value: $(tt.params.gitrepositoryurl)
```

**创建 EventListener 对象**

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: git-listener
spec:
  serviceAccountName: gitsa
  triggers:
  - name: git-trigger1
    bindings:
    - ref: git-binding
    template:
      ref: git-template
```

**调用接口触发事件**

正常情况下一般是通过 ingerss 暴露 EventListener 服务，配置到 gitlab、github 仓库中作为 webhook，不过手动调用也可以触发

```bash
curl http://xxx.com -d '{"repository":{"url":"git@github.com:xxx/xxx.git"}}'

tkn taskrun list
NAME                       STARTED          DURATION     STATUS
trigger-run-ck8f9          2 minutes ago    32 seconds   Succeeded
```



## API 调用

**初始化 tekton 客户端**

```go
import (
	tektonVersiond "github.com/tektoncd/pipeline/pkg/client/clientset/versioned"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"pixelk8/src/properties"
)

func K8sRestConfig() *rest.Config {
	config, err := clientcmd.BuildConfigFromFlags("", "./kubeconfig")
	if err != nil {
		log.Fatal(err)
	}
	return config
}

func InitTektonClient() *tektonVersiond.Clientset {
	client, err := tektonVersiond.NewForConfig(this.K8sRestConfig())
	if err != nil {
		log.Fatal(err)
	}
	return client
}
```

**informer 监听**

```go
import (
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"log"
	"pixelk8/pkg/tekton"
	"pixelk8/src/properties"
)

// 动态informers处理
var taskResource = schema.GroupVersionResource{Group: "tekton.dev", Resource: "tasks", Version: "v1beta1"}
var pipelineResource = schema.GroupVersionResource{Group: "tekton.dev", Resource: "pipelines", Version: "v1beta1"}
var pipelineRunResource = schema.GroupVersionResource{Group: "tekton.dev", Resource: "pipelineruns", Version: "v1beta1"}

func InitGenericInformer() dynamicinformer.DynamicSharedInformerFactory {
	client, err := dynamic.NewForConfig(this.K8sRestConfig())
	if err != nil {
		log.Fatal(err)
	}
	di := dynamicinformer.NewDynamicSharedInformerFactory(client, 0)

	di.ForResource(taskResource).Informer().AddEventHandler(TektonTaskHandler)
	di.ForResource(pipelineResource).Informer().AddEventHandler(TektonPipelineHandler)
	di.ForResource(pipelineRunResource).Informer().AddEventHandler(TektonPipelineRunHandler)

	di.Start(wait.NeverStop)
	return di
}
```
