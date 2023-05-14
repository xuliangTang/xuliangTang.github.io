---
title: "Linux 资源管理之 cgroup"
date: 2023-05-14
draft: false
Categories: [Linux]
---

[cgroup](https://man7.org/linux/man-pages/man7/cgroups.7.html) 是 Linux kernel 的一项功能：它是在一个系统中运行的层级制进程组，你可对其进行资源分配（如 CPU 时间、系统内存、网络带宽或者这些资源的组合）。通过使用 cgroup，系统管理员在分配、排序、拒绝、管理和监控系统资源等方面，可以进行精细化控制。硬件资源可以在应用程序和用户间智能分配，从而增加整体效率

cgroup 和 namespace 类似，也是将进程进行分组，但它的目的和  namespace 不一样，namespace 是为了隔离进程组之间的资源，而 cgroup 是为了对一组进程进行统一的资源监控和限制



## cgroups 组成

- 任务 task：系统的一个进程
- 控制组 cgroup：用来设定资源的配额，task 可以加入到某个组，也可以迁移。可以包含多个子系统
- 子系统 subsystem：资源调度控制器（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量
- 层级 hierarchy：由一系列 cgroup 以一个树状结构排列而成，每个 hierarchy 通过绑定对应的 subsystem 进行资源调度。 hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy



## Subsystem

cgroups 为每种可以控制的资源定义了一个子系统（Subsystem）

- blkio: blkio 子系统控制并监控 cgroup 中的task对块设备的 I/O 的访问。如：限制访问及带宽等
- cpu: 主要限制进程的 cpu 使用率
- cpuacct: 可以统计 cgroup 中进程的 cpu 使用报告
- cpuset: 可以为 cgroup 中的进程分配独立的 cpu 和内存节点
- memory: 自动生成 cgroup 中 task 使用的内存资源报告，并对该 cgroup 的 task 进行内存使用限制
- devices: 可以控制进程能否访问某些设备
- net_cls: 使用等级标识符 (clssid) 标记网络数据包，可允许 Linux 流量控制程序 (tc) 识别从具体 cgroup 中生成的数据包
- freezer: 可以挂起挂起或回复 cgroup 中的进程
- ns: 可以使不同 cgroup 中的进程使用不同的 namespace



## cgroup 限制

安装用来操作 cgroup 相关的命令

```bash
[root@lain1 myapp]# yum install -y libcgroup-tools.x86_64
```

cgroup 以文件挂载的方式存在，其中包含 /sys/fs/cgroup 文件夹，它是挂载到 tmpfs（临时内存文件夹）中的

###  CPU 使用率

在 cpu 目录下创建一个 myapp 文件夹，会自动生成对应的配置文件，用来产生对应的限制

```bash
[root@lain1 myapp]# cd /sys/fs/cgroup/cpu
[root@lain1 myapp]# mkdir myapp
[root@lain1 myapp]# echo 100000 > cpu.cfs_period_us		# 一个CFS调度时间周期长度，默认100000微秒
[root@lain1 myapp]# echo 10000 > cpu.cfs_quota_us		# 在上方一个周期内，允许的时间，默认-1不限制
```

 Linux 内核功能 cfs（[Completely Fair Scheduler](https://kernel.org/doc/Documentation/scheduler/sched-bwc.txt) 完全公平调度器）负责进程调度。配置 `cfs_quota_us / cpu.cfs_period_us = 0.1 ` 好比只能使用0.1个 cpu（10%）

```bash
[root@lain1 myapp]# cgexec -g cpu:myapp ./myapp		# 在指定的cgroup中运行程序myapp
```

### 内存使用率

在 memory 目录下创建一个 myapp 文件夹，修改 memory.limit_in_bytes 和 memory.swappiness

```bash
[root@lain1 myapp]# echo 1g > memory.limit_in_bytes		# 设定用户内存（包括文件缓存）的最大用量，默认单位是字节
[root@lain1 myapp]# echo 0 > memory.swappiness			# 设置如何使用swap分区。0表示最大限度使用物理内存，然后才是swap空间
```

假设 memory.swappiness 设置为60，当内存使用到40%时（100-60）开始 swap 交换

```bash
[root@lain1 myapp]# cgexec -g memory:myapp ./myapp		# 在指定的cgroup中运行程序myapp，当内存超出时会被kill
Killed
```



*参考：*

- *[Linux Cgroup 入门教程：基本概念 – 云原生实验室](https://icloudnative.io/posts/understanding-cgroups-part-1-basics/)*
- *[浅谈Cgroups](https://xigang.github.io/2018/07/08/cgroups/)*
- *[一篇搞懂容器技术的基石： cgroup](https://segmentfault.com/a/1190000040980305)*
