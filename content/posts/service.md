---
title: "Service"
date: 2022-12-11
draft: false
Categories: [kubernetes]
---

在 Kubernetes 中，pod 是应用程序的载体，我们可以通过 pod 的 ip 来访问应用程序，但是 pod 的 ip 地址不是固定的，这也就意味着不方便直接采用 pod 的 ip 对服务进行访问

为了解决这个问题，Kubernetes 提供了 service 资源，service 会对提供同一个服务的多个 pod 进行聚合，并且提供一个统一的入口地址，通过访问 service 的入口地址就能访问到后面的 pod 服务。

通过 service 可以提供负载均衡和服务自动发现

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210809170841691-1245411009.png)



## 服务类型

- ClusterIP：k8s 默认的 ServiceType，通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问
- NodePort：用来对集群外暴露 Service，你可以通过访问集群内的每个 NodeIP:NodePort 的方式，访问到对应 Service 后端的 Endpoint
- LoadBalancer: 这也是用来对集群外暴露服务的，不同的是这需要外部负载均衡器的云提供商，比如 AWS 等
- ExternalName：这个也是在集群内发布服务用的，需要借助 KubeDNS(version >= 1.7) 的支持，就是用KubeDNS 将该 service 和 ExternalName 做一个 Map，KubeDNS 返回一个 CNAME 记录。

每种服务类型都是会指定一个 clusterIP 的，由 clusterIP 进入对应代理模式实现负载均衡，如果强制 `spec.clusterIP: "None"`（即 headless service），集群无法为它们实现负载均衡，直接通过 pod 域名访问pod，典型是应用是 StatefulSet。



## Service 使用

### 创建

创建 deployment 信息，设置 app=nginx 的标签

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
```

创建一个名为 nginx-svc 的 service 对象，它会将请求代理到80端口且具有标签 `` app: nginx`` 的 pod 上

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:		# 通过selector和pod建立关联
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```



## EndPoint

Endpoint 是 k8s 中的一个资源对象，存储在 etcd 中，用来记录一个 service 对应的所有 pod 的访问地址，它是根据 service 配置文件中的 selector 描述产生的

一个 service 由一组 pod 组成，这些 pod 通过 endpoints 暴露出来，**endpoints 是实现实际服务的端点集合**。换句话说，service 和 pod 之间的联系是通过 endpoints 实现的。

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210811115326320-1833457826.png)

```bash
$ kubectl get endpoints
NAME         ENDPOINTS                        AGE
kubernetes   192.168.0.111:6443               12d
nginx-svc    10.244.0.181:80,10.244.3.32:80   59m
```



## kube-proxy

主要负责 pod 网络代理，维护网络规则和四层负载均衡工作

service 在很多情况下只是一个概念，真正起作用的其实是 kube-proxy 服务进程，每个 node 节点上都运行一个kube-proxy 服务进程，当创建 service 的时候会通过 api-server 向 etcd 写入创建的 service 信息，而 kube-proxy 会基于监听的机制发现这种 service 的变动，然后**它会将最新的 service 信息转换成对应的访问规则**

kube-proxy 监听 10249 和 10256 端口，对外提供 /metrics 和 /healthy 的访问

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210809171742862-1608927520.png)

```bash
# 查看配置
$ kubectl describe cm kube-proxy -n kube-system

# 查看 kube-proxy pod
$ kubectl get pods -n kube-system | grep kube-proxy
kube-proxy-bxk96                1/1     Running   2          11d
kube-proxy-xbv75                1/1     Running   4          12d
```

### userspace 模式(废弃)

### iptables 模式(默认模式)

iptables 模式下，节点上 kube-proxy 持续监听 Service 以及 Endpoints 对象的变化，为 service 后端的每个 pod 创建对应的 iptables 规则，当捕获到 Service 的 clusterIP 和端口请求，利用注入的 iptables，将请求重定向到 Service 的对应的 Pod

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210810103911436-340397863.png)

### IPVS 模式

IPVS 模式是利用 linux 的 IPVS 模块实现，同样是由 kube-proxy 实时监视集群的 service 和 endpoint。基于内核内哈希表，有更高的网络流量吞吐量（iptables 模式在大规模集群，比如10000 个服务中性能下降显著），并且具有更复杂的负载均衡算法（最小连接、局部性、 加权、持久性）

当 kube-proxy 以 IPVS 代理模式启动时，它将验证 IPVS 内核模块是否可用。 如果未检测到 IPVS 内核模块，则 kube-proxy 将退回到以 iptables 代理模式运行。

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/2164474-20210810104618211-702040131.png)



## 无头(HeadLiness) 类型的 Service

在某些场景中，开发人员可能不想使用 Service 提供的负载均衡功能，而希望自己来控制负载均衡策略，针对这种情况，k8s 提供了 HeadLiness Service，这类 Service 不会分配 ClusterIP，如果想要访问 Service，只能通过service 的域名进行查询

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  clusterIP: "None"		# 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```



## 宿主机访问 Service

### 安装 bind-utils

```bash
$ sudo yum install bind-utils -y

# 无法直接查询到service对应的ip
$ nslookup nginx-svc
** server can't find nginx-svc: NXDOMAIN
```

### 设置解析

查看 kube-dns clusterIp

```bash
$ kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   11d
```

修改 ``/etc/resolv.conf`` 文件，设置DNS服务器IP地址、DNS域名和设置主机的域名搜索顺序。加入内容

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local
```

访问

```bash
$ curl nginx-svc
```
