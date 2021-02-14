# 1分布式文档存储

文件是如何被分布存储到集群的，又是如何从集群中获取的呢？

### **1.1 路由一个文档到一个分片中**

公式：

> **shard =** **hash(routing) % number_of_primary_shards**

根据公式创建文档的时候确定一个文档存放在哪个一个分片中。routing是一个可变值，默认是文档的_id，也可以设置成一个自定义的值。 routing通过hash函数生成一个数字，然后这个数字再除以number_of_primary_shards（主分片的数量）后得到余数 。这个分布在0到number_of_primary_shards - 1之间的余数，就是我们所寻求的文档所在分片的位置。



### **1.2 主分片和副分片的交互**

以官网的例子进行分析，从图中能看出一个集群由三个节点组成，有两个分片，两个副本。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0afVja3d8GQFHibtKnicZd9S1OjhTtEkKcUxqOMtEQnica788oczpA9BJuA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图中集群中的任意一个节点有能力处理任意的请求，每个节点都知道集群中任意一个文档所处的位置。

###  

### **1.3 写一个文档操作**

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0aCkOTGyTWcOcMKykAza59bYibZahEQgdssceVr5ektrCdtv98YI2AJ2w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图中写操作必须在主分片上面完成之后，才能被复制到其他节点作为分片副本，新建、索引和删除请求都是写操作。



1）客户端向master发送写入请求，该节点作为协调节点；

2）根据文档的_id确定分片,图中请求文档属于分片0，协调节点请求转到节点的主分片；

3）在节点3上执行请求，成功之后，节点3根据副本数将请求并行转到副本分片对应节点，一旦副本分片执行完成，都向节点3报告成功，节点3将向协调节点报告成功，协调节点再向客户端报告成功。



客户端收到成功响应时，则变更操作是安全的。这个过程中有些请求参数影响效率。



### **1.4 取回一个文档**

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0azaMSzphpFHs1kICrxboBRNBVOxMt17JrldgnYhibibicJWUbD4tGI046A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1）客户端向master发送获取请求，该节点作为协调节点；

2）根据文档的_id确定分片,图中请求文档属于分片0，分片0的主副分片都在三个节点上，这儿将请求转到节点2；

3）节点2将文档返回给节点1，然后将文档返回给客户端。



在文档被检索时，已经被索引的文档可能已经存在于主分片上，但是还没有复制到副本分片。 在这种情况下，副本分片可能会报告文档不存在，但是主分片可能成功返回文档。 一旦索引请求成功返回给用户，文档在主分片和副本分片都是可用的。

###  

### **1.5 局部更新文档**

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0a2PHVgTPHxwr3EUSKwJqW3wias2XTAcjxiax87G8xfkFbU7lcDAKwC5Bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1）客户端向master发送更新请求，该节点作为协调节点；

2）根据文档的_id确定分片,图中请求文档属于分片0，协调节点将请求转到节点3；

3）节点3从主分片检索文档，修改 _source字段中的JSON ，并且尝试重新索引主分片的文档。如果文档已经被另一个进程修改，它会重试步骤3，超过 retry_on_conflict次后放弃；

4）如果节点3成功地更新文档，它将新版本的文档并行转发到其他节点上的副本分片，重新建立索引。一旦所有副本分片都返回成功，节点3向协调节点也返回成功，协调节点向客户端返回成功。

###  

### **1.6 多文档模式**



mget和bulk API的模式类似于单文档模式。区别在于协调节点知道每个文档存在于哪个分片中。 它将整个多文档请求分解成 每个分片 的多文档请求，并且将这些请求并行转发到每个参与节点。协调节点一旦收到来自每个节点的应答，就将每个节点的响应收集整理成单个响应，返回给客户端。



# 2使用mget取回多个文档

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0apeB9JpSpvYQ1uJPpqfHsUlCVtoBunArtSicoIYLjxWCyIaicWAs3lqYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1）客户端向master发送mget请求，该节点作为协调节点；

2）节点1为每个分片构建多文档获取请求，然后并行转发这些请求到托管在每个所需的主分片或者副本分片的节点上。一旦收到所有答复，节点1构建响应并将其返回给客户端。



# 3使用bulk修改多个文档

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0aMIXd3d3dEvF74XnGTYN6SFicleuiadhL86gMUWHVYytzEURE1IT4G4fg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1）客户端向master发送bluk请求，该节点作为协调节点；

