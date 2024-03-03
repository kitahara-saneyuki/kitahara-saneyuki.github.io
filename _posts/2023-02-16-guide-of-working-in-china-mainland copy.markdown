---
layout:     single
title:      "指南：使用阿里云ecs服务器进行内网穿透，ssh远程连接"
date:       2023-02-16 03:37:14 +0800
categories: Terraform
published:  true
---

受不了天天上学背着习武之人六斤重的笔电？
想要在背着轻薄本满世界乱跑的时候和大语言模型愉快玩耍？
恨透了笔电跑数据时候风扇旋转制造的噪音和热量？
想要在家里架设服务器暴露到网络上？
这一切都可以解决！

配置环境：
- 笔电1：Kubuntu 23.10
- 笔电2：任何一个你能玩 VSCode 的平台，你如果足够吃多了撑的，可以在 iPad Pro 12.9 寸的屏幕上装个 UTM + Ubuntu + VSCode 跑

## 架设 ssh 服务器

在笔电 1 上安装 openssh

```bash
sudo apt install openssh-server openssh-client
```

## 在阿里云ECS服务器端安装 frp Server

我们需要建立一个配置文件 `frps.toml`

```toml
# frps.toml
bindPort = 7000 # 服务端与客户端通信端口

transport.tls.force = true # 服务端将只接受 TLS链接

auth.token = "token" # 身份验证令牌，frpc要与frps一致

# Server Dashboard，可以查看frp服务状态以及统计信息
webServer.addr = "0.0.0.0" # 后台管理地址
webServer.port = 7500 # 后台管理端口
webServer.user = "admin" # 后台登录用户名
webServer.password = "token" # 后台登录密码
```

运行下述命令，安装 frps

```bash
wget https://github.com/fatedier/frp/releases/download/v0.54.0/frp_0.54.0_linux_amd64.tar.gz
tar xzvf "frp_0.54.0_linux_amd64.tar.gz"
rm frp_0.54.0_linux_amd64.tar.gz*
sudo cp ./frp_0.54.0_linux_amd64/frps /usr/bin
rm -rf "frp_0.54.0_linux_amd64"
sudo mkdir -p /etc/frp /var/frp
sudo cp ./frp/frps.toml /etc/frp/frps.toml
```

建立配置文件 `frps.service`

```ini
[Unit]
Description=Frp Server Service
After=network.target
[Service]
Type=simple
DynamicUser=yes
Restart=on-failure
RestartSec=5s
ExecStart=/usr/bin/frps -c /etc/frp/frps.toml
LimitNOFILE=1048576
[Install]
WantedBy=multi-user.target
```

运行下述命令，安装并启动 frps 服务

```bash
sudo cp ./frp/frps.service /etc/systemd/system/frps.service
sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
```

## 在内网服务器端安装 frp Client

我们需要建立一个配置文件 `frpc.toml`

```toml
# frpc.toml
transport.tls.enable = true # 从 v0.50.0版本开始，transport.tls.enable的默认值为 true
serverAddr = "<你的阿里云ECS服务器IP地址>"
serverPort = 7000 # 公网服务端通信端口

auth.token = "token" # 令牌，与公网服务端保持一致

[[proxies]]
name = "ssh"
type = "tcp"
localIP = "127.0.0.1"
localPort = 22
remotePort = 6000
```

运行下述命令，安装 frpc

```bash
wget https://github.com/fatedier/frp/releases/download/v0.54.0/frp_0.54.0_linux_amd64.tar.gz
tar xzvf "frp_0.54.0_linux_amd64.tar.gz"
rm frp_0.54.0_linux_amd64.tar.gz*
sudo cp ./frp_0.54.0_linux_amd64/frpc /usr/bin
rm -rf "frp_0.54.0_linux_amd64"
sudo mkdir -p /etc/frp /var/frp
sudo cp ./frp/frpc.toml /etc/frp/frpc.toml
```

建立配置文件 `frpc.service`

```ini
[Unit]
Description=Frp Client Service
After=network.target
[Service]
Type=simple
DynamicUser=yes
Restart=on-failure
RestartSec=5s
ExecStart=/usr/bin/frpc -c /etc/frp/frpc.toml
ExecReload=/usr/bin/frpc reload -c /etc/frp/frpc.toml
LimitNOFILE=1048576
[Install]
WantedBy=multi-user.target
```

运行下述命令，安装并启动 frpc 服务

```bash
sudo cp ./frp/frpc.service /etc/systemd/system/frpc.service
sudo systemctl daemon-reload
sudo systemctl enable frpc
sudo systemctl start frpc
```

## 测试

随便找一台笔电，运行 VSCode ，点击左下角的 `><` 按钮，点击 SSH 连接，输入你的阿里云 ECS 服务 IP 和你在笔电 1 系统里的用户名密码， voilà

ps 个人强烈不建议使用明文用户名密码进行 SSH 登录，可以学习使用 `.ssh/authorized_keys` 公钥进行登录，这一步骤不属于本文讨论内容，可参考[^1]。

[^1]: https://www.runoob.com/w3cnote/set-ssh-login-key.html
