+++
title = 'Nginx Ingress重定向'
date = 2024-04-08T10:46:46+08:00
draft = true
+++

官网参考：https://kubernetes.github.io/ingress-nginx/examples/rewrite/

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/app-root: /pumer/project/**
```

测试访问：

```sh
[root@k8sworker1 ~]# curl portal.**.com
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>nginx/1.13.9</center>
</body>
</html>
```

