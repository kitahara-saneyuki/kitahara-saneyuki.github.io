---
layout:     single
title:      "指南：如何对Linux和WSL Shell进行配置，高效利用国际互联网资源进行开发"
date:       2023-12-16 03:37:14 +0800
categories: Terraform
published:  false
---

# 指南：如何对Linux和WSL Shell进行配置，高效利用国际互联网资源进行开发

**作者严正声明 Disclaimer：**

**作者本人遵守中华人民共和国法律法规，坚定拥护中国共产党的领导，坚决拥护党的路线、方针、政策。**

**撰写本文目的纯粹出于方便科研和工程人员更高效地完成科学探索和工程开发，为祖国的社会主义建设事业添砖加瓦。**

**作者坚决反对不法分子通过本文所叙述的技术手段从事非法活动，并保留通过一切法律手段进行追究的权利。**

配置环境：
- 笔电1：Kubuntu 23.10
- 笔电2：Windows 10 + WSL2 + Ubuntu 22.04

## 配置镜像站：apt、pip、docker、npm

### apt

我们使用这篇文章[^1]里介绍的方法，对`/etc/apt/sources.list`文件进行配置：

```bash
sudo mv /etc/apt/sources.list /etc/apt/sources.list.backup
sudo gedit /etc/apt/sources.list
```

建议对不同源进行测速，在笔者个人的网络环境下，发现中科大源最快。

```
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse

# 预发布软件源，不建议启用
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-proposed main restricted universe multiverse
```

对于Ubuntu 23.10，将`jammy`替换成`mantic`即可。

执行`sudo apt update`命令以应用改变。

### pip

创建`~/.pip`文件夹，在该文件夹下`pip.conf`文件，键入如下代码：

```conf
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```

### docker

### npm

## 在shell终端使用科学上网工具

我们在这里使用久不更新的Clash Verge[^2]。

在`~/.bashrc`末尾插入两行shell命令：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

## 一些题外话：配置Kubuntu 23.10工作机

系统环境：一台老掉牙的Thinkpad T480s

### VSCodium使用VSCode Marketplace



### 指纹识别登录系统


### QQ和输入法



[^1]: [Ubuntu 22.04 更换国内源 清华源 阿里源 中科大源 163源](https://www.linuxmi.com/ubuntu-22-04-apt-sources-list.html)
[^2]: [Index of /clients/clash-verge/releases/latest/](https://dl.jichangzhu.com/clients/clash-verge/releases/latest/)

