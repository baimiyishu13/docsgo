+++
title = '第一章：Argocd-Kubernetes的GitOpsCD'
date = 2024-04-26T14:32:59+08:00
draft = true

+++

## 1-简述

ArgoCD 是一种 GitOps 持续交付工具，在 DevOps 中越来越受欢迎。

 <img src="../../static/images/image-20240510145239858.png" alt="image-20240510145239858" title="Title" style="zoom:50%;" />

官网：https://argo-cd.readthedocs.io/en/stable/

学习前必备知识：

+ docker 、kubernetes、HELM(可选)、CICD工具其一即可（gitlab、github、jenkins）

Argo CD : 持续交付工具，将Argocd理解成CD工具或持续交付的工具，对比实验 Jenkins 或者 gitlab 中的持续交付。

所以，Argocd 并不存在取代 jenkins 或者 gitlab等等说法

设计实验仓库：

https://gitlab.com/mymicroservice-cicd9154417



### 1-1 不使用 ArgoCD 的 CD 工作流程

假设我们有一套服务运行在 k8s集群。

当我们新增功能或者代码发生改变时，使用 jenkins 或者 gitlab 构建的 ci 管道构建了新的镜像，并将其推送到仓库，此时这个镜像如何部署？

1. 使用 yml 更新镜像tag，部署到集群，这些操作是 CI 管道的延续
2. 因此 jenkins 或 gitlab 将更新应用程序的部署 yml，并使用 kubectl 去更新部署

这种方法存在的挑战：

1. 需要设置安装kubectl 或者 helm等工具来访问集群
2. 还需要为这些工具配置访问，连接凭据 【如果使用的是云集群，还需要配置 ak、sk等等】

引出的问题：配置工作量、安全问题【将集群凭证和 云凭证提供给了外部服务】

另一个问题：

1. 使用管道部署后，无法进一步了解应用程序状态。管道并不知道执行 更新后负载的状态 【即使可以通过脚本去实现，但将会使管道冗余和加长管道执行时间】

### 1-2 使用 ArgoCD 的 CD 工作流程

Argo CD 结合 Gitlab 将更加高效，可以持续交付的 kubernetes。

Argocd 如何解决之前的困难：【反转流程】不再使用jenkins 、gitlab的管道直接使用kubectl 去更新

1. Argocd 本身可以是 kubernetes 集群一部分
2. Argocd 通过 pull 拉取 git仓库，并将更改应用。

替换管道中kubectl后流程：

1. CI管道：TEST、BUILD、PUSH镜像仓库、Update yml
2. 提交后 Argo cd pull git仓库发现需要更改。【配置了持续监视】

【git 一个最佳实践】：开发代码与 部署yml 独立

当仅仅是应用程序 部署的yml需要改变时，仅仅更新部署 git yml 仓库即可

+ 无需 运行整个 CI管道
+ 无需 登入集群内 修改

也不希望在构建管道时有非常仓复的逻辑

部署存储库可以是：

1. 纯 yml
2. helm 等等，最终生成的都是存 yml 文件



最终：将拥有 单独的 CI、CD管道。

+ CI 管道，由开发拥有
+ CD 管道：运维 或 Devops 镜像配置使用



### 1-3 将 GitOps 与 ArgoCD 结合使用的好处

好处

1. 将k8s配置定义为 git 存储库中代码，意味着 版本/权限 控制
2. 某人在集群中使用kubectl 操作，要比 使用 git 更改提交慢很多
3. 监视 git仓库更改、监视集群内的更改。不匹配时【可选1、同步git仓库。2、不允许更改】
   + 覆盖手动更改的 【非常有用】
   + 也可以不覆盖，而是发生告警，说明集群中的更改
4. 版本控制，提供历史更改记录，以及谁更改了集群



轻松回滚，只需要 git 回退即可实现

集群灾难恢复：这也变得非常容易，在另一个集群中直接使用 git存储库部署即可



### 1-4 使用 Git 和 ArgoCD 进行 K8s 访问控制

不希望每个团队都对集群访问更改，可以轻松配置访问规则

每个人都可以发起提出对集群的更改，但是需要审批【审批】

只需要：

+ 让工程师访问 git仓库即可



### 1-5 ArgoCD 作为Kubernetes 扩展

直接部署在集群，实际上是 kubernetes API 扩展。利用了 k8s 资源本身，并不是自己构建所有功能，使用现有的功能完成。

好处：

1. 集群中的可见性
2. 更改时可实时看到 过程



### 如何配置 ArgoCD？

https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/
