+++
title = '01 Efk'
date = 2024-04-28T11:08:02+08:00
draft = true
+++

在集群内使用EFK堆栈

HELM 安装



实验：将创建2个 程序进行测试

地址：https://gitlab.com/baimiyishu13/node-app

```sh
➜  node-app git:(master) ls
Dockerfile        app               node_modules      package.json
Readme.md         deployment.yaml   package-lock.json
```

构建测试镜像：

```sh
docker build -t node-app .
docker tag node-app demo-app:node-1.0
➜  node-app git:(master) docker images | grep node-1.0
demo-app                       node-1.0   1e9ad3a8910c   13 seconds ago   117MB
```

第二个程序

```sh
docker build -t java-app .
docker tag java-app demo-app:java-1.0
```





### 部署前须知概念

ElasticSearch：

1. 将部署多副本的有状态ElasticSearch程序

简述： 用于搜索和分析的数据存储，可以存储许多不同的形式，例如SQL数据库，仅以结构化形式存储数据 表/行/列。

ElasticSearch 可以存储费结构化数据，着意味着无需任何域定义的形式

主要优势：强大的搜索机制

























