```
ansible all -m copy -a "src=./bin2/node_exporter dest=/usr/local/bin/node_exporter mode=0755"

ansible all -m template -a "src=node_exporter.service.j2 dest=/etc/systemd/system/node_exporter.service"

ansible all -m systemd -a "name=node_exporter.service state=started,enable=yes"
ansible all -m systemd -a "name=node_exporter.service state=enabled"
ansible all -m command -a "systemctl status node_exporter.service"
```



```
'172.15.89.83:9101','172.15.89.84:9101','172.15.89.85:9101','172.15.89.86:9101','172.15.89.87:9101','172.15.89.88:9101','172.15.89.89:9101','172.15.89.91:9101','172.15.89.92:9101','172.15.89.93:9101','172.15.89.94:9101','172.15.89.95:9101','172.15.89.96:9101','172.15.89.97:9101','172.15.89.98:9101','172.15.89.99:9101','172.15.89.100:9101','172.15.89.111:9101','172.15.89.112:9101','172.15.89.113:9101
```



```sh
'172.15.89.89:2379','172.15.89.91:2379','172.15.89.92:2379'
```



```sh
10.54.1.90 m1
10.54.1.91 m2
10.54.1.92 d1
10.54.1.93 d2
10.54.1.94 d3
10.54.1.95 elastic
10.54.1.96 w1
10.54.1.97 w2
10.54.1.98 w3

'10.54.1.90:9101','10.54.1.91:9101','10.54.1.92:9101','10.54.1.93:9101','10.54.1.94:9101','10.54.1.95:9101','10.54.1.96:9101','10.54.1.97:9101','10.54.1.98:9101'
```



```sh
cat <<EOF > /etc/systemd/system/mongodb_exporter.service
[Unit]
Description=mongodb_exporter
After=network.target

[Service]
Type=simple
Environment="MONGODB_URI=mongodb://prometheus:prometheus@10.54.1.92:27017/admin?ssl=true&sslclientcertificatekeyfile=/etc/ssl/mongo/bl-datanode01.pem&sslinsecure=true&sslcertificateauthorityfile=/etc/ssl/mongo/ca.crt"
ExecStart=/usr/local/bin/mongodb_exporter

[Install]
WantedBy=multi-user.target
EOF
```

