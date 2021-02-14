# 源码分析（一）

## 1 前言

Elasticsearch（ES）是一个基于Lucene的分布式存储和搜索分析系统，本文希望从源码的角度分析ES在保证数据的可靠性、实时性和一致性前提下，其写入的具体流程。

> 写入也是整个ES系统里面，最主要的流程之一，便于更好的理解ES的内部原理和逻辑，关于ES数据存储结构请参考：[【Elasticsearch】原理-Elasticsearch数据存储结构与写入流程](https://blog.csdn.net/wudingmei1023/article/details/103897487)。

## 2 写入基本流程

> 图片来自官网，源代码取自6.7.1版本：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200111111751894.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3d1ZGluZ21laTEwMjM=,size_16,color_FFFFFF,t_70)
ES的写入采用一主多副的模式，写操作一般会经过三种节点：协调节点、主分片所在节点、副本分片所在节点。
客户端发送请求到Node1（相当于协调节点），协调节点收到请求之后，确认写入的文档属于分片P0，于是将请求转发给P0所在的节点Node3，Node3写完成之后将请求转发到P0所属的副本R0所在的节点Node1和Node2。

> 什么时候给客户端返回成功呢？

**特别注意：** 取决于wait_for_active_shards参数：需要确认的分片数，默认为1，即主分片写入成功就返回客户端结果。

```java
    /**
     * The number of active shard copies to check for before proceeding with a write operation.
     */
    public static final Setting<ActiveShardCount> SETTING_WAIT_FOR_ACTIVE_SHARDS =
        new Setting<>("index.write.wait_for_active_shards",
                      "1",
                      ActiveShardCount::parseString,
                      Setting.Property.Dynamic,
                      Setting.Property.IndexScope);
123456789
```

以上是写入的大体流程，整个详细的流程，通过源码进行分析。

## 3 写入源码分析

ES的写入官方提供了两种写入方式：index，逐条写入；Bulk，批量写入。对于这两种方式，ES都会转化成Bulk写入。

## 3.1 bulk请求分发

ES的写入请求一般会进过两层处理，首先的Rest层（进行请求参数解析），另一层是Transport层（进行实际的请求处理）。在每一层处理前都有一次请求分发：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200111140627564.png)
客户端发送过来的HTTP请求由HttpServerTransport初步处理后进入RestController模块进行实际的分发过程：

```java
    public void dispatchRequest(RestRequest request, RestChannel channel, ThreadContext threadContext) {
        if (request.rawPath().equals("/favicon.ico")) {
            handleFavicon(request, channel);
            return;
        }
        try {
        	//找出所有可能的handlers，然后分发这些请求
            tryAllHandlers(request, channel, threadContext);
        } catch (Exception e) {
          .......
        }
    }
123456789101112
```

上面dispatchRequest方法，会通过tryAllHandlers方法找出所有可能的handlers，并分发请求，代码如下：

```java
    void tryAllHandlers(final RestRequest request, final RestChannel channel, final ThreadContext threadContext) throws Exception {
        for (String key : headersToCopy) {
            String httpHeader = request.header(key);
            if (httpHeader != null) {
                threadContext.putHeader(key, httpHeader);
            }
        }
     .....
        // 获取所有可能的Handler，并尝试分发request请求
        Iterator<MethodHandlers> allHandlers = getAllHandlers(request);
        for (Iterator<MethodHandlers> it = allHandlers; it.hasNext(); ) {
            final Optional<RestHandler> mHandler = Optional.ofNullable(it.next()).flatMap(mh -> mh.getHandler(request.method()));
            //进行request请求分发，如果分发成功，则返回true
            requestHandled = dispatchRequest(request, channel, client, mHandler);
            if (requestHandled) {
                break;
            }
        }
        .....
    }
1234567891011121314151617181920
```

首先根据request找到其对应的handler，然后在dispatchRequest中调用handler的handleRequest方法处理请求。那么getHandler是如何根据请求找到对应的handler的呢？这块的逻辑如下：

