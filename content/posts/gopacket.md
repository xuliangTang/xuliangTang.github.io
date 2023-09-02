---
title: "数据包捕获库 gopacket"
date: 2023-09-02
draft: false
Categories: [go]
---

[gopacket](https://github.com/google/gopacket) 包为用 C 编写的 libpcap 提供了一个 go 包装器，Libp­cap 提供的接口函数实现和封装了与数据包截获有关的过程，可以在绝大多数 Linux 平台上运行，主要的功能有：

- 数据包捕获：捕获流经网卡的原始数据包
- 自定义数据包发送：构造任何格式的原始数据包
- 流量采集与统计：采集网络中的流量信息
- 规则过滤：提供自带规则过滤功能，按需要选择过滤规则

使用 gopacket 包之前，要确保在 windows 平台安装 [npcap](https://npcap.com/) 或在 linux 平台安装 libpcap

```bash
yum install libpcap libpcap-devel
```



## 实验

### 获取网络设备

```go
func main() {
	// 得到所有的(网络)设备
	devs, err := pcap.FindAllDevs()
	if err != nil {
		log.Fatalln(err)
	}
	// 打印设备信息
	for _, dev := range devs {
		fmt.Println("Name: ", device.Name)
        fmt.Println("Description: ", device.Description)
        fmt.Println("Devices addresses: ", device.Description)
        for _, address := range device.Addresses {
            fmt.Println("- IP address: ", address.IP)
            fmt.Println("- Subnet mask: ", address.Netmask)
        }
	}
}
```

### 抓包网卡

```go
func main() {
	// 创建handler 打开网络设备eth0
    // device: 网络设备的名称，如 eth0,也可以填充 pcap.FindAllDevs() 返回的设备的 Name
    // snaplen: 每个数据包读取的最大长度
    // promisc: 是否将网口设置为混杂模式,即是否接收目的地址不为本机的包
    // timeout: 设置抓到包返回的超时。如果设置成30s，那么每30s才会刷新一次数据包；设置成负数，会立刻刷新数据包，即不做等待
	handler, err := pcap.OpenLive("eth0", 1024, false, time.Second*5)
	if err != nil {
		log.Fatalln(err)
	}
	defer handler.Close()

	// 创建数据包源，监听网卡解析网卡数据
    // 第一个参数为 OpenLive 的返回值，指向 Handle 类型的指针变量 handle
    // 第二个参数为 handle.LinkType() 此参数默认是以太网链路，一般我们抓包，也是从2层以太网链路上抓取
	source := gopacket.NewPacketSource(handler, handler.LinkType())

	// 从chan读取数据包
	for packet := range source.Packets() {
		fmt.Println(packet.String()) // 每隔5s 打印网卡数据
	}
}
```

### 抓取http包

```go
for packet := range source.Packets() {
	// 获取网络传输层
	if transportLayer := packet.TransportLayer(); transportLayer != nil {
		// 只抓取tcp包
		if tcpLayer, ok := transportLayer.(*layers.TCP); ok {
			// 目标端口或源端口为8080的数据包
			if tcpLayer.DstPort == 8080 || tcpLayer.SrcPort == 8080 {
				fmt.Println(string(tcpLayer.Payload))
			}
		}
	}
}
```

### 抓包TCP三次握手

**步骤**

1. 在初始化时，双端处于 CLOSE 状态，服务端为了提供服务，会主动监听某个端口， 进入 LISTEN 状态
2. 客户端主动发送连接的 「SYN 」包，之后进入 STN-SENT 状态·，服务端在收到客户端发来的 「SYN」 包后，回复 「SYN,ACK」 包，之后进入 SYN-RCVD 状态
3. 客户端收到服务端发来的 「SYN-ACK」 包后，可以确认对方存在，此时回复 「ACK」 包，并进入 ESTABLISHED 状态
4. 服务端收到最后一个 「ACK」 包后，也进入 ESTABLISHED 状态

<img src="https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202309021744317.png" alt="TCP 三次握手" />

**序列号和确认号**：用于确认数据是否准确，是否能够正常通信

- 序号(sequence number)：seq 序号，标记从 TCP 源端向目的端发送的字节流，发起方发送数据时对此进行标记
- 确认号(acknowledgement number)：ack 序号，占32位，只有 ACK 标志位为1时，确认序号字段才有效，ack = seq + 1

![TCP 头格式](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202309021850552.png)

基本的交互流程：

- 第一次：seq 为 x (x 为任意值)，无 ack number
- 第二次：seq 为 y (y 为任意值)，ack number = 接收包seq + 1 (即 x + 1)
- 第三次：seq 等于上一个本机发送包 seq + 1 (即 x + 1)，ack number 等于接收包 seq + 1 (即 y + 1)

**代码模拟**

TCP 服务端：

```go
func main() {
	l, err := net.Listen("tcp", "0.0.0.0:8080")
	if err != nil {
		log.Fatalln(err)
	}

	for {
		// 接收客户端连接
		conn, err := l.Accept()
		if err != nil {
			log.Fatalln(err)
		}

		fmt.Printf("Received message %s -> %s \n", conn.RemoteAddr(), conn.LocalAddr())
		go handler(conn)
	}
}

// 处理连接
func handler(conn net.Conn) {
	for {
		buf := make([]byte, 1024)
		len, err := conn.Read(buf)
		if err != nil {
			fmt.Println(err)
			break
		}
		fmt.Printf("Received data: %v", string(buf[:len]))

		// 回复
		len, err = conn.Write([]byte("ok"))
		if err != nil {
			fmt.Println(err)
			break
		}
	}
}
```

TCP 客户端：

```go
func main() {
	conn, err := net.Dial("tcp", "serverip:8080")
	if err != nil {
		log.Fatalln(err)
	}

	for {
		// 发送
		_, err = conn.Write([]byte("hello"))
		if err != nil {
			log.Fatalln(err)
		}
		time.Sleep(time.Second * 2)
	}
}
```

抓包获取三次握手标记

```go
for packet := range source.Packets() {
	// 获取网络传输层
	if transportLayer := packet.TransportLayer(); transportLayer != nil {
		// 只抓取tcp包
		if tcpLayer, ok := transportLayer.(*layers.TCP); ok {
			// 目标端口或源端口为8080的数据包
			if tcpLayer.DstPort == 8080 || tcpLayer.SrcPort == 8080 {
				// 分别打印来源端口 目标端口 SYN标记 ACK标记 seq序号 ack序号和数据包的长度(0)
				fmt.Printf("%d-->%d, SYN=%v,ACK=%v,seq=%v,ackNum=%v PayloadLength=%d\n",
						tcpLayer.SrcPort, tcpLayer.DstPort, tcpLayer.SYN, tcpLayer.ACK, tcpLayer.Seq, tcpLayer.Ack, len(tcpLayer.Payload))
			}
		}
	}
}
```

### SYN 攻击原理

攻击者短时间内伪造不同IP地址发送大量的 SYN 包 ，服务端每接收到一个 SYN 报文，就进入 SYN_RCVD 状态回应 SYN,ACK 包，但无法得到未知 IP 主机的 ACK 应答，默认清空下会重试5次 (/etc/sysctl.conf)，久而久之就会占满服务端的半连接队列，使得服务端不能为正常用户服务

![SYN 攻击](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202309021911675.png)

**模拟 SYN 攻击**

安装 hping3：一款 TCP/IP 数据包编辑器/分析器，是安全审计、防火墙测试等工作的标配工具。支持 TCP、UDP、ICMP 和 RAW-IP 协议，具有跟踪路由模式，在覆盖通道之间发送文件的功能以及许多其他功能，优势在于能够定制数据包的各个部分，因此用户可以灵活对目标机进行细致地探测

```bash
yum install -y hping3
```

模拟攻击：

```bash
# -c 发送数据包的个数
# -a 伪造ip
# --flood 疯狂的发送数据包
hping3 -I eth0 -c 1 -a 192.168.10.60 192.168.0.111 --syn -p 8080
```



*参考：*

- *[TCP 三次握手与四次挥手面试题](https://www.xiaolincoding.com/network/3_tcp/tcp_interview.html)*
