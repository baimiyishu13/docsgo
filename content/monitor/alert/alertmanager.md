+++
title = 'Alertmanager'
date = 2024-03-27T16:03:21+08:00
draft = true
+++

二进制安装：

下载：https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

下载减压 - > `/usr/local/bin`

```sh
cta <<EOF >/etc/systemd/system/alertmanager.service 
### Alertmanager systemd 
[Unit]
Description=Alertmanager
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/etc/alertmanager/alertmanager.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

配置文件：alertmanager.yml

+ 参考官方配置

示例：企业微信

```sh
global:
  resolve_timeout: 5m
 
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 1m
  repeat_interval: 30m
  receiver: 'web.hook'
 
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://localhost:8999/webhook'
    send_resolved: true
```



访问：9093端口