```java
    Iterator<MethodHandlers> getAllHandlers(final RestRequest request) {
        final Map<String, String> originalParams = new HashMap<>(request.params());
        return handlers.retrieveAll(getPath(request), () -> {
            request.params().clear();
            request.params().putAll(originalParams);
            return request.params();
        });
    }

    public void registerHandler(RestRequest.Method method, String path, RestHandler handler) {
        if (handler instanceof BaseRestHandler) {
            usageService.addRestHandler((BaseRestHandler) handler);
        }
        handlers.insertOrUpdate(path, new MethodHandlers(path, handler, method), (mHandlers, newMHandler) -> {
            return mHandlers.addMethods(handler, method);
        });
    }
1234567891011121314151617
```

ES会通过RestController的registerHandler方法，提前把handler注册到对应http请求方法（GET、PUT、POST、DELETE等）的handlers列表。这样用户请求到达时，就可以通过RestController的getHandler方法，并根据http请求方法和路径取出对应的handler。对于bulk操作，其请求对应的handler是RestBulkAction，该类会在其构造函数中将其注册到RestController，代码如下：

```java
    public RestBulkAction(Settings settings, RestController controller) {
        super(settings);
        controller.registerHandler(POST, "/_bulk", this);
        controller.registerHandler(PUT, "/_bulk", this);
        controller.registerHandler(POST, "/{index}/_bulk", this);
        controller.registerHandler(PUT, "/{index}/_bulk", this);
        controller.registerHandler(POST, "/{index}/{type}/_bulk", this);
        controller.registerHandler(PUT, "/{index}/{type}/_bulk", this);
        this.allowExplicitIndex = MULTI_ALLOW_EXPLICIT_INDEX.get(settings);
    }
12345678910
```

RestBulkAction会将RestRequest解析并转化为BulkRequest，然后再对BulkRequest做处理，这块的逻辑在prepareRequest方法中，部分代码如下：

```java
    public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
        //根据RestRequest构建bulkRequest
        ......
        //处理bulkRequest请求
        return channel -> client.bulk(bulkRequest, new RestStatusToXContentListener<>(channel));
    }
123456
```

NodeClient在处理BulkRequest请求时，会将请求的action转化为对应Transport层的action，然后再由Transport层的action来处理BulkRequest，action转化的代码如下：

```java
    public <    Request extends ActionRequest,
                Response extends ActionResponse
            > Task executeLocally(GenericAction<Request, Response> action, Request request, TaskListener<Response> listener) {
        return transportAction(action).execute(request, listener);
    }
	
    private <    Request extends ActionRequest,
                Response extends ActionResponse
            > TransportAction<Request, Response> transportAction(GenericAction<Request, Response> action) {
        .....
        //actions是个action到transportAction的Map，这个映射关系是在节点启动时初始化的
        TransportAction<Request, Response> transportAction = actions.get(action);
        ......
        return transportAction;
    }	
123456789101112131415
```

然后进入TransportAction，TransportAction#execute(Request request, ActionListener listener) -> TransportAction#execute(Task task, Request request, ActionListener listener) -> TransportAction#proceed(Task task, String actionName, Request request, ActionListener listener)。TransportAction会调用一个请求过滤链来处理请求，如果相关的插件定义了对该action的过滤处理，则先会执行插件的处理逻辑，然后再进入TransportAction的处理逻辑，过滤链的处理逻辑如下：

```java
        public void proceed(Task task, String actionName, Request request, ActionListener<Response> listener) {
            int i = index.getAndIncrement();
            try {
                if (i < this.action.filters.length) {
                	//应用插件的逻辑
                    this.action.filters[i].apply(task, actionName, request, listener, this);
                } else if (i == this.action.filters.length) {
                	//执行TransportAction的逻辑
                    this.action.doExecute(task, request, listener);
                } else {
                   ......
                }
            } catch(Exception e) {
               .....
            }
        }
12345678910111213141516
```

对于Bulk请求，这里的TransportAction对应的具体对象是TransportBulkAction的实例，到此，Rest层转化为Transport层的流程完成，下节将详细介绍TransportBulkAction的处理逻辑。

## 3.2 bulk写入流程

> 代码入口：TransportBulkAction#doExecute(Task task, BulkRequest bulkRequest, ActionListener listener)。

## 3.2.1 pipeline预处理

首先判断bulk请求中是否指定了pipeline参数，则先使用相应的pipeline进行处理。如果本节点不具备预处理（Ingest）的资格，则将请求转发到有资格的节点。如果没有Ingest节点则继续往下走。

