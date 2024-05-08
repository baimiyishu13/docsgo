+++
title = '第7章:  Kubernetes中的高级调度'
date = 2024-05-05T23:25:04+08:00
draft = true
+++

Kubernetes 节点选择器

Kubernetes 节点亲和性 

Kubernetes Pod 反亲和性

Kubernetes Pod 亲和性

Kubernetes污点和容忍



```
stages:
  - build
  - test
  - generate

pip-install:
  stage: build
  image: python:3.7
  script:
    - pip install wheel
    - mkdir wheels
    - pip wheel --wheel-dir ./wheels -r requirements.txt
  artifacts:
    paths:
      - ./wheels/

test-install:
  stage: test
  image: python:3.7
  dependencies:
    - pip-install
  script:
    - cd wheels
    - |
      while read wheel
      do
        pip install --find-links ./ "$wheel"
      done < ../whl.txt

generate-wheel-and-md5:
  stage: generate
  image: python:3.7
  script:
    - chmod +x ./generate_dependencies.sh
    - ./generate_dependencies.sh
  artifacts:
    paths:
      - ./dependencies.md
```
