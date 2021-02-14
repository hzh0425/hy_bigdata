ES 的读取分为GET 和Search 两种操作，这两种读取操作有较大的差异， GET/MGET 必须指
定三元组： index 、type 、id 。也就是说，根据文档id 从正排索引中获取内容。而Search 不指
定i d ，根据关键词从倒排索引中获取内容。本章分析GET/MGET 过程，下一章分析Search 过程。

![image-20210208105215843](C:\Users\86180\AppData\Roaming\Typora\typora-user-images\image-20210208105215843.png)

# 1.可选参数

![image-20210208105258817](https://gitee.com/zisuu/picture/raw/master/img/20210208105258.png)



> ### elasticsearch中有两个比较重要的操作：refresh 和 flush
>
> ##### refresh操作
>
> 当我们向ES发送请求的时候，我们发现es貌似可以在我们发请求的同时进行搜索。而这个实时建索引并可以被搜索的过程实际上是一次es 索引提交（commit）的过程，如果这个提交的过程直接将数据写入磁盘（fsync）必然会影响性能，所以es中设计了一种机制，即：先将index-buffer中文档（document）解析完成的segment写到filesystem cache之中，这样避免了比较损耗性能io操作，又可以使document可以被搜索。以上从index-buffer中取数据到filesystem cache中的过程叫做refresh。
>
>  
>
>  
>
> refresh操作可以通过API设置：
>
> POST /index/_settings
>
> {“refresh_interval”: “10s”}
>
> 当我们进行大规模的创建索引操作的时候，最好将将refresh关闭。
>
> POST /index/_settings
>
> {“refresh_interval”: “-1″}
>
>  
>
> es默认的refresh间隔时间是1s，这也是为什么ES可以进行近乎实时的搜索。
>
>  
>
> ##### flush操作与translog
>
> 我们可能已经意识到如果数据在filesystem cache之中是很有可能在意外的故障中丢失。这个时候就需要一种机制，可以将对es的操作记录下来，来确保当出现故障的时候，保留在filesystem的数据不会丢失，并在重启的时候可以从这个记录中将数据恢复过来。elasticsearch提供了translog来记录这些操作。
>
> 当向elasticsearch发送创建document索引请求的时候，document数据会先进入到index buffer之后，与此同时会将操作记录在translog之中，当发生refresh时（数据从index buffer中进入filesystem cache的过程）translog中的操作记录并不会被清除，而是当数据从filesystem cache中被写入磁盘之后才会将translog中清空。而从filesystem cache写入磁盘的过程就是flush。可能有点晕，我画了一个图帮大家理解这个过程：
>
> ![img](https://gitee.com/zisuu/picture/raw/master/img/20210208110946.jpeg)
>
> ##### 总结一下translog的功能：
>
> 1.保证在filesystem cache中的数据不会因为elasticsearch重启或是发生意外故障的时候丢失。
>
> 2.当系统重启时会从translog中恢复之前记录的操作。
>
> 3.当对elasticsearch进行CRUD操作的时候，会先到translog之中进行查找，因为tranlog之中保存的是最新的数据。
>
> 4.translog的清除时间时进行flush操作之后（将数据从filesystem cache刷入disk之中）。
>
>  
>
>  
>
> ##### 再总结一下flush操作的时间点：
>
> 1.es的各个shard会每个30分钟进行一次flush操作。
>
> 2.当translog的数据达到某个上限的时候会进行一次flush操作。
>
>  
>
>  
>
> ##### 有关于translog和flush的一些配置项：
>
> index.translog.flush_threshold_ops:当发生多少次操作时进行一次flush。默认是 unlimited。
>
> index.translog.flush_threshold_size:当translog的大小达到此值时会进行一次flush操作。默认是512mb。
>
> index.translog.flush_threshold_period:在指定的时间间隔内如果没有进行flush操作，会进行一次强制flush操作。默认是30m。
>
> index.translog.interval:多少时间间隔内会检查一次translog，来进行一次flush操作。es会随机的在这个值到这个值的2倍大小之间进行一次操作，默认是5s。
>
>  

# 2.Get基本流程

![image-20210208105414451](https://gitee.com/zisuu/picture/raw/master/img/20210208105414.png)



# 3.Get详细分析

## 1.协调结点

![image-20210209195504745](https://gitee.com/zisuu/picture/raw/master/img/20210209195504.png)

![image-20210209195702287](https://gitee.com/zisuu/picture/raw/master/img/20210209195702.png)

![image-20210209195712297](https://gitee.com/zisuu/picture/raw/master/img/20210209195712.png)

![image-20210209195723705](https://gitee.com/zisuu/picture/raw/master/img/20210209195723.png)

![image-20210209195736542](https://gitee.com/zisuu/picture/raw/master/img/20210209195736.png)

## 2.数据结点

![image-20210209201922342](https://gitee.com/zisuu/picture/raw/master/img/20210209201922.png)

shardOperation由子类实现,这里就是TransportGetAction:

![image-20210209202011566](https://gitee.com/zisuu/picture/raw/master/img/20210209202011.png)

![image-20210209202019068](https://gitee.com/zisuu/picture/raw/master/img/20210209202019.png)

![image-20210209202047974](https://gitee.com/zisuu/picture/raw/master/img/20210209202048.png)

![image-20210209202054590](https://gitee.com/zisuu/picture/raw/master/img/20210209202054.png)

# 4.MGET流程分析

![image-20210209202146356](https://gitee.com/zisuu/picture/raw/master/img/20210209202146.png)

![image-20210209202236767](https://gitee.com/zisuu/picture/raw/master/img/20210209202237.png)