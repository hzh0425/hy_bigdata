# Elasticsearch

Elasticsearch是一个基于[Apache Lucene(TM)](https://link.jianshu.com?t=https://lucene.apache.org/core/)的开源搜索引擎。无论在开源还是专有领域，Lucene可以被认为是迄今为止最先进、性能最好的、功能最全的搜索引擎库。
 但是，Lucene只是一个库。想要使用它，你必须使用Java来作为开发语言并将其直接集成到你的应用中，更糟糕的是，Lucene非常复杂，你需要深入了解检索的相关知识来理解它是如何工作的。
 Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API
 来隐藏Lucene的复杂性，从而让全文搜索变得简单。
 不过，Elasticsearch不仅仅是Lucene和全文搜索，我们还能这样去描述它：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据

# 安装

1. [Java](https://link.jianshu.com?t=https://www.java.com/)
2. [Elasticsearch](https://link.jianshu.com?t=https://www.elastic.co/downloads/elasticsearch)
3. Marvel(*可选*)
4. head 等其他插件(*可选*)

# 部分基础概念

#### 集群和节点

**节点(node)是一个运行着的Elasticsearch实例。
 **集群**(cluster)是一组具有相同cluster.name的节点集合，他们协同工作，共享数据并提供故障转移和扩展功能，\*当然一个节点也可以组成一个集群。*

#### term

term是一个被索引的精确的值。terms：foo, Foo, FOO 是不等价的。Terms可以使用term query查询。例如："我爱中国"被分词为,我/爱/中国 ,那么有3个term，分别是：我，爱，中国 。

#### analysis

把字符串转换为terms的过程。基于它使用可那些分词器，短语：FOO BAR, Foo-Bar, foo,bar有可能分词为terms：foo和bar。这些terms会被存储在索引中。全文索引查询（不是term query）“FoO:bAR”，同样也会被分词为terms：foo,bar，并且也匹配存在索引中的terms。分词过程（包括索引或搜索过程）使es可以执行全文查询。

#### cluster

集群由一个或多个节点组成，它们都使用同一个集群名。每个集群都有单个master节点，他是自动被集群选举的，如果当前master节点发生故障，可以用候选master节点代替它。

#### document

文档是一个json文档，它存储在es中。它就好像是关系数据库中table中的一行。每个文档都保存在索引中，并且属于一个type和有一个id字段。文档是一个json对象（在其他语言中是hash/hashmap/管关联数组）它包含0个或多个字段，或key-value对。被索引的原始的json文档会被保存在_source字段，当执行getting或searching操作时会默认返回该字段。

#### id

识别文档的id。文档的index/type/id必须是唯一的，如果没有提供id，将会自动生成id

#### field

文档包含一系列的字段，或者key-value 对。每个值都有可以是单个值（string，intger，date）或者是一个像数组的nested 结构或者对象。字段和关系数据库中的列类似。不要和文档type混淆。

#### index

索引就像关系数据库中的一个表，它包含一个mapping，mapping定义索引的字段，它由多个type组成。索引是一个逻辑命名空间，它映射到一个或多个主分片和0个或多个复制分片。

#### mapping

mapping就像关系数据库中的schema。每个索引都有一个mapping，它定义了index中的type，和一系列索引level的选项。mapping可以显示定义或者在索引文档时自动生成。

#### node

节点是一个运行中的es实例，它属于一个集群。可以在一个服务器启动多个节点用于测试，但通常都是一个服务器一个节点。在启动的时候节点会使用单播查找一个具有相同集群名并且已存在的集群，并尝试加入它。

#### primary shard

每个文档都保存在单个主分片中。当你索引一个文档，他首先会在主分片索引，然后告诉所有该主分片的复制分片。默认，索引有5个主分片，你可以根据你的文档数量指定更小或更多的主分片。当索引创建之后，你不能改变主分片的数量。

#### replica shard

每个主分片都有一个或多个复制分片。复制分片是主分片的副本，用于以下两个目的：

1. 提升容错能力，复制分片提升为主分片，如果主分片不可用。
2. 提升性能，get和search请求可以被主分片或者复制分片处理。默认，每个主分片都有一个复制分片，可以动态修改复制分片的数量。永远不要在相同的节点同时存放主分片和复制分片。

#### routing

当你索引一个文档，它被保存在单个主分片，并基于对routing的值进行hash计算映射到对应的分片。默认被hash的值来自文档id或者文档的父文档id（确保父文档和子文档存储在同一个分片）。这个值可以在索引期间指定routing参数或在mapping中使用routing字段修改。

#### shard

分片是单个lucene索引实例。ta是一个底层的“woker”单位，它由es自动管理。一个索引是一个逻辑的命名空间，它指向主分片和复制分片。而不是定义主分片和复制分片的数量，你从来都不需要直接引用主分片。相反，你的代码仅仅需要引用索引。es会在集群的所有节点中分发分片，并可以自动在节点之间移动分片，当节点出现故障时或者添加额外的节点。

#### source field

默认，你索引的JSON文档会被存储在_source字段，并会在所有的get和search请求中返回。这允许你直接从搜索结果中直接访问原始的对象，而不需要先查询id然后再获取文档。

#### text

text是普通非结构化的文本，例如段落。默认，text会被分词成terms，索引实际上储存的是terms。text 字段需要在索引期间进行分词，这样才可以进行全文搜索，查询字符串在搜索期间也需要被分词。fuzhi

#### type

type呈现文档的类型，例如 email，user，或weet。search API 可以根据type过滤文档。索引可以包含多个type，每个type包含一系列的字段。同一个索引不同type字段名相同的字段必须具有相同的mapping。

#### docvalues

它保存某一列的数据，并索引它，用于加快聚合和排序的速度。

#### fileddata

它保存某一列的数据，并索引它，用于加快聚合和排序的速度。和docvalues不一样的是，fielddata保存的是text类型的字段分词后的terms，而不是保存源字段数据。

#### shardcopies

分片副本集合，包含主分片和复制分片

#### segment

每个分片都分为多个segment存储。小的segment和定期合并成大的segment

#### master

master 节点用于执行轻量的集群操作，例如：创建删除索引，跟踪节点，决定分片分发到哪个节点等。每个集群同时只有一个master节点和0个或多个master候选节点。

# API / 如何使用

*这里仅限Python*
 [官方文档](https://link.jianshu.com?t=http://elasticsearch-py.readthedocs.io/en/master/api.html)
 [一个简单的Demo](https://link.jianshu.com?t=http://www.cnblogs.com/letong/p/4749234.html)

# 基于HTTP协议，以JSON为数据交互格式的RESTful API

其他所有程序语言都可以使用**RESTful API**，通过9200端口的与Elasticsearch进行通信，你可以使用你喜欢的WEB客户端，事实上，如你所见，你甚至可以通过curl命令与Elasticsearch通信。

> **NOTE**
>  Elasticsearch官方提供了多种程序语言的客户端——Groovy，Javascript， .NET，PHP，Perl，Python，以及 Ruby——还有很多由社区提供的客户端和插件，所有这些可以在[文档](https://link.jianshu.com?t=http://www.elasticsearch.org/guide/)中找到。

向Elasticsearch发出的请求的组成部分与其它普通的HTTP请求是一样的：



```linux
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

- **VERB** HTTP方法：GET, POST, PUT, HEAD, DELETE
- **PROTOCOL** http或者https协议（只有在Elasticsearch前面有https代理的时候可用）
- **HOST** Elasticsearch集群中的任何一个节点的主机名，如果是在本地的节点，那么就叫localhost
- **PORT** Elasticsearch HTTP服务所在的端口，默认为9200
- **PATH** API路径（例如`_count`将返回集群中文档的数量），PATH可以包含多个组件，例如_cluster/stats或者_nodes/stats/jvm
- **QUERY_STRING** 一些可选的查询请求参数，例如`?pretty`参数将使请求返回更加美观易读的JSON数据
- **BODY** 一个JSON格式的请求主体（如果请求需要的话）

举例说明，为了计算集群中的文档数量，我们可以这样做：+



```json
curl -XGET 'http://localhost:9200/_count?pretty' -d '
{
    "query": {
        "match_all": {}
    }
}
'
```

Elasticsearch返回一个类似200 OK的HTTP状态码和JSON格式的响应主体（除了HEAD请求）。上面的请求会得到如下的JSON格式的响应主体：



```json
{
    "count" : 0,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

# 面向文档

Elasticsearch是**面向文档(document oriented)**的，这意味着它可以存储整个对象或**文档(document)**。然而它不仅仅是存储，还会**索引(index)**每个文档的内容使之可以被搜索。在Elasticsearch中，你可以对文档（而非成行成列的数据）进行索引、搜索、排序、过滤。这种理解数据的方式与以往完全不同，这也是Elasticsearch能够执行复杂的全文搜索的原因之一。

- JSON
   ELasticsearch使用**Javascript对象符号(JavaScript Object Notation)**，也就是[**JSON**](https://link.jianshu.com?t=http://en.wikipedia.org/wiki/Json)，作为文档序列化格式。JSON现在已经被大多语言所支持，而且已经成为NoSQL领域的标准格式。它简洁、简单且容易阅读。
   以下使用JSON文档来表示一个用户对象：
   { "email": "[john@smith.com](https://link.jianshu.com?t=mailto:john@smith.com)", "first_name": "John", "last_name": "Smith", "info": { "bio": "Eco-warrior and defender of the weak", "age": 25, "interests": [ "dolphins", "whales" ] }, "join_date": "2014/05/01"}

尽管原始的user
 对象很复杂，但它的结构和对象的含义已经被完整的体现在JSON中了，在Elasticsearch中将对象转化为JSON并做索引要比在表结构中做相同的事情简单的多。

> **NOTE**
>  尽管几乎所有的语言都有相应的模块用于将任意数据结构转换为JSON，但每种语言处理细节不同。具体请查看“serialization” or “marshalling”两个用于处理JSON的模块。[Elasticsearch官方客户端](https://link.jianshu.com?t=http://www.elasticsearch.org/guide)会自动为你序列化和反序列化JSON。

### Demo

[一个简单的关于员工的示例](https://link.jianshu.com?t=https://es.xiaoleilu.com/010_Intro/25_Tutorial_Indexing.html)

在Elasticsearch中，文档归属于一种类型(type),而这些类型存在于索引(index)中，我们可以画一些简单的对比图来类比传统关系型数据库：

> Relational DB -> Databases -> Tables -> Rows -> Columns
>  Elasticsearch -> Indices   -> Types  -> Documents -> Fields

Elasticsearch集群可以包含多个**索引**(indices)（数据库），每一个索引可以包含多个**类型**(types)（表），每一个类型包含多个**文档**(documents)（行），然后每个文档包含多个**字段**(Fields)（列）。

> **NOTE**
>  「索引」含义的区分
>  你可能已经注意到索引(index)这个词在Elasticsearch中有着不同的含义，所以有必要在此做一下区分:
>  索引（名词） 如上文所述，一个索引(index)就像是传统关系数据库中的数据库，它是相关文档存储的地方，index的复数是indices 或indexes。
>  索引（动词） 「索引一个文档」表示把一个文档存储到索引（名词）里，以便它可以被检索或者查询。这很像SQL中的INSERT关键字，差别是，如果文档已经存在，新的文档将覆盖旧的文档。
>  倒排索引 传统数据库为特定列增加一个索引，例如B-Tree索引来加速检索。Elasticsearch和Lucene使用一种叫做倒排索引(inverted index)的数据结构来达到相同目的。

### 索引

默认情况下，文档中的所有字段都会被**索引**（拥有一个倒排索引），只有这样他们才是可被搜索的。
 我们将会在[倒排索引](https://link.jianshu.com?t=https://es.xiaoleilu.com/052_Mapping_Analysis/35_Inverted_index.html)章节中更详细的讨论。
 所以为了创建员工目录，我们将进行如下操作：
 为每个员工的**文档(document)**建立索引，每个文档包含了相应员工的所有信息。
 每个文档的类型为`employee`
 employee类型归属于索引`megacorp`
 megacorp索引存储在`Elasticsearch`集群中。

实际上这些都是很容易的（尽管看起来有许多步骤）。我们能通过一个命令执行完成的操作：



```json
PUT /megacorp/employee/1
{ 
    "first_name" : "John",
    "last_name" : "Smith", 
    "age" : 25, 
    "about" : "I love to go rock climbing", 
    "interests": [ "sports", "music" ]
}
```

| 名字     |     说明     |
| -------- | :----------: |
| megacorp |    索引名    |
| employee |    类型名    |
| 1        | 这个员工的ID |

我们看到path:/megacorp/employee/1
 包含三部分信息：

| 名字     |     说明     |
| -------- | :----------: |
| megacorp |    索引名    |
| employee |    类型名    |
| 1        | 这个员工的ID |

请求实体（JSON文档），包含了这个员工的所有信息。他的名字叫“John Smith”，25岁，喜欢攀岩。
 很简单吧！它不需要你做额外的管理操作，比如创建索引或者定义每个字段的数据类型。我们能够直接索引文档，**Elasticsearch已经内置所有的缺省设置，所有管理操作都是透明的。**
 接下来，让我们在目录中加入更多员工信息：



```json
PUT /megacorp/employee/2
{ 
    "first_name" : "Jane", 
    "last_name" : "Smith", 
    "age" : 32, 
    "about" : "I like to collect rock albums", 
    "interests": [ "music" ]
}
```



```json
PUT /megacorp/employee/3
{ 
    "first_name" : "Douglas", 
    "last_name" : "Fir", 
    "age" : 35, 
    "about": "I like to build cabinets", 
    "interests": [ "forestry" ]
}
```

### 检索文档

现在Elasticsearch中已经存储了一些数据，我们可以根据业务需求开始工作了。

##### 第一个需求是能够检索单个员工的信息。

这对于Elasticsearch来说非常简单。我们只要执行HTTP GET请求并指出文档的“地址”——索引、类型和ID既可。根据这三部分信息，我们就可以返回原始JSON文档：



```undefined
GET /megacorp/employee/1
```

响应的内容中包含一些文档的元信息，John Smith的原始JSON文档包含在_source字段中。



```json
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```

我们通过HTTP方法**GET**来检索文档，同样的，我们可以使用**DELETE**方法删除文档，使用**HEAD**方法检查某文档是否存在。**如果想更新已存在的文档，我们只需再PUT一次。**

##### 简单搜索

GET请求非常简单——你能轻松获取你想要的文档。让我们来进一步尝试一些东西，比如简单的搜索！
 我们尝试一个最简单的搜索全部员工的请求：



```undefined
GET /megacorp/employee/_search
```

你可以看到我们依然使用`megacorp`索引和`employee`类型，但是我们在结尾使用**关键字_search**来取代原来的文档ID。响应内容的hits数组中包含了我们所有的三个文档。默认情况下搜索会返回前10个结果。



```json
{
   "took":      6,
   "timed_out": false,
   "_shards": { ... },
   "hits": {
      "total":      3,
      "max_score":  1,
      "hits": [
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "3",
            "_score":         1,
            "_source": {
               "first_name":  "Douglas",
               "last_name":   "Fir",
               "age":         35,
               "about":       "I like to build cabinets",
               "interests": [ "forestry" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "1",
            "_score":         1,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            "_index":         "megacorp",
            "_type":          "employee",
            "_id":            "2",
            "_score":         1,
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

注意：
 **响应内容不仅会告诉我们哪些文档被匹配到，而且这些文档内容完整的被包含在其中—我们在给用户展示搜索结果时需要用到的所有信息都有了。**

##### 接下来，让我们搜索姓氏中包含“Smith”的员工。

要做到这一点，我们将在命令行中使用轻量级的搜索方法。这种方法常被称作查询字符串(query string)搜索，因为我们像传递URL参数一样去传递查询语句：



```undefined
GET /megacorp/employee/_search?q=last_name:Smith
```

我们在请求中依旧使用_search关键字，然后**将查询语句传递给参数q=**。这样就可以得到所有姓氏为Smith的结果：



```json
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

### 使用DSL语句查询

查询字符串搜索便于通过命令行完成特定(ad hoc)的搜索，但是它也有局限性（参阅简单搜索章节）。Elasticsearch提供丰富且灵活的查询语言叫做**DSL查询(Query DSL)**,它允许你构建更加复杂、强大的查询。
 DSL(Domain Specific Language特定领域语言)以JSON请求体的形式出现。我们可以这样表示之前关于“Smith”的查询:



```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

这会返回与之前查询相同的结果。你可以看到有些东西改变了，我们不再使用查询字符串(query string)做为参数，而是使用请求体代替。这个请求体使用JSON表示，其中使用了match语句（查询类型之一，具体我们以后会学到）。
 更复杂的搜索

我们让搜索稍微再变的复杂一些。我们依旧想要找到姓氏为“Smith”的员工，但是我们只想得到年龄大于30岁的员工。我们的语句将添加过滤器(filter),它使得我们高效率的执行一个结构化搜索：



```undefined
GET /megacorp/employee/_search
```



```json
{
    "query" : {
        "filtered" : {
            "filter" : {
                "range" : {
                    "age" : { "gt" : 30 } <1>
                }
            },
            "query" : {
                "match" : {
                    "last_name" : "smith" <2>
                }
            }
        }
    }
}
```

*<1> 这部分查询属于区间过滤器(range filter),它用于查找所有年龄大于30岁的数据——gt为"greater than"的缩写。*
 *<2> 这部分查询与之前的match语句(query)一致。*

现在不要担心语法太多，我们将会在以后详细的讨论。你只要知道我们添加了一个**过滤器(filter)**用于执行区间搜索，然后重复利用了之前的**match**语句。现在我们的搜索结果只显示了一个32岁且名字是“Jane Smith”的员工：



```json
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.30685282,
      "hits": [
         {
            ...
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

### 全文搜索 (模糊查找)

> 到目前为止搜索都很简单：搜索特定的名字，通过年龄筛选。让我们尝试一种更高级的搜索，全文搜索——一种传统数据库很难实现的功能。

我们将会搜索所有喜欢“rock climbing”的员工：
 在这里使用了 **about**



```json
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
```

*你可以看到我们使用了之前的***match***查询，从***about字段中搜索***"rock climbing"，我们得到了两个匹配文档：*



```json
{
   ...
   "hits": {
      "total":      2,
      "max_score":  0.16273327,
      "hits": [
         {
            ...
            "_score":         0.16273327, <1>
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         },
         {
            ...
            "_score":         0.016878016, <2>
            "_source": {
               "first_name":  "Jane",
               "last_name":   "Smith",
               "age":         32,
               "about":       "I like to collect rock albums",
               "interests": [ "music" ]
            }
         }
      ]
   }
}
```

*<1><2> 结果相关性评分。*

默认情况下，Elasticsearch根据结果相关性评分来对结果集进行排序，所谓的「结果相关性评分」就是文档与查询条件的匹配程度。很显然，排名第一的John Smith的about字段明确的写到“rock climbing”。
 但是为什么Jane Smith也会出现在结果里呢？原因是“rock”在她的abuot字段中被提及了。因为只有“rock”被提及而“climbing”没有，所以她的_score要低于John。
 这个例子很好的解释了Elasticsearch如何在各种文本字段中进行全文搜索，并且返回相关性最大的结果集。相关性(relevance)的概念在Elasticsearch中非常重要，而这个概念在传统关系型数据库中是不可想象的，因为传统数据库对记录的查询只有匹配或者不匹配。

### 短语搜索(精确匹配)

目前我们可以在字段中搜索单独的一个词，这挺好的，但是有时候你想要确切的匹配若干个单词或者短语(phrases)。例如我们想要查询同时包含"rock"和"climbing"（并且是相邻的）的员工记录。
 要做到这个，我们只要将**match**查询变更为**match_phrase**查询即可:



```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
```

毫无疑问，该查询返回John Smith的文档：



```json
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            }
         }
      ]
   }
}
```

### 高亮我们的搜索

很多应用喜欢从每个搜索结果中高亮(highlight)匹配到的关键字，这样用户可以知道为什么这些文档和查询相匹配。在Elasticsearch中高亮片段是非常容易的。
 让我们在之前的语句上增加highlight参数：



```json
GET /megacorp/employee/_search
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
```

当我们运行这个语句时，会命中与之前相同的结果，但是在返回结果中会有一个新的部分叫做**highlight**，这里包含了来自about字段中的文本，并且用**<em></em>来标识匹配到的单词**。



```json
{
   ...
   "hits": {
      "total":      1,
      "max_score":  0.23013961,
      "hits": [
         {
            ...
            "_score":         0.23013961,
            "_source": {
               "first_name":  "John",
               "last_name":   "Smith",
               "age":         25,
               "about":       "I love to go rock climbing",
               "interests": [ "sports", "music" ]
            },
            "highlight": {
               "about": [
                  "I love to go <em>rock</em> <em>climbing</em>" <1>
               ]
            }
         }
      ]
   }
}
```

*<1> 原有文本中高亮的片段*

# 分析

最后，我们还有一个需求需要完成：允许管理者在职员目录中进行一些分析。 Elasticsearch有一个功能叫做聚合(aggregations)，它允许你在数据上生成复杂的分析统计。它很像SQL中的`GROUP BY`但是功能更强大。

### 举个例子，让我们找到所有职员中最大的共同点（兴趣爱好）是什么：



```json
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```

暂时先忽略语法只看查询结果：



```json
{
   ...
   "hits": { ... },
   "aggregations": {
      "all_interests": {
         "buckets": [
            {
               "key":       "music",
               "doc_count": 2
            },
            {
               "key":       "forestry",
               "doc_count": 1
            },
            {
               "key":       "sports",
               "doc_count": 1
            }
         ]
      }
   }
}
```

我们可以看到两个职员对音乐有兴趣，一个喜欢林学，一个喜欢运动。**这些数据并没有被预先计算好，它们是实时的从匹配查询语句的文档中动态计算生成的。**如果我们想知道所有姓"Smith"的人最大的共同点（兴趣爱好），我们只需要增加合适的语句既可：



```json
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```

all_interests聚合已经变成只包含和查询语句相匹配的文档了：



```json
  ...
  "all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2
        },
        {
           "key": "sports",
           "doc_count": 1
        }
     ]
  }
```

聚合也允许**分级汇总**。

##### 例如，让我们统计每种兴趣下职员的平均年龄：



```json
GET /megacorp/employee/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```

虽然这次返回的聚合结果有些复杂，但任然很容易理解：



```json
  ...
  "all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2,
           "avg_age": {
              "value": 28.5
           }
        },
        {
           "key": "forestry",
           "doc_count": 1,
           "avg_age": {
              "value": 35
           }
        },
        {
           "key": "sports",
           "doc_count": 1,
           "avg_age": {
              "value": 25
           }
        }
     ]
  }
```

该聚合结果比之前的聚合结果要更加丰富。我们依然得到了兴趣以及数量（指具有该兴趣的员工人数）的列表，但是现在每个兴趣额外拥有avg_age字段来显示具有该兴趣员工的平均年龄。
 即使你还不理解语法，但你也可以大概感觉到通过这个特性可以完成相当复杂的聚合工作，你可以处理任何类型的数据。

# 分布式的特性

在章节的开始我们提到Elasticsearch可以扩展到上百（甚至上千）的服务器来处理PB级的数据。然而我们的教程只是给出了一些使用Elasticsearch的例子，并未涉及相关机制。Elasticsearch为分布式而生，而且它的设计隐藏了分布式本身的复杂性。
 Elasticsearch在分布式概念上做了很大程度上的透明化，在教程中你不需要知道任何关于分布式系统、分片、集群发现或者其他大量的分布式概念。所有的教程你既可以运行在你的笔记本上，也可以运行在拥有100个节点的集群上，其工作方式是一样的。
 Elasticsearch致力于隐藏分布式系统的复杂性。以下这些操作都是在底层自动完成的：
 将你的文档分区到不同的容器或者分片(shards)中，它们可以存在于一个或多个节点中。

- 将分片均匀的分配到各个节点，对索引和搜索做负载均衡。
- 冗余每一个分片，防止硬件故障造成的数据丢失。
- 将集群中任意一个节点上的请求路由到相应数据所在的节点。
- 无论是增加节点，还是移除节点，分片都可以做到无缝的扩展和迁移。

# 集群

Elasticsearch集群具有非常优秀的特性

- 负载均衡
- 数据备份
- 集群监控，获取节点状态
- 故障转移
- 横向扩展
- 分布式查询

一个**节点(node)**就是一个Elasticsearch实例，而一个**集群(cluster)**由一个或多个节点组成，它们具有相同的`cluster.name`，它们协同工作，分享数据和负载。当加入新的节点或者删除一个节点时，集群就会感知到并平衡数据。

集群中一个节点会被选举为**主节点(master)**,它将临时管理集群级别的一些变更，例如新建或删除索引、增加或移除节点等。主节点不参与文档级别的变更或搜索，这意味着在流量增长的时候，该主节点不会成为集群的瓶颈。任何节点都可以成为主节点。我们例子中的集群只有一个节点，所以它会充当主节点的角色。
 做为用户，我们能够与集群中的任何节点通信，包括主节点。每一个节点都知道文档存在于哪个节点上，它们可以转发请求到相应的节点上。我们访问的节点负责收集各节点返回的数据，最后一起返回给客户端。这一切都由Elasticsearch处理。

# 数据

程序中大多的实体或对象能够被序列化为包含键值对的JSON对象，**键(key)**是**字段(field)**或**属性(property)**的名字，**值(value)**可以是字符串、数字、布尔类型、另一个对象、值数组或者其他特殊类型，比如表示日期的字符串或者表示地理位置的对象。



```json
{
    "name":         "John Smith",
    "age":          42,
    "confirmed":    true,
    "join_date":    "2014-06-01",
    "home": {
        "lat":      51.5,
        "lon":      0.1
    },
    "accounts": [
        {
            "type": "facebook",
            "id":   "johnsmith"
        },
        {
            "type": "twitter",
            "id":   "johnsmith"
        }
    ]
}
```

通常，我们可以认为**对象(object)**和**文档(document)**是等价相通的。不过，他们还是有所差别：**对象(Object)**是一个**JSON结构体**——类似于哈希、hashmap、字典或者关联数组；**对象(Object)中还可能包含其他对象(Object)**。 在Elasticsearch中，**文档(document)**这个术语有着特殊含义。它特指最**顶层结构或者根对象(root object)序列化成的JSON数据**（以唯一ID标识并存储于Elasticsearch中）。

### 文档元数据

一个文档不只有数据。它还包含了**元数据(metadata)**——关于文档的信息。三个必须的元数据节点是

| 节点   |        说明        |
| ------ | :----------------: |
| _index |   文档存储的地方   |
| _type  | 文档代表的对象的类 |
| _id    |   文档的唯一标识   |

##### _index

> 索引(index)类似于关系型数据库里的“数据库”——它是我们存储和索引关联数据的地方。

提示：
 事实上，我们的数据被存储和索引在分片(shards)中，索引只是一个把一个或多个分片分组在一起的逻辑空间。然而，这只是一些内部细节——我们的程序完全不用关心分片。对于我们的程序而言，文档存储在索引(index)中。剩下的细节由Elasticsearch关心既可。
 我们将会在《索引管理》章节中探讨如何创建并管理索引，但现在，我们将让Elasticsearch为我们创建索引。我们唯一需要做的仅仅是选择一个索引名。**这个名字必须是全部小写，不能以下划线开头，不能包含逗号**。让我们使用website做为索引名

##### _type

> 在应用中，我们使用对象表示一些“事物”，例如一个用户、一篇博客、一个评论，或者一封邮件。每个对象都属于一个类(class)，这个类定义了属性或与对象关联的数据。user类的对象可能包含姓名、性别、年龄和Email地址。

在关系型数据库中，我们经常将相同类的对象存储在一个表里，因为它们有着相同的结构。同理，在Elasticsearch中，我们使用**相同类型(type)的文档表示相同的“事物”**，**因为他们的数据结构也是相同的**。
 每个类型(type)都有自己的映射(mapping)或者结构定义，就像传统数据库表中的列一样。所有类型下的文档被存储在同一个索引下，但是类型的映射(mapping)会告诉Elasticsearch不同的文档如何被索引。 我们将会在《映射》章节探讨如何定义和管理映射，但是现在我们将依赖Elasticsearch去自动处理数据结构。
 **_type的名字可以是大写或小写，不能包含下划线或逗号。我们将使用blog做为类型名。**

##### _id

> id仅仅是一个字符串，它与_index和_type组合时，就可以在Elasticsearch中唯一标识一个文档。当创建一个文档，**你可以自定义_id，也可以让Elasticsearch帮你自动生成。**

##### 其它元数据

还有一些其它的元数据，我们将在《映射》章节探讨。使用上面提到的元素，我们已经可以在Elasticsearch中存储文档并通过ID检索——换言说，把Elasticsearch做为文档存储器使用了。

### 索引一个文档

文档通过**index API**被索引——使数据可以被存储和搜索。但是首先我们需要决定文档所在。正如我们讨论的，文档通过其`_index`、`_type`、`_id`唯一确定。们可以自己提供一个_id，或者也使用index API 为我们生成一个。

### 使用自己的ID

如果你的文档有自然的标识符（例如user_account
 字段或者其他值表示文档），你就可以提供自己的_id
 ，使用这种形式的`index API`：



```json
PUT /{index}/{type}/{id}{ "field": "value", ...}
```

例如我们的索引叫做“website”，类型叫做“blog”，我们选择的ID是“123”，那么这个索引请求就像这样：



```json
PUT /website/blog/123{ "title": "My first blog entry", "text": "Just trying this out...", "date": "2014/01/01"}
```

Elasticsearch的响应：



```json
{ "_index": "website", "_type": "blog", "_id": "123", "_version": 1, "created": true}
```

响应指出请求的索引已经被成功创建，这个索引中包含_index、_type和_id元数据，以及一个新元素：`_version`

- _version
   Elasticsearch中每个文档都有版本号，每当文档变化（包括删除）都会使_version
   增加。在《版本控制》章节中我们将探讨如何使用_version
   号确保你程序的一部分不会覆盖掉另一部分所做的更改。

### 自增ID

如果我们的数据没有自然ID，我们可以让Elasticsearch自动为我们生成。请求结构发生了变化：`PUT方法`——`“在这个URL中存储文档”`变成了`POST方法`——`"在这个类型下存储文档"`。*（译者注：原来是把文档存储到某个ID对应的空间，现在是把这个文档添加到某个`_type`下）。*
 URL现在只包含`_index`和`_type`两个字段：



```json
POST /website/blog/{ "title": "My second blog entry", "text": "Still trying this out...", "date": "2014/01/01"}
```

响应内容与刚才类似，**只有_id字段变成了自动生成的值**：



```json
{ "_index": "website", "_type": "blog", "_id": "wM0OSFhDQXGZAWDf0-drSA", "_version": 1, "created": true}
```

自动生成的ID有22个字符长，URL-safe, Base64-encoded string universally unique identifiers, 或者叫 [UUIDs](https://link.jianshu.com?t=http://en.wikipedia.org/wiki/Uuid)。

### 检索文档

想要从Elasticsearch中获取文档，我们使用同样的`_index`、`_type`、`_id`，但是HTTP方法改为`GET`：



```undefined
GET /website/blog/123?pretty
```

响应包含了现在熟悉的元数据节点，增加了_source字段，它包含了在创建索引时我们发送给Elasticsearch的原始文档。



```json
{
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "title": "My first blog entry",
      "text":  "Just trying this out...",
      "date":  "2014/01/01"
  }
}
```

> ##### pretty

在任意的查询字符串中增加pretty参数，类似于上面的例子。会让Elasticsearch**美化输出**(pretty-print)JSON响应以便更加容易阅读。**_source字段不会被美化，它的样子与我们输入的一致。**

GET请求返回的响应内容包括{"found": true}。这意味着文档已经找到。**如果我们请求一个不存在的文档，依旧会得到一个JSON，不过found值变成了false。**
 此外，**HTTP响应状态码也会变成'404 Not Found'代替'200 OK'**。我们可以在curl后加-i参数得到响应头：



```cpp
curl -i -XGET http://localhost:9200/website/blog/124?pretty
```

现在响应类似于这样：



```json
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=UTF-8
Content-Length: 83

{
  "_index" : "website",
  "_type" :  "blog",
  "_id" :    "124",
  "found" :  false
}
```

### 检索文档的一部分

通常，GET请求将返回文档的全部，存储在_source参数中。但是可能你感兴趣的字段只是title。请求个别字段可以使用_source参数。多个字段可以使用逗号分隔：



```undefined
GET /website/blog/123?_source=title,text
```

_source字段现在只包含我们请求的字段，而且过滤了date字段：



```json
{
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 1,
  "exists" :   true,
  "_source" : {
      "title": "My first blog entry" ,
      "text":  "Just trying this out..."
  }
}
```

或者你只想得到_source字段而不要其他的元数据，你可以这样请求：



```undefined
GET /website/blog/123/_source
```

它仅仅返回:



```json
{
   "title": "My first blog entry",
   "text":  "Just trying this out...",
   "date":  "2014/01/01"
}
```

### 创建一个新文档

##### 当索引一个文档，我们如何确定是完全创建了一个新的还是覆盖了一个已经存在的呢？

请记住`_index`、`_type`、`_id`三者唯一确定一个文档。所以要想保证文档是新加入的，

- 最简单的方式是使用POST方法让Elasticsearch自动生成唯一_id：



```undefined
POST /website/blog/
{ ... }
```

然而，如果想使用自定义的_id，我们必须告诉Elasticsearch应该在_index、_type、_id三者都不同时才接受请求。为了做到这点有两种方法，它们其实做的是同一件事情。你可以选择适合自己的方式：

- 第一种方法使用op_type查询参数：



```undefined
PUT /website/blog/123?op_type=create
{ ... }
```

- 或者第二种方法是在URL后加/_create做为端点：



```undefined
PUT /website/blog/123/_create
{ ... }
```

如果请求成功的创建了一个新文档，Elasticsearch将返回正常的元数据且响应状态码是**201 Created**。
 另一方面，如果包含相同的_index、_type和_id的文档已经存在，Elasticsearch将返回**409 Conflict**响应状态码，错误信息类似如下：



```json
{
  "error" : "DocumentAlreadyExistsException[[website][4] [blog][123]:
             document already exists]",
  "status" : 409
}
```

### 删除文档

删除文档的语法模式与之前基本一致，只不过要使用DELETE方法：



```undefined
DELETE /website/blog/123
```

如果文档被找到，Elasticsearch将返回**200 OK**状态码和以下响应体。注意_version数字已经增加了。



```json
{
  "found" :    true,
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 3
}
```

如果文档未找到，我们将得到一个**404 Not Found**状态码，响应体是这样的：



```json
{
  "found" :    false,
  "_index" :   "website",
  "_type" :    "blog",
  "_id" :      "123",
  "_version" : 4
}
```

尽管文档不存在——"found"的值是false——_version依旧增加了。这是内部记录的一部分，**它确保在多节点间不同操作可以有正确的顺序。**

### 处理冲突

当使用`index API`更新文档的时候，我们读取原始文档，做修改，然后将**整个文档(whole document)**一次性重新索引。最近的索引请求会生效——Elasticsearch中只存储最后被索引的任何文档。如果其他人同时也修改了这个文档，他们的修改将会丢失。
 很多时候，这并不是一个问题。或许我们主要的数据存储在关系型数据库中，然后拷贝数据到Elasticsearch中只是为了可以用于搜索。或许两个人同时修改文档的机会很少。亦或者偶尔的修改丢失对于我们的工作来说并无大碍。
 但有时丢失修改是一个**很严重**的问题。想象一下我们使用Elasticsearch存储大量在线商店的库存信息。每当销售一个商品，Elasticsearch中的库存就要减一。
 一天，老板决定做一个促销。瞬间，我们每秒就销售了几个商品。想象两个同时运行的web进程，两者同时处理一件商品的订单：

![img](https:////upload-images.jianshu.io/upload_images/5617720-a52335606600a377.png?imageMogr2/auto-orient/strip|imageView2/2/w/750/format/webp)

img-data-lww


 web_1让stock_count失效是因为web_2没有察觉到stock_count
 的拷贝已经过期（译者注：web_1取数据，减一后更新了stock_count。可惜在web_1更新stock_count前它就拿到了数据，这个数据已经是过期的了，当web_2再回来更新stock_count时这个数字就是错的。这样就会造成看似卖了一件东西，其实是卖了两件，这个应该属于幻读。）。结果是我们认为自己确实还有更多的商品，最终顾客会因为销售给他们没有的东西而失望。
 变化越是频繁，或读取和更新间的时间越长，越容易丢失我们的更改。
**在数据库中，有两种通用的方法确保在并发更新时修改不丢失：**



##### 悲观并发控制（Pessimistic concurrency control）

这在关系型数据库中被广泛的使用，假设冲突的更改经常发生，为了解决冲突我们把访问区块化。典型的例子是在读一行数据前锁定这行，然后确保只有加锁的那个线程可以修改这行数据。

##### 乐观并发控制（Optimistic concurrency control）：

被Elasticsearch使用，假设冲突不经常发生，也不区块化访问，然而，如果在读写过程中数据发生了变化，更新操作将失败。这时候由程序决定在失败后如何解决冲突。实际情况中，可以重新尝试更新，刷新数据（重新读取）或者直接反馈给用户。

### 乐观并发控制

Elasticsearch是分布式的。当文档被创建、更新或删除，文档的新版本会被复制到集群的其它节点。**Elasticsearch即是同步的又是异步的**，意思是这些复制请求都是**平行发送**的，并**无序(out of sequence)**的到达目的地。这就需要一种方法确保老版本的文档永远不会覆盖新的版本。
 上文我们提到index、get、delete
 请求时，我们指出每个文档都有一个_version号码，这个号码在文档被改变时加一。Elasticsearch使用这个_version
 保证所有修改都被正确排序。当一个旧版本出现在新版本之后，它会被简单的忽略。
 我们利用_version的这一优点确保数据不会因为修改冲突而丢失。我们可以指定文档的version
 来做想要的更改。如果那个版本号不是现在的，我们的请求就失败了。
 让我们创建一个新的博文：



```json
PUT /website/blog/1/_create{ "title": "My first blog entry", "text": "Just trying this out..."}
```

响应体告诉我们这是一个新建的文档，它的_version
 是1
 。现在假设我们要编辑这个文档：把数据加载到web表单中，修改，然后保存成新版本。
 首先我们检索文档：



```undefined
GET /website/blog/1
```

响应体包含相同的_version是1



```json
{ "_index" : "website", "_type" : "blog", "_id" : "1", "_version" : 1, "found" : true, "_source" : { "title": "My first blog entry", "text": "Just trying this out..." }}
```

现在，当我们通过重新索引文档保存修改时，我们这样指定了version
 参数：



```json
PUT /website/blog/1?version=1 <1>{ "title": "My first blog entry", "text": "Starting to get the hang of this..."}
```

*<1> 我们只希望文档的_version是1时更新才生效。*

请求成功，响应体告诉我们_version已经增加到2



```json
{ "_index": "website", "_type": "blog", "_id": "1", "_version": 2 "created": false}
```

然而，如果我们重新运行相同的索引请求，依旧指定version=1，Elasticsearch将返回**409 Conflict**状态的HTTP响应。响应体类似这样：



```json
{ "error" : "VersionConflictEngineException[[website][2] [blog][1]: version conflict, current [2], provided [1]]", "status" : 409}
```

**这告诉我们当前_version是2，但是我们指定想要更新的版本是1**
 。
 我们需要做什么取决于程序的需求。我们可以告知用户其他人修改了文档，你应该在保存前再看一下。而对于上文提到的商品stock_count，我们需要重新检索最新文档然后申请新的更改操作。所有更新和删除文档的请求都接受version参数，它可以允许在你的代码中增加乐观锁控制。

### 使用外部版本控制系统

一种常见的结构是使用一些**其他的数据库做为主数据库**，然后**使用Elasticsearch搜索数据**，这意味着所有主数据库发生变化，就要将其拷贝到Elasticsearch中。如果有多个进程负责这些数据的同步，就会遇到上面提到的并发问题。
 如果主数据库有版本字段——或一些类似于timestamp等可以用于版本控制的字段——是你就可以在Elasticsearch的查询字符串后面添加version_type=external来使用这些版本号。版本号必须是整数，大于零小于9.2e+18——Java中的正的long。
 外部版本号与之前说的内部版本号在处理的时候有些不同。它不再检查_version
 是否与请求中指定的**一致**，而是检查是否**小于**指定的版本。如果请求成功，外部版本号就会被存储到_version
 中。
 外部版本号不仅在索引和删除请求中指定，也可以在**创建(create)**新文档中指定。
 例如，创建一个包含外部版本号5的新博客，我们可以这样做：



```json
PUT /website/blog/2?version=5&version_type=external
{ "title": "My first external blog entry", "text": "Starting to get the hang of this..."}
```

在响应中，我们能看到当前的_version
 号码是5



```json
{ "_index": "website", "_type": "blog", "_id": "2", "_version": 5, "created": true}
```

现在我们更新这个文档，指定一个新version
 号码为10



```json
PUT /website/blog/2?version=10&version_type=external
{ "title": "My first external blog entry", "text": "This is a piece of cake..."}
```

请求成功的设置了当前_version
 为10



```json
{ "_index": "website", "_type": "blog", "_id": "2", "_version": 10, "created": false}
```

如果你重新运行这个请求，就会返回一个像之前一样的冲突错误，因为指定的外部版本号不大于当前在Elasticsearch中的版本。

### 检索多个文档

像Elasticsearch一样，检索多个文档依旧非常快。合并多个请求可以避免每个请求单独的网络开销。如果你需要从Elasticsearch中检索多个文档，相对于一个一个的检索，更快的方式是在一个请求中使用multi-get或者mget API。+

mget API参数是一个docs数组，数组的每个节点定义一个文档的_index、_type、_id元数据。如果你只想检索一个或几个确定的字段，也可以定义一个_source参数：



```json
POST /_mget
{
   "docs" : [
      {
         "_index" : "website",
         "_type" :  "blog",
         "_id" :    2
      },
      {
         "_index" : "website",
         "_type" :  "pageviews",
         "_id" :    1,
         "_source": "views"
      }
   ]
}
```

响应体也包含一个docs数组，每个文档还包含一个响应，它们按照请求定义的顺序排列。每个这样的响应与单独使用get request响应体相同：



```json
{
   "docs" : [
      {
         "_index" :   "website",
         "_id" :      "2",
         "_type" :    "blog",
         "found" :    true,
         "_source" : {
            "text" :  "This is a piece of cake...",
            "title" : "My first external blog entry"
         },
         "_version" : 10
      },
      {
         "_index" :   "website",
         "_id" :      "1",
         "_type" :    "pageviews",
         "found" :    true,
         "_version" : 2,
         "_source" : {
            "views" : 2
         }
      }
   ]
}
```

如果你想检索的文档在同一个_index中（甚至在同一个_type中），你就可以在URL中定义一个默认的/_index或者/_index/_type。
 **你依旧可以在单独的请求中使用这些值**：



```json
POST /website/blog/_mget
{
   "docs" : [
      { "_id" : 2 },
      { "_type" : "pageviews", "_id" :   1 }
   ]
}
```

事实上，如果所有文档具有相同_index和_type，你可以通过简单的ids数组来代替完整的docs数组：



```json
POST /website/blog/_mget
{
   "ids" : [ "2", "1" ]
}
```

注意到我们请求的第二个文档并不存在。我们定义了类型为blog，但是ID为1的文档类型为pageviews。这个不存在的文档会在响应体中被告知。



```json
{
  "docs" : [
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "2",
      "_version" : 10,
      "found" :    true,
      "_source" : {
        "title":   "My first external blog entry",
        "text":    "This is a piece of cake..."
      }
    },
    {
      "_index" :   "website",
      "_type" :    "blog",
      "_id" :      "1",
      "found" :    false  <1>
    }
  ]
}
```

*<1> 这个文档不存在*
 事实上第二个文档不存在并不影响第一个文档的检索。每个文档的检索和报告都是独立的。
 注意：

> ##### note
>
> 尽管前面提到有一个文档没有被找到，但HTTP请求状态码还是200。事实上，就算所有文档都找不到，请求也还是返回200，原因是mget请求本身成功了。如果想知道每个文档是否都成功了，你需要检查found标志。

### 更新时的批量操作

就像mget允许我们一次性检索多个文档一样，bulk API允许我们使用单一请求来实现多个文档的create、index、update或delete。这对索引类似于日志活动这样的数据流非常有用，它们可以以成百上千的数据为一个批次按序进行索引。

### 不要重复

你可能在同一个index下的同一个type里批量索引日志数据。为每个文档指定相同的元数据是多余的。就像mget API，bulk请求也可以在URL中使用/_index或/_index/_type:

### 多大才算太大？

整个批量请求需要被加载到接受我们请求节点的内存里，所以请求越大，给其它请求可用的内存就越小。有一个最佳的bulk请求大小。超过这个大小，性能不再提升而且可能降低。
 最佳大小，当然并不是一个固定的数字。它完全取决于你的硬件、你文档的大小和复杂度以及索引和搜索的负载。幸运的是，这个最佳点(sweetspot)还是容易找到的：
 试着批量索引标准的文档，随着大小的增长，当性能开始降低，说明你每个批次的大小太大了。开始的数量可以在1000~5000个文档之间，如果你的文档非常大，可以使用较小的批次。
 通常着眼于你请求批次的物理大小是非常有用的。一千个1kB的文档和一千个1MB的文档大不相同。**一个好的批次最好保持在5-15MB大小间。**

# 搜索

### 空搜索

最基本的搜索API表单是空搜索(empty search)，它没有指定任何的查询条件，只返回集群索引中的所有文档：



```undefined
GET /_search
```

响应内容（为了编辑简洁）类似于这样：



```json
{
   "hits" : {
      "total" :       14,
      "hits" : [
        {
          "_index":   "us",
          "_type":    "tweet",
          "_id":      "7",
          "_score":   1,
          "_source": {
             "date":    "2014-09-17",
             "name":    "John Smith",
             "tweet":   "The Query DSL is really powerful and flexible",
             "user_id": 2
          }
       },
        ... 9 RESULTS REMOVED ...
      ],
      "max_score" :   1
   },
   "took" :           4,
   "_shards" : {
      "failed" :      0,
      "successful" :  10,
      "total" :       10
   },
   "timed_out" :      false
}
```

> ### hits

响应中最重要的部分是hits，它包含了total字段来表示匹配到的文档总数，hits数组还包含了匹配到的前10条数据。
 hits数组中的每个结果都包含_index、_type和文档的_id字段，被加入到_source字段中这意味着在搜索结果中我们将可以直接使用全部文档。这不像其他搜索引擎只返回文档ID，需要你单独去获取文档。
 每个节点都有一个_score字段，这是相关性得分(relevance score)，它衡量了文档与查询的匹配程度。默认的，返回的结果中关联性最大的文档排在首位；这意味着，它是按照_score降序排列的。这种情况下，我们没有指定任何查询，所以所有文档的相关性是一样的，因此所有结果的_score都是取得一个中间值1
 max_score指的是所有文档匹配查询中_score的最大值。

> ### took

took告诉我们整个搜索请求花费的毫秒数。

> ### shards

_shards节点告诉我们参与查询的分片数（total字段），有多少是成功的（successful字段），有多少的是失败的（failed字段）。通常我们不希望分片失败，不过这个有可能发生。如果我们遭受一些重大的故障导致主分片和复制分片都故障，那这个分片的数据将无法响应给搜索请求。这种情况下，Elasticsearch将报告分片failed，但仍将继续返回剩余分片上的结果。

> ### timeout

time_out值告诉我们查询超时与否。一般的，搜索请求不会超时。如果响应速度比完整的结果更重要，你可以定义timeout参数为10或者10ms（10毫秒），或者1s（1秒）
 GET /_search?timeout=10ms
 Elasticsearch将返回在请求超时前收集到的结果。
 超时不是一个断路器（circuit breaker）（译者注：关于断路器的理解请看警告）。

> ## 警告
>
> 需要注意的是timeout不会停止执行查询，它仅仅告诉你目前顺利返回结果的节点然后关闭连接。在后台，其他分片可能依旧执行查询，尽管结果已经被发送。
>  使用超时是因为对于你的业务需求（译者注：SLA，Service-Level Agreement服务等级协议，在此我翻译为业务需求）来说非常重要，而不是因为你想中断执行长时间运行的查询。

### 多索引和多类别

你注意到空搜索的结果中不同类型的文档——user和tweet——来自于不同的索引——us和gb。
 通过限制搜索的不同索引或类型，我们可以在集群中跨所有文档搜索。Elasticsearch转发搜索请求到集群中平行的主分片或每个分片的复制分片上，收集结果后选择顶部十个返回给我们。
 通常，当然，你可能想搜索一个或几个自定的索引或类型，我们能通过定义URL中的索引或类型达到这个目的，像这样：

- `/_search`
   在所有索引的所有类型中搜索
- `/gb/_search`
   在索引gb的所有类型中搜索
- `/gb,us/_search`
   在索引gb和us的所有类型中搜索
- `/g*,u*/_search`
   在以g或u开头的索引的所有类型中搜索
- `/gb/user/_search`
   在索引gb的类型user中搜索
- `/gb,us/user,tweet/_search`
   在索引gb和us的类型为user和tweet中搜索
- `/_all/user,tweet/_search`
   在所有索引的user和tweet中搜索 search types user and tweet in all indices

当你搜索包含单一索引时，Elasticsearch转发搜索请求到这个索引的主分片或每个分片的复制分片上，然后聚集每个分片的结果。搜索包含多个索引也是同样的方式——只不过或有更多的分片被关联。

### 分页

和SQL使用`LIMIT`关键字返回只有一页的结果一样，Elasticsearch接受`from`和`size`参数：

- **size**: 结果数，默认10
- **from**: 跳过开始的结果数，默认0

如果你想每页显示5个结果，页码从1到3，那请求如下：



```csharp
GET /_search?size=5
GET /_search?size=5&from=5
GET /_search?size=5&from=10
```

应该当心分页太深或者一次请求太多的结果。结果在返回前会被排序。但是记住一个搜索请求常常涉及多个分片。每个分片生成自己排好序的结果，它们接着需要集中起来排序以确保整体排序正确。

### 简易搜索

search API有两种表单：一种是“简易版”的查询字符串(query string)将所有参数通过查询字符串定义，另一种版本使用JSON完整的表示请求体(request body)，这种富搜索语言叫做结构化查询语句（DSL）
 查询字符串搜索对于在命令行下运行点对点(ad hoc)查询特别有用。

##### 例如这个语句查询所有类型为tweet并在tweet字段中包含elasticsearch字符的文档：



```undefined
GET /_all/tweet/_search?q=tweet:elasticsearch
```

##### 下一个语句查找name字段中包含"john"和tweet字段包含"mary"的结果。实际的查询只需要：

`+name:john +tweet:mary`
 但是百分比编码(percent encoding)（译者注：就是url编码）需要将查询字符串参数变得更加神秘：



```undefined
GET /_search?q=%2Bname%3Ajohn+%2Btweet%3Amary
```

- **"+"前缀表示语句匹配条件必须被满足**。
- **类似的"-"前缀表示条件必须不被满足**。
- **所有条件如果没有+或-表示是可选的——匹配越多**，相关的文档就越多。

#### _all字段

返回包含"mary"字符的所有文档的简单搜索：
 GET /_search?q=mary
 在前一个例子中，我们搜索tweet或name字段中包含某个字符的结果。然而，这个语句返回的结果在三个不同的字段中包含"mary"：

1. 用户的名字是“Mary”
2. “Mary”发的六个推文
3. 针对“@mary”的一个推文

##### Elasticsearch是如何设法找到三个不同字段的结果的？

当你索引一个文档，Elasticsearch把所有字符串字段值连接起来放在一个大字符串中，它被索引为一个特殊的字段_all。例如，当索引这个文档：



```json
{
    "tweet":    "However did I manage before Elasticsearch?",
    "date":     "2014-09-14",
    "name":     "Mary Jones",
    "user_id":  1
}
```

这好比我们增加了一个叫做_all的额外字段值：
 "However did I manage before Elasticsearch? 2014-09-14 Mary Jones 1"
 若没有指定字段，查询字符串搜索（即q=xxx）使用_all字段搜索。

### 更复杂的语句

下一个搜索推特的语句：
 `_all` field

- `name`字段包含`"mary"`或`"john"`
- `date`晚于2014-09-10
- `_all`字段包含`"aggregations"`或"geo"



```css
+name:(mary john) +date:>2014-09-10 +(aggregations geo)
```

编码后的查询字符串变得不太容易阅读：



```ruby
?q=%2Bname%3A(mary+john)+%2Bdate%3A%3E2014-09-10+%2B(aggregations+geo)
```

就像你上面看到的例子，**简单(lite)查询字符串搜索惊人的强大**。它的查询语法，会在《查询字符串语法》章节阐述。参考文档允许我们简洁明快的表示复杂的查询。这对于命令行下一次性查询或者开发模式下非常有用。
 然而，你可以看到简洁带来了隐晦和调试困难。而且它很脆弱——查询字符串中一个细小的语法错误，像-、:、/或"错位就会导致返回错误而不是结果。
 最后，查询字符串搜索允许任意用户在索引中任何一个字段上运行潜在的慢查询语句，可能暴露私有信息甚至使你的集群瘫痪。

> TIP
>  因为这些原因，我们不建议直接暴露查询字符串搜索给用户，除非这些用户对于你的数据和集群可信。
>  取而代之的，生产环境我们一般依赖全功能的请求体搜索API，它能完成前面所有的事情，甚至更多。在了解它们之前，我们首先需要看看数据是如何在Elasticsearch中被索引的。



# 结构化查询 DSL

### 请求体查询

简单查询语句(lite)是一种有效的命令行*adhoc*查询。但是，如果你想要善用搜索，你必须使用请求体查询(requestbody search)API。之所以这么称呼，是因为大多数的参数以JSON格式所容纳而非查询字符串。

请求体查询(下文简称查询)，并不仅仅用来处理查询，而且还可以高亮返回结果中的片段，并且给出帮助你的用户找寻最好结果的相关数据建议。

##### 空查询

我们以最简单的 search
 API开始，空查询将会返回索引中所有的文档。



```xml
GET /_search
{} <1>
```

*<1> 这是一个空查询数据。*

同字符串查询一样，你可以查询一个，多个或`_all`索引(indices)或类型(types)：



```jsx
GET /index_2014*/type1,type2/_search
{}
```

你可以使用`from`
 及`size`
 参数进行分页：



```bash
GET /_search{ "from": 30, "size": 10}
```

携带内容的`GET`请求？

任何一种语言(特别是js)的HTTP库都不允许GET
 请求中携带交互数据。 事实上，有些用户很惊讶GET
 请求中居然会允许携带交互数据。
 真实情况是，[RFC 7231](https://link.jianshu.com?t=http://tools.ietf.org/html/rfc7231#page-24)， 一份规定HTTP语义及内容的RFC中并未规定GET
 请求中允许携带交互数据！ 所以，有些HTTP服务允许这种行为，而另一些(特别是缓存代理)，则不允许这种行为。
 Elasticsearch的作者们倾向于使用GET提交查询请求，因为他们觉得这个词相比POST来说，能更好的描述这种行为。 然而，因为携带交互数据的GET请求并不被广泛支持，所以search
 API同样支持POST请求，类似于这样：



```bash
POST /_search{ "from": 30, "size": 10}
```

这个原理同样应用于其他携带交互数据的GET
 API请求中。
 我们将在后续的章节中讨论聚合查询，但是现在我们把关注点仅放在查询语义上。
 相对于神秘的查询字符串方法，请求体查询允许我们使用结构化查询Query DSL(Query Domain Specific Language)

### 结构化查询 Query DSL

结构化查询是一种灵活的，多表现形式的查询语言。 Elasticsearch在一个简单的JSON接口中用结构化查询来展现Lucene绝大多数能力。 你应当在你的产品中采用这种方式进行查询。它使得你的查询更加灵活，精准，易于阅读并且易于debug。
 使用结构化查询，你需要传递`query`参数：



```json
GET /_search
{
    "query": YOUR_QUERY_HERE
}
```

空查询 - {} - 在功能上等同于使用match_all查询子句，正如其名字一样，匹配所有的文档：



```json
GET /_search
{
    "query": {
        "match_all": {}
    }
}
```

### 查询子句

一个查询子句一般使用这种结构：



```json
{
    QUERY_NAME: {
        ARGUMENT: VALUE,
        ARGUMENT: VALUE,...
    }
}
```

或指向一个指定的字段：



```json
{
    QUERY_NAME: {
        FIELD_NAME: {
            ARGUMENT: VALUE,
            ARGUMENT: VALUE,...
        }
    }
}
```

例如，你可以使用match查询子句用来找寻在tweet字段中找寻包含elasticsearch的成员：



```json
{
    "match": {
        "tweet": "elasticsearch"
    }
}
```

完整的查询请求会是这样：



```json
GET /_search
{
    "query": {
        "match": {
            "tweet": "elasticsearch"
        }
    }
}
```

### 合并多子句

查询子句就像是搭积木一样，可以合并简单的子句为一个复杂的查询语句，比如：

- 叶子子句(leaf clauses)(比如match子句)用以在将查询字符串与一个字段(或多字段)进行比较
- 复合子句(compound)用以合并其他的子句。例如，bool子句允许你合并其他的合法子句，must，must_not或者should，如果可能的话：



```json
{
    "bool": {
        "must":     { "match": { "tweet": "elasticsearch" }},
        "must_not": { "match": { "name":  "mary" }},
        "should":   { "match": { "tweet": "full text" }}
    }
}
```

**复合子句能合并 任意其他查询子句，包括其他的复合子句。**

> 这就意味着复合子句可以相互嵌套，从而实现非常复杂的逻辑。

以下实例查询的是邮件正文中含有“business opportunity”字样的星标邮件或收件箱中正文中含有“business opportunity”字样的非垃圾邮件：



```json
{
    "bool": {
        "must": { "match":      { "email": "business opportunity" }},
        "should": [
             { "match":         { "starred": true }},
             { "bool": {
                   "must":      { "folder": "inbox" }},
                   "must_not":  { "spam": true }}
             }}
        ],
        "minimum_should_match": 1
    }
}
```

不用担心这个例子的细节，我们将在后面详细解释它。 重点是复合子句可以合并多种子句为一个单一的查询，无论是叶子子句还是其他的复合子句。

### 查询与过滤

前面我们讲到的是关于结构化查询语句，事实上我们可以使用两种结构化语句： 结构化查询（Query DSL）和结构化过滤（Filter DSL）。 查询与过滤语句非常相似，但是它们由于使用目的不同而稍有差异。
 一条过滤语句会询问每个文档的字段值是否包含着特定值：

- created 的日期范围是否在 2013 到 2014 ?
- status 字段中是否包含单词 "published" ?
- lat_lon 字段中的地理位置与目标点相距是否不超过10km ?
   一条查询语句与过滤语句相似，但问法不同：
   查询语句会询问每个文档的字段值与特定值的**匹配程度**如何？
   查询语句的典型用法是为了找到文档：
- 查找与 full text search 这个词语最佳匹配的文档
- 查找包含单词 run ，但是也包含runs, running, jog 或 sprint的文档
- 同时包含着 quick, brown 和 fox --- 单词间离得越近，该文档的相关性越高
- 标识着 lucene, search 或 java --- 标识词越多，该文档的相关性越高
   **一条查询语句会计算每个文档与查询语句的相关性**，会给出一个相关性评分 `_score`，并且 按照相关性对匹配到的文档进行排序。 这种评分方式非常适用于一个没有完全配置结果的全文本搜索。

### 性能差异

**使用过滤语句得到的结果集 -- 一个简单的文档列表，快速匹配运算并存入内存是十分方便的**， 每个文档仅需要1个字节。这些缓存的过滤结果集与后续请求的结合使用是非常高效的。
 **查询语句不仅要查找相匹配的文档，还需要计算每个文档的相关性，所以一般来说查询语句要比 过滤语句更耗时**，并且查询结果也不可缓存。
 幸亏有了倒排索引，一个只匹配少量文档的简单查询语句在百万级文档中的查询效率会与一条经过缓存 的过滤语句旗鼓相当，甚至略占上风。 但是一般情况下，一条经过缓存的过滤查询要远胜一条查询语句的执行效率。
 过滤语句的目的就是缩小匹配的文档结果集，所以需要仔细检查过滤条件。

### 什么情况下使用

原则上来说，使用查询语句做全文本搜索或其他需要进行相关性评分的时候，剩下的全部用过滤语句

# 最重要的查询过滤语句

Elasticsearch 提供了丰富的查询过滤语句，而有一些是我们较常用到的。 我们将会在后续的《深入搜索》中展开讨论，现在我们快速的介绍一下 这些最常用到的查询过滤语句。

- term 过滤

term主要用于**精确匹配**哪些值，比如数字，日期，布尔值或 not_analyzed的字符串(未经分析的文本数据类型)：



```json
    { "term": { "age":    26           }}
    { "term": { "date":   "2014-09-01" }}
    { "term": { "public": true         }}
    { "term": { "tag":    "full_text"  }}
```

- terms 过滤

terms 跟 term 有点类似，但 **terms 允许指定多个匹配条件**。 如果某个字段指定了多个值，那么文档需要一起去做匹配：



```json
{
    "terms": {
        "tag": [ "search", "full_text", "nosql" ]
        }
}
```

- range 过滤

range过滤允许我们按照**指定范围查找**一批数据：



```json
{
    "range": {
        "age": {
            "gte":  20,
            "lt":   30
        }
    }
}
```

范围操作符包含：

- gt :: 大于
- gte:: 大于等于
- lt :: 小于
- lte:: 小于等于
- exists 和 missing 过滤

exists 和 missing 过滤可以用于查找文档中是否**包含指定字段或没有某个字段**，类似于SQL语句中的**IS_NULL**条件



```json
{
    "exists":   {
        "field":    "title"
    }
}
```

这两个过滤只是针对已经查出一批数据来，但是想区分出某个字段是否存在的时候使用。

- bool 过滤

bool 过滤可以用来**合并多个过滤条件查询结果的布尔逻辑**，它包含一下操作符：

- must :: 多个查询条件的完全匹配,相当于 and。
- must_not :: 多个查询条件的相反匹配，相当于 not。
- should :: 至少有一个查询条件匹配, 相当于 or。
   这些参数可以分别继承一个过滤条件或者一个过滤条件的数组：



```json
{
    "bool": {
        "must":     { "term": { "folder": "inbox" }},
        "must_not": { "term": { "tag":    "spam"  }},
        "should": [
                    { "term": { "starred": true   }},
                    { "term": { "unread":  true   }}
        ]
    }
}
```

- match_all 查询

使用match_all 可以**查询到所有文档**，是没有查询条件下的默认语句。



```json
{
    "match_all": {}
}
```

此查询常用于合并过滤条件。 比如说你需要检索所有的邮箱,所有的文档相关性都是相同的，所以得到的_score为1

- match 查询

match查询是一个标准查询，不管你需要全文本查询还是精确查询基本上都要用到它。
 如果你使用 match 查询一个全文本字段，它会在真正查询之前用分析器先分析match一下查询字符：



```json
{
    "match": {
        "tweet": "About Search"
    }
}
```

如果用match下指定了一个**确切值**，在遇到数字，日期，布尔值或者not_analyzed 的字符串时，它将为你搜索你给定的值：



```json
{ "match": { "age":    26           }}
{ "match": { "date":   "2014-09-01" }}
{ "match": { "public": true         }}
{ "match": { "tag":    "full_text"  }}
```

提示： 做精确匹配搜索时，你最好用过滤语句，因为过滤语句可以缓存数据。
 不像我们在《简单搜索》中介绍的字符查询，match查询不可以用类似"+usid:2 +tweet:search"这样的语句。 它只能就指定某个确切字段某个确切的值进行搜索，而你要做的就是为它指定正确的字段名以避免语法错误。

- multi_match 查询

multi_match查询允许你**做match查询的基础上同时搜索多个字段**：



```json
{
    "multi_match": {
        "query":    "full text search",
        "fields":   [ "title", "body" ]
    }
}
```

- bool 查询

bool 查询与 bool 过滤相似，用于合并多个查询子句。不同的是，**bool 过滤可以直接给出是否匹配成功， 而bool 查询要计算每一个查询子句的 _score （相关性分值）。**

- must:: 查询指定文档一定要被包含。
- must_not:: 查询指定文档一定不要被包含。
- should:: 查询指定文档，有则可以为文档相关性加分。
   以下查询将会找到 title 字段中包含 "how to make millions"，并且 "tag" 字段没有被标为 spam。 如果有标识为 "starred" 或者发布日期为2014年之前，那么这些匹配的文档将比同类网站等级高：



```json
{
    "bool": {
        "must":     { "match": { "title": "how to make millions" }},
        "must_not": { "match": { "tag":   "spam" }},
        "should": [
            { "match": { "tag": "starred" }},
            { "range": { "date": { "gte": "2014-01-01" }}}
        ]
    }
}
```

> **提示**： 如果bool 查询下没有must子句，那至少应该有一个should子句。但是 如果有must子句，那么没有should子句也可以进行查询。

# 查询与过滤条件的合并

查询语句和过滤语句可以放在各自的上下文中。 在 ElasticSearch API 中我们会看到许多带有`query`或 `filter` 的语句。 这些语句既可以包含单条 `query`语句，也可以包含一条`filter` 子句。 换句话说，**这些语句需要首先创建一个query或filter的上下文关系。**
 复合查询语句可以加入其他查询子句，复合过滤语句也可以加入其他过滤子句。 通常情况下，一条查询语句需要过滤语句的辅助，全文本搜索除外。
 所以说，查询语句可以包含过滤子句，反之亦然。 以便于我们切换 query 或 filter 的上下文。这就要求我们在读懂需求的同时构造正确有效的语句。
 带过滤的查询语句

## 过滤一条查询语句

比如说我们有这样一条查询语句:



```json
{
    "match": {
        "email": "business opportunity"
    }
}
```

然后我们想要让这条语句加入 term 过滤，在收信箱中匹配邮件：



```json
{
    "term": {
        "folder": "inbox"
    }
}
```

search API中只能包含 query 语句，所以我们需要用 filtered 来同时包含 "query" 和 "filter" 子句：



```json
{
    "filtered": {
        "query":  { "match": { "email": "business opportunity" }},
        "filter": { "term":  { "folder": "inbox" }}
    }
}
```

我们在外层再加入 query 的上下文关系：



```json
GET /_search
{
    "query": {
        "filtered": {
            "query":  { "match": { "email": "business opportunity" }},
            "filter": { "term": { "folder": "inbox" }}
        }
    }
}
```

## 单条过滤语句

在 query 上下文中，如果你只需要一条过滤语句，比如在匹配全部邮件的时候，你可以 省略 query 子句：



```json
GET /_search
{
    "query": {
        "filtered": {
            "filter":   { "term": { "folder": "inbox" }}
        }
    }
}
```

如果一条查询语句没有指定查询范围，那么它默认使用 match_all 查询，所以上面语句 的完整形式如下：



```json
GET /_search
{
    "query": {
        "filtered": {
            "query":    { "match_all": {}},
            "filter":   { "term": { "folder": "inbox" }}
        }
    }
}
```

## 查询语句中的过滤

有时候，你需要在 filter 的上下文中使用一个 query 子句。下面的语句就是一条带有查询功能 的过滤语句， 这条语句可以过滤掉看起来像垃圾邮件的文档：



```json
GET /_search
{
    "query": {
        "filtered": {
            "filter":   {
                "bool": {
                    "must":     { "term":  { "folder": "inbox" }},
                    "must_not": {
                        "query": { <1>
                            "match": { "email": "urgent business proposal" }
                        }
                    }
                }
            }
        }
    }
}
```

> <1> 过滤语句中可以使用query查询的方式代替 bool 过滤子句。
>  提示： 我们很少用到的过滤语句中包含查询，保留这种用法只是为了语法的完整性。 只有在过滤中用到全文本匹配的时候才会使用这种结构。

# 验证查询

查询语句可以变得非常复杂，特别是与不同的分析器和字段映射相结合后，就会有些难度。
 `validate API` **可以验证一条查询语句是否合法。**



```json
GET /gb/tweet/_validate/query
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

以上请求的返回值告诉我们这条语句是非法的：



```json
{
  "valid" :         false,
  "_shards" : {
    "total" :       1,
    "successful" :  1,
    "failed" :      0
  }
}
```

### 理解错误信息

想知道语句非法的具体错误信息，需要加上 `explain` 参数：



```json
GET /gb/tweet/_validate/query?explain <1>
{
   "query": {
      "tweet" : {
         "match" : "really powerful"
      }
   }
}
```

*<1> explain 参数可以提供语句错误的更多详情。*
 很显然，我们把 query 语句的 match 与字段名位置弄反了：



```json
{
  "valid" :     false,
  "_shards" :   { ... },
  "explanations" : [ {
    "index" :   "gb",
    "valid" :   false,
    "error" :   "org.elasticsearch.index.query.QueryParsingException:
                 [gb] No query registered for [tweet]"
  } ]
}
```

## 理解查询语句

如果是合法语句的话，使用 explain 参数可以返回一个带有查询语句的可阅读描述， 可以帮助了解查询语句在ES中是如何执行的：



```bash
GET /_validate/query?explain
{
   "query": {
      "match" : {
         "tweet" : "really powerful"
      }
   }
}
```

explanation 会为每一个索引返回一段描述，因为每个索引会有不同的映射关系和分析器：+



```json
{
  "valid" :         true,
  "_shards" :       { ... },
  "explanations" : [ {
    "index" :       "us",
    "valid" :       true,
    "explanation" : "tweet:really tweet:powerful"
  }, {
    "index" :       "gb",
    "valid" :       true,
    "explanation" : "tweet:really tweet:power"
  } ]
}
```

从返回的 explanation 你会看到 match 是如何为查询字符串 "really powerful" 进行查询的， 首先，它被拆分成两个独立的词分别在 tweet 字段中进行查询。
 而且，在索引us中这两个词为"really"和"powerful"，在索引gb中被拆分成"really" 和 "power"。 这是因为我们在索引gb中使用了english分析器。

# Why Elasticsearch ?

**我们为什么要使用ElasticSeach?**
 *以下基于目前我对于Elasticsearch的认识与理解*

# Q : 用一句话介绍Elasticsearch

> A : ElasticSearch是一个基于**Lucene**的搜索服务器。它提供了一个**分布式**、**多用户能力**的**全文搜索**引擎，基于**RESTful web**接口。有丰富的数据存取方式及完善的相关文档。

# Q : 现在有谁在用Elasticsearch

> github : 代码检索
>  百度、Amazon : 商品，搜索内容检索
>  知乎、等一些其他系统 : 对于日志系统的检索

> 在这里我额外说明非常重要的一点。Elasticsearch有着非常优异的全文检索性能。但是**检索**这个概念并不等同于我们平时操作数据库的query，后者一般被叫做**查询** 检索和查询最明显的区别就是主体对于结果是否有更多的已知信息，举个例子来讲 我们在百度搜索搜索你的qq号，可能会出来很多和你预期的结果非常无关的内容。有一些被搜索出来可能仅仅是因为他的url中正好含有你的qq号而已,而我们在数据库的某张表进行以qq号作为关键字查询数据的时候，我们已经预计了搜索的范围 比如只搜索这个qq号的主人的名字。再举一个并不十分恰当的例子 es就像是给所有字段都做了索引的mysql,你可能在设置搜索范围的时候速度会远远超出mysql，但是精确到某几个字段或者使用联合索引的时候 es的性能未必会比mysql强。就像python给内置数据结构list做了非常丰富的特效和查找方法以及优化，但是用下标取值的时候必然要比一个最简单的数组要慢。我们是否使用es 更应该取决于我们对于系统查询的结果的预计 例如我们到底是要一条精确的whois记录对应的域名 还是想在海量的whois字符串里检索出一些新的内容

# Q : Elasticsearch的学习成本

> 1~2 天即可看完官网文档， 基于python操作的话 学习1个官方文档和1个demo即可上手
>  总时间约在 20~50 h

# Q : Elasticsearch的工作/交互方式

> 通过API进行交互，通过http构造请求，返回相应的json数据

# Q : 某项目如果使用Elasticsearch会/不会用到那些特性



```diff
+ 高效的检索特性
+ 多种查询方式与过滤方式
+ 安全的多线程操作 
+ 高效的倒排索引
- 分布式架构 / 方便的多集群横向扩展
- 优秀的分词解析
- 高效的模糊匹配
- 高校的全文检索 
```

# Q : Elasticsearch与MySQL

> Elasticsearch 和 MySQL 天生结合的非常好，在Elasticsearch官方文档中也提到，Elasticsearch最常见的用途是作为副库来满足检索要求，而想MySQL等SQL DB来作为主库存储数据。 这样其实起到了主写从读的效果，相当于Elasticsearch分担了主库之前因为要满足查询所做的优化与索引的开销。其次,Elasticsearch 的结构天生和MySQL非常相似，几乎有完美的对应关系，同时网上有非常多的MySQL同步到ES的框架和第三方库，进一步的减小了工作量



作者：卡尔是正太
链接：https://www.jianshu.com/p/258c0eef3d31
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。