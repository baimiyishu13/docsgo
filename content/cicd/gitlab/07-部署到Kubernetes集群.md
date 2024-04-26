+++
title = '第 8 章: 部署到Kubernetes集群'
date = 2024-04-26T11:03:38+08:00
draft = true

+++

准备k8s集群

部署到集群需要：kube-config

+ shell runner 上安装 kubectl 命令 【https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/】

不应该给 gitlab 直接访问集群的权限，如果 gitlab 被攻破，集群也会

+ 为gitlab创建一个单独的 sa，赋予特定权限
+ gitlab-kubeconfig

```sh
kubectl create namespace my-micro-service
kubectl create sa icd-sa -n my-micro-service
vi cicd-role.yml # 配置role
kubectl create rolebinding cicd-rb --role=cicd-role --serviceaccount=cicd-sa -n my-micro-service

```



deploy-k8s.yml

```yml
deploy:
    stage: deploy
    before_script:
        - export IMAGE_NAME=$CI_REGISTRY_IMAGE/microservice/$MICRO_SERVICE
        - export IMAGE_TAG=$SERVICE_VERSION
        - export MICRO_SERVICE=$MICRO_SERVICE
        - export SERVICE_PORT=$SERVICE_PORT
        - export REPLICAS=$REPLICAS
        - export KUBECONFIG=$KUBE_CONFIG
    script:
        - kubectl create secret docker-registry my-registry-key --docker-server=$CI_REGISTRY --docker-username=$GITLAB_USER --docker-password=$GITLAB_PASSWORD -n my-micro-service --dry-run=client -o yaml | kubectl apply -f -
        - envsubst < kubernetes/deployment.yaml | kubectl apply -f -
        - envsubst < kubernetes/service.yaml | kubectl apply -f -

```

