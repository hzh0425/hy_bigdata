## 前言

“Elasticsearch分布式一致性原理剖析”系列将会对Elasticsearch的分布式一致性原理进行详细的剖析，介绍其实现方式、原理以及其存在的问题等(基于6.2版本)。前一篇的内容包括了ES的集群组成、节点发现与Master选举、错误检测与集群扩缩容等。本篇将在前一篇的基础上，重点分析ES中meta更新的一致性问题，为了便于读者理解 ，本文还介绍了Master管理集群的方式、meta组成、存储方式等等。目录如下：

1. Master如何管理集群
2. Meta组成、存储和恢复
3. ClusterState的更新流程
4. 如何解决当前的一致性问题
5. 小结

## Master如何管理集群

在上一篇文章中，我们介绍了ES集群的组成，如何发现节点，选举Master等。那么在选举出Master之后，Master如何管理集群呢？比如以下问题：

1. Master如何处理新建或删除Index？
2. Master如何对Shard进行重新调度，实现负载均衡？

既然要管理集群，那么Master节点必然需要以某种方式通知其他节点，从而让其他节点执行相应的动作，来完成某些事情。比如建立一个新的Index就需要将其Shard分配在某些节点上，在这些节点上需要创建出对应Shard的目录，并在内存中创建对应Shard的一些结构等。

在ES中，Master节点是通过发布ClusterState来通知其他节点的。Master会将新的ClusterState发布给其他的所有节点，当节点收到新的ClusterState后，会把新的ClusterState发给相关的各个模块，各个模块根据新的ClusterState判断是否要做什么事情，比如创建Shard等。即这是一种通过Meta数据来驱动各个模块工作的方式。

在Master进行Meta变更并通知所有节点的过程中，需要考虑Meta变更的一致性问题，假如这个过程中Master挂掉了，那么可能只有部分节点按照新的Meta执行了操作。当选举出新的Master后，需要保证所有节点都要按照最新的Meta执行操作，不能回退，因为已经有节点按照新的Meta执行操作了，再回退就会导致不一致。

ES中只要新Meta在一个节点上被commit，那么就会开始执行相应的操作。因此我们要保证一旦新Meta在某个节点上被commit，此后无论谁是master，都要基于这个commit来产生更新的meta，否则就可能产生不一致。本文会分析ES处理这一问题的策略和存在的问题。

## Meta的组成、存储和恢复

### 1.Meta的组成

在介绍Meta更新流程前，我们先介绍一下ES中Meta的组成、存储方式和恢复方式，不关心这部分内容的读者可以略过本节。

####  Meta：ClusterState、MetaData、IndexMetaData

Meta是用来描述数据的数据。在ES中，Index的mapping结构、配置、持久化状态等就属于meta数据，集群的一些配置信息也属于meta。这类meta数据非常重要，假如记录某个index的meta数据丢失了，那么集群就认为这个index不再存在了。ES中的meta数据只能由master进行更新，master相当于是集群的大脑。

#### ClusterState

集群中的每个节点都会在内存中维护一个当前的ClusterState，表示当前集群的各种状态。ClusterState中包含一个MetaData的结构，MetaData中存储的内容更符合meta的特征，而且需要持久化的信息都在MetaData中，此外的一些变量可以认为是一些临时状态，是集群运行中动态构建出来的。

```text
ClusterState内容包括：
    long version: 当前版本号，每次更新加1
    String stateUUID：该state对应的唯一id
    RoutingTable routingTable：所有index的路由表
    DiscoveryNodes nodes：当前集群节点
    MetaData metaData：集群的meta数据
    ClusterBlocks blocks：用于屏蔽某些操作
    ImmutableOpenMap<String, Custom> customs: 自定义配置
    ClusterName clusterName：集群名
```

#### MetaData

上面提到，MetaData更符合meta的特征，而且需要持久化，那么我们看下这个MetaData中主要包含哪些东西：

