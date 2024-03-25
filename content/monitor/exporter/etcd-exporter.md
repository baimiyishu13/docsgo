+++
title = 'Etcd Exporter'
date = 2024-03-25T09:28:11+08:00
draft = false
+++
特定环境，每个节点 `/etc/ssl/etcd/ssl` 有etcd 证书

+ 将以下配置写入：`vmagent.yaml`

‼️ 注意修改证书名为实际证书名，修改 `etcd-ip`

```sh
  - job_name: "etcd"
    scheme: https
    tls_config:
      insecure_skip_verify: true
      cert_file: /etc/ssl/etcd/ssl/node-dev-master.pem
      key_file: /etc/ssl/etcd/ssl/node-dev-master-key.pem
      ca_file: /etc/ssl/etcd/ssl/ca.pem
    static_configs:
      - targets: ['etcd-ip1:2379','etcd-ip2:2379','etcd-ip3:2379']
```

