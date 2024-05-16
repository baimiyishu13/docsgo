+++
title = 'cka-2: 核心概念'
date = 2024-05-15T15:13:55+08:00
draft = true

+++

​		这将是第一部分，将从较高层次了解集群体系结构，然后了解一些API，例如 Pod、Replicaset、deployment等

# 核心概念

## 集群架构

### 基本概述

Kubernetes 集群由一组节点组成，可以是 物理节点、虚拟节点、本地节点、云节点等。

以容器的方式托管应用程序

+ worker nodes：运行容器
+ master worker：负责管理 Kuberntes 集群、存储有关不同节点信息、规划容器去向、监控节点及其上的容器等。

master 节点使用一组控制组件完成这些操作

+ Etcd 集群：记录集群中所有信息

+ Kube-scheduler：调度器基于容器资源要求，工作节点容量或者任何其他策略或约束（例如，污点、容忍、亲和性规则）来识别容器放置的正确节点。

+ Controller：控制器处理不同的区域，

  + 节点控制器：负责管理节点、将节点加载到集群、处理节点变得不可以或者被破坏等情况。
  + Replicaset 控制器：负责确保始终运行所需数量的容器。

  【说明】现在已经有这么多组件，谁对它们进行高级管理？

+ kube APIserver：负责协调集群中所有操作，公开了API（外部用户使用该API在集群上执行管理操作）以及各种控制器（用于监控集群状态并根据需要进行必要更改） 和 工作节点（用于与服务器通信）

+ Containerd：集群中所有节点安装，CRI支持的引擎

+ Kubelet：

  + 负责与 kube-Apiserver 通信，收集节点信息和节点上Pod状态等等。
  + 每个节点上运行的代理，监听KubeAPIServer指令，并根据需要在节点上部署或者销毁容器。
  + KubeAPIServer 定期从Kubelet 获取状态报告，以监视节点和容器状态

  【说明】Kubelet 负责管理Pod、但在工作节点上运行的应用程序需要能够相互通信

+ Kube-Proxy：确保了工作节点上有必要的规则以运行这些节点上运行的容器相互访问

+ CoreDNS：负责集群中域名解析

 

### ETCD 介绍

#### what is ETCD？

​		一种分布式可靠的键值存储、简单、安全且快速

#### what is 键值存储 ？

​		传统中 数据库采用 表格格式，SQL 或关系型数据库。

![0](/images/image-20240515173954950.png "Title")  

#### key-value 

+ 通常通过 JSON 或 YAML等数据格式进行处理

![1](/images/image-20240515174126219.png "Title")

#### Install ETCD

1. 下载

```sh
curl -L https://github.com/etcd-io/etcd/releases/download/v3.3.11/etcd-v3.3.11-linux-amd64.tar.gz -o etcd-v3.3.11-linux-amd64.tar.gz
```

2. 减压

```sh
tar xzvf etcd-v3.3.11-linux-amd64.tar.gz
```

3. Run ETCD Service

```sh
./etcd
```

默认情况下会启动监听 2379端口的服务，可以连接到 ETCD服务存储和检索信息

ETCD 附带的默认客户端：etcdctl

```sh
./etcdctl set key1 value1
./etcdctl get key1
```

#### ETCD 版本

生产环境中可能遇到不同版本的 ETCD，了解ETCD版本历史非常重要

v0.1: 2013年8月发布

v0.5: 官方稳定版本，发布在2015年，这是RAFT算法被重新设计的时候，支持每秒1万次写入

v2.0: 2015年发布

v3.0: 2017年发布，包含更多的优化和性能改进

+ 2018年，在CNCF中孵化

最重要的是 v2.0和 v3.0的变化，API发生了改变，意味着 etcdctl 发生了变化

查看版本：

```sh
✖ ./etcdctl --version
etcdctl version: 3.3.11
API version: 2
```

对于较新的 ETCD API版本为3，因此使用前看下 API版本

使用 API v3，在运行命令前，每个命令的环境变量 `ETCDCTL_API` 设置为 3

```sh
✖ export ETCDCTL_API=3
➜  ./etcdctl version
etcdctl version: 3.3.11
API version: 3.3
```

在 v3 中 ，查看版本是 `version` 而不是 `--version`



### Kubernetes 中的 ETCD

etcd 存储有关集群的信息，例如：

+ nodes、Pods、Configs、Secrets、Accounts、Roles、Bindings、Others