## 3.2.2 创建索引

判断是否需要创建索引，即needToChec方法，返回的autoCreateIndex的开关，默认是true，即自动创建索引是打开的；

```java
    boolean needToCheck() {
        return autoCreateIndex.needToCheck();
    }
    
    public static final Setting<AutoCreate> AUTO_CREATE_INDEX_SETTING =
        new Setting<>("action.auto_create_index", "true", AutoCreate::new, Property.NodeScope, Setting.Property.Dynamic);
123456
```

如果自动创建索引已关闭，则直接准备下一步操作：

```java
 executeBulk(task, bulkRequest, startTime, listener, responses, emptyMap());
```

# 源码分析（二）

如果需要自动创建索引，则需要遍历bulk的所有index，然后检查index是否需要自动创建，对于不存在的index，则会加入到自动创建的集合中，然后会调用createIndex方法创建index。index的创建由master来把控，master会根据分片分配和均衡的算法来决定在哪些data node上创建index对应的shard，然后将信息同步到data node上，由data node来执行具体的创建动作。

```java
  // Step 1: 对bulkRequest进行过滤，获取所有的索引名。主要为opType和versionType，其中opType为索引操作类型，支持INDEX、CREATE,UPDATE,DELETE四种。DELETE请求如果索引不存在，不应该创建索引，除非external versioning正在使用。
            final Set<String> indices = bulkRequest.requests.stream()
                .filter(request -> request.opType() != DocWriteRequest.OpType.DELETE
                    || request.versionType() == VersionType.EXTERNAL
                    || request.versionType() == VersionType.EXTERNAL_GTE)
                .map(DocWriteRequest::index)
                .collect(Collectors.toSet());
// Step 2: 对各个索引进行检查，indicesThatCannotBeCreated用来存储无法创建索引的信息Map，autoCreateIndices用来存储可以自动创建索引的Set。
//索引是否可以正常自动创建，主要检查：1.是否存在该索引或别名（存在则无法创建）；2.该索引是否被允许自动创建（二次检查，为了防止check信息丢失）；3.动态mapping是否被禁用（如果被禁用，则无法创建）；4.创建索引的匹配规则是否存在并可以正常匹配（如果表达式非空，且该索引无法匹配上，则无法创建）。
            final Map<String, IndexNotFoundException> indicesThatCannotBeCreated = new HashMap<>();
            Set<String> autoCreateIndices = new HashSet<>();
            ClusterState state = clusterService.state();
            for (String index : indices) {
                boolean shouldAutoCreate;
                try {
                    shouldAutoCreate = shouldAutoCreate(index, state);
                } catch (.....) { .....}
                if (shouldAutoCreate) {
                    autoCreateIndices.add(index);
                }
            }
// Step 3: 如果没有索引需要创建，直接executeBulk到下一步；如果存在需要创建的索引，则逐个创建索引，并监听结果，成功计数器减1.失败的话，将bulkRequest中对应的request的value值设置为null，计数器减1，当所有索引执行"创建索引"操作结束后，即计数器为0时，进入executeBulk。
            if (autoCreateIndices.isEmpty()) {
                executeBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);
            } else {
                final AtomicInteger counter = new AtomicInteger(autoCreateIndices.size());
                for (String index : autoCreateIndices) {
                    createIndex(index, bulkRequest.timeout(), new ActionListener<CreateIndexResponse>() {
                        @Override
                        public void onResponse(CreateIndexResponse result) {
                            if (counter.decrementAndGet() == 0) {
                                executeBulk(task, bulkRequest, startTime, listener, responses, indicesThatCannotBeCreated);
                            }
                        }
                        @Override
                        public void onFailure(Exception e) {
						..........
                    });
                }
123456789101112131415161718192021222324252627282930313233343536373839
```

## 3.2.3 协调节点处理并转发请求

创建完index之后，index的各shard已在数据节点上建立完成，接着协调节点将会转发写入请求到文档对应的primary shard。进入到BulkOperation#doRun中。
首先会检查集群无BlockException后（存在BlockedException会不断重试，直至超时），然后遍历BulkRequest的所有子请求，然后根据请求的操作类型生成相应的逻辑，对于写入请求，会首先根据IndexMetaData信息，resolveRouting方法为每条IndexRequest生成路由信息，并通过process方法按需生成doc id（不指定的话默认是UUID）。

```java
            for (int i = 0; i < bulkRequest.requests.size(); i++) {
                DocWriteRequest docWriteRequest = bulkRequest.requests.get(i);
				.......
                Index concreteIndex = concreteIndices.resolveIfAbsent(docWriteRequest);
                try {
                    switch (docWriteRequest.opType()) {
                        case CREATE:
                        case INDEX:
                            .......
                            indexRequest.resolveRouting(metaData);
                            indexRequest.process(indexCreated, mappingMd, concreteIndex.getName());
                            break;
							......
                    }
                } catch (.......) {.......}
            }
12345678910111213141516
```

然后根据每个IndexRequest请求的路由信息（如果写入时未指定路由，则es默认使用doc id作为路由）得到所要写入的目标shard id，并将DocWriteRequest封装为BulkItemRequest且添加到对应shardId的请求列表中。代码如下：

```java
			//requestsByShard的key是shard id，value是对应的单个doc写入请求（会被封装成BulkItemRequest）的集合
            Map<ShardId, List<BulkItemRequest>> requestsByShard = new HashMap<>();
            for (int i = 0; i < bulkRequest.requests.size(); i++) {
            	//从bulk请求中得到每个doc写入请求
                DocWriteRequest request = bulkRequest.requests.get(i);
                ......
                String concreteIndex = concreteIndices.getConcreteIndex(request.index()).getName();
                //根据路由，找出doc写入的目标shard id
                ShardId shardId = clusterService.operationRouting().indexShards(clusterState, concreteIndex, request.id(),
                    request.routing()).shardId();
                List<BulkItemRequest> shardRequests = requestsByShard.computeIfAbsent(shardId, shard -> new ArrayList<>());
                shardRequests.add(new BulkItemRequest(i, request));
            }
12345678910111213
```

计算ShardId的代码如下所示：这里的partitionOffset是根据参数index.routing_partition_size获取的，默认为1，写入时指定id，可能导致分布不均，可调大该参数，让分片id可变范围更大，分布更均匀。routingFactor默认为1，主要是在做spilt和shrink时改变。

```java
    private static int calculateScaledShardId(IndexMetaData indexMetaData, String effectiveRouting, int partitionOffset) {
        final int hash = Murmur3HashFunction.hash(effectiveRouting) + partitionOffset;
        return Math.floorMod(hash, indexMetaData.getRoutingNumShards()) / indexMetaData.getRoutingFactor();
    }
1234
```

上一步已经找出每个shard及其所需执行的doc写入请求列表的对应关系，这里就相当于将请求按shard进行了拆分，接下来会将每个shard对应的所有请求封装为BulkShardRequest并交由TransportShardBulkAction来处理：即将相同shard id的请求合并，并转发TransportShardBulkAction请求。

```java
            for (Map.Entry<ShardId, List<BulkItemRequest>> entry : requestsByShard.entrySet()) {
                final ShardId shardId = entry.getKey();
                final List<BulkItemRequest> requests = entry.getValue();
                // 对每个shard id及对应的BulkItemRequest集合，合并为一个BulkShardRequest
                BulkShardRequest bulkShardRequest = new BulkShardRequest(shardId, bulkRequest.getRefreshPolicy(),
                    requests.toArray(new BulkItemRequest[requests.size()]));
                ......
                if (task != null) {
                    bulkShardRequest.setParentTask(nodeId, task.getId());
                // 处理请求（在listener中等待响应，响应都是按shard返回的，如果一个shard中有部分请求失败，将异常填到response中，所有请求完成，即计数器为0，调用finishHim()，整体请求做成功处理）：
                shardBulkAction.execute(bulkShardRequest, new ActionListener<BulkShardResponse>() {
         			........
                });
            }
1234567891011121314
```

## 3.2.4 向主分片发送请求

转发TransportShardBulkAction请求，最后进入到TransportReplicationAction#doExecute方法，然后进入到TransportReplicationAction.ReroutePhase#doRun方法。这里会通过ClusterState获取到primary shard的路由信息，然后得到primay shard所在的node，如果node为当前协调节点则直接将请求发往本地，否则发往远端：