```text
MetaData中需要持久化的包括：
    String clusterUUID：集群的唯一id。
    long version：当前版本号，每次更新加1
    Settings persistentSettings：持久化的集群设置
    ImmutableOpenMap<String, IndexMetaData> indices: 所有Index的Meta
    ImmutableOpenMap<String, IndexTemplateMetaData> templates：所有模版的Meta
    ImmutableOpenMap<String, Custom> customs: 自定义配置
```

我们看到，MetaData主要是集群的一些配置，集群所有Index的Meta，所有Template的Meta。下面我们再分析一下IndexMetaData，后面还会讲到，虽然IndexMetaData也是MetaData的一部分，但是存储上却是分开存储的。

#### IndexMetaData

IndexMetaData指具体某个Index的Meta，比如这个Index的shard数，replica数，mappings等。

```text
IndexMetaData中需要持久化的包括：
    long version：当前版本号，每次更新加1。
    int routingNumShards: 用于routing的shard数, 只能是该Index的numberOfShards的倍数，用于split。
    State state: Index的状态, 是个enum，值是OPEN或CLOSE。
    Settings settings：numbersOfShards，numbersOfRepilicas等配置。
    ImmutableOpenMap<String, MappingMetaData> mappings：Index的mapping
    ImmutableOpenMap<String, Custom> customs：自定义配置。
    ImmutableOpenMap<String, AliasMetaData> aliases： 别名
    long[] primaryTerms：primaryTerm在每次Shard切换Primary时加1，用于保序。
    ImmutableOpenIntMap<Set<String>> inSyncAllocationIds：处于InSync状态的AllocationId，用于保证数据一致性，下一篇文章会介绍。
```

### 2. Meta的存储

首先，在启动ES的一个节点时，会配置一个data目录，例如下面这个目录。该节点只有一个单shard的Index。

```text
$tree
.
`-- nodes
    `-- 0
        |-- _state
        |   |-- global-1.st
        |   `-- node-0.st
        |-- indices
        |   `-- 2Scrm6nuQOOxUN2ewtrNJw
        |       |-- 0
        |       |   |-- _state
        |       |   |   `-- state-0.st
        |       |   |-- index
        |       |   |   |-- segments_1
        |       |   |   `-- write.lock
        |       |   `-- translog
        |       |       |-- translog-1.tlog
        |       |       `-- translog.ckp
        |       `-- _state
        |           `-- state-2.st
        `-- node.lock
