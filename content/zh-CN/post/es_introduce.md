+++
author = "花荣"
title = "ES使用入门"
date = "2022-02-02"
description = "es使用入门"
tags = [
    "es",
]
+++

ES使用入门 ES使用教程
<!--more-->

# elastic_search 使用入门

![0.png](https://s2.loli.net/2022/02/03/zBpjeNYJdHKULqT.png)

# 基本概念

- 索引(index)

一个 索引 类似于传统关系数据库中的一个 表 ，是一个存储关系型文档的地方

- 文档类型(type) 【7.0版本后移除】 👉 [原文](https://www.elastic.co/guide/en/elasticsearch/reference/7.13/removal-of-types.html#_why_are_mapping_types_being_removed)
- 文档(doc)

一个doc代表索引里的一条数据，像数据库表里的一条记录，doc是用json格式来存储数据

# es架构设计

![1.png](https://s2.loli.net/2022/02/03/TkR7DACh2g8BjHE.png)

架构各组件简单释义:

- gateway 底层存储系统，一般为文件系统，支持多种类型。
- distributed lucence directory 基于lucence的分布式框架，封装了建立倒排索引、数据存储、translog、segment等实现。
- 模块层 ES的主要模块，包含索引模块、搜索模块、映射模块。
- Discovery 集群node发现模块，用于集群node之间的通信，选举coordinate node操作，支持多种发现机制，如zen，ec2等。
- script 脚本解析模块，用来支持在查询语句中编写的脚本，如painless，groovy，python等。
- plugins 第三方插件，各种高级功能可由插件提供，支持定制。
- transport/jmx 通信模块，数据传输，底层使用netty框架
- restful/node 对外提供的访问Elasticsearch集群的接口
- x-pack elasticsearch的一个扩展包，集成安全、警告、监视、图形和报告功能，无缝接入，可插拔设计。

作者：1黄鹰

链接：https://juejin.cn/post/6844903994054148110

来源：掘金

![2.png](https://s2.loli.net/2022/02/03/76T91nXByNUhRHf.png)
来源：架构师修炼 [https://cloud.tencent.com/developer/article/1663950](https://cloud.tencent.com/developer/article/1663950)

# 创建、删除、变更操作

## [创建索引](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/docs-index_.html)

```tsx
PUT /<index>
PUT /ks_advertiser_report_index_1
{
    "settings": {
        "number_of_shards": 3,
        "number_of_replicas": 1
    },
    "mappings": {
        "properties": {
            "action_cost": {
                "type": "double"
            },
            "activation": {
                "type": "long"
            },
            "advertiser_name": {
                "type": "text",
                "fields":{
                    "keyword":{
                        "type":"keyword"
                    }
                }
            },
            "cover_url": {
                "type": "keyword"
            },
            "created_at": {
                "type": "date",
                "format": "yyyy-MM-dd HH:mm:ss || yyyy-MM-dd HH:mm:ss.SSS ||yyyy-MM-dd || epoch_millis || strict_date_optional_time || yyyy-MM-dd'T'HH:mm:ss'+'08:00"
            }
        }
    }
}
```

## 创建doc

```tsx
#指定id
PUT /<index>/_doc/<_id>
PUT /ks_advertiser_report_index_1/_doc/1
{
  "cover_url":"test2",
	"activation":100
}

#不指定id
POST /<index>/_doc
POST /ks_advertiser_report_index_1/_doc/
{
  "cover_url":"test2",
  "activation":200
}

#创建时如果已经存在doc, 报异常
PUT /ks_advertiser_report_index_1/_doc/10/_create
{
  "cover_url": "test10",
  "activation": 1000
}
```

## [批量创建](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/docs-bulk.html)和操作

```json
POST _bulk
{"index":{"_index":"ks_advertiser_report_index_1","_id":"1"}}
{"cover_url":"test1","activation":100, "advertiser_name":"张三, 今天星期一"}
{"index":{"_index":"ks_advertiser_report_index_1","_id":"2"}}
{"cover_url":"test2","activation":200,"advertiser_name":"李四，今天星期二"}
{"index":{"_index":"ks_advertiser_report_index_1","_id":"3"}}
{"cover_url":"test3","activation":300,"advertiser_name":"王五 今天星期三"}
{"index":{"_index":"ks_advertiser_report_index_1","_id":"4"}}
{"cover_url":"test4","activation":400, "advertiser_name":"马六 今天星期四"}
{"create":{"_index":"ks_advertiser_report_index_1","_id":"5"}}
{"cover_url":"test5","activation":500, "advertiser_name": "广告主"}
{"delete":{"_index":"ks_advertiser_report_index_1","_id":"5"}}
{"update":{"_id":"1","_index":"ks_advertiser_report_index_1"}}
{"doc":{"cover_url":"test11"}}
```

## 更新index字段

es不支持对已存在字段做类型变更，如需要变更，需要reindex

```tsx
PUT /<index>/_mapping
PUT /ks_advertiser_report_index_1/_mapping
{
    "properties": {
        "creative_create_time": {
            "type": "date",
            "format": "yyyy-MM-dd HH:mm:ss"
        }
    }
}
```

## 更新doc

```tsx
#按id更新
PUT /<index>/_doc/<_id>
PUT /ks_advertiser_report_index_1/_doc/227
{
	"advertiser_name" : "嗨寿司-李"
}

#按条件更新
POST ks_advertiser_report_index_1/_update_by_query
{
  "query": {
    "term": {
      "advertiser_name": "嗨寿司-李"
    }
  } ,
  "script": {
    "source": "ctx._source['advertiser_name'] = '嗨寿司'"
  }
}
```

## 删除索引

```tsx
DELETE /<index>
DELETE /ks_advertiser_report_index_1
```

## 删除doc

```tsx
DELETE /<index>/_doc/<_id>
DELETE /ks_advertiser_report_index_1/_doc/137
```

## 清空doc

```json
POST /ks_advertiser_report_index_1/_delete_by_query
{
    "query": {
       "match_all": {}
     }
}
```

# 常用的查询类型

## 一般查询

- 查询index全部数据

```json
GET /ks_advertiser_report_index_1/_search
{   
  "query": {
    "match_all": {}
  }
}
```

- 指定id查询

```tsx
GET /ks_advertiser_report_index_1/_doc/1
```

- 模糊查询

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
    "wildcard": {
      "cover_url": {
        "value": "*4*"
      }
    }
  }
}
```

- 存在字段(exists)查询

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
    "exists": {
      "field": "cover_url"
    }
  }
}
```

- 范围查询

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query":{
    "range": {
      "activation": {
        "gt": 1,
        "lte": 200
      }
    }
  }
}
```

- 精准查询

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query":{
    "term": {
      "_id": {
        "value": "1"
      }
    }
  }
}
```

- 正则查询

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
   "regexp": { 
       "advertiser_name":"张*"
    } 
  }
}
```

- 复合查询（bool查询）

**bool query 可以用来合并多个过滤条件查询结果，它包含这如下几个操作符:**

**must** : 查询必须出现在匹配的文档中

**filter:**  对于must，查询的分数将被忽略。缓存查询

**must_not** : 查询不得出现在匹配的文档中

**should** : 查询应该出现在匹配的文档中。(不强制匹配，如果匹配到，score更大，如果不存在must，至少一个should查询匹配)

<aside>
💡 所有 `must` 语句必须匹配，所有 `must_not` 语句都必须不匹配，但有多少 `should` 语句应该匹配呢？默认情况下，没有 `should` 语句是必须匹配的，只有一个例外：那就是当没有 `must` 语句的时候，至少有一个 `should` 语句必须匹配。就像我们能控制 `[match` 查询的精度](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-multi-word.html#match-precision) 一样，我们可以通过 `minimum_should_match` 参数控制需要匹配的 `should` 语句的数量，它既可以是一个绝对的数字，又可以是个百分比 .

</aside>

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "terms": {
            "_id": [
              "1",
              "2"
            ]
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "cover_url": {
              "value": "test2"
            }
          }
        }
      ],
      "should": [
        {
          "term": {
            "cover_url": {
              "value": "test1"
            }
          }
        }
      ]
    }
  }
}
```

## 文本查询

目前线上es文本分词用的是IK分词器

[官方提供的分词器：](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-analyzers.html)

[Fingerprint、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-fingerprint-analyzer.html)[Keyword、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-keyword-analyzer.html)[Language、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-lang-analyzer.html)[Pattern、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-pattern-analyzer.html)[Simple、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-simple-analyzer.html)**[Standard、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-standard-analyzer.html)**[Stop、](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-stop-analyzer.html)[Whitespace](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/analysis-whitespace-analyzer.html)

IK分词(中文)：

IK analyzer:  ik_smart, ik_max_word

- [math](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/query-dsl-match-query.html)

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
    "match": {
      "advertiser_name": {
        "query": "马六 今天",
        "analyzer": "standard"
      }
    }
  }
}
```

- math_phrase

```tsx
GET /ks_advertiser_report_index_1/_search
{
  "query": {
    "match_phrase": {
      "advertiser_name": {
        "query": "马六 今天",
        "analyzer": "standard"
      }
    }
  }
}
```

- 查看分析效果

```tsx
POST /_analyze?pretty=true
{
  "text":"张三, 今天星期一",
  "tokenizer":"ik_max_word"
}
```

## 聚合查询(Aggregation)

- **bucket aggregation**

<aside>
💡 相当于db的group by

</aside>

**分捅聚合**

```json
GET /ks_advertiser_report_index_1/_search
{
  "query":{
    "range":{
      "activation":{
        "gte":300
      }
    }
  },
  "size": 0,
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "advertiser_name.keyword"
      }
    }
  }
}
```

- **metric aggregation**

<aside>
💡 相当于db的聚合函数；max、avg

</aside>

```json
GET /ks_advertiser_report_index_1/_search
{
  "query":{
    "range":{
      "activation":{
        "gte":300
      }
    }
  },
  "size": 0,
  "aggs": {
    "my-agg-name-avg":{
      "avg": {
        "field": "activation"
      }
    }
  }
}
```

- **bucket 和 metric 组合查询**

```json
GET /ks_advertiser_report_index_1/_search?pretty
{
  "size":0,
  "aggs": {
    "my-agg-name": {
      "terms": {
        "field": "advertiser_name.keyword"
      },
      "aggs": {
        "my-sub-agg-name": {
          "avg": {
            "field": "activation"
          }
        }
      }
    }
  }
}
```

- **pipe line aggregation**

管道聚合处理的对象是其它聚合的输出（桶或者桶的某些权值），而不是直接针对文档。

管道聚合的作用是为输出增加一些有用信息。

buckets_path：用于计算均值的权值路径

```json
{
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "avg_monthly_sales": {
            "avg_bucket": {             //对所有月份的销售总 sales 求平均值
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
```

## 集群基本信息查询

- 查看集群基本信息

```tsx
GET /
```

- 查看集群节点信息

```tsx
GET /_cluster/health?pretty
```

- 查看集群所有index

```tsx
GET /_cat/indices
```

- 查看index mapping信息

```tsx
GET /ks_advertiser_report_index_1/_mapping
```

# 例子

## 快手投放

```json
POST /ks_unit_report_index/_search
{
    "aggregations":{
        "count":{
            "cardinality":{
                "field":"unit_id"
            }
        },
        "group":{
            "aggregations":{
                "bucket_sort":{
                    "bucket_sort":{
                        "size":20
                    }
                },
                "lately_top_hit":{
                    "top_hits":{
                        "size":1,
                        "sort":[
                            {
                                "unit_create_time":{
                                    "order":"desc"
                                }
                            }
                        ]
                    }
                },
                "max_create_time":{
                    "max":{
                        "field":"unit_create_time"
                    }
                },
                "sum_aclick":{
                    "sum":{
                        "field":"aclick"
                    }
                },
                "sum_bclick":{
                    "sum":{
                        "field":"bclick"
                    }
                },
                "sum_charge":{
                    "sum":{
                        "field":"charge"
                    }
                },
                "sum_photo_click":{
                    "sum":{
                        "field":"photo_click"
                    }
                },
                "sum_show":{
                    "sum":{
                        "field":"show"
                    }
                }
            },
            "terms":{
                "field":"unit_id",
                "order":[
                    {
                        "max_create_time":"desc"
                    }
                ],
                "size":20
            }
        },
        "sum_aclick":{
            "sum":{
                "field":"aclick"
            }
        },
        "sum_bclick":{
            "sum":{
                "field":"bclick"
            }
        },
        "sum_charge":{
            "sum":{
                "field":"charge"
            }
        },
        "sum_photo_click":{
            "sum":{
                "field":"photo_click"
            }
        },
        "sum_show":{
            "sum":{
                "field":"show"
            }
        }
    },
    "from":0,
    "query":{
        "bool":{
            "filter":[
                {
                    "range":{
                        "stat_date":{
                            "format":"yyyy-MM-dd",
                            "from":"2021-06-08",
                            "include_lower":true,
                            "include_upper":true,
                            "to":"2021-06-08"
                        }
                    }
                },
                {
                    "term":{
                        "campaign_id":62935142
                    }
                }
            ]
        }
    },
    "size":0
}
```

```json
{
    "took":47,
    "timed_out":false,
    "_shards":{
        "total":5,
        "successful":5,
        "skipped":0,
        "failed":0
    },
    "hits":{
        "total":{
            "value":1,
            "relation":"eq"
        },
        "max_score":null,
        "hits":[

        ]
    },
    "aggregations":{
        "sum_show":{
            "value":"1.0"
        },
        "sum_charge":{
            "value":825.295
        },
        "sum_photo_click":{
            "value":"0.0"
        },
        "count":{
            "value":1
        },
        "sum_aclick":{
            "value":"230261.0"
        },
        "sum_bclick":{
            "value":"6419.0"
        },
        "group":{
            "doc_count_error_upper_bound":0,
            "sum_other_doc_count":0,
            "buckets":[
                {
                    "key":245146720,
                    "doc_count":1,
                    "sum_show":{
                        "value":"1.0"
                    },
                    "sum_charge":{
                        "value":825.295
                    },
                    "max_create_time":{
                        "value":"1.621912523E12",
                        "value_as_string":"2021-05-25 03:15:23"
                    },
                    "sum_photo_click":{
                        "value":"0.0"
                    },
                    "lately_top_hit":{
                        "hits":{
                            "total":{
                                "value":1,
                                "relation":"eq"
                            },
                            "max_score":null,
                            "hits":[
                                {
                                    "_index":"ks_unit_report_index",
                                    "_type":"_doc",
                                    "_id":"214",
                                    "_score":null,
                                    "_source":{
                                        "advertiser_id":10175341,
                                        "advertiser_name":"DraGon",
                                        "created_at":"2021-06-11T14:08:46.275+08:00",
                                        "stat_year":2021,
                                        "stat_week":2684,
                                        "stat_month":"2021-06-01",
                                        "stat_date":"2021-06-08",
                                        "stat_hour":0,
                                        "charge":825.295,
                                        "show":1,
                                        "photo_click":0,
                                        "aclick":230261,
                                        "bclick":6419,
                                        "photo_click_ratio":"0.0",
                                        "play_3_s_ratio":"0.0",
                                        "action_ratio":0.027877061247888267,
                                        "impression_1_k_cost":"825295.0",
                                        "photo_click_cost":"0.0",
                                        "action_cost":0.12857064963389936,
                                        "share":60
                                    },
                                    "sort":[
                                        1621912523000
                                    ]
                                }
                            ]
                        }
                    },
                    "sum_aclick":{
                        "value":"230261.0"
                    },
                    "sum_bclick":{
                        "value":"6419.0"
                    }
                }
            ]
        }
    }
}
```

## 快手报表

```json
POST /ks_creative_report_index/_search
{
    "aggregations":{
        "count":{
            "cardinality":{
                "script":{
                    "source":"doc['stat_month'].value +'####'+ doc['creative_id'].value"
                }
            }
        },
        "group":{
            "aggregations":{
                "bucket_sort":{
                    "bucket_sort":{
                        "size":20
                    }
                },
                "max_stat_time":{
                    "max":{
                        "field":"stat_month"
                    }
                },
                "sumAClick":{
                    "sum":{
                        "field":"aclick"
                    }
                },
                "sumActionCost":{
                    "sum":{
                        "field":"action_cost"
                    }
                },
                "sumActionRatio":{
                    "sum":{
                        "field":"action_ratio"
                    }
                },
                "sumBClick":{
                    "sum":{
                        "field":"bclick"
                    }
                },
                "sumCharge":{
                    "sum":{
                        "field":"charge"
                    }
                },
                "sumPhotoClick":{
                    "sum":{
                        "field":"photo_click"
                    }
                },
                "sumPhotoClickCost":{
                    "sum":{
                        "field":"photo_click_cost"
                    }
                },
                "sumPhotoClickRatio":{
                    "sum":{
                        "field":"photo_click_ratio"
                    }
                },
                "sumShow":{
                    "sum":{
                        "field":"show"
                    }
                },
                "top_hits_fields":{
                    "top_hits":{
                        "size":1,
                        "sort":[
                            {
                                "created_at":{
                                    "order":"desc"
                                }
                            }
                        ]
                    }
                }
            },
            "terms":{
                "order":[
                    {
                        "max_stat_time":"desc"
                    }
                ],
                "script":{
                    "source":"doc['stat_month'].value +'####'+ doc['creative_id'].value"
                },
                "size":20
            }
        }
    },
    "from":0,
    "query":{
        "bool":{
            "must":[
                {
                    "terms":{
                        "advertiser_id":[
                            10175341
                        ]
                    }
                },
                {
                    "range":{
                        "stat_date":{
                            "format":"yyyy-MM-dd",
                            "from":"2021-03-21",
                            "include_lower":true,
                            "include_upper":true,
                            "to":"2021-06-26"
                        }
                    }
                }
            ]
        }
    },
    "size":0
}
```

```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 14,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "count" : {
      "value" : 4
    },
    "group" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "2021-06-01T00:00:00.000Z####4535424809",
          "doc_count" : 2,
          "sumBClick" : {
            "value" : 2121.0
          },
          "top_hits_fields" : {
            "hits" : {
              "total" : {
                "value" : 2,
                "relation" : "eq"
              },
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "ks_creative_report_index",
                  "_type" : "_doc",
                  "_id" : "252",
                  "_score" : null,
                  "_source" : {
                    "advertiser_id" : 10175341,
                    "advertiser_name" : "DraGon",
                    "created_at" : "2021-06-11T14:08:48.772+08:00",
                    "stat_year" : 2021,
                    "stat_week" : 2684,
                    "stat_month" : "2021-06-01",
                    "stat_date" : "2021-06-09",
                    "stat_hour" : 0,
                    "charge" : 0.0,
                    "show" : 0,
                    "photo_click" : 0,
                    "aclick" : 2,
                    "bclick" : 0,
                    "photo_click_ratio" : 0.0,
                    "play_3_s_ratio" : 0.0,
                    "action_ratio" : 0.0,
                    "impression_1_k_cost" : 0.0,
                    "photo_click_cost" : 0.0,
                    "action_cost" : 0.0,
                    "share" : 0,
                    "comment" : 0
                  },
                  "sort" : [
                    1623391728772
                  ]
                }
              ]
            }
          },
          "sumShow" : {
            "value" : 0.0
          },
          "sumCharge" : {
            "value" : 174.705
          },
          "sumPhotoClickCost" : {
            "value" : 0.0
          },
          "sumActionCost" : {
            "value" : 0.08236916548797737
          },
          "sumPhotoClickRatio" : {
            "value" : 0.0
          },
          "max_stat_time" : {
            "value" : 1.6225056E12,
            "value_as_string" : "2021-06-01 00:00:00"
          },
          "sumActionRatio" : {
            "value" : 0.04498886414253898
          },
          "sumPhotoClick" : {
            "value" : 0.0
          },
          "sumAClick" : {
            "value" : 47147.0
          }
        },
        {
          "key" : "2021-06-01T00:00:00.000Z####4535562980",
          "doc_count" : 3,
          "sumBClick" : {
            "value" : 6419.0
          },
          "top_hits_fields" : {
            "hits" : {
              "total" : {
                "value" : 3,
                "relation" : "eq"
              },
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "ks_creative_report_index",
                  "_type" : "_doc",
                  "_id" : "251",
                  "_score" : null,
                  "_source" : {
                    "advertiser_id" : 10175341,
                    "advertiser_name" : "DraGon",
                    "created_at" : "2021-06-11T14:08:48.696+08:00",
                    "stat_year" : 2021,
                    "stat_week" : 2684,
                    "stat_month" : "2021-06-01",
                    "stat_date" : "2021-06-09",
                    "stat_hour" : 0,
                    "charge" : 0.0,
                    "show" : 0,
                    "photo_click" : 0,
                    "aclick" : 6,
                    "bclick" : 0,
                    "photo_click_ratio" : 0.0
                  },
                  "sort" : [
                    1623391728696
                  ]
                }
              ]
            }
          },
          "sumShow" : {
            "value" : 1.0
          },
          "sumCharge" : {
            "value" : 825.295
          },
          "sumPhotoClickCost" : {
            "value" : 0.0
          },
          "sumActionCost" : {
            "value" : 0.12857064963389936
          },
          "sumPhotoClickRatio" : {
            "value" : 0.0
          },
          "max_stat_time" : {
            "value" : 1.6225056E12,
            "value_as_string" : "2021-06-01 00:00:00"
          },
          "sumActionRatio" : {
            "value" : 0.027877061247888267
          },
          "sumPhotoClick" : {
            "value" : 0.0
          },
          "sumAClick" : {
            "value" : 230267.0
          }
        },
        {
          "key" : "2021-05-01T00:00:00.000Z####4535424809",
          "doc_count" : 5,
          "sumBClick" : {
            "value" : 3573.0
          },
          "top_hits_fields" : {
            "hits" : {
              "total" : {
                "value" : 5,
                "relation" : "eq"
              },
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "ks_creative_report_index",
                  "_type" : "_doc",
                  "_id" : "247",
                  "_score" : null,
                  "_source" : {
                    "advertiser_id" : 10175341,
                    "advertiser_name" : "DraGon",
                    "created_at" : "2021-06-11T14:08:48.373+08:00",
                    "stat_year" : 2021,
                    "stat_week" : 2683,
                    "stat_month" : "2021-05-01",
                    "stat_date" : "2021-05-30",
                    "stat_hour" : 0,
                    "charge" : 0.0,
                    "show" : 0,
                    "photo_click" : 0,
                    "aclick" : 0,
                    "bclick" : 0
                  },
                  "sort" : [
                    1623391728373
                  ]
                }
              ]
            }
          },
          "sumShow" : {
            "value" : 0.0
          },
          "sumCharge" : {
            "value" : 281.904
          },
          "sumPhotoClickCost" : {
            "value" : 0.0
          },
          "sumActionCost" : {
            "value" : 0.17471894132044885
          },
          "sumPhotoClickRatio" : {
            "value" : 0.0
          },
          "max_stat_time" : {
            "value" : 1.6198272E12,
            "value_as_string" : "2021-05-01 00:00:00"
          },
          "sumActionRatio" : {
            "value" : 0.055509164973328154
          },
          "sumPhotoClick" : {
            "value" : 0.0
          },
          "sumAClick" : {
            "value" : 125287.0
          }
        },
        {
          "key" : "2021-05-01T00:00:00.000Z####4535562980",
          "doc_count" : 4,
          "sumBClick" : {
            "value" : 5550.0
          },
          "top_hits_fields" : {
            "hits" : {
              "total" : {
                "value" : 4,
                "relation" : "eq"
              },
              "max_score" : null,
              "hits" : [
                {
                  "_index" : "ks_creative_report_index",
                  "_type" : "_doc",
                  "_id" : "245",
                  "_score" : null,
                  "_source" : {
                    "advertiser_id" : 10175341,
                    "advertiser_name" : "DraGon",
                    "created_at" : "2021-06-11T14:08:48.198+08:00",
                    "stat_year" : 2021,
                    "stat_week" : 2683,
                    "stat_month" : "2021-05-01",
                    "stat_date" : "2021-05-28",
                    "stat_hour" : 0,
                    "charge" : 0.0,
                    "show" : 0
                  },
                  "sort" : [
                    1623391728198
                  ]
                }
              ]
            }
          },
          "sumShow" : {
            "value" : 5.0
          },
          "sumCharge" : {
            "value" : 718.096
          },
          "sumPhotoClickCost" : {
            "value" : 0.0
          },
          "sumActionCost" : {
            "value" : 0.2582872492878599
          },
          "sumPhotoClickRatio" : {
            "value" : 0.0
          },
          "max_stat_time" : {
            "value" : 1.6198272E12,
            "value_as_string" : "2021-05-01 00:00:00"
          },
          "sumActionRatio" : {
            "value" : 0.20659449424240856
          },
          "sumPhotoClick" : {
            "value" : 0.0
          },
          "sumAClick" : {
            "value" : 278472.0
          }
        }
      ]
    }
  }
}
```

# 其他

[es数据类型和sql的数据类型对照表](https://www.elastic.co/guide/en/elasticsearch/reference/7.x/sql-data-types.html)

# 使用中遇到的问题

- es中字段没有实时更新

<aside>
💡 es 更新数据是异步更新

</aside>

- create sql 生成dsl脚本

```python
#coding:utf-8
import sqlparse
import json
import sys
import os

