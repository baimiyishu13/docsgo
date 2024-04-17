+++
title = 'Ansible 分子测试'
date = 2024-04-17T22:03:44+08:00
draft = true
+++

Ansible Molecule 是一个用于 Ansible 角色的测试框架。它可以帮助我们创建、运行和测试 Ansible Playbooks。Molecule 提供了一种创建和管理虚拟机的方法，这些虚拟机可以用于测试 Ansible Playbooks。它还提供了一种方法来验证 Playbooks 的行为，这是通过运行一系列的测试来完成的。 

 结合 GitHub 使用 Ansible Molecule 的好处包括：  

1. 版本控制：GitHub 提供了一个平台，可以跟踪和管理 Ansible Playbooks 的版本。这意味着你可以轻松地回滚到以前的版本，或者查看 Playbook 的历史记录。  
2. 协作：GitHub 允许多个开发者同时在同一个项目上工作。这意味着你可以与团队成员共享和协作 Ansible Playbooks，而不必担心版本冲突。  
3. 持续集成/持续部署（CI/CD）：GitHub Actions 可以自动运行 Molecule 测试，每当你提交新的代码到 GitHub 时，它都会自动运行测试。这可以确保你的 Ansible Playbooks 总是按预期工作。
4. 代码审查：GitHub 提供了代码审查工具，这可以帮助你确保 Ansible Playbooks 的质量。你可以在合并代码之前，要求其他开发者审查你的代码。  
5. 文档：GitHub 提供了一个平台，可以编写和存储 Ansible Playbooks 的文档。这可以帮助你和你的团队更好地理解和使用 Playbooks。

分子测试文件：node-exporter为例

converge.yml

```yaml
---
- name: Converge
  hosts: all
  become: true

  pre_tasks:
    - name: Update yum cache.
      yum:
        update_cache: yes
      when: ansible_os_family == 'RedHat'
      changed_when: false

  roles:
    - role: node-exporter
    
```

molecule.yml

```yaml
---
role_name_check: 1
dependency:
  name: galaxy
  options:
    ignore-errors: true
driver:
  name: docker
platforms:
  - name: instance
    image: "geerlingguy/docker-${MOLECULE_DISTRO}-ansible:latest"
    command: ${MOLECULE_DOCKER_COMMAND:-""}
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    cgroupns_mode: host
    privileged: true
    pre_build_image: true
provisioner:
  name: ansible
  playbooks:
    converge: ${MOLECULE_PLAYBOOK:-converge.yml}

```



CI文件

```sh
---
name: CI
'on':
  pull_request:
  push:
    branches:
      - main
  schedule:
    - cron: "30 4 * 1 1"

defaults:
  run:
    working-directory: 'node-exporter'

jobs:

  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Check out the codebase.
        uses: actions/checkout@v2
        with:
          path: 'node-exporter'

      - name: Set up Python 3.
        uses: actions/setup-python@v2
        with:
          python-version: '3.x'

      - name: Install test dependencies.
        run: pip3 install yamllint

      - name: Lint code.
        run: |
          yamllint .

  molecule:
    name: Molecule
    runs-on: ubuntu-latest
    strategy:
      matrix:
        distro:
          - centos7

    steps:
      - name: Check out the codebase.
        uses: actions/checkout@v2
        with:
          path: 'node-exporter'

      - name: Set up Python 3.
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install test dependencies.
        run: pip3 install ansible molecule molecule-plugins[docker] docker

      - name: Run Molecule tests.
        run: molecule test
        env:
          PY_COLORS: '1'
          ANSIBLE_FORCE_COLOR: '1'
          MOLECULE_DISTRO: ${{ matrix.distro }}
```



