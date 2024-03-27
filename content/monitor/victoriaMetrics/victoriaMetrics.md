+++
title = 'VictoriaMetrics'
date = 2024-03-27T16:33:08+08:00
draft = true
+++

安装方式多种参考官网文档

二进制安装：

1. 下载：https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.99.0/victoria-metrics-linux-amd64-v1.99.0.tar.gz

解压到 `/usr/local/bin`

2. 创建数据目录：例如 `/data/victoria-metrics`

启动文件：

+ 兼容 prometheus 配置文件格式

```sh
cat <<EOF >/etc/systemd/system/victoria-metrics.service 
[Unit]
Description=Description=VictoriaMetrics service
After=network.target

[Service]
Type=simple
LimitNOFILE=2097152
ExecStart=/usr/local/bin/victoria-metrics-prod -storageDataPath=/data/victoria-metrics -promscrape.config=/etc/victoriametrics/prometheus.yaml
SyslogIdentifier=victoriametrics
Restart=always

PrivateTmp=yes
ProtectHome=yes
NoNewPrivileges=yes

ProtectSystem=full

[Install]
WantedBy=multi-user.target
```



暴露端口：8428
