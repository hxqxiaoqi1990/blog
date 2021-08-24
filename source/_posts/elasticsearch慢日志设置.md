---
title: elasticstarch慢日志设置
tags: [es]
categories: 数据库
date: 2020-07-29 19:00:00
---

# 介绍

Elasticsearch中的慢日志主要有两种：搜索慢日志 （`search slow logs`）和索引慢日志 （`index slow logs`）。

**Search Slow Logs**

搜索慢速日志用于记录慢速搜索。 慢度阈值取决于应用程序及其Elasticsearch实现细节。 每个应用程序可以具有不同的阈值。

在Elasticsearch中进行搜索分为两个阶段：

1. 查询阶段-在查询阶段，Elasticsearch收集相关结果的文档ID。 完成此阶段后，仅返回与搜索匹配的文档的ID，并且不会再出现其他信息，例如字段或它们的值等。
2. 获取阶段-在获取阶段，使用来自查询阶段的文档ID来获取实际文档，由此可以说搜索请求是完整的。

搜索慢速日志显示查询和查询的获取阶段的拆分时间。 因此，我们能够完整地了解完成查询和获取阶段所花费的时间，并且能够检查整个查询本身。

**Index Slow Logs**

索引慢日志用于记录索引过程。 在Elasticsearch中对文档建立索引后，慢速索引日志会记录请求的记录，这些记录需要花费较长的时间才能完成。 同样，在这里，时间窗口也可以在索引日志的配置设置中进行调整。

默认情况下，启用后，Elasticsearch将文档的前1000行记录到日志文件中。 可以将其更改为null或记录整个文档，具体取决于我们如何配置设置。

在下一部分中，让我们看看如何配置日志并检查上面讨论的两种慢速日志类型。

## 搜索慢日志

```bash
PUT testindex-slowlogs/_settings
{
  "index.search.slowlog.level": "info",
  "index.search.slowlog.threshold.query.warn":"5s",
  "index.search.slowlog.threshold.query.info":"2s",
  "index.search.slowlog.threshold.query.debug":"1s",
  "index.search.slowlog.threshold.query.trace":"400ms",
  "index.search.slowlog.threshold.fetch.warn":"1s",
  "index.search.slowlog.threshold.fetch.info":"800ms",
  "index.search.slowlog.threshold.fetch.debug":"500ms",
  "index.search.slowlog.threshold.fetch.trace":"200ms"
}
```

1. 上面例子是设置单索引日志，全局设置改为 `PUT _all/_settings`
2. `index.search.slowlog.level` ：设置日志等级
3. 如果要关闭慢日志，值可以设置为：-1，如果是测试可以全部设置为：0s

# 索引慢速日志记录设置

```bash
PUT testindex-slowlogs/_settings
{
  "index.indexing.slowlog.threshold.index.warn": "10s",
  "index.indexing.slowlog.threshold.index.info": "5s",
  "index.indexing.slowlog.threshold.index.debug": "2s",
  "index.indexing.slowlog.threshold.index.trace": "500ms",
  "index.indexing.slowlog.level": "info",
  "index.indexing.slowlog.source": "1000"
}
```

`index.indexing.slowlog.source` ：默认情况下，Elasticsearch将在慢速日志中记录_source的前1000个字符。 您可以使用index.indexing.slowlog.source进行更改。 将其设置为false或0将完全跳过对源的日志记录，将其设置为true将不考虑大小而记录整个源。 默认情况下，原始_source会重新格式化，以确保它适合单个日志行。 如果保留原始文档格式很重要，则可以通过将index.indexing.slowlog.reformat设置为false来关闭重新格式化，这将导致源按“原样”记录，并可能跨越多个日志行。