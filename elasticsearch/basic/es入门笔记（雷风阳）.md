# 一 基础概念

## 一、什么是 Elasticsearch

Elasticsearch 是一个分布式的开源搜索和分析引擎，适用于所有类型的数据，包括文本、数字、地理空间、结构化和非结构化数据。Elasticsearch 在 Apache Lucene 的基础上开发而成，由 Elasticsearch N.V.（即现在的 Elastic）于 2010 年首次发布。Elasticsearch 以其简单的 REST 风格 API、分布式特性、速度和可扩展性而闻名，是 Elastic Stack 的核心组件；Elastic Stack 是适用于数据采集、充实、存储、分析和可视化的一组开源工具。人们通常将 Elastic Stack 称为 ELK Stack（代指 Elasticsearch、Logstash 和 Kibana），目前 Elastic Stack 包括一系列丰富的轻量型数据采集代理，这些代理统称为 Beats，可用来向 Elasticsearch 发送数据。

## 二、Elasticsearch 的用途是什么

Elasticsearch 在速度和可扩展性方面都表现出色，而且还能够索引多种类型的内容，这意味着其可用于多种用例：

- 应用程序搜索
- 网站搜索
- 企业搜索
- 日志处理和分析
- 基础设施指标和容器监测
- 应用程序性能监测
- 地理空间数据分析和可视化
- 安全分析
- 业务分析

## 三、底层实现

Elastic的底层是开源库 Lucene。但是,你没法直接用 scene，必须自己写代码去调用它的接口。 Elastic是 Lucene的封装,提供了 REST API的操作接口，开箱即用。

REST API：天然的跨平台。

## 四、基本概念

### 1. 索引（index）

动词，相当于MySQL的insert

名次，相当于MySQL的Datebase

### 2. 类型（Type）

在 Index(索引)中,可以定义一个或多个类型。类似于 MySQL中的 Table；每一种类型的数据放在一起;

\* PS：在新版本中Type已经作废掉了

> **1、index、type的初衷**
>
> 之前es将index、type类比于关系型数据库（例如mysql）中database、table，这么考虑的目的是“方便管理数据之间的关系”。
>
>  
>
> **2、为什么现在要移除type？**
>
> 2.1 在关系型数据库中table是独立的（独立存储），但es中同一个index中不同type是存储在同一个索引中的（lucene的索引文件），因此不同type中相同名字的字段的定义（mapping）必须一致。
>
> 2.2 不同类型的“记录”存储在同一个index中，会影响lucene的压缩性能。
>
>  
>
> **3、替换策略**
>
> 3.1 一个index只存储一种类型的“记录”
>
> 这种方案的优点：
>
> a）lucene索引中数据比较整齐（相对于稀疏），利于lucene进行压缩。
>
> b）文本相关性打分更加精确（tf、idf，考虑idf中命中文档总数）
>
> 3.2 用一个字段来存储type
>
> 如果有很多规模比较小的数据表需要建立索引，可以考虑放到同一个index中，每条记录添加一个type字段进行区分。
>
> 这种方案的优点：
>
> a）es集群对分片数量有限制，这种方案可以减少index的数量。
>
>  
>
> **4、迁移方案**
>
> 之前一个index上有多个type，如何迁移到3.1、3.2方案？
>
> 4.1 先针对实际情况创建新的index，[3.1方案]有多少个type就需要创建多少个新的index，[3.2方案]只需要创建一个新的index。
>
> 4.2 调用_reindex将之前index上的数据同步到新的索引上。

### 3. 文档（Document）

保存在某个索引（Index）下,某种类型（Type）的一个数据（Document）,文档是 JSON格式的, Document就像是 MySQL中的某个 Table里面的内容

