---
title: "Pod"
date: 2022-12-04
draft: false
Categories: [kubernetes]
---

文档：[Pod | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/)

Pod 是可以在 Kubernetes 中创建和管理的、最小的可部署的计算单元。它是一个或多个容器的组合。这些容器共享存储、网络和命名空间，以及如何运行的规范。其它的资源对象都是用来支撑或者扩展 Pod 对象功能的，比如控制器对象是用来管控 Pod 对象的，Service 或者 Ingress 资源对象是用来暴露 Pod 引用对象的，PersistentVolume 资源对象是用来为 Pod 提供存储等等，K8S 不会直接处理容器，而是 Pod，Pod 是由一个或多个 container 组成。基本的好处有：

1. 方便部署、扩展和收缩、方便调度等

2. Pod中的容器共享数据和网络空间，统一的资源管理与分配

在Pod中，所有容器都被同一安排和调度，并运行在共享的上下文中。对于具体应用而言，Pod是它们的逻辑主机，Pod包含业务相关的多个应用容器。



![image-20221206200429843](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/image-20221206200429843.png )

每一个 Pod 都有一个特殊的被称为 “根容器” 的 Pause 容器。Pause 容器对应的镜像属于 Kubernetes 平台的一部分，除了 Pause 容器，每个 Pod 还包含一个或多个紧密相关的用户业务容器。Pause 容器的作用：

- 扮演 Pid=1 的，回收僵尸进程
- 基于 Linux 的 namespace 的共享

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/370d5cae852e417cb610697eb4eb4a31.jpeg)



## Pod 基本使用

Pod 通常不是直接创建的，而是使用工作负载资源创建的

### 创建

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myngx
spec:
  containers:
  - name: ngx
    image: "nginx:1.18-alpine"
```

展示详细信息

```bash
$ kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
myngx   1/1     Running   0          13m   10.244.3.7   lain2   <none>           <none>
```



## 使用 Deployment

deployment 运行一组相同的 Pod（副本水平扩展）、滚动更新。通过副本集管理和创建POD

### 挂载

挂载 ``hostPath`` 主机目录卷实例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: ngx
        image: "nginx:1.18-alpine"
        volumeMounts:  # 声明容器中的挂载位置
        - name: mydata
          mountPath: /data
      - name: alpine  # 测试多容器
        command: ["sh","-c","echo this is alpine && sleep 3600"]
        image: "alpine:3.12"
      volumes:
      - name: mydata
        hostPath:
          path: /home/txl/yaml/data	 # 声明主机节点目录 
          type: Directory  # 指定type
```

hostPath 支持的 type 值如下：

![image-20221204171417323](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/image-20221204171417323.png)

### 共享文件夹

同一个 pod 内的容器都能读写 ``EmptyDir`` 中的文件。常用于临时空间、多容器共享，如日志或者tmp文件需要的临时目录

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: sharedata
              mountPath: /data
        - name: alpine
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","echo this is alpine && sleep 36000"]
          volumeMounts:
            - name: sharedata
              mountPath: /data
      volumes:
        - name:  sharedata
          emptyDir: {}
```

### Init 容器

文档：[Init 容器 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/init-containers/)

Init 容器是一种特殊容器，在 [Pod](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 内的应用容器启动之前运行。Init 容器可以包括一些应用镜像中不存在的实用工具和安装脚本。

Init 容器与普通的容器非常像，除了如下两点：

- 它们总是运行到完成。
- 每个都必须在下一个启动之前成功完成。

如果 Pod 的 Init 容器失败，kubelet 会不断地重启该 Init 容器直到该容器成功为止。 然而，如果 Pod 对应的 restartPolicy 值为 "Never"，Kubernetes 不会重新启动 Pod。

#### 原理

在 Pod 启动过程中，每个 Init 容器会在网络和数据卷初始化之后按顺序启动。 依据 Init 容器在 Pod spec 配置中的出现顺序依次运行。由于 Pod 可能各种原因多次重启，所以 Init 容器中的操作，须具备幂等性。

#### 应用场景

- 环境检查：例如确保应用容器依赖的服务启动后再启动应用容器
- 初始化配置：例如给应用容器准备配置文件

#### 基本配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myngx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      initContainers:
        - name: init-mydb
          image: alpine:3.12
          command: ['sh', '-c', 'echo wait for db && sleep 35 && echo done'] # 模拟等待35s
      containers:
        - name: ngx
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: sharedata
              mountPath: /data
        - name: alpine
          image: alpine:3.12
          imagePullPolicy: IfNotPresent
          command: ["sh","-c","echo this is alpine && sleep 36000"]
          volumeMounts:
            - name: sharedata
              mountPath: /data
      volumes:
        - name:  sharedata
          emptyDir: {}
```

查看 init 容器状态

```bash
$ kubectl get pod
NAME                    READY   STATUS     RESTARTS   AGE
myngx-89dd8586b-k59pl   0/2     Init:0/1   0          8s
```

| 状态                           | 含义                                                 |
| ------------------------------ | ---------------------------------------------------- |
| `Init:N/M`                     | Pod 包含 `M` 个 Init 容器，其中 `N` 个已经运行完成。 |
| `Init:Error`                   | Init 容器已执行失败。                                |
| `Init:CrashLoopBackOff`        | Init 容器执行总是失败。                              |
| `Pending`                      | Pod 还没有开始执行 Init 容器。                       |
| `PodInitializing` or `Running` | Pod 已经完成执行 Init 容器。                         |
