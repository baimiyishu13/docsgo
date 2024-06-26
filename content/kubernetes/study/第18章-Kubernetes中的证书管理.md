+++
title = '第18章: Kubernetes 中的证书管理'
date = 2024-05-11T15:56:21+08:00
draft = true

+++

### 集群证书续期

集群已经运行了半年多，一个成员突然提出

+ 应该检查证书何时到期，并在需要的时候续订

检查1：

openssl 命令来读取证书的到期时间

```sh
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

检查2：

1. 生成新的密钥 - 使用 kubeadm 将会非常简单

```sh
kubeadm certs --helm
kubeadm certs check-expiration
```

证书默认1年

更新：

1. openssl 生成新的证书和私钥
2. 或者 使用 kubeadm

更新证书注意：

+ 更新集群，kubeadm 会自动续订证书



### 为 kubelet 配置证书轮换

https://kubernetes.io/zh-cn/docs/tasks/tls/certificate-rotation/

Kubelet 使用证书进行 Kubernetes API 的认证。 默认情况下，这些证书的签发期限为一年，所以不需要太频繁地进行更新。

Kubernetes v1.8 和更高版本的 kubelet 实现了对客户端证书与/或服务证书进行轮换这一特性。 请注意，服务证书轮换是一项 **Beta** 特性，需要 kubelet 上 `RotateKubeletServerCertificate` 特性的支持（默认启用）。

