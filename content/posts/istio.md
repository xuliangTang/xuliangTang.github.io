---
title: "Istio 快速入门"
date: 2023-02-02
draft: false
Categories: [Istio]
---

[Istio](https://istio.io/latest/zh/docs/) 是一个开源的微服务管理、保护和监控框架，它有如下特性：

1. 流量管理：利用配置，我们可以控制服务间的流量。设置断路器、超时或重试都可以通过简单的配置改变来完成。
2. 可观察性：Istio 通过跟踪、监控和记录让我们更好地了解服务，让我们能够快速发现和修复问题。
3. 安全性：Istio 可以在代理层面上管理认证、授权和通信的加密。我们可以通过快速的配置变更在各个服务中执行策略。

## Istio 组件

Istio 服务网格有两个部分：数据平面和控制平面

- 数据平面由一组智能代理 (Envoy) 组成，被部署为 sidecar，控制服务之间的通信
- 控制平面管理并配置代理来进行流量路由

下图展示了组成每个平面的不同组件：

![Istio架构图](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302022108721.png)



## 安装

从 [Github](https://github.com/istio/istio/releases) 上下载 Istio，执行

```bash
istioctl manifest apply --set profile=demo
```



## Sidecar 注入

网格中的 Pod 必须运行一个 Istio Sidecar 代理，两种方法：使用 istioctl 手动注入或启用 Pod 所属命名空间的 Istio sidecar 注入器自动注入，启用自动注入后，自动注入器会使用准入控制器在创建 Pod 时自动注入代理配置

![Sidecar 模式示意图](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302022122730.jpeg)

### 手动注入

```bash
 istioctl kube-inject -f api.yaml | kubectl apply -f –
```

### 自动注入

将 myistio 命名空间标记为 `istio-injection=enabled`

```bash
kubectl label namespace myistio istio-injection=enabled
```



## 示例配置

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: p-gateway
  namespace: myistio
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - p.virtuallain.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: prodvs
  namespace: myistio
spec:
  hosts:
    - prodsvc
    - reviewsvc
    - p.virtuallain.com
  gateways:
    - p-gateway
    - mesh # 用于内部服务互访
  http:
  - match:
    - uri:
       prefix: "/p"
    rewrite:
       uri: "/prods" # 路径重写
    route:
     - destination:
        host: prodsvc
        subset: v1svc # 服务端点
        port:
          number: 80
    fault: # 延迟故障注入
      delay:
        fixedDelay: 1ms
        percentage:
          value: 15
    timeout: 1s
  - match:
    - uri:
        prefix: /
    route:
      - destination:
          host: reviewsvc
          port:
            number: 80
#    fault:
#      abort:
#        httpStatus: 500
#        percentage:
#          value: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: prod-rule
  namespace: myistio
spec:
  host: prodsvc
  trafficPolicy:
    loadBalancer: # 负载均衡
#      simple: ROUND_ROBIN
      consistentHash:
        httpHeaderName: myname
    connectionPool: # 限流
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection: # 熔断
      consecutive5xxErrors: 2
      interval: 10s
      maxEjectionPercent: 50
      baseEjectionTime: 10s
  subsets: # 定义服务端点集合
    - name: v1svc
      labels:
        # version: v2
        app: prod
```



## Gateway

网格服务的负载均衡器，负责网格边缘的服务暴露。可以使用 Gateway 资源来配置网关，网关资源描述了负责均衡器的暴露端口、协议、SNI (服务器名称指示) 配置等。网关资源在背后控制着 Envoy 代理在网络接口上的监听方式以及它出示的证书，其中

- Ingress-gateway: 用于管理网格边缘入站的流量，通过入口网关将网格内部的服务暴露到外部提供访问，配合 VirtualService 
- Egress gateway: 控制网格内服务访问外部服务，配合 DestinationRule   



## Virtual Service

虚拟服务可以理解为 k8s service 的一种增强，我们可以使用 VirtualService 资源在 Istio 服务网格中定义流量路由规则，并在客户端试图连接到服务时应用这些规则

1. 配置如何在服务网格内将请求路由到服务
2. 和网关整合并配置流量规则来控制出入流量



## Destination Rule

目标规则用于配置将流量转发到实际工作负载时应用的策略，如流量拆分、灰度发布、负载均衡等

subset (子集) 是服务端点的集合，可以用于 A/B 测试或者分版本路由等场景

### 负载均衡器设置

基本配置参数：

1. 简单类型 (`simple`)：
   - `ROUND_ROBIN`: 轮询 (默认)
   - `LEAST_CONN`: 最少连接，选择请求较少的主机
   - `RANDOM`: 随机选择

2. 一致性哈希 (`consistentHash`): 目的是让同一用户的请求一直转发到后端同一实例
   - `httpHeaderName`: 基于 header 头
   - `httpCookie`: 基于 cookie
   - `useSourceIp`: 基于 ip
   - `minimumRingSize`: 环形哈希算法，适用于较多节点、频繁新增和删除节点的场景，填数字虚拟节点数量

### connectionPool 限流设置

ConnectionPool 可以对上游服务的并发连接数和请求数进行限制。适用于 HTTP 和 TCP 服务

基本配置参数：

**TCP**

- `maxConnections`: 最大 HTTP1 / TCP 连接数 (默认值2 ^ 32-1)
- `connectTimeout`: 连接超时。格式为 1h / 1m / 1s / 1ms (默认值为10秒)
- `tcpKeepalive`: keepalive 设置
  - probes: 确认连接dead之前继续发送探测数据，发送几次 (默认9)
  - time: 发送probe之前连接的空闲时间 (默认2小时)
  - interval: 两次probe发送的间隔 (默认75秒)

**HTTP**

- `http1MaxPendingRequests`: 等待时将排队的最大请求数 就绪的连接池连接 (默认为1024)
- `http2MaxRequests`: 对目标的活动请求的最大数量 (默认为1024)
- `maxRequestsPerConnection`: 每个连接的最大请求数量。如果将这一参数设置为 1 则会禁止 keepalive 特性
- `idleTimeout`: 上游连接池连接的空闲超时，当达到空闲超时时，连接将被关闭
- `maxRetries`: 最大重试次数 (默认为3)

### outlierDetection 熔断设置

跟踪每个状态的断路器实现上游服务中的单个主机。适用于 HTTP 和 TCP 服务

基本配置参数：

- `consecutive5xxErrors`: 连续 5xx 错误异常数
- `interval`: 错误异常的扫描间隔 (默认10秒) ，期间连续发生指定数量个异常则熔断
- `baseEjectionTime`: 驱逐时间 (默认30秒)
- `maxEjectionPercent`: 最大驱逐百分比 (默认10%)
- `minHealthPercent`: 至少最低健康百分比的主机，就将启用异常检测。当负载平衡池中的正常主机百分比降至此阈值以下时，异常检测将被禁用默认值为0％，因为它通常不适用于每项服务只有少量 pod 的 k8s 环境



## 故障注入

为了帮助我们提高服务的弹性，我们可以使用故障注入功能。我们可以在 HTTP 流量上应用故障注入策略，在转发目的地的请求时指定一个或多个故障注入

有两种类型的故障注入。我们可以在转发前延迟（delay）请求，模拟缓慢的网络或过载的服务，我们可以中止（abort） HTTP 请求，并返回一个特定的 HTTP 错误代码给调用者。通过中止，我们可以模拟一个有故障的上游服务