![img](https://img-blog.csdnimg.cn/20210126153735617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

## 五、倒排索引

分词：将整句分拆为单词

保存的记录

1-红海行动

2-探索红海行动

3-红海特别行动

4-红海记录篇

5-特工红海特别探索

检索:

1、红海特工行动  2、红海行动

根据相关性得分进行结果排序

![img](https://img-blog.csdnimg.cn/20210126154005391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

# 二 PUT、POST增加文档和GET搜索文档



### 一、cat

GET _cat/nodes: 查看所有节点

GET _cat/health: 查看es健康状况

GET _cat/master: 查看主节点

GET _cat/indices: 查看所有索引 show databases

![img](https://img-blog.csdnimg.cn/20210126170024776.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

### 二、索引一个文档（保存）

保存一个数据,保存在哪个索引的哪个类型下,指定用哪个唯一标识，在 customer索引下的 external类型下保存1号数据为

```java
PUT mall/_doc/1
{
  "name":"hzh"
}
```

![img](https://img-blog.csdnimg.cn/20210126173859642.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

其实这个请求还支持使用POST方式发送，路径和上面一样的时候结果一致，

POST可以不带ID发送（不带最后的 /1），ES会自动生成一个ID，如果再次请求也会再次新增一个ID

PUT方式如果不带ID将会出现405错误：请求方式不允许

![img](https://img-blog.csdnimg.cn/20210126174626910.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

### 三、查询

```java
GET customer/external/1
```

![img](https://img-blog.csdnimg.cn/20210126174951396.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

_index：表示在哪个索引下

_type：类型

_id：添加时的id

_version：版本号

_seq_no：并发控制字段，序列号，每次更新+1 （乐观锁操作使用）

_primary_term：分片，作用同上，重启会变化

_source：真正的内容

### 四、乐观锁演示

```
PUT http://~:9200/customer/external/1?if_seq_no=2&if_primary_term=1
```

如果效验失败将会显示错误信息（409错误）

![img](https://img-blog.csdnimg.cn/20210126175714981.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

效验通过将会进行修改

![img](https://img-blog.csdnimg.cn/20210126175805705.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIwMDUxNTM1,size_16,color_FFFFFF,t_70)

# 三 更新数据、删除数据和批量操作

### 一、PUT / POST更新数据

POST访问地址（PUT访问提示405方法不允许）：

```
http://127.0.0.1:9200/customer/external/1/_update
```

Body数据：

```
POST mall/_doc/1/_update
{
  "doc":{
    "name":"hzh"
  }
}
```

返回结果：

```
#! Deprecation: [types removal] Specifying types in document update requests is deprecated, use the endpoint /{index}/_update/{id} instead.
{
  "_index" : "mall",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 5,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 5,
  "_primary_term" : 1
}

```

当我们再次发送请求时会发现请求成功，但是版本号没有发生变化，返回结果中result变成了noop，表示没有发生变化

```
"result": "noop",
```

我们可以得出结论：

在更新操作时，URL带_update时，更新前会比对新老数据，如果我们新数据和旧数据完全相同，将不会进行任何操作，不会影响序列号、版本号信息。

如果URI不带_update，不会检查原数据

### 二、增加属性更新

增加属性更新和上方规则相同，直接添加属性都可以正常保存。下面仅以PUT为例：

访问地址：

```
PUT http://127.0.0.1:9200/customer/external/1
```

Body请求数据：

```
{



    "name": "JSON Doe",



    "age": 30



}
```

再次使用GET查询返回结果：



### 三、删除数据信息

访问信息：

```
DELETE http://127.0.0.1:9200/customer/external/1
```

返回结果：

```
{



    "_index": "customer",



    "_type": "external",



    "_id": "1",



    "_version": 10,



    "result": "deleted",



    "_shards": {



        "total": 2,



        "successful": 1,



        "failed": 0



    },



    "_seq_no": 10,



    "_primary_term": 1



}
```

尝试使用GET方法查询，返回没找到：

```
{



    "_index": "customer",



    "_type": "external",



    "_id": "1",



    "found": false



}
```

### 四、删除整个索引

访问信息：

```
DELETE http://127.0.0.1:9200/customer/
```

返回结果：

```
{



    "acknowledged": true



}
```

尝试使用GET方法查询，返回404错误：

```
{
    "acknowledged": true
}
```

### 五、bluk批量操作

由于POSTMan无法完成本功能，会出现400错误，出错原因是由于POSTMan自动优化了换行符号。

我们这时需要采用Kibana完成，官网就能下，可以搜一下教程几个命令安装很简单。

【案例一】

在左侧边栏（如果没有左上角有一个 ”三“ 点一下）点击 Dev Tool输入下面内容：

```
POST /customer/external/_bulk



{"index":{"_id":"1"}}



{"name": "JSON Doe"}



{"index":{"_id":"2"}}



{"name": "Jone Doe"}
```

返回结果：

```
#! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
{
  "took" : 753,
  "errors" : false,
  "items" : [
    {
      "index" : {
        "_index" : "customer",
        "_type" : "external",
        "_id" : "1",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 201
      }
    },
    {
      "index" : {
        "_index" : "customer",
        "_type" : "external",
        "_id" : "2",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}
```

【案例二】

Dev Tool输入下面内容：

```
POST /_bulk



{"delete": {"_index": "website", "_type": "blog", "_id": "123"}}



{"create": {"_index": "website", "_type": "blog", "_id": "123"}}



{"title": "My first blog post"}
```

返回结果：

```
#! Deprecation: [types removal] Specifying types in bulk requests is deprecated.
{
  "took" : 940,
  "errors" : false,
  "items" : [
    {
      "delete" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "123",
        "_version" : 1,
        "result" : "not_found",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 1,
        "status" : 404
      }
    },
    {
      "create" : {
        "_index" : "website",
        "_type" : "blog",
        "_id" : "123",
        "_version" : 2,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 1,
        "status" : 201
      }
    }
  ]
}
```

由此可见批量操作中的每一个操作相互独立，可以独立成功或失败，彼此没有影响。

# 四 基本查询方式

## 一、导入测试数据

ElasticSearch官方为我们准备了一部分测试数据供调试使用，我们可以在安装完成Kinaba后进行数据导入处理

### 1. 获取数据

打开 https://github.com/elastic/elasticsearch/blob/master/docs/src/test/resources/accounts.json 

复制全部数据（点击 Raw 按钮，新页面 Ctrl + A）

### 2. 执行批量添加

打开 Kinaba ：xxxx:5601/app/dev_tools#/console，第一行输入下面第一行，第二行开始粘贴测试数据，点击 ▶️ 运行

```
POST /bank/account/_bulk



{"index":{"_id":"1"}}



{"account_number":1,"balance":39225,"f....(测试数据)
```

## 二、ES支持两种基本方式检索

一个是通过使用 REST request URI发送搜索参数(uri+检索参数)

另一个是通过使用 REST request body来发送它们(uri请求体)

**例子1：**

```
GET bank/_search?q=*&sort=account_number:asc
```

返回结果：

结果并不会返回所有数据而是返回10条数据，类似于分页

```
{
  "took" : 43,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "0",
        "_score" : null,
        "_source" : {
          "account_number" : 0,
          "balance" : 16623,
          "firstname" : "Bradshaw",
          "lastname" : "Mckenzie",
          "age" : 29,
          "gender" : "F",
          "address" : "244 Columbus Place",
          "employer" : "Euron",
          "email" : "bradshawmckenzie@euron.com",
          "city" : "Hobucken",
          "state" : "CO"
        },
        "sort" : [
          0
        ]
      },
...
```

**例子2：**

先按照account_number进行降序，如果相同按照balance进行降序

```
GET bank/_search
{
  "query": {"match_all": {}},
  "sort": [
    {
      "account_number": "desc"
    },
    {
      "balance": "desc"
    }
  ]
}
```

返回结果：

```
{
  "took" : 12,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "999",
        "_score" : null,
        "_source" : {
          "account_number" : 999,
          "balance" : 6087,
          "firstname" : "Dorothy",
...
```

## 三、Query DSL

### 1. 基本语法格式

ElasticSearch提供了一个可以执行查询的son风的DSL（domain-specific language领域特定语言)。这个被称为 Query DSL。该查询语言非常全面,并且刚开始的时候感觉有点复杂,真正学好它的方法是从一些基础的示例开始的。

一个查询语句的典型结构：

```
{
    QUERY_NAME:{
         ARGUMENT:VALUE,
         ARGUMENT:VALUE...
    }    
}
```

例如查询所有：

```
GET bank/_search
{
  "query": {
    "match_all": {}
  }
}
```

如果是针对某个字段查询，查询结构：

```
{
    QUERY_NAME:{
        FIELD NAME:{
            ARGUMENI:VALUE,
            ARGUMENT:VALUE...
        }
    }
}
```

例如按照 balance 降序查询：

```
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "balance": {
        "order": "desc"
      }
    }
  ]
}
```

其实还有一种简单的表达形式，效果相同

```
"balance": {
    "order": "desc"
}
可以简写为：
"balance": "desc"
```

- query定义如何查询
- match_all 查询类型【代表查询所有的所有】，es中可以在 query 中组合非常多的查询类型完成复杂查询
- 除了 query参数之外,我们也可以传递其它的参数以改变查询结果。如sort，size
- from + size限定，完成分页功能
- sort排序，多字段排序，会在前序字段相等时后续字内部排序，否则以前序为准

分页查询且只查询部分属性的例子：

```
GET bank/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "balance": {
        "order": "desc"
      }
    }
  ],
  "from": 0,
  "size": 5,
  "_source": ["balance", "account_number"]
}
```

#  五 Match与Match_phrase

### 一、基本类型（非字符串），精确匹配

查询 account_number 是 20 的所有结果：

```
GET bank/_search
{
  "query": {
    "match": {
      "account_number": 20
    }
  }
}
```

返回内容：

```
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 1.0,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      }
    ]
  }
}
```

### 二、字符串模糊匹配查询

比如我们希望查询所有 address 中包含 Kings 的数据：

```
GET bank/_search
{
  "query": {
    "match": {
      "address": "Kings"
    }
  }
}
```

返回结果：

可以看到 “282 Kings Place” 和 “305 Kings Hwy” 两条记录都返回了

全文检索会按照评分进行排序，会对检索条件进行分词匹配。

```
{
  "took" : 157,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 5.990829,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "20",
        "_score" : 5.990829,
        "_source" : {
          "account_number" : 20,
          "balance" : 16418,
          "firstname" : "Elinor",
          "lastname" : "Ratliff",
          "age" : 36,
          "gender" : "M",
          "address" : "282 Kings Place",
          "employer" : "Scentric",
          "email" : "elinorratliff@scentric.com",
          "city" : "Ribera",
          "state" : "WA"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "722",
        "_score" : 5.990829,
        "_source" : {
          "account_number" : 722,
          "balance" : 27256,
          "firstname" : "Roberts",
          "lastname" : "Beasley",
          "age" : 34,
          "gender" : "F",
          "address" : "305 Kings Hwy",
          "employer" : "Quintity",
          "email" : "robertsbeasley@quintity.com",
          "city" : "Hayden",
          "state" : "PA"
        }
      }
    ]
  }
}
```

### 三、Match_phrase 短语匹配

默认的match搜索会对搜索内容进行分词，比如：mill lane 会分成 mill 和 lane 之后搜索的结果可能包含仅有其中一项的结果，但是此类结果分数较低。

如果不希望被分词可以使用 match_phrase 进行搜索

例子：

查询 地址 包含 mill lane 的结果：

```
GET bank/_search
{
  "query": {
    "match_phrase": {
      "address": "Mill Lane"
    }
  }
}GET bank/_search



{



  "query": {



    "match_phrase": {



      "address": "Mill Lane"



    }



  }



}
```

返回结果：

# 六Multi_match多字段匹配、bool复合查询

### 一、Multi_match多字段匹配

例：查询 address 和 city 中任意一项包含 mill 的结果

```
GET bank/_search
{
  "query": {
    "multi_match": {
      "query": "mill",
      "fields": ["address", "email"]
    }
  }
}
```

返回结果：

```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 5.4032025,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
...
```

### 二、bool复合查询

如果我们面对更加复杂的查询条件需要采用 bool 复合查询

例如：

我们希望查询 gender 是 M、address 包含 mill、年龄不能是28、lastname最好是Hines的结果：

```
GET bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "gender": "M"
          }
        },
        {
          "match": {
            "address": "mill"
          }
        }
      ],
      "must_not": [
        {
          "match": {
            "age": "28"
          }
        }
      ],
      "should": [
        {
          "match": {
            "lastname": "Hines"
          }
        }
      ]
    }
  }
}
```

返回：

```
{
  "took" : 13,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 12.585751,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 12.585751,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 6.0824604,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      }
    ]
  }
}
```

#  七 filter查询

布尔查询中的每个must、should和must not元素都称为查询子句。文档满足每个 must 或 should 子句中的标准的程度有助于文档的相关性得分。分数越高，文档就越符合您的搜索条件。默认情况下，Elasticsearch返回按这些相关性得分排序的文档。

must_not 子句中的条件被视为 *filter*。它影响文档是否包含在结果中，***filter、must_not 都不影响文档的得分\***。

还可以显式指定任意过滤器，以包含或排除基于结构化数据的文档。

例如，我们查找年龄在 10 - 30 的数据

```
GET /bank/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "age": {
              "gte": 10,
              "lte": 30
            }
          }
        }
      ]
    }
  }
}
```

返回结果：

```
{
  "took" : 10,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 498,
      "relation" : "eq"
    },
    "max_score" : 1.0,  //注意这里，使用must贡献了相关性得分
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "13",
        "_score" : 1.0,  //注意这里，使用must贡献了相关性得分
        "_source" : {
          "account_number" : 13,
          "balance" : 32838,
          "firstname" : "Nanette",
          "lastname" : "Bates",
          "age" : 28,
          "gender" : "F",
```

我们也可以使用Filter：

```
GET /bank/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "range": {
            "age": {
              "gte": 10,
              "lte": 30
            }
          }
        }
      ]
    }
  }
}
```

返回的结果是：

```
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 498,
      "relation" : "eq"
    },
    "max_score" : 0.0,   //注意这里没有贡献相关性得分
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "13",
        "_score" : 0.0,   //注意这里没有贡献相关性得分
        "_source" : {
          "account_number" : 13,
          "balance" : 32838,
          "firstname" : "Nanette",
          "lastname" : "Bates",
          "age" : 28,
          "gender" : "F",
          "address" : "789 Madison Street",
          "employer" : "Quility",
          "email" : "nanettebates@quility.com",
          "city" : "Nogal",
          "state" : "VA"
```

#  八 Term和keyword精确查询

### 一、Term查询

返回在提供的字段中包含确切信息的文档内容。

您可以使用精确的值（例如价格，产品ID或用户名）利用 Term 查询查找文档。

比如我们查询年龄是33岁的：

```
GET /bank/_search
{
  "query": {
    "term": {
      "age": 33
    }
  }
}
```

注意：

避免term对text字段使用查询。

默认情况下，Elasticsearch更改text字段的值作为analysis的一部分。这会使查找text字段值的精确匹配变得困难。

要搜索text字段值，请改用match查询。

### 二、文本精确查询

比如查询地址完整包含 435 Furman Street 的：

```
GET /bank/_search
{
  "query": {
    "match_phrase": {
      "address": "435 Furman Street"  //这个的搜索结果在改为435 Furman时依旧会展示
    }
  }
}
```

查询地址值必须是 435 Furman Street 的（精确匹配 keyword）：

```
GET /bank/_search
{
  "query": {
    "match": {
      "address.keyword": "435 Furman Street"  //这个的搜索结果在改为435 Furman时不会展示
    }
  }
}
```

我们一般规定：全文检索字段用 match,其他非text字段匹配用term

# 九 Aggregations执行聚合

聚合提供了从数据中分组和提取数据的能力。最简单的聚合方法大致等于 SQL GROUP BY 和 SQL 聚合函数。在 Elasticsearch 中，您有执行索返回 hits（命中结果），并且同时返回聚合结果，把一个响应中的所有hits（命中结果）隔开的能力。这是非常强大且有效的，您可以执行查询和多个聚合，并且在一次使用得到各自的(任何一个的)返回结果，使用一次简洁和简化的AP来避免网络往返。

【例子1】

搜索 address中包含mill的所有人的年龄分布以及平均年龄,但不显示这些人的详情。

```
GET /bank/_search
{
  "query": {
    "match": {
      "address": "mill" 
    }
  },
  "aggs": {    //获取聚合
    "ageAgg": {      //自定义的聚合名
      "terms": {       //获取结果的不同数据个数
        "field": "age",    //获取字段是age
        "size": 10      //可能有很多很多可能，只获取前10种
      }
    },
    "ageAvg":{   //自定义的聚合名
      "avg": {      //求平均值
        "field": "age"    //获取字段是age
      }
    }
  }
}
```

返回的结果：

```
{
  "took" : 27,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 4,
      "relation" : "eq"
    },
    "max_score" : 5.4032025,
    "hits" : [
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "970",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 970,
          "balance" : 19648,
          "firstname" : "Forbes",
          "lastname" : "Wallace",
          "age" : 28,
          "gender" : "M",
          "address" : "990 Mill Road",
          "employer" : "Pheast",
          "email" : "forbeswallace@pheast.com",
          "city" : "Lopezo",
          "state" : "AK"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "136",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 136,
          "balance" : 45801,
          "firstname" : "Winnie",
          "lastname" : "Holland",
          "age" : 38,
          "gender" : "M",
          "address" : "198 Mill Lane",
          "employer" : "Neteria",
          "email" : "winnieholland@neteria.com",
          "city" : "Urie",
          "state" : "IL"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "345",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 345,
          "balance" : 9812,
          "firstname" : "Parker",
          "lastname" : "Hines",
          "age" : 38,
          "gender" : "M",
          "address" : "715 Mill Avenue",
          "employer" : "Baluba",
          "email" : "parkerhines@baluba.com",
          "city" : "Blackgum",
          "state" : "KY"
        }
      },
      {
        "_index" : "bank",
        "_type" : "account",
        "_id" : "472",
        "_score" : 5.4032025,
        "_source" : {
          "account_number" : 472,
          "balance" : 25571,
          "firstname" : "Lee",
          "lastname" : "Long",
          "age" : 32,
          "gender" : "F",
          "address" : "288 Mill Street",
          "employer" : "Comverges",
          "email" : "leelong@comverges.com",
          "city" : "Movico",
          "state" : "MT"
        }
      }
    ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 38,
          "doc_count" : 2
        },
        {
          "key" : 28,
          "doc_count" : 1
        },
        {
          "key" : 32,
          "doc_count" : 1
        }
      ]
    },
    "ageAvg" : {
      "value" : 34.0
    }
  }
}
```

如果我们不希望返回数据，只需要分析结果，可以设置 size 为 0

```
GET /bank/_search
{
  "query": {~},
  "aggs": {~},
  "size": 0
}
```

【例子2】

按照年龄聚合，并且请求这些年龄段的这些人的平均薪资

```
GET /bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {  //子聚合
        "ageAvg": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}
```

返回结果：

```
{
  "took" : 16,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 31,
          "doc_count" : 61,
          "ageAvg" : {
            "value" : 28312.918032786885
          }
        },
        {
          "key" : 39,
          "doc_count" : 60,
          "ageAvg" : {
            "value" : 25269.583333333332
          }
        },
        {
          "key" : 26,
          "doc_count" : 59,
          "ageAvg" : {
            "value" : 23194.813559322032
          }
        },
...
```

【例子3】

查询出所有年龄分布，并且这些 年龄段中 性别为 M 的平均薪资 和 性别为 F 的平均薪资 以及 这个年龄段的总体平均薪资

```
GET /bank/_search
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "ageAgg": {
      "terms": {
        "field": "age",
        "size": 100
      },
      "aggs": {
        "genderAgg":{
          "terms": {
            "field": "gender.keyword",
            "size": 10
          },
          "aggs": {
            "balanceAvg": {
              "avg": {
                "field": "balance"
              }
            }
          }
        },
        "ageBlanace":{
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  },
  "size": 0
}
```

返回结果：

```
{
  "took" : 16,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1000,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  },
  "aggregations" : {
    "ageAgg" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : 31,
          "doc_count" : 61,
          "genderAgg" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "M",
                "doc_count" : 35,
                "balanceAvg" : {
                  "value" : 29565.628571428573
                }
              },
              {
                "key" : "F",
                "doc_count" : 26,
                "balanceAvg" : {
                  "value" : 26626.576923076922
                }
              }
            ]
          },
          "ageBlanace" : {
            "value" : 28312.918032786885
          }
        },
        {
          "key" : 39,
          "doc_count" : 60,
          "genderAgg" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "F",
                "doc_count" : 38,
                "balanceAvg" : {
                  "value" : 26348.684210526317
                }
              },
              {
                "key" : "M",
                "doc_count" : 22,
                "balanceAvg" : {
                  "value" : 23405.68181818182
                }
              }
            ]
          },
          "ageBlanace" : {
            "value" : 25269.583333333332
          }
        },
        {
          "key" : 26,
          "doc_count" : 59,
          "genderAgg" : {
            "doc_count_error_upper_bound" : 0,
            "sum_other_doc_count" : 0,
            "buckets" : [
              {
                "key" : "M",
                "doc_count" : 32,
                "balanceAvg" : {
                  "value" : 25094.78125
                }
              },
              {
                "key" : "F",
                "doc_count" : 27,
                "balanceAvg" : {
                  "value" : 20943.0
                }
              }
            ]
          },
          "ageBlanace" : {
            "value" : 23194.813559322032
          }
        },
...
```

 

# 十 分词器

一个 tokenizer（分词器）接收一个字符流，将之割为独立的 tokens（词元，通常是独立的单词），然后输出 tokens流。

例如，whitespace tokenizer遇到空白字符时分割文。它会将文本 "Quick brown fox!“ 分割为 [Quick, brown, fox]。该 tokenizer（分词器）还负责记录各个term（词条）的顺序或 position 位置（用于 phrase短语和 word proximity 词近邻查询），以及term（词条）所代表的原始word（单词）的 start（起始）和end（结束）的 character offsets（字符偏移量）（用于高亮显示搜索的内容）。

ElasticSearch 提供了很多内置的分词器，可以用来构建 custom analyzers（自定义分词器）

### 一、分词预览查询

```
POST _analyze
{
  "analyzer": "standard",
  "text": "Note however that storage is optimized based on the actual values that are stored"
}
```

返回结果

```
{
  "tokens" : [
    {
      "token" : "note",
      "start_offset" : 0,
      "end_offset" : 4,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "however",
      "start_offset" : 5,
      "end_offset" : 12,
      "type" : "<ALPHANUM>",
      "position" : 1
    },
    {
      "token" : "that",
      "start_offset" : 13,
      "end_offset" : 17,
      "type" : "<ALPHANUM>",
      "position" : 2
    },
    {
      "token" : "storage",
      "start_offset" : 18,
      "end_offset" : 25,
      "type" : "<ALPHANUM>",
      "position" : 3
    },
...
```

但是如果是中文场景下就会出现一些问题：

```
POST _analyze
{
  "analyzer": "standard",
  "text": "更改其他索引的字段的映射"
}
```

返回结果如下，可见将每一个汉字进行了分割，明显不符合实际情况。



### 二、安装 IK分词器

在es目录下的`plugins`目录下创建一个新文件夹，命名为`ik`，然后把上面的压缩包中的内容解压到该目录中。

把解压出来的内容放到es/plugins/ik中。之后，需要**重新启动es**。

![img](https://img-blog.csdnimg.cn/20210127163957145.png)

再次测试：

> ### 1、ik_max_word
>
> 会将文本做最细粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为“中华人民共和国、中华人民、中华、华人、人民共和国、人民、共和国、大会堂、大会、会堂等词语。
>
> ### 2、ik_smart
>
> 
> 会做最粗粒度的拆分，比如会将“中华人民共和国人民大会堂”拆分为中华人民共和国、人民大会堂。
>
> 测试两种分词模式的效果：
>
> 发送：post localhost:9200/_analyze
>
>
> 测试ik_max_word
> {“text”:“中华人民共和国人民大会堂”,“analyzer”:“ik_max_word” }
> 测试ik_smart
> {“text”:“中华人民共和国人民大会堂”,“analyzer”:“ik_smart” }

```
POST _analyze
{
  "analyzer": "ik_smart",
  "text": "你是列文虎克吗"
}
```

结果：

```
{
  "tokens" : [
    {
      "token" : "你",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "是",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "CN_CHAR",
      "position" : 1
    },
    {
      "token" : "列",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "CN_CHAR",
      "position" : 2
    },
    {
      "token" : "文虎",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "克",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "CN_CHAR",
      "position" : 4
    },
    {
      "token" : "吗",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "CN_CHAR",
      "position" : 5
    }
  ]
}
```

### 三、自定义分词

从上面的例子中可以看到 列文虎克 被拆开了，因为ik分词器依旧不支持部分内容，我们可以自定义分词词库

在elasticsearch-5.6.8\plugins\ik\config下

新增一个z_SelfAdd.dic文件，在里面加上新的单词，保存为UTF-8

然后在当前目录下的IKAnalyzer.cfg.xml配置文件中下加上<entry key="ext_dict">z_SelfAdd.dic</entry>

将刚才命名的文件加入

重启就生效了