---
title: "GitOps 持续部署工具 Argo CD"
date: 2023-04-07
draft: false
Categories: [kubernetes]
---

Argo CD 是以 Kubernetes 作为基础设施，遵循声明式 GitOps 理念的持续交付（continuous delivery, CD）工具，支持多种配置管理工具，包括 ksonnet/jsonnet、kustomize 和 Helm 等。它的配置和使用非常简单，并且自带一个简单易用的可视化界面

按照官方定义，Argo CD 被实现为一个 Kubernetes 控制器，它会持续监控正在运行的应用，并将当前的实际状态与 Git 仓库中声明的期望状态进行比较，如果实际状态不符合期望状态，就会更新应用的实际状态以匹配期望状态



## 对比传统 CD 工作流

和传统 CI/CD 工具一样，CI 部分并没有什么区别，无非就是测试、构建镜像、推送镜像、修改部署清单等等。重点在于 CD 部分

Argo CD 会被部署在 Kubernetes 集群中，使用的是基于 Pull 的部署模式，它会周期性地监控应用的实际状态，也会周期性地拉取 Git 仓库中的配置清单，并将实际状态与期望状态进行比较，如果实际状态不符合期望状态，就会更新应用的实际状态以匹配期望状态

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202304072331407.png)

无论是通过 CI 流水线触发更新 K8s 编排文件，还是 DevOps 工程师直接修改 K8s 编排文件，Argo CD 都会自动拉取最新的配置并应用到 K8s 集群中

最终会得到一个相互隔离的 CI 与 CD 流水线，CI 流水线通常由研发人员（或者 DevOps 团队）控制，CD 流水线通常由集群管理员（或者 DevOps 团队）控制

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202304072333308.png)



## Argo CD 的优势

**1. Git 作为应用的唯一真实来源**

所有 K8s 的声明式配置都保存在 Git 中，并把 Git 作为应用的唯一事实来源，我们不再需要手动更新应用（比如执行脚本，执行 kubectl apply 或者 helm install 命令），只需要通过统一的接口（Git）来更新应用

此外，Argo CD 不仅会监控 Git 仓库中声明的期望状态，还会监控集群中应用的实际状态，并将两种状态进行对比，只要实际状态不符合期望状态，实际状态就会被修正与期望状态一致。所以即使有人修改了集群中应用的状态（比如修改了副本数量），Argo CD 还是会将其恢复到之前的状态。这就真正确保了 Git 仓库中的编排文件可以作为集群状态的唯一真实来源

当然，有时候我们需要快速更新应用并进行调试，通过 Git 来触发更新还是慢了点，这也不是没有办法，我们可以修改 Argo CD 的配置，使其不对手动修改的部分进行覆盖或者回退，而是直接发送告警，提醒管理员不要忘了将更新提交到 Git 仓库中

**2. 快速回滚**

Argo CD 会定期拉取最新配置并应用到集群中，一旦最新的配置导致应用出现了故障（比如应用启动失败），我们可以通过 Git History 将应用状态快速恢复到上一个可用的状态

如果你有多个 Kubernetes 集群使用同一个 Git 仓库，这个优势会更明显，因为你不需要分别在不同的集群中通过 `kubectl delete` 或者 `helm uninstall` 等手动方式进行回滚，只需要将 Git 仓库回滚到上一个可用的版本，Argo CD 便会自动同步

**3. 集群灾备**

