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

从 [Github](https://github.com/istio/istio/releases) 上下载

```bash
# 查看内置配置文件内容
istioctl profile dump demo
istioctl profile dump --config-path components.ingressGateways
```

### demo环境

```bash
istioctl manifest apply --set profile=demo
```

### 配置文件安装

首先使用 istioctl 安装控制平面组件

```bash
istioctl install --set profile=minimal
```

编写 [IstioOperator](https://istio.io/latest/zh/docs/setup/additional-setup/gateway/) 配置文件，示例如下：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: ingress
spec:
  profile: minimal
  meshConfig:
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY # 阻止未注册的外部访问
  components:
    ingressGateways:
      - name: ingressgateway
        namespace: istio-system
        enabled: true
        label:
          istio: ingressgateway
        k8s:
          service:  # 设置ingressgateway service
            type: ClusterIP
  values:
    gateways:
      istio-ingressgateway:
        # Enable gateway injection
        injectionTemplate: gateway
```

然后执行

```bash
istioctl instal -f xxx.yaml
```

卸载

```bash
istioctl x uninstall --purge
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



## 配置网关 JWT 验证

Istio 支持使用 JWT 对终端用户进行身份验证，支持多种 JWT 签名算法，常见的 JWT 签名算法：

- HS256: 对称加密，加密和解密用的是同一个秘钥 
- RS256: RSA 私钥签名，公钥进行验证
- ES256: 类似 RS256

Istio 要求提供 JWKS 格式的信息，用于 JWT 签名验证，是一个 JSON 格式的文件，如：

```json
{
   "keys":[
      {
         "e":"AQAB",
         "kid":"DHFbpoIUqrY8t2zpA2qXfCmr5VO5ZEr4RzHU_-envvQ",
         "kty":"RSA",
         "n":"xAE7eB6qugXyCAG3yhh7pkD..."
      }
   ]
}
```

JWKS 描述一组 JWK 密钥。它能同时描述多个可用的公钥，应用场景之一是密钥的 Rotate。而 JWK，全称是 Json Web Key，它描述了一个加密密钥（公钥或私钥）的各项属性，包括密钥的值。Istio 使用 JWK 描述验证 JWT 签名所需要的信息。在使用 RSA 签名算法时，JWK 描述的应该是用于验证的 RSA 公钥。一个 RSA 公钥的 JWK 描述如下：

```json
{
    "alg": "RS256",  # 算法「可选参数」
    "kty": "RSA",    # 密钥类型
    "use": "sig",    # 被用于签名「可选参数」
    "kid": "NjVBRjY5MDlCMUIwNzU4RTA2QzZFMDQ4QzQ2MDAyQjVDNjk1RTM2Qg",  # key 的唯一 id
    "n": "yeNlzlub94YgerT030codqEztjfU_S6X4DbDA_iVKkjAWtYfPHDzz_sPCT1Axz6isZdf3lHpq_gYX4Sz-cbe4rjmigxUxr-FgKHQy3HeCdK6hNq9ASQvMK9LBOpXDNn7mei6RZWom4wo3CMvvsY1w8tjtfLb-yQwJPltHxShZq5-ihC9irpLI9xEBTgG12q5lGIFPhTl_7inA1PFK97LuSLnTJzW0bj096v_TMDg7pOWm_zHtF53qbVsI0e3v5nmdKXdFf9BjIARRfVrbxVxiZHjU6zL6jY5QJdh1QCmENoejj_ytspMmGW7yMRxzUqgxcAqOBpVm0b-_mW3HoBdjQ",
    "e": "AQAB"
}
```

RSA 是基于大数分解的加密/签名算法，上述参数中，`e` 是公钥的模数(modulus)，`n` 是公钥的指数(exponent)，两个参数都是 base64 字符串。

JWK 中 RSA 公钥的具体定义参见 [RSA Keys - JSON Web Algorithms (JWA)](https://tools.ietf.org/html/rfc7518#page-30)



### JWK 生成

RS256 使用 RSA 算法进行签名，可通过如下命令生成 RSA 密钥：

```bash
# 1. 生成 2048 位（不是 256 位）的 RSA 密钥
openssl genrsa -out rsa-private-key.pem 2048

# 2. 通过密钥生成公钥
openssl rsa -in rsa-private-key.pem -pubout -out rsa-public-key.pem
```

接下来使用 [jwx](https://github.com/lestrrat-go/jwx) 库生成 JWK

```go
func pubKey() []byte {
	f, _ := os.Open("./rsa-public-key.pem")
	b, _ := io.ReadAll(f)
	return b
}

func main() {
	key, err := jwk.ParseKey(pubKey(), jwk.WithPEM(true))
	if err != nil {
		log.Fatalln(err)
	}

	if rsaKey, ok := key.(jwk.RSAPublicKey); ok {
		b, err := json.Marshal(rsaKey)
		if err != nil {
			log.Fatalln(err)
		}
		fmt.Println(string(b))
	}
}
```

可以在 [jwt.io](https://jwt.io/) 中测试密钥的可用性和生成 Token



### 定义 RequestAuthentication

启用 Istio 的身份验证

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-test
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway # 在带有这些 labels 的 ingressgateway/sidecar 上生效
  jwtRules:
    # issuer 即签发者，需要和 JWT payload 中的 iss 属性完全一致
    - issuer: "user@virtuallain.com"
      jwks: |
        {
            "keys": [
                {
                    "e":"AQAB",
                    "kty":"RSA",
                    "n":"tFLKvS2EMOu3vgPnUPkdn5xVau9-dWf0z30_EdbpadQLiVsHH0FqWl-8CgtNtxnUjrI6WN__BMX8jLzvEqKrdZnbTMS0EaTh8lfGbFxNd0qziVHlYZTH-gtPNI4r815y9OuY7DEuR8fG-B_iHuCslN3BcJ4TDF_tzKCF0USGzzEiiRPR4SBtZgz0tmteQgRTv1NfciOwCtedEtXRKnGI5W1GV5u2dmF6UCiWJdgsqHMsVzTXJz_wliVvKhczwhrFZfqvdBoOe_aays89AjcO4x7eUntZVvOlkowaD-UeUeT6ZL8q4oTWGpswA4YNJ_daZmtAU5ho11EW6F3q1YHjVQ"
                 }
            ]
        }
      # jwks 或 jwksUri 二选其一 (测试中发现 jwksUri 会出现各种问题)
      # jwksUri: "http://nginx.test.local/istio/jwks.json"
      forwardOriginalToken: true  # 转发 Authorization 请求头
      outputPayloadToHeader: "Userinfo" # 转发 jwt payload 数据 (base64编码)
```

可以看到 jwtRules 是一个列表，因此可以为每个 issuers 配置不同的 jwtRule.

对同一个 issuers（jwt 签发者），可以通过 jwks 设置多个公钥，以实现JWT签名密钥的轮转。 JWT 的验证规则是：

1. JWT 的 payload 中有 issuer 属性，首先通过 issuer 匹配到对应的 istio 中配置的 jwks。
2. JWT 的 header 中有 kid 属性，第二步在 jwks 的公钥列表中，中找到 kid 相同的公钥。
3. 使用找到的公钥进行 JWT 签名验证。

> 配置中的 `spec.selector` 可以省略，这样会直接在整个 namespace 中生效，而如果是在 `istio-system` 名字空间，该配置将在全集群的所有 sidecar/ingressgateway 上生效

加了转发后，流程图如下：

![image-20230204002255161](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302040023467.png)



### 跨域配置

给 vs 增加 corsPolicy 节点

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: prodvs
  namespace: myistio
spec:
  hosts:
    - prodsvc
    - p.virtuallain.com
  gateways:
    - pp-gateway
    - mesh
  http:
  - match:
    - uri:
       prefix: /
    route:
     - destination:
        host: prodsvc
        subset: v1svc
        port:
          number: 80
    corsPolicy:
      allowOrigins:
        - exact: "*"
      allowMethods:
        - GET
        - POST
        - PATCH
        - PUT
        - DELETE
        - OPTIONS
      allowCredentials: true
      allowHeaders:
        - authorization
      maxAge: "24h"
```



### 定义 Envoy Filter

RequestsAuthentication 不支持自定义响应头信息，这导致对于前后端分离的 Web API 而言， 一旦 JWT 失效，Istio 会直接将 401 返回给前端 Web 页面。 因为响应头中不包含 `Access-Crontrol-Allow-Origin`，响应将被浏览器拦截，需要通过 EnvoyFilter 自定义响应头，添加跨域信息

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: reorder-cors-before-jwt
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.cors"
      patch:
        operation: REMOVE
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.jwt_authn"
      patch:
        operation: INSERT_BEFORE
        value:
          name: "envoy.filters.http.cors"
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.http.cors.v3.Cors"
```



## Authorization Policy

Istio 允许我们使用 AuthorizationPolicy (授权策略) 资源在网格、命名空间和工作负载层面定义访问控制。 支持 DENY、ALLOW、AUDIT 和 CUSTOM 操作。

rule 规则包括 `from` (来源)、`to` (操作) 和 `when` (条件)

**来源**

- `principals` (如 my-service-account): 具有 my-service-account 的工作负载，需要开启 mTLS (双向认证)
- `requestPrincipals` (如 my-issuer/hello): 具有有效 JWT 和请求主体 my-issuer/hello 的工作负载
- `namespaces` (如 default): 任何来自 default 命名空间的工作负载 
- `ipBlocks` (如 ["1.2.3.4", "9.8.7.6/15"]): 任何 IP 或者 CIDR 块的 IP 的工作负载
- `remoteIpBlocks`: 针对 remote.ip (如 X-Forwarded-For)

每个选项都有一个 notXxx 作为反义词

**操作**

- `hosts` 和 `notHosts`
- `ports` 和 `notPorts`
- `methods` 和 `notMethods`
- `path` 和 `notPath`

所有这些操作都适用于请求属性。例如，要在一个特定的请求路径上进行匹配，我们可以使用路径 (如 ["/api/*", "/admin"]) 或特定的端口 (["8080"])

**条件**

为了指定条件，我们必须提供一个 key 字段，key 字段是一个 Istio 属性的名称。例如 `request.headers`、`source.ip`、`request.auth.claims[xx]` 等。条件的第二部分是 values 或 notValues 的字符串列表

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: prod-authpolicy
  namespace: istio-system
spec:
  action: ALLOW
  selector:
    matchLabels:
      istio: ingressgateway
  rules:
    - from:
        - source:
            requestPrincipals: ["*"]
      to:
       - operation:
          methods: ["GET"]
          paths: ["/prods/*"]
    - to:
        - operation:
            methods: ["GET","POST"]
            paths: ["/admin"]
      when: # payload 中必须包含值为 admin 或 superadmin 的 role 字段
        - key: request.auth.claims[role]
          values: ["admin","superadmin"]
```



## 开启自动 mTLS 

服务中的工作负载之间的通信是通过 Envoy 代理进行的。当一个工作负载使用 mTLS 向另一个工作负载发送请求时，Istio 会将流量重新路由到 sidecar 代理（Envoy）

然后，sidecar Envoy 开始与服务器端的 Envoy 进行 mTLS 握手。在握手过程中，调用者会进行安全命名检查，以验证服务器证书中的服务账户是否被授权运行目标服务。一旦 mTLS 连接建立，Istio 就会将请求从客户端的 Envoy 代理转发到服务器端的 Envoy 代理。在服务器端的授权后，sidecar 将流量转发到工作负载

可以创建 PeerAuthentication 资源，首先在每个命名空间中分别执行严格模式。然后，我们可以在根命名空间创建一个策略，在整个服务网格中执行该策略。或者指定 selector，仅应用于网格中的特定工作负载。它有三大模式：

- `PERMISSIVE`：同时接受未加密连接和双向加密连接
- `STRICT`：只接受加密连接
- `DISABLE`：关闭双向加密连接

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: testmtls
  namespace: myistio
spec:
  selector:
    matchLabels:
      app: reviews
  mtls:
    mode: STRICT
```
