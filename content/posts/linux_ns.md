---
title: "Linux Namespace 机制"
date: 2023-05-13
draft: false
Categories: [Linux]
---

Linux Namespace 提供了一种内核级别隔离系统资源的方法，通过将系统的全局资源放在不同的 Namespace 中，来实现资源隔离的目的。不同 Namespace 的程序，可以享有一份独立的系统资源。目前Linux中提供了六类系统资源的隔离机制，分别是：

- `Mount`: 隔离文件系统挂载点
- `UTS`: 隔离主机名和域名信息
- `IPC`: 隔离进程间通信
- `PID`: 隔离进程的ID
- `Network`: 隔离网络资源
- `User`: 隔离用户和用户组的ID

namespace 的主要作用：封装抽象，限制，隔离，使命名空间内的进程看起来拥有他们自己的全局资源



## Network Namespace

Network namespaces 隔离了与网络相关的系统资源（这里罗列一些）：

- network devices - 网络设备
- IPv4 and IPv6 protocol stacks - IPv4、IPv6 的协议栈
- IP routing tables - IP 路由表
- firewall rules - 防火墙规则
- /proc/net （即 /proc/PID/net）
- /sys/class/net
- /proc/sys/net 目录下的文件
- 端口、socket
- UNIX domain abstract socket namespace

```bash
[root@lain1 ns]# cd /proc/$$/ns	# $$代表当前shell进程
[root@lain1 ns]# ll	# 格式为：namespace类型:[inode number]
```

### 创建网络隔离

首先我们可以创建一个命名为 test_ns 的 network namespace

```bash
[root@lain1 ns]# ip netns add test_ns
[root@lain1 ns]# ip netns list
test_ns
```

当 ip 命令工具创建一个 network namespace 时，会默认创建一个回环设备（loopback interface：lo），并在 /var/run/netns 目录下绑定一个挂载点，这就保证了就算 network namespace 中没有进程在运行也不会被释放，也给系统管理员对新创建的 network namespace 进行配置提供了充足的时间

通过 ip netns exec 命令可以在新创建的 network namespace 下运行网络管理命令

