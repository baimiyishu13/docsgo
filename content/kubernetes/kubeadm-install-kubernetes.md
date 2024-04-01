+++
title = 'Kubeadm Install Kubernetes'
date = 2024-04-01T11:18:20+08:00
draft = true
+++

# Kubernetes + KubeSphere


## 准备工作

#### Harbor 部署

> harbor-offline-installer-v2.1.2.tgz

https://github.com/goharbor/harbor/releases/tag/v2.1.2

可参考安装文档：官方：https://goharbor.io/docs/2.1.0/install-config/quick-install-script/

上传harbor包，与 docker-compose-Linux-x86_64

```shell
mv /tmp/docker-compose-Linux-x86_64 /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

编辑配置文件

```shell
tar -zxvf /tmp/harbor-offline-installer-v2.1.2.tgz -C /root
```

进入解压的 `harbor` 目录

⚠️：拷贝harbor.yml的备份文件为 `harbor.yml`

1. 编辑文件 `/root/harbor/harbor.yml` 注释掉https的相关配置
2. 默认端口：`8080`
3. 主机地址：`hostname:`  修改为主机 `IP`
4. 存储目录: `data_volume: /data/harbor` 

运行脚本：

```shell
bash install.sh
```

创建Harbor 库 `bash create_project_harbor.sh`

⚠️：按需添加需要的库名称，运行该脚本

```shell
#!/usr/bin/env bash
url="https://dockerhub.kubekey.local"  
user="admin"
passwd="Harbor12345"

harbor_projects=(library
    kubesphere
    calico
)
for project in "${harbor_projects[@]}"; do
    echo "creating $project"
    curl -u "${user}:${passwd}" -X POST -H "Content-Type: application/json" "${url}/api/v2.0/projects" -d "{ \"project_name\": \"${project}\", \"public\": true}" -k
done
```

------

#### 镜像

1.20.6链接: https://caiyun.139.com/m/i?1L5BSzYqo0WAo  提取码:xeh2  

查看k8s版本对应的images

```shell
kubeadm config images list --kubernetes-version=1.20.6
```

------

#### 3）安装ansible配置

⚠️：建议在 Harbor 节点安装，作为主控端，其余节点作为被控端。

脚本`init_ansible.sh` 用于快速安装和配置 Ansible 环境，包括禁用主机密钥检查和设置解释器。

```shell
cat <<EOF > init_ansible.sh
#!/bin/bash
yum install -y ansible &>/dev/null
sed -i 's/^#host_key_checking = False/host_key_checking = False/' /etc/ansible/ansible.cfg
sed -i '/^\[defaults\]$/a interpreter_python = auto_legacy_silent' /etc/ansible/ansible.cfg
EOF
bash init_ansible.sh
```

生成 Ansible 的主机清单文件 `/etc/ansible/hosts`。这个清单文件定义了一组主机及其相关变量，参考一下示例：

```shell
cat <<EOF > /etc/ansible/hosts
[all]
c3-master1 ansible_host=10.232.23.157
c3-master2 ansible_host=10.232.23.131 
c3-master3 ansible_host=10.232.23.134
c3-node1 ansible_host=10.232.23.156

[all:vars]
ansible_become=true
ansible_become_method=sudo
ansible_become_pass=**-**
ansible_ssh_user=swdys
ansible_ssh_pass=**-**
EOF
```

------

## GFS部署(可选)

部署环境(研发环境)

参考：https://docs.gluster.org/en/latest/Install-Guide/Install/#community-packages



```shell
ansible gfs -m ping
ansible gfs -m shell -a 'yum install glusterfs-server -y'
ansible gfs -m shell -a 'systemctl start glusterd'
ansible gfs -m shell -a 'systemctl enable glusterd'
gluster volume create gv0 replica 3 master1:/data node1:/data node2:/data force
gluster volume start gv0
gluster volume info
ansible all -m shell -a 'yum install glusterfs-client -y'
cat <<EOF > gfs.yaml
---
- name: Create Mount Directories and Update /etc/fstab
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Install nfs-utils package
      package:
        name: glusterfs-client
        state: present

    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
      loop:
        - /var/openebs
        - /dockerdata-nfs

    - name: Add mount configuration to /etc/fstab
      lineinfile:
        path: /etc/fstab
        line: "{{ item.source }} {{ item.destination }} glusterfs defaults,_netdev 0 0"
      loop:
        - { source: "master1:/gv0", destination: "/var/openebs" }
        - { source: "master1:/gv0", destination: "/dockerdata-nfs" }

    - name: Mount file system
      command: mount -a
