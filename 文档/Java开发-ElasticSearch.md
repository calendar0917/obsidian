---
title: "ElasticSearch"
subtitle: "记录 ElasticSearch 的学习"
summary: "记录 ElasticSearch 的学习"
description: "记录 ElasticSearch 的学习"
date: 2025-10-04
lastmod: 2025-10-04
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## 初识

- 分布式的开源搜索引擎

- 提供 Restful 接口，所有语言均可调用

ELK技术栈：

- 结合 kibana（可视化）、Logstash、Beats
- 用于日志分析、事实监控等

### 倒排索引

正向索引：

- 传统数据库使用，查询需要逐一遍历

倒排索引：

- document：文档，每条数据就是一个文档
- term：词条，由文档按语义划分，有限且唯一
- 先搜词条，再根据词条找文档

### IK分词器

作为 ES 插件导入

根据现有词典（可拓展）对文档进行划分

### 基础概念

索引库：

- 相同类型的文档（Json存储）的集合

映射：

- 索引库中对文档的约束

![image-20251004090358005](https://raw.githubusercontent.com/calendar0917/images/master/image-20251004090358005.png)

#### mapping 属性

- type：字段数据类型 
- index：是否创建索引
- analyzer：分词器

- propertis：嵌套的子字段

### 索引库操作

RESTFUL 规范

- 不同请求方式对应不同请请求类型

创建索引库和mapping的请求语法如下：

```json
PUT /索引库名称
{
  "mappings": {
    "properties": {
      "字段名1": {
        "type": "text",
        "analyzer": "ik_smart"
      },
      "字段名2": {
        "type": "keyword",
        "index": "false"
      },
      "字段名3": {
        "properties": {
          "子字段": {
            "type": "keyword"
          }
        }
      },
      // …略
    }
  }
}
```

支持 put、get、delete

### 文档操作

新增文档：

```json
POST /索引库名/_doc/文档id
{
  "字段1": "值1",
  "字段2": "值2",
  "字段3": {
    "子属性1": "值3",
    "子属性2": "值4"
  },
  // …
}
```

修改

- put 全量修改，先删除再新建
- post 增量修改

批处理

```
POST /_bulk
{ "index" : { "_index" : "索引库名", "_id" : "1" } }
{ "字段1" : "值1", "字段2" : "值2" }
{ "index" : { "_index" : "索引库名", "_id" : "1" } }
{ "字段1" : "值1", "字段2" : "值2" }
{ "index" : { "_index" : "索引库名", "_id" : "1" } }
{ "字段1" : "值1", "字段2" : "值2" }
{ "delete" : { "_index" : "test", "_id" : "2" } }
{ "update" : {"_id" : "1", "_index" : "test"} }
{ "doc" : {"field2" : "value2"} }
```

### DSL 查询

分类：

- 叶子查询：特定字段查询特定值
- 复合查询：逻辑方式组合叶子查询

基本语法：

```Json
GET /indexName/_search
{
  "query": {
    "查询类型": {
      "查询条件": "条件值"
    }
  }
}
```

#### 叶子

- 全文检索：分词
  - match 查询
  - mult_match 允许同时查询多个字段
- 精确查询：不分词，直接精确匹配
  - term 查询，整体到**词条中**寻找
  - range
- 地理查询：用于搜索地理位置

#### 复合

- 基于逻辑运算组合叶子
  - bool：子句must、should、must_not、filer
- 基于算法修改查询时的文档相关性算分，从而改变排名
  - function_score
  - dis_max

#### 排序和分页

排序：

- 添加 `sort` 标签，默认按照 _score 排序

分页：

- 添加 `from 、size` 

#### 深度分页问题：

- es 一般对数据进行分片存储，导致查询数据时需要汇总各个分片的数据

解决方案：

- `search after`，分页时需要排序，每次查询从上一次的排序值开始。但只能向后逐页查询
- `scrool`，将排序数据形成快照，保存在内存
- 设置上限

#### 高亮显示

在搜索结果中把搜索关键字突出显示

- `field`标签加上`pre_tags、post_tags`

## Java 客户端

JavaRestClient

### 初始化

```java
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(
       HttpHost.create("http://192.168.150.101:9200")
));
```

### Mapping 映射

结合业务分析所需字段（区分是否需要和是否搜索）

- 搜索字段
- 排序字段
- 展示字段

### 索引库操作

基于 RestFul格式：

```java
@Test
void testCreateHotelIndex() throws IOException {
    // 1. 创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("items");
    // 2. 请求参数，MAPPING_TEMPLATE是静态常量字符串，内容是JSON格式请求体
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // 3. 发起请求
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

### 文档操作

新增文档的 API

```java
@Test
void testIndexDocument() throws IOException {
    // 1. 创建request对象
    IndexRequest request = new IndexRequest("indexName").id("1");
    // 2. 准备JSON文档
    request.source("{\"name\": \"Jack\", \"age\": 21}", XContentType.JSON);
    // 3. 发送请求
    client.index(request, RequestOptions.DEFAULT);
}
```

### 文档批处理

add 多个 index，然后统一请求即可

- 完成批量导入数据

```java
void testBulk() throws IOException {
    // 1. 创建Bulk请求
    BulkRequest request = new BulkRequest();
    // 2. 添加要批量提交的请求：这里添加了两个新增文档的请求
    request.add(new IndexRequest("indexName")
            .id("101").source("json source", XContentType.JSON));
    request.add(new IndexRequest("indexName")
            .id("102").source("json source2", XContentType.JSON));
    // 3. 发起bulk请求
    client.bulk(request, RequestOptions.DEFAULT);
}
```

1. 准备文档数据

2. 准备请求参数

3. 发送请求

### RTL 查询

1. `SearchRequest` 对象，发请求
2. 解析结果

3. 得到 Hits 属性，结果是数组

####  复合、排序、分页

用指定对象、设置指定参数即可

`boolquery`、`source`

```Java
@Test
void testBool() throws IOException {
    // 1.创建Request
    SearchRequest request = new SearchRequest("items");
    // 2.组织请求参数
    // 2.1.准备bool查询
    BoolQueryBuilder bool = QueryBuilders.boolQuery();
    // 2.2.关键字搜索
    bool.must(QueryBuilders.matchQuery("name", "脱脂牛奶"));
    // 2.3.品牌过滤
    bool.filter(QueryBuilders.termQuery("brand", "德亚"));
    // 2.4.价格过滤
    bool.filter(QueryBuilders.rangeQuery("price").lte(30000));
    request.source().query(bool);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);
}
```

分页：

```Java
@Test
void testPageAndSort() throws IOException {
    int pageNo = 1, pageSize = 5;

    // 1.创建Request
    SearchRequest request = new SearchRequest("items");
    // 2.组织请求参数
    // 2.1.搜索条件参数
    request.source().query(QueryBuilders.matchQuery("name", "脱脂牛奶"));
    // 2.2.排序参数
    request.source().sort("price", SortOrder.ASC);
    // 2.3.分页参数
    request.source().from((pageNo - 1) * pageSize).size(pageSize);
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);
}
```

高亮：

```Java
@Test
void testHighlight() throws IOException {
    // 1.创建Request
    SearchRequest request = new SearchRequest("items");
    // 2.组织请求参数
    // 2.1.query条件
    request.source().query(QueryBuilders.matchQuery("name", "脱脂牛奶"));
    // 2.2.高亮条件
    request.source().highlighter(
            SearchSourceBuilder.highlight()
                    .field("name")
                    .preTags("<em>")
                    .postTags("</em>")
    );
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);
}
```

### 数据聚合

对文档数据进行统计、分析

- 桶：对文档做而非女足
- 度量 Metric：计算某些特定值
- 管道 Pipeline：以其他聚合的结果为基础做聚合

#### DSL

`aggs` 定义聚合

```json
GET /items/_search
{
  "query": {"match_all": {}}, // 可以省略
  "size": 0, // 设置size为0，结果中不包含文档，只包含聚合结果
  "aggs": { // 定义聚合
    "cateAgg": { // 给聚合起个名字
      "terms": { // 聚合的类型，按照品牌值聚合，所以选择term
        "field": "category", // 参与聚合的字段
        "size": 20 // 希望获取的聚合结果数量
      }
    }
  }
}
```

#### RestClient 构造聚合

指定名称、类型、字段

```java
request.source().size(0);
request.source().aggregation(
        AggregationBuilders
                .terms("brand_agg")
                .field("brand")
                .size(20)
);
```