

## 1.RestController

当一个请求到达时,会先通过restController进行请求转发

在restController中注册了一系列的handler,用于处理不同请求路径的请求

并通过责任链模式进行请求转发

比如对于一个get请求,对应了RestGetActionHandler

```
 public RestGetAction(final Settings settings, final RestController controller) {
        super(settings);
        controller.registerHandler(GET, "/{index}/_doc/{id}", this);
        controller.registerHandler(HEAD, "/{index}/_doc/{id}", this);

        // Deprecated typed endpoints.
        controller.registerHandler(GET, "/{index}/{type}/{id}", this);
        controller.registerHandler(HEAD, "/{index}/{type}/{id}", this);
 }

```

并在启动时注册到了restController中

```
public class ActionModule extends AbstractModule {
  public void initRestHandlers(Supplier<DiscoveryNodes> nodesInCluster) {
    ······
    registerHandler.accept(new RestGetAction(settings, restController));  
    ······  
  }  
}    

```



责任链调用流程如下:

![image-20210209141523276](C:\Users\86180\AppData\Roaming\Typora\typora-user-images\image-20210209141523276.png)

![image-20210209141539355](https://gitee.com/zisuu/picture/raw/master/img/20210209141539.png)

## 2.BaseRestHandler

紧接着请求就会被转发到对应的handler的handleRequest方法上上进行处理

所有的handler都实现了RestHandler,需要实现以下方法:

```
    /**
     * Handles a rest request.
     * @param request The request to handle
     * @param channel The channel to write the request response to
     * @param client A client to use to make internal requests on behalf of the original request
     */
    void handleRequest(RestRequest request, RestChannel channel, NodeClient client) throws Exception;

    default boolean canTripCircuitBreaker() {
        return true;
    }

    /**
     * Indicates if the RestHandler supports content as a stream. A stream would be multiple objects delineated by
     * {@link XContent#streamSeparator()}. If a handler returns true this will affect the types of content that can be sent to
     * this endpoint.
     */
    default boolean supportsContentStream() {
        return false;
    }
```

比如get请求,就转发到RestGetAction,实际上是其父类BaseRestHandler的handleRequest

在BaseRestHandler的handleRequest中,会先调用由子类实现的模板方法prepareRequest进行请求参数的预处理

**RestGetAction#prepareRequest:**

```
    @Override
    public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
        final GetRequest getRequest = new GetRequest(request.param("index"), request.param("type"), request.param("id"));
        getRequest.refresh(request.paramAsBoolean("refresh", getRequest.refresh()));
        getRequest.routing(request.param("routing"));
        getRequest.parent(request.param("parent"));
        getRequest.preference(request.param("preference"));
        getRequest.realtime(request.paramAsBoolean("realtime", getRequest.realtime()));
        if (request.param("fields") != null) {
            throw new IllegalArgumentException("the parameter [fields] is no longer supported, " +
                "please use [stored_fields] to retrieve stored fields or [_source] to load the field from _source");
        }
        final String fieldsParam = request.param("stored_fields");
        if (fieldsParam != null) {
            final String[] fields = Strings.splitStringByCommaToArray(fieldsParam);
            if (fields != null) {
                getRequest.storedFields(fields);
            }
        }

        getRequest.version(RestActions.parseVersion(request));
        getRequest.versionType(VersionType.fromString(request.param("version_type"), getRequest.versionType()));

        getRequest.fetchSourceContext(FetchSourceContext.parseFromRestRequest(request));

        return channel -> client.get(getRequest, new RestToXContentListener<GetResponse>(channel) {
            @Override
            protected RestStatus getStatus(final GetResponse response) {
                return response.isExists() ? OK : NOT_FOUND;
            }
        });
    }
```

这里实际上返回了一个函数式接口,也即最后还是会调用client.get

回到刚刚的

**BaseRestHandler#handleRequest**

在之前的handleRequest方法中获取到这个接口对象，然后进行一些参数和请求体的判断逻辑，然后就要执行这个方法了:

```
public final void handleRequest(RestRequest request, RestChannel channel, NodeClient client) throws Exception {
        // prepare the request for execution; has the side effect of touching the request parameters
        // 获取到了这个接口函数
  			final RestChannelConsumer action = prepareRequest(request, client);

        // validate unconsumed params, but we must exclude params used to format the response
        // use a sorted set so the unconsumed parameters appear in a reliable sorted order
        final SortedSet<String> unconsumedParams =
            request.unconsumedParams().stream().filter(p -> !responseParams().contains(p)).collect(Collectors.toCollection(TreeSet::new));

        // validate the non-response params
        if (!unconsumedParams.isEmpty()) {
            final Set<String> candidateParams = new HashSet<>();
            candidateParams.addAll(request.consumedParams());
            candidateParams.addAll(responseParams());
            throw new IllegalArgumentException(unrecognized(request, unconsumedParams, candidateParams, "parameter"));
        }

        if (request.hasContent() && request.isContentConsumed() == false) {
            throw new IllegalArgumentException("request [" + request.method() + " " + request.path() + "] does not support having a body");
        }

        usageCount.increment();
        // execute the action
  	    // 去执行里面定义好的方法
        action.accept(channel);
    }

```

## 3.NodeClient

上面的client指的是NodeClient

通过debug跟踪,最后会执行NodeClient的executeLocally这个方法:

```
    @Override
    public <    Request extends ActionRequest,
                Response extends ActionResponse,
                RequestBuilder extends ActionRequestBuilder<Request, Response, RequestBuilder>
            > void doExecute(Action<Request, Response, RequestBuilder> action, Request request, ActionListener<Response> listener) {
        // Discard the task because the Client interface doesn't use it.
        executeLocally(action, request, listener);
    }
```

首先获取这个请求对应的transportAction，GET对应的Action是GetAction.INSTANCE -> TransportGetAction, 这个也是在节点启动的时候注册好的：

```
    /**
     * Execute an {@link Action} locally, returning that {@link Task} used to track it, and linking an {@link ActionListener}. Prefer this
     * method if you don't need access to the task when listening for the response. This is the method used to implement the {@link Client}
     * interface.
     */
    public <    Request extends ActionRequest,
                Response extends ActionResponse
            > Task executeLocally(GenericAction<Request, Response> action, Request request, ActionListener<Response> listener) {
        return transportAction(action).execute(request, listener);
    }
```

```
#启动时注册:
actions.register(GetAction.INSTANCE, TransportGetAction.class);
```

## 4.TransportAction

获取到这个Action在去执行它的execute方法，TransportGetAction继承了TransportSingleShardAction，他的execute方法在父类里面org.elasticsearch.action.support.TransportAction#execute，看下具体执行的内容：

```
public final void execute(Task task, Request request, ActionListener<Response> listener) {
        ActionRequestValidationException validationException = request.validate();
        if (validationException != null) {
            listener.onFailure(validationException);
            return;
        }

        if (task != null && request.getShouldStoreResult()) {
            listener = new TaskResultStoringActionListener<>(taskManager, task, listener);
        }
        //定义了一个批处理类，会将有关的一些插件内容在具体执行方法前执行
        RequestFilterChain<Request, Response> requestFilterChain = new RequestFilterChain<>(this, logger);
        requestFilterChain.proceed(task, actionName, request, listener);
}

public void proceed(Task task, String actionName, Request request, ActionListener<Response> listener) {
            int i = index.getAndIncrement();
            try {
                if (i < this.action.filters.length) {
                    //  执行插件逻辑
                    this.action.filters[i].apply(task, actionName, request, listener, this);
                } else if (i == this.action.filters.length) {
                    // 执行action的逻辑
                    this.action.doExecute(task, request, listener);
                } else {
                    listener.onFailure(new IllegalStateException("proceed was called too many times"));
                }
            } catch(Exception e) {
                logger.trace("Error during transport action execution.", e);
                listener.onFailure(e);
            }
}

```

然后就是需要执行Action的doExecute方法了，对与TransportGetAction，它的doExecute在TransportSingleShardAction里面，

```
    @Override
    protected void doExecute(Request request, ActionListener<Response> listener) {
        new AsyncSingleAction(request, listener).start();
    }
```

这里new了一个对象，我们先看下它的构造方法：

```java
private AsyncSingleAction(Request request, ActionListener<Response> listener) {
            ······
            this.shardIt = shards(clusterState, internalRequest);
        }
1234
```

这里比较重要的就是获取的请求的分片信息，这个方法是一个抽象方法，需要继承这个接口的类去实现：

```java
    /**
     * Returns the candidate shards to execute the operation on or <code>null</code> the execute
     * the operation locally (the node that received the request)
     */
		// 返回要在其上执行操作的候选碎片或<code>null</code>在本地执行操作（接收请求的节点）
    @Nullable
    protected abstract ShardsIterator shards(ClusterState state, InternalRequest request);
```

官方注释的意思是如果没有找到分片则本地执行返回null，其他节点执行返回具体需要执行的分片信息。这是一个迭代器，方便失败时在下一个分片执行。构建完对象以后就可以去执行start方法了：

```java
public void start() {
            if (shardIt == null) {
                // just execute it on the local node
                final Writeable.Reader<Response> reader = getResponseReader();
                // 没有找到shard将请求发向本地
                transportService.sendRequest(clusterService.localNode(), transportShardAction, internalRequest.request(),
                    new TransportResponseHandler<Response>() {
                    @Override
                    public Response read(StreamInput in) throws IOException {
                        return reader.read(in);
                    }

                    @Override
                    public String executor() {
                        return ThreadPool.Names.SAME;
                    }

                    @Override
                    public void handleResponse(final Response response) {
                        listener.onResponse(response);
                    }

                    @Override
                    public void handleException(TransportException exp) {
                        listener.onFailure(exp);
                    }
                });
            } else {
                // 向其他节点发送请求
                perform(null);
            }
}
```

再看下perform()方法，失败了会调用onFailure方法：

```java
private void perform(@Nullable final Exception currentFailure) {
            ······
            // 获取routing   失败了会使用下一个分片去请求结果
            final ShardRouting shardRouting = shardIt.nextOrNull();
            ·····
            // 获取routing所在的节点
            DiscoveryNode node = nodes.get(shardRouting.currentNodeId());
            if (node == null) {
                onFailure(shardRouting, new NoShardAvailableActionException(shardRouting.shardId()));
            } else {
                  ······
                }
                final Writeable.Reader<Response> reader = getResponseReader();
                transportService.sendRequest(node, transportShardAction, internalRequest.request(),
                    new TransportResponseHandler<Response>() {
                       ······
                        @Override
                        public void handleException(TransportException exp) {
                            onFailure(shardRouting, exp);
                        }
                });
            }
}

// 失败会一直调用上面这个方法，直到成功或者所有分片都失败
private void onFailure(ShardRouting shardRouting, Exception e) {
            if (e != null) {
                logger.trace(() -> new ParameterizedMessage("{}: failed to execute [{}]", shardRouting,
                    internalRequest.request()), e);
            }
            perform(e);
}
```

## 5.TransportService

1. 下一步需要执行的就是 transportService.sendRequest(node, transportShardAction ·····)，对于transportShardAction这个string，是在TransportGetAction这个类new的时候构造的，看下具体的构造方法：

```java
protected TransportSingleShardAction(String actionName, ThreadPool threadPool, ClusterService clusterService,
                                         TransportService transportService, ActionFilters actionFilters,
                                         IndexNameExpressionResolver indexNameExpressionResolver, 		  
                                         Supplier<Request> request,
                                         String executor) {
        super(actionName, actionFilters, transportService.getTaskManager());
        this.threadPool = threadPool;
        this.clusterService = clusterService;
        this.transportService = transportService;
        this.indexNameExpressionResolver = indexNameExpressionResolver;
				// 在这里定义了这个string，
        this.transportShardAction = actionName + "[s]";
        this.executor = executor;
        // 注册一个使其它client调用的Handler
        if (!isSubAction()) {
            transportService.registerRequestHandler(actionName, request, ThreadPool.Names.SAME, new TransportHandler());
        }
  			// 注册transportShardAction 对应的Handler，也就是上面我们需要执行的Handler
        transportService.registerRequestHandler(transportShardAction, request, ThreadPool.Names.SAME, new ShardTransportHandler());
    }
```

transportService.sendRequest这个方法不管是像本地发送请求，还是像其他节点发送请求，最终都会调用根据这个actionName（transportShardAction）注册的Handler下的messageReceived方法：

```
    // action[s]执行的方法
    private class ShardTransportHandler implements TransportRequestHandler<Request> {

        @Override
        public void messageReceived(final Request request, final TransportChannel channel, Task task) throws Exception {
            if (logger.isTraceEnabled()) {
                logger.trace("executing [{}] on shard [{}]", request, request.internalShardId);
            }
            asyncShardOperation(request, request.internalShardId, new ChannelActionListener<>(channel, transportShardAction, request));
        }
    }
```

最终执行的方法就是继承TransportSingleShardAction类中实现的shardOperation方法：

```
// 这也是一个抽象类，需要具体执行的Action去实现
protected abstract Response shardOperation(Request request, ShardId shardId) throws IOException;
```

所以我们到TransportGetAction类中去找GET请求最终要实现的代码（org.elasticsearch.action.get.TransportGetAction#shardOperation）：

```java
protected GetResponse shardOperation(GetRequest request, ShardId shardId) {
        IndexService indexService = indicesService.indexServiceSafe(shardId.getIndex());
        IndexShard indexShard = indexService.getShard(shardId.id());

        // 如果
        if (request.refresh() && !request.realtime()) {
            indexShard.refresh("refresh_flag_get");
        }
				
        // 带上分文档信息去获取结果
        GetResult result = indexShard.getService().get(request.type(), request.id(), request.storedFields(),
                request.realtime(), request.version(), request.versionType(), request.fetchSourceContext());
        return new GetResponse(result);
}	
```

1. 去获取文档信息的方法最后会执行到org.elasticsearch.index.engine.InternalEngine#get，在这个方法里面执行最终的GET操作：

```java
// 读取文档的具体流程
    public GetResult get(Get get, BiFunction<String, SearcherScope, Searcher> searcherFactory) throws EngineException {
            ·······
            refresh("realtime_get", SearcherScope.INTERNAL, true);
            ······
            // no version, get the version from the index, we know that we refresh on flush
            // 调用searcher读取数据
            // 使用Lucene接口获取文档信息
            return getFromSearcher(get, searcherFactory, scope);
        }
    }
```