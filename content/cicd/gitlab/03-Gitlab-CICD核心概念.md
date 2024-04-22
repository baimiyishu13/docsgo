+++
title = '第3章：Gitlab CICD核心概念 作业、阶段、变量...'
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

分支只有一个阶段被执行



### workflow

定义工作流的规则。这些规则可以用来控制整个 pipeline 的行为

例如，你可以使用 workflow: 关键字来定义当 pipeline 应该运行，或者当某个 job 失败时是否应该继续执行其他的 jobs

只有main 执行

CI_PIPELINE_SOURCE 和 CI_COMMIT_BRANCH 是 GitLab CI/CD 中的预定义环境变量。

+ CI_PIPELINE_SOURCE：这个变量描述了触发当前 pipeline 的事件类型。可能的值包括 "push"、"web"、"trigger"、"schedule"、"api"、"external" 等。例如，如果当前 pipeline 是由定时任务触发的，那么 CI_PIPELINE_SOURCE 的值就会是 "schedule"。  
+ CI_COMMIT_BRANCH：这个变量表示当前提交所在的分支名称。例如，如果你在 "main" 分支上进行了一次提交，并触发了 CI/CD pipeline，那么 CI_COMMIT_BRANCH 的值就会是 "main"。

```yml
---
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: always
    - when: never
```

when: always 表示这个 pipeline 将始终运

查看变量：https://docs.gitlab.com/ee/ci/variables/predefined_variables.html



具体的分支测试

1. 一旦开发测试完分支，合并，再次进行测试确保没有问题
2. 对于合并请求：只执行 test

 ![image-20240422130212466](/images/image-20240422130212466.png  "Title")

```yaml
---
workflow:
  rules:
    - if: $CI_COMMIT_BRANCH != "main" && $CI_PIPELINE_SOURCE != "merge_request_event"
      when: never
    - when: always

stages:
  - test
  - build
  - deploy

run_unit_tests:
  stage: test
  before_script:
    - bash prepare-tests.sh
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
  only:
    - main
  stage: build
  script:
    - echo "Building image"

push_image:
  only:
    - main
  stage: build
  needs:
    - build_image
  script:
    - echo "Pushing image"

deploy_image:
  only:
    - main
  stage: deploy
  script:
    - echo "Deploying image "

```



### 变量

除了Gitlab 提供的变量外，实际上还可以定义自己的变量

+ 多个微服务运行管道，传递微服务名称作为参数
+ 假设都有个环境，该服务应该部署到哪个环节

![image-20240422145028484](/images/image-20240422145028484.png "Title") 

![image-20240422145709289](/images/image-20240422145709289.png "Title")

引用变量：

```sh
deploy_image:
  only:
    - main
  stage: deploy
  script:
    - echo "Deploying docker image to $DEPLOYMENT_ENVIRONMENT"
```

 ![image-20240422150525141](/images/image-20240422150525141.png "Title")



变量文件

我们经常需要一些 外部配置文件或某种属性，为构建应用程序和运行测试。

+ 可能是包含数据库配置的文件等等
+ 其他服务的配置等

内容可以是 yml json

 ![image-20240422154359732](/images/image-20240422154359732.png "Title")



**variables**

```yml
variables:
  image_repository: docker.io/my-docker-id/myapp
  image_tag: v1.0
  
build_image:
  only:
    - main
  stage: build
  script:
    - echo "Building image"
    - echo "docker image $image_repository:$image_tag"
```



