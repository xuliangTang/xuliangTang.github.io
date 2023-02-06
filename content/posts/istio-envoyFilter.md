---
title: "Istio Envoy Filter"
date: 2023-02-05
draft: false
Categories: [Istio]
---

EnvoyFilter 资源允许你定制由 Istio Pilot 生成的 Envoy 配置。使用该资源，你可以更新数值，添加特定的过滤器，甚至添加新的监听器、集群等等。小心使用这个功能，因为不正确的定制可能会破坏整个网格的稳定性。

这些 EnvoyFilter 被应用的顺序是：首先是配置在根命名空间中的所有 EnvoyFilter，其次是配置在工作负载命名空间中的所有匹配的 EnvoyFilter。当多个 EnvoyFilter 被绑定到给定命名空间中的相同工作负载时，将按照创建时间的顺序依次应用。如果有多个 EnvoyFilter 配置相互冲突，那么将无法确定哪个配置被应用。

![image-20230205201019523](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302052010087.png)

配置参考文档：[Istio EnvoyFilter](https://istio.io/latest/docs/reference/config/networking/envoy-filter/#EnvoyFilter)、[Envoy](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters)



## 基本示例

过滤器：

- Network Filters：网络过滤器。处理连接的核心
- HTTP Filters：HTTP 过滤器。由特殊的网络过滤器 HttpConnectionManager 管理，处理 HTTP1/HTTP2/gRPC 请求

可以参考 envoy  文档：全部 [http_filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters)、全部 [filter](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/filter/filter)

### 增加响应头

下面的示例中在响应中添加了一个名为 api-version 的头

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: myfilter
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY # 网关侦听器
        proxy:
          proxyVersion: ^1\.11.*
        listener:
          filterChain: # 匹配侦听器中的特定筛选器链
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter: # 此筛选器中要匹配的下一级筛选器
                name: "envoy.filters.http.router"
      patch:
        operation: INSERT_BEFORE
        value:
          name: my.lua
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
            inlineCode: |
              function envoy_on_response(response_handle)
                 response_handle:headers():add("api-version", "1.0")
              end
```

再创建一个 filter，响应时给 api-version 头加上前缀

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: myfilter-prefix
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        proxy:
          proxyVersion: ^1\.11.*
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "my.lua"
      patch:
        operation: INSERT_BEFORE # 在 my.lua 之前插入
        value:
          name: myprefix.lua
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
            inlineCode: |
              function envoy_on_response(response_handle)
                 local ver = response_handle:headers():get("Api-version")
                 response_handle:headers():replace("Api-version", "version_"..ver)
              end
```

filter 在请求时会按照从前向后的顺序执行，响应时则相反

### 增加请求头

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: myfilter-adduserid
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        proxy:
          proxyVersion: ^1\.11.*
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "my.lua"
      patch:
        operation: INSERT_BEFORE
        value:
          name: adduserid.lua
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
            inlineCode: |
              function envoy_on_request(request_handle)
                 request_handle:headers():add("userid", "101")
              end
```

### 查看动态配置

在 istio-ingressgateway 服务的 pod 中执行

```bash
curl http://localhost:15000/config_dump?resource=dynamic_listeners
```

### 打印 Lua 日志

默认情况只会打印 err 以上级别的日志，可以进入 pod 临时开启

```bash
curl -X POST http://localhost:15000/logging?level=info
```

输出 info 日志

```lua
function envoy_on_request(request_handle)
	local userid = request_handle:headers():get("userid")
	request_handle:logInfo("userId="..userid)
end
```

### 结束响应

如请求时没有携带 appid 头，则直接结束响应

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: myfilter-checkappid
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        proxy:
          proxyVersion: ^1\.11.*
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.cors"
      patch:
        operation: INSERT_AFTER # 在cors后插入，确保响应时携带跨域头
        value:
          name: checkappid.lua
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
            inlineCode: |
              function envoy_on_request(request_handle)
                 local appid = request_handle:headers():get("appid")
                 if(appid == nil) then
                    request_handle:respond(
                      {[":status"] = "400"},
                      "error appid"
                    )
                 end
              end
```



## Envoy 将 gRPC 转码为 HTTP/JSON 

一旦有了一个可用的 gRPC 服务，可以通过向服务添加一些额外的注解（annotation）将其作为 HTTP/JSON API 发布。你需要一个代理来转换 HTTP/JSON 调用并将其传递给 gRPC 服务。我们称这个过程为转码。然后你的服务就可以通过 gRPC 和 HTTP/JSON 访问。

### 步骤1：使用HTTP选项标注服务进行转码

在每个 rpc 操作的花括号中可以添加选项，允许你指定如何将操作转换到 HTTP 请求（endpoint）。在 proto 中引入 ‘ google/api/annotations.proto’ 即可使用该选项

```
import "google/api/annotations.proto";
```

**转码为 GET 方法**

```protobuf
service ProdService {
  rpc GetProd(ProdRequest) returns (ProdResponse) {
    option (google.api.http) = {
      get: "/detail/{prod_id}"
    };
  }
}
```

在 URL 中有一个名为 prod_id 的路径变量，这个变量会自动映射到输入操作中同名的字段

**转码为 POST 方法**

```protobuf
service ProdService {
  rpc GetProd(ProdRequest) returns (ProdResponse) {
    option (google.api.http) = {
      post: "/detail"
      body: "*"
    };
  }
}
```

### 步骤2：生成 descriptor 

descriptor 文件是 ProtoBuf 提供的动态解析机制，通过提供对应类（对象）的 Descriptor 对象，在解析时就可以动态获取类成员

```bash
protoc --proto_path=gsrc/protos --include_imports --include_source_info --descriptor_set_out=prod.descriptor prod_service.proto
```

### 步骤3：转码

可以使用 [grpc-transcoder](https://github.com/AliyunContainerService/grpc-transcoder) 库

```bash
go get github.com/AliyunContainerService/grpc-transcoder
```

执行

```bash
grpc-transcoder --version 1.11 --service_port 80 --service_name gprodsvc.myistio  --proto_svc ProdService --descriptor prod.descriptor 
```

- service_port：service 端口
- service_name：service 全路径名称
- proto_svc：proto service 名称
- descriptor：生成的 descriptor 文件

执行成功会在当前目录下生成一个 `grpc-transcoder-envoyfilter.yaml` 和 `header2metadata-envoyfilter.yaml` 文件

### 步骤4：创建过滤器

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: grpcfilter
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: grpc-ingressgateway
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        proxy:
          proxyVersion: ^1\.11.*
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.router"
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.grpc_json_transcoder
          typed_config:
            '@type': "type.googleapis.com/envoy.extensions.filters.http.grpc_json_transcoder.v3.GrpcJsonTranscoder"
            proto_descriptor_bin: ...
            services:
              - ProdService
            print_options:
              add_whitespace: true
              always_print_primitive_fields: true
              always_print_enums_as_ints: false
              preserve_proto_field_names: false
```

### 通过 HTTP 访问服务

```go
// 根据 ca 和证书获取 tlsConfig 配置对象
func getTLSConfig() *tls.Config {
	cert, err := tls.LoadX509KeyPair("tools/out/clientgrpc.crt", "tools/out/clientgrpc.key")
	if err != nil {
		log.Fatal(err)
	}
	certPool := x509.NewCertPool()
	ca, err := os.ReadFile("tools/out/virtuallainCA.crt")
	if err != nil {
		log.Fatal(err)
	}
	certPool.AppendCertsFromPEM(ca)

	return &tls.Config{
		Certificates: []tls.Certificate{cert},
		ServerName:   "grpc.virtuallain.com",
		RootCAs:      certPool,
	}
}

func main() {
	req, _ := http.NewRequest("POST", "https://grpc.virtuallain.com:30090/detail", strings.NewReader(`{"prod_id":101}`))
	tr := &http.Transport{
		TLSClientConfig: getTLSConfig(),
	}
	client := http.Client{
		Transport: tr,
	}
	rsp, _ := client.Do(req)
	fmt.Println(rsp.Header)
	defer rsp.Body.Close()
	b, _ := io.ReadAll(rsp.Body)
	fmt.Println(string(b))
}
```



## Envoy 限流过滤器

Envoy 支持两种速率限制：全局和本地。本地限流是在envoy内部提供一种令牌桶限速的功能，全局限流需要访问外部限速服务。下面是一个使用全局限流的示例

### 基本配置

**1. 限流配置**

这个 ConfigMap 是限速服务用到的配置文件，在 EnvoyFilter 中被引用。这里配置了 /prods 每分钟限流3个请求，其他 url 限流每分钟100个请求

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ratelimit-config
data:
  config.yaml: |
    domain: prod-ratelimit
    descriptors:
      - key: PATH
        value: "/prods"
        rate_limit:
          unit: minute
          requests_per_unit: 3
      - key: PATH
        rate_limit:
          unit: minute
          requests_per_unit: 100
```

**2. 独立限流服务**

参考 [官方 rate-limit-service.yaml](https://github.com/istio/istio/blob/release-1.9/samples/ratelimit/rate-limit-service.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  ports:
    - name: redis
      port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - image: redis:alpine
          imagePullPolicy: Always
          name: redis
          ports:
            - name: redis
              containerPort: 6379
      restartPolicy: Always
      serviceAccountName: ""
---
apiVersion: v1
kind: Service
metadata:
  name: ratelimit
  labels:
    app: ratelimit
spec:
  ports:
    - name: http-port
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: grpc-port
      port: 8081
      targetPort: 8081
      protocol: TCP
    - name: http-debug
      port: 6070
      targetPort: 6070
      protocol: TCP
  selector:
    app: ratelimit
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ratelimit
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ratelimit
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: ratelimit
    spec:
      containers:
        - image: envoyproxy/ratelimit:6f5de117 # 2021/01/08
          imagePullPolicy: Always
          name: ratelimit
          command: ["/bin/ratelimit"]
          env:
            - name: LOG_LEVEL
              value: debug
            - name: REDIS_SOCKET_TYPE
              value: tcp
            - name: REDIS_URL
              value: redis:6379
            - name: USE_STATSD
              value: "false"
            - name: RUNTIME_ROOT
              value: /data
            - name: RUNTIME_SUBDIRECTORY
              value: ratelimit
          ports:
            - containerPort: 8080
            - containerPort: 8081
            - containerPort: 6070
          volumeMounts:
            - name: config-volume
              mountPath: /data/ratelimit/config/config.yaml
              subPath: config.yaml
      volumes:
        - name: config-volume
          configMap:
            name: ratelimit-config
```

**3. 创建 EnvoyFilter**

这个 EnvoyFilter 作用在网关上，配置了 http 过滤器 envoy.filters.http.ratelimit，和一个 cluster。http 过滤器的 cluster 地址指向 cluster 配置的地址，就是 ratelimit service 所在的地址。domain 和步骤1中 configmap 的值一致，failure_mode_deny 表示超过请求限值就拒绝，rate_limit_service 配置 ratelimit 服务的地址（cluster），可以配置 grpc 类型或 http 类型

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-ratelimit
  namespace: istio-system
spec:
  workloadSelector:
    # select by label in the same namespace
    labels:
      istio: ingressgateway
  configPatches:
    # The Envoy config you want to modify
    - applyTo: HTTP_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
              subFilter:
                name: "envoy.filters.http.router"
      patch:
        operation: INSERT_BEFORE
        # Adds the Envoy Rate Limit Filter in HTTP filter chain.
        value:
          name: envoy.filters.http.ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
            # domain can be anything! Match it to the ratelimter service config
            domain: prod-ratelimit
            failure_mode_deny: true
            rate_limit_service:
              grpc_service:
                envoy_grpc:
                  cluster_name: rate_limit_cluster
                timeout: 10s
              transport_api_version: V3
    - applyTo: CLUSTER
      match:
        cluster:
          service: ratelimit.default.svc.cluster.local
      patch:
        operation: ADD
        # Adds the rate limit service cluster for rate limit service defined in step 1.
        value:
          name: rate_limit_cluster
          type: STRICT_DNS
          connect_timeout: 10s
          lb_policy: ROUND_ROBIN
          http2_protocol_options: {}
          load_assignment:
            cluster_name: rate_limit_cluster
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: ratelimit.default.svc.cluster.local
                          port_value: 8081
```

**4. 创建 Action EnvoyFilter**

这个 EnvoyFilter 作用在入口网关处，给80端口的虚拟主机配置了一个 rate_limits 动作，descriptor_key 用于选择在 configmap 里配置的 key

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-ratelimit-svc
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: VIRTUAL_HOST
      match:
        context: GATEWAY
        routeConfiguration:
          vhost:
            name: "p.virtuallain.com:80"
            route:
              action: ANY
      patch:
        operation: MERGE
        # Applies the rate limit rules.
        value:
          rate_limits:
            - actions:
              - request_headers:
                  header_name: :path  # 内置的头匹配器，有 :path :method
                  descriptor_key: "PATH"
```

### 使用 header_value_match

参考 [文档](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#config-route-v3-ratelimit-action-headervaluematch)

**修改 Action EnvoyFilter**

下面第一个 action 配置了 /prods/\d+ 路由规则的匹配。第二个 action 配置了存在头 version=v2 的匹配

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: filter-ratelimit-svc
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: VIRTUAL_HOST
      match:
        context: GATEWAY
        routeConfiguration:
          vhost:
            name: "p.virtuallain.com:80"
            route:
              action: ANY
      patch:
        operation: MERGE
        # Applies the rate limit rules.
        value:
          rate_limits:
            - actions:
                - header_value_match:
                    descriptor_value: path
                    headers:
                      - name: :path
                        # exact_match: /prods
                        safe_regex_match: # 正则匹配
                          google_re2: {}
                          regex: /prods/\d+
             - actions:
                - header_value_match:
                    descriptor_value: version-v2
                    headers:
                      - name: version
                        exact_match: v2
```

基本的匹配方式有：

- exact_match：精确匹配
- safe_regex_match：正则匹配
- range_match：范围匹配（数字范围，如[-10,0)）
- prefix_match：前缀匹配
- suffix_match：后缀匹配
- contains_match：包含匹配
- invert_match：反向匹配

**修改 ConfigMap 配置**

下面配置了 /prods/\d+ 的路由每分钟限流5次，当存在 header 头 version=v2 时每分钟限流2次。同时匹配到多个规则时优先生效次数少的规则。value 关联 header_value_match 里的 descriptor_value

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ratelimit-config
data:
  config.yaml: |
    domain: prod-ratelimit
    descriptors:
      - key: header_match
        value: path
        rate_limit:
          requests_per_unit: 5
          unit: minute
      - key: header_match
        value: version-v2
        rate_limit:
          requests_per_unit: 2
          unit: minute
```

### IP 限流

**修改 Action EnvoyFilter**

```yaml
rate_limits:
  - actions:
      - remote_address: {}
```

**修改 ConfigMap 配置**

```yaml
data:
  config.yaml: |
    domain: prod-ratelimit
    descriptors:
      - key: remote_address
        rate_limit:
          requests_per_unit: 10
          unit: minute
```

**X-Forwarded-For 配置** 

当存在多个受信任代理的环境中，需要配置生效的 XFF 是第几个，参考 [文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-forwarded-for)。实际运行可以用 nginx-ingress 来反代 istio 的 gateway 从而自动传递这个头

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: xff-trust-hops
  namespace: istio-system
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: NETWORK_FILTER
      match:
        context: ANY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: MERGE
        value:
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
            use_remote_address: true
            xff_num_trusted_hops: 1 # Change as needed
```

### IP 组合条件限流

**修改 Action EnvoyFilter**

```yaml
rate_limits:
  - actions:
      - header_value_match:
          descriptor_value: path
          headers:
            - name: :path
              safe_regex_match:
                google_re2: {}
                regex: /prods/\d+
      - remote_address: {}
```

**修改 ConfigMap 配置**

下面配置了每个 ip 在 /prods/\d+ 的路由每分钟限流5次

```yaml
data:
  config.yaml: |
    domain: prod-ratelimit
    descriptors:
      - key: header_match
        value: path
        descriptors:
          - key: remote_address
            rate_limit:
              requests_per_unit: 5
              unit: minute
```

### 自定义限流服务

上面使用的是官方 [envoyproxy/ratelimit](https://github.com/envoyproxy/ratelimit) 限流服务，也可以自己实现一个。示例：

```go
package main

import (
	"context"
	pb "github.com/envoyproxy/go-control-plane/envoy/service/ratelimit/v3"
	"google.golang.org/grpc"
	"log"
	"net"
	"time"
)

type MyServer struct{}

func NewMyServer() *MyServer {
	return &MyServer{}
}

// 实现限流方法
func (s *MyServer) ShouldRateLimit(ctx context.Context, request *pb.RateLimitRequest) (*pb.RateLimitResponse, error) {
	var overallCode pb.RateLimitResponse_Code
	if time.Now().Unix()%2 == 0 {
		log.Println("限流了")
		overallCode = pb.RateLimitResponse_OVER_LIMIT
	} else {
		log.Println("通过了")
		overallCode = pb.RateLimitResponse_OK
	}
	response := &pb.RateLimitResponse{OverallCode: overallCode}
	return response, nil
}

func main() {
	lis, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	pb.RegisterRateLimitServiceServer(s, NewMyServer())
	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```
