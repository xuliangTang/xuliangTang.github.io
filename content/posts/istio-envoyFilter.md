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
