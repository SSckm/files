## 说明

```
最近在做一个搜索博客项目，能够通过关键字来进行搜索。以前也使用过Elasticsearch，所以就直接使用ES来作为搜索服务器，本篇文章主要讲解搜索条件的使用
```

## 环境准备

```
ubuntu 16.04
elasticsearch-6.1.1
```

## 安装

```
直接下载elasticsearch就可使用，jdk安装就不讲了。
下载地址：https://www.elastic.co/downloads/elasticsearch
```

## 启动ES

```
bin/elasticsearch -d （后台运行）
```

## 安装IK中文分词

执行下面的命令

```
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.1.1/elasticsearch-analysis-ik-6.1.1.zip
```

IK分词github地址为:https://github.com/medcl/elasticsearch-analysis-ik(要找对版本哦)

## 创建索引

```
curl -XPUT "http://127.0.0.1:9200/blog"
返回：
{"acknowledged":true,"shards_acknowledged":true,"index":"yourindex"}
创建索引成功
```

## 创建mapping

```
curl -H 'Content-Type:application/json' -XPOST http://127.0.0.1:9200/blog/blog/_mapping -d'
{
    "properties": {
        "content": {
            "type": "text",
            "analyzer": "ik_max_word"
        },
        "createDate": {
            "type": "date"
        }
        "id": {
            "type": "long"
        },
        "tags": {
            "properties": {
                "name": {
                    "type": "text"
                }
            }
        },
        "title": {
            "type": "text",
            "analyzer": "ik_max_word"
        }
    }
}'
```

## 需要注意的地方

```
ES在6.0.0版本以前是可以在同一个index下面创建多个type的，但是在6.0.0以后就不允许这样做了，只能映射一个type
详情请参开：https://www.elastic.co/guide/en/elasticsearch/reference/6.x/removal-of-types.html，而如果还是要添加两个type则会报错，这点一定要注意。
```

## 添加一个索引

```
curl -H 'Content-Type:application/json' -XPOST http://127.0.0.1:9200/blog/blog -d'
{
    "content": "在项目开发中特别是业务系统开发，Session都起到了很重要的作用，Session存放于服务端中，但是在集群模式下Session的实现都有哪几种方式呢，以及需要Load Balance需要配合怎么做呢",
    "createDate": "2017-12-12",
    "id": 88,
    "tag": [
        {
            "name": "session"
        },
        {
            "name": "java"
        },
        {
            "name": "分布式"
        }
    ],
    "title": "session在分布式场景下的处理方式"
}'
```

## 查询条件说明-match查询

```
查询关键字为：Session 谷歌
{
  "query": {
    "match": {
        "content" : {
            "query" : "Session 百度"
        }
    }
  }
}
```

说明：你输入的关键字会被分词成 Session和百度

match查询中还可以设置一个minimum_should_match字段：他的作用是至少匹配四个词汇中的三个才能匹配成功。

```
{
  "query": {
    "match": {
        "content" : {
            "query" : "Session 百度",
            "operator": "and" （默认值为OR）
        }
    }
  }
}
```

```
match查询体里面有一个operator参数，作用是：
1.operator， 默认值为OR，就拿上面的例子来说吧，如果operator=OR（默认值），那么这个文档是会被搜索出来的因为匹配到了Session，OR的意思就是匹配上Session或者百度的就会被查询出来
```

### 使用and等同于下面的查询条件

```
{
    "bool": {
        "must": [
            { "term": { "content": "Session" }},
            { "term": { "content": "百度" }}
        ]
    }
}
```

## 查询条件说明-multi_match查询

```
{
    "multi_match": {
        "query": "Session 百度",
        "fields": [
            "content",
            "tags.name",
            "title"
        ],
        "type": "best_fields",
        "operator": "OR",
        "analyzer": "ik_max_word"
    }
}
```

### 说明如下

```
operator:OR和上面提到的match一样，就不再阐述了，analyzer是选择关键字要用什么分词器去分词，比如我这里选择的就是ik_max_word，就会被分词成Session和百度，"type": "best_fields",就是同时匹配上Session 百度的词条评分会比较高。
fields：根据关键字去文档里面的这些字段里面去查询
```



## 查询条件说明-Term  Query

