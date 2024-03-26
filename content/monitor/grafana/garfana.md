+++
title = 'Garfana'
date = 2024-03-26T10:57:21+08:00
draft = true
+++

### 安装Grafna

访问：https://grafana.com/docs/grafana/latest/setup-grafana/installation/



### 设置 Grafana HTTPS 以确保网络流量安全

访问：https://grafana.com/docs/grafana/latest/setup-grafana/set-up-https/

步骤

1. 生成自签名证书
2. 生成自签名证书

打开该`grafana.ini`文件并编辑以下配置参数：

```ini
[server]
http_addr =
http_port = 3000
domain = mysite.com
root_url = https://subdomain.mysite.com:3000
cert_key = /etc/grafana/grafana.key
cert_file = /etc/grafana/grafana.crt
enforce_domain = False
protocol = https
```

**注意**：SSL 流量的标准端口是 443，您可以使用它来代替 Grafana 的默认端口 3000。