```java
            setPhase(task, "routing"); //标识为routing阶段
            final ClusterState state = observer.setAndGetObservedState();
            .......
            } else {
                // 获取主分片所在的shard路由信息，得到主分片所在的node节点
                final IndexMetaData indexMetaData = state.metaData().index(concreteIndex);
                .........
                final DiscoveryNode node = state.nodes().get(primary.currentNodeId());
                if (primary.currentNodeId().equals(state.nodes().getLocalNodeId())) {
                	//是当前节点，继续执行
                    performLocalAction(state, primary, node, indexMetaData);
                } else {
                	//不是当前节点，转发到对应的node上进行处理
                    performRemoteAction(state, primary, node);
                }
            }
12345678910111213141516
```

如果分片在当前节点，task当前阶段置为“waiting_on_primary”，否则为“rerouted”，两者都走到同一入口，即performAction(…)， 在performAction方法中，会调用TransportService的sendRequest方法，将请求发送出去。
如果对端返回异常，比如对端节点故障或者primary shard挂了，对于这些异常，协调节点会有重试机制，重试的逻辑为等待获取最新的集群状态，然后再根据集群的最新状态（通过集群状态可以拿到新的primary shard信息）重新执行上面的doRun逻辑；如果在等待集群状态更新时超时，则会执行最后一次重试操作（执行doRun）。这块的代码如下：

```java
        void retry(Exception failure) {
            if (observer.isTimedOut()) {
                // 超时时已经做过最后一次尝试，这里将不再重试，超时默认1min
                finishAsFailed(failure);
                return;
            }
            setPhase(task, "waiting_for_retry");
            request.onRetry();
            observer.waitForNextChange(new ClusterStateObserver.Listener() {
                @Override
                public void onNewClusterState(ClusterState state) {
                    run(); //会调用doRun
                }
                .......
                @Override
                public void onTimeout(TimeValue timeout) { //超时，做最后一次重试
                    // Try one more time...
                    run(); //会调用doRun
                }
            });
        }
```

# 源码分析（三）

## 3.2.5 写主分片节点流程

代码入口：TransportReplicationAction.PrimaryOperationTransportHandler#messageReceived，然后进入AsyncPrimaryAction#doRun方法。
**检查请求：** 1.当前是否为主分片；2.allocationId是否是预期值；3.PrimaryTerm是否是预期值

```java
            if (shardRouting.primary() == false) {
                .....
            }
            final String actualAllocationId = shardRouting.allocationId().getId();
            if (actualAllocationId.equals(targetAllocationID) == false) {
                ......
            }
            final long actualTerm = indexShard.getPendingPrimaryTerm();
            if (actualTerm != primaryTerm) {
              	......
            }
1234567891011
```

**查看主分片是否迁移：**
如果已经迁移：1.将phase状态设为“primary_delegation”；2.关闭当前分片的primaryShardReference，及时释放资源；3.获取已经迁移到的目标节点，将请求转发到该节点，并等待执行结果；4.拿到结果后，将task状态更新为“finish”。

```java
                    transportService.sendRequest(relocatingNode, transportPrimaryAction,
                        new ConcreteShardRequest<>(request, primary.allocationId().getRelocationId(), primaryTerm),
                        transportOptions,
                        new TransportChannelResponseHandler<Response>(logger, channel, "rerouting indexing to target primary " + primary,
                            reader) {
                            @Override
                            public void handleResponse(Response response) {
                                setPhase(replicationTask, "finished");
                                super.handleResponse(response);
                            }
                            @Override
                            public void handleException(TransportException exp) {
                                setPhase(replicationTask, "finished");
                                super.handleException(exp);
                            }
                        });
12345678910111213141516
```

如果没有迁移：
1.将task状态更新为“primary”；2.主分片准备操作(主要部分)；3.转发请求给副本分片

```java
setPhase(replicationTask, "primary");
                    final ActionListener<Response> listener = createResponseListener(primaryShardReference);
                    createReplicatedOperation(request,
                        ActionListener.wrap(result -> result.respond(listener), listener::onFailure),
                        primaryShardReference).execute(); //入口
12345
```

primary所在的node收到协调节点发过来的写入请求后，开始正式执行写入的逻辑，写入执行的入口是在ReplicationOperation类的execute方法，该方法中执行的两个关键步骤是，首先写主shard，如果主shard写入成功，再将写入请求发送到从shard所在的节点。