运行 `kubectl get` 看到的所有信息都来自 ETCD

对集群所做的所有更改都将在 ETCD 中更新，只有当 ETCD 中被更新时才能认为更改完成。

根据集群部署方式，ETCD 的部署方式也会有所不同。

+ 二进制
+ kubeadm

#### 从头开始部署设置集群

可以通过以下方式部署 ETCD：

+ 自己下载 ETCD 二进制文件

```sh
wget -q --https-only \
"https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

并且设置 Systemd启动

```sh
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
```

会有很多的传入项：

+ 证书有关
+ 需要注意的选项 `--advertise-client-urls https://${INTERNAL_IP}:2379`

当 KubeAPIServer尝试访问 ETCD 服务是应该在Kube API服务器上配置此 URL



#### 使用 Kubeadm 部署集群

kubeadm 会将 ETCD 服务器作为 Pod 部署到 kube-system 系统命名空间中

```sh
$ kubectl get pods -n kube-system
kube-system etcd-master 1/1 Running 0 1h
```

列出 kubernetes 存储的所有密钥：

```sh
$ kubectl exec etcd-master –n kube-system etcdctl get / --prefix –keys-only
/registry/apiregistration.k8s.io/apiservices/v1.
/registry/apiregistration.k8s.io/apiservices/v1.apps
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling
/registry/apiregistration.k8s.io/apiservices/v1.batch
/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
```

kubernetes 会将数据存储在特定的目录结构中，根目录是一个 `registry` , 在该目录下有各种 kubernets 结构。



#### ETCD 高可用环境

集群中会有多个主节点，会拥有多个分布在主节点上的 ETCD 实例

通过 `--initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \\` 指定 ETCD 服务的不同实例。



之后会深入研究



### KubeAPI Server

kube-apiserver集群的主要管理组件。

当 使用 kubectl 时，实际上是在访问 kube-apiserver

1. 首先  kube-apiserver 对请求进行身份验证
2. 然后   kube-apiserver 从 etcd 集群中检索数据，使用请求信息进行响应

实际上并不必须 kubectl 也可以，可以通过发送 POST 请求直接调用 API

```sh
$ curl –X POST /api/v1/namespaces/default/pods ...[other]
Pod created!
```

1. 身份认证
2. 验证请求
3. 检索数据
4. 更新 ETCD
5. 调度程序：持续监听 kube-apiserver ，并发现一个新的Pod未分配节点，确认节点后将消息传回 kube-apiserver，kube-apiserver 更新 etcd 集群中信息。
6. kubelet：kube-apiserver 将信息传递给 对应工作节点的额 kubelet，kubelet 在节点上创建Pod，并指定 Container CRI 部署应用程序镜像。一旦完成，kubelet 将状态更新回 API 服务器，然后 kube-apiserver 将数据更新回 etcd 集群

![2](/images/image-20240516134646195.png "Title")

kube-apiserver 是在集群中进行更改所需执行的所有不同任务中心

kube-apiserver  是唯一一个与 ETCD 直接交互的组件

其他组件：sechduler、kube-controller-manager、kubelet 使用 kube-apiserver 在集群中各自的区域执行更新



如果使用 kubeadm 工具部署集群，那么可能不需要知道这些

