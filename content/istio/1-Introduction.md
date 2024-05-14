+++
title = '第1章：Istio 简介'
date = 2024-05-14T14:04:39+08:00
draft = true

+++

 🚀 Istio将K8s提升到了一个新的高度，那是因为 Istio 提供了一套非常丰富的工具负责监测、管理和保障k8s集群。

目标：可理解、可使用在生产项目中

微服务架构非常复杂，Istio 会提供一个 UI视图到Pod级别，看到它们是如何连接在一起的，存在问题的地方，还有分布式追踪，还可以在不进入集群的情况下改变流量。

例如：将未经过测试的新软件投入到一个生产集群，可以对普通用户隐藏，只有运维工程师将能够访问它。可以做像金丝雀释放一样的事情，可以改进系统的弹性和安全性，增加熔断器。着一直是微服务架构最佳实践，但是传统中很难做，Istio使它变得简单。

安全 - 整个集群中设置加密



学习计划安排：

```
1 - Introduction
2 - Getting Started
3 - Optional Installing a Local Kubernetes Environment
4 - Hands on Demo
5 - Introducing Envoy
6 - Telemetry
7 - Traffic Management
8 - Load Balancing
9 - Gateways
10 - Dark Releases
11 - Fault Injection
12 - Circuit Breaking
13 - Mutual TLS
14 - Customizing and Installing Istio with Istioctl
15 - Upgrading Istio
16 - Goodbye
```

