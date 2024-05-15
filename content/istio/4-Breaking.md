+++
title = '第四章：Istio 演示'
date = 2024-05-14T14:04:40+08:00
draft = true

+++

## 演示

[存储库](https://github.com/DickChesterwood/istio-fleetman/tree/master/_course_files)

### 1 - 运行Istio

```sh
➜  1-Telemetry ls
1-istio-init.yaml              3-kiali-secret.yaml            5-application-no-istio.yaml
2-istio-minikube.yaml          4-label-default-namespace.yaml
```

执行：

1. 创建 CRDS （kubernetes API）

```sh
$ kubectl create -f 1-istio-init.yaml
$ kubectl get ns
NAME              STATUS   AGE
istio-system      Active   2m10s
```

2. 安装 Istio 

+ Grafnan 和 prometheus
+ egressgateway 和 ingressgateway ： 控制进出的流量
+ jaeger：端到端分布式追踪系统
+ Istiod
+ kiali： 图形界面

```sh
kubectl create -f 2-istio-minikube.yaml
➜  1-Telemetry kubectl get pod -n istio-system      
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-686cc64b9b-rdrrc                1/1     Running   0          5m22s
istio-egressgateway-595dfb96b8-d9pch    1/1     Running   0          5m22s
istio-ingressgateway-66cdbc7dc8-7j64x   1/1     Running   0          5m22s
istiod-685fc94db9-kf7n6                 1/1     Running   0          5m22s
jaeger-664d88ccf4-47gnz                 1/1     Running   0          5m22s
kiali-87c6d84b7-gmclk                   1/1     Running   0          5m21s
prometheus-84cf88dfb8-zgt78             2/2     Running   0          5m22s
```

3. 为kiali 设置账户 admin admin

```sh
kubectl create -f 3-kiali-secret.yaml 
```

### 2 - 启用 Sidecar 注入

启用 Istio

```yml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    istio-injection: enabled
  name: default
```

创建：

```sh
kubectl apply -f  4-label-default-namespace.yaml
```

部署程序到 default ns

```sh
kubectl create -f  5-application-no-istio.yaml 
```

会发现有两个容器



### 3 - 使用 Kiali 可视化系统

暴露端口访问可以使用

```sh
minikube service  -n istio-system  kiali --url  
```



### 4 - 查找性能问题