2）节点1为每个节点创建一个批量请求，并将这些请求并行转发到每个包含主分片的节点主机；

3）主分片一个接一个按顺序执行每个操作。当每个操作成功时，主分片并行转发新文档（或删除）到副本分片，然后执行下一个操作。一旦所有的副本分片报告所有操作成功，该节点将向协调节点报告成功，协调节点将这些响应收集整理并返回给客户端。



# 4分片内部原理

分片，可以把他当成ES中最小的工作单元，它是如何保证近实时的？为何对文档的CRUD操作是实时的？是如何保证断电时也不丢数的？



### **4.1 倒排索引**

如何让文本可以被搜索的，Lucene是一个高性能的java全文检索工具包，最好的支持一个字段多个值需求的数据结构，它使用的是倒排文件索引结构。如果对倒排索引不熟悉，关系型数据库的索引应该会有了解，在关系型数据库中，索引是检索数据最有效率的方式，采用的是B+树来查询的，但是对于搜索引擎，并不能满足其特征要求。倒排索引的数据结构到底是怎么样的呢?一个倒排索引由文档中所有不重复词的列表构成，对于其中每个词，有一个包含它的文档列表。它会保存每一个词项出现过的文档总数，在对应的文档中一个具体词项出现的总次数，词项在文档中的顺序，每个文档的长度，所有文档的平均长度，等等。这些统计信息允许ES决定哪些词比其它词更重要，哪些文档比其它文档更重要。而不需要索引的field是可以设置为不被倒排索引的。



为了能够实现文档可以被搜索功能，倒排索引需要知道集合中的所有文档的位置，这是需要意识到的关键问题。早期的全文检索会为整个文档集合建立一个很大的倒排索引并将其写入到磁盘。一旦新的索引就绪，旧的就会被其替换，这样最近的变化便可以被检索到。

###  

### **4.2 不变性**

倒排索引被写入磁盘后是 `不可改变`(immutable)：永远不会被修改。不变性有如下几个重要的优势：

- 不需要锁。如果你没有必要更新索引，你就没有必要担心多进程会同时修改数据。
- 一旦索引被读入内核的文件系统缓存中，由于其不会改变，便会留在那里。只要文件系统缓存中还有足够的空间，那么大部分读请求会直接请求内存，而不会命中磁盘。这提供了很大的性能提升。
- 其它缓存(例如filter缓存)，在索引的生命周期内始终保持有效。因为数据不会改变，不需要在每次数据改变时被重建。
- 写入一个大的倒排索引中允许数据被压缩，减少磁盘 I/O 和 缓存索引所需的RAM量。

当然，一个不变的索引也有缺点。主要是它是不可变的! 你不能修改它。如果你需要让一个新的文档可被搜索，你需要重建整个索引。这对索引可以包含的数据量或可以更新索引的频率造成很大的限制。



### **4.3 动态更新索引**