#类型映射
type_map = {}
type_map["int"] = "integer"
type_map["bigint"] = "long"
type_map["datetime"] = "date"
type_map["longtext"] = "keyword"
type_map["varchar"] = "keyword"
type_map["double"] = "double"
type_map["tinyint"] = "byte"
type_map["text"] = "keyword"

def extract_definitions(token_list):
    # assumes that token_list is a parenthesis
    definitions = []
    tmp = []
    par_level = 0
    for token in token_list.flatten():
        if token.is_whitespace:
            continue
        elif token.match(sqlparse.tokens.Punctuation, '('):
            par_level += 1
            continue
        if token.match(sqlparse.tokens.Punctuation, ')'):
            if par_level == 0:
                break
            else:
                par_level += 1
        elif token.match(sqlparse.tokens.Punctuation, ','):
            if tmp:
                definitions.append(tmp)
            tmp = []
        else:
            tmp.append(token)
    if tmp:
        definitions.append(tmp)
    return definitions

def print_dsl(sql):
    parsed = sqlparse.parse(sql)[0]
    # extract the parenthesis which holds column definitions
    n, par = parsed.token_next_by(i=sqlparse.sql.Parenthesis)
    columns = extract_definitions(par)

    result = """
    {
        "settings": {
            "number_of_shards": 5,
            "number_of_replicas": 1
        },
        "mappings":
            %s

    }
    """

    fields = {}
    for column in columns:
        try:
            name = str(column[0]).strip("`")
            type_str = str(column[1])
            if type_str in type_map:
                field_define = {}
                field_define["type"] = type_map[type_str]
                if type_str == "datetime":
                    field_define["format"] = "yyyy-MM-dd HH:mm:ss || yyyy-MM-dd HH:mm:ss.SSS ||yyyy-MM-dd || epoch_millis || strict_date_optional_time || yyyy-MM-dd'T'HH:mm:ss'+'08:00"
                fields[name] = field_define
            else:
                #print("%s filed, %s type not support........" % (type_str, name))
                print("====", column)
        except Exception as e:
            print(e)

    properties = {"properties": fields}

    prop_str = json.dumps(properties, indent=4)
    print(result % prop_str)

