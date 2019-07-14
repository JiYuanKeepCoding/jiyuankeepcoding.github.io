---
layout:     post
title:      简述elasticsearch搜索与评分
date:       2019-07-14
author:     JY
header-img: img/cat0_dns.jpeg
catalog: true
tags:
    - elasticsearch
    - 数据库
    - 搜索引擎
---

#### 一个MLT(More Like This Query)例子
```
GET /qna-altus1/qna/_search
{
    "explain": true, 
    "query": {
        "more_like_this" : {
            "fields" : ["question"],
            "like" : "how to flexup flexdown",
            "min_term_freq" : 1,
            "max_query_terms" : 12,
            "min_doc_freq": 1
        }
    }
}
```
返回结果
```
{
  "took": 3,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.5753642,
    "hits": [
      {
        "_shard": "[qna-altus1][0]",
        "_node": "rAJc7xZOTH6i9pZBjHjOhg",
        "_index": "qna-altus1",
        "_type": "qna",
        "_id": "L1aV72sBtT_cxm0ENP44",
        "_score": 0.5753642,
        "_source": {
          "question": "flexup flexdown vm",
          "answer": "wikilink"
        },
        "_explanation": {
          "value": 0.5753642,
          "description": "sum of:",
          "details": [
            {
              "value": 0.2876821,
              "description": "weight(question:flexdown in 0) [PerFieldSimilarity], result of:",
              "details": [
                {
                  "value": 0.2876821,
                  "description": "score(doc=0,freq=1.0 = termFreq=1.0\n), product of:",
                  "details": [
                    {
                      "value": 0.2876821,
                      "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:",
                      "details": [
                        {
                          "value": 1,
                          "description": "docFreq",
                          "details": []
                        },
                        {
                          "value": 1,
                          "description": "docCount",
                          "details": []
                        }
                      ]
                    },
                    {
                      "value": 1,
                      "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:",
                      "details": [
                        {
                          "value": 1,
                          "description": "termFreq=1.0",
                          "details": []
                        },
                        {
                          "value": 1.2,
                          "description": "parameter k1",
                          "details": []
                        },
                        {
                          "value": 0.75,
                          "description": "parameter b",
                          "details": []
                        },
                        {
                          "value": 3,
                          "description": "avgFieldLength",
                          "details": []
                        },
                        {
                          "value": 3,
                          "description": "fieldLength",
                          "details": []
                        }
                      ]
                    }
                  ]
                }
              ]
            },
            {
              "value": 0.2876821,
              "description": "weight(question:flexup in 0) [PerFieldSimilarity], result of:",
              "details": [
                {
                  "value": 0.2876821,
                  "description": "score(doc=0,freq=1.0 = termFreq=1.0\n), product of:",
                  "details": [
                    {
                      "value": 0.2876821,
                      "description": "idf, computed as log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5)) from:",
                      "details": [
                        {
                          "value": 1,
                          "description": "docFreq",
                          "details": []
                        },
                        {
                          "value": 1,
                          "description": "docCount",
                          "details": []
                        }
                      ]
                    },
                    {
                      "value": 1,
                      "description": "tfNorm, computed as (freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength)) from:",
                      "details": [
                        {
                          "value": 1,
                          "description": "termFreq=1.0",
                          "details": []
                        },
                        {
                          "value": 1.2,
                          "description": "parameter k1",
                          "details": []
                        },
                        {
                          "value": 0.75,
                          "description": "parameter b",
                          "details": []
                        },
                        {
                          "value": 3,
                          "description": "avgFieldLength",
                          "details": []
                        },
                        {
                          "value": 3,
                          "description": "fieldLength",
                          "details": []
                        }
                      ]
                    }
                  ]
                }
              ]
            }
          ]
        }
      }
    ]
  }
}
```

#### elasticsearch是如何用like里面的input匹配文档的
这里要先追溯到index创建阶段。由于没有指定Analyzer，所以索引会使用默认的Standard Analyzer。根据ES文档中给出的定义，一个analyzer可以提供以下三个功能:
1. 根据字符流filter掉一些指定的字符比如"."，","，"${}"
2. 分词，最简单的例子就是根据空格来分词
3. 过滤词，这个时候我们可以更新，删除句子里面的单词。这里可以做诸如同义词处理，大写转小写，去除一些常用词如"the", "a"这些

Anaylzer不仅在插入一个新的文档时analyze文档中的字段，也在搜索的时候analyze来自api的input。这样才能完成用户输入跟文档内容的匹配，需要注意的是需要analyzed的字段的type需要指定成"Text"

到了这一步，问题就变成了token array之间的匹配。elasticsearch需要在众多文档中选出相关度最高的。

#### elasticsearch打分算法
从上面例子的返回结果中我们可以看到匹配到的唯一的文档总分是0.5753642分。"flexup","flexdown"两个词各自都在与hit到的文档匹配的过程中得到了0.2876821分。
由此可见打分算法就是把input的每一个词与目标文档的匹配度加总。
在单个词的匹配中有以下一些要素:
1.关键词在文档中出现的频率
2.比如input换成是how to flexup flexdown vm,匹配到了10个文档，其中9个文档里面都包含how to，然后flexup flexdown都没有一个匹配得到.那么how to这两个词的得分就会很低，可能低于我们指定的score threshold。这样就避免了匹配到各种不相关内容。
IDF算法就是用来处理这种情况的。

具体公式如下:

<img src="img/es_score_formula.png" />

N表示将查询关键字分词后得到的N个term。

IDF=log(1 + (docCount - docFreq + 0.5) / (docFreq + 0.5))

tfNorm=(freq * (k1 + 1)) / (freq + k1 * (1 - b + b * fieldLength / avgFieldLength))

 

docCount：查询中满足查询条件的所有文档

docFreq：满足本条term的查询文档数目

 

IDF反映的是term的影响因子，如果docCount很大，docFreq很小，标示该term在doc之间具有很好的分辨力，当然IDF值也就越大。

 

freq：查询term在本doc的field中出现的次数

K1：调优参数默认为1.2

b:调优参数，默认为0.75

fieldLength：是满足查询条件的doc的filed的长度

avgFieldLength：是满足查询条件的所有doc的filed的长度.

tfNorm反映的该term在所有满足条件的doc中field中的重要性，一般来说，相同的freq 下，field的长度越短，那么取值就越高。
