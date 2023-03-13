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
  -subj "/C=cn/OU=virtuallain/O=virtuallain/CN=grpc.jtthink.com" \
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
