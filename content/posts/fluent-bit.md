---
title: "Fluent Bit 日志收集"
date: 2023-02-25
draft: false
Categories: [kubernetes]
---

[fluent-bit](https://docs.fluentbit.io/manual/) 是一种在 Linux，OSX 和 BSD 系列操作系统运行，兼具快速、轻量级日志处理器和转发器。它非常注重性能，通过简单的途径从不同来源收集日志事件

数据分析通常发生在数据存储和数据库索引之后，但对于实时和复杂的分析需求，在日志处理器中处理仍在运行的数据会带来很多好处，这种方法被称为**边缘流处理**（Stream Processing on the Edge）

## 工作原理

日志通过数据管道从数据源发送到目的地，一个数据管道通常由 Input、Parser、Filter、Buffer、Routing 和 Output组成

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302251337719.png)

- Input：用于从数据源抽取数据，一个数据管道中可以包含多个 Input
- Parser：负责将 Input 抽取的非结构化数据转化为标准的结构化数据，每个 Input 均可以定义自己的 Parser（可选）
- Filter：负责对格式化数据进行过滤和修改。一个数据管道中可以包含多个 Filter，Filter 会顺序执行，其执行顺序与配置文件中的顺序一致
- Buffer：用户缓存经过 Filter 处理的数据，默认情况下 Buffer 把 Input 插件的数据缓存到内存中，直到路由传递到 Output 为止
- Routing：将 Buffer 中缓存的数据路由到不同的 Output
- Output：负责将数据发送到不同的目的地，一个数据管道中可以包含多个 Output

Fluent Bit 支持多种类型的 Input、Parser、Filter、Output 插件，可以应对各种场景：

