+++
title = 'Mongodb Exporter'
date = 2024-03-25T09:28:25+08:00
draft = false
+++
🎉 mongodb exporter

---

下载：

```sh
wget https://github.com/percona/mongodb_exporter/releases/download/v0.11.2/mongodb_exporter-0.11.2.linux-amd64.tar.gz
```

减压后将 `mongodb_exporter` 二进制文件移动：

```sh
mv mongodb_exporter /usr/local/bin/
```

在 MongoDB 中创建一个名为 `prometheus` 的用户

1. 进入mongodb

```sh
mongo mongo.skydns.local --ssl --sslPEMKeyFile /etc/ssl/mongo/client.pem --sslCAFile /etc/ssl/mongo/ca.crt  --authenticationDatabase '$external' --authenticationMechanism MONGODB-X509
```

2. 创建用户

```sh
use admin
db.createUser({
    user: "prometheus",
    pwd: "prometheus",
    roles: [
        { role: "read", db: "admin" },
        { role: "readAnyDatabase", db: "admin" },
        { role: "clusterMonitor", db: "admin" }
    ]
});
```



创建service文件

‼️ 修改证书路径 和 IP

```sh
cat <<EOF > /etc/systemd/system/mongodb_exporter.service
[Unit]
Description=mongodb_exporter
After=network.target

[Service]
Type=simple
Environment="MONGODB_URI=mongodb://prometheus:prometheus@1.1.1.1:27017/admin?ssl=true&sslclientcertificatekeyfile=/etc/ssl/mongo/dev-data1.pem&sslinsecure=true&sslcertificateauthorityfile=/etc/ssl/mongo/ca.crt"
ExecStart=/usr/local/bin/mongodb_exporter

[Install]
WantedBy=multi-user.target
EOF
```

sed 修改 `mongodb_exporter.service` 文件中`1.1.1.1`  为 	`mongodb host IP`

```sh
sed -i "s/1.1.1.1/mongodb-hostip/g" /etc/systemd/system/mongodb_exporter.service
```

重新加载 systemd 的配置

```sh
systemctl daemon-reload
systemctl restart mongodb_exporter.service
systemctl enable mongodb_exporter.service
systemctl status mongodb_exporter.service
```

验证：

```sh
curl 127.0.0.1:9216/metrics
```

配置

```sh
  - job_name: mongodb
    static_configs:
    - targets: ['10.0.0.5:9216']
```

