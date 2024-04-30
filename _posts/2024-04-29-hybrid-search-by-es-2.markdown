---
layout:     single
title:      "实战：使用 ElasticSearch 8.13 实现混合搜索（2）：构建本地测试环境"
date:       2024-04-29 19:52:59 +0800
categories: NLP
published:  false
---

## 构建开发环境： ELK-B 

开发环境： Windows 10, WSL 2, Ubuntu 22.04 LTS, Docker Desktop ，老生常谈了

我们使用 docker-compose 构建开发环境，根据 [Elastic 官网提供的教程](https://www.elastic.co/blog/getting-started-with-the-elastic-stack-and-docker-compose) ，初始化代码仓库。
为简化操作，我们直接 fork [ES 官方提供的代码仓库](https://github.com/elkninja/elastic-stack-docker-part-one/tree/main) 。
根据文中提示，我们需要改变 __用户名/密码__ 、 __端口__ ，请勿使用 `latest` 作为 __STACK_VERSION__ ，我们建议使用硬编码的版本号。
由于我们需要使用大量 ElasticSearch （下称 ES ）的最新功能，在本文中我们使用 `8.13.0` 版本。


