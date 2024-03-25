+++
title = 'Elasticsearch Exporter'
date = 2024-03-25T09:27:13+08:00
draft = false
+++
 监控 9114端口

下载减压：

```sh
wget https://github.com/prometheus-community/elasticsearch_exporter/releases/download/v1.5.0/elasticsearch_exporter-1.5.0.linux-amd64.tar.gz
```

2.减压后node_exporter移动

```sh
cp elasticsearch_exporter-1.5.0.linux-amd64/elasticsearch_exporter /usr/local/bin/
```

3.service 文件

⚠️ --es.uri 参数地址指定为 elasticsearch 节点IP

```sh
cat <<EOF | tee /etc/systemd/system/elasticsearch_exporter.service
[Unit]
Description=elasticsearch_exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/elasticsearch_exporter  --es.uri http://10.61.200.236:9200 --web.listen-address 0.0.0.0:9114
Restart=always

[Install]
WantedBy=multi-user.target

EOF
```

4. 启动服务

```shell
systemctl daemon-reload
systemctl restart elasticsearch_exporter
systemctl status elasticsearch_exporter
systemctl enable elasticsearch_exporter
```

5. 接入：

```shell
  - job_name: 'elasticsearch_exporter'
    static_configs:
    - targets: ['10.61.200.236:9114']
```



Grafana 导入ID：14191