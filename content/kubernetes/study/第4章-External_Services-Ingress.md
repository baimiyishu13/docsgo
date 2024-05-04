+++
title = '第4章: External_Services'
date = 2024-04-30T10:41:13+08:00
draft = true

+++

详细官方文档：【https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#publishing-services-service-types】

SVC 服务的IP从何来？

SVC的IP和PodIP在完全不同的范围

如果更新了 svc的 CIDR，旧的分配的IP将会保留

默认service 类型： ClusterIP



创建：

```sh
[root@local ~]# kubectl create service clusterip test-new-cidr --tcp=80:80 --dry-run=client -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: test-new-cidr
  name: test-new-cidr
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: test-new-cidr
  type: ClusterIP
```

deployment 同样：

```sh
[root@local ~]# kubectl create deployment my-nginx -n default --image=nginx --dry-run=client -o yaml
```



## Service 

Service API 是 Kubernetes 的组成部分，它是一种抽象，帮助你将 Pod 集合在网络上公开出去。 每个 Service 对象定义端点的一个逻辑集合（通常这些端点就是 Pod）以及如何访问到这些 Pod 的策略。



## 云原生服务发现

如果你想要在自己的应用中使用 Kubernetes API 进行服务发现，可以查询 [API 服务器](https://kubernetes.io/zh-cn/docs/concepts/overview/components/#kube-apiserver)， 寻找匹配的 EndpointSlice 对象。 只要 Service 中的 Pod 集合发生变化，Kubernetes 就会为其更新 EndpointSlice。



## 服务类型

### **ClusterIP**

内部 IP 公开 Service，选择该值时 Service 只能够在集群内部访问

### **NodePort**

每个节点上的 IP 和静态端口（`NodePort`）公开 Service。 为了让 Service 可通过节点端口访问

+ 范围内分配端口（默认值：30000-32767）



### LoadBalancer

在使用支持外部负载均衡器的云平台时，如果将 `type` 设置为 `"LoadBalancer"`， 则平台会为 Service 提供负载均衡器。

```yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

来自外部负载均衡器的流量将被直接重定向到后端各个 Pod 上，云平台决定如何进行负载平衡。

1. CluserIP-SVC
2. Node Port
3. Loadbanancer

三种类型合一



AWS上创建 LB，选择VPC、访问LB的Port、选择Port【node Port暴露出的端口】等等

![image-20240504151116681](/images/image-20240504151116681.png "Title")





## 服务发现

对于在集群内运行的客户端，Kubernetes 支持两种主要的服务发现模式：环境变量和 DNS