```

我们看到，ES进程会把Meta和Data都写入这个目录中，其中目录名为_state的代表该目录存储的是meta文件，根据文件层级的不同，共有3种meta的存储：

- nodes/0/_state/:

这层目录在节点级别，该目录下的global-1.st文件存储的是上文介绍的MetaData中除去IndexMetaData的部分，即一些集群级别的配置和templates。node-0.st中存储的是NodeId。

- nodes/0/indices/2Scrm6nuQOOxUN2ewtrNJw/_state/:

这层目录在index级别，2Scrm6nuQOOxUN2ewtrNJw是IndexId，该目录下的state-2.st文件存储的是上文介绍的IndexMetaData。

- nodes/0/indices/2Scrm6nuQOOxUN2ewtrNJw/0/_state/:

这层目录在shard级别，该目录下的state-0.st存储的是ShardStateMetaData，包含是否是primary和allocationId等信息。ShardStateMetaData是在IndexShard模块中管理，与其他Meta关联不大，本文不做过多介绍。

可以看到，集群相关的MetaData和Index的MetaData是在不同的目录中存储的。另外，集群相关的Meta会在所有的MasterNode和DataNode上存储，而Index的Meta会在所有的MasterNode和存储了该Index数据的DataNode上存储。

这里有个问题是，MetaData是由Master管理的，为什么DataNode上也要保存MetaData呢？主要原因是考虑到数据的安全性，很多用户没有考虑Master节点的高可用和数据高可靠，在部署ES集群时只配置了一个MasterNode，如果这个节点不可用，就会出现Meta丢失，后果非常严重。

### 3. Meta的恢复

假设ES集群重启了，那么所有进程都没有了之前的Meta信息，需要有一个角色来恢复Meta，这个角色就是Master。所以ES集群需要先进行Master选举，选出Master后，才会进行故障恢复。

当Master选举出来后，Master进程还会等待一些条件，比如集群当前的节点数大于某个数目等，这是避免有些DataNode还没有连上来，造成不必要的数据恢复等。

当Master进程决定进行恢复Meta时，它会向集群中的MasterNode和DataNode请求其机器上的MetaData。对于集群的Meta，选择其中version最大的版本。对于每个Index的Meta，也选择其中最大的版本。然后将集群的Meta和每个Index的Meta再组合起来，构成当前的最新Meta。

## ClusterState的更新流程

现在我们开始分析ClusterState的更新流程，并通过这个流程来看ES如何保证Meta更新的一致性。

### 1. master进程内不同线程更改ClusterState时的原子性保证

首先，master进程内不同线程更改ClusterState时要保证是原子的。试想一下这个场景，有两个线程都在修改ClusterState，各自更改其中的一部分。假如没有任何并发的保护，那么最后提交的线程可能就会覆盖掉前一个线程的修改，或者产生不符合条件的状态变更的发生。

ES解决这个问题的方式是，每次需要更新ClusterState时提交一个Task给MasterService，MasterService中只使用一个线程来串行处理这些Task，每次处理时把当前的ClusterState作为Task中execute函数的参数。即保证了所有的Task都是在currentClusterState的基础上进行更改，然后不同的Task是串行执行的。

### 2. ClusterState更改如何保证一旦commit，后续就一定会在此基础上commit，不会回退

这里是为了解决这样一个问题，我们知道，新的Meta一旦在某个节点上commit，那么这个节点就会执行相应的操作，比如删除某个Shard等，这样的操作是不可回退的。而假如此时Master节点挂掉了，新产生的Master一定要在新的Meta上进行更改，不能出现回退，否则就会出现Meta回退了但是操作无法回退的情况。本质上就是Meta更新没有保证一致性。

早期的ES版本没有解决这个问题，后来引入了两阶段提交的方式([Add two phased commit to Cluster State publishing](https://link.zhihu.com/?target=https%3A//github.com/elastic/elasticsearch/pull/13062))。所谓的两阶段提交，是把Master发布ClusterState分成两步，第一步是向所有节点send最新的ClusterState，当有超过半数的master节点返回ack时，再发送commit请求，要求节点commit接收到的ClusterState。如果没有超过半数的节点返回ack，那么认为本次发布失败，同时退出master状态，执行rejoin重新加入集群。

![img](https://gitee.com/zisuu/picture/raw/master/img/20210131182250.jpeg)

两阶段提交可以解决部分一致性问题，比如以下这种场景：

1. NodeA本来是Master节点，但由于某些原因NodeB成了新的Master节点，而NodeA由于探测不及时还未发现。
2. NodeA认为自己仍然是Master，于是照常发布新的ClusterState。
3. 由于此时NodeB是Master，说明超过半数的Master节点认为NodeB才是新的Master，于是超过半数的Master节点不会返回ack给NodeA。
4. NodeA收集不到足够的ack，于是本次发布失败，同时退出master状态。
5. 新的ClusterState不会在任何节点上commit，于是没有不一致发生。

但是这种方式也存在很多一致性的问题，我们下一节来具体分析。

## 一致性问题分析

ES中，Master发送commit的原则是只要有超过半数MasterNode(master-eligible node)接收了新的ClusterState就发送commit。那么实际上就是认为只要超过半数节点接收了新的ClusterState，这个ClusterState就一定可以被commit，不会在各种场景下回退。

### 问题1

第一阶段master节点send新的ClusterState，接收到的节点只是把新的ClusterState放入内存一个队列中，就会返回ack。这个过程没有持久化，所以当master接收到超过半数的ack后，也不能认为这些节点上都有新的ClusterState了。

### 问题2

如果master在commit阶段，只commit了少数几个节点就出现了网络分区，将master与这几个少数节点分在了一起，其他节点可以互相访问。此时其他节点构成多数派，会选举出新的master，由于这部分节点中没有任何节点commit了新的ClusterState，所以新的master仍会使用更新前的ClusterState，造成Meta不一致。

ES官方仍在追踪这个bug：

```text
https://www.elastic.co/guide/en/elasticsearch/resiliency/current/index.html

