+++
title = 'cka-3: Scheduling'
date = 2024-05-15T15:13:55+08:00
draft = true

+++

**🎉 研究 Kubernets 中与调度相关的各种概念**



## 手动调度

如果集群中没有调度，可能是不想使用默认的调度器，而是想自己调度Pod

例子开始：pod-definition.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
	labels:
		name: nginx
spec:
	containers:
 	- name: nginx
  	image: nginx
		ports:
			- containerPort: 8080
	nodeName: node02
```

nodeName：每个Pod都有一个 nodeName 字段，默认情况下未设置该字段。创建Pod  yaml 文件时，通常不会指定该字段。

+ kubernets 会自动添加

调度程序将查找未设置该属性的Pod，通过调度算法选择正确的节点

如果没有调度程序，调度 Pod 最简单的方式是在创建 Pod 是将`nodeName`字段设置为 想调度到的节点名称。



## 标签和选择器

如何指定标签：

+ 在 `metadata/labels` 下指定

```yml
apiVersion: v1
kind: Pod
metadata:
	name: nginx
	labels:
		app: app1
		function: nginx
spec:
	containers:
```

使用标签搜索 Pod

```sh
kubectl get pods --selector app=app1
```

Kuberntes 内部使用 标签选择器将不同的对象连接在一起。

定义了Pod，然后 replicaset/ svc 等等使用选择器对Pod分组，



**Annotations**

记录其他详细的信息，例如工具名称、版本、联系人信息、电话、邮件等等



## 污点和容忍度

详读：https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/taint-and-toleration/

记住：污点和容忍度只是为了限制节点接受某些Pod。

记住：污点和容忍不告你 Pod 去哪个节点，告诉你节点应该接受特定容忍度的Pod

记住：配置 nodeName 会绕过调度器，如果节点上存在 NoExecute 则会被驱逐 除非有容忍  ，如果是NoSchedule 则不会。

1. 节点上有污染
2. Pod对这种污染的容忍度

污点和容忍差用于设置节点上可调度的 pod 的限制。

假设需要在 node1 上运行特殊的应用程序

1. 首先设置污点，放置所有Pod都被调度到该节点（默认情况下，Pod没有容忍）
2. 指定那些Pod 可以容忍这些污点

node 添加污染：

```sh
# kubectl taint nodes node-name key=value:taint-effect
kubectl taint nodes node1 app=myapp:NoSchedule
```

删除污染：

```sh
kubectl taint nodes node1 app=myapp:NoSchedule -
```

- `NoExecute`

  这会影响已在节点上运行的 Pod，具体影响如下：如果 Pod 不能容忍这类污点，会马上被驱逐。如果 Pod 能够容忍这类污点，但是在容忍度定义中没有指定 `tolerationSeconds`， 则 Pod 还会一直在这个节点上运行。如果 Pod 能够容忍这类污点，而且指定了 `tolerationSeconds`， 则 Pod 还能在这个节点上继续运行这个指定的时间长度。 这段时间过去后，节点生命周期控制器从节点驱除这些 Pod。

- `NoSchedule`

  除非具有匹配的容忍度规约，否则新的 Pod 不会被调度到带有污点的节点上。 当前正在节点上运行的 Pod **不会**被驱逐。

- `PreferNoSchedule`

  `PreferNoSchedule` 是“偏好”或“软性”的 `NoSchedule`。 控制平面将**尝试**避免将不能容忍污点的 Pod 调度到的节点上，但不能保证完全避免。

示例：

```sh
kubectl taint nodes node1 key1=value1:NoSchedule
```

值 一 一对应

```yml
tolerations:
- key: "key1"
  operator: "Equal"
  value: "value1"
  effect: "NoSchedule"
```

`operator` 的默认值是 `Equal`。

一个容忍度和一个污点相“匹配”是指它们有一样的键名和效果，并且：

- 如果 `operator` 是 `Exists`（此时容忍度不能指定 `value`），或者
- 如果 `operator` 是 `Equal`，则它们的值应该相等。



如果仅仅是配置了污点和容忍，也不一定 Pod 会调度到具有污点的节点，其他节点没有配置污点，都是可以调度



默认污点：

当前内置的污点包括：

- `node.kubernetes.io/not-ready`：节点未准备好。这相当于节点状况 `Ready` 的值为 "`False`"。
- `node.kubernetes.io/unreachable`：节点控制器访问不到节点. 这相当于节点状况 `Ready` 的值为 "`Unknown`"。
- `node.kubernetes.io/memory-pressure`：节点存在内存压力。
- `node.kubernetes.io/disk-pressure`：节点存在磁盘压力。
- `node.kubernetes.io/pid-pressure`：节点的 PID 压力。
- `node.kubernetes.io/network-unavailable`：节点网络不可用。
- `node.kubernetes.io/unschedulable`：节点不可调度。
- `node.cloudprovider.kubernetes.io/uninitialized`：如果 kubelet 启动时指定了一个“外部”云平台驱动， 它将给当前节点添加一个污点将其标志为不可用。在 cloud-controller-manager 的一个控制器初始化这个节点后，kubelet 将删除这个污点。

案例：

1. 专用节点（污点）
2. 配备了特殊硬件的节点（污点 + 亲和性）
3. 基于污点驱逐



## Node Selectors

规约中设置你希望目标节点所具有的[节点标签](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#built-in-node-labels)

将在特定节点上运行

```sh
kubectl get nodes --show-labels
```

添加标签

```sh
kubectl label nodes <your-node-name> disktype=ssd
```

Pod 配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

Node Selectors 是 Kubernetes 中的一种基本的节点选择策略，它允许你指定 Pod 应该在具有特定标签的节点上运行。然而，Node Selectors 的功能相对有限，无法满足一些复杂的需求。

以下是一个例子：  

+ 假设你有一个集群，其中包含两种类型的节点：cpuType: high 和 cpuType: low，分别表示高性能 CPU 节点和低性能 CPU 节点。你的应用有两种类型的 Pod：计算密集型 Pod 和 I/O 密集型 Pod。你希望计算密集型 Pod 在高性能 CPU 节点上运行，而 I/O 密集型 Pod 在低性能 CPU 节点上运行。  使用 Node Selectors，你可以为计算密集型 Pod 设置 nodeSelector: {cpuType: high}，为 I/O 密集型 Pod 设置 nodeSelector: {cpuType: low}。这样可以满足基本的需求。  



然而，如果你的需求更复杂一些，例如：  

+ 如果高性能 CPU 节点不足，计算密集型 Pod 可以在低性能 CPU 节点上运行。
+ 如果低性能 CPU 节点不足，I/O 密集型 Pod 可以在高性能 CPU 节点上运行。
+ 计算密集型 Pod 在选择节点时，优先选择高性能 CPU 节点。

​		这种情况下，Node Selectors 就无法满足需求了，因为它只能进行简单的标签匹配，无法实现优先级或条件选择。  为了满足这种复杂的需求，Kubernetes 提供了更强大的节点选择策略，如 Node Affinity/Anti-Affinity 和 Taints & Tolerations。



## Node Affinity

概念上类似于 `nodeSelector`， 它使你可以根据节点上的标签来约束 Pod 可以调度到哪些节点上

 ![image-20240520175942989](/images/image-20240520175942989.png "Title")