如果你在 [青云](https://www.qingcloud.com/)北京3区中的 [KubeSphere](https://kubesphere.com.cn/) 集群出现故障，且短期内不可恢复，可以直接创建一个新集群，然后将 Argo CD 连接到 Git 仓库，这个仓库包含了整个集群的所有配置声明。最终新集群的状态会与之前旧集群的状态一致，完全不需要人工干预

**4. 使用 Git 实现访问控制**

通常在生产环境中是不允许所有人访问 Kubernetes 集群的，如果直接在 Kubernetes 集群中控制访问权限，必须要使用复杂的 RBAC 规则。在 Git 仓库中控制权限就比较简单了，例如所有人（DevOps 团队，运维团队，研发团队，等等）都可以向仓库中提交 Pull Request，但只有高级工程师可以合并 Pull Request

这样做的好处是，除了集群管理员和少数人员之外，其他人不再需要直接访问 Kubernetes 集群，只需访问 Git 仓库即可。对于程序而言也是如此，类似于 Jenkins 这样的 CI 工具也不再需要访问 Kubernetes 的权限，因为只有 Argo CD 才可以 apply 配置清单，而且 Argo CD 已经部署在 Kubernetes 集群中，必要的访问权限已经配置妥当，这样就不需要给集群外的任意人或工具提供访问的证书，可以提供更强大的安全保障

**5. 扩展 Kubernetes**

虽然 Argo CD 可以部署在 Kubernetes 集群中，享受 Kubernetes 带来的好处，但这不是 Argo CD 专属的呀！Jenkins 不是也可以部署在 Kubernetes 中吗？Argo CD 有啥特殊的吗？

那当然有了，没这金刚钻也不敢揽这瓷器活啊，Argo CD 巧妙地利用了 Kubernetes 集群中的很多功能来实现自己的目的，例如所有的资源都存储在 Etcd 集群中，利用 Kubernetes 的控制器来监控应用的实际状态并与期望状态进行对比，等等

这样做最直观的好处就是**可以实时感知应用的部署状态**。例如，当你在 Git 仓库中更新配置清单中的镜像版本后，Argo CD 会将集群中的应用更新到最新版本，你可以在 Argo CD 的可视化界面中实时查看更新状态（比如 Pod 创建成功，应用成功运行并且处于健康状态，或者应用运行失败需要进行回滚操作）

## 部署

### Argo CD

参考 [Getting Started - Argo CD](https://argo-cd.readthedocs.io/en/stable/getting_started/)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

需要手动修改一些拉取不下来的镜像地址，参考 [示例 yaml](/argocd/argocd2.12.yaml)

### 客户端工具

从 [argoproj/argo-cd](https://github.com/argoproj/argo-cd/releases/tag/v2.1.2) 下载

### 创建 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-argocd
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.xxx.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

### 修改 nginx-ingress 启动参数

例如使用 helm 安装，修改 values.yaml 文件 (约150行左右)，给 extraArgs 节点增加 enable-ssl-passthrough 配置

```yaml
extraArgs:
  enable-ssl-passthrough: ""
```

更新 nginx-ingress

```bash
helm upgrade ingress-nginx -n ingress-nginx .
```

**获取初始密码**

初始密码以明文形式存储在 Secret `argocd-initial-admin-secret` 中，可以通过以下命令获取：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```



## 使用客户端工具

修改 argocd-server 暴露 NodePort 端口

```bash
kubectl edit svc argocd-server -n argocd
```

登录

```bash
argocd --insecure login localhost:32443 --name myargocd
```

输入用户名密码成功后默认生成配置文件到 ~/.argocd/config 中



## Demo

**1. 创建 Repositories 关联 Git 仓库**

来源可以是原生的 Kubernetes 配置清单，也可以是 Helm Chart 或者 Kustomize 部署。示例：

```yaml
// kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - svc.yaml
  - deploy.yaml
  
// deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      imagePullSecrets:
        - name: aliyun-img
      containers:
        - name: test
          image: registry.cn-hangzhou.aliyuncs.com/xxx/test:v1
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: config
              mountPath: /app/application.yml
              subPath: application.yml
          ports:
            - containerPort: 80
      volumes:
        - name: config
          configMap:
            name: test
            
// svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: test
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: test
```

**2. 创建 Application** ( 可选手动或自动同步 )

**3. 使用客户端工具手动同步**

```bash
argocd app sync myapp
```

### 使用 Tekton 任务触发 argocd

创建 argocd 登录 secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argo-auth
  namespace: default
type: Opaque
stringData:
  username: admin
  password:
```

创建 tekton task 示例

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: argocdtask
spec:
  volumes:
    - name: argo-auth
      secret:
        secretName: argo-auth
  steps:
    - name: deploy
      image: argoproj/argocd:v2.1.2
      volumeMounts:
        - name: argo-auth
          readOnly: true
          mountPath: /var/.argo
      env:
        - name: argocdserver
          value: "argocd-server.argocd.svc.cluster.local"
      command:
       - sh
      args:
       - -c
       - |
         argocd --insecure login ${argocdserver} --username $(cat /var/.argo/username) --password $(cat /var/.argo/password)
         argocd app sync myapp
         argocd app wait myapp --health
```

测试运行 task

```bash
tkn task start argocdtask
```



*参考：[Argo CD 入门教程 – 云原生实验室](https://icloudnative.io/posts/getting-started-with-argocd/)*
