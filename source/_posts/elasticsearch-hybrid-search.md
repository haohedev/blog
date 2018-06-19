---
title: Elasticsearch中文汉字拼音混合搜索
date: 2018-06-19 09:32:58
tags:
    - JavaScript
categories:
    - Elasticsearch
---

## 背景
因为在工作上用到了`Elasticsearch`搜索，需求中包含`中文`+`拼音`混合搜索，在网上看到了[Elasticsearch高级搜索排序（ 中文+拼音+首字母+简繁转换+特殊符号过滤）](http://www.cnblogs.com/clonen/p/6674888.html)这篇文章，提供了思路。但是该文章使用的应该是`Elasticsearch 2.X`版本，因此本文同样采用该思路，但是利用`Elasticsearch 5.6.3` + `JavaScript API`的方案来简易实现一版。`Elasticsearch 5.X`和`Elasticsearch 6.X`版本，在本文中差异不大，因此都可以使用。

## Elasticsearch插件
安装插件肯定是最先开始的工作，这里用到了[IK分词器](https://github.com/medcl/elasticsearch-analysis-ik)和[Pinyin分词器](https://github.com/medcl/elasticsearch-analysis-pinyin)，需要选择与`Elasticseach`对应的版本。 这里提供一个`Dockerfile`文件，生成基于`Elasticsearch 5.6.3`版本的镜像并且包含了`IK分词器`和`Pinyin分词器`：
```
FROM elasticsearch:5.6.3-alpine

ENV VERSION=5.6.3

ADD https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v${VERSION}/elasticsearch-analysis-ik-$VERSION.zip /tmp/
RUN /usr/share/elasticsearch/bin/elasticsearch-plugin install file:///tmp/elasticsearch-analysis-ik-$VERSION.zip

ADD https://github.com/medcl/elasticsearch-analysis-pinyin/releases/download/v${VERSION}/elasticsearch-analysis-pinyin-$VERSION.zip /tmp/
RUN /usr/share/elasticsearch/bin/elasticsearch-plugin install file:///tmp/elasticsearch-analysis-pinyin-$VERSION.zip

RUN rm -rf /tm/*

```

<!-- more -->
## 创建Elasticsearch客户端对象

```javascript
var client = new elasticsearch.Client({ host: 'localhost:9200' });
```
## 创建索引

``` javascript
client.indices.create({
    index: 'myindex',
    body: {
        settings: {
            analysis: {
                filter: {
                    edge_ngram_filter: {
                        type: 'edge_ngram',
                        min_gram: 1,
                        max_gram: 50
                    },
                    pinyin_simple_filter: {
                        type: 'pinyin',
                        keep_first_letter: true,
                        keep_separate_first_letter: false,
                        keep_full_pinyin: false,
                        keep_original: false,
                        limit_first_letter_length: 50,
                        lowercase: true
                    },
                    pinyin_full_filter: {
                        type: 'pinyin',
                        keep_first_letter: false,
                        keep_separate_first_letter: false,
                        keep_full_pinyin: true,
                        none_chinese_pinyin_tokenize: true,
                        keep_original: false,
                        limit_first_letter_length: 50,
                        lowercase: true
                    }
                },
                analyzer: {
                    ngramIndexAnalyzer: {
                        type: 'custom',
                        tokenizer: 'keyword',
                        filter: ["edge_ngram_filter", "lowercase"]
                    },
                    ngramSearchAnalyzer: {
                        type: 'custom',
                        tokenizer: 'keyword',
                        filter: ["lowercase"],
                    },
                    pinyiSimpleIndexAnalyzer: {
                        tokenizer: 'keyword',
                        filter: ["pinyin_simple_filter", "edge_ngram_filter", "lowercase"]
                    },
                    pinyiSimpleSearchAnalyzer: {
                        tokenizer: "keyword",
                        filter: ["pinyin_simple_filter", "lowercase"]
                    },
                    pinyiFullIndexAnalyzer: {
                        tokenizer: "keyword",
                        filter: ["pinyin_full_filter", "lowercase"]
                    },
                    pinyiFullSearchAnalyzer: {
                        tokenizer: "keyword",
                        filter: ["pinyin_full_filter", "lowercase"]
                    }
                }
            }
        },
        mappings: {
            doc: {
                properties: {
                    name: {
                        type: 'text',
                        index: "analyzed",
                        analyzer: "ngramIndexAnalyzer",
                        search_analyzer: "ngramSearchAnalyzer",
                        fields: {
                            SPY: {
                                type: "text",
                                index: "analyzed",
                                analyzer: "pinyiSimpleIndexAnalyzer",
                                search_analyzer: "pinyiSimpleSearchAnalyzer"
                            },
                            FPY: {
                                type: "text",
                                index: "analyzed",
                                analyzer: "pinyiFullIndexAnalyzer",
                                search_analyzer: "pinyiFullSearchAnalyzer"
                            },
                            IKS: {
                                type: "text",
                                index: "analyzed",
                                analyzer: "ik_smart",
                                search_analyzer: "ik_smart"
                            }
                        }
                    }
                }
            }
        }
    }
}, function (err, resp) {
    if (err) {
        console.error("创建索引失败: " + err.statusCode + " " + err.message);
    } else {
        console.success(`创建索引完成`);
    }
});
```

下面主要看一下`settings`中的部分。`分析器(analyzer)` = `字符过滤器(char_filter)` + `分词器(tokenizer)` + `词单元过滤器(filter)`。因为我们要自定义分析器，所以我们需要先定义好`过滤器(filter)`和`分词器(tokenizer)`。

### 过滤器(filter)
**edge_ngram_filter：** 将每个词都进行进一步的切分，用于即时搜索(instant search)。`min_gram`表示只要用户搜索了一个字符我们就去进行匹配。`max_gram`表示匹配的最大长度，最大长度越长越占用空间。更多细节可以参见[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-edgengram-tokenizer.html)或者[指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_ngrams_for_partial_matching.html)。  
**pinyin_simple_filter：** 拼音首字母的过滤器  
**pinyin_full_filter：** 拼音全拼的过滤器  

### 分析器(analyzer)
同样的分析器，我们分别定义了`索引分析器`和`查询分析器`。即在索引时，我们利用分析器的规则进行索引，在查询时会提高查询速度。当然也可以存储原始数据，在查询时利用分析器去匹配。

### 映射(mappings)
我们映射采用的是名为`name`的[多数字段](https://www.elastic.co/guide/cn/elasticsearch/guide/current/most-fields.html)，采用了`ngramIndexAnalyzer`作为索引时的分析器，用`ngramSearchAnalyzer`作为搜索分析器。`name.SPY`、`name.FPY`和`name.IKS`分别为`name`的`首字母拼音`、`全拼`以及`关键字`的映射。


## 搜索

```javascript
function search(text) {
    const response = await client.search({
        index: 'myindex',
        search_type: 'dfs_query_then_fetch', //在测试环境使用
        body: {
            query: {
                multi_match: {
                    query: text,
                    fields: [
                        "name^5",
                        "name.FPY",
                        "name.SPY",
                        "name.IKS^0.8"
                    ],
                    type: "best_fields"
                }
            }
        }
    });
    return response;
}
```

可以看到，我们使用了`multi_match`查询，指定了不同字段不同的权重。如果从头开始完全匹配，则权重值最高，我们设置为`5`，如果匹配上了全拼或者首字母拼音，权重其次，我们设置为`1`，最后当有关键字时，我们设置权重为`0.8`。因为我们这里仅匹配一个字段，所以`multi_match`的`type`我们设置为`best_fields`。在实际项目中，我们可以为不同的需求设置不同的权重值，如果还有其他字段参与相关度的评分，例如描述等，我们可以设置`type`为`most_fields`或`cross_fields`。更多细节参阅指南的[跨字段实体搜索](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cross_fields_entity_search.html)、[字段中心式查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/field-centric.html)和[corss-fields跨字段查询](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_cross_fields_queries.html)

> 由于我们没有只在主分片上创建索引，并且在测试的环境下样本数量会比较少，为保证测试时返回的相关度是正确的，因此指定了`search_type`为`dfs_query_then_fetch`。当在生产环境时，随着数据量的增加，指定`dfs_query_then_fetch`无疑会增加搜索时间并且是毫无意义的。更多信息请查看指南中的[被破坏的相关度!](https://www.elastic.co/guide/cn/elasticsearch/guide/current/relevance-is-broken.html)


---
> **本文作者：** 郝赫   
> **本文链接：** https://zyzz.xyz/elasticsearch-hybrid-search/   
> **版权声明：** 本作品采用 [知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh) 进行许可。转载请注明出处！  
> ![license](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png)