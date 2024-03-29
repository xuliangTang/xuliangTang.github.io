---
title: "GO 使用 gRPC"
date: 2023-03-12
draft: false
Categories: [go]
---

gRPC 是一种现代化开源的高性能RPC框架，能够运行于任意环境之中。最初由谷歌进行开发。它使用HTTP/2作为传输协议

在 gRPC 里，客户端可以像调用本地方法一样直接调用其他机器上的服务端应用程序的方法，帮助你更容易创建分布式应用程序和服务。与许多 RPC 系统一样，gRPC 是基于定义一个服务，指定一个可以远程调用的带有参数和返回类型的的方法。在服务端程序中实现这个接口并且运行 gRPC 服务处理客户端调用。在客户端，有一个 stub 提供和服务端相同的方法

使用gRPC， 我们可以一次性的在一个`.proto`文件中定义服务并使用任何支持它的语言去实现客户端和服务端，反过来，它们可以应用在各种场景中，从 Google 的服务器到你自己的平板电脑—— gRPC 帮你解决了不同语言及环境间通信的复杂性。使用`protocol buffers`还能获得其他好处，包括高效的序列化，简单的 IDL 以及容易进行接口更新。总之一句话，使用 gRPC 能让我们更容易编写跨语言的分布式代码 

## 证书认证

gRPC 建立在 HTTP/2 协议之上，对 TLS 提供了很好的支持。没有提供证书支持客户端在连接服务器中通过 `grpc.WithInsecure()` 选项跳过了对服务器证书的验证。没有启用证书的 gRPC 服务在和客户端进行的是明文通讯，信息面临被任何第三方监听的风险。为了保障 gRPC 通信不被第三方监听篡改或伪造，我们可以对服务器启动 TLS 加密特性

> Go1.15 之后改用 SAN 证书，SAN(Subject Alternative Name) 是 SSL 标准 x509 中定义的一个扩展。使用了 SAN 字段的 SSL 证书，可以扩展此证书支持的域名，使得一个证书可以支持多个不同域名的解析

生成服务端根证书：

```bash
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt
```

修改 openSSL 配置，yum 安装的默认配置文件在 `/etc/pki/tls/openssl.cnf`，拷贝到当前目录

```bash
cp /etc/pki/tls/openssl.cnf .
```

在 `[ req ]` 节点下加入 `req_extetions = v3_req`

在 `[ v3_req ]` 节点下加入 `subjectAltName = @alt_names`

加入 `[ alt_names ]` 节点，加入内容 `[ DNS.1 = grpc.virtuallain.com ]`

生成私钥：

```bash
openssl genpkey -algorithm RSA -out server.key
```

生成证书请求文件：

```bash
openssl req -new -nodes -key server.key -out server.csr -days 3650 \
  -subj "/C=cn/OU=virtuallain/O=virtuallain/CN=grpc.virtuallain.com" \
  -config ./openssl.cnf -extensions v3_req
  
# 查看
openssl req -noout -text -in server.csr
```

签发证书：

```bash
openssl x509 -req -days 3650 -in server.csr -out server.pem \
  -CA ./ca.crt -CAkey ./ca.key -CAcreateserial \
  -extfile ./openssl.cnf -extensions v3_req

# 查看
openssl x509 -noout -text -in server.pem
```

### 单向认证

服务端：

```go
func main() {
	creds, err := credentials.NewServerTLSFromFile("certs/server.pem", "certs/server.key")
	if err != nil {
		log.Fatal(err)
	}

	server := grpc.NewServer(grpc.Creds(creds))
	// 创建服务
	pbfiles.RegisterProdServiceServer(server, services.NewProdService())
	// 监听8080
	lis, _ := net.Listen("tcp", ":8080")
	if err := server.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```

客户端：

