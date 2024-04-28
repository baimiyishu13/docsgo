+++
title = '第1章 Kubernetes核心概念'
date = 2024-04-26T22:44:23+08:00
draft = true
+++

重新学习Kubernetes ，弥补遗漏知识。



常用组件：Pod、service、ingress、configmap、secret、volume、deployment、statefulset、daemonset

### 容器编排

容器编排工具的任务？

从时间顺序上该问题：

+ 微服务的兴起导致了容器技术使用的增加，容器技术提供了一个完美的主机
+ 在都个主机环境使用脚本或者自制的工具管理会非常麻烦，这种情况下出现了对容器编排的需求。

所以kubernetes 所做的就是

1. 高可用性：高可用意味着：程序没有停机时间，总是可以让用户访问
2. 扩展性：快速扩展程序 【动态调整负载】
3. 灾难恢复，基础设施出现问题、数据丢失、服务器崩溃等，应用程序不会丢失任何数据。

### 核心组件

每个节点

+ container runtime
+ kubelet
+ kube-proxy：service本身就是LB，捕捉请求指向应用程序

master：

+ api server
+ 





### Pod

Pod：抽象概念，Pod的作用基本是就是创建容器运行的环境。或在容器上面的一个层。



Pod之间通信：

+ k8s 的CNI 提供了 虚拟网络，每个Pod都会有一个IP。
+ 这个IP是会变化的。【容器奔溃重启等等原因】会分配一个新的IP。



### Service

service：一个静态的IP或永久的地址，可以连接到每个通过标签匹配到的Pod。

好处：service 和 Pod的生命周期不相互连接。【即使Pod死亡，Service 和 它的IP也会保持不变】



显然接下来希望应用程序通过外部访问

Ingress 代替 NodePort



### Volume

外部配置：configmap

加密配置：secret，使用base64，实际上并不能加密，所以会使用第三方工具。



如果希望数据或日志数据可靠的存储下来，可以做啊都这件事的组件：Volume

存储：可以是节点磁盘，也可以是远程存储 。



### 工作负载

deployment

statefulset：部署数据库、有状态服务

+ 使用有状态部署数据库在k8s集群，会有性能等问题，所以使用集群外托管数据库是一种常见的做法。

 demonset：每个节点部署一个Pod，负责代理、日志收集、守护进程等。



