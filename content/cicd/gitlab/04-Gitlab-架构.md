+++
title = '第4章：GitLab架构: Runners, 执行器'
date = 2024-04-22T16:47:48+08:00
draft = true
+++

## GItLab

### 架构

触发的管道

问题：gitlab 在哪里执行这些操作，就想执行的echo、ls、创建文件等等，实际上发生在哪里。

必须在某个服务器或者环境中运行

1. gitlab.com 本身托管的gitlab 实例
2. gitlab runner：安装在单独服务器，而不是运行在gitlab 服务器上，连接到gitlab服务器就可以执行gitlab服务器发给它的job。所谓的运行器【可供所有用户使用】



## 执行器

选择那种类型的执行器：

每个都有自己的用例

+ 一般使用linux 构建标准的 ci cd 管道 docker 执行器
+ kubernetes 集群中执行

定义和配置 gitlab runner 执行器

 可以在一台服务器运行多个执行器



### Shell 执行器

gitlab runner 实际上就是一个中介，它将从 gitlab 获取到的作业命令移交给它正在运行的服务器的 shell，实际上是在 gitlab runner 运行的服务器。

执行作业的服务器，通常也称为：shell 执行器 

意味着需要使用命令：

+ docker
+ go
+ npm
+ maven

等等，都需要安装这些工具，升级版本等等。意味着需要管理员大量的操作



### Docker 执行器

可以随意安装服务、工具，不必在服务端配置其他。

每个job 都会在干净环境中启动



### VM 执行器

每个作业都需要加载一个 vm 整个操作系统，意味着会非常慢



### Kubernetes 执行器

意味着现在有一个集群，将创建一个新的Kubernetes Pod 作为运行环境



### Docker Machine 执行器

docker 的扩展

创建docker主机，在一台机器上管理多个Docker 实例



总结：

三部分：gitlab server、gitlab runner、execute 执行器

1、 gitlab runner 拉取gitlab server 的job

2、一旦有了job，执行器将去编译job 发送到执行器

3、gitlab 执行器之间从git 仓库拉取代码执行 





### Docker 执行器

gitlab.com 默认使用 Preparing the "docker+machine" executor

同样注意：每个作业实际上可能在一个完全不同的容器中执行，所以并不意味着所有作业都在同一台机器行执行。



![image-20240422221707819](/images/image-20240422221707819.png "Title")



### 覆盖Docker 执行器

默认使用：

+ docker+machine
+ image：ruby

优势：可以使用安装工具后的镜像（比如 npm node java ），需要什么就使用内涵什么的镜像

```yml
---
image: node:21-alpine
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH != "main" && $CI_PIPELINE_SOURCE != "merge_request_event"
      when: never
    - when: always

stages:
  - test
  - build
  - deploy
```