> 下一个需要解决的问题是如何更新倒排索引，而不会失去其不变性的好处？ 答案是：`使用多个索引`。
>
> 通过增加一个新的补充索引来反映最近的修改，而不是直接重写整个倒排索引。每一个倒排索引都会被轮流查询–从最旧的开始–再对各个索引的查询结果进行合并。
>
> Lucene 是 Elasticsearch 所基于的Java库，引入了 `按段搜索` 的概念。 每一个段本身就是一个倒排索引， 但 Lucene 中的 `index` 除表示段 `segments` 的集合外，还增加了提交点 `commit point` 的概念，一个列出了所有已知段的文件，如下图所示展示了带有一个提交点和三个分段的 Lucene 索引:
>
> ![image-20210131210639306](https://gitee.com/zisuu/picture/raw/master/img/20210131210639.png)
>
> 新文档首先被添加到内存中的索引缓冲区中，如下图所示展示了一个在内存缓存中包含新文档准备提交的Lucene索引:
>
> ![image-20210131210646718](https://gitee.com/zisuu/picture/raw/master/img/20210131210646.png)
>
> 然后写入到一个基于磁盘的段，如下图所示展示了在一次提交后一个新的段添加到提交点而且缓存被清空:
>
> ![image-20210131210653998](https://gitee.com/zisuu/picture/raw/master/img/20210131210654.png)
>
> #### 2.1 索引与分片
>
> 一个 Lucene 索引就是我们 Elasticsearch 中的分片`shard`，而 Elasticsearch 中的一个索引是分片的集合。当 Elasticsearch 搜索索引时，它将查询发送到属于该索引的每个分片(Lucene索引)的副本(主分片，副本分片)上，然后将每个分片的结果聚合成全局结果集，如[ElasticSearch 内部原理之分布式文档搜索](http://smartsi.club/2016/10/26/elasticsearch-internal-distributed-document-search/)中描述。
>
> #### 2.2 按段搜索过程
>
> (1) 新文档被收集到内存索引缓冲区中，如上第一图；
>
> (2) 每隔一段时间，缓冲区就被提交：
>
> - 一个新的段(补充的倒排索引)被写入磁盘。
> - 一个新的提交点`commit point`被写入磁盘，其中包括新的段的名称。
> - 磁盘进行`同步` — 所有在文件系统缓冲区中等待写入的都 `flush` 到磁盘，以确保它们被写入物理文件。
>
> (3) 新分段被开启，使其包含的文档可以被搜索。
>
> (4) 内存缓冲区被清除，并准备好接受新的文档。
>
> 当一个查询被触发，所有已知的段按顺序被查询。词项统计会对所有段的结果进行聚合，以保证每个词和每个文档的关联都被准确计算。 这种方式可以用相对较低的成本将新文档添加到索引。
>
> ### 3. 删除与更新
>
> 段是不可变的，因此无法从旧的段中删除文档，也不能更新旧的段来反映文档的更新。相反，每个提交点 `commit point` 都包括一个 `.del` 文件，文件列出了哪个文档在哪个段中已经被删除了。
>
> 当文档被’删除’时，它实际上只是在 `.del` 文件中被标记为已删除。标记为已删除的文档仍然可以匹配查询，但在最终查询结果返回之前，它将从结果列表中删除。
>
> 文档更新也以类似的方式工作：当文档更新时，旧版本文档被标记为已删除，新版本文档被索引到新的段中。也许文档的两个版本都可以匹配查询，但是在查询结果返回之前旧的标记删除版本的文档会被移除。
>
> 在[ElasticSearch 段合并](http://smartsi.club/2016/11/05/elasticsearch-base-sgement-merge/)中，我们将展示如何从文件系统中清除已删除的文档。



# 5近实时搜索

### 1. 按段搜索

随着 `按段搜索` 的发展，索引文档与文档可被搜索的延迟显着下降。新文档可以在数分钟内可被搜索，但仍然不够快。

在这里磁盘是瓶颈。提交一个新的段到磁盘需要 `fsync` 来确保段被物理性地写入磁盘，这样在断电的时候也不会丢失数据。但是 `fsync` 代价很大; 如果每次索引一个文档都去执行一次的话会造成很大的性能问题。

我们需要的是一个更轻量的方式来使文档可被搜索，这意味着要从整个过程中移除 `fsync`。

在 Elasticsearch 和磁盘之间的是文件系统缓存。如前所述，内存中索引缓冲区中的文档(如下第一图)被写入新的段(如下第二图)．但是新的段首先被写入到文件系统缓存中 - 成本较低 - 只是稍后会被刷到磁盘 - 成本较高。但一旦文件在缓存中，它就可以像任何其他文件一样打开和读取。

![image-20210131211819341](https://gitee.com/zisuu/picture/raw/master/img/20210131211819.png)

![image-20210131211825759](https://gitee.com/zisuu/picture/raw/master/img/20210131211825.png)

Lucene 允许新段被写入和打开，使其包含的文档在没有进行一次完整提交之前便对搜索可见。这是一种比提交更轻量级的过程，并在不影响性能的前提下可以被频繁地执行。

### 2. Refresh API

在 ElasticSearch 中，这种轻量级写入和打开新片段的过程称为刷新`refresh`。默认情况下，每个分片每秒会自动刷新一次。这就是为什么我们说 Elasticsearch 是近实时搜索：文档更改不会立即对搜索可见，但会在1秒之内对搜索可见。

这可能会让新用户感到困惑：他们索引文档后并尝试搜索它，但是没有搜索到。这个问题的解决办法是使用 Refresh API 手动刷新一下：

```javascript
POST /_refresh
POST /blogs/_refresh
```

>  第一个语句刷新所有索引，第二个语句只是刷新`blogs`索引 

虽然刷新比提交(一次完整提交会将段刷到磁盘)更轻量级，但是仍然具有性能成本。编写测试时手动刷新可能很有用，但在生产环境中不要每次索引文档就去手动刷新。它会增大性能开销。相反，你的应用需要意识到 Elasticsearch 的近实时的性质，并做相应的补偿措施。

并非所有场景都需要每秒刷新一次。也许你正在使用 Elasticsearch 来索引数百万个日志文件，而你更希望优化索引速度，而不是近实时搜索。你可以通过设置 `refresh_interval` 来降低每个索引的刷新频率：

```javascript
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s"
  }
}
```

>  每30秒刷新一次`my_logs`索引。 

`refresh_interval` 可以在现有索引上动态更新。你可以在构建大型新索引时关闭自动刷新，然后在生产环境中开始使用索引时将其重新打开：

```javascript
PUT /my_logs/_settings
{ "refresh_interval": -1 }

PUT /my_logs/_settings
{ "refresh_interval": "1s" }
```

>  第一个语句禁用自动刷新,第二个语句每秒自动刷新一次; 

`refresh_interval` 需要一个持续时间值， 例如 1s （1 秒） 或 2m （2 分钟）。 一个绝对值 1 表示的是 1毫秒 –无疑会使你的集群陷入瘫痪(每一毫秒刷新一次)。



# 6Translog提供的磁盘同步控制

保证这期间发生主机错误、硬件故障等异常情况，数据不会丢失。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0avoh3vfrt5dcbgj4wibvwknK5nKaPXJgLSicjTbcykzK7fRBMjXeCgDZQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Refresh只是保证写到文件系统缓存，而写到磁盘这通过这步的操作来控制的。ES把数据写到内存缓存的同时，其实还同时记录了一个translog的日志数据。refresh发生的时候，translog日志文件依然保持原样。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Refresh完成后, 缓存被清空,但是事务日志不会。在这期间发生异常，ES会从commit位置开始，恢复整个translog文件中的记录，保证数据一致性。Translog文件要等到segment刷到磁盘，而且commit文件更新的时候，才能清空。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

以上的进程会继续工作，更多的文档被添加到内存缓冲区和追加到事务日志，事务日志不断积累文档。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

Translog变得越来越大，索引被执行flush；一个新的translog被创建，并且一个全量提交被执行。在flush之后，segment被全量提交，并且事务日志被清空。执行一个提交并且截断translog的行为被称一次flush,默认参数30分钟一次flush,或者translog文件大小超过500M的时候，可以调整以下参数：



```
1index.translog.flush_threshold_period
2index.translog.flush_threshold_size
3index.translog.flush_threshold_ops
```

#### 

####  

# 7Translog的安全性

文件被fsync到磁盘前，被写入的文件在重启之后就会丢失。默认translog是每5秒被 fsync 刷新到硬盘，或者在每次写请求完成之后执行。在2.0版本以后，为了保证不丢数据，每次 index、bulk、delete、update完成的时候，一定触发刷新translog到磁盘上，才给请求返回200。这个改变在提高数据安全性的同时当然也降低了一点性能。设置如下参数：



```
1"index.translog.durability": "async"
2"index.translog.sync_interval": "5s"
```







![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0aumicUibtRj5l4X97PcsvH8J6PNicmCiab7bzP5rNSYYznKIQFMdmenmQUA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

文件系统缓存被提交，新的段被追加到倒排索引序列后面，新的段被开启，而且可以被搜索，此时内存缓存被清空，等待接受的文档。

####  

# 8段合并（Segment merging）

自动刷新流程每秒会创建一个新的段，段的数据会暴增，段太多会消耗文件句柄、内存和CPU运行周期，这样导致段越多搜索就越慢。

![图片](https://mmbiz.qpic.cn/mmbiz_png/Rcon9f6LyEufA99KQqpJKWewUAZfjz0aOic8WibyRYt4Rs7DFzPpxrdz0V0lJiaa9zqmqu8NJTxlV67CHOs8IuKcg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

为了解决这个问题，利用小的段合并到大的段，然后继续合并大的段，合并过程中会把已删除的文档从文件系统中清除，这个过程是自动运行的，开发人员无感知。图中可以看出来将两个提交了的段和一个未提交的段正则进行到一个更多的段中。这个阶段如果有索引，刷新操作会创建新的段并将段打开，并提供给搜索使用，在合并过程中不会中断索引。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

合并结束的时候，老的段将被删除，新的段将被打开用于搜索。合并大的段需要消耗大量的I/O和CPU资源，如果大的segment一直合并会影响搜索性能。默认对合并流程进行资源限制，optimize API大可看做是强制合并，会指定大小的段的数量，通过检索段的数据来提升搜素性能。



# 9存储文件类型和格式

**9.1存储类型**

默认文件系统的实现为fs,以下部分列出了支持的所有不同存储类型：



1）fs

将根据操作环境选择最佳实现，该操作环境目前mmapfs在所有受支持的系统上，但可能会发生变化。



2）simplefs

Simple FS类型是SimpleFsDirectory使用随机访问文件直接实现文件系统存储（映射到Lucene ）。此实现具有较差的并发性能（多线程将成为瓶颈）。niofs当你需要索引持久性时，通常最好使用它。



3）niofs

NIO FS类型使用NIO将分片索引存储在文件系统上（映射到Lucene NIOFSDirectory）。它允许多个线程同时从同一个文件中读取。由于SUN Java实现中存在错误，因此不建议在Windows上使用。



4）mmapfs

MMap FS类型MMapDirectory通过将文件映射到内存（mmap）来将分片索引存储在文件系统上（映射到Lucene ）。内存映射使用进程中虚拟内存地址空间的一部分，等于要映射的文件的大小。在使用此类之前，请确保您已经拥有足够的 虚拟地址空间。



默认情况下，Elasticsearch完全依赖操作系统文件系统缓存来缓存I/O操作。可以设置index.store.preload 以便在打开时告诉操作系统将热索引文件的内容加载到存储器中。但需要注意，这可能会减慢索引的打开速度，因为它们只有在数据加载到物理内存后才可用。

###  

### **9.2 文件格式**

Lucene负责写和维护Lucene索引文件，而Elasticsearch在Lucene之上写与功能相关的元数据，例如字段映射，索引设置和其他集群元数据。找到Lucene索引文件之前，让我们了解下Elasticsearch编写的外部级别数据。



# 10启动Elatisearch时生成的文件

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

1）图中的路径/data1/es/data/nodes/0，这部分（/data1/es/data）在每个节点可以通过elasticsearch.yml配置path.data即可，可以多目录，不过多目录没有rebalance功能。nodes是恒定的文件夹名。

2）node.lock文件用于确保一次只能从一个数据目录读取或写入一个ES相关安装信息。

3）global-15.st文件。global-前缀表示这是一个全局状态文件，

4）而.st扩展名表示这是一个包含元数据的状态文件。



# 11Lucene中文件命名

/data1/es/data/nodes/0/indices目录的下一级目录是index的uuid，再下一级是分片id，如图下图所示的tree型图中，第一级就是uuid。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Rcon9f6LyEufA99KQqpJKWewUAZfjz0a9dL4PkMEmYl9M0DPy5j0Vj8icguStQJpBk86vqqn46vkZmfMqQPVjsQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

......

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Rcon9f6LyEufA99KQqpJKWewUAZfjz0aUicaRCajY4tChFSicPJkdD9Z0pxIvowmSZCo7cldSju29xweaMsGw5wA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

个段的所有文件具有相同的名称，有不同的扩展名。当使用复合索引文件的时候，这些文件将被压缩成单个.cfs文件。保存磁盘的时候，会创建一个从未使用过的文件名字。下表总结了Lucene中文件的名称和扩展名。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/Rcon9f6LyEufA99KQqpJKWewUAZfjz0a19VmicGxOicibia8opdGfgzNx5kGAsTkWQEfClMZRgLguUBd01m7kv9wHw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

索涉及相关的文件格式的流程，在贝壳智搜发布的[《搜索原理和lucene简介》](https://mp.weixin.qq.com/s?__biz=MzU3OTY2MjQ2NQ==&mid=2247483669&idx=1&sn=1c90e5b8f772cb557170ad5fe03fc873&scene=21#wechat_redirect)中有详细的介绍，这儿就不过多阐述。



# 12内容总结

本文对提出的问题进行分析底层原理，以上内容原理属于ES核心基础的其中一部分，对实时查询大数据级的需求可以很好的支持，能了解存储和查询的的内部运行原理，对于理解内部原理有很好的帮助。如果对以上内容讲解不是很清楚，可以参考《ES官网》（https://www.elastic.co/guide/en/elasticsearch/guide/2.x/index.html）文档进行分析理解。