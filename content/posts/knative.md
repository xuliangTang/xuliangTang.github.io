---
title: "Knative 入门"
date: 2023-11-10
draft: false
Categories: [Istio]
---

Knative 是一款基于 Kubernetes 的 Serverless 框架。其目标是制定云原生、跨平台的 Serverless 编排标准。Knative 通过整合容器构建（或者函数）、工作负载管理（动态扩缩）以及事件模型这三者来实现的这一 Serverless 标准

Knative 包含以下两个组件：

- Eventing：它是一组 API，使用标准的 HTTP POST 请求产生生产者/消费者模式以及响应 HTTP 请求。这些事件符合 CloudEvents 规范，允许以任何编程语言创建、解析、发送和接收事件
- Serving：管理 Serverless 工作负载，提供了应用部署、多版本管理、基于请求的自动弹性、灰度发布等能力，而且在没有服务需要处理的时候可以缩容到零个实例。定义了一组特定的对象，包括 Revision(修订版本)、Configuration(配置)、Route(路由)和Service(服务)。Knative 通过 Kubernetes CRD 的方式实现这些 Kubernetes 对象

## 安装

文档地址：[About YAML-based installation](https://knative.dev/docs/install/yaml-install/)

需要把 gcr.io 无法拉取的镜像地址替换为 docker.io/gcrioknative

**安装 Serving 组件**

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.12.0/serving-core.yaml
```

**安装 Istio 网络层控制器**

```bash
kubectl apply -f https://github.com/knative/net-istio/releases/download/knative-v1.12.0/net-istio.yaml
```

**安装 Eventing 组件**

```bash
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/eventing-crds.yaml
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.12.0/eventing-core.yaml
```

## 实验

### 发布一个服务

文档地址：[Deploying a Knative Service](https://knative.dev/docs/getting-started/first-service/)

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: ksvc1
  namespace: myknative
spec:
  template:
    spec:
      containers:
        - image: nginx:1.18-alpine
          ports:
            - containerPort: 80
```

查看

```bash
$ kubectl get deploy -n myknative
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
ksvc1-00001-deployment   1/1     1            1           39m
$ kubectl get ksvc -n myknative
NAME    URL                                        LATESTCREATED   LATESTREADY   READY     REASON
ksvc1   http://ksvc1.myknative.svc.cluster.local   ksvc1-00001     ksvc1-00001   Unknown   IngressNotConfigured
```

### 配置副本自动伸缩

文档地址：[Supported autoscaler types](https://knative.dev/docs/serving/autoscaling/autoscaler-types/#global-versus-per-revision-settings)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-autoscaler
  namespace: knative-serving
  labels:
    app: test
data:
  enable-scale-to-zero: "false"       # 是否会收缩为0副本
  scale-to-zero-grace-period: "30s"
```

### 配置域名

文档地址：[Configuring domain names](https://knative.dev/docs/serving/using-a-custom-domain/) ，配置实例：[configmaps/domain.yaml](https://github.com/knative/serving/blob/main/config/core/configmaps/domain.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
  labels:
    app.kubernetes.io/name: knative-serving
    app.kubernetes.io/component: controller
    app.kubernetes.io/version: devel
data:
  test.com: |
    selector:	
      app: test
```

### 使用函数创建服务

下载 [knative/func](https://github.com/knative/func/) 客户端，执行命令创建一个项目

```bash
func create -l go  kweb
```

修改 func.yaml 文件

```yaml
specVersion: 0.35.0
name: kweb
runtime: go
registry: docker.io/xxx
image: docker.io/xxx/kweb:v1
imageDigest: ""
created: xxx
build:
  builderImages:
    pack: anjia0532/paketo-buildpacks.builder:base
  buildpacks:
    - paketo-buildpacks/go-dist
    - ghcr.io/boson-project/go-function-buildpack:tip
  builder: pack
  buildEnvs:
    - name: GOPROXY
      value: "https://goproxy.cn"
run:
  volumes: []
  envs: []
deploy:
  namespace: default
  remote: false
  annotations: {}
  options: {}
  labels: []
  healthEndpoints:
    liveness: /health/liveness
    readiness: /health/readiness
```

编译并创建服务

```bash
// 本地测试
func run
func invoke
// 执行编译
func build
// 发布镜像
docker tag kweb:v1 docker.io/xxx/kweb:v1
docker push docker.io/xxx/kweb:v1
// 创建服务
func deploy --build=false --push=false
```

