+++
title = '第5章：Ingress'
date = 2024-05-04T15:40:14+08:00
draft = true

+++

[官方文档](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/)

## 简述

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#ingress-v1-networking-k8s-io) 提供从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源所定义的规则来控制。

![ingress-diagram](https://kubernetes.io/zh-cn/docs/images/ingress.svg "Title")

使用LB存在的缺点：

1. 如果需要包括多个服务，将会得到一组 LB，我们想让用户访问：通过域名，而非LB的域名或者IP。没有一个域名可以让我们访问所有LB服务
2. 在服务器上将会使 NodePort 暴露多个端口。



Ingress：集群入口组件，接受请求并路由到服务

1. 部署在集群内，将ingress svc 端口暴露出去。【所以我们将只需要一个LB】
2. Ingress 不会随意公开端口或协议。 将 HTTP 和 HTTPS 以外的服务开放到 Internet 时，通常使用 [Service.Type=NodePort](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport) 或 [Service.Type=LoadBalancer](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer) 类型的 Service。

Ingress语法：

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80

```

https：

```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

secret: tls.crt tls.key



## 安装 Ingress 控制器

Helm 安装 Nginx ingress Controller

在使用云时，有云各个厂商的 ingress

安装参考：https://kubernetes.github.io/ingress-nginx/deploy/

安装完成后：

1. AWS 创建 LB，端口绑定到 ingress-nginx-controller 的80/443 映射出来的端口

![image-20240504220840906](/images/image-20240504220840906.png "Title")

