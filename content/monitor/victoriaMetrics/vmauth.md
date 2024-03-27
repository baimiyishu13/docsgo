+++
title = 'Vmauth'
date = 2024-03-27T16:38:42+08:00
draft = true
+++

简单身份验证代理

下载：https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.99.0/vmutils-linux-amd64-v1.99.0.tar.gz

减压到：`/usr/local/bin`



system 文件

+ 使用了TLS 加密
+ 配置文件参考官方文档即可

```sh
cat <<EOF > /etc/systemd/system/vmauth.service
[Unit]
Description=Description=vmauth.service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/vmauth-prod -auth.config=/etc/auth/config.yml -tls -tlsCertFile=/root/.ssh/cert.pem -tlsKeyFile=/root/.ssh/key.pem
SyslogIdentifier=victoriametrics
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

