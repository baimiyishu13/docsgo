+++
title = 'Kubernetes DNS'
date = 2024-04-28T17:36:34+08:00
draft = true
+++

CoreDNS 深入探讨

好文：https://pracucci.com/kubernetes-dns-resolution-ndots-options-and-why-it-may-affect-application-performances.html

CoreDNS 最著名的是云原生生态系统中 Kubernetes 的默认集群 DNS，它也是一个可扩展且灵活的 DNS 服务器，专注于服务发现。 CoreDNS 的可扩展性来自其独特的基于插件的架构，可以轻松添加新功能。

在k8s这种Pod IP 会发生变化的集群中，使用IP访问是很糟糕的

DNS：IP翻译成域名

文件：

+ /etc/hosts
+ /etc/resolv.conf



当集群中存在数百个服务，这将是大量的

/etc/resolv.conf中 

nameserver 10.233.0.3 为 DNS服务器



1.12版本前使用kube-dns，之后使用 CoreDNS

```sh
[root@local ~]# kubectl get pod -A --show-labels 
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE   LABELS
kube-system   coredns-6799fbcd5-c869r                   1/1     Running   0          47m   k8s-app=kube-dns,pod-template-hash=6799fbcd5
```

Pod内：

```sh
[root@local ~]# kubectl exec -ti nginx-7854ff8877-5dtzt  -- sh
# cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.42.0.9	nginx-7854ff8877-5dtzt
# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.43.0.10
options ndots:5
```



不同命名空间访问

创建测试：

```sh
kubectl create deployment nginx-n-test --image=nginx -n tes
kubectl expose deployment -n test nginx-n-test --port 80
```

从default 的Pod中访问

+ 【service-name.namespace-name.svc.cluster.local】
+ 也可以是：`【<service-name>.<namespace>.svc.cluster.local.】`



```sh
[root@local ~]# kubectl get svc -n test 
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-n-test   ClusterIP   10.43.162.130   <none>        80/TCP    3m45s
[root@local ~]# kubectl exec -ti nginx-7854ff8877-5dtzt  -- sh
# curl nginx-n-test.test:80
<!DOCTYPE html>
```



详解 Pod 中的resolv.conf

```sh
# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.43.0.10
options ndots:5
```

共有三个指令：

1. 名称服务器 IP 是 Kubernetes 服务 IP`kube-dns`
2. 指定了4 个本`search`地域
3. 有一个`ndots:5`选项

此配置的有趣部分是本`search`地域和`ndots:5`设置如何协同工作。为了理解它，我们需要了解 DNS 解析如何处理非完全限定名称。

**什么是完全限定名称？**

完全限定名称是不会执行本地搜索的名称，并且在解析过程中该名称将被视为绝对名称。按照惯例，如果名称以句号 ( `.`) 结尾，则 DNS 软件会认为名称完全限定，否则视为非完全限定。 IE。`google.com.`是完全合格的，`google.com`不是。

**如何执行非完全限定名称解析？**

当应用程序连接到由名称指定的远程主机时，通常通过系统调用执行 DNS 解析，例如`getaddrinfo()`.如果名称不是完全限定的（不以 结尾`.`），系统调用会首先尝试将名称解析为绝对名称，还是会首先遍历本地搜索域？这取决于`ndots`选项。



CorDNS创建了一个子域，

1. service name
2. Namespace
3. Type ：svc
4. Root：cluster.local 【根域】

所以集群中如何服务名称都是：例如【nginx.default.svc.cluster.local】options ndots:5

不需要指定：svc.cluster.local ，这是默认的

+ 因为：【search】子域列表













