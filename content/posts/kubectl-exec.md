---
title: "kubectl exec 原理"
date: 2023-08-03
draft: false
Categories: [kubernetes]
---

执行 `kubectl exec` 时，首先 kubectl 会向 apiserver 发起 GET 和 POST 请求，apiserver 返回 `101 Upgrade` 响应，向客户端表示已经升级到 SPDY 协议

随后由 apiserver 向 pod 所在节点上的 kubelet 发送流式请求 /exec。kubelet 接到 apiserver 的请求后，通过 CRI 接口向 CRI Shim 请求 exec 的 URL，CRI Shim 向 Kubelet 返回 exec URL，kubelet 再将这个 URL 以 `Redirect` 的方式返回给 apiserver

最后 apiserver 重定向流式请求到对应 Streaming Server 上发起的 exec 请求，并维护长链

![kubectl](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/202308031440678.png)

整个过程，由 apiserver 代理 kubelet，kubelet 再代理 container，完成了 kubectl 和容器的交互，最终的响应都是由容器执行的

## 源码实现

kubelet 的源码阉割模拟

```go
func main() {
	container := restful.NewContainer()
	ws := new(restful.WebService)
	ws.Path("/exec")
	{
		ws.Route(ws.GET("/{podNamespace}/{podID}/{containerName}").
			To(GetExec).
			Operation("getExec"))
		ws.Route(ws.POST("/{podNamespace}/{podID}/{containerName}").
			To(GetExec).
			Operation("getExec"))
		ws.Route(ws.GET("/{podNamespace}/{podID}/{uid}/{containerName}").
			To(GetExec).
			Operation("getExec"))
		ws.Route(ws.POST("/{podNamespace}/{podID}/{uid}/{containerName}").
			To(GetExec).
			Operation("getExec"))
	}
	container.Add(ws)

	klog.Info("启动http服务，监听9090端口")
	http.ListenAndServe(":9090", container)
}

func GetExec(request *restful.Request, response *restful.Response) {
	url, err := GetUrl()
	if err != nil {
		streaming.WriteError(err, response.ResponseWriter)
		return
	}
	proxyStream(response.ResponseWriter, request.Request, url)
}

func GetUrl() (*url.URL, error) {
	req := &runtimeapi.ExecRequest{
		ContainerId: ContainerId,
		Cmd:         []string{"ls"},
		Tty:         false,
		Stdin:       true,
		Stdout:      true,
		Stderr:      true,
	}
	resp, err := runtimeExec(req)
	if err != nil {
		return nil, err
	}

	// 需要修改containerd配置：vi /etc/containerd/config.toml
	// stream_server_address改为0.0.0.0 (默认127.0.0.1 只能本地访问)
	// stream_server_port改为一个固定的端口如6595 (默认0 代表随机生成端口)

	// 这个地址是容器运行时生成的，每次都不一样，他会启动时监听一个地址用于给我们exec
	klog.Info("得到的URL是：", resp.Url)
	resp.Url = strings.Replace(resp.Url, "[::]", RemoteRuntimeIp, -1)
	klog.Info("修改过后的URL是：", resp.Url)
	return url.Parse(resp.Url)
}

const RemoteRuntimeAddress = "" // 远程的containerd地址
const RemoteRuntimeIp = ""
const ContainerId = "" // 容器id kubectl get pod -o yaml查看

func initRuntimeClient() runtimeapi.RuntimeServiceClient {
	gopts := []grpc.DialOption{
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)
	defer cancel()
	conn, err := grpc.DialContext(ctx, RemoteRuntimeAddress, gopts...)
	if err != nil {
		log.Fatalln(err)
	}
	return runtimeapi.NewRuntimeServiceClient(conn)
}

func runtimeExec(req *runtimeapi.ExecRequest) (*runtimeapi.ExecResponse, error) {
	runtimeClient := initRuntimeClient()
	ctx, cancel := context.WithTimeout(context.TODO(), time.Second*5)
	defer cancel()

  // 发起grpc调用让运行时准备一个流式通信的端点，该端点用于在容器执行命令	
	resp, err := runtimeClient.Exec(ctx, req)
	if err != nil {
		klog.ErrorS(err, "Exec cmd from runtime service failed", "containerID", req.ContainerId, "cmd", req.Cmd)
		return nil, err
	}
	klog.V(10).InfoS("[RemoteRuntimeService] Exec Response")

	if resp.Url == "" {
		errorMessage := "URL is not set"
		err := errors.New(errorMessage)
		klog.ErrorS(err, "Exec failed")
		return nil, err
	}

	return resp, nil
}

type responder struct{}

func (r *responder) Error(w http.ResponseWriter, req *http.Request, err error) {
	klog.ErrorS(err, "Error while proxying request")
	http.Error(w, err.Error(), http.StatusInternalServerError)
}

func proxyStream(w http.ResponseWriter, r *http.Request, url *url.URL) {
	handler := proxy.NewUpgradeAwareHandler(url, nil /*transport*/, false /*wrapTransport*/, true /*upgradeRequired*/, &responder{})
	handler.ServeHTTP(w, r)
}
```

模拟客户端直接请求 kubelet

```go
func main() {
	execUrl := "http://localhost:9090/exec/default/mypod/mycontainer"
	req, _ := http.NewRequest("GET", execUrl, nil)
	req.Header.Set("Upgrade", "SPDY/3.1")
	req.Header.Set("Connection", "Upgrade")
	tlsConfig := &tls.Config{
		InsecureSkipVerify: true,
	}
	rt := spdy.NewRoundTripper(tlsConfig, true, false)

	executor, err := remotecommand.NewSPDYExecutorForTransports(rt, rt, http.MethodGet, req.URL)
	if err != nil {
		log.Fatal(err)
	}
	err = executor.Stream(remotecommand.StreamOptions{
		Stdin:  os.Stdin,
		Stdout: os.Stdout,
		Stderr: os.Stderr,
		Tty:    true,
	})
	if err != nil {
		log.Fatalln(err)
	}
}
```

apiserver 的源码阉割模拟

```go
func main() {
	urlStr := "http://localhost:9090/exec/default/mypod/mycontainer"
	urlObj, err := url.Parse(urlStr)
	if err != nil {
		panic(err)
	}
	proxyHandler := proxy.NewUpgradeAwareHandler(urlObj, http.DefaultTransport, false, true, proxy.NewErrorResponder(nil))

	log.Println("启动假的apiserver")
	http.ListenAndServe(":6443", proxyHandler)
}

// 创建一个代理的handler，代理kubelet
func newThrottledUpgradeAwareProxyHandler(location *url.URL, transport http.RoundTripper, wrapTransport, upgradeRequired, interceptRedirects bool, responder rest.Responder) *proxy.UpgradeAwareHandler {
	handler := proxy.NewUpgradeAwareHandler(location, transport, wrapTransport, upgradeRequired, proxy.NewErrorResponder(responder))
	handler.InterceptRedirects = interceptRedirects && utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StreamingProxyRedirects)
	handler.RequireSameHostRedirects = utilfeature.DefaultFeatureGate.Enabled(genericfeatures.ValidateProxyRedirects)
	handler.MaxBytesPerSec = capabilities.Get().PerConnectionBandwidthLimitBytesPerSec
	return handler
}
```

修改模拟客户端的 execUrl

```go
execUrl := "http://localhost:6443"
```