二进制安装 kube-apiserver

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-apiserver
```

kube-apiserver.service

```sh
ExecStart=/usr/local/bin/kube-apiserver \\
--advertise-address=${INTERNAL_IP} \\
--allow-privileged=true \\
--apiserver-count=3 \\
--authorization-mode=Node,RBAC \\
--bind-address=0.0.0.0 \\
--enable-admission-
plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,Reso
urceQuota \\
--enable-swagger-ui=true \\
--etcd-servers=https://127.0.0.1:2379 \\
--event-ttl=1h \\
--experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
--runtime-config=api/all \\
--service-account-key-file=/var/lib/kubernetes/service-account.pem \\
--service-cluster-ip-range=10.32.0.0/24 \\
--service-node-port-range=30000-32767 \\
--v=2
```

很多参数

如果使用的是 kubeadm 部署的集群

```sh
$ cat /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
containers:
- command:
- kube-apiserver
- --authorization-mode=Node,RBAC
- --advertise-address=172.17.0.32
- --allow-privileged=true
- --client-ca-file=/etc/kubernetes/pki/ca.crt
- --disable-admission-plugins=PersistentVolumeLabel
- --enable-admission-plugins=NodeRestriction
- --enable-bootstrap-token-auth=true
- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
- --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
- --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
- --etcd-servers=https://127.0.0.1:2379
- --insecure-port=0
- --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
- --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
- --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
- --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
- --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
- --requestheader-allowed-names=front-proxy-client
- --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
- --requestheader-extra-headers-prefix=X-Remote-Extra-
- --requestheader-group-headers=X-Remote-Group
- --requestheader-username-headers=X-Remote-User
```

如果是二进制安装的

```sh
$ cat /etc/systemd/system/kube-apiserver.service
[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
--advertise-address=${INTERNAL_IP} \\
--allow-privileged=true \\
--apiserver-count=3 \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/var/log/audit.log \\
--authorization-mode=Node,RBAC \\
--bind-address=0.0.0.0 \\
--client-ca-file=/var/lib/kubernetes/ca.pem \\
--enable-admission-
plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,Defa
ultStorageClass,ResourceQuota \\
--enable-swagger-ui=true \\
--etcd-cafile=/var/lib/kubernetes/ca.pem \\
--etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
--etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
--etcd-
servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
--event-ttl=1h \\
--experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml
\\
--kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
--kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
```

还可以通过：

```sh
ps -aux | grep kube-apiserver
```



### Kube-Controller-Manager



Kube-Controller-Manager 控制器是管理 kubernets 中的各种控制器

持续监控系统内各个组件的状态，使整个系统达到所需的功能状态，在出现问题的时候作出补救。

都是通过 kuber apiserver，

例如：Node Controller

+ 监视 node的状态，并采取必要的动作保持应用程序运行
+ 5s 进行一次检测健康状况
+ 如果停止接受心跳，会等待 40s，然后将其标记为不可访问。
+ 不可访问后会有 5分钟的时间来恢复，如果没有它将删除分配给改 Node 的Pod。

Resplices 控制器：

+ 监视副本集的状态，确保副本集始终有所需数量的 Pod 可用

这只是两个例子，像这样的控制器有很多。

像deploymen、job、svc、sa、pv、pvc、namespace、等等都是通过不同的控制器实现的。



位于集群位置？

+ 它们被打包成到一个 Kubernetes Controller Manager 的 进程中，会安装不同的控制器。

安装：二进制

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-controller-manager
```

kube-controller-manager.service

```sh
[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
--address=0.0.0.0 \\
--cluster-cidr=10.200.0.0/16 \\
--cluster-name=kubernetes \\
--cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
--cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
--kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
--leader-elect=true \\
--root-ca-file=/var/lib/kubernetes/ca.pem \\
--service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
--service-cluster-ip-range=10.32.0.0/24 \\
--use-service-account-credentials=true \\
--v=2
Restart=on-failure
RestartSec=5
```

默认情况下所有的 选项都是启动状态

```sh
--controllers stringSlice Default: [*]
A list of controllers to enable. '*' enables all on-by-default controllers, 'foo' enables the controller
named 'foo', '-foo' disables the controller named 'foo'.
All controllers: attachdetach, bootstrapsigner, clusterrole-aggregation, cronjob, csrapproving,
csrcleaner, csrsigning, daemonset, deployment, disruption, endpoint, garbagecollector,
horizontalpodautoscaling, job, namespace, nodeipam, nodelifecycle, persistentvolume-binder,
persistentvolume-expander, podgc, pv-protection, pvc-protection, replicaset, replicationcontroller,
resourcequota, root-ca-cert-publisher, route, service, serviceaccount, serviceaccount-token, statefulset,
tokencleaner, ttl, ttl-after-finished
Disabled-by-default controllers: bootstrapsigner, tokencleaner
```



安装方式：kubeadmin

```sh
$ kubectl get pods -n kube-system
kube-system kube-controller-manager-master 1/1 Running 0 15m
```

配置文件：

```sh
$ cat /etc/kubernetes/manifests/kube-controller-manager.yaml
spec:
containers:
- command:
- kube-controller-manager
- --address=127.0.0.1
- --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
- --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
- --controllers=*,bootstrapsigner,tokencleaner
- --kubeconfig=/etc/kubernetes/controller-manager.conf
- --leader-elect=true
- --root-ca-file=/etc/kubernetes/pki/ca.crt
- --service-account-private-key-file=/etc/kubernetes/pki/sa.key
- --use-service-account-credentials=true
```

还可以通过

```sh
ps -aux | grep kube-controller-manager
```



### Kube-Scheduler

调度器 只决定哪个 Pod 在哪个节点上运行。

实际上并没有将Pod放置在节点上，这是 Kubelet 的工作，调度程序只决定 Pod去哪里。

1. 排除不符合调度的节点
2. 优先级函数为节点分配分数

可以写自己的调度程序，还需要了解

+ 资源请求、限制、污点、容忍、亲和、node selecter、nodename 等



安装：

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-scheduler
```

kube-scheduler.service

```sh
ExecStart=/usr/local/bin/kube-scheduler \\
--config=/etc/kubernetes/config/kube-scheduler.yaml \\
--v=2
```

kube-scheduler.yaml

```yml
$ cat /etc/kubernetes/manifests/kube-scheduler.yaml
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
```



### Kubelet

Kubelet 领导 Node的一切活动，master 是它们的唯一联络点

定期向 Kube-APIServer 返回节点信息及Pod信息

kubernetes 中的工作节点中的 kubelet 将节点注册到 Kubernets 集群，当 kubelet 收到 加载容器或Pod的指令，它请求 CRL 拉取所需镜像并运行实例。

kubelet 持续监控 Pod 和其中容器的状态，并及时向 Kube-apiserver  服务器报告。

安装：

kubeadm 不会自动安装 kubelet，必须在工作节点手动安装。

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kubelet
```

kubelet.service

```sh
ExecStart=/usr/local/bin/kubelet \\
--config=/var/lib/kubelet/kubelet-config.yaml \\
--container-runtime=remote \\
--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
--image-pull-progress-deadline=2m \\
--kubeconfig=/var/lib/kubelet/kubeconfig \\
--network-plugin=cni \\
--register-node=true \\
--v=2
```



### Kube-Proxy

默认情况下，在 Kubernets 中 每个Pod都能到达其他 Pod，这是通过 Pod的 网络解决方案部署到集群实现的。

Pod Network：内部的虚拟网络，跨越集群中所有单元连接到所有节点，通过这个网络它们能够相互交流，有许多的解决方案。

因为无法保证 Pod的IP始终保持不变，使用 service 事更好的方法，可以使用 SVC 名称访问数据库。

Kube-Proxy 时在每一个节点上运行的进程，它的工作是寻找新服务，每次创建新服务时，会在每个节点上创建适当的规则。

方法：

1. iptables 规则
2. ipvs 规则

如果使用 cilium 将不会有 kube-proxy 组件，因为 cilium 使用 EBPF 内核技术。

安装：二进制

```sh
wget https://storage.googleapis.com/kubernetes-release/release/v1.13.0/bin/linux/amd64/kube-proxy
```

kube-proxy.service

```sh
ExecStart=/usr/local/bin/kube-proxy \\
--config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5
```



安装：kubeadm

```sh
$ kubectl get pods -n kube-system
kube-system kube-proxy-lzt6f 1/1 Running 0 16m
```



## Pod

### 回顾 Pod

最终目标是以容器的形式将应用程序部署在工作节点上

Pod 通常和 运行的应用程序的容器具有一对一关系。

Multi-Container PODs

+ 一个Pod中可以有多个容器
+ 例如 为 web 服务执行某种支持任务，处理用户输入、输出数据、处理用户上传的文件等。
+ 共享相同的网络 Namespace 空间：通过 localhost 通信
+ 共享存储



简单的 Docker 容器：

+ `docker run python-app`
+ 当负载增加是：需要运行多次 `docker run`
+ 需要自定义网络，容器之间建立网络连接



在 Kubermetes 中，只需要定义一组包含哪些容器。

Pod 可以使应用程序适应未来的架构和更改和扩展。

部署：kubectl

```sh
kubectl run nginx –-image nginx
```



### Pods with Yaml

kubernets yaml 文件始终包含：

+ apiVersion：创建对象的 Kubernets API的版本
+ kind：对象类型
+ metadata: 对象名称、标签等
+ spec

```yml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

这些都是顶级属性，必须字段

```sh
$ kubectl get pod -n ns
$ kubectl describe pod -n ns podname
```

创建：

```sh
controlplane ~ ➜  kubectl apply -f nginx.yml
pod/nginx created
controlplane ~ ➜  kubectl get pod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15s
```



### 练习 - Pods

1. 查看系统中的Pod （default）

```sh
$ kubectl get pod
```

2. 创建一个 Nginx Pod

```sh
$ kubectl run nginx --image=nginx
```

3. 查看创建Pod的Image

```sh
✖ kubectl describe pod nginx | grep -i image:
    Image:          nginx
```

4. Pod 在哪个节点

```sh
 ✖ kubectl get pod nginx -oyaml | grep -i nodename
  nodeName: node01

➜  kubectl get pod -owide
```

5. Pod的状态

```sh
➜  kubectl get pod
➜  kubectl describe pod nginx | grep -i state:
    State:          Running
```

6. 删除Pod

```sh
➜  kubectl delete pod nginx
```

7. 创建一个Pod redis image:redis123 的yml文件

```sh
 ➜  kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yml
```

8. 修改 redis image 

```sh
✖ kubectl edit pod redis
✖ kubectl set image pod/redis redis=redis:6.0

✖ vi redis.yml 
➜  kubectl apply -f redis.yml
```



### 回顾 replication

Replication Controller ： 复制控制器

#### 高可用性

+ 如果由于某种原因，程序奔溃，Pod失败了，用户将无法访问。所以我们希望运行多个实例或Pod。如果一个故障，我们仍然可以使用另一个。
+ 即使只有一个Pod，Replication 也会在出现故障时自动启动一个新的Pod

帮助在集群中运行多个实例，从而提供高可用性



#### LB 和 扩展

创建多个Pod以在它们之间共享负载

跨越集群中多个节点

![3](/images/image-20240516182031605.png "Title")

`Replication Controller` VS `Replica Set`

复制器 和 副本集：两个相似的术语

+ 复制器是正在被 副本集取代的较旧技术,存在细微的差异

```yml
✖ kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml
spec:
  replicas: 3
➜  kubectl get replicasets.apps 
NAME               DESIRED   CURRENT   READY   AGE
nginx-7854ff8877   3         3         3       15s
```

查看API 版本

```sh
✖ kubectl explain replicaset
GROUP:      apps
KIND:       ReplicaSet
VERSION:    v1
```



## Deployment

滚动更新：在升级Pod版本时希望无缝升级

```sh
✖ kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

**创建 NGINX Pod**

```
kubectl run nginx --image=nginx
```

**生成 POD 清单 YAML 文件 (-o yaml)。不要创建它（--dry-run）**

```
kubectl run nginx --image=nginx --dry-run=client -o yaml
```

**创建部署**

```
kubectl create deployment --image=nginx nginx
```

**生成部署 YAML 文件 (-o yaml)。不要创建它（--dry-run）**

```
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

**生成部署 YAML 文件 (-o yaml)。不要使用 4 个副本 (--replicas=4) 创建它(--dry-run)**

```
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml
```

**将其保存到文件，对文件进行必要的更改（例如，添加更多副本），然后创建部署。**

```
kubectl create -f nginx-deployment.yaml
```

**或者**

**在 k8s 1.19+ 版本中，我们可以指定 --replicas 选项来创建具有 4 个副本的部署。**

```
kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml
```



## Service

kubernetes Service 支持应用程序内部和外部的各种组件之间的通信。

![4](/images/image-20240516223758894.png "Title")



**Service Type**

+ Node Port
+ ClusterIP
+ LoadBalancer



### **Node Port**

通过将节点上的端口映射到 Pod 上的端口来实现访问

![5](/images/image-20240516224346020.png "Title")

TargetPort：目标端口，应用程序本身端口，请求会转发到该端口

Port：Service 端口

Node Port：节点上的端口

最后，还需配置 `selector` 去标签匹配 Pod





### Cluster IP

kubernets 环境中，完整的 Web 应用程序通常包含：前段、后端、数据库

服务或层之间建立连接的正确方法：

![6](/images/image-20240516231528433.png "Title")



### LoadBalancer

Node Port 有助于从节点上接受流量，并将流量路由到相应的Pod。

访问结果就是 所有NodeIP: nodeport

少数节点还好，如果是有很多节点，虽然可以访问，但是这是有问题的。

 给用户提供的需要是一个域名：w w w.***.com

方法：

+ 创建一个 LB



## 命令式与声明式

声明式方法是指定要做什么，而不是如何做

提供基础设施的命令性方法的一个例子是：逐步编写命令，例如安装服务。

声明式中：声明需求

