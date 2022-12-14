---
title: "PV 和 PVC"
date: 2022-12-11
draft: false
Categories: [kubernetes]
---

为了能够屏蔽底层存储实现的细节，方便用户使用，k8s 引入 PV 和 PVC 两种资源对象。Persistent Volume 提供存储资源（并实现），Persistent Volume Claim 描述需要的存储标准，然后从现有 PV 中匹配或者动态建立新的资源，最后将两者进行绑定。

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210816114501146-1639684923.png)



## 持久卷（Persistent Volume）

PV 是持久化卷的意思，是对底层的共享存储的一种抽象。一般情况下 PV 由 k8s 管理员进行创建和配置，它与底层具体的共享存储技术有关，并通过插件（如：local、NFS）等具体的底层技术来实现完成与共享存储的对接。

### 持久卷的类型

- cephfs - CephFS volume
- csi - 容器存储接口 (CSI)
- fc - Fibre Channel (FC) 存储
- hostPath - HostPath 卷 （仅供单节点测试使用；不适用于多节点集群；请尝试使用 local 卷作为替代）
- iscsi - iSCSI (SCSI over IP) 存储
- local - 节点上挂载的本地存储设备
- nfs - 网络文件系统 (NFS) 存储
- rbd - Rados 块设备 (RBD) 卷

### 创建

使用 local 卷的资源清单

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
     - ReadWriteOnce
  storageClassName: ""		# 存储类别
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/txl/data
  nodeAffinity:
    required:				# 指定必须满足的硬性节点约束
      nodeSelectorTerms:	# 节点选择器条件的列表
      - matchExpressions:	# 基于节点标签所设置的节点选择器要求的列表
        - key: pv			# 适用的标签主键
          operator: In		# 代表主键与值集之间的关系
          values:
          - local
```

### 关键配置参数

- **储存能力** ``capacity``：目前只支持存储空间的设置（storage=1Gi）
- **卷模式** ``volumeMode``：设置为 `Filesystem` 的卷会被 Pod 挂载（Mount）到某个目录。 如果卷的存储来自某块设备而该设备目前为空，Kuberneretes 会在第一次挂载卷之前在设备上创建文件系统
- **访问模式** ``accessModes``：用于描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：
  1. ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载
  2. ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载
  3. ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

- **存储类别** ``storageClassName``：PV 可以通过 storageClassName 参数指定一个存储类别：
  - 具有特定类别的 PV 只能与请求了该类别的 PVC 进行绑定
  - 未设定类别的 PV 只能与不请求任何类别的 PVC 进行绑定

- **回收策略** ``persistentVolumeReclaimPolicy``：当 PV 不再被使用了之后，对其的处理方式。目前支持三种策略：
  1. Retain（保留）：保留数据，需要管理员手动清理数据
  2. Recycle（回收）：清除PV中的数据，效果相当于执行 rm -rf /thevolume/*
  3. Delete（删除）：与PV相连的后端存储完成volume的删除操作，当然这常见于云服务商的存储服务

- **节点亲和性** ``NodeAffinity``：定义一些约束，进而限制从哪些节点上可以访问此卷。``matchExpressions`` 的 operator包括：
  - In：label 的值在某个列表中
  - NotIn：label 的值不在某个列表中
  - Exists：某个 label 存在
  - DoesNotExist：某个 label 不存在
  - Gt：label 的值大于某个值（字符串比较）
  - Lt：label 的值小于某个值（字符串比较）

- **状态** ``status``：一个 PV 的生命周期中，可能会处于4种不同的阶段
  1. Available（可用）：表示可用状态，还未被任何PVC绑定
  2. Bound（已绑定）：表示PV已经被PVC绑定
  3. Released（已释放）：表示PVC被删除，但是资源还未被集群重新声明
  4. Failed（失败）：表示该PV的自动回收失败

### 标签

查看标签

````bash
$ kubectl get node --show-labels=true
````

给 node lain1 打一个标签  pv=local

````bash
$ kubectl label nodes lain1 pv=local
````

删除标签

````bash
$ kubectl label nodes lain1 pv-
````



## 持久卷声明（PersistentVolumeClaim）

PVC 是资源的申请，用来声明对存储空间、访问模式、存储类别需求信息

### 创建

绑定：spec 关键字段要匹配，storageClassName 字段必须一致  

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ngx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 1Gi
```



## 存储类（StorageClass）

Kubernetes 提供了一套可以自动创建 PV 的机制，即：Dynamic Provisioning。而这个机制的核心在于StorageClass 这个 API 对象。StorageClass 对象会定义下面两部分内容:

1. PV 的属性，如存储类型，Volume 的大小等。
2. 创建这种 PV 需要用到的存储插件，即存储制备器。

有了这两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass，之后Kubernetes 就会调用该 StorageClass 声明的存储插件，进而创建出需要的 PV。

### 为什么需要 StorageClass

在一个大规模的 Kubernetes 集群里，可能有成千上万个 PVC，这就意味着运维人员必须实现创建出这个多个 PV，此外，随着项目的需要，会有新的 PVC 不断被提交，那么运维人员就需要不断的添加新的，满足要求的 PV，否则新的 Pod 就会因为 PVC 绑定不到 PV 而导致创建失败。而且通过 PVC 请求到一定的存储空间也很有可能不足以满足应用对于存储设备的各种需求。

而且不同的应用程序对于存储性能的要求可能也不尽相同，比如读写速度、并发性能等，为了解决这一问题，Kubernetes 又为我们引入了一个新的资源对象：StorageClass，通过 StorageClass 的定义，管理员可以将存储资源定义为某种类型的资源，比如快速存储、慢速存储等，用户根据 StorageClass 的描述就可以非常直观的知道各种存储资源的具体特性了，这样就可以根据应用的特性去申请合适的存储资源了。

### 创建

资源清单如下

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: Local		# 卷插件（如：local NFS）
reclaimPolicy: Retain	# 回收策略
volumeBindingMode: Immediate	# 绑定模式
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/txl/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
          - key: pv
            operator: In
            values:
            - local
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ngx-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx-sc
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
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: mydata
              mountPath: /data
          ports:
            - containerPort: 80
      volumes:
        - name: mydata
          persistentVolumeClaim:
            claimName: ngx-pvc
```

### 关键配置参数

- **绑定模式** ``WaitForFirstConsumer``：控制卷绑定和动态制备应该发生在什么时候
  - Immediate：一旦创建 PVC 就绑定
  - WaitForFirstConsumer：延迟绑定，直到使用该 PVC 的 Pod 被创建



## 挂载到 Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ngx-pv
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
        - name: ngx1
          image: nginx:1.18-alpine
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: mydata
              mountPath: /data
          ports:
            - containerPort: 80
      volumes:
        - name: mydata
          persistentVolumeClaim:
            claimName: ngx-pvc
```