if __name__== "__main__":
    f = open("ks.sql", "r")
    result = f.read()
    f.close()
    print_dsl(result)
```

## 设置ES配置项

- 查看节点配置

GET /_nodes/stats?pretty

```bash

{
   "_nodes" : {
     "total" : 1,
     "successful" : 1,
     "failed" : 0
   },
   "cluster_name" : "docker-cluster",
   "nodes" : {
     "X943-KJnQri4wKRII-lHFg" : {
       "timestamp" : 1627285589316,
       "name" : "elasticsearch-0",
       "transport_address" : "172.20.23.40:9300",
       "host" : "172.20.23.40",
       "ip" : "172.20.23.40:9300",
       "roles" : [
         "ingest",
         "master",
         "data"
       ],
       "attributes" : {
         "ml.machine_memory" : "2126512128",
         "xpack.installed" : "true",
         "ml.max_open_jobs" : "20"
       },
       "indices" : {
         "docs" : {
           "count" : 932402,
           "deleted" : 5046
         },
         "store" : {
           "size_in_bytes" : 211372933
         },
```

- 查看indices配置

```bash
GET /{index}/_settings
```

- 查看集群配置

```bash
GET /_cluster/settings?pretty
```

- 查看字段内存分配

```bash
GET /_stats/fielddata?fields=*
```

## 常见错误

[常见错误](https://www.notion.so/e9656644c3d24feabe6320eb432b70a7)
