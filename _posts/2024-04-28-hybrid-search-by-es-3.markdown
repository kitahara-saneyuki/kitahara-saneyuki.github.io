---
layout:     single
title:      "实战：使用 ElasticSearch 8.13 实现混合搜索（3）：构建长文拆分管线（ chunking ）"
date:       2024-04-29 19:52:59 +0800
categories: NLP
published:  true
---

## 长文拆分

与支持无限长文分析的 BM25 技术不同， Cohere 提供的 embedding 算法是不能处理超过 1024 个 token 的文字的。对于长度超过 128 个 token 的文本， Cohere 的算法将会切分其 embedding 结果向量，取其平均值[^1]。

根据 ElasticSearch 官网博客提供的[指南](https://www.elastic.co/search-labs/blog/chunking-via-ingest-pipelines)，我们对 Kibana 控制台输入以下命令。
请注意这里的向量维度被更改为了 1024 维，相似性算法也不再是指南中提供的向量点乘，而是余弦，以适应 Cohere 的 embedding 算法。

```json
PUT chunker
{
  "mappings": {
    "dynamic": "true",
    "properties": {
      "passages": {
        "type": "nested",
        "properties": {
          "vector": {
            "properties": {
              "predicted_value": {
                "type": "dense_vector",
                "index": true,
                "dims": 1024,
                "similarity": "cosine"
              }
            }
          }
        }
      }
    }
  }
}
```

我们继续，构建 Ingest 管线。
例子中提供的管线包括两个预处理步骤：
1.  使用正则表达式把 `body_content` 中的内容切分为一组句子，这里的正则表达式规避了不正确切分的做法（如在 Mr. 或 Mrs. 上切分）。
1.  尝试把句子块连缀起来，使其尽可能接近我们设定的长度上限。

预处理步骤之后，管线对每个句子执行 embedding 算法。
请注意我们每段文章有若干段文本块（ `chunk` ，在我们的索引中是 `passage.text` ），每个文本块都有其 embedding 之后的向量，即之前所定义的 `passage.vector.predicted_value`。

```json
PUT _ingest/pipeline/chunker
{
  "processors": [
    {
      "script": {
        "description": "Chunk body_content into sentences by looking for . followed by a space",
        "lang": "painless",
        "source": """
          String[] envSplit = /((?<!M(r|s|rs)\.)(?<=\.) |(?<=\!) |(?<=\?) )/.split(ctx['body_content']);
          ctx['passages'] = new ArrayList();
          int i = 0;
          boolean remaining = true;
          if (envSplit.length == 0) {
            return
          } else if (envSplit.length == 1) {
            Map passage = ['text': envSplit[0]];ctx['passages'].add(passage)
          } else {
            while (remaining) {
              Map passage = ['text': envSplit[i++]];
              while (i < envSplit.length && passage.text.length() + envSplit[i].length() < params.model_limit) {passage.text = passage.text + ' ' + envSplit[i++]}
              if (i == envSplit.length) {remaining = false}
              ctx['passages'].add(passage)
            }
          }
          """,
        "params": {
          "model_limit": 400
        }
      }
    },
    {
      "foreach": {
        "field": "passages",
        "processor": {
          "inference": {
            "model_id": "cohere_embeddings",
            "input_output": [
              { 
                "input_field": "_ingest._value.text",
                "output_field": "_ingest._value.vector.predicted_value"
              }
            ],
            "on_failure": [
              {
                "append": {
                  "field": "_source._ingest.inference_errors",
                  "value": [
                    {
                      "message": "Processor 'inference' in pipeline 'ml-inference-title-vector' failed with message '{{ _ingest.on_failure_message }}'",
                      "pipeline": "ml-inference-title-vector",
                      "timestamp": "{{{ _ingest.timestamp }}}"
                    }
                  ]
                }
              }
            ]
          }
        }
      }
    }
  ]
}
```

插入小样本实验数据：

```json
PUT chunker/_doc/1?pipeline=chunker
{
"title": "Adding passage vector search to Lucene",
"body_content": "Vector search is a powerful tool in the information retrieval tool box. Using vectors alongside lexical search like BM25 is quickly becoming commonplace. But there are still a few pain points within vector search that need to be addressed. A major one is text embedding models and handling larger text input. Where lexical search like BM25 is already designed for long documents, text embedding models are not. All embedding models have limitations on the number of tokens they can embed. So, for longer text input it must be chunked into passages shorter than the model’s limit. Now instead of having one document with all its metadata, you have multiple passages and embeddings. And if you want to preserve your metadata, it must be added to every new document. A way to address this is with Lucene's “join” functionality. This is an integral part of Elasticsearch’s nested field type. It makes it possible to have a top-level document with multiple nested documents, allowing you to search over nested documents and join back against their parent documents. This sounds perfect for multiple passages and vectors belonging to a single top-level document! This is all awesome! But, wait, Elasticsearch® doesn’t support vectors in nested fields. Why not, and what needs to change? The key issue is how Lucene can join back to the parent documents when searching child vector passages. Like with kNN pre-filtering versus post-filtering, when the joining occurs determines the result quality and quantity. If a user searches for the top four nearest parent documents (not passages) to a query vector, they usually expect four documents. But what if they are searching over child vector passages and all four of the nearest vectors are from the same parent document? This would end up returning just one parent document, which would be surprising. This same kind of issue occurs with post-filtering."
}

PUT chunker/_doc/3?pipeline=chunker
{
"title": "Automatic Byte Quantization in Lucene",
"body_content": "While HNSW is a powerful and flexible way to store and search vectors, it does require a significant amount of memory to run quickly. For example, querying 1MM float32 vectors of 768 dimensions requires roughly 1,000,000∗4∗(768+12)=3120000000≈31,000,000∗4∗(768+12)=3120000000bytes≈3GB of ram. Once you start searching a significant number of vectors, this gets expensive. One way to use around 75% less memory is through byte quantization. Lucene and consequently Elasticsearch has supported indexing byte vectors for some time, but building these vectors has been the user's responsibility. This is about to change, as we have introduced int8 scalar quantization in Lucene. All quantization techniques are considered lossy transformations of the raw data. Meaning some information is lost for the sake of space. For an in depth explanation of scalar quantization, see: Scalar Quantization 101. At a high level, scalar quantization is a lossy compression technique. Some simple math gives significant space savings with very little impact on recall. Those used to working with Elasticsearch may be familiar with these concepts already, but here is a quick overview of the distribution of documents for search. Each Elasticsearch index is composed of multiple shards. While each shard can only be assigned to a single node, multiple shards per index gives you compute parallelism across nodes. Each shard is composed as a single Lucene Index. A Lucene index consists of multiple read-only segments. During indexing, documents are buffered and periodically flushed into a read-only segment. When certain conditions are met, these segments can be merged in the background into a larger segment. All of this is configurable and has its own set of complexities. But, when we talk about segments and merging, we are talking about read-only Lucene segments and the automatic periodic merging of these segments. Here is a deeper dive into segment merging and design decisions."
}

PUT chunker/_doc/2?pipeline=chunker
{
"title": "Use a Japanese language NLP model in Elasticsearch to enable semantic searches",
"body_content": "Quickly finding necessary documents from among the large volume of internal documents and product information generated every day is an extremely important task in both work and daily life. However, if there is a high volume of documents to search through, it can be a time-consuming process even for computers to re-read all of the documents in real time and find the target file. That is what led to the appearance of Elasticsearch® and other search engine software. When a search engine is used, search index data is first created so that key search terms included in documents can be used to quickly find those documents. However, even if the user has a general idea of what type of information they are searching for, they may not be able to recall a suitable keyword or they may search for another expression that has the same meaning. Elasticsearch enables synonyms and similar terms to be defined to handle such situations, but in some cases it can be difficult to simply use a correspondence table to convert a search query into a more suitable one. To address this need, Elasticsearch 8.0 released the vector search feature, which searches by the semantic content of a phrase. Alongside that, we also have a blog series on how to use Elasticsearch to perform vector searches and other NLP tasks. However, up through the 8.8 release, it was not able to correctly analyze text in languages other than English. With the 8.9 release, Elastic added functionality for properly analyzing Japanese in text analysis processing. This functionality enables Elasticsearch to perform semantic searches like vector search on Japanese text, as well as natural language processing tasks such as sentiment analysis in Japanese. In this article, we will provide specific step-by-step instructions on how to use these features."
}

PUT chunker/_doc/5?pipeline=chunker
{
"title": "We can chunk whatever we want now basically to the limits of a document ingest",
"body_content": """Chonk is an internet slang term used to describe overweight cats that grew popular in the late summer of 2018 after a photoshopped chart of cat body-fat indexes renamed the "Chonk" scale grew popular on Twitter and Reddit. Additionally, "Oh Lawd He Comin'," the final level of the Chonk Chart, was adopted as an online catchphrase used to describe large objects, animals or people. It is not to be confused with the Saturday Night Live sketch of the same name. The term "Chonk" was popularized in a photoshopped edit of a chart illustrating cat body-fat indexes and the risk of health problems for each class (original chart shown below). The first known post of the "Chonk" photoshop, which classifies each cat to a certain level of "chonk"-ness ranging from "A fine boi" to "OH LAWD HE COMIN," was posted to Facebook group THIS CAT IS C H O N K Y on August 2nd, 2018 by Emilie Chang (shown below). The chart surged in popularity after it was tweeted by @dreamlandtea[1] on August 10th, 2018, gaining over 37,000 retweets and 94,000 likes (shown below). After the chart was posted there, it began growing popular on Reddit. It was reposted to /r/Delighfullychubby[2] on August 13th, 2018, and /r/fatcats on August 16th.[3] Additionally, cats were shared with variations on the phrase "Chonk." In @dreamlandtea's Twitter thread, she rated several cats on the Chonk scale (example, shown below, left). On /r/tumblr, a screenshot of a post featuring a "good luck cat" titled "Lucky Chonk" gained over 27,000 points (shown below, right). The popularity of the phrase led to the creation of a subreddit, /r/chonkers,[4] that gained nearly 400 subscribers in less than a month. Some photoshops of the chonk chart also spread on Reddit. For example, an edit showing various versions of Pikachu on the chart posted to /r/me_irl gained over 1,200 points (shown below, left). The chart gained further popularity when it was posted to /r/pics[5] September 29th, 2018."""
}
```

如果对 embedding 之后的结果不太放心，我们可以手动观察一下：

```json
GET chunker/_search?size=5
{
    "query": {
        "match_all": {}
    }
}
```

### 搜索测试

我们先用官方文档中测试一下，当然 kNN 的搜索 k 不一定一定是 1 ，我们改成 2 。

```json
GET chunker/_search
{
  "_source": false,
  "fields": [
    "title"
  ],
  "knn": {
    "inner_hits": {
      "_source": false,
      "fields": [
        "passages.text"
      ]
    },
    "field": "passages.vector.predicted_value",
    "k": 2,
    "num_candidates": 100,
    "query_vector_builder": {
      "text_embedding": {
        "model_id": "cohere_embeddings",
        "model_text": "Can I use multiple vectors per document now?"
      }
    }
  }
}
```

搜索结果：

```json
{
  "took": 285,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 0.78212404,
    "hits": [
      {
        "_index": "chunker",
        "_id": "1",
        "_score": 0.78212404,
        "_ignored": [
          "body_content.keyword",
          "passages.text.keyword"
        ],
        "fields": {
          "title": [
            "Adding passage vector search to Lucene"
          ]
        },
        "inner_hits": {
          "passages": {
            "hits": {
              "total": {
                "value": 6,
                "relation": "eq"
              },
              "max_score": 0.78212404,
              "hits": [
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 3
                  },
                  "_score": 0.78212404,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "This sounds perfect for multiple passages and vectors belonging to a single top-level document! This is all awesome! But, wait, Elasticsearch® doesn’t support vectors in nested fields. Why not, and what needs to change? The key issue is how Lucene can join back to the parent documents when searching child vector passages."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 0
                  },
                  "_score": 0.7376485,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Vector search is a powerful tool in the information retrieval tool box. Using vectors alongside lexical search like BM25 is quickly becoming commonplace. But there are still a few pain points within vector search that need to be addressed. A major one is text embedding models and handling larger text input."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 4
                  },
                  "_score": 0.7086177,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Like with kNN pre-filtering versus post-filtering, when the joining occurs determines the result quality and quantity. If a user searches for the top four nearest parent documents (not passages) to a query vector, they usually expect four documents. But what if they are searching over child vector passages and all four of the nearest vectors are from the same parent document?"
                        ]
                      }
                    ]
                  }
                }
              ]
            }
          }
        }
      },
      {
        "_index": "chunker",
        "_id": "2",
        "_score": 0.704564,
        "_ignored": [
          "body_content.keyword",
          "passages.text.keyword"
        ],
        "fields": {
          "title": [
            "Use a Japanese language NLP model in Elasticsearch to enable semantic searches"
          ]
        },
        "inner_hits": {
          "passages": {
            "hits": {
              "total": {
                "value": 6,
                "relation": "eq"
              },
              "max_score": 0.704564,
              "hits": [
                {
                  "_index": "chunker",
                  "_id": "2",
                  "_nested": {
                    "field": "passages",
                    "offset": 3
                  },
                  "_score": 0.704564,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Elasticsearch enables synonyms and similar terms to be defined to handle such situations, but in some cases it can be difficult to simply use a correspondence table to convert a search query into a more suitable one. To address this need, Elasticsearch 8.0 released the vector search feature, which searches by the semantic content of a phrase."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "2",
                  "_nested": {
                    "field": "passages",
                    "offset": 4
                  },
                  "_score": 0.6868271,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Alongside that, we also have a blog series on how to use Elasticsearch to perform vector searches and other NLP tasks. However, up through the 8.8 release, it was not able to correctly analyze text in languages other than English. With the 8.9 release, Elastic added functionality for properly analyzing Japanese in text analysis processing."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "2",
                  "_nested": {
                    "field": "passages",
                    "offset": 0
                  },
                  "_score": 0.6548239,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Quickly finding necessary documents from among the large volume of internal documents and product information generated every day is an extremely important task in both work and daily life. However, if there is a high volume of documents to search through, it can be a time-consuming process even for computers to re-read all of the documents in real time and find the target file."
                        ]
                      }
                    ]
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```

可见文档 1 的相关性有 0.78 ，的确远比相关性只有 0.7 的文档 2 更贴近我们的问题。

我们换一个搜索关键字， `scalar quantization` ，搜索结果如下：

```json
{
  "took": 424,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 2,
      "relation": "eq"
    },
    "max_score": 0.8421624,
    "hits": [
      {
        "_index": "chunker",
        "_id": "3",
        "_score": 0.8421624,
        "_ignored": [
          "body_content.keyword",
          "passages.text.keyword"
        ],
        "fields": {
          "title": [
            "Automatic Byte Quantization in Lucene"
          ]
        },
        "inner_hits": {
          "passages": {
            "hits": {
              "total": {
                "value": 6,
                "relation": "eq"
              },
              "max_score": 0.8421624,
              "hits": [
                {
                  "_index": "chunker",
                  "_id": "3",
                  "_nested": {
                    "field": "passages",
                    "offset": 2
                  },
                  "_score": 0.8421624,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Meaning some information is lost for the sake of space. For an in depth explanation of scalar quantization, see: Scalar Quantization 101. At a high level, scalar quantization is a lossy compression technique. Some simple math gives significant space savings with very little impact on recall."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "3",
                  "_nested": {
                    "field": "passages",
                    "offset": 1
                  },
                  "_score": 0.7212088,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "One way to use around 75% less memory is through byte quantization. Lucene and consequently Elasticsearch has supported indexing byte vectors for some time, but building these vectors has been the user's responsibility. This is about to change, as we have introduced int8 scalar quantization in Lucene. All quantization techniques are considered lossy transformations of the raw data."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "3",
                  "_nested": {
                    "field": "passages",
                    "offset": 0
                  },
                  "_score": 0.62174904,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "While HNSW is a powerful and flexible way to store and search vectors, it does require a significant amount of memory to run quickly. For example, querying 1MM float32 vectors of 768 dimensions requires roughly 1,000,000∗4∗(768+12)=3120000000≈31,000,000∗4∗(768+12)=3120000000bytes≈3GB of ram. Once you start searching a significant number of vectors, this gets expensive."
                        ]
                      }
                    ]
                  }
                }
              ]
            }
          }
        }
      },
      {
        "_index": "chunker",
        "_id": "1",
        "_score": 0.60923594,
        "_ignored": [
          "body_content.keyword",
          "passages.text.keyword"
        ],
        "fields": {
          "title": [
            "Adding passage vector search to Lucene"
          ]
        },
        "inner_hits": {
          "passages": {
            "hits": {
              "total": {
                "value": 6,
                "relation": "eq"
              },
              "max_score": 0.60923594,
              "hits": [
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 3
                  },
                  "_score": 0.60923594,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "This sounds perfect for multiple passages and vectors belonging to a single top-level document! This is all awesome! But, wait, Elasticsearch® doesn’t support vectors in nested fields. Why not, and what needs to change? The key issue is how Lucene can join back to the parent documents when searching child vector passages."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 0
                  },
                  "_score": 0.59735155,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "Vector search is a powerful tool in the information retrieval tool box. Using vectors alongside lexical search like BM25 is quickly becoming commonplace. But there are still a few pain points within vector search that need to be addressed. A major one is text embedding models and handling larger text input."
                        ]
                      }
                    ]
                  }
                },
                {
                  "_index": "chunker",
                  "_id": "1",
                  "_nested": {
                    "field": "passages",
                    "offset": 2
                  },
                  "_score": 0.59269404,
                  "fields": {
                    "passages": [
                      {
                        "text": [
                          "And if you want to preserve your metadata, it must be added to every new document. A way to address this is with Lucene's “join” functionality. This is an integral part of Elasticsearch’s nested field type. It makes it possible to have a top-level document with multiple nested documents, allowing you to search over nested documents and join back against their parent documents."
                        ]
                      }
                    ]
                  }
                }
              ]
            }
          }
        }
      }
    ]
  }
}
```

这次则是 0.84 对比 0.6 ，相关性差别十分显著。

## 下文预告

我们在本节中使用了 ES 的 ingest pipeline 引入长文数据，测试了针对长文的语义搜索功能。
工程实现效果是令人满意的。

下文中我们将导入真实世界测试数据，即 Kaggle 上的 [Six TripAdvisor Datasets for NLP Tasks](https://www.kaggle.com/datasets/inigolopezrioboo/a-tripadvisor-dataset-for-nlp-tasks)[^2] 中的纽约餐厅评论，以验证 ElasticSearch 的 kNN 搜索算法针对大规模数据集的威力。
测试用例暂定为：

> 找寻纽约最好的意大利面饭馆

### 实验计划

1.  [ ] 小样本测试
    1.  [x] 语义搜索：验证 Cohere 提供的 embedding 算法和 ElasticSearch 的 ANN 搜索。
    1.  [x] 建立 ES 的导入数据 Ingest 管线， chunking 长文到合适规模。
    1.  [ ] 重排序：验证 Cohere 提供的 re-ranking 算法
1.  [ ] 大样本测试：
    1.  [x] 构建本地测试环境
    1.  [ ] 导入真实世界测试数据

本节实现了 1.2。

[^1]: [Cohere launches larger Embed models](https://cohere.com/blog/cohere-launches-larger-embed-models-2)
[^2]: [A TripAdvisor Dataset for Dyadic Context Analysis](https://zenodo.org/records/6583422)