EOF
```

## 初始化环境

#### 1）修改hosts文件

编辑 `/etc/hosts` 文件，将 IP 地址和主机名的映射关系添加到该文件中，参考一下示例：

```shell
cat <<EOF >/etc/hosts
10.232.23.157 c3-master1
10.232.23.131 c3-master2
10.232.23.134 c3-master3
10.232.23.156 c3-node1 
EOF
```

用于设置主机的主机名:

```shell
cat <<EOF > hostname.yaml
- name: Hostname
  hosts: all
  become: true
  gather_facts: false

  tasks:
    - name: Set hostname
      shell: hostnamectl set-hostname "{{ inventory_hostname }}"
EOF
```

批量将指定的 IP 地址和主机名添加到所有主机的 `/etc/hosts` 文件中

```shell
cat <<EOF > hosts-all.yaml
- name: Append /etc/hosts
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Append /etc/hosts
      lineinfile:
        dest: /etc/hosts
        line: "{{ item.ip }} {{ item.hostname }}"
      loop:
        - { ip: "10.209.8.66", hostname: "mirrors.bclinux.org" }
        - { ip: "10.232.23.157", hostname: "c3-master1" }
        - { ip: "10.232.23.131", hostname: "c3-master2" }
        - { ip: "10.232.23.134", hostname: "c3-master3" }
        - { ip: "10.232.23.156", hostname: "c3-node1" }
EOF
```

#### 2）sudo账户（可选）

示例：创建用户wlznhpt 密码: 12345 sudu设置ALL=(ALL) NOPASSWD: ALL

```shell
cat <<EOF > create_user.yml   
---
- name: Create user and configure sudo permissions
  hosts: all
  become: yes
  gather_facts: false
  vars:
    username: wlznhpt
    password: 12345
  tasks:
    - name: Create user
      user:
        name: "{{ username }}"
        password: "{{ password | password_hash('sha512') }}"
        createhome: yes
      become: yes
      become_user: root

    - name: Add user to the wheel group
      user:
        name: "{{ username }}"
        groups: wheel
        append: yes
      become: yes
      become_user: root

    - name: Configure sudoers
      lineinfile:
        path: /etc/sudoers
        line: "{{ username }} ALL=(ALL) NOPASSWD: ALL"
        validate: 'visudo -cf %s'
        state: present
        backup: yes
      become: yes
      become_user: root

EOF
```

shell脚本 

```shell
cat <<EOF > sudo.sh
#!/bin/bash

# 设置要创建的用户名和密码
username="wlznhpt"
password="Wgzyc#@2017"

# 创建用户并设置密码
useradd -m "${username}"
echo "$username:${password}" | chpasswd

# 添加用户到wheel组
usermod -aG wheel "${username}"

# 配置sudoers
echo "${username} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# 重新加载sudoers文件以使更改生效
visudo -c -f /etc/sudoers && echo "sudoers file is valid" || echo "sudoers file is NOT valid"
```

#### 3）关闭swap、防火墙、selinux、修改limit

任务包括：

1. 设置 SELinux：通过修改 `/etc/selinux/config` 文件，将 SELinux 状态设置为 `disabled`。
2. 禁用 Swap：使用 `swapoff -a` 命令禁用当前的 Swap 分区，并通过修改 `/etc/fstab` 文件来永久禁用 Swap。
3. 停止和禁用防火墙
4. 修改 `limits.conf`：向 `/etc/security/limits.conf` 文件中添加行，修改系统的限制设置。

```shell
cat <<EOF > system.yml   
---
- name: System Settings
  hosts: all
  become: yes
  gather_facts: false
  tasks:
    - name: "Disable SELinux enforcing"
      command: /usr/sbin/setenforce 0  
      ignore_errors: true
  
    - name: Modify SELinux configuration file
      lineinfile:
        dest: /etc/selinux/config
        regexp: '^SELINUX='
        line: 'SELINUX=disabled'

    - name: Disable Swap
      shell: swapoff -a

    - name: Permanently Disable Swap
      lineinfile:
        dest: /etc/fstab
        state: absent
        regexp: '^.*\sswap\s.*$'

    - name: Stop and Disable Firewall
      service:
        name: firewalld
        state: stopped
        enabled: no
        
    - name: Modify limits
      lineinfile:
        path: /etc/security/limits.conf
        line: "{{ item }}"
      loop:
        - "*   hard   nofile  65535"
        - "*   soft   nofile  65535"
        - "*   soft   nproc   65535"
        - "*   hard   nproc   65535"
