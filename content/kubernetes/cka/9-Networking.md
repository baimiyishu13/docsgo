+++
title = 'CKA-9： Networking'
date = 2024-05-15T15:13:55+08:00
draft = true

+++

## 先决条件

**Switching and Routing**：Switching、Routing、Default Gateway、NAT

**Linux Interfaces for Virtual Networking**：Bridge Network、VLAN、VXLAN。

**IP Address Management & Name Resolution**： DNS、IPAM、 DHCP

**Firewalls**

 **Load-Balancers**

**Tools**: Ping、 NC - NetCat、TCPDUMP、IPTABLES



### Router

![image-20240517003344051](/images/image-20240517003344051.png "Title")



### DNS 

文件：

+ /etc/hosts
+ /etc/reslof.conf

命令：nsloojup、dig



### CronDNS

CoreDNS 二进制文件可以从其 Github 发布页面下载，也可以作为 docker 镜像下载

```sh
wget https://github.com/coredns/coredns/releases/download/v1.4.0/coredns_1.4.0_linux_amd64.tgz
```

减压后访问：

```sh
tar –xzvf coredns_1.4.0_linux_amd64.tgz
./coredns
.:53
...
```

默认情况下，它侦听端口 53，这是 DNS 服务器的默认端口。



## Network Namespaces

Docker 等容器使用 Namespace 来实现网络隔离。

容器是使用 Namespace 与底层主机分离。

### 进程命名空间

创建容器时，希望确保该容器是隔离的

+ 使用 Namespace 在主机上创建了一个特殊的空间，
+ 就容器而已，它只能看到它运行的进程，并认为它是独立的。

底层主机可以看到所有进程，包括容器内运行的进程

当查看容器内进程：

```sh
# ps aux
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
root 1 0.0 0.0 4528 828 ? Ss 03:06 0:00 nginx
```

查看主机进程时，会看到所有其他容器中运行的进程：

```sh
ps aux
USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND
project 3720 0.1 0.1 95500 4916 ? R 06:06 0:00 sshd: project@pts/0
project 3725 0.0 0.1 95196 4132 ? S 06:06 0:00 sshd: project@notty
project 3727 0.2 0.1 21352 5340 pts/0 Ss 06:06 0:00 -bash
root 3802 0.0 0.0 8924 3616 ? Sl 06:06 0:00 docker-containerd-shim -namespace m
root 3816 1.0 0.0 4528 828 ? Ss 06:06 0:00 nginx
```

是在容器内外以不同的进程ID运行的同一进程，这就是 Namespace 的工作原理。



### 网络命名空间

主机有自己的接口，可以连接局域网

+ 有自己的路由表和 ARP表

创建容器时，为其创建一个网络 Namespace ，这样它就无法查看主机上的如何网络信息。

+ 容器有自己的虚拟接口、路由、ARP表

**创建网络NS**

```sh
ip netns add red
ip netns add blue
```

查看:

```sh
✖ ip netns add red
➜  ip netns ls
red
cni-bb47fc98-77d9-c49b-35b2-eaad1c410985 (id: 2)
cni-d9cb37c5-3603-29cd-9f8c-8da6bd682bc2 (id: 1)
```

列出主机上 接口

```sh
ip link  
```

ns 中列出

```sh
ip -n red link
ip netns exec red ip link 
```

通过 NS 成功阻止了容器看到主机接口。

查看ARP

```sh
controlplane ~ ✖ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.17.0.2               ether   36:6b:28:cc:c9:87   C                     cni0
172.17.0.3               ether   86:a4:e2:67:19:d0   C                     cni0
172.17.1.0               ether   32:1f:33:3d:6c:32   CM                    flannel.1
169.254.1.1              ether   ee:ee:ee:ee:ee:ee   C                     eth0
10-130-0-29.calico-typh  ether   ee:ee:ee:ee:ee:ee   C                     eth0

controlplane ~ ➜  ip netns exec red arp
null
```

两个命名空间可以使用 隧道/管道技术连接

```sh
➜  ip link add veth-red type veth peer name veth-blue
```

将接口加入到 NS

```sh
 ip link set veth-red netns red
 ip link set veth-blue netns blue
 ip -n blue addr add 192.168.15.2 dev veth-blue
 ip -n red addr add 192.168.15.1 dev veth-red
```

开启

```sh
ip -n blue link set veth-blue up
ip -n red link set veth-red up
```

 ![image-20240517105550847](/images/image-20240517105550847.png "Title")



##  Docker网络

安装了Docker的服务器，主机具有本地网络接口

**None 默认**