![img](https://raw.githubusercontent.com/xuliangTang/picbeds/main/picgo/202302251404004.png)



## 关键概念

### 事件或记录（Event or Record）

Fluent Bit 检索到的每一个属于日志或指标的输入数据都被视为事件或记录，以 Syslog 文件为例：

```ini
Jan 18 12:52:16 flb systemd[2222]: Starting GNOME Terminal Server
Jan 18 12:52:16 flb dbus-daemon[2243]: [session uid=1000 pid=2243] Successfully activated service 'org.gnome.Terminal'
Jan 18 12:52:16 flb systemd[2222]: Started GNOME Terminal Server.
Jan 18 12:52:16 flb gsd-media-keys[2640]: # watch_fast: "/org/gnome/terminal/legacy/" (establishing: 0, active: 0)
```

它包含四行，它们代表四个独立的事件。在内部，一个事件总是有两个组件（以数组形式）：

```
[TIMESTAMP, MESSAGE]
```

### 过滤（Filtering）

在某些情况下，需要对事件内容执行修改，更改、填充或删除事件的过程称为过滤

有许多需要过滤的用例，如：

- 向事件附加特定信息，如 IP 地址或元数据
- 选择一个特定的事件内容
- 处理匹配特定模式的事件

### 标签（Tag）

每一个进入 Fluent Bit 的事件都会被分配一个标签。这个标签是一个内部字符串，路由器稍后会使用它来决定必须通过哪个 Filter 或 Output 阶段

大多数标签都是在配置中手动分配的。如果没有指定标签，那么 Fluent Bit 将指定生成事件的输入插件实例的名称作为标签

> 唯一不分配标签的输入插件是 Forward 输入。这个插件使用名为 Forward 的 Fluentd wire 协议，其中每个事件都有一个相关的标签。Fluent Bit 将始终使用客户端设置的传入标签

标签记录必须始终具有匹配规则。要了解关于标签和匹配的更多信息，请查看路由部分

### 时间戳（Timestamp）

时间戳表示事件被创建的时间，每个事件都包含一个相关联的时间戳。时间戳的格式：

```
SECONDS.NANOSECONDS
```

Seconds 是自 Unix epoch 以来经过的秒数。Nanoseconds 是小数秒或十亿分之一秒

> 时间戳总是存在的，要么由 Input 插件设置，要么通过数据解析过程发现

### 匹配（Match）

Fluent Bit 允许将收集和处理的事件传递到一个或多个目的地，这是通过路由阶段完成的。匹配表示一个简单的规则，用于选择与已定义规则匹配的事件

### 结构化消息（Structured Messages）

源事件可以有或没有结构。结构在事件消息中定义了一组键和值。作为一个例子，考虑以下两个消息：

没有结构化的消息

```
"Project Fluent Bit created on 1398289291"
```

结构化消息：

```
{"project": "Fluent Bit", "created": 1398289291}
```

在较低的级别上，两者都只是字节数组，但结构化消息定义了键和值，具有结构有助于实现对数据快速修改的操作

> Fluent Bit 总是将每个事件消息作为结构化消息处理。出于性能原因，我们使用名为 MessagePack 的二进制序列化数据格式
>
> 可以把 MessagePack 看作是 JSON 的二进制版本



## 容器日志

docker 容器中输出到 stdout 的日志，都会以 *-json.log 的命名方式保存在 /var/lib/docker/containers/ 目录下， 且在 /var/log/containers/ 目录下生成的软链接，如：

```json
{"log":"10.244.0.1 - - [24/Feb/2023:17:33:29 +0000] \"GET / HTTP/1.1\" 200 612 \"-\" \"curl/7.29.0\" \"-\"\n","stream":"stdout","time":"2023-02-24T17:33:29.239451924Z"}
```

其中包含三个字段：log、stream 和 time，当然仅仅这三个字段还不够，还需要知道容器属于哪个POD，信息越详细越好，这些 fluent-bit 都通过插件支持好了



## 部署

参考文档：[installation](https://docs.fluentbit.io/manual/installation/kubernetes)

先创建 `logging` 命名空间，从 [fluent-bit-kubernetes-logging](https://github.com/fluent/fluent-bit-kubernetes-logging) 仓库里执行安装 role、role-binding、service-account、configmap 和 ds 资源

示例配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
  labels:
    k8s-app: fluent-bit
data:
  # Configuration files: server, input, filters and output
  # ======================================================
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    @INCLUDE input.conf
    @INCLUDE output.conf
    @INCLUDE filter.conf

  input.conf: |
    [INPUT]
        Name   mem				# 获取内存信息
        Tag    memory
    [INPUT]
        Name              tail	# 获取docker日志文件
        Tag               kube.*
        Path              /var/log/containers/*.log
        Path_Key          logfile
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB	# 内存缓冲大小
        Skip_Long_Lines   On
        Refresh_Interval  5

  output.conf: |
    [OUTPUT]
       Name  es				# 输出到elasticsearch
       Match memory			# 匹配mem
       Host  192.168.0.111
       Port  9200
       Index mem_index
       Trace_Error On
    [OUTPUT]
       Name  es
       Match kube.*			# 匹配docker日志
       Host  192.168.0.111
       Port  9200
       Index pod_index

  filter.conf: |
    [FILTER]
        Name record_modifier
        Match kube.*
        # 手动添加nodename字段，值为fluent-bit的环境变量NODE_NAME，ds中配置env示例:
        #env:
        #    - name: NODE_NAME
        #      valueFrom:
        #        fieldRef:
        #          fieldPath: spec.nodeName
        Record nodename ${NODE_NAME}
    [FILTER]
        Name                kubernetes	# 关联的k8s元信息
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off

  parsers.conf: |
    [PARSER]
        Name   nginx
        Format regex
        Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z

    [PARSER]
        Name        docker
        Format      json
        Time_Key    time	# 时间戳字段
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On

    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S
```

没有热更新，修改配置后删除 pod

```bash
kubectl get pod -n logging | grep fluent-bit | awk '{print $1}' | xargs kubectl delete pod -n logging
```

