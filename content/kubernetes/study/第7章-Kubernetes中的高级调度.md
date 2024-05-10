+++
title = '第7章:  Kubernetes中的高级调度'
date = 2024-05-05T23:25:04+08:00
draft = true
+++

kubernetes 最大优势就是可以抽象出单独的服务器，不需要担心各个服务器细节。

内置调度器：决定将工作负载放在哪里，例如想调度一个Pod

1. 先过滤掉不符合条件的节点，剩余的节点成为：可调度节点
2. 运行一组函数对节点进行打分，然后会选最高分的节点来运行【如果是多个相同最高分节点，随机选】这个过程称为绑定

实际的生产环境中可能有不同的节点，高CPU、高内存、高性能存储、GPU、云上按需计费、包年...,性能差异的节点等等



### nodeName

直接将 Pod 固定在一个节点上

```yml
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
  nodeName: node1
```





## Kubernetes Node Selector

将 `nodeSelector` 字段添加到 Pod 的规约中，Kubernetes 只会将 Pod 调度到拥有你所指定的每个标签的节点上。

缺点：不灵活

+ 可以使用：亲和性和反亲和性扩展了你可以定义的约束类型

```sh
kubectl get node --show-labels
```

可以为特定节点上分配特定标签，也有一些默认的标签

```sh
[root@local ~]# kubectl get node --show-labels 
NAME      STATUS   ROLES                  AGE   VERSION        LABELS
monitor   Ready    control-plane,master   11d   v1.29.3+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=monitor,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
```

添加标签：

```sh
kubectl label nodes <your-node-name> disktype=ssd
```

Pod

```yml
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





亲和性功能由两种类型的亲和性组成：

- **节点亲和性**功能类似于 `nodeSelector` 字段，但它的表达能力更强，并且允许你指定软规则。
- Pod 间亲和性/反亲和性允许你根据其他 Pod 的标签来约束 Pod。

## Kubernetes 节点亲和性 

两种：

1. 软亲和：调度器会尝试寻找满足对应规则的节点。如果找不到匹配的节点，调度器仍然会调度该 Pod。
2. 硬亲和：调度器只有在规则被满足的时候才能执行调度

配置示例：

```yml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:2.0
```





## Kubernetes Pod 反亲和性

## Kubernetes Pod 亲和性

## Kubernetes污点和容忍