```go
func main() {
	creds, err := credentials.NewClientTLSFromFile("certs/server.pem", "grpc.virtuallain.com")
	if err != nil {
		log.Fatal(err)
	}
	client, err := grpc.DialContext(context.Background(), ":8080", grpc.WithTransportCredentials(creds))
	rsp := &pbfiles.ProdResponse{}
	err = client.Invoke(context.Background(),
		"/ProdService/GetProd",
		&pbfiles.ProdRequest{ProdId: 123}, rsp)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(rsp.Result)
}
```

### 双向认证

生成私钥：

```bash
openssl genpkey -algorithm RSA -out client.key
```

生成证书请求文件：

```bash
openssl req \
  -new \
  -key client.key \
  -subj '/CN=myclient' \
  -out client.csr 
```

签发证书：

```bash
openssl x509 \
  -req \
  -in client.csr \
  -CA ./ca.crt \
  -CAkey ./ca.key \
  -CAcreateserial \
  -days 3650 \
  -out client.crt
```

客户端：

```go
func main() {
	cert, _ := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
	certPool := x509.NewCertPool()
	ca, _ := os.ReadFile("certs/ca.crt")
	certPool.AppendCertsFromPEM(ca)

	creds := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert}, // 客户端证书
		ServerName:   "grpc.virtuallain.com",
		RootCAs:      certPool,
	})

	client, err := grpc.DialContext(context.Background(), ":8080", grpc.WithTransportCredentials(creds))
	rsp := &pbfiles.ProdResponse{}
	err = client.Invoke(context.Background(),
		"/ProdService/GetProd",
		&pbfiles.ProdRequest{ProdId: 123}, rsp)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(rsp.Result)
}
```



## Descriptor

descriptor 文件是 ProtoBuf 提供的动态解析机制，包含 proto 文件的描述信息：文件名、包名、选项、文件中定义的所有 message、所有 service、 定义的 extension、 依赖文件（import）等

生成：

```bash
protoc320 --proto_path=protos --include_imports --include_source_info --descriptor_set_out=prod.pb service.proto
```



## 字段验证

使用 [protoc-gen-validate](github.com/envoyproxy/protoc-gen-validate) 库，拷贝 Go Modules 下的 validate/validate.proto 到当前目录

```bash
go get -d github.com/envoyproxy/protoc-gen-validate
```

在 proto 中定义验证规则：

```protobuf
syntax = "proto3";
option go_package = "src/pbfiles";
import "validate.proto";

message ProdRequest {
  int32 prod_id=1 [(validate.rules).int32 = {gte: 100}];
}
```

编译需要增加 `--validate_out` 参数：

```bash
protoc320 --proto_path=protos --go_out=./ --validate_out="lang=go:./" models.proto
```

在 service 中验证：

```go
func (p ProdService) GetProd(ctx context.Context, request *pbfiles.ProdRequest) (*pbfiles.ProdResponse, error) {
	if err := request.Validate(); err != nil {
		return nil, err
	}
}
```



## 身份验证

gRPC 定义了 PerRPCCredentials 接口，将认证信息添加到每个 RPC 方法的上下文中，用于自定义认证

实现接口：

```go
type Auth struct {
	Token string
}

func NewAuth(token string) *Auth {
	return &Auth{Token: token}
}

// 获取认证所需元数据
func (this Auth) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
	return map[string]string{
		"token": this.Token,
	}, nil
}

// 是否需要  TLS
func (a Auth) RequireTransportSecurity() bool {
	return true
}

var _ credentials.PerRPCCredentials = &Auth{}
```

修改客户端：

```go
client, err := grpc.DialContext(
	context.Background(),
	":8080",
	grpc.WithTransportCredentials(creds),
	grpc.WithPerRPCCredentials(NewAuth("test-token")),
)
```

