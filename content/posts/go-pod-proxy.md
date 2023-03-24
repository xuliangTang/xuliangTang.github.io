---
title: "使用正向代理远程访问 k8s 服务"
date: 2023-03-24
draft: false
Categories: [go]
---

## HTTP

原理：部署一个僵尸 Pod 正向代理 k8s 服务，通过暴露 NodePort 对外提供访问

```go
func getRsp(request *http.Request) (*http.Response, error) {
	transport := http.DefaultTransport
	outReq := new(http.Request)
	*outReq = *request
	// 构建roundTrip
	rsp, err := transport.RoundTrip(outReq)
	if err != nil {
		return nil, err
	}
	return rsp, nil
}

func copyHeader(writer http.ResponseWriter, header map[string][]string) {
	for key, value := range header {
		for _, v := range value {
			writer.Header().Add(key, v)
		}
	}
}
func copyRspBody(writer http.ResponseWriter, rsp *http.Response) {
	// 写入http statusCode 状态码
	writer.WriteHeader(rsp.StatusCode)
	io.Copy(writer, rsp.Body) // 写入body到writer中
}

func main() {
	http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		log.Println(fmt.Sprintf("收到请求: method:%s "+
			"Host:%s Path:%s", request.Method, request.Host, request.URL.Path))
		// 第一步 获取响应
		rsp, err := getRsp(request)
		if err != nil {
			log.Println(err)
			writer.WriteHeader(http.StatusBadGateway)
			return
		}
		defer rsp.Body.Close()
		// 拷贝响应头
		copyHeader(writer, rsp.Header)
		// 拷贝响应内容
		copyRspBody(writer, rsp)
	})

	http.ListenAndServe(":8080", nil)
}
```

远程访问时携带 proxy 配置去请求，由正向代理去访问真实的服务并将结果返回

```go
func main() {
	proxy, _ := url.Parse("http://ip:NodePort")

	req, err := http.NewRequest("GET", "http://serviceName", nil)
	if err != nil {
		log.Fatal(err)
	}
	
	client := &http.Client{
		Transport: &http.Transport{
			Proxy: http.ProxyURL(proxy),	// 代理
		},
		Timeout: time.Second * 3,
	}
	
	rsp, err := client.Do(req)
	if err != nil {
		log.Fatal(err)
	}
	defer rsp.Body.Close()
	b, _ := io.ReadAll(rsp.Body)
	
	fmt.Println(string(b))
}
```



## TCP

使用 socks5 协议，客户端连代理服务并发送 socks 协议版本以及认证方式，完成客户端和服务端的安全认证，代理服务器访问目标服务并转发流量给客户端，实现了 socks5 的第三方库：[go-socks5](https://github.com/armon/go-socks5)

```go
func main() {
	conf := &socks5.Config{}
	server, err := socks5.New(conf)
	if err != nil {
		panic(err)
	}
	if err := server.ListenAndServe("tcp", "0.0.0.0:9000"); err != nil {
		panic(err)
	}
}
```

客户端连接 redis 示例：

```go
func main() {
    // 使用SOCKS5创建拨号器来进行代理连接
	dialer, err := proxy.SOCKS5("tcp", "ip:NodePort", nil, proxy.Direct)
	if err != nil {
		fmt.Fprintln(os.Stderr, "proxy connection error:", err)
		os.Exit(1)
	}

	rdb := redis.NewClient(&redis.Options{
		Addr:     "redis:6379",
		Password: "",
		DB:       0,
		Dialer: func(ctx context.Context, network, addr string) (net.Conn, error) {
			return dialer.Dial(network, addr)
		},
	})

	ctx := context.Background()
	rdb.Set(ctx, "name", "txl", time.Second*10)
	fmt.Println(rdb.Get(ctx, "name"))
}
```