网络为 “none” ，Docker 容器不会连接到如何网络，无法到达外部，任何人也无法进入无法通行。

**hosts 默认**

如果运行了一个容器映射了一个80端口，无需执行如何其他端口映射，直接通过主机网络进行访问。

+ 共享主机网络，不能有端口冲突

**BRIDGE 桥接**

将在内部创建一个专用网络，Docker 主机和容器都连接到该网络

在安装docker 时，默认会创建一个 bridge

+ docker 0



## 容器网络接口CNI

网络命名空间的原理：如何创建

1. 创建网络命名空间
2. 创建桥接网络/接口
3. 创建 VETH 对（管道、虚拟电缆）
4. 将 vEth 附加到命名空间
5. 将其他 vEth 连接到网桥
6. 分配IP地址
7. 调出接口
8. 启用 NAT – IP 伪装

创建一个程序来完成这些工作

CNI 是一组标准，定义了应用如何开发程序以解决容器运行时环境中的网络挑战。

CNI 标准：

+  容器运行时必须创建网络命名空间
+ 识别容器必须连接到的网络
+ 添加容器时，容器运行时调用网络插件（桥）。
+ 当容器被删除时，容器运行时调用网络插件（桥）。
+ JSON格式的网络配置
+ 必须支持命令行参数ADD/DEL/CHECK
+ 必须支持容器id、网络ns等参数。
+ 必须管理 POD 的 IP 地址分配
+ 必须以特定格式返回结果

第三方插件：

+ Weaveworks
+ Flannel
+ Cilium
+ VMware NSX
+ calico

注意：docker 不实现 CNI，docker 有一套自己的 CNM

🔔 kubernets 创建时它会在 none 网络上创建它们，然后调用配置的 CNI 插件负责其余的配置。

![image-20240517130736337](/images/image-20240517130736337.png "Title")



## CLuster 网络

kubenrets 中集群主节点和工作节点所需的网络配置。

 ![image-20240517131011502](/images/image-20240517131011502.png "Title")

在集群中需要打开的端口：[访问](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)

可以使用 ns 命令检查：

```sh
nc 127.0.0.1 6443 -v
```

防火墙需放通端口：[访问](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)



## Pod 网络

确保主机配置了防火墙和网络安全组以允许 Kubernetes 控制组件相互访问

+ 每个 POD 都应该有一个 IP 地址
+ 每个POD 应该能够与同一节点中的所有其他POD 进行通信。
+ 每个POD 都应该能够在没有NAT 的情况下与其他节点上的每个其他POD 进行通信。

 ![image-20240517132339938](/images/image-20240517132339938.png "Title")



## CNI in Kubernets

CNI 根据 runtime 定义容器运行时的任务。

Kubernets 负责创建容器网络 Namespac， 通过调用 CNI 插件识别 Namespace并将其附加到正确的网络。

✓ 容器运行时必须创建网络命名空间
✓ 识别容器必须连接的网络
✓ 添加容器时，容器运行时调用网络插件（桥）。
✓ 当容器被删除时，容器运行时调用网络插件（桥）。
✓ JSON 格式的网络配置



**Config** - kubelet.service

```sh
ExecStart=/usr/local/bin/kubelet \\
--config=/var/lib/kubelet/kubelet-config.yaml \\
--container-runtime=remote \\
--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
--image-pull-progress-deadline=2m \\
--kubeconfig=/var/lib/kubelet/kubeconfig \\
--network-plugin=cni \\
--cni-bin-dir=/opt/cni/bin \\
--cni-conf-dir=/etc/cni/net.d \\
--register-node=true \\
--v=2
```

查看 kubelet 选项

```sh
ps -aux | grep kubelet
ls /opt/cni/bin
ls /etc/cni/net.d
cat /etc/cni/net.d/10-bridge.conf
```



## WeaveWorks CNI

可以定制的 CNI 脚本并继承到了 Kubelet

手动设置的网络解决方案有一个路由表，映射了那些网络位于那些主机上。

+ 当一个数据包发送到另一端时，会查找路由比

仅适用于小型环境和简单的网络

weaveworks 解决方案

+ CNI的插件部署在每个集群上
+ 彼此之间通信交换关于它们内的节点网络和Pod信息，这样便知道了其他节点的Pod及IP



## IP 地址管理编织

CNI - 定义标准

网络提供商负责分配IP的容器

kubernetes 不在乎这些，都是需要提供CNI插件去做

+ 最简单的方式就是放在一些列表中，有脚本去管理它

CNI 提供了两个内置插件，可以将此任务分配给他们，而不是自己脚本中写代码



