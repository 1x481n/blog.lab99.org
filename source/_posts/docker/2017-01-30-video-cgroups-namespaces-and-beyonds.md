---
layout: post
category: docker
title: 视频笔记：容器是如何炼成的？ - Jérôme Petazzoni
date: 2017-01-30
tags: [docker, dockercon15-eu, youtube, notes]
---

<!-- toc -->

# 视频信息

Cgroups, namespaces and beyond: what are containers made from?
by Jérôme Petazzoni (@jpetazzo)
at DockerCon EU 2015

{% owl youtube sK5i-N34im8 %}

<https://www.youtube.com/watch?v=sK5i-N34im8>

幻灯地址：<http://www.slideshare.net/Docker/cgroups-namespaces-and-beyond-what-are-containers-made-from>

# 自我介绍

Jérôme Petazzoni ([@jpetazzo](https://github.com/jpetazzo))，是法国软件工程师，现在住在加州。他是 [dotCloud](https://en.wikipedia.org/wiki/DotCloud) 的老员工了，构建了其 PaaS 平台，也就是说早在 Docker 成为 Docker 之前，他就已经在为 Docker 写代码了。都快 5 年了。

非常喜欢宣讲容器的好处，第一次估计是 2013 年 2 月在洛杉矶的 SCALE 大会上的讲座。

非常自豪自己是 Bash 堂的成员。（就是说如果觉得太烦了，就会把某些代码换成一段脚本来实现……😹）

# 大纲

这次讲座是满满的干货：

* 从上层和底层分别讲解到底什么是容器。
* 讲解容器的基石：`namespaces`, `cgroups`, `copy-on-write storage`
* 各种容器运行时：`Docker`, `LXC`, `rkt`, `runC`, `systemd-nspawn`, `OpenVZ`, `Jails`, `Zones`
* 最后现场用纯手工来制作一个容器👍

# 什么是容器？

## 从上层视角去看：它是一个轻量级虚拟机

最初介绍容器的时候，总会用这样的说法，”嗯，容器嘛，可以理解为一个轻量级虚拟机”，而实际过程中，我们却又经常说"Docker 不是虚拟机，不要在这么理解 Docker“。

为什么出现这种情况呢？因为如果用户啥都不懂的时候，用户从表面上会觉得和虚拟机接近：

* 用户可以取得一个 Shell，输入命令和返回结果
* 用起来感觉上**像是**虚拟机
	* 有自己的进程空间
		* 运行 `ps` 的时候只能看到容器内的进程
	* 有自己的网络接口
		* `ip addr`, `ifconfig` 时可以看到自己独立的网络接口
	* 还可以以 `root` 身份运行东西
	* 甚至还可以用 `apt`/`yum` 来安装软件
	* 还可以启动服务
	* 还可以随意的鼓捣路由表、`iptables`……


## 但是从底层去看：它是一个玩嗨了的 `chroot`

它并不像虚拟机，因为：

* 它用的是宿主的内核
* 它不能启动到别的系统（比如容器里跑个 FreeBSD)
* 它不能加载自己内核模块
* 甚至不需要作为 `PID` 为 `1` 的 `init`
* 而且里面根本不跑 `syslogd`, `cron` 等等系统服务

所以容器只是一堆进程，可以直接在宿主 `ps` 看到所有容器的进程。这点和虚拟机完全不同，虚拟机是不透明的。

而这些容器内的进程只可以看到自己的空间内的东西，而看不到其它容器或者宿主空间内的东西。所以它们生活在自己的世界里，可以拥有自己的操作系统（当然指Linux不同的发行版本）。

# 那容器是怎么实现的？

5年前 Jérôme 加入 dotCloud 的时候，想理解一下啥是容器，于是来看 Linux 内核源代码吧！

嗯，赶紧跑到 [LXR](http://lxr.free-electrons.com/) 去查阅 Linux 代码

结果发现，搜索 `LXC`，一条记录都没有 **(⊙﹏⊙)b**；再去搜索 `container`，嗯，这次倒是有了一千多条结果，不过仔细一看，都是一帮子数据结构，比如 `ACPI containers` 之类的，这是什么👻？完全跟我们说的容器没关系嘛。

倒是有些文档里或隐或现的提到了一些关于我们所说的容器，不过那是文档，不是代码啊。

好吧，那是心里不得不问一句，容器这鬼难道不在内核里？之前听到容器用的是内核技术都是骗人的不成？甚至……*容器*不会压根就不存在吧？

经过一番研究，搞明白了*容器*不在内核里，在内核里的是实现容器的、著名的 `cgroups` 和 `namespaces`。

# 容器基石 - 1：`Control Groups`

## 什么是 `Control Groups`？

`Control Groups`，受控组，也称为 `cgroups`，顾名思义，就是控制狂，它的目的是对一切进行限制。比如：

* 对资源使用的测量和限制，包括内存、CPU、块设备 I/O、网络等
* 对设备（`/dev/*`）的访问控制
* 拥塞控制

## 概论

* 每个子系统（内存、CPU等）都有一个树形的层级结构
* 层级结构间是独立的，互相不影响。因此内存和CPU可以是不同的层级结构
* 每一个进程在每一个树形层级结构中只可以属于一个节点
* 每一个树形层级结构都起始于一个起点（`root`）
* 所有的进程最初都是属于每个层级结构的根(`root`) 节点
* 每一个节点里是一组进程

因此即使你没有使用 Docker，你的所有进程依旧是在容器里运行的，因为他们都属于各个 cgroups 里的根节点。整个机器就是一个大的容器。所以如果你要是觉得你的进程不在容器里就会更快些，这是绝对不可能的，无论用不用 docker，无论是容器内进程还是宿主进程，都一样的位于容器内。

比如，下面这个是 CPU 的树形层级结构，最终的数字是 `PID`：

```bash
cpu
├── batch
│   ├── bitcoins
│   │   └── 52
│   └── hadoop
│       ├── 109
│       └── 88
└── realtime
    ├── nginx
    │   ├── 25
    │   ├── 26
    │   └── 27
    ├── postgres
    │   └── 524
    └── redis
        └── 1008
```

而下面这个则是内存的树形层级结构，可以看到还是这一批进程，但是和CPU 的层级结构完全不同：

```bash
memory
├── 109
├── 25
├── 26
├── 27
├── 52
├── 88
└── databases
    ├── 1008
    └── 524
```

## 内存 `cgroup`：记账

记录每一组进程使用的内存页（控制的细粒度是*页*）：

* 文件（从块设备的读、写、映射）
* 匿名（栈、堆、匿名映射）
* 活跃（最近访问过的）
* 不活跃（将被清理的）

每一内存页都被记到一个**组**的账上。内存页可以跨多组共享，比如多个进程从一组相同的文件中读取信息。当页被共享的时候，只有一个组会为此买单。

## 内存 `cgroup`：限制

每一个组都可以有 `hard` 和 `soft` 的限制。

* `soft` 限制并不是强制的，它在内存压力下可以影响回收资源。
*  `hard` 则是强制的，直接会触发该组的 `OOM killer`

`OOM Killer` 可以定制其行为。

* 比如，当 `hard limit` 超了的时候，冻结组内所有进程，通知用户空间（而不是 `rampage`），然后可以杀了进程、提升 `limit`、然后在运行新的容器，当一切都好了的时候，在解冻该组。

限制可以设置于物理内存、内核内存或总共内存。

## 内存 `cgroup`：一些需要注意的细节

每次内核给进程一个页面，或者撤走一个页面，它都会更新计数器。这会增加一些额外开销，而且这无法针对特定进程关闭，只可以引导参数上设置。

之前说过，如果多组使用相同的页的时候，只有一个组会买单。不过如果这个组停止使用这个页了，那么账单会跑到另一个使用它的组上。

## `HugeTLB cgroup`

这是用来控制一个进程可以使用的 `huge pages` 的数量。如果你不知道这是啥的话，可以跳过去，没问题。

## `CPU cgroup`

* 记录用户/系统的 CPU 时间
* 记录每 CPU 的使用情况
* 允许设置权重
* 无法设置 CPU 限制

## `cpuset cgroup`

* 可以将组固定在特定的 CPU 上
* 可以保留 CPU 给特定的 app
* 可以避免进程来回在不同的 CPU 上跳来跳去
* 这也和 NUMA 系统相关
* 提供额外的调节能力（每区域内存压力、进程迁移开销……）

## `blkio cgroup`

* 记录每组的 I/O 数目
	* 每个块设备
	* 读、写
	* 同步、异步

* 对每个组设置吞吐量限制
	* 每个块设备
	* 读、写
	* ops 或者 bytes

* 对每个组设置相对权重

**需要注意的是，大部分写操作都是写到缓存的，因此普通的写操作最初会看起来像没有被限制吞吐量一样。**

## `net_cls` 和 `net_prio` `cgroup`

* 自动为组中进程产生的网络流量设置分类和优先级；
* 只对外出流量有效
* `net_cls` 将用于指定流量一个分类
	* 当然，这个分类需要在 `tc`/`iptables` 来匹配从而进行控制，否则会和普通流量一样。
* `net_prio` 会指定流量的优先级
	* 队列机制将会使用这个优先级进行处理

## 设备 `cgroup`

控制一个组可以对一个设备节点做些什么。比如，读、写、`mknod` 等

典型的用法有：

* 允许 `/dev/{tty,zero,random,null}`...
* 拒绝所有其它的。

一些比较有趣的设备节点：

* `/dev/net/tun`： 网络界面维护
* `/dev/fuse`： 用户空间的文件系统
* `/dev/kvm`：容器内的虚拟机？！😂OMG, 虚拟化版盗梦空间？
* `/dev/dri`：GPU

## `freezer cgroup`

允许冻结和唤醒一组进程。功能上相似于 `SIGSTOP/SIGCONT`，但是不同的是进程无法感知到这个状态变化，而 `SIGSTOP/SIGCONT` 是进程可以处理的信号。而且并不会妨碍 `ptrace` 调试。

特殊用法：

* 集群成批调度
* 进程迁移

## 一些注意事项

* 每一个层级关系的根节点都是 `PID 1` 的进程
* 新启动的进程将属于它们的父进程所在的组
* 组将具化为一个或多个伪文件系统
	* 一般位于 `/sys/fs/cgroup` 下
* 创建组的办法就是在那些伪文件系统中用 `mkdir` 创建目录
* 要移动一个进程到一个组只需要：
	* `echo $PID > /sys/fs/cgroup/.../tasks`
* `cgroup` 的战争：`systemd` vs `cgmanager` vs ...

# 容器基石 - 2：`namespaces`

* 为进程提供自己的系统视角
* `cgroups` 限制你用多少；`namespaces` 限制你能看到什么。
* 有多种 `namespaces`：
	* `pid`
	* `net`
	* `mnt`
	* `uts`
	* `ipc`
	* `user`
* 对于某类 `namespace` 而言，每个进程属于一个 `namespace`。

## `pid namespace`

* 一个 `pid namespace` 的进程只能看到在同 `pid namespace` 的其它进程
* 每个 `pid namespace` 都有自己的 `pid` 编号，从 `1` 开始
* 当 `pid 1` 挂了，整个 `namespace` 都会被干掉
* `namespace` 可以嵌套
* 因此一个进程可能会有多个 `PID`，每个 `namespace` 一个 `pid`

## `net namespace`

### 理论上

在一个给定网络命名空间中的进程会拥有自己的私有网络栈，包括：

* 网络接口（包括本地回环）
* 路由表
* iptables 规则
* sockets (ss, netstat)

你可以从一个 `netns` 移动一个网络接口到另一个网络命名空间

```bash
ip link set dev eth0 netns PID
```

### 实际上

典型的使用方式：

* 使用 `veth` 对（两个虚拟接口像使用交叉线连接一样）
* 容器网络命名空间的 `eth0` 和宿主网络命名空间的 `vethXXX` 配对
* 并且所有的 `vethXXX` 都桥接在一起（在Docker中这个桥接如果是默认桥接就是 docker0）

除此之外，还有一种是使用 `--net container:容器` 的使用方式，可以共享网络栈。

## `mnt namespace`

* 进程可以拥有自己的 `rootfs`（就像 `chroot` 一样）
* 进程还可以拥有自己的 *私有* 的挂载
	* `/tmp` （每用户、每服务）
	* `/proc` 和 `/sys` 的 masking
	* NFS 自动挂载之类的
* 挂载可以是完全私有，也可以是共享的
* 没有简单的办法可以将一个挂载转移到另一个命名空间

## `uts namespace`

* `gethostname` / `sethostname`

## `ipc namespace`

* 有人知道什么是 `IPC` 么？
* 有人关心 `IPC` 么？
* 运行进程（或一组进程）可以拥有自己的：
	* `IPC` 信号量
	* `IPC` 消息队列
	* `IPC` 共享内存
* 而不会和其它的命名空间的进程发生冲突

## `user namespace`

* 允许映射 `UID/GID`，比如
	* 容器 `C1` 内的 `UID 0 → 1999` 对应到宿主的 `UID 10000 → 11999`
	* 容器 `C2` 内的 `UID 0 → 1999` 对应到宿主的 `UID 12000 → 13999`
* 避免容器内的额外配置
* 这样还可以让 `UID 0 (root)` 变成宿主一个非特权的普通用户
* 可以提高安全性
* **但是**：细节是魔鬼……

## `namespace` 操作

* `namespace` 由 `clone()` 系统调用创建
* `namespace` 的具体表现为伪文件形式：
	* `/proc/<pid>/ns`
* 当命名空间中最后一个进程退出时，命名空间即被销毁。
	* 但是可以保留绑定挂载的伪文件
* 可以用 `setns()` 来*进入*一个命名空间
	* 比如 `util-linux` 中的 `nsenter` 命令。

# 容器基石 - 3：`copy-on-write`

## `copy-on-write` 存储

* 可以即时创建容器（无需复制整个文件系统）
* 存储可以记录有哪些内容发生了改动
* 有很多可选择的途径：
	* 文件层面：`AUFS`, `overlay`
	* 块层面：`device mapper`
	* 文件系统层面：`btrfs`, `zfs`
* 显著的降低了系统开销以及启动时间

这是非常重要的基石之一，这里无法展开讲，单单这个问题就可以讲一整节课。进一步的信息可以看这篇博文[《深入 Docker 存储驱动》](http://jpetazzo.github.io/assets/2015-07-01-deep-dive-into-docker-storage-drivers.html#1)

# 其它细节

## 正交性

* 所有之前提到的东西都是互相独立的
* 如果只需要资源隔离，只要用几个 `cgroups` 就可以
* 可以用网络命名空间模拟路由网络
* 让 debugger 运行于容器的命名空间，但是却不在其 cgroups 里，这样不受资源限制约束
* 在一个隔离的环境中设置网络接口，然后移动到另一个命名空间中。
* 等等

## Capabilities

* 可以将  `root` / `非root` 这种分类进行进一步的细粒度的权限控制。
* 允许保留 `root`，但是不提供危险的操作的能力
* 但是 `CAP_SYS_ADMIN` 依旧是什么能力都有，需要注意

## SELinux / AppArmor

* 让容器确实可以严格的约束限制住其内的进程
* 这是一个复杂的话题
* 可以看一下这个 Github 项目 https://github.com/jessfraz/bane

# 使用 `cgroups` 和 `namespaces` 的容器运行时

## LXC

* 一套用户态的工具
* 一个容器就是 `/var/lib/lxc` 下的一个目录
* 一个简单的配置文件 + 根文件系统
* 早期版本不支持 CoW
* 早期版本不支持移动镜像
* 需要很复杂的准备工作
	* 对系统管理员(Ops) 可能比较简单，但是对于开发人员(Devs) 则比较复杂。

## systemd-nspawn

* 按照其 manpage 的说法：
	* “用于调试、测试和构建”
	* “和 `chroot` 相似，但是更强大”
	* “实现了容器接口”
* 貌似是将自己视为中间件接口
* 最近增加了*docker镜像*的支持（竟然不敢正大光明的提及 docker 名字）

```c
#define INDEX_HOST "index.do" /* the URL we get the data from */ "cker.io"
#define HEADER_TOKEN "X-Do" /* the HTTP header for the auth token */ "cker-Token:"
#define HEADER_REGISTRY "X-Do" /*the HTTP header for the registry */ "cker-Endpoints:"
```

## Docker Engine

* 引擎由 REST API 控制
* docker 最初的版本是使用 LXC，而现在使用自己的 `libcontainer` 运行时。（而2016年底则使用开放标准的 runC 了）
* 管理容器、镜像、构建以及更多的东西
* 一些人认为它做的太多了，应该少而精。

## rkt, runC

* 回到更基础的东西
* 只关注于容器执行
	* 没有 API、没有镜像管理、没有镜像构建等等
* `rkt` 和 `runC` 实现了不同的标准：
	* `rkt` 实现的是 `appc` (App Container) 标准
	* `runC` 实现的是 `OCP` (Open Container Project) 标准
	* `runC` 是利用了 Docker 的 `libcontainer`

## 哪一个最好？

* 首先需要强调的是，它们全都是使用同一套内核功能。所以性能上它们完全一样。
* 所以需要观察其*功能*、*设计*、*生态系统*。

# 使用其它机制实现的容器运行时

## OpenVZ

* 当然，这还是 Linux
* 更古老一些，但是是经过了实战检验的
	* Travis CI 中如果使用 `root`，就会给你建立一个 `OpenVZ` 的容器/虚拟机
* 还有一堆很不错的功能
	* `ploop` （对容器而言更有效的块设备）
	* checkpoint/restore，热迁移
	* `venet`（更有效率的 veth）
* 还在开发和维护

## Jails / Zones

* 这是 FreeBSD / Solaris 特有的
* 相对于 `namespace` 和 `cgroups` 而言，是更粗粒度的控制
* 重点强调的是**安全性**
* 很适合托管主机提供商
* 对开发者来说意义不大
	* 没有等同于 `docker run -it ubuntu` 的东西
* [Illumos](https://wiki.illumos.org/display/illumos/illumos+Home) （OpenSolaris 的社区 fork）可以运行 Linux 可执行文件
	* 用 [Joyent Triton](https://www.joyent.com/blog/triton-docker-and-the-best-of-all-worlds)
* 而 Oracle 的 Solaris 则只可以运行 Solaris 可执行文件，不可以运行 Linux 可执行文件

# 如何手动建立容器

*仅为教育目的😏*

这里使用 `btrfs` 来进行 CoW 的存储层管理，利用其 subvolume 和 snapshot 机制。

先确保挂载点都是 `rprivate`，否则容器内的挂载如果污染了宿主的挂载就乱套了。

```bash
mount --make-rprivate /
```

然后创建镜像、容器的目录，并且创建 btrfs 的 subvolume，并且填充入 alpine 的镜像内容。

```bash
# 创建镜像、容器的目录
$ mkdir -p images containers
# 创建 btrfs 的 subvolume
$ btrfs subvol create images/alpine
Create subvolume 'images/alpine'
$ btrfs subvol list .
ID 257 gen 8 top level 5 path images/alpine
# 填充入 alpine 的镜像内容，这里直接借用 docker 镜像 rootfs
$ CID=$(docker run -d alpine true)
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
0a8490d0dfd3: Pulling fs layer
0a8490d0dfd3: Verifying Checksum
0a8490d0dfd3: Download complete
0a8490d0dfd3: Pull complete
Digest: sha256:dfbd4a3a8ebca874ebd2474f044a0b33600d4523d03b0df76e5c5986cb02d7e8
Status: Downloaded newer image for alpine:latest
$ echo $CID
434cf134e44ee47211fab5c2da31de2ec13a22d9ed2a3e3da2a9831d7585d71e
$ docker export $CID | tar -C images/alpine/ -xf-
$ ls images/alpine/
bin  etc   lib    mnt   root  sbin  sys  usr
dev  home  media  proc  run   srv   tmp  var
```

以镜像 subvolume 为基础，使用 snapshot 创建容器 tupperware。

```bash
$ btrfs subvol snapshot images/alpine/ containers/tupperware
Create a snapshot of 'images/alpine/' in 'containers/tupperware'
$ ls containers/tupperware/
bin  etc   lib    mnt   root  sbin  sys  usr
dev  home  media  proc  run   srv   tmp  var
# 为了确保可以认出这是我们的容器，建一个标志文件
$ touch containers/tupperware/THIS_IS_TUPPERWAARE
```

我们可以用 `chroot` 进入我们的rootfs，可以用 alpine 的包管理 apk 了。当然，暂时还不能称为容器。

```bash
$ chroot containers/tupperware/ sh
$ ls
THIS_IS_TUPPERWAARE  media                srv
bin                  mnt                  sys
dev                  proc                 tmp
etc                  root                 usr
home                 run                  var
lib                  sbin
$ apk
apk-tools 2.6.8, compiled for x86_64.
...
```

现在我们可以开启 namespace 了，可以利用 `unshare` 命令：

```bash
$ unshare --mount --uts --ipc --net --pid --fork bash
```

这个命令执行后，实际上已经进入容器了，但是感知不到，特别是 ps 的时候观察其 pid，还是宿主的 pid。

```bash
 $ ps
  PID TTY          TIME CMD
 1846 pts/0    00:00:00 sudo
 1847 pts/0    00:00:00 bash
 2386 pts/0    00:00:00 unshare
 2387 pts/0    00:00:00 bash
 2412 pts/0    00:00:00 ps
```

其实这些 `pid` 已经无法访问到了，比如我们试图 `kill` 其中的进程，会失败，说找不到该进程：

```bash
$ kill 2386
bash: kill: (2386) - No such process
```

因此我们需要做的是重新挂载 `/proc`，然后 `ps` 就真的显示容器内的信息了：

```bash
$ mount -t proc none /proc
$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 03:43 pts/0    00:00:00 bash
root        28     1  0 03:45 pts/0    00:00:00 ps -ef
```

接下来是文件系统，这东西依旧还是宿主，我们需要切换到 tuppwerware 下，不过这里 `pivot_root` 直接永会报错：

```bash
$ cd containers/tupperware
$ mkdir oldroot
$ pivot_root . oldroot/
pivot_root: failed to change root from `.' to `oldroot/': Invalid argument
```

解决办法是利用 bind-mount。

```bash
$ cd /
$ mount --bind /mnt/data/containers/tupperware/ /mnt/data/containers/tupperware/
$ mount --move /mnt/data/containers/tupperware/ /btrfs
$ cd /btrfs/
$ ls
THIS_IS_TUPPERWAARE  dev  home  media  oldroot  root  sbin  sys  usr
bin                  etc  lib   mnt    proc     run   srv   tmp  var
$ pivot_root . oldroot/
$ cd /
$ ls -al
total 4
drwxr-xr-x    1 root     root           180 Jan 30 03:46 .
drwxr-xr-x    1 root     root           180 Jan 30 03:46 ..
-rwxr-xr-x    1 root     root             0 Jan 30 03:32 .dockerenv
-rw-r--r--    1 root     root             0 Jan 30 03:40 THIS_IS_TUPPERWAARE
drwxr-xr-x    1 root     root           808 Dec 26 21:32 bin
drwxr-xr-x    1 root     root            34 Jan 30 03:32 dev
drwxr-xr-x    1 root     root           524 Jan 30 03:32 etc
drwxr-xr-x    1 root     root             0 Dec 26 21:32 home
drwxr-xr-x    1 root     root           278 Dec 26 21:32 lib
drwxr-xr-x    1 root     root            28 Dec 26 21:32 media
drwxr-xr-x    1 root     root             0 Dec 26 21:32 mnt
drwxr-xr-x   26 root     root          4096 Jan 30 03:50 oldroot
drwxr-xr-x    1 root     root             0 Dec 26 21:32 proc
drwx------    1 root     root            24 Jan 30 03:41 root
drwxr-xr-x    1 root     root             0 Dec 26 21:32 run
drwxr-xr-x    1 root     root           778 Dec 26 21:32 sbin
drwxr-xr-x    1 root     root             0 Dec 26 21:32 srv
drwxr-xr-x    1 root     root             0 Dec 26 21:32 sys
drwxrwxrwt    1 root     root             0 Dec 26 21:32 tmp
drwxr-xr-x    1 root     root            40 Dec 26 21:32 usr
drwxr-xr-x    1 root     root            78 Dec 26 21:32 var
$ mount -t proc none /proc
$ ps -ef
PID   USER     TIME   COMMAND
    1 root       0:00 bash
   44 root       0:00 ps -ef
```

现在看起来就更像在容器里了，有了自己的 rootfs，有了自己的 pid 空间。不过 mount 还是宿主的。我们需要卸掉。

```bash
$ umount -a
umount: can't unmount /oldroot/mnt/data: Resource busy
umount: can't unmount /oldroot: Resource busy
```

可以看到这里 `/oldroot` 不让卸载，因为设备忙。这又是一个小地方需要绕过的。

```bash
$ umount -l /oldroot/
$ mount -t proc none /proc
$ mount
/dev/dm-2 on / type btrfs (ro,relatime,space_cache,subvolid=258,subvol=/containers/tupperware)
none on /proc type proc (rw,relatime)
```

这回挂载就正常了，仅剩容器的挂载了。

接下来是网络，如果这时在容器内想访问外界网络，会发现无法实现，其原因是容器内还没有自己的网络接口呢。

```bash
$ ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
$ ifconfig -a
lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```

我们需要**新开一个终端**，在*宿主*执行命令建立容器网络接口。

```bash
# 先查到我们容器的进程 ID
$ pidof unshare
2386

# 然后添加一对 veth 接口
$ ip link add name h2386 type veth peer name c2386
$ ip link
...
6: c2386@h2386: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 86:ce:e2:44:86:e5 brd ff:ff:ff:ff:ff:ff
7: h2386@c2386: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether c6:9e:00:0e:df:4d brd ff:ff:ff:ff:ff:ff
```

这里我们称宿主的那端叫 `h2386`，容器那端叫 `c2386`。

然后我们将容器的那端置于容器命名空间内，而宿主这端连入 `docker0` 桥接网络。

```bash
$ ip link set c2386 netns 2386
$ ip link set h2386 master docker0 up
```

*切换到容器内*，我们就可以看到连入进来的网络接口了：

```bash
$ ifconfig -a
c2386     Link encap:Ethernet  HWaddr 86:CE:E2:44:86:E5
          BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          LOOPBACK  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
$ ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: c2386@if7: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN qlen 1000
    link/ether 86:ce:e2:44:86:e5 brd ff:ff:ff:ff:ff:ff
```

然后我们将容器内的接口名称设为 `eth0`，并且给定地址和网关。

```bash
$ ip link set lo up
$ ip link set c2386 name eth0 up
$ ip addr add 172.17.10.123/16 dev eth0
$ ip route add default via 172.17.0.1
```

然后我们的容器就可以上网了：

```bash
$ ip route
default via 172.17.0.1 dev eth0
172.17.0.0/16 dev eth0  src 172.17.10.123
$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=61 time=6.956 ms
64 bytes from 8.8.8.8: seq=1 ttl=61 time=6.405 ms
^C
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 6.405/6.680/6.956 ms
```

*视频中做到这一步时由于网关写错了，导致最后一步网络未能联通。*

由于时间有限，还有大量的主题在这个例子中无法演示，比如 cgroups、devices、capabilities, selinux 等等大量的问题。

所以在演示之前就提到，*仅为教育目的*。或许我们可以用过几十行的 bash 脚本就可以做一个自己的容器，但是**能这么做不等于应该这么做**。
