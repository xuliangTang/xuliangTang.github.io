---
title: "使用 kubeadm 部署 k8s"
date: 2022-12-03
draft: flase
Categories: [kubernetes]
---

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具

## 部署
文档地址：[安装 kubeadm | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm)

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/202212151842316.png)

### 添加 kubenetes 的 yum 源

在每个节点上分别执行

```bash
$ su -

$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

$ yum makecache
```

### 安装 kubeadm、kubelet 和 kubectl

在每个节点上分别执行

```bash
$ sudo yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0
```

安装后查看列表

```bash
$ rpm -aq kubelet kubectl kubeadm
```

把kubelet设置为开机启动

```bash
$ sudo systemctl enable kubelet
```

>  ``kubeadm init`` 集群的快速初始化，部署Master节点的各个组件
>
>  ``kubeadm join`` 节点加入到指定集群中
>
>  ``kubeadm token`` 管理用于加入集群时使用的认证令牌 (如list，create)
>
>  ``kubeadm reset`` 重置集群，如删除构建文件以回到初始状态 

### 使用 systemd 作为 docker 的 cgroup driver

在每个节点上执行

```bash
$ sudo vi  /etc/docker/daemon.json
```

加入内容

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

重启docker

```bash
$ systemctl daemon-reload  && systemctl restart docker
```

验证结果

```bash
$ docker info |grep Cgroup
 Cgroup Driver: systemd
 Cgroup Version: 1
```

### 关闭 swap

- 临时关闭

```bash
$ swapoff -a
```

- 永久关闭

```bash
$ sudo vi /etc/fstab

# 注释掉SWAP分区项
# swap was on /dev/sda11 during installation
# UUID=0xxxxxxxxxxxxxx4f69 none  swap    sw      0       0
```

### 初始化集群

```bash
$ sudo kubeadm init --kubernetes-version=v1.21.0  --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16
```

根据输出提示操作

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

默认token的有效期为24小时，当过期之后，该token就不可用了。

**查看token列表：** ``$ sudo kubeadm token list``

**重新生成token：** ``$ sudo kubeadm token create  --print-join-command``

### 安装网络组件

#### CNI (Container Network Interface)

容器网络接口，为了让用户在容器创建或销毁时都能够更容易地配置容器网络。常见的组件有：

- Flannel: 最基本的网络组件
- Calico: 支持网络策略
- Canal: 前两者的合体
- Weave: 同样支持策略机制，还支持加密

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/202212151842523.png)

#### 使用 flannel

Github地址：[https://github.com/coreos/flannel](https://github.com/coreos/flannel)

在每个节点执行

![image-20221203185728214](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/image-20221203185728214.png)

```bash
$ sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

在master节点执行

```bash
$ kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 去污点
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

查看

```bash
kubectl get pods --all-namespaces
```

#### 可能出现的错误

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/202212151846852.png)

基本排查命令

```bash
$ kubectl describe

$ journalctl -f -u kubelet

$ for p in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name); do kubectl logs --namespace=kube-system $p; done
```

##### coredns

```bash
$ kubectl describe pod -n kube-system coredns-xxx
```

> network: open /run/flannel/subnet.env: no such file or directory

手动在每个节点上创建

```bash
$ sudo vi /run/flannel/subnet.env

# 加入内容
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```



> "cni0" already has an IP address different from 10.244.1.1/24

```bash
$ sudo ip link delete cni0

# 重启pod
$ kubectl delete pod xxx
```



> Readiness probe failed: HTTP probe failed with statuscode: 503

```bash
$ systemctl stop kubelet
$ systemctl stop docker
$ iptables --flush
$ iptables -tnat --flush
$ systemctl start kubelet
$ systemctl start docker
```

##### 节点 not ready

![](https://raw.githubusercontent.com/xuliangTang/picbeds/main/typora/202212151847731.png)

```bash
$ kubectl describe node lain2
```

> failed to find plugin "xxx" in path [/opt/cni/bin]

把master节点的 ``/opt/cni/bin`` 拷贝过来，在master节点执行

```bash
$ cd /opt/cni
$ sudo scp -r bin root@192.168.0.105:/opt/cni/bin
```

原因可能是重装k8s的时候没有删除 ``/etc/cni/net.d`` 目录

### 加入子节点

在每个node节点执行刚刚初始化集群时生成的token

```bash
$ sudo kubeadm join 192.168.0.111:6443 --token fnq8dx.doxwr7sctdm57p0t \
    --discovery-token-ca-cert-hash sha256:114acfe6e30bc0181a93b9135296af62c5f946fa590691577071a4ebf21fc3ee
```

### 查看集群健康状况

```bash
$ kubectl get cs
```

> controller-manager && scheduler Unhealthy: dial tcp 127.0.0.1:10252 connection refuse

在master节点执行

```
$ sudo vi /etc/kubernetes/manifests/kube-scheduler.yaml
$ sudo vi /etc/kubernetes/manifests/kube-controller-manager.yaml

# 注释
# - --port=0

# 重启kublet
$ systemctl restart kubelet
```



## 将集群导入 Rancher

找一个节点执行下载v2.6

```bash
$ sudo docker run --privileged -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher:v2.6-head
```

根据指引执行命令
