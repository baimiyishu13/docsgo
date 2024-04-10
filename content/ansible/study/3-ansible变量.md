+++
title = '3 Ansible变量'
date = 2024-04-10T12:51:25+08:00
draft = true
+++

## Handler

Playbook handlers, environment vars, and variables

```
handler:
  - name: restart httpd
  - name: restart nginx

...
  notifi: 
    - restart httpd
    - restart nginx
    
---
handler:
  - name: restart httpd
    systemd: ...
    notifi: restart nginx
  - name: restart nginx

...
  notifi: restart httpd
```

handler 将在剧本末尾执行



如果想立刻重启：

```sh
- name: make restart handlers
  meta: flush_handlers 
```



如果任务中有失败，那么久不会到最后一步执行handler

参数 

```
--force-handlers
```





## Vars

```yaml
---
vars:
  k: v
vars_files:
  - vars/main.yaml
```





yaml

```yaml
---
- name: install
  hosts: app
  become: true

  tasks:
    - name: Add env var
      lineinfile:
        path: "/etc/bashrc"
        regexp: '^ENV_VAR='
        state: present
        line: 'EVC_VAR=value'
```



使用 shell 模块会更简单

```yaml
  tasks:
    - name: Add env var
      lineinfile:
        path: "/etc/bashrc"
        regexp: '^ENV_VAR='
        state: present
        line: 'ENV_VAR=value'
    
    - name: Get ENV
      shell: >
        source /etc/bashrc && echo $ENV_VAR
      register: foo
    
    - debug: msg="message variable is {{ foo.stdout }}"
```



vars

```yaml
---
- name: install
  hosts: app
  become: true

  vars:
    proxy_vars:
      http: http://example.com:80
      https: https://example.com:80


  tasks:
    - name: test
      debug:
        msg: "{{ proxy_vars.http }}"
```



```yaml
vars_files:
  - vars/os-{{ version }}.yaml
```



## Ansible VARS

```sh
ansible app -m setup
```

默认收集信息

```
gather_facts
```



```yml
  register: foo
- debug: var=foo
```



