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



  也可以为每个任务添加image

```yaml
run_unit_tests:
  image: node:21-alpine
  stage: test
  before_script:
    - chmod +x prepare-tests.sh
    - ./prepare-tests.sh

run_line_test:
  image: node:21-alpine
  stage: test
  before_script:
    - echo "Setting up test environment"
```

不指定全局image，则默认使用ruby



## 特定的 Runner

假设团队希望使用单独的Runner用于项目，但是不希望弄乱其他项目的管道。

一家公司，多个项目团队。

一个团队可能不想与其他团队共享 Runner，而是在自己的隔离环境中运行项目作业。

特定的 Runner

+ 可以将其连接到特定的项目，仅用于特定的项目。其他项目无法使用 

github.com 上共享 runners 由gaitab 本身管理。

特定的 runners 由自我管理 

+ 意味着必须管理、创建它们。【服务器 - 安装 gitlab runner】  -  连接到 gitlab server



## runners 配置

可以在本地 MAC、windows上安装 runners

实验选择在 ECS2 公有云服务器安装：

安装：https://docs.gitlab.com/runner/install/

![image-20240423010913131](/images/image-20240423010913131.png  "Title")

参考提示创建：

```sh
gitlab-runner register  --url https://gitlab.com  --token glrt-WXbwYof8_63zzZmtxiTz
```

![image-20240423013027728](/images/image-20240423013027728.png "Title")



即使创建了 runner 运行管道还是会实验 gitlab 共享的 runner

默认：使用共享 runner



配置特定的 runner 在特定的 Job

```yaml
run_unit_tests:
  tags:
    - ec2
    - aws
    - remote
  stage: test
  before_script:
    - chmod +x prepare-tests.sh
    - ./prepare-tests.sh
    - npm --version
  script:
    - echo "Running unit tests for micro server $MICRO_SERVER_NAME"
  after_script:
    - echo "Cleaning up unit test environment"
```

如果出现浅克隆问题，可设置 禁用

```yaml
variables:
  GIT_DEPTH: 0
```

 ![image-20240423121539687](/images/image-20240423121539687.png "Title")



## Docker Runner

安装：https://docs.gitlab.com/runner/install/docker.html

同一台主机上添加多个 Runner

 ![image-20240423141618017](/images/image-20240423141618017.png "Title")



## kubernetes Runner

安装：https://docs.gitlab.com/runner/install/kubernetes.html#installing-gitlab-runner-using-the-helm-chart

helm 安装

```sh
helm repo add gitlab https://charts.gitlab.io
helm show values gitlab/gitlab-runner --version 0.64.0  > values.yaml
helm install --namespace gitlab-runner gitlab-runner -f ./values.yaml gitlab/gitlab-runner 
```

![image-20240423124236135](/images/image-20240423124236135.png "Title")



使用标签，可以灵活的去选择 Runner 执行特定的 Job

假设我们有几十个 Runner，希望配置不同的 runner 执行不同的 Job。

+ 多个具有相同标签的 runner 将形成高可用性
+ 如果项目单一，拥有20个linux 和docker 标签的 runner 也是正常的，基本是只是具有分配负载的功能



## Runners 组

已经创建三个 runner ，当前专用于测试项目。

![image-20240423143636737](/images/image-20240423143636737.png "Title")

实际上可以通过：锁定到当前项目（当Runner被锁定时，不能将其分配给其他项目）

一个项目多个微服务，每个微服务一个仓库。

假设有 20个，每个都有自己的管道



Groups：对项目进行分组 【多个项目，多个团队】

+ 好处1，管理组的权限，不用管理每个项目的权限



创建群组 Runner

![image-20240423151943003](/images/image-20240423151943003.png "Title")

---



## 管理 Gilab 实例



到目前为止：

1. gitlab.com 作为 gitlab server，托管存储库，直接构建CICD
2. 可以使用 gitlab 托管的 runner 共享执行器
3. 配置添加特定的组，拥有自己的 gitlab runner

除此之外，还需要自己运行配置 Gitlab Server

+ 高安全性
+ 项目/用户隔离
+ 绝对的控制
