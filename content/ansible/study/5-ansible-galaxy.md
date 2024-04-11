+++
title = '5 Ansible Galaxy'
date = 2024-04-11T09:41:25+08:00
draft = true
+++

示例文件

```sh
root@ubuntu-c:~/ansible/episode/00-galaxy# ls
ansible.cfg  main.yml  requiremerts.yml
```

ansible.cfg

+ 指定 role path

```yaml
[defaults]
INVENTORY = inventory
roles_path = ./roles
```

main.yml

```yaml
---
- name: install
  hosts: app
  become: true

  roles:
    - role: elliotweiser.osx-command-line-tools
    - role: geerlingguy.homebrew
```

requiremerts.yml

```yaml
---
roles:
  - name: elliotweiser.osx-command-line-tools
    version: 2.3.0
  - name: geerlingguy.homebrew
    version: 4.0.0
```

---

执行安装请求：

```sh
ansible-galaxy install -r requiremerts.yml
```







确保 ansible playbook 正确

1. yamllint
2. ansible-playbook --syntax-check
3. ansible-lint
4. molecule test
5. ansible-playbook --check





fail\assect