这里借用一下ES官网上的例子

### 首先我们建立一个索引的映射关系

```
{
    "mappings": {
        "my_type": {
            "properties": {
                "full_text": {
                    "type": "text"
                },
                "exact_value": {
                    "type": "keyword"
                }
            }
        }
    }
}
```

此处需要说明full_text 类型为text在建立文档的时候会进行分词，exact_value在建立文档的时候不会分词

### 建立文档

```
{
    "full_text": "Quick Foxes!",
    "exact_value": "Quick Foxes!"
}
```

此时：full_text将包含两个词汇  ["Quick","Foxes"], 而exact_value则只包含一个词汇 ["Quick Foxes!"]

### 查询实例

```
查询一：
{
    "query": {
        "term": {
            "exact_value": "Quick Foxes!"
        }
    }
}
查询二：
{
    "query": {
        "term": {
            "full_text": "Quick Foxes!"
        }
    }
}
查询三：
{
    "query": {
        "term": {
            "full_text": "foxes"
        }
    }
}
查询四：
{
    "query": {
        "match": {
            "full_text": "Quick Foxes!"
        }
    }
}
```

### 结果

```
查询一 ： 匹配成功， term不对query的关键字分词，所以匹配成功
查询二 ： 匹配失败， 查询关键字为Quick Foxes!， 查询类型是term不对查询内容分词，而full_text里面的两个词汇都不匹配Quick Foxes!， 所以匹配失败
查询三 ： 匹配成功， 不对foxes分词，full_text的两个词汇中的Foxes匹配成功
查询四 ： 匹配成功， match查询对Quick Foxes!进行分词，两个都能匹配上，查询成功
```

### term中的boost字段

```
{
    "query": {
        "bool": {
            "should": [
                {
                    "term": {
                        "status": {
                            "value": "urgent",
                            "boost": 2
                        }
                    }
                },
                {
                    "term": {
                        "status": "normal"
                    }
                }
            ]
        }
    }
}
```

说明：boost字段的意思是该字段如果匹配urgent成功的重要性是匹配normal条件的两倍，boost的默认值为1

## 查询条件说明-bool联合查询-should, must, must_not

### 例子

```
{
    "query": {
        "bool": {
            "filter": [
                {
                    "term": {
                        "isDelete": {
                            "value": 0,
                            "boost": 1
                        }
                    }
                }
            ],
            "must": {
                "match": {
                    "content": "2017-12-14"
                }
            },
            "must": {
                "term": {
                    "content": "2017-12-14"
                }
            },
            "must_not": {
                "match": {
                    "title": "分布式"
                }
            },
            "should": [
                {
                    "multi_match": {
                        "query": "入门",
                        "fields": [
                            "content^1.0",
                            "tags.name^1.0",
                            "title^1.0"
                        ],
                        "type": "best_fields",
                        "operator": "OR",
                        "analyzer": "ik_max_word",
                        "slop": 0,
                        "prefix_length": 0,
                        "max_expansions": 50,
                        "zero_terms_query": "NONE",
                        "auto_generate_synonyms_phrase_query": true,
                        "fuzzy_transpositions": true,
                        "boost": 1
                    }
                },
                {
                    "regexp": {
                        "title": {
                            "value": "~*入门~*",
                            "flags_value": 65535,
                            "max_determinized_states": 10000,
                            "boost": 1
                        }
                    }
                },
                {
                    "regexp": {
                        "content": {
                            "value": "~*入门~*",
                            "flags_value": 65535,
                            "max_determinized_states": 10000,
                            "boost": 1
                        }
                    }
                }
            ],
            "adjust_pure_negative": true,
            "minimum_should_match": "75%",
            "boost": 1
        }
    }
}
```

## 说明

```
1.filter：isDelete = 0 是过滤条件，严格匹配。不再多说
2.must和must_not根据字面意思就是必须必须不条件匹配比较简单，must中可以是match查询也可以是term查询
3.这里重点收下should, should下面有很多查询条件比如regexp， multi_match等，那么需要匹配几个条件才能查询出文档呢，下面有字段minimum_should_match， 三个条件中的75%会折算成2，即三条 should 语句中至少有两条必须匹配。也可以直接写成"minimum_should_match": "2"，两个值等价。
```