```java
    public void execute() throws Exception {
        .......
        //关键，这里开始执行写主分片
        primaryResult = primary.perform(request);
		.......
        final ReplicaRequest replicaRequest = primaryResult.replicaRequest();
        if (replicaRequest != null) {
			........
            markUnavailableShardsAsStale(replicaRequest, replicationGroup);
            // 关键步骤，写完primary后这里转发请求到replicas
            performOnReplicas(replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes, replicationGroup);
        }
        successfulShards.incrementAndGet();  // mark primary as successful
        decPendingAndFinishIfNeeded();
    }
123456789101112131415
```

下面，我们来看写primary的关键代码，写primary入口函数为TransportShardBulkAction#shardOperationOnPrimary，最后走入到index(…) --> InternalEngine#index()的过程，这是写数据的主要过程。先是通过index获取对应的策略，即plan，通过plan执行对应操作，如要正常写入，则到了indexIntoLucene(…)，然后写translog。如下所示：

```java
    public IndexResult index(Index index) throws IOException {
    			.......
                final IndexResult indexResult;
                if (plan.earlyResultOnPreFlightError.isPresent()) {
                    indexResult = plan.earlyResultOnPreFlightError.get();
                    assert indexResult.getResultType() == Result.Type.FAILURE : indexResult.getResultType();
                } else if (plan.indexIntoLucene || plan.addStaleOpToLucene) {
                	// 将数据写入lucene，最终会调用lucene的文档写入接口
                    indexResult = indexIntoLucene(index, plan);
                } else {
                    indexResult = new IndexResult(
                        plan.versionForIndexing, getPrimaryTerm(), plan.seqNoForIndexing, plan.currentNotFoundOrDeleted);
                }
                if (index.origin().isFromTranslog() == false) {
                    final Translog.Location location;
                    if (indexResult.getResultType() == Result.Type.SUCCESS) {
                        location = translog.add(new Translog.Index(index, indexResult)); //写translog
                    ......
                    indexResult.setTranslogLocation(location);
                }
              .......
        }
12345678910111213141516171819202122
```

ES的写入操作是先写lucene，将数据写入到lucene内存后再写translog。ES之所以先写lucene后写log主要原因大概是写入Lucene时，Lucene会再对数据进行一些检查，有可能出现写入Lucene失败的情况。如果先写translog，那么就要处理写入translog成功但是写入Lucene一直失败的问题，所以ES采用了先写Lucene的方式。

在写完primary后，会继续写replicas，接下来需要将请求转发到从节点上，如果replica shard未分配，则直接忽略；如果replica shard正在搬迁数据到其他节点，则将请求转发到搬迁的目标shard上，否则，转发到replica shard。replicaRequest是在写入主分片后，从primaryResult中获取，并非原始Request。这块代码如下：

```java
    private void performOnReplicas(final ReplicaRequest replicaRequest, final long globalCheckpoint,
                                   final long maxSeqNoOfUpdatesOrDeletes, final ReplicationGroup replicationGroup) {
        totalShards.addAndGet(replicationGroup.getSkippedShards().size());
        final ShardRouting primaryRouting = primary.routingEntry();
        for (final ShardRouting shard : replicationGroup.getReplicationTargets()) {
            if (shard.isSameAllocation(primaryRouting) == false) {
                performOnReplica(shard, replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes);
            }
        }
    }
12345678910
```

performOnReplica方法会将请求转发到目标节点，如果出现异常，如对端节点挂掉、shard写入失败等，对于这些异常，primary认为该replica shard发生故障不可用，将会向master汇报并移除该replica。这块的代码如下：

