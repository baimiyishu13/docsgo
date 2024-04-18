+++
title = 'Ansible Playbook'
date = 2024-04-09T10:45:10+08:00
draft = true
+++



工作中大多时间使用的 playbook

1. 传递清单文件
2. 哪些服务器执行
3. 命令
4. -f 指定并行、--limit指定主机

-b：成为 root，默认使用 sudo

后台运行：

```sh
ansible multi -B 3600 -P 0 -a "yum -y update"
```

-B 3600：这个选项设置了任务的超时时间（以秒为单位）

-P 0：这个选项设置了Ansible的轮询间隔（以秒为单位）。在这个例子中，Ansible将不会轮询任务的状态，也就是说，Ansible将会立即返回，而不会等待任务完成。

提供了一个JobID

```sh
ansible multi -b -m async_status -a "jid=j993071582105.3469"
```



shell script

```sh
#!/bin/bash
yum install --quiet -y httpd httpd-devel

# copy config files
cp httpd.conf /etc/httpd/conf/httpd.conf
cp httpd-vhosts /etc/httpd/conf/httpd-vhosts.conf

service httpd start
chkconfig httpd on
```



ansible playbook

```yml
root@ubuntu-c:~/ansible/study/chapter2# cat main.yml 
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
    - name: restart solr
      systemd:
        name: solr
        state: restarted
        enabled: yes
  tasks:
    - name: install java.
      yum:
        name: java-11-openjdk
        state: present

    - name: Download solr
      get_url:
        url: "https://downloads.apache.org/lucene/solr/{{ solr_version }}/solr-{{ solr_version }}.tgz"
        dest: "{{ download_dir }}/solr-{{ solr_version }}.tgz"
        checksum: "{{ solr_chcksum }}"

    - name: Expand Solr.
      unarchive:
        src: "{{ download_dir }}/solr-{{ solr_version }}.tgz"
        dest: "{{ download_dir }}"
        remote_src: yes
        creates: "{{ download_dir }}/solr-{{ solr_version }}/README.md"

    - name: Run solr install
      shell: >
       {{ download_dir }}/solr-{{ solr_version }}/bin/install_solr_service.sh
       {{ download_dir }}/solr-{{ solr_version }}.tgz
       -i /opt
       -d /var/solr
       -u solr
       -s solr
       -p 8983
       creates={{ solr_dir }}/bin/solr 
  
    - name: enabled solr
      systemd:
        name: solr
        state: restarted
        enabled: yes
```





查看主机清单：

```sh
ansible-inventory l --list
```



检查语法：

```sh
ansible-playbook main.yml --syntax-check
```













