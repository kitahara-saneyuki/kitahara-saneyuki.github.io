---
layout:     single
title:      "指南：如何对Linux和WSL Shell进行配置，高效利用国际互联网资源进行开发"
date:       2023-12-16 03:37:14 +0800
categories: Terraform
published:  true
---

**作者严正声明 Disclaimer：**

**作者本人遵守中华人民共和国法律法规，坚定拥护中国共产党的领导，坚决拥护党的路线、方针、政策。**

**撰写本文目的纯粹出于方便科研和工程人员更高效地完成科学探索和工程开发，为祖国的社会主义建设事业添砖加瓦。**

**作者坚决反对不法分子通过本文所叙述的技术手段从事非法活动，并保留通过一切法律手段进行追究的权利。**

配置环境：
- 笔电1：Kubuntu 23.10
- 笔电2：Windows 10 + WSL2 + Ubuntu 22.04

## 配置镜像站：apt、pip、docker、npm、cargo

### apt

我们使用这篇文章[^1]里介绍的方法，对`/etc/apt/sources.list`文件进行配置：

```bash
sudo mv /etc/apt/sources.list /etc/apt/sources.list.backup
sudo gedit /etc/apt/sources.list
```

建议对不同源进行测速，在笔者个人的网络环境下，发现清华源最快。

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

创建文件`~/.pip/pip.conf`，键入如下代码：

```conf
[global]
index-url=https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=pypi.tuna.tsinghua.edu.cn
```

### cargo

新建文件`$HOME/.cargo/config`，内容如下[^5]：

```toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
# 指定镜像
replace-with = 'sjtu' # 如：tuna、sjtu、ustc，或者 rustcc

# 注：以下源配置一个即可，无需全部
# 目前 sjtu 相对稳定些

# 中国科学技术大学
[source.ustc]
registry = "https://mirrors.ustc.edu.cn/crates.io-index"

# 上海交通大学
[source.sjtu]
registry = "https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index/"

# 清华大学
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"

# rustcc社区
[source.rustcc]
registry = "https://code.aliyun.com/rustcc/crates.io-index.git"
```

### npm

在终端键入命令[^4]：

```bash
# 查询源
npm config get registry
# 更换国内源
npm config set registry https://registry.npm.taobao.org/
# 恢复官方源
npm config set registry https://registry.npmjs.org
# 删除注册表
npm config delete registry
```

### docker

#### Linux 环境下

创建文件`/etc/docker/daemon.json`，键入以下代码并保存：

（经过试验，在天津移动环境下，南京大学源较快）

```json
{
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com",
        "https://reg-mirror.qiniu.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://dockerhub.azk8s.cn",
        "https://docker.mirrors.sjtug.sjtu.edu.cn",
        "https://mirror.baidubce.com",
        "http://hub-mirror.c.163.com",
        "https://docker.nju.edu.cn"
    ]
}
```

键入以下命令以使之生效：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

#### WSL2 Ubuntu 环境下

双击任务栏中的 docker 图标，点击配置按钮（齿轮状），选择 docker engine ，将上述代码中的 `registry-mirrors` 项粘贴到配置 JSON 中

### conda

根据清华大学开源软件镜像站提供的帮助[^6]。在 `~` 目录下创建 `.condarc` 文件，修改其内容为

```
channels:
  - defaults
show_channel_urls: true
default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  msys2: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  bioconda: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  menpo: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch-lts: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  simpleitk: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  deepmodeling: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/
```

即可添加 Anaconda Python 免费仓库。

运行 `conda clean -i` 清除索引缓存，保证用的是镜像站提供的索引。

运行 `conda create -n myenv numpy` 测试一下吧。

## 在 shell 终端使用科学上网工具

### Linux 环境下

我们在这里使用久不更新的 Clash Verge[^2]。

在`~/.bashrc`末尾插入两行 shell 命令：

```bash
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

在 Clash Verge 客户端 GUI 中开启`局域网链接`（ Allow LAN ）和`系统代理`（ System Proxy ）即可。

### WSL2 Ubuntu 环境下

在`~/.bashrc`末尾插入两行 shell 命令：

```bash
export hostip=$(cat /etc/resolv.conf |grep -oP '(?<=nameserver\ ).*')
export https_proxy="http://${hostip}:7890"
export http_proxy="http://${hostip}:7890"
```

在Clash客户端GUI中开启`局域网链接`（Allow LAN）和`系统代理`（System Proxy）即可。

### VSCodium / VSCode 使用Clash

进入`文件 > 首选项 > 设置`，搜索`proxy`，键入`http://127.0.0.1:7890`

[^1]: [Ubuntu 22.04 更换国内源 清华源 阿里源 中科大源 163源](https://www.linuxmi.com/ubuntu-22-04-apt-sources-list.html)
[^2]: [Index of /clients/clash-verge/releases/latest/](https://dl.jichangzhu.com/clients/clash-verge/releases/latest/)
[^3]: [VSCodium: How to use a different extension gallery](https://github.com/VSCodium/vscodium/blob/master/docs/index.md#how-to-use-a-different-extension-gallery)
[^4]: [国内npm源镜像（npm加速下载） 指定npm镜像](https://blog.csdn.net/qq_43940789/article/details/131449710)
[^5]: [rustup、cargo设置为国内镜像](https://www.jianshu.com/p/17ca4c56fc5e)
[^6]: [清华大学开源软件镜像站：Anaconda 镜像使用帮助](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)