```java
    private void performOnReplica(final ShardRouting shard, final ReplicaRequest replicaRequest,
                                  final long globalCheckpoint, final long maxSeqNoOfUpdatesOrDeletes) {
        .....
        totalShards.incrementAndGet();
        pendingActions.incrementAndGet();
        replicasProxy.performOn(shard, replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes, new ActionListener<ReplicaResponse>() {
            @Override
            public void onResponse(ReplicaResponse response) {
                successfulShards.incrementAndGet();
                try {
                    primary.updateLocalCheckpointForShard(shard.allocationId().getId(), response.localCheckpoint());
                    primary.updateGlobalCheckpointForShard(shard.allocationId().getId(), response.globalCheckpoint());
                }
                ......
                decPendingAndFinishIfNeeded();
            }
            @Override
            public void onFailure(Exception replicaException) {
                if (TransportActions.isShardNotAvailableException(replicaException) == false) {
                    RestStatus restStatus = ExceptionsHelper.status(replicaException);
                    shardReplicaFailures.add(new ReplicationResponse.ShardInfo.Failure(
                        shard.shardId(), shard.currentNodeId(), replicaException, restStatus, false));
                }
                String message = String.format(Locale.ROOT, "failed to perform %s on replica %s", opType, shard);
                replicasProxy.failShardIfNeeded(shard, message, replicaException,
                    ActionListener.wrap(r -> decPendingAndFinishIfNeeded(), ReplicationOperation.this::onNoLongerPrimary));
            }
        });
    }
1234567891011121314151617181920212223242526272829
```

replica的写入逻辑和primary类似，这里不再具体介绍。

# 源码分析（四）

前面讲到了一个checkpoint（检查点的概念），在每次写入数据过程都需要更新LocalCheckpoint（本地检查点）和GlobalCheckpoint（全局检查点）。

## 3.3 更新checkpoint

> 了解checkpoint之前，先来看下Primary Terms和Sequence Numbers：

**Primary Terms：** 由主节点分配给每个主分片，每次主分片发生变化时递增。主要作用是能够区别新旧两种主分片，只对最新的Terms进行操作。

**Sequence Numbers：** 标记发生在某个分片上的**写操作**。由主分片分配，只对写操作分配。假设索引test有两个主分片一个副本分片，当0号分片的序列号增加到5时，它的主分片离线，副本提升为新的主，对于后续的写操作，序列号从6开启递增。1号分片有自己独立的Sequence Numbers。

主分片在每次向副本转发写请求时，都会带上这两个值。

> 有了Primary Terms和Sequence Numbers，理论上好像就可以检测出分片之间的差异（从旧的主分片删除新的主分片操作历史中不存在的操作，并且将缺少的操作索引到旧主分片），但是当同时为每秒成百上千的事件做索引时，比较数百万个操作的历史是不切实际的，且耗费大量的存储成本，所以ES维护了一个GlobalCheckpoint的安全标记。

> 先来看下checkpoint的概念和作用：

**GlobalCheckpoint：** 全局检查点是所有活跃分片历史都已经对齐的序列号，即所有低于全局检查点的操作都保证已被所有活跃的分片处理完毕。这意味着，当主分片失效时，我们只需要比较新主分片和其他副本分片之间的最后一个全局检查点之后的操作即可。当就主分片恢复时，使用它知道的全局检查点，与新的主分片进行比较。这样，我们只需要进行小部分操作比较，而不是全部。

主分片负责推进全局检查点，它通过跟踪副本上完成的操作来实现。一旦检测到有副本分片已经超出给定序列号，它将相应的更新全局检查点。副本分片不会跟踪所有操作，而是维护一个本地检查点。

**LocalCheckpoint：** 本地检查点也是一个序列号，所有序列号低于它的操作都已在该分片上（写lucene和translog成功）处理完毕。

全局检查点和本地检查点在内存中维护，但也会保存在每个lucene提交的元数据中。

> 我们通过源码来看，写入过程中是如何更新本地检查点和全局检查点的：

主分片写入成功之后，会进行LocalCheckpoint的更新操作，代码入口：ReplicationOperation#execute()->PrimaryShardReference#updateLocalCheckpointForShard(…)->IndexShard#updateLocalCheckpointForShard(…)->ReplicationTracker#updateLocalCheckpoint(…)，代码如下：

