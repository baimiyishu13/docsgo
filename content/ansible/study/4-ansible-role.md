+++
title = 'Ansible Role'
date = 2024-04-10T20:28:09+08:00
draft = true
+++



```yaml
  handlers:
    - name: restart solr
      systemd:
        name: solr
        state: restarted
        enabled: yes
```

写为：

```yaml
  handlers:
    - ipmort_tasks: handlers/apache.yml
```

文件

```sh
root@ubuntu-c:/ansible/study/chapter4# cat handlers/apache.yml 
---
- name: restart solr
  systemd:
     name: solr
     state: restarted
     enabled: yes
```

作用：导入，开始运行之前



tasks部分：更具可读性

1. ipmort_tasks  作用：导入，开始运行之前
2. 运行中产生的：动态定义 

```sh
---
- hosts: solr
  become: true

  vars_files:
    - vars.yml

  pre_tasks:
    - name: Update yum
      yum:
        update_cache: yes

  handlers:
    - ipmort_tasks: handlers/apache.yml
  tasks:
    - ipmort_tasks: tasks/apache.yml
    - incude_tasks:  tasks/logs.yml
    
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - ipmort_tasks: tasks/apache.yml
    - name: clone git
      git:
        repo: 'https://foosball.example.org/path/to/repo.git'
        dest: /srv/checkout
        version: release-0.22
```