服务端获取到元数据后，可以结合 [jwt](https://github.com/dgrijalva/jwt-go) 完成验证

```go
md, ok := metadata.FromIncomingContext(ctx)
if !ok {
	return nil, status.Error(codes.Unauthenticated, "metadata error")
}

fmt.Println("token: ", md.Get("token"))
```

### 服务端拦截器

[UnaryServerInterceptor ](https://godoc.org/google.golang.org/grpc#UnaryServerInterceptor)是服务端的一元拦截器类型，它的函数签名是

```go
func(ctx context.Context, req interface{}, info *UnaryServerInfo, handler UnaryHandler) (resp interface{}, err error)
```

统一拦截验证 token 实现：

```go
func checkToken(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "metadata error")
	}

	token, ok := md["token"]
	if !ok {
		return nil, status.Error(codes.Unauthenticated, "token error")
	}

	fmt.Println("token: ", token)

	return handler(ctx, req)
}
```

服务端配置拦截器：

```go
server := grpc.NewServer(grpc.Creds(creds), grpc.UnaryInterceptor(checkToken))
```



## 权限认证

根据 token 获取角色判断权限最简示例：

```go
var AuthMap map[string]string	// 角色对应的权限

func init() {
	AuthMap = make(map[string]string)
	AuthMap["Admin"] = "/ProdService/GetProd"
}

func RBAC(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	md, _ := metadata.FromIncomingContext(ctx)
	role := md.Get("token")[0]	// 通过jwt解析角色，略
	if _, ok := AuthMap[role]; ok {	// 验证权限
		return handler(ctx, req)
	}
	return nil, status.Errorf(codes.Unauthenticated, "没有权限")
}
```

### 结合 casbin

```ini
// casbin/model.conf
[request_definition]
r = sub, obj

[policy_definition]
p = sub, obj

[role_definition]
g = _, _

[policy_effect]
e = some(where (p.eft == allow))

[matchers]
m = g(r.sub, p.sub) && r.obj == p.obj


// casbin/p.csv
p, member, /ProdService/GetProd
p, admin, /ProdService/UpdateProd

g, admin, member
g, lain, admin
g, zhangsan, member
```

修改服务端

```go
var E *casbin.Enforcer

func init() {
	e, err := casbin.NewEnforcer("casbin/model.conf", "casbin/p.csv")
	if err != nil {
		log.Fatal(err)
	}
	E = e
}

func RBAC(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Errorf(codes.Unavailable, "请求错误")
	}
	tokens := md.Get("token")
	if len(tokens) != 1 {
		return nil, status.Errorf(codes.Unauthenticated, "参数错误")
	}
	b, err := E.Enforce(tokens[0], info.FullMethod)
	if !b || err != nil {
		return nil, status.Errorf(codes.Unauthenticated, "没有权限")
	}
	return handler(ctx, req)
}
```



## gRPC-Gateway 

[gRPC-Gateway](https://github.com/grpc-ecosystem/grpc-gateway) 是 Google protocol buffers compiler(protoc) 的一个插件，读取 protobuf 定义然后生成反向代理服务器，将 RESTful HTTP API 转换为 gRPC

当 HTTP 请求到达 gRPC-Gateway 时，它将 JSON 数据解析为 Protobuf 消息。然后，它使用解析的 Protobuf 消息发出正常的 Go gRPC 客户端请求。Go gRPC 客户端将 Protobuf 结构编码为 Protobuf 二进制格式，然后将其发送到 gRPC 服务器。gRPC 服务器处理请求并以 Protobuf 二进制格式返回响应。Go gRPC 客户端将其解析为 Protobuf 消息，并将其返回到 gRPC-Gateway，后者将 Protobuf 消息编码为 JSON 并将其返回给原始客户端

```bash
# 安装gRPC-Gateway插件
go get github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway
```

### 基本流程

由 protoc 将 .proto 文件编译成 protobuf 格式的数据，将编译后的数据传递到各个插件，生成对应语言、对应模块的源代码：

- Go Plugins 用于生成 .pb.go 文件
- gRPC Plugins 用于生成 _grpc.pb.go
- gRPC-Gateway 则是 pb.gw.go

比如以下命令会同时生成 Go、gRPC 、gRPC-Gateway 需要的 3 个文件：

```bash
protoc --go_out . --go-grpc_out . --grpc-gateway_out . test.proto
```

### 示例

**1. proto 文件增加http相关注解**

需要引入 google/api/annotations.proto 文件，因为添加的注解依赖该文件。可以从 [googleapis/googleapis](https://github.com/googleapis/googleapis) 获取并拷贝到同级目录

```protobuf
syntax = "proto3";
option go_package = "src/pbfiles";
import "models.proto";
import "google/api/annotations.proto";

service ProdService {
  rpc GetProd(ProdRequest) returns (ProdResponse){
    option (google.api.http) = {
      get: "/prod/{prod_id}"
    };
  }
  rpc UpdateProd(ProdRequest) returns (ProdResponse){
    option (google.api.http) = {
      post: "/prod/update"
      body: "*"
    };
  }
}
```

**2. 编译**

```bash
protoc320 --proto_path=protos --go_out=./ --validate_out="lang=go:./" models.proto
protoc320 --proto_path=protos --go_out=./ --go-grpc_out=./ --grpc-gateway_out=./ service.proto
```

**3. http 反向代理**

```go
func run() error {
	ctx := context.Background()
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	cert, _ := tls.LoadX509KeyPair("certs/client.crt", "certs/client.key")
	certPool := x509.NewCertPool()
	ca, _ := os.ReadFile("certs/ca.crt")
	certPool.AppendCertsFromPEM(ca)

	creds := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert}, //客户端证书
		ServerName:   "grpc.virtuallain.com",
		RootCAs:      certPool,
	})
	
	mux := runtime.NewServeMux()
	opts := []grpc.DialOption{grpc.WithTransportCredentials(creds)}
	err := pbfiles.RegisterProdServiceHandlerFromEndpoint(ctx, mux, "localhost:8080", opts)
	if err != nil {
		return err
	}
	
	return http.ListenAndServe(":8081", mux)
}

func main() {
	flag.Parse()
	defer glog.Flush()

	if err := run(); err != nil {
		glog.Fatal(err)
	}
}
```



## gRPC Web

[gRPC Web](https://github.com/grpc/grpc-web) 结合 Envoy（或 [gRPC Web 代理](https://github.com/improbable-eng/grpc-web)）是完成 web 调用 gRPC 的一种方式

首先下载  protoc-gen-grpc-web 可执行程序放入 bin 目录

编译 proto 生成 js 文件到 html 目录下：

```bash
protoc320 --proto_path=protos --js_out=import_style=commonjs:html --grpc-web_out=import_style=commonjs,mode=grpcwebtext:html models.proto
protoc320 --proto_path=protos --js_out=import_style=commonjs:html --grpc-web_out=import_style=commonjs,mode=grpcwebtext:html service.proto
```

安装前端运行时库：

```bash
npm install google-protobuf
npm install grpc-web
```

示例：

```js
import { ProdServiceClient } from '@/grpc/service_grpc_web_pb'
import { ProdRequest } from '@/grpc/models_pb'

const client = new ProdServiceClient('http://localhost:8081'); // 代理地址

const req = new ProdRequest()
req.setProdId("101")
const metadata = {"Content-Type": "application/grpc-web-text"};
client.getProd(req,metadata,(err,rsp)=>{
  if (err) {
    console.log(err.message);
  } else {
    console.log(rsp);
  }
})
```

启动代理：

```bash
grpcwebproxy --backend_addr=localhost:8080 --server_http_debug_port=8081 --allow_all_origins --server_tls_cert_file=./server.pem  --server_tls_key_file=./server.key --backend_client_tls_cert_file=./client.crt --backend_client_tls_key_file=./client.key --backend_tls_ca_files=./ca.crt --backend_tls=true
```