```java
        primaryResult = primary.perform(request);  //写主，写lucene和translog
        primary.updateLocalCheckpointForShard(primaryRouting.allocationId().getId(), primary.localCheckpoint()); //更新LocalCheckpoint

    public synchronized void updateLocalCheckpoint(final String allocationId, final long localCheckpoint) {
        .....
        //获取主分片本地的checkpoints,包括LocalCheckpoint和GlobalCheckpoint
        CheckpointState cps = checkpoints.get(allocationId); 
        .....
        // 检查是否需要更新LocalCheckpoint，即需要更新的值是否大于当前已有值
        boolean increasedLocalCheckpoint = updateLocalCheckpoint(allocationId, cps, localCheckpoint);
        // pendingInSync是一个保存等待更新LocalCheckpoint的Set，存放allocation IDs
        boolean pending = pendingInSync.contains(allocationId);
        // 如果是待更新的，且当前的localCheckpoint大于等于GlobalCheckpoint(每次都是先更新Local再Global，正常情况下，Local应该大于等于Global)
        if (pending && cps.localCheckpoint >= getGlobalCheckpoint()) {
        	//从待更新集合中移除
            pendingInSync.remove(allocationId);
            pending = false;
            //此分片是否同步，用于更新GlobalCheckpoint时使用
            cps.inSync = true;
            replicationGroup = calculateReplicationGroup();
            logger.trace("marked [{}] as in-sync", allocationId);
            notifyAllWaiters();
        }
        //更新GlobalCheckpoint
        if (increasedLocalCheckpoint && pending == false) {
            updateGlobalCheckpointOnPrimary();
        }
        assert invariant();
    }
1234567891011121314151617181920212223242526272829
```

继续看是如何更新GlobalCheckpoint的：

```java
    private synchronized void updateGlobalCheckpointOnPrimary() {
        assert primaryMode;
        final CheckpointState cps = checkpoints.get(shardAllocationId);
        final long globalCheckpoint = cps.globalCheckpoint;
        // 计算GlobalCheckpoint，即检验无误后，取Math.min(cps.localCheckpoint, Long.MAX_VALUE)
        final long computedGlobalCheckpoint = computeGlobalCheckpoint(pendingInSync, checkpoints.values(), getGlobalCheckpoint());
        // 需要更新到的GlobalCheckpoint值比当前的global值大，则需要更新
        if (globalCheckpoint != computedGlobalCheckpoint) {
            cps.globalCheckpoint = computedGlobalCheckpoint;
            logger.trace("updated global checkpoint to [{}]", computedGlobalCheckpoint);
            onGlobalCheckpointUpdated.accept(computedGlobalCheckpoint);
        }
    }
12345678910111213
```

主分片的检查点更新完成之后，会向副本分片发送对应的写请求，发送请求时同时传入了globalCheckpoint和SequenceNumbers。

```java
            final long globalCheckpoint = primary.globalCheckpoint();
            final long maxSeqNoOfUpdatesOrDeletes = primary.maxSeqNoOfUpdatesOrDeletes();
            final ReplicationGroup replicationGroup = primary.getReplicationGroup();
            markUnavailableShardsAsStale(replicaRequest, replicationGroup);
            performOnReplicas(replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes, replicationGroup);
12345
```

写副本分片时，进入performOnReplica方法，当监听到分片写入成功之后，则开始更新本地检查点，然后更新全局检查点，更新分方法和之前一样，通过比较当前的检查点是否大于历史检查点，如果是则更新。

```java
        replicasProxy.performOn(shard, replicaRequest, globalCheckpoint, maxSeqNoOfUpdatesOrDeletes, new ActionListener<ReplicaResponse>() {
            @Override
            public void onResponse(ReplicaResponse response) {
                successfulShards.incrementAndGet();
                try {
                	//更新LocalCheckpoint
                    primary.updateLocalCheckpointForShard(shard.allocationId().getId(), response.localCheckpoint());
                    //更新globalCheckpoint
                    primary.updateGlobalCheckpointForShard(shard.allocationId().getId(), response.globalCheckpoint());
                } catch (final AlreadyClosedException e) {
                   ....
                } catch (final Exception e) {
              		....
                }
                decPendingAndFinishIfNeeded();
            }
            @Override
            public void onFailure(Exception replicaException) {
               .....
            }
        });
123456789101112131415161718192021
```

## 总结思考

总体的写入流程源码已经分析完成：

1. 数据可靠性：ES通过副本和Translog保障数据的安全；
2. 服务可用性：在可用性和一致性取舍上，ES更倾向于可用性，只要主分片可用即可执行写入操作；
3. 数据一致性：只要主写入成功，数据就可以被读取，所以查询时操作在主分片和副本分片上可能会得到不同的结果；
4. 原子性：索引的读写、别名操作都是原子操作，不会出现中间状态。但是bulk不是原子操作，不能用来实现事物；