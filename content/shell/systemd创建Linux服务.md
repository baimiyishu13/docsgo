+++
title = 'Systemd创建Linux服务'
date = 2024-04-01T17:03:58+08:00
draft = true

+++

## Systemd创建Linux服务

🤔 需要创建一个脚本并在后台运行它

### 可以采取几种方法

1. 也许最简单的方法就是在命令末尾添加一个`&`  另外，如果想关闭终端，追加`nohup`。⚠️ 如果的脚本由于某种原因存在，需要手动重新启动它。
2. 更好的方法是使用Linux 的： [systemd](https://systemd.io/)

展示如何创建一个简单的 systemd 服务，该服务可以在后台运行脚本并在出现故障时重新启动它



### systemd 启动示例

1. 创建一个简单的 Python HTTP 服务器 

```sh
vim my_server.py
```

2. 接受 HTTP GET 请求并将`Hello`字符串返回给客户端

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

hostName = "localhost"
serverPort = 8080


class MyServer(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("Hello\n", "utf-8"))


if __name__ == "__main__":
    webServer = HTTPServer((hostName, serverPort), MyServer)
    print("Server started http://%s:%s" % (hostName, serverPort))

    try:
        webServer.serve_forever()
    except KeyboardInterrupt:
        pass

    webServer.server_close()
```

尝试在前台运行，确保py脚本正常

```sh
python3 my_server.py
```

然后打开一个新窗口并，尝试使用命令访问服务器`curl`。

```sh
curl localhost:8080/hello
```



3. 创建 Systemd Linux 服务

```sh
vim /etc/systemd/system/my-server.service
```

确保指定的命令

+  L`User`和`ExecStart`启动服务器的命令
+ `Environment`：向脚本提供变量。（可选）
+ `Restart`：重要的参数，可以将其设置为`always` 或 `on-failure`。默认情况下，`Restart=always,  如果在 10 秒间隔内启动失败超过 5 次，systemd 将放弃重新启动服务。`RestartSec`指令也会对结果产生影响：如果将其设置为 3 秒后重新启动，那么永远不会在 10 秒内重试 5 次失败。

​	**始终有效的简单修复方法是设置**`StartLimitIntervalSec=0`.这样，systemd 将尝试永远重新启动服务。不过，最好设置`RestartSec`为至少 1 秒，以避免在出现问题时给服务器带来太大压力。

```sh
[Unit]
Description=My Server
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/ubuntu/my_server.py

User=ubuntu

Environment=ENV=production

Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

测试

```sh
sudo systemctl start my-server
sudo systemctl enable my-server
```

然后检查状态：

```sh
sudo systemctl status my-server
```

如果失败，并且需要查找原因，或者只想跟踪日志，则可以使用命令`journactl`。

```sh
journalctl -u my-server -f --no-pager
```

用于`curl`测试。

```
curl localhost:8080/hello
```



### 管理服务的依赖项

讨论的最后一件事是可以管理服务的**依赖项**。

​	例如，如果服务器需要数据库，可以使用`After=postgresql.service`指令指示 systemd 仅在 Postgres 启动后启动 python 服务器。

​	**只有当 Linux 启动并且这两个服务都启用时，这才会起作用**。

​	如果需要，可以添加另一个`Requires=postgresql.service`指令。即使 Progress 已关闭并且尝试启动 Python 服务器，systemd 也会首先启动 Postgres，然后启动程序。
