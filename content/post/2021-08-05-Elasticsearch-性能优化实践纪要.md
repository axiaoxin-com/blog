---
title: "Elasticsearch 性能优化实践纪要"
author: "axiaoxin"
date: 2021-08-05T16:27:00+08:00
subtitle: ""
image: ""
tags: ["elasticsearch", "debug"]
slug: ""
---

## ES 性能优化实践：节点负载不均衡

**现象**：

每个索引使用默认的 1 个分片，1 个副本，在 3 节点集群出现负载不均衡现象，两个节点 cpu 负载很高，另外一个节点 cpu 负载更低。

**原因**：

分片+副本一共 2 个，只能被分散到两个节点上，查询请求只能命中这两个节点。

**解决方案**：

ES 创建索引后无法动态修改分片数，但是可以动态修改副本数，增加副本数可以分散查询请求。

在不改变分片数的情况下，通过调整分片的副本数，使分片副本数+原始分片数的总数为节点数基数或倍数，以达到节点负载均衡；
将索引的分片副本数增加到 2，加上分片自身，一共3个，刚好均分给3个节点，调整后节点cpu负载到达均衡。

## ES 性能优化实践：慢查询

**现象**：

一个简单的查询，耗时需要 500ms

查询请求参数中使用terms条件查询如下：

```json
{
    "terms": {
        "my_int_field": [
            5,0,0
        ]
    }
}
```

**原因**：

生产环境的数据中， `my_int_field` 字段的默认值为0，这里需求是查特定非 0 值，由于代码逻辑没有过滤默认值，查询字段中出现0，导致几乎全部记录被匹配，于是产生了慢查询。

## ES 性能优化实践：开启ES慢查询日志：

ES 默认没有开启慢查询日志，需要手动调用 api 进行开启。

如下对全部索引都配置开启慢查询日志：

```json
PUT */_settings
{
    "index.indexing.slowlog.threshold.index.debug" : "5ms",
    "index.indexing.slowlog.threshold.index.info" : "50ms",
    "index.indexing.slowlog.threshold.index.warn" : "100ms",
    "index.search.slowlog.threshold.fetch.debug" : "10ms",
    "index.search.slowlog.threshold.fetch.info" : "50ms",
    "index.search.slowlog.threshold.fetch.warn" : "100ms",
    "index.search.slowlog.threshold.query.debug" : "100ms",
    "index.search.slowlog.threshold.query.info" : "200ms",
    "index.search.slowlog.threshold.query.warn" : "500ms"
}
```

具体时长可以根据实际情况调整。

## ES 性能优化实践：使用 filter 子句优化 bool 查询

默认情况下，Elasticsearch 按相关性分数对匹配的搜索结果进行排序，相关性分数衡量每个文档与查询的匹配程度。

相关性分数是一个正浮点数，在搜索 API 的 `_score` 元数据字段中返回。 `_score` 越高，文档越相关。

分数计算决于查询子句是在 query context 中还是在 filter context 中运行。

对于 bool 查询，must 和 should 子句使用 query context，filter 和 must_not 使用 filter context。

query context 关注的是文档的匹配程度，因此查询除了关注文档是否匹配以外还需要计算匹配度得分。

filter context 关注的只是文档是否匹配，没有额外计算，并且 ES 会自动缓存 filter context 的查询结果。

如果只关注是否匹配，尽量使用 filter context 构造查询子句。

> 参考文档：<https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html>

**ES 性能优化实践：查询性能分析**

在 kibana 的 dev tool 中，使用 search profile 可以查看具体耗时分析

![ES 查询性能分析](https://user-images.githubusercontent.com/2876405/128311432-04d11aa8-1192-4981-ada6-a0efb6522beb.png)


从 View details 中可以看到主要耗时是在 should 子句的计算得分

![ES 查询性能分析](https://user-images.githubusercontent.com/2876405/128311453-10bc608d-b4f9-496d-b7ae-767709294cb3.png)


这里的 should 子句在使用 filter context 后，es 会缓存结果，当 3 个节点都有缓存时耗时明显减少，在 console 中进行查询时基本可以在 30 毫秒内返回。


## ES 性能优化实践：terms 查询优化：

将 terms 替换为多个 term 查询可以提高性能。精确匹配使用 term 查询，不要使用 match 查询。

should 子句中使用了 terms 查询，可以看到耗时较多，类型为 PointInSetQuery，这里主要耗时也是在计算得分。

![ES 性能优化实践：terms 查询优化](https://user-images.githubusercontent.com/2876405/128311653-4ff5c67e-4b7e-495d-82a2-02bdd9238ee5.png)

![ES 性能优化实践：terms 查询优化](https://user-images.githubusercontent.com/2876405/128311687-a892fce6-d3a9-4e46-bcd1-848188fc53bb.png)


将 terms 查询条件替换为多个 term 的组合查询，可以看到查询类型变成了 PointRangeQuery 类型，性能又得到了相当大的提升！

![ES 性能优化实践：terms 查询优化](https://user-images.githubusercontent.com/2876405/128311767-5ec8f0a9-c7a0-476a-83f9-fbde686f548f.png)


term 查询数字类型的值使用 range 查询，查询类型会变成 IndexOrDocValuesQuery，性能可以更进一步提升：

![ES 性能优化实践：terms 查询优化](https://user-images.githubusercontent.com/2876405/128311827-4c9dee3d-5ba1-4eb1-91bb-2b719fd2244b.png)


同样的查询场景，耗时从最开始的 270ms->99ms->12ms，性能提升20多倍。
