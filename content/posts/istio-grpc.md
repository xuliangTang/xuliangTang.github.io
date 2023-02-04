---
title: "Istio 部署 gRPC 服务并配置网关证书"
date: 2023-02-04
draft: false
Categories: [Istio]
---

## gRPC 环境

gRPC 是 Google公司基于 Protobuf 开发的跨语言的开源 RPC 框架。gRPC 基于 HTTP/2 协议设计，可以基于一个HTTP/2链接提供多个服务

### 安装

**1. protobuf**

从 [protobuf](https://github.com/protocolbuffers/protobuf/releases/tag/v3.20.1) 这里下载，把 bin 目录加入环境变量

**2. go protobuf 库**

```bash
go get -u github.com/golang/protobuf@v1.5.2
```

**3. go 插件**

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
```

该插件会根据 `.proto` 文件生成一个后缀为 `.pb.go` 的文件，包含所有 `.proto` 文件中定义的类型及其序列化方法

```bash
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

该插件会生成一个后缀为 `_grpc.pb.go` 的文件，其中包含：

- 一种接口类型(或存根) ，供客户端调用的服务方法。
- 服务器要实现的接口类型。

上述命令会默认将插件安装到 `$GOPATH/bin`，为了 `protoc` 编译器能找到这些插件，请确保你的`$GOPATH/bin`在环境变量中

**4. grpc 库**

```bash
go get -u google.golang.org/grpc@v1.46.2
```

### 编写 .proto 定义服务

src/prod_model.proto:

```protobuf
syntax = "proto3";
option go_package = "src/pbfiles";

message ProdRequest {
    int32 prod_id =1;
}
message ProdModel {
    int32 id=1;
    string name=2;
}
message ProdResponse {
      ProdModel result=1;
}
```

src/prod_service.proto:

```protobuf
syntax = "proto3";
import "prod_model.proto";
option go_package = "src/pbfiles";

service ProdService {
  rpc GetProd(ProdRequest) returns (ProdResponse);
}
```

在项目更目录执行以下命令，根据 proto 生成 go 源码文件

```bash
protoc320 --proto_path=src --go_out=./ prod_model.proto
protoc320 --proto_path=src --go-grpc_out=./ prod_service.proto
```

编写 server 端：

```go
type ProdService struct {
	pbfiles.UnimplementedProdServiceServer
}

func NewProdService() *ProdService {
	return &ProdService{}
}

func (this *ProdService) GetProd(ctx context.Context, req *pbfiles.ProdRequest) (*pbfiles.ProdResponse, error) {
	model := &pbfiles.ProdModel{
		Id:   req.ProdId,
		Name: fmt.Sprintf("%s%d", "测试商品", req.ProdId),
	}

	rsp := &pbfiles.ProdResponse{Result: model}

	return rsp, nil
}

func main() {
	myserver := grpc.NewServer()
	// 创建服务
	pbfiles.RegisterProdServiceServer(myserver, services.NewProdService())
	// 监听8080
	lis, _ := net.Listen("tcp", ":8080")
	if err := myserver.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```

### 测试客户端

```go
func main() {
	client, err := grpc.DialContext(context.Background(), ":8080", grpc.WithInsecure())
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



## 部署网格内 gRPC 服务

示例如下

### 创建 gRPC Gate 网关

demo.yaml

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    egressGateways:
      - enabled: true
        name: istio-egressgateway
    ingressGateways:
      - enabled: true
        name: istio-ingressgateway
      - enabled: true
        label:
          app: grpc-ingressgateway	# 自定义标签
          istio: grpc-ingressgateway
        k8s:
          resources:
            requests:
              cpu: 10m
              memory: 40Mi
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: http2
                port: 80
                targetPort: 8080
              - name: https
                port: 443
                targetPort: 8443
              - name: tcp
                port: 31400
                targetPort: 31400
              - name: tls
                port: 15443
                targetPort: 15443
        name: grpc-ingressgateway
```

执行部署：

```bash
istioctl install -f demo.yaml
```

### 创建网关规则

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-gateway
  namespace: myistio
spec:
  selector:
    istio: grpc-ingressgateway
  servers:
    - port:
        number: 80
        name: grpc
        protocol: HTTPS
      hosts:
        - "*"
```

### 创建虚拟服务

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpcvs
  namespace: myistio
spec:
  hosts:
    - "*"
  gateways:
    - grpc-gateway
  http:
    -  route:
       - destination:
          host: gprodsvc
          port:
            number: 80
```



## 配置网关证书

### 配置方式

**1. 文件挂载的方式**

Istio 网关将会自动加载 istio-system 命名空间下名称为 `stio-ingressgateway-certs` 的 secret，并分别挂载到  `/etc/istio/ingressgateway-certs/tls.crt` 和 ` /etc/istio/ingressgateway-certs/tls.key`。指定 `serverCertificate` 和 `privateKey` 字段

**2. 指定密钥的方式**

指定 `credentialName` 字段

### 配置 HTTP TLS 证书

```bash
kubectl create -n istio-system secret tls istio-ingressgateway-certs --key api.virtuallain.com.key --cert api.virtuallain.com.crt
```

Gateway 加入 tls 节点：

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
        - api.virtuallain.com
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        serverCertificate: /etc/istio/ingressgateway-certs/tls.crt # 使用挂载证书文件
        privateKey: /etc/istio/ingressgateway-certs/tls.key
        # credentialName: ssl-ingressgateway-certs
      hosts:
        - api.virtuallain.com
```

### 配置 gRPC 证书（单向认证）

使用 [certstrap](https://github.com/square/certstrap/releases/tag/v1.2.0) 开源库生成证书

**1. 自签 CA 证书**

```bash
./cert init --common-name "virtuallainCA" --expires "20 years"
```

**2. 服务端证书**

```bash
# 得到证书请求文件
./cert request-cert -cn grpc.virtuallain.com  -domain "*.virtuallain.com"
# CA去签名请求文件
./cert sign grpc.virtuallain.com --CA virtuallainCA
```

**3. 导入 k8s**

```bash
kubectl create -n istio-system secret tls grpc-ingressgateway-certs --key=grpc.virtuallain.com.key  --cert=grpc.virtuallain.com.crt

# 效果同上
kubectl create -n istio-system secret generic grpc-ingressgateway-certs --from-file=key=grpc.virtuallain.com.key --from-file=cert=grpc.virtuallain.com.crt
```

**4. 配置 Gateway**

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-gateway
  namespace: myistio
spec:
  selector:
    istio: grpc-ingressgateway
  servers:
    - port:
        number: 80
        name: grpc
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: grpc-ingressgateway-certs
      hosts:
        - "*"
```

**5. 客户端测试**

```go
func main() {
	creds, err := credentials.NewClientTLSFromFile("tools/out/grpc.virtuallain.com.crt",
		"grpc.virtuallain.com")
	if err != nil {
		log.Fatal(err)
	}
	client, err := grpc.DialContext(context.Background(),
		"grpc.virtuallain.com:30090",
		grpc.WithTransportCredentials(creds))
	if err != nil {
		log.Fatal(err)
	}

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

### 配置 gRPC 证书（双向认证）

**1. 同步生成客户端证书**

```bash
./cert request-cert -cn clientgrpc 
./cert sign clientgrpc --CA virtuallainCA
```

**2. 重新导入证书**

```bash
kubectl create -n istio-system secret generic grpc-ingressgateway-certs --from-file=key=grpc.virtuallain.com.key --from-file=cert=grpc.virtuallain.com.crt --from-file=cacert=virtuallainCA.crt
```

**3. 配置 Gateway**

修改 mode 为 `MUTUAL`

```yaml
tls:
  mode: MUTUAL
  credentialName: grpc-ingressgateway-certs
```

**4. 客户端测试**

```go
func main() {
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
	creds := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{cert},
		ServerName:   "grpc.virtuallain.com",
		RootCAs:      certPool,
	})
	client, err := grpc.DialContext(context.Background(),
		"grpc.virtuallain.com:30090",
		grpc.WithTransportCredentials(creds))

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
