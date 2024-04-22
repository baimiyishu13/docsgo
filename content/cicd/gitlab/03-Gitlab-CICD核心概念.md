+++
title = '03 Gitlab CICD核心概念'
date = 2024-04-21T17:46:46+08:00
draft = true
+++

gitlab-ci.yml

## Jobs

Job: 管道中的一个步骤或执行某个操作的工作流

```
script: 执行如何命令、Linux 命令
before_scrip: 安装依赖项、设置环境变量等
after_script: 通常用于清理作业环境，例如删除临时文件、停止服务等
```



pipeine

```yml
---
run_tests:
  before_script:
    - echo "Setting up test environment"
    - apk add yamllint
  script:
    - echo "Running tests"
    - yamllint gitlab-ci.yml
  after_script:
    - echo "Cleaning up test environment"

build_image:
  script:
    - echo "Building image"
    - docker build -t node_exporter .

push_image:
  script:
    - echo "Pushing image"
    - docker push node_exporter
```





实验项目：https://gitlab.com/baimiyishu13/mynodeapp-cicd-project

创建：.gitlab-ci.yml 会自动识别为管道文件



### stage & stages

如果不想入上并行，则使用 stage 

+ 如果一个 stage 失败，则不会执行下一个

![image-20240422111721437](/images//image-20240422111721437.png "Title")

![image-20240421220113191](/images//image-20240421220113191.png "Title")

```yml
---
stages:
  - test
  - build
  - deploy

run_unit_tests:
  stage: test
  before_script:
    - echo "Setting up test environment"
  script:
    - echo "Running unit tests"
  after_script:
    - echo "Cleaning up unit test environment"

run_line_test:
  stage: test
  before_script:
    - echo "Setting up test environment"
  script:
    - echo "Running unit tests"
  after_script:
    - echo "Cleaning up line test environment"

build_image:
  stage: build
  script:
    - echo "Building image"

push_image:
  stage: build
  script:
    - echo "Pushing image"

deploy_image:
    stage: deploy
    script:
        - echo "Deploying image"
```



### needs

实现：build_image 完成后才执行push_image

+ needs: 需要

```
build_image:
  stage: build
  script:
    - echo "Building image
    - docker build -t my-image .

push_image:
  stage: build
  needs:
    - build_image
  script:
    - echo "Pushing image"
```



### 查看依赖

 ![image-20240421221339496](/images//image-20240421221339496.png "Title") 

![image-20240421223042109](/images//image-20240421223042109.png "Title")



### 执行脚本

如果使用 

```yaml
  before_script:
    - echo "Setting up test environment"
    - pwd
    - ls -la
    - mkdir -p /tmp/test
    - ls -la
```

如果有更复杂的逻辑，就不适合这样去做，会降低可读性

```sh
run_unit_tests:
  stage: test
  before_script:
    - bash /prepare-tests.sh
```



注意： 每次cicd 都是在干净的环境中执行，甚至不必清理创建的测试文件

### only

only 关键字用于定义一组规则，这些规则决定了特定的 job 在何种情况下才会被执行。例如，你可以指定 job 只在特定的分支、标签或者当有特定的文件更改时才执行。

分支触发

```sh
git checkout -b feature/test-cicd-pipeline
```

 ![image-20240422112622863](/images//image-20240422112622863.png "Title")

这意味着  .gitlab-ci.yml 适用于每个分支

但是，并不想提交任何分支时就执行管道：

+ test：分支在合并到main 之前执行test
+ 合并到主分支时，进行构建部署



```yaml
build_image:
  only:
    - main
  stage: build
  script:
    - echo "Building image"
```

等

 ![image-20240422114005373](/images//image-20240422114005373.png "Title")