EOF
```

#### 4）基础软件安装

用于在目标主机上安装所需的软件包

```shell
cat <<EOF > install_soft.yaml
- name: install
  hosts: all
  become: yes
  tasks:
    - name: Install required packages
      yum:
        name: "{{ item }}"
        state: present
      loop:
        - socat
        - conntrack
        - ebtables
        - ipset
        - ipvsadm
        - bash-completion
        - openssl
        - openssl-devel
        - vim
        - telnet
EOF
```

#### 5）内核模块加载

用于在目标主机上加载内核模块。

```shell
cat <<EOF > load_kernel.yaml
- name: load_kernel
  hosts: all
  become: yes
  gather_facts: false
  tasks:  
    - name: "modprobe"
      modprobe:
       name: "{{ item }}"
       state: present
      with_items:
       - ip_vs
       - ip_vs_rr
       - ip_vs_wrr
       - ip_vs_sh
       - nf_conntrack 
    - name: "lsmod"  
      shell: /sbin/lsmod | grep {{ item }}
      with_items:
       - ip_vs
       - ip_vs_rr
       - ip_vs_wrr
       - ip_vs_sh
       - nf_conntrack
EOF
```

#### 6）安装Docker

版本：v 19.03.9

将生成一个名为 `/etc/docker/daemon.json` 的文件，并将以下内容写入该文件

⚠️注意修改仓库地址：`"insecure-registries":["10.232.23.144:8080"],` 为 `Harbor IP`

```shell
cat <<EOF > /etc/docker/daemon.json
{ 
  "data-root": "/data/docker/data-root",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "insecure-registries":["172.32.165.74:8080"],
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```

```shell
cat <<EOF > /etc/docker/daemon.json
{ 
  "data-root": "/data/docker/data-root",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "insecure-registries":["mirrors.com:80"],
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```



用于在目标主机上安装和配置 Docker（19.03.13版本：

```shell
cat <<EOF > docker.yaml
- name: Install Docker
  hosts: all
  become: yes
  gather_facts: false
  vars:
    docker_package: "docker-v.19.03.13.zip"
  tasks:  
    - name: Create deployment directory
      file:
        path: /root/deployment/Kube-v1.20.6/kube/rpm
        state: directory
        mode: '0755'
        
    - name: Copy Docker RPM package
      copy:
        src: "/root/deployment/Kube-v1.20.6/kube/rpm/{{ docker_package }}"
        dest: "/root/deployment/Kube-v1.20.6/kube/rpm/{{ docker_package }}"
        mode: 0644

    - name: Unzip Docker
      shell: "unzip /root/deployment/Kube-v1.20.6/kube/rpm/{{ docker_package }} -d /root/deployment/Kube-v1.20.6/kube/rpm/"

    - name: Install Docker RPM
      shell: "yum install -y /root/deployment/Kube-v1.20.6/kube/rpm/dockerrpm/*.rpm"

    - name: Start Docker
      systemd:
        name: docker
        state: started
        enabled: yes
    
    - name: Copy daemon.json configuration file
      copy:
        src: /etc/docker/daemon.json
        dest: /etc/docker/daemon.json
      notify: Restart Docker

  handlers:
    - name: Restart Docker
      systemd:
        name: docker
        state: restarted
EOF
```

所有主机登陆：`admin/Harbor12345`:Harbor默认地址（注意修改仓库地址）

```shell
 ansible all -m shell -a 'docker login 172.32.165.74:8080:8080 -uadmin -pHarbor12345'
```

#### 7）内核配置

用于在目标主机上修改内核参数。

```shell
cat <<EOF > modify_kernel.yaml
- name: Modify Kernel Parameters
  hosts: all
  become: yes
  gather_facts: false
  tasks:
    - name: Set sysctl parameters
      lineinfile:
        path: /etc/sysctl.conf
        line: "{{ item }}"
        state: present
      loop:
        - "net.bridge.bridge-nf-call-ip6tables = 1"
        - "net.bridge.bridge-nf-call-iptables = 1"
        - "vm.swappiness=0"
        - "net.core.somaxconn=32768"
        - "net.ipv4.ip_forward=1"
        - "fs.inotify.max_user_watches = 2097152"
      become: true

    - name: Reload sysctl
      command: sysctl -p
      become: true
EOF
```

#### 8）NFS挂载（可选）

在所有主机上创建挂载目录、安装 nfs-utils 包、配置 /etc/fstab 文件并挂载 NFS 文件系统

```shell
cat <<EOF > nfs.yaml
---
- name: Create Mount Directories and Update /etc/fstab
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: Install nfs-utils package
      package:
        name: nfs-utils
        state: present

    - name: Create directories
      file:
        path: "{{ item }}"
        state: directory
      loop:
        - /var/openebs
        - /dockerdata-nfs

    - name: Add mount configuration to /etc/fstab
      lineinfile:
        path: /etc/fstab
        line: "{{ item.source }} {{ item.destination }} nfs rw,nfsvers=3 0 0"
      loop:
        - { source: "zdnb1.dynamic2.cmcc.com:/share-d74931bb", destination: "/var/openebs" }
        - { source: "zdnb1.dynamic2.cmcc.com:/share-d74931bb", destination: "/dockerdata-nfs" }

    - name: Mount file system
      command: mount -a
EOF
```

#### 9）时间同步

⚠️：先查看是否开启时间同步，已有则忽略此步骤

在所有主机上进行时间同步的过程。

- 包括安装 `chronyd` 软件包、修改配置文件、重启服务、开启时间同步功能，并显示节点的当前时间

```shell
cat <<EOF > chronyd.yaml
---
- name: Time Synchronization
  hosts: all
  become: true
  gather_facts: false
  vars:
    ntp_server: # NTP server address
    ntp_cidr: # Allow hosts from the local network to access the time server
  tasks:
    - name: Install the chronyd package
      dnf:
        name: chrony
        state: present

    - name: Modify the time server configuration file
      copy:
        content: |
          server {{ ntp_server }} iburst
          allow {{ ntp_cidr }}
          local stratum 10
          driftfile /var/lib/chrony/drift
          makestep 1.0 3
          rtcsync
          logdir /var/log/chrony
          log measurements statistics tracking
        dest: "/etc/chrony.conf"

    - name: Restart chronyd service
      service:
        name: chronyd
        state: restarted
        enabled: true

    - name: Enable time synchronization
      command: timedatectl set-ntp true

    - name: Display node time
      command: date
      register: current_date

    - name: Print node time
      debug:
        var: current_date.stdout
EOF
```

## 高可用配置

#### 1）Keepalived 和 HAproxy

使用 Keepalived 和 HAproxy 创建高可用 Kubernetes 集群

高可用 Kubernetes 集群能够确保应用程序在运行时不会出现服务中断，这也是生产的需求之一

用于在`master`节点上安装和配置 HAProxy 和 Keepalived。

⚠️：修改 `hosts` 为master节点 的`hostname`

```shell
cat <<EOF > lb.yaml
- name: Load Balancer Configuration
  hosts: c3-master1,c3-master2,c3-master3
  become: yes
  gather_facts: false
  tasks:
    - name: Install required packages
      package:
        name:
          - haproxy
          - keepalived
        state: present

    - name: Start and enable HAProxy on boot
      service:
        name: haproxy
        state: started
        enabled: true
    
    - name: Start and enable Keepalived on boot
      service:
        name: keepalived
        state: started
        enabled: true
EOF
```

用于负载均衡 Kubernetes API Server。文件中包含了全局设置和默认配置，以及针对 `kube_apiserver` 的前端和后端配置：

⚠️：需修改 `backend kube_apiserver_back` 中 `server` 为master节点信息

```shell
cat <<EOF > /root/deploy/lb/haproxy.cfg
global
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     40000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
#    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000


frontend kube_apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kube_apiserver_back
backend kube_apiserver_back
    mode        tcp
    balance     roundrobin
    server  c3-master1 10.232.23.157:6443 check
    server  c3-master2 10.232.23.131:6443 check
    server  c3-master3 10.232.23.134:6443 check
EOF
```



复制 `haproxy.cfg` 文件到指定位置，并根据主机名的条件复制相应的 `keepalived.conf` 文件

⚠️：修改五处信息：`hosts`、`vars`、`interface_name`、`src` 、`when`

```shell
cat <<EOF > lbfile.yaml
- name: Load Balancer Configuration
  hosts: c3-master1,c3-master2,c3-master3
  become: yes
  gather_facts: false
  vars:
    virtual_ip: "10.232.23.159"
    interface_name: "eth1"        
  tasks:
    - name: Copy haproxy.cfg file
      copy:
        src: /root/deploy/lb/haproxy.cfg
        dest: /etc/haproxy/haproxy.cfg
        
    - name: Copy master1_keepalived.conf to c3-master1
      copy:
        content: |
          ! Configuration File for keepalived
          global_defs {
             router_id LVS_DEVEL
          }
          vrrp_instance VI_1 {
              state MASTER
              interface {{ interface_name }}
              virtual_router_id 126
              priority 120
              nopreempt
              advert_int 1
              authentication {
                  auth_type PASS
                  auth_pass K8SHA_KA_AUTH
              }
              virtual_ipaddress {
                  {{ virtual_ip }}
              }
          }
        dest: /etc/keepalived/keepalived.conf
      when: inventory_hostname == ''
    
    - name: Copy master2_keepalived.conf to c3-master2
      copy:
        content: |
          ! Configuration File for keepalived
          global_defs {
             router_id LVS_DEVEL
          }
          vrrp_instance VI_1 {
              state BACKUP
              interface {{ interface_name }}
              virtual_router_id 126
              priority 100
              nopreempt
              advert_int 1
              authentication {
                  auth_type PASS
                  auth_pass K8SHA_KA_AUTH
              }
              virtual_ipaddress {
                  {{ virtual_ip }}
              }
          }
        dest: /etc/keepalived/keepalived.conf
      when: inventory_hostname == ''
    
    - name: Copy master3_keepalived.conf to c3-master3
      copy:
        content: |
          ! Configuration File for keepalived
          global_defs {
             router_id LVS_DEVEL
          }
          vrrp_instance VI_1 {
              state BACKUP
              interface {{ interface_name }}
              virtual_router_id 126
              priority 80
              nopreempt
              advert_int 1
              authentication {
                  auth_type PASS
                  auth_pass K8SHA_KA_AUTH
              }
              virtual_ipaddress {
                  {{ virtual_ip }}
              }
          }
        dest: /etc/keepalived/keepalived.conf
      when: inventory_hostname == ''
      
    - name: Restart HAProxy service
      service:
        name: haproxy
        state: restarted
        
    - name: Restart Keepalived service
      service:
        name: keepalived
        state: restarted
EOF
```

最后使用查看是否监听

```shell
netstart -lntp | grep 16443
```

## 初始化集群

### kubeadm + ks

#### kubeadm

```shell
cat <<EOF > kube120.yaml
- name: kube 120
  hosts: all
  become: yes
  gather_facts: false
  vars:
    kube_package: kube120.zip
    dir: "/home/secu/"
  tasks:  
    - name: cp kube RPM all
      copy:
        src: {{ dir }}{{ kube_package }}
        dest: /home/secu/{{ kube_package }}
        mode: 0644
      
    - name: uzip kube120 
      shell: "unzip /home/secu/{{ kube_package }} -d /home/secu/"

    - name: install kube120 RPM 
      shell: "yum install -y /home/secu/kube120/*.rpm"

    - name: start kubelet 
      service:
        name: kubelet
        state: started
        enabled: true
EOF
```

kubeadm命令初始化

```shell
kubeadm init --kubernetes-version=v1.20.6 \
  --apiserver-advertise-address=172.32.165.68 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/16 \
  --upload-certs \
  --v=5 \
  --image-repository=10.232.23.144:8080/kubesphere \
  --control-plane-endpoint=10.232.23.159:16443
```

初始化失败执行：然后重新初始化

```shell
kubeadm reset -f ; ipvsadm --clear  ; rm -rf ~/.kube
```

查看kubelet 日志



#### calico安装

仅在 `master1` 节点上执行以下操作：

下载 Calico 配置文件，确保版本匹配：

- calico验证版本是否适配k8s集群版本：https://docs.tigera.io/calico/3.25/getting-started/kubernetes/requirements
- YAML 文件：[calico.yaml v3.25](https://docs.projectcalico.org/v3.25/manifests/calico.yaml)

修改 `calico.yaml` 文件中的镜像地址

```shell
sed -i 's/image: calico/image: 10.217.2.67:8080\/calico/g' calico.yaml
kubectl apply -f calico.yaml
```

修改`calico.yaml`

在

```shell
- name: CLUSTER_TYPE
  value: "k8s,bgp"
```

下增加两行

```shell
- name:IP_AUTODETECTION_METHOD
  value:"interface=enp0s3"
```

`enp0s3`是我机器的网卡名

#### NFS SC (可选)

Storage Classes

使用kubeadm + ks 需要提前 提供 默认的Storage Classes

NFS：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner

目的：ks部署时需要：根据PVC 去动态制备PV

准备工作：

1. 安装 Helm
2. 镜像 registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
3. 准备 Charts 包

```shell
helm install my-release nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=x.x.x.x --set nfs.path=/exported/path -n kube-system
```

#### GFS

https://www.kubesphere.io/zh/docs/v3.4/reference/storage-system-installation/glusterfs-server/

```shell
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
type: kubernetes.io/glusterfs
data:
  key: "MTIzNDU2"    #请替换为您自己的密钥。Base64 编码。
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/supported-access-modes: '["ReadWriteOnce","ReadOnlyMany","ReadWriteMany"]'
  name: glusterfs
parameters:
  clusterid: "ab2a2129-4728-4ae5-af26-30816bb5c9c5"    #请替换为您自己的 GlusterFS 集群 ID。
  gidMax: "50000"
  gidMin: "40000"
  restauthenabled: "true"
  resturl: "http://192.168.0.2:8080"    #Gluster REST 服务/Heketi 服务 URL 可按需供应 gluster 存储卷。请替换为您自己的 URL。
  restuser: admin
  secretName: heketi-secret
  secretNamespace: kube-system
  volumetype: "replicate:3"    #请替换为您自己的存储卷类型。
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

#### KS

建议参考官方文档部署（离线安装）：[部署文档](https://v3-1.docs.kubesphere.io/zh/docs/installing-on-kubernetes/on-prem-kubernetes/install-ks-on-linux-airgapped/)

部署：

1. 镜像仓库，准备安装镜像，推送镜像至私有仓库
2. 下载部署两个文件

```shell
https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/cluster-configuration.yaml
https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/kubesphere-installer.yaml
```

1. 编辑 r 添加您的私有镜像仓库

⚠️：`dockerhub.kubekey.local` 换成：HarborIP:8080

```shell
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  local_registry: dockerhub.kubekey.local # Add this line manually; make sure you use your own registry address.
```

1. 安装

```shell
kubectl apply -f kubesphere-installer.yaml
kubectl apply -f cluster-configuration.yaml
```

检查安装日志：

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

通过 NodePort (`IP:30880`) 使用默认帐户和密码 (`admin/P@88w0rd`) 访问 Web 控制台。



### KubeKey

------

参考： [在Linux上安装 KubeSphere 和 Kubernetes](https://www.kubesphere.io/zh/docs/v3.3/installing-on-linux/introduction/air-gapped-installation/)

版本对应：https://kubesphere.io/zh/docs/v3.4/installing-on-linux/introduction/kubekey/

#### 离线安装集群方式

部署文件

```shell
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/cluster-configuration.yaml
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.1.1/kubesphere-installer.yaml
```

执行以下命令下载 KubeKey 并解压：

```shell
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -
```

一、示例配置文件

执行以下命令生成示例配置文件用于安装：

```shell
./kk create config [--with-kubernetes version] [--with-kubesphere version] [(-f | --file) path]
```

例如：

```shell
./kk create config --with-kubernetes v1.17.9 --with-kubesphere v3.1.1 -f config-sample.yaml
```

备注

- 请确保 Kubernetes 版本和您下载的版本一致。
- 如果您在这一步的命令中不添加标志 `--with-kubesphere`，则不会部署 KubeSphere，只能使用配置文件中的 `addons` 字段安装，或者在您后续使用 `./kk create cluster` 命令时再次添加这个标志。

二、编辑配置文件

注意点：

1. 离线安装时，必须指定 `privateRegistry`，在本示例中是 `dockerhub.kubekey.local`。
2. k8s 版本
3. lb.kubesphere.local 地址
4. privateRegistry 镜像仓库地址

```shell
apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, password: Qcloud@123}
  - {name: node1, address: 192.168.0.3, internalAddress: 192.168.0.3, password: Qcloud@123}
  - {name: node2, address: 192.168.0.4, internalAddress: 192.168.0.4, password: Qcloud@123}
  roleGroups:
    etcd:
    - master
    master:
    - master
    worker:
    - master
    - node1
    - node2
  controlPlaneEndpoint:
    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.17.9
    imageRepo: kubesphere
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    registryMirrors: []
    insecureRegistries: []
    privateRegistry: dockerhub.kubekey.local  # Add the private image registry address here. 
  addons: []


---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.1.1
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  zone: ""
  local_registry: ""
  etcd:
    monitoring: false
    endpointIps: localhost
    port: 2379
    tlsEnable: true
  common:
    redis:
      enabled: false
    redisVolumSize: 2Gi
    openldap:
      enabled: false
    openldapVolumeSize: 2Gi
    minioVolumeSize: 20Gi
    monitoring:
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
    es:
      elasticsearchMasterVolumeSize: 4Gi
      elasticsearchDataVolumeSize: 20Gi
      logMaxAge: 7
      elkPrefix: logstash
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchUrl: ""
      externalElasticsearchPort: ""
  console:
    enableMultiLogin: true
    port: 30880
  alerting:
    enabled: false
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:
    enabled: false
  devops:
    enabled: false
    jenkinsMemoryLim: 2Gi
    jenkinsMemoryReq: 1500Mi
    jenkinsVolumeSize: 8Gi
    jenkinsJavaOpts_Xms: 512m
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
  events:
    enabled: false
    ruler:
      enabled: true
      replicas: 2
  logging:
    enabled: false
    logsidecar:
      enabled: true
      replicas: 2
  metrics_server:
    enabled: false
  monitoring:
    storageClass: ""
    prometheusMemoryRequest: 400Mi
    prometheusVolumeSize: 20Gi
  multicluster:
    clusterRole: none
  network:
    networkpolicy:
      enabled: false
    ippool:
      type: none
    topology:
      type: none
  notification:
    enabled: false
  openpitrix:
    store:
      enabled: false
  servicemesh:
    enabled: false
  kubeedge:
    enabled: false
    cloudCore:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      cloudhubPort: "10000"
      cloudhubQuicPort: "10001"
      cloudhubHttpsPort: "10002"
      cloudstreamPort: "10003"
      tunnelPort: "10004"
      cloudHub:
        advertiseAddress:
          - ""
        nodeLimit: "100"
      service:
        cloudhubNodePort: "30000"
        cloudhubQuicNodePort: "30001"
        cloudhubHttpsNodePort: "30002"
        cloudstreamNodePort: "30003"
        tunnelNodePort: "30004"
    edgeWatcher:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      edgeWatcherAgent:
        nodeSelector: {"node-role.kubernetes.io/worker": ""}
        tolerations: []
```

#### 开始安装

确定完成上面所有步骤后，您可以执行以下命令。

```shell
./kk create cluster -f config-sample.yaml
```

将可执行文件 `kk` 和包含 Kubernetes 二进制文件的文件夹 `kubekey` 传输至任务机机器用于安装后，必须将它们放在相同目录中，然后再执行上面的命令。

检查安装日志：

```shell
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
```

通过 `http://{IP}:30880` 使用默认帐户和密码 `admin/P@88w0rd` 访问 KubeSphere 的 Web 控制台。