```bash
[root@lain1 ns]# ip netns exec test_ns ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

上面的命令为我们展示了新建的 namespace 下可见的网络链接，可以看到状态是 DOWN，需要再通过命令去启动。可以看到，此时执行 ping 命令是无效的

```bash
[root@lain1 ns]# ip netns exec test_ns ping 127.0.0.1
connect: Network is unreachable
```

启动命令如下，可以看到启动后再测试就可以 ping 通

```bash
[root@lain1 ns]# ip netns exec test_ns ip link set lo up
[root@lain1 ns]# ip netns exec test_ns ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[root@lain1 ns]# ip netns exec test_ns ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.031 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.022 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.023 ms
```

这样只是启动了本地的回环，要实现与外部 namespace 进行通信还需要再建一个网络设备（veth）对，在宿主机执行如下命令

```bash
[root@lain1 ns]# ip link add veth0 type veth peer name veth1
[root@lain1 ns]# ip a
252: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 32:04:e1:55:5e:7b brd ff:ff:ff:ff:ff:ff
253: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3e:8b:dc:8a:c1:90 brd ff:ff:ff:ff:ff:ff
```

在宿主机上把 veth0 这一端分配到 test_ns 这个 namespace 上，把 veth1 这一端分配到 test_ns2 这个 namespace 上

```bash
[root@lain1 ns]# ip link set veth0 netns test_ns && ip link set veth1 netns test_ns2
[root@lain1 ns]# ip netns exec test_ns ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
253: veth0@if252: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 3e:8b:dc:8a:c1:90 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```

分别给 ns 的网络设备分配 ip

```bash
[root@lain1 ns]# ip netns exec test_ns ip addr add local 10.1.1.1/24 dev veth0
[root@lain1 ns]# ip netns exec test_ns2 ip addr add local 10.1.1.2/24 dev veth1
```

分别启动 veth 设备，此时两个 ns 就可以互通了

```bash
[root@lain1 ns]# ip netns exec test_ns ip link set veth0 up
[root@lain1 ns]# ip netns exec test_ns2 ip link set veth1 up
[root@lain1 ns]# ip netns exec test_ns ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.051 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=0.039 ms
```

删除 net namespace

```bash
[root@lain1 ns]# ip netns delete test_ns
```



## UTS Namespace

UTS namespace 提供了主机名和域名的隔离，这样每一个容器就可以拥有独立的主机名和域名，在网络上可以被视为一个独立的节点而非宿主机上的一个进程

使用 Linux（centos7）集成工具 `unshare` 命令，可以创建不同的 namespace

```bash
[root@lain1 ~]# unshare --fork --uts /bin/bash # 启动主机名隔离
[root@lain1 ~]# hostname test
[root@lain1 ~]# hostname
test
[root@lain1 ~]# exit
[root@lain1 ~]# hostname
lain1
```



## Mount Namespace

Mount namespace 通过隔离文件系统挂载点对隔离文件系统提供支持。隔离后，不同的 mount namespace 中的文件结构发生变化也互不影响。你可以通过 `/proc/[pid]/mounts` 查看到所有挂载在当前 namesapce 中的文件系统，还可以通过 `/proc/[pid]/mountstats` 看到 mount namespace 中文件设备的统计信息，包括挂载的文件名称，文件系统类型，挂载位置等等

进程在创建 mount namespace 的时候，会把当前结构复制给新的 namespace。 新的 namespace 中的所有 mount 操作都影响自身的文件系统，而对外界不会产生任何影响。这样做就严格地实现了隔离

```bash
[root@lain1 ~]# mkdir /mnt/mytemp						# 创建一个文件夹
[root@lain1 ~]# unshare --fork --uts --mount /bin/bash	# 同时实现主机名隔离和挂载隔离
[root@lain1 ~]# mount -t tmpfs myfs /mnt/mytemp			# 执行挂载
[root@lain1 ~]# df -h
myfs            3.9G     0  3.9G   0% /mnt/mytemp
```



## User Namespace

User namespace 主要隔离了安全相关的标识符和属性，包括用户ID、用户组ID、root目录等。通俗点就是：一个普通用户的进程通过 clone() 创建新的进程在新 user namespace 中可以拥有不同的用户和用户组。这意味着一个进程在容器外属于一个没有特殊权限的普通用户，但是它创建的容器进程却属于拥有所有权限的超级用户，这个技术为容器提供了极大的自由

安装一些库

```bash
[root@lain1 ~]# curl https://forensics.cert.org/cert-forensics-tools-release-el7.rpm -o cert-forensics-tools-release-el7.rpm 
[root@lain1 ~]# rpm -Uvh cert-forensics-tools-release*rpm
[root@lain1 ~]# yum --enablerepo=forensics install -y musl-libc-static
```

下载 [alpine](https://www.alpinelinux.org/downloads/)，进入 bin 目录执行命令进入一个新的 sh 终端，好比进入容器进行隔离

```bash
[root@lain1 ~]# ./busybox sh
```

### 创建用户隔离

默认情况下，需要修改 max_user_namespaces 文件的值，默认是0

```bash
[root@lain1 ~]# echo 65535 > /proc/sys/user/max_user_namespaces
```

隔离创建新的用户 namespace，隔离所对应的进程是 busybox sh

```bash
[txl@lain1 ~]$ unshare --fork --user /home/txl/alpine/bin/busybox sh

/home/txl $ id			# 容器内默认的uid是65534
uid=65534 gid=65534 groups=65534

/home/txl $ echo $$		# 当前进程ID(还没有做进程隔离，所以显示的是宿主机的PID)
31920
```

默认会映射 `/proc/sys/kernel/overflowuid` 和 `/proc/sys/kernel/overflowgid`

为 busybox 设置 capability 特权

> linux 内核2.2之后引入了 capabilities 机制，来对 root 权限进行更加细粒度的划分。如果进程不是特权进程，而且也没有 root 的有效 id，系统就会去检查进程的 capabilities，来确认该进程是否有执行特权操作的的权限

```bash
[txl@lain1 ~]$ sudo setcap cap_setgid,cap_setuid+ep /home/txl/alpine/bin/busybox 
[txl@lain1 ~]$ getcap /home/txl/alpine/bin/busybox # 查看
/home/txl/alpine/bin/busybox = cap_setgid,cap_setuid+ep

[txl@lain1 ~]$ sudo setcap cap_setgid,cap_setuid-ep /home/txl/alpine/bin/busybox # 取消
```

给容器映射用户，方法是添加映射信息到 `/proc/$$/uid_map` 和 `/proc/$$/gid_map` 中

```bash
# 父 namespace 中的 1000~1256 映射到新 user namespace 中的 0~256
[root@lain1 ~]# echo '0 1000 256' > /proc/31920/uid_map
[root@lain1 ~]# echo '0 1000 256' > /proc/31920/gid_map
```

此时容器内会立刻产生变化

```bash
/home/txl # id
uid=0(root) gid=0(root) groups=0(root),65534
```



*参考：[浅谈 Linux Namespace](https://xigang.github.io/2018/10/14/namespace-md/)*
