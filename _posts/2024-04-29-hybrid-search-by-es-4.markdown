---
layout:     single
title:      "实战：使用 ElasticSearch 8.13 实现混合搜索（4）：使用开源模型，导入真实世界数据"
date:       2024-04-29 21:19:51 +0800
categories: NLP
published:  true
---

从本文开始我们终于要开始使用 Python 了，数据量实在超越了 Kibana 控制台能处理的规模。
文末会附带 Jupyter Notebook 示例代码，以保证实验的可复现性。

## 使用开源模型

Cohere 不是免费的，如果把我们的 50 万条数据都丢给 Cohere embedding 算法处理的话，恐怕要产生一笔我们并不想要的信用卡债。
这一步骤事实上已经在 ElasticSearch 官网博客提供的[指南](https://www.elastic.co/search-labs/blog/chunking-via-ingest-pipelines) 及其[示例代码 Jupyter Notebook](https://github.com/elastic/elasticsearch-labs/blob/main/notebooks/document-chunking/with-index-pipelines.ipynb) 中提及。
测试中发现的易出 bug 点在于 `eland[pytorch]` 库需要 Python 3.10 才能较为顺利的安装。

这里还有一个问题，我们部署本地开源模型的话，需要使用 GPU 加速的云服务，这笔信用卡债似乎很难躲开。
但如果我们的数据规模扩张到难以控制的程度，这笔实践经验就会显得十分宝贵。
我们使用 AWS 最便宜的 [g4dn.xlarge](https://aws.amazon.com/ec2/instance-types/g4/) 实例运行我们的 PyTorch 服务。
虽然他提供的 [nVidia T4 GPU](https://www.techpowerup.com/gpu-specs/tesla-t4.c3316) 已经是五年前发布的，但是 T4 GPU 似乎为半精度浮点计算（ FP16 ）做了特殊的优化，其半精度浮点计算能力高达 65.13 TFLOPS ，十分符合我们的需求。

当然，云服务价格比 `r7gd.large` 贵了四倍。
而且需要从 AWS 申请限额，可能要等一天才能完成。
我们先在本机上玩，玩透了大概 AWS 也批准了我们的限额请求了。

笔者笔电的 RTX 2070 Max-Q 在大概 4 秒钟之内完成了 Cohere embedding 需要一分钟， CPU 十分钟不一定跑的完的数据，赢麻了。

另外还有一个容易卡住的点，可以参考[这一节](https://kitahara-saneyuki.github.io/terraform/guide-of-working-in-china-mainland/#conda)。

## 使用 Celery / RabbitMQ 管理大规模数据操作




### 实验计划

1.  [ ] 小样本测试
    1.  [x] 语义搜索：验证 Cohere 提供的 embedding 算法和 ElasticSearch 的 ANN 搜索。
    1.  [x] 建立 ES 的导入数据 Ingest Pipeline， chunking 长文到合适规模。
    1.  [ ] 重排序：验证 Cohere 提供的 re-ranking 算法
1.  [x] 大样本测试：
    1.  [x] 构建本地测试环境
    1.  [x] 导入实际测试数据

本节完成了 2.2 。

## 下文预告

我们在本节中使用 ES 的 ingest pipeline 引入大规模数据，测试语义搜索功能。

下一节将测试 Cohere 提供的 re-ranking 算法，在前 100 条

> 纽约最好的意大利面饭馆

中，重新排序出第一二三四名来。
