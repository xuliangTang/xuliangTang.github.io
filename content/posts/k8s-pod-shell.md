---
title: "k8s远程进入容器terminal"
date: 2023-01-05
draft: false
Categories: [k8s-go]
---

K8s 实现的“进入某个容器”的功能，底层本质是 Docker 容器通过 exec 进入容器的扩展。本质是新建了一个“与目标容器，共享 namespace 的”新的 shell 进程。所以该 shell 进程，看到的世界，就是容器内的世界了。

通过 client-go 提供的方法，实现通过网页进入 kubernetes 任意容器的终端操作

## remotecommand

`http://k8s.io/client-go/tools/remotecommand` 是 kubernetes client-go 提供的 remotecommand 包，提供了方法与集群中的容器建立长连接，并设置容器的 stdin，stdout 等。

remotecommand 包提供基于 [SPDY](https://en.wikipedia.org/wiki/SPDY) 协议的 Executor interface，进行和 pod 终端的流的传输。初始化一个 Executor 很简单，只需要调用 remotecommand 的 NewSPDYExecutor 并传入对应参数。

```go
func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "kubeconfig")
	if err != nil {
		log.Fatal(err)
	}

	client, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatal(err)
	}

	option := &coreV1.PodExecOptions{
		Container: "nginxtest",		// 容器名称
		Command:   []string{"sh", "-c", "ls"},	// 命令
		Stdin:     true,
		Stdout:    true,
		Stderr:    true,
	}

	req := client.CoreV1().RESTClient().Post().Resource("pods").
		Namespace("default").
		Name("myngx-79bdb4ccf8-nbln7").	// pod名称
		SubResource("exec").
		VersionedParams(option, scheme.ParameterCodec)
	
    // 这里初始化了一个 remote-cmd 的对象
	exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
	if err != nil {
		log.Fatal(err)
	}
	
    // 这里开始，将输入输出，进行实时传递（Stream）
	err = exec.StreamWithContext(context.Background(), remotecommand.StreamOptions{
		Stdin:  os.Stdin,
		Stdout: os.Stdout,
		Stderr: os.Stderr,
		Tty:    true,
	})
	if err != nil {
		log.Fatal(err)
	}
}
```

将 `TTY` 设置为 true，命令设置为 `sh` 进入容器交互式执行

```go
option := &coreV1.PodExecOptions{
		Container: "nginxtest",
		Command:   []string{"sh"},
		Stdin:     true,
		Stdout:    true,
		Stderr:    true,
		TTY:       true,
	}
```



## websocket

Executor 的 StreamWithContext 方法，会建立一个流传输的连接，直到服务端和调用端一端关闭连接，才会停止传输。常用的做法是定义一个你想用的客户端，实现 `Read(p []byte) (int, error) `和 `Write(p []byte) (int, error)` 方法即可，调用 Stream 方法时，只要将 StreamOptions 的 Stdin Stdout 都设置为该客户端，Executor 就会通过你定义的 write 和 read 方法来传输数据。

```go
var Upgrader websocket.Upgrader

func init() {
	Upgrader = websocket.Upgrader{
		CheckOrigin: func(r *http.Request) bool {
			return true
		},
	}
}

type WsShellClient struct {
	client *websocket.Conn
}

func NewWsShellClient(client *websocket.Conn) *WsShellClient {
	return &WsShellClient{client: client}
}

// 实现 io.Writer
func (this *WsShellClient) Write(p []byte) (n int, err error) {
	err = this.client.WriteMessage(websocket.TextMessage, p)
	if err != nil {
		return 0, err
	}
	return len(p), nil
}

// 实现 io.Reader
func (this *WsShellClient) Read(p []byte) (n int, err error) {
	_, b, err := this.client.ReadMessage()

	if err != nil {
		return 0, err
	}
	return copy(p, string(b)+"\n"), nil
}

```

```go
func main() {
	r := gin.New()
	r.GET("/", func(c *gin.Context) {
		wsClient, err := ws.Upgrader.Upgrade(c.Writer, c.Request, nil)
		if err != nil {
			log.Println(err)
			return
		}

		shellClient := ws.NewWsShellClient(wsClient)

		option := &coreV1.PodExecOptions{
			Container: "nginxtest",
			Command:   []string{"sh"},
			Stdin:     true,
			Stdout:    true,
			Stderr:    true,
			TTY:       true,
		}
		req := client.CoreV1().RESTClient().Post().Resource("pods").
			Namespace("default").
			Name("myngx-79bdb4ccf8-nbln7").
			SubResource("exec").
			VersionedParams(option, scheme.ParameterCodec)

		exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
		if err != nil {
			log.Println(err)
		}

		err = exec.StreamWithContext(c, remotecommand.StreamOptions{
			Stdin:  shellClient,
			Stdout: shellClient,
			Stderr: shellClient,
			Tty:    true,
		})
		if err != nil {
			log.Println(err)
		}
	})
	r.Run(":8080")
}
```

测试html示例

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>

</head>
<body>
<div>
    <div id="message" style="width: 500px;height:300px;border:solid 1px gray;overflow:auto">

    </div>
    <div>
        <input type="type" id="txtCmd"/>
        <input type="button" id="cmdBtn" value="发送"/>
        <input type="button" onclick="document.getElementById('message').innerHTML=''" value="清空"/>
    </div>
</div>
<script>
    var ws = new WebSocket("ws://localhost:8080/");
    ws.onopen = function(){
        console.log("open");
    }
    ws.onmessage = function(e){
         let html=document.getElementById("message").innerHTML;
        html+='<p>服务端消息:' + e.data + '</p>'
        document.getElementById("message").innerHTML=html
    }
    ws.onclose = function(e){
        console.log("close");
    }
    ws.onerror = function(e){
        console.log(e);
    }
    document.getElementById("cmdBtn").onclick= ()=>{
        console.log(document.getElementById("txtCmd").value)
        ws.send(document.getElementById("txtCmd").value)
    }
</script>
</body>
</html>
```



## xterm.js

前端页面使用 [xterm.js](https://github.com/xtermjs/xterm.js) 进行模拟terminal展示，只要 javascript 监听 Terminal 对象的对应事件及 websocket 连接的事件，进行对应的页面展示和消息推送就可以了。
