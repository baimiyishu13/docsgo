+++
title = 'Node Exporter'
date = 2024-03-25T09:28:48+08:00
draft = true
+++
🎉 默认端口 9100，因为与业务端口冲突，所以此处为9101

##### node exporter

1.下载

```sh
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
```

2.减压后node_exporter移动

```sh
tar -zxvf node_exporter-1.7.0.linux-amd64.tar.gz -C /usr/local/bin/ --strip-components=1
```

3.service 文件

端口默认9100，会与yugabyte 端口冲突

```sh
cat <<EOF | tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/node_exporter  --web.listen-address=:9101
Restart=always

[Install]
WantedBy=multi-user.target

EOF
```

4. 启动服务

```sh
systemctl daemon-reload
systemctl restart node_exporter.service
systemctl status node_exporter.service
systemctl enable node_exporter.service
```

接入：
```shell
- job_name: 'node'
  static_configs:
  - targets: ['localhost:9101']
```