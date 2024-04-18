+++
title = 'Ansible 变量'
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





## 加密

未加密

```yaml
root@ubuntu-c:/ansible/study/chapter3# cat api_key.yml 
---
myapp_api_key: "dgbvuiabufihnaeifhnajdnfa"
root@ubuntu-c:/ansible/study/chapter3# cat main.yml 
---
- name: install
  hosts: app
  become: true

  vars_files:
    - api_key.yml

  tasks:
    - shell: echo $API_KEY
      environment:
        API_KEY: "{{ myapp_api_key }}"
      register: echo_apikey 
    - debug: var=echo_apikey.stdout
```

加密：

```yaml
root@ubuntu-c:/ansible/study/chapter3# ansible-vault encrypt api_key.yml 
New Vault password: 
Confirm New Vault password: 
Encryption successful
root@ubuntu-c:/ansible/study/chapter3# cat api_key.yml 
$ANSIBLE_VAULT;1.1;AES256
33383965396666646635313061353936373236646132383765373937633765623337323835333464
3434353266316337346335323466626432303566363033340a656561653836613762326561313434
32316563623434633838613634363937666630333463326564633132316664656430613366316663
6230303233373934330a363038343261346131363863656565623363326634376364393831306331
61313566373832306665386334643435656438656535643930313662363134313234373236303630
6333336330623265306636343766383430653435643861626530
```

执行：

```sh
ansible-playbook  main.yml --ask-vault-password
```

其实这样很不方便，尤其是是 CICD中。



