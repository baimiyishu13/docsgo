+++
title = 'Istio入门'
date = 2024-04-04T14:52:05+08:00
draft = true
+++

实验环境：minikube

```sh
➜  ~ kubectl get node
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   11m   v1.27.4
```



简述：架构、安装、实验 （了解总体再逐个细化）



Istio：开源服务网格，用来保护微服务

构建在 Envoy 之上：一个处理微服务之间所有流量的高效代理

IStio 工作原理：在环境中每个微服务旁注入一个 sidecar 容器，sidecar 容器拦截进出的微服务的所有流量，处理流量路由，负载均衡，服务发现和其他重要的网络任务。

Istio还提供：

1. 高级流量管理功能：金丝雀，A/B测试
2. 故障注入



首先，Istio 为现代微服务架构提供了多项优势，通过抽象服务发现、负载均衡、流量路由的复杂性来简化网络管理。

其次，Istio 提供了高级安全功能，列入双向的TLS 身份验证、基于角色的访问控制和流量加密。

最后，Istio 提供了可观察性功能，例如分布式跟踪、指标收集和日志记录。



安装：大多数使用 Helm 安装

参考：https://istio.io/latest/zh/docs/setup/install/helm/

```sh
➜  ~ helm search repo istio/base
NAME      	CHART VERSION	APP VERSION	DESCRIPTION                                       
istio/base	1.21.0       	1.21.0     	Helm chart for deploying Istio cluster resource...
```

查看默认值：

```sh
helm show values istio/base
```

使用 treeaform创建

