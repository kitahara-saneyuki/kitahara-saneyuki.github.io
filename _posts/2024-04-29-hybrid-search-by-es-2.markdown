---
layout:     single
title:      "实战：使用 ElasticSearch 8.13 实现混合搜索（2）：构建本地测试环境"
date:       2024-04-29 19:52:59 +0800
categories: NLP
published:  true
---

## 构建开发环境： ELK-B 

开发环境： AWS EC2 spot instance r7gd.large, ap-southeast-1, [Debian 12 (bookworm)](https://wiki.debian.org/Cloud/AmazonEC2Image/Bookworm)

（笔者现在的网络环境不是特别好，用流量下载 docker 镜像实在不太现实）

（ `r7gd.large` 有 16 GiB 内存，足够我们搞一搞原型开发。
它提供的 118 GiB NVMe SSD 会在断电后立刻擦除全部数据，可以暂缓对其利用）

我们使用 docker-compose 构建开发环境，根据 [Elastic 官网提供的教程](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose) ，初始化代码仓库。
为简化操作，我们直接 fork [ES 官方提供的代码仓库](https://github.com/elkninja/elastic-stack-docker-part-one/tree/main) 。
根据文中提示，我们需要改变 `.env` 文件中定义的默认 __密码__ （ `changeme` ）、 __端口__ ，请勿使用 `latest` 作为 __STACK_VERSION__ ，我们建议使用硬编码的版本号。
由于我们需要使用大量 ElasticSearch （下称 ES ）的最新功能，在本文中我们使用 `8.13.0` 版本。

键入 `docker compose up` 以开始我们的 ES 之旅。

### ES docker 镜像 debug

好事多磨。
一般来说第一次启动 ES 总是不会太成功。
debug 步骤如下：

1.  使用 `docker container ls -a` 命令寻找正在执行的 ES 镜像。
1.  使用 `docker logs <container_id> &> es01.log` 命令得到镜像日志，搜索 `error`
1.  谷歌搜索错误及其解决方案。

ES 官网提供了一个[常见虚拟内存限制错误的解决方案](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose)。
笔者遇到的错误是， 1GiB 内存不够 ES 使用，将 `.env` 文件中的 `ES_MEM_LIMIT` 调整为 4GiB 即可正确启动 ES 。

为了在本机访问部署在 AWS 上的 ES 集群，我们需要在 AWS EC2 控制台中暴露 9200 端口。
这实在老生常谈了，不再赘述。

## 确认安装成功

进入 VSCode 替我们自动转发的 5601 端口，即 Kibana 控制台，输入默认用户 `elastic` 和密码以确认我们安装成功。

根据[前文](https://kitahara-saneyuki.github.io/nlp/hybrid-search-by-es-1/)所述的 Cohere 混合检索实现，确认我们安装的最新 ES 版本支持混合检索。

### 实验计划

1.  [ ] 小样本测试
    1.  [x] 语义搜索：验证 Cohere 提供的 embedding 算法和 ElasticSearch 的 ANN 搜索。
    1.  [ ] 重排序：验证 Cohere 提供的 re-ranking 算法
1.  [ ] 大样本测试：
    1.  [x] 构建本地测试环境
    1.  [ ] 使用 ES 给定的 Ingest Pipeline，导入数据并 chunking 数据到合适规模。

## 下文预告

我们在本节中构建了 ES 本地测试环境以方便下一步的研究。
在下一节中我们将使用 ES 的 ingest pipeline 引入大规模数据，测试语义搜索功能。