Repeated network partitions can cause cluster state updates to be lost (STATUS: ONGOING)
...This problem is mostly fixed by #20384 (v5.0.0), which takes committed cluster state updates into account during master election. This considerably reduces the chance of this rare problem occurring but does not fully mitigate it. If the second partition happens concurrently with a cluster state update and blocks the cluster state commit message from reaching a majority of nodes, it may be that the in flight update will be lost. If the now-isolated master can still acknowledge the cluster state update to the client this will amount to the loss of an acknowledged change. Fixing that last scenario needs considerable work. We are currently working on it but have no ETA yet.
```

### 什么情况下会有问题

在两阶段提交中，很重要的一个条件是，“超过半数节点返回ack，表示接收了这个ClusterState”，这个条件并不能带来任何保证。一方面接收ClusterState并不会持久化，另一方面接收了ClusterState也不会对未来选举新Master产生任何干扰，因为选举时只考虑已经commit的ClusterState。

在两阶段过程中，只到Master在超过半数MasterNode(master-eligible node)上commit了新的ClusterState，并且这些节点都完成了新ClusterState的持久化时，才到达一个安全的状态。在开始commit到达到这个状态之间，如果Master发送故障，就可能导致Meta发生不一致。

## 如何解决当前的一致性问题

既然ES目前Meta更新存在一些一致性问题，那么我们来脑洞一下，如何解决这些一致性问题呢？

### 1. 实现一个标准的一致性算法，比如raft

第一种方式是实现一个标准的一致性算法，比如raft。在上一篇中，我们也比较了ES的选举算法与raft算法的异同，下面我们继续比较一下ES的meta更新流程与raft的日志复制流程。

相同点：

1. 都是在得到超过半数节点应答之后，执行commit。

不同点：

1. raft算法中，follower接收到日志后就会进行持久化，写到磁盘上。ES中，节点接收到ClusterState只是放到内存中的一个队列中即返回，并不持久化。
2. raft算法可以保证在超过半数节点应答之后，这条日志一定可以被commit，而ES中没有保证这一点，目前还存在一致性问题。

通过上面的比较，我们再次看到，ES中meta更新的算法与raft相比很相似，raft使用了更多的机制来保证一致性，而ES还存在一些问题。用ES官方的话来说，fix这些问题还需要大量的工作(considerable work)。

### 2. 借助额外的组件保证meta一致性

比如使用Zookeeper来保存Meta，用Zookeeper来保证Meta的一致性。这样可以解决一致性的问题，但是性能的问题还需要再评估一下，比如Meta是否会过大而导致不保存在Zookeeper中，每次请求全量Meta还是Diff等。

### 3. 使用共享存储来保存Meta

首先保证不会出现脑裂，然后可以使用共享存储来保存Meta，解决Meta一致性的问题。这种方式会引入一个共享存储，这个存储系统需要具备高可靠、高可用特征。

## 小结

作为Elasticsearch分布式一致性原理系列的第二篇，本文主要介绍了ES集群中Master节点发布Meta更新的流程，分析了其中的一致性问题。此外还介绍了Meta数据的组成和存储方式等。下一篇文章会分析ES中如何保证数据的一致性，介绍其写入流程和算法模型等。