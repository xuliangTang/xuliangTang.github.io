---
title: "Kustomize"
date: 2023-02-07
draft: false
Categories: [kubernetes]
---

kustomize 是一个通过 kustomization 文件定制 kubernetes 对象的工具，它可以通过一些资源生成一些新的资源，也可以定制不同的资源的集合

一个比较典型的场景是我们有一个应用，在不同的环境例如生产环境和测试环境，它的 yaml 配置绝大部分都是相同的，只有个别的字段不同，这时候就可以利用 kustomize 来解决

文档：[The Kustomization File](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/)

## 结构

```
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── dev
    │   ├── kustomization.yaml
    │   └── patch.yaml
    ├── prod
    │   ├── kustomization.yaml
    │   └── patch.yaml
    └── staging
        ├── kustomization.yaml
        └── patch.yaml
```

一个常见的项目 kustomize 项目布局如上所示，可以看到每个环境文件夹里面都有一个 `kustomization.yaml` 文件，这个文件里面就类似配置文件，里面指定源文件以及对应的一些转换文件，例如 patch 等



## 基础模板

base/kustomization.yaml 示例

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
  - deployment.yaml
```

使用 `kustomize build` 命令运行后会把两个文件连接在一起



## 定制

现在我们想要针对一些特定场景进行定制，比如，针对生产环境和测试环境需要由不同的配置

overlays/dev/kustomization.yaml 示例

```yaml
namespace: dev
namePrefix: dev-
commonLabels:
  someName: someValue
  
bases:
  - ../../base
```

overlays/prod/kustomization.yaml 示例

```yaml
namespace: prod
namePrefix: prod-
commonLabels:
  someName: someValue
  
bases:
  - ../../base
```



## patchesStrategicMerge

可以覆盖一些在 base 文件中已有的配置。如修改 dev 环境的副本数量为2个，同时修改 container1 容器的镜像名

overlays/dev/kustomization.yaml 示例

```yaml
namespace: prod
namePrefix: prod-
commonLabels:
  someName: someValue
  
bases:
  - ../../base
  
patchesStrategicMerge:
  - replica.yaml
```

overlays/dev/replica.yaml 示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: container1
          image: dev-image:1.0
```



## patchesJson6902

patchesJson6902 列表中的每个条目都应可以解析为 kubernetes 对象和将应用于该对象的 [JSON patch](https://tools.ietf.org/html/rfc6902)

目标字段指向的 kubernetes 对象的 group、 version、 kind、 name 和 namespace 在同一 kustomization 内 path 字段内容是 JSON patch 文件的相对路径

如修改 dev 环境下 myngx deployoment 的 containerPort

base/deployment.yaml 示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
  namespace: default
spec:
  selector:
    matchLabels:
      app: myngx
  replicas: 1
  template:
    metadata:
      labels:
        app: myngx
    spec:
      containers:
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
			- containerPort: 443
```

overlays/dev/kustomization.yaml 示例

```yaml
namespace: prod
namePrefix: prod-
commonLabels:
  someName: someValue
  
bases:
  - ../../base
  
patchesStrategicMerge:
  - replica.yaml
  
patchesJson6902:
  - target:
      group: app
      version: v1
      kind: Deployment
      name: myngx
    path: port.yaml
```

overlays/dev/port.yaml 示例

```yaml
- op: replace
  path: /spec/template/spec/containers/0/ports/0/containerPort
  value: 8080
- op: replace
  path: /spec/template/spec/containers/0/ports/1/containerPort
  value: 8443
```



## configMapGenerator

可以生成 ConfigMap 资源

```yaml
configMapGenerator:
  - name: myconfig
    literals:
      - host=192.168.0.111
      - port=1234
    files:
      - mycnf.prop
      - mysql57=mysql.cnf
```



## generatorOptions

控制生成 ConfigMap 和 Secret 的行为

```yaml
generatorOptions:
  labels:
    kustomize.generated.resources: somevalue
  annotations:
    kustomize.generated.resource: somevalue
  disableNameSuffixHash: true
```



## 总结

Kustomize 给 Kubernetes 的用户提供一种可以重复使用配置的声明式应用管理，从而在配置工作中用户只需要管理和维护 Kubernetes 的原生 API 对象，而不需要使用复杂的模版。同时，使用 kustomzie 可以仅通过 Kubernetes 声明式 API 资源文件管理任何数量的 kubernetes 定制配置，可以通过 fork/modify/rebase 这样的工作流来管理海量的应用描述文件
