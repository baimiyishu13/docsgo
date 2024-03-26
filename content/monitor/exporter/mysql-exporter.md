+++
title = 'Mysql Exporter'
date = 2024-03-25T09:28:39+08:00
draft = true
+++
🎉 暴露端口 9104

1. 创建 /etc/mysqlexporter 目录并在其中创建 .my.cnf 文件，该文件包含 MySQL 用户名和密码。

🔔 user、password 数据库用户密码

```sh
mkdir -p /etc/mysqlexporter
echo -e "[client]\nuser=$user\npassword=$password" > /etc/mysqlexporter/.my.cnf
```



2. 将 mysqld_exporter 复制到 /usr/local/bin/ 目录。

```sh
cp ./bin/mysqld_exporter /usr/local/bin/
```

3. 创建一个名为 mysql_exporter.service 的 systemd 服务文件，该文件在 /etc/systemd/system/ 目录下。

```sh
cat <<EOF > /etc/systemd/system/mysql_exporter.service
[Unit]
Description=mysql Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter  --config.my-cnf=/etc/mysqlexporter/.my.cnf

[Install]
WantedBy=multi-user.target
EOF
```

4. 重新加载 systemd 的配置，启动 mysql_exporter.service 服务，并将其设置为开机启动。

```sh
systemctl daemon-reload
systemctl start mysql_exporter.service
systemctl enable mysql_exporter.service
```

5. 验证

```sh
curl 127.0.0.1:9104/metrics
```

接入：

```shell
- job_name: 'mysql'
  static_configs:
  - targets: ['mysql-exporter:9104']
```

Grafana 模版 ID ：17320

