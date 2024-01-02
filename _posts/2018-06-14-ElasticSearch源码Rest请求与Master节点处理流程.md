---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码Rest请求与Master节点处理流程
索引创建、索引删除等操作是由Master节点完成，本文主要讨论Client节点发起索引创建、索引删除等请求，服务端Master节点操作的主流程，与任何具体的操作无关。在ES中，客户端Client和服务端Broker之间的请求称为REST请求，集群内不同Node之间的请求称为RPC请求，REST请求和RPC请求都称为Action。

## Rest请求注册与映射

### ActionMoudle初始化
Node初始化：Node启动初始化操作中，包含ActionModule类的初始化。
```
//绑定Http的TransportActions与节点内部RPC的Actions关系
 ActionModule actionModule = new ActionModule(false, settings, clusterModule.getIndexNameExpressionResolver(),
                settingsModule.getIndexScopedSettings(), settingsModule.getClusterSettings(), settingsModule.getSettingsFilter(),
                threadPool, pluginsService.filterPlugins(ActionPlugin.class), client, circuitBreakerService, usageService, clusterService);
modules.add(actionModule);         
```
其中 new ActionModule 进行ActionModule初始化中，这里比较重要的是两个方法：
```
public ActionModule(......){
    .......................................................
    if (transportClient) {
    //客户端transportClient已经废弃，同时这里是服务端Node启动，可以忽略
    restController = null;
   } else {
    restController = new RestController(headers, restWrapper, nodeClient, circuitBreakerService, usageService);
}
```

### Rest请求注册
ActionModule类初始化完成后，RestHandlers（HTTP handlers）初始化如下：
```
//对REST请求的处理就是定义某个URI应该由哪个模块处理。在ES中, REST请求和RPC请求都称为Action，
// 对REST请求执行处理的类命名规则为Rest* Action。
actionModule.initRestHandlers(() -> clusterService.state().nodes());
```
initRestHandlers 方法中，所有的Rest**Action都是BaseRestHandler的子类或者实现RestHandler接口。
```
//ActionModule类中注册了某个类对某个REST请求的处理，并对外提供这种映射关系，
// 每个REST处理类继承自RestNodesInfoAction。处理类在ActionModule中的注册过程如下：
public void initRestHandlers(Supplier<DiscoveryNodes> nodesInCluster) {
    List<AbstractCatAction> catActions = new ArrayList<>();
    Consumer<RestHandler> registerHandler = a -> {
        if (a instanceof AbstractCatAction) {
            catActions.add((AbstractCatAction) a);
        }
    };
    registerHandler.accept(new RestAddVotingConfigExclusionAction(restController));
    registerHandler.accept(new RestClearVotingConfigExclusionsAction(restController));
    registerHandler.accept(new RestMainAction(restController));
    .......................................................
//插件类REST请求处理Handler注册
for (ActionPlugin plugin : actionPlugins) {
    for (RestHandler handler : plugin.getRestHandlers(settings, restController, clusterSettings, indexScopedSettings,
            settingsFilter, indexNameExpressionResolver, nodesInCluster)) {
        registerHandler.accept(handler);
    }
}
registerHandler.accept(new RestCatAction(restController, catActions));
```
initRestHandlers方法执行完成后，有19个Rest**Action

### RestController注册
以RestAddVotingConfigExclusionAction为例，new RestAddVotingConfigExclusionAction初始化代码
```
public RestAddVotingConfigExclusionAction(RestController controller) {
    controller.registerHandler(RestRequest.Method.POST, "/_cluster/voting_config_exclusions/{node_name}", this);
}
```
把RestAddVotingConfigExclusionAction注册到RestController，对应方法如下：

initRestHandlers，提到的controller.registerHandler。 RestController在Node节点启动时，把所有负责处理的是来自Client的请求Action，服务端Node进行处理的Handler进行注册；

## Master节点请求
MetaDataCreateIndexService（索引创建）、MetaDataDeleteIndexService（索引删除）、MetaDataIndexAliasesService（索引别名）等操作是由Master节点完成。

哪些请求需要Master节点处理？

1.MasterNodeRequest是Master节点处理请求的基类，需要Master节点处理的Request需要继承改类或其子类；
```
/**
 * A based request for master based operation.
 */
public abstract class MasterNodeRequest<Request extends MasterNodeRequest<Request>> extends ActionRequest {
```
2.TransportMasterNodeAction是Master节点处理请求的Action，需要Master节点处理的Action需要继承改类或其子类；
```
/**大部分需要和master节点进行交互的操作全要继承这个Action
 *
 * A base class for operations that needs to be performed on the master node.
 */
public abstract class TransportMasterNodeAction<Request extends MasterNodeRequest<Request>, Response extends ActionResponse>
```
ES的代码风格比较清晰，MasterNodeRequest、TransportMasterNodeAction、MasterNodeOperationRequestBuilder是一一对应关系，其他类型的请求或处理，基本都一致，都有对应的Request/RequestBuilder/NodeAction（如下图示）。

## Master节点处理Client请求过程
在生产环境，创建索引应该由ElasticSearch的维护人员直接连接Master节点进行，并禁止自动创建索引。以客户端发起创建Index，Master节点接收请求并进行处理为例，说明Master节点处理Client请求过程，流程图如下：

### 接收网络请求
Netty4HttpRequestHandler处理来自Client的请求，对应的核心方法：
```
protected void channelRead0(ChannelHandlerContext ctx, HttpPipelinedRequest<FullHttpRequest> msg) {
    .......................................................
    //从HttpPipelinedRequest读取数据
    Netty4HttpRequest httpRequest = new Netty4HttpRequest(copiedRequest, msg.getSequence());
    //若读取失败
    if (request.decoderResult().isFailure()) {
        Throwable cause = request.decoderResult().cause();
        if (cause instanceof Error) {
            ExceptionsHelper.maybeDieOnAnotherThread(cause);
            serverTransport.incomingRequestError(httpRequest, channel, new Exception(cause));
        } else {
            serverTransport.incomingRequestError(httpRequest, channel, (Exception) cause);
        }
    } else {
        //核心是调用dispatchRequest进行处理
        serverTransport.incomingRequest(httpRequest, channel);
    }
}
--------------------------------------------------------------------------------
/**
 * This method handles an incoming http request.
 *
 * @param httpRequest that is incoming
 * @param httpChannel that received the http request
 */
public void incomingRequest(final HttpRequest httpRequest, final HttpChannel httpChannel) {
    handleIncomingRequest(httpRequest, httpChannel, null);
}
--------------------------------------------------------------------------------
private void handleIncomingRequest(final HttpRequest httpRequest, final HttpChannel httpChannel, final Exception exception) {
    final RestRequest restRequest;
    {
        RestRequest innerRestRequest;
        try {
            //验证HttpRequest是否合法，并转换为RestRequest
            innerRestRequest = RestRequest.request(xContentRegistry, httpRequest, httpChannel);
        } 
	.......................................................
        restRequest = innerRestRequest;
    }

    final RestChannel channel;
    {
        RestChannel innerChannel;
        ThreadContext threadContext = threadPool.getThreadContext();
        try {
            //构建DefaultRestChannel
            innerChannel = new DefaultRestChannel(httpChannel, httpRequest, restRequest, bigArrays, handlingSettings, threadContext);
        } 
	.......................................................
        channel = innerChannel;
    }
    //读取数据restRequest，Channel对应DefaultRestChannel，开始分发请求
    dispatchRequest(restRequest, channel, badRequestCause);
}
```

### 网络请求分发
服务端Master节点收到Client请求后，在AbstractHttpServerTransport类 dispatchRequest方法内进行分发（与SpringMVC 的DispatchServlet类似），并由RestController进行处理。
```
void dispatchRequest(final RestRequest restRequest, final RestChannel channel, final Throwable badRequestCause) {
    final ThreadContext threadContext = threadPool.getThreadContext();
    //dispatcher对应: RestController restController = actionModule.getRestController();
    try (ThreadContext.StoredContext ignore = threadContext.stashContext()) {
        if (badRequestCause != null) {
            //异常请求处理
            dispatcher.dispatchBadRequest(channel, threadContext, badRequestCause);
        } else {
            //正常请求处理
            dispatcher.dispatchRequest(restRequest, channel, threadContext);
        }
    }
}
--------------------------------------------------------------------------------
//public方法 发送的是http请求，es也有一套http请求处理逻辑，和spring的mvc类似
// 当Netty收到HTTP请求后，调用dispatchRequest，该方法根据定义好的Action调用对应的Rest*Action处理类。
@Override
public void dispatchRequest(RestRequest request, RestChannel channel, ThreadContext threadContext) {
.......................................................    
    try {
        tryAllHandlers(request, channel, threadContext);
    } catch (Exception e) {
        }
    }
}
--------------------------------------------------------------------------------
//ES对于请求的处理过程。
//在抽象类AbstractHttpServerTransport中做了request和channel的进一步包装，
//然后将请求分发给RestController，在这个类中做了实际的HTTP请求header校验和最重要的部分——URL匹配。
private void tryAllHandlers(final RestRequest request, final RestChannel channel, final ThreadContext threadContext) throws Exception {
.......................................................
    final String rawPath = request.rawPath();
    final String uri = request.uri();
    final RestRequest.Method requestMethod;
    try {
        // Resolves the HTTP method and fails if the method is invalid
        requestMethod = request.method();
        // Loop through all possible handlers, attempting to dispatch the request
        // 首先，根据Request请求url获取所有可能的MethodHandlers
        Iterator<MethodHandlers> allHandlers = getAllHandlers(request.params(), rawPath);
        while (allHandlers.hasNext()) {
            final RestHandler handler;
            //再根据具体Method信息，获取详细的RestHandler（Action）
            final MethodHandlers handlers = allHandlers.next();
            if (handlers == null) {
                handler = null;
            } else {
                handler = handlers.getHandler(requestMethod);
            }

            //开始执行RestHandler（Action）
            if (handler == null) {
              if (handleNoHandlerFound(rawPath, requestMethod, uri, channel)) {
                  return;
              }
            } else {
                //private方法，具体的分发请求
                dispatchRequest(request, channel, handler);
                return;
            }
        }
    } catch (final IllegalArgumentException e) {
        handleUnsupportedHttpMethod(uri, null, channel, getValidHandlerMethodSet(rawPath), e);
        return;
    }
    // If request has not been handled, fallback to a bad request error.
    handleBadRequest(uri, requestMethod, channel);
}
--------------------------------------------------------------------------------
//private方法，具体的分发请求
private void dispatchRequest(RestRequest request, RestChannel channel, RestHandler handler) throws Exception {
.......................................................
    RestChannel responseChannel = channel;
    try {
        //是否可以使用断路器，默认返回true
        if (handler.canTripCircuitBreaker()) {
            inFlightRequestsBreaker(circuitBreakerService).addEstimateBytesAndMaybeBreak(contentLength, "<http_request>");
        } else {
            inFlightRequestsBreaker(circuitBreakerService).addWithoutBreaking(contentLength);
        }
        // if we could reserve bytes for the request we need to send the response also over this channel
        responseChannel = new ResourceHandlingHttpChannel(channel, circuitBreakerService, contentLength);
        // Controller把Request分发给具体的Controller处理,真正处理请求的地方
        handler.handleRequest(request, responseChannel, client);
    } catch (Exception e) {
        responseChannel.sendResponse(new BytesRestResponse(responseChannel, e));
    }
}
```
handler.handleRequest最后调用BaseRestHandler的handleRequest方法。每个URL都与具体的RestAction对应，当匹配上时，就会将请求分发给实际的action来处理。
```
//每个REST请求处理类需要实现一个prepareRequest 函数，用于在收到请求时，对请求执行验证工作等，
// 当一个请求到来时，网络层调用BaseRestHandler#handleRequest。
// 在这个函数中，会调用子类的prepareRequest,然后执行这个Action：
@Override
public final void handleRequest(RestRequest request, RestChannel channel, NodeClient client) throws Exception {
    // prepare the request for execution; has the side effect of touching the request parameters
    // prepareRequest定义了实际处理请求的内容，注意最后返回一个consumer，实际执行是在BaseRestHandler中
    // 调用子类的prepareRequest，（RestChannelConsumer自定义的函数式接口）
    final RestChannelConsumer action = prepareRequest(request, client);

.......................................................
    // execute the action
    // RestChannelConsumer自定义的函数式接口，入参是channel
    //执行子类定义的任务,对Action的具体处理定义在处理类的prepareRequest的Lambda表达式
    action.accept(channel);
}
```
这里有两个重要步骤：

（1）RestChannelConsumer action =prepareRequest(request, client) 调用子类的prepareRequest获取对应的RestChannelConsumer。prepareRequest有子类实现，具体调用的子类，在tryAllHandlers方法内确定。 以创建索引为例，具体的子类是RestCreateIndexAction：
```
public class RestCreateIndexAction extends BaseRestHandler {

    public RestCreateIndexAction(RestController controller) {
        controller.registerHandler(RestRequest.Method.PUT, "/{index}", this);
    }
}
```

RestController对Action的注册，在1.2 initRestHandlers内已经实现。

（2）action.accept(channel) action本质是函数式编程的一个消费者，具体逻辑都在prepareRequest内实现，入参是channel。
```
public RestChannelConsumer prepareRequest(final RestRequest request, final NodeClient client) throws IOException {
.......................................................
    //执行NodeClient的execute方法
    return channel -> client.admin().indices().create(createIndexRequest, new RestToXContentListener<>(channel));
}
其中，client来自ActionModule类初始化，创建Node时完成的。
//创建一个节点客户端 NodeClient
client = new NodeClient(settings, threadPool);
--------------------------------------------------------------------------------
```

### RestAction与TransportAction映射
client.admin().indices().create中create方法调用IndicesAdmin类的create方法，再调用execute方法的入参是 CreateIndexAction.INSTANCE
```
@Override
public void create(final CreateIndexRequest request, final ActionListener<CreateIndexResponse> listener) {
    execute(CreateIndexAction.INSTANCE, request, listener);
}
```
client.admin().indices().create对应的是AbstractClient（NodeClient的父类）的execute方法。
```
public void create(final CreateIndexRequest request, final ActionListener<CreateIndexResponse> listener) {
    execute(CreateIndexAction.INSTANCE, request, listener);
}
--------------------------------------------------------------------------------
/**
 * This is the single execution point of *all* clients.
 */
@Override
public final <Request extends ActionRequest, Response extends ActionResponse> void execute(
    ActionType<Response> action, Request request, ActionListener<Response> listener) {
    listener = threadedWrapper.wrap(listener);
    doExecute(action, request, listener);
}}
--------------------------------------------------------------------------------
//NodeClient的doExecute方法
public <Request extends ActionRequest, Response extends ActionResponse>
void doExecute(ActionType<Response> action, Request request, ActionListener<Response> listener) {
    // Discard the task because the Client interface doesn't use it.
    executeLocally(action, request, listener);
}
--------------------------------------------------------------------------------
public <    Request extends ActionRequest,
            Response extends ActionResponse
        > Task executeLocally(ActionType<Response> action, Request request, ActionListener<Response> listener) {
    return transportAction(action).execute(request, listener);
}
```
transportAction(action)会根据action获取TransportAction的实现类，其中actions参数在Node初始化的时候赋值，详细代码如下：
```
private <    Request extends ActionRequest,
            Response extends ActionResponse
        > TransportAction<Request, Response> transportAction(ActionType<Response> action) {
    if (actions == null) {
        throw new IllegalStateException("NodeClient has not been initialized");
    }
   //根据action获取TransportAction的实现类
    TransportAction<Request, Response> transportAction = actions.get(action);
    if (transportAction == null) {
        throw new IllegalStateException("failed to find action [" + action + "] to execute");
    }
    return transportAction;
}
```
transportAction(action).execute(request, listener);具体execute()方法 ：
```
public final Task execute(Request request, ActionListener<Response> listener) {
    //注册任务
    Task task = taskManager.register("transport", actionName, request);
    //执行任务
    execute(task, request, new ActionListener<Response>() {
        @Override
        public void onResponse(Response response) {
            try {
                taskManager.unregister(task);
            } finally {
                listener.onResponse(response);
            }
        }

        @Override
        public void onFailure(Exception e) {
            try {
                taskManager.unregister(task);
            } finally {
                listener.onFailure(e);
            }
        }
    });
    return task;
}
--------------------------------------------------------------------------------
/**对请求的处理方法execute定义在TransportAction 类中，
 * 它先检测请求的合法性，然后调用Transport*Action中定义的doExecute函数执行真正的RPC处理逻辑。
 *
 * Use this method when the transport action should continue to run in the context of the current task
 */
public final void execute(Task task, Request request, ActionListener<Response> listener) {
    //先检测请求的合法性
    ActionRequestValidationException validationException = request.validate();
    if (validationException != null) {
        listener.onFailure(validationException);
        return;
    }

    if (task != null && request.getShouldStoreResult()) {
        listener = new TaskResultStoringActionListener<>(taskManager, task, listener);
    }

    RequestFilterChain<Request, Response> requestFilterChain = new RequestFilterChain<>(this, logger);
    //调用Action定义的doExecute函数执行用户定义的处理
    requestFilterChain.proceed(task, actionName, request, listener);
}
--------------------------------------------------------------------------------
public void proceed(Task task, String actionName, Request request, ActionListener<Response> listener) {
    int i = index.getAndIncrement();
    try {
        if (i < this.action.filters.length) {
            // 先处理过滤器
            this.action.filters[i].apply(task, actionName, request, listener, this);
        } else if (i == this.action.filters.length) {
            // 执行action操作
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
this.action.doExecute(task, request, listener); 中action对应的是TransportMasterNodeAction。

### TransportMasterNodeAction
Master节点相关操作都在TransportMasterNodeAction中完成，对应的方法是doExecute
```
protected void doExecute(Task task, final Request request, ActionListener<Response> listener) {
    new AsyncSingleAction(task, request, listener).start();
}
--------------------------------------------------------------------------------
然后调用doStart方法
protected void doStart(ClusterState clusterState) {
    try {
        final Predicate<ClusterState> masterChangePredicate = MasterNodeChangePredicate.build(clusterState);
        final DiscoveryNodes nodes = clusterState.nodes();
        /**
         * 协调节点是把请求转发到master node主节点进行处理的。
         * 如果当前节点是master直接内部执行，不是的话就调用网络请求master node
         */
        // 判断当前节点是否是master节点×××
        if (nodes.isLocalNodeElectedMaster() || localExecute(request)) {
            // check for block, if blocked, retry, else, execute locally
            final ClusterBlockException blockException = checkBlock(request, clusterState);
            if (blockException != null) {
                if (!blockException.retryable()) {
                    listener.onFailure(blockException);
                } else {
                    logger.trace("can't execute due to a cluster block, retrying", blockException);
                    retry(blockException, newState -> {
                        try {
                            ClusterBlockException newException = checkBlock(request, newState);
                            return (newException == null || !newException.retryable());
                        } catch (Exception e) {
                            // accept state as block will be rechecked by doStart() and listener.onFailure() then called
                            logger.trace("exception occurred during cluster block checking, accepting state", e);
                            return true;
                        }
                    });
                }
            } else {
                ActionListener<Response> delegate = ActionListener.delegateResponse(listener, (delegatedListener, t) -> {
                    if (t instanceof FailedToCommitClusterStateException || t instanceof NotMasterException) {
                        logger.debug(() -> new ParameterizedMessage("master could not publish cluster state or " +
                            "stepped down before publishing action [{}], scheduling a retry", actionName), t);
                        retry(t, masterChangePredicate);
                    } else {
                        delegatedListener.onFailure(t);
                    }
                });
                //是主节点，本地直接开任务执行
                threadPool.executor(executor)
                    .execute(ActionRunnable.wrap(delegate, l -> masterOperation(task, request, clusterState, l)));
            }
```
这里假设客户端直接请求的Master节点，不用考虑协调者节点把请求转发到Master节点的请情况。

### TransportCreateIndexAction
masterOperation的操作对应的是TransportCreateIndexAction类的实现方法
```
protected void masterOperation(Task task, Request request, ClusterState state, ActionListener<Response> listener) throws Exception {
    masterOperation(request, state, listener);
}
--------------------------------------------------------------------------------
protected void masterOperation(final CreateIndexRequest request, final ClusterState state,
                               final ActionListener<CreateIndexResponse> listener) {
.......................................................
    //根据request获取indexName
    final String indexName = indexNameExpressionResolver.resolveDateMathExpression(request.index());
    //客户端提交创建索引的基本信息（索引名称、分区数、副本数等），提交到服务端，
    //服务端将CreateIndexRequest封装成CreateIndexClusterStateUpdateRequest
    final CreateIndexClusterStateUpdateRequest updateRequest =
        new CreateIndexClusterStateUpdateRequest(cause, indexName, request.index())
            .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
            .settings(request.settings()).mappings(request.mappings())
            .aliases(request.aliases())
            .waitForActiveShards(request.waitForActiveShards());

    createIndexService.createIndex(updateRequest, ActionListener.map(listener, response ->
        new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcknowledged(), indexName)));
}
```
1.客户端发起请求，Master节点通过网络的dispatch找到RestCreateIndexAction；

2.RestCreateIndexAction通过client.create()方法，确定TransportMasterNodeAction（Master节点操作的抽象父类）对应的Master节点操作；

3.最后由TransportCreateIndexAction（索引删除的实现类）对应的具体Master操作执行详细的任务提交；

## Master节点创建索引任务提交的流程

### TransportCreateIndexAction
以创建索引为例，TransportCreateIndexAction是Master节点创建索引的流程，对应的方法入口是masterOperation：
```
protected void masterOperation(final CreateIndexRequest request, final ClusterState state,
                               final ActionListener<CreateIndexResponse> listener) {
.......................................................
    //根据request获取indexName
    final String indexName = indexNameExpressionResolver.resolveDateMathExpression(request.index());
    //客户端提交创建索引的基本信息（索引名称、分区数、副本数等），提交到服务端，
    //服务端将CreateIndexRequest封装成CreateIndexClusterStateUpdateRequest
    final CreateIndexClusterStateUpdateRequest updateRequest =
        new CreateIndexClusterStateUpdateRequest(cause, indexName, request.index())
            .ackTimeout(request.timeout()).masterNodeTimeout(request.masterNodeTimeout())
            .settings(request.settings()).mappings(request.mappings())
            .aliases(request.aliases())
            .waitForActiveShards(request.waitForActiveShards());

    createIndexService.createIndex(updateRequest, ActionListener.map(listener, response ->
        new CreateIndexResponse(response.isAcknowledged(), response.isShardsAcknowledged(), indexName)));
}
--------------------------------------------------------------------------------
public void createIndex(final CreateIndexClusterStateUpdateRequest request,
                        final ActionListener<CreateIndexClusterStateUpdateResponse> listener) {
    // 将创建索引的请求封装为task，然后submit到线程池，外部留了个钩子：listener
    onlyCreateIndex(request, ActionListener.wrap(response -> {
        if (response.isAcknowledged()) {
            activeShardsObserver.waitForActiveShards(new String[]{request.index()}, request.waitForActiveShards(), request.ackTimeout(),
                shardsAcknowledged -> {
                    if (shardsAcknowledged == false) {
                        logger.debug("[{}] index created, but the operation timed out while waiting for " +
                                         "enough shards to be started.", request.index());
                    }
                    listener.onResponse(new CreateIndexClusterStateUpdateResponse(response.isAcknowledged(), shardsAcknowledged));
                }, listener::onFailure);
        } else {
            listener.onResponse(new CreateIndexClusterStateUpdateResponse(false, false));
        }
    }, listener::onFailure));
}
--------------------------------------------------------------------------------
// 发起提交集群状态更新任务
private void onlyCreateIndex(final CreateIndexClusterStateUpdateRequest request,
                             final ActionListener<ClusterStateUpdateResponse> listener) {
    Settings.Builder updatedSettingsBuilder = Settings.builder();
    Settings build = updatedSettingsBuilder.put(request.settings()).normalizePrefix(IndexMetaData.INDEX_SETTING_PREFIX).build();
    indexScopedSettings.validate(build, true); // we do validate here - index setting must be consistent
    request.settings(build);
    //clusterService.submitStateUpdateTask将new IndexCreationTask包装为UpdateTask，
    // 开单线程任务去通过MasterService#runTasks方法去向其他节点分发集群状态信息ClusterState。
    // 发起提交集群状态更新任务
    clusterService.submitStateUpdateTask(
            "create-index [" + request.index() + "], cause [" + request.cause() + "]",
            new IndexCreationTask(
                    logger,
                    allocationService,
                    request,
                    listener,
                    indicesService,
                    aliasValidator,
                    xContentRegistry,
                    settings,
                    //索引名称/索引Settings校验
                    this::validate,
                    indexScopedSettings));
}
```
Master节点的操作，大部分是通过clusterService.submitStateUpdateTask提交任务，并由其他线程去执行。这里会存在一个问题，如果一个请求要创建索引“test”，另一个请求要删除索引“test”,提交任务并由其他线程会出现异常吗？

接下来讨论两个内容

1.submitStateUpdateTask，如何提交任务；

2.IndexCreationTask，提交的具体任务，是如何执行的；

## 任务提交
任务提交处理，提交的是ClusterStateUpdateTask的一个实现类或匿名内部类，最终形成回环，

### ClusterService#submitStateUpdateTask
Trasansport**Action把任务提交到ClusterService服务中，最后调masterService.submitStateUpdateTasks方法执行。
```
/**不能批量提交task
 *
 * Submits a cluster state update task; unlike {@link #submitStateUpdateTask(String, Object, ClusterStateTaskConfig,
 * ClusterStateTaskExecutor, ClusterStateTaskListener)}, submitted updates will not be batched.
 *
 * @param source     the source of the cluster state update task
 * @param updateTask the full context for the cluster state update
 *                   task
 *
 */
public <T extends ClusterStateTaskConfig & ClusterStateTaskExecutor<T> & ClusterStateTaskListener>
    void submitStateUpdateTask(String source, T updateTask) {
    //ClusterStateUpdateTask实现ClusterStateUpdateTask实现ClusterStateTaskConfig, ClusterStateTaskExecutor, ClusterStateTaskListener接口
    // 这里updateTask, updateTask, updateTask, updateTask分别获取对应的子类
    // T updateTask入参需要继承ClusterStateUpdateTask
    submitStateUpdateTask(source, updateTask, updateTask, updateTask, updateTask);
}
--------------------------------------------------------------------------------
/**批量提交任务
 * Submits a batch of cluster state update tasks; submitted updates are guaranteed to be processed together,
 * potentially with more tasks of the same executor.
 */
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
    masterService.submitStateUpdateTasks(source, tasks, config, executor);
}
```

### MasterService#submitStateUpdateTasks
在MasterService类submitStateUpdateTasks方法中，把入参Map<T,ClusterStateTaskListener> tasks又进行了一次封装为Batcher.UpdateTask类型。UpdateTask 是 BatchedTask的子类，BatchedTask继承了SourcePrioritizedRunnable （具有优先级的Runnable）
```
/**各个模块触发的集群状态更新最终在org.elasticsearch.cluster.service.MasterService#submitStateUpdateTasks
 * 方法中构造UpdateTask对象实例，并通过submitTasks方法提交任务执行。
 * 额外需要注意的是：集群状态更新任务可以以批量执行方式提交，具体看org.elasticsearch.cluster.service.TaskBatcher的实现
 *
 * Submits a batch of cluster state update tasks; submitted updates are guaranteed to be processed together,
 * potentially with more tasks of the same executor.
 *
 */
public <T> void submitStateUpdateTasks(final String source,
                                       final Map<T, ClusterStateTaskListener> tasks, final ClusterStateTaskConfig config,
                                       final ClusterStateTaskExecutor<T> executor) {
    final ThreadContext threadContext = threadPool.getThreadContext();
    final Supplier<ThreadContext.StoredContext> supplier = threadContext.newRestorableContext(true);
    //下上下文暂存
    try (ThreadContext.StoredContext ignore = threadContext.stashContext()) {
        threadContext.markAsSystemContext();
        //封装为UpdateTask(抽象类PrioritizedRunnable的实现,能够在PrioritizedEsThreadPoolExecutor中运行)
        List<Batcher.UpdateTask> safeTasks = tasks.entrySet().stream()
            .map(e -> taskBatcher.new UpdateTask(config.priority(), source, e.getKey(), safe(e.getValue(), supplier), executor))
            .collect(Collectors.toList());
        //提交到TaskBatcher
        taskBatcher.submitTasks(safeTasks, config.timeout());
    } catch (EsRejectedExecutionException e) {
}
```
TaskBatcher能够批量提交任务，MasterService的子类Batcher继承了TaskBatcher；

BatchedTask能够添加任务，是TaskBatcher的子类，具体对应关系如下图：

### TaskBatcher#submitTasks
提交任务前，必须确保同一批次的任务有同一个batchingKey，并由同一个Executor执行。提交任务去重的时机有两方面：

提交的任务列表本身的去重。
提交的任务在任务队列tasksPerBatchingKey中已存在，也就是存在尚未执行的相同任务，此时新提交的任务被追加到tasksPerBatchingKey某个k对应的v中。
```
//各个入口的submitStateUpdateTask最终通过TaskBatcher提交任务
public void submitTasks(List<? extends BatchedTask> tasks, @Nullable TimeValue timeout) throws EsRejectedExecutionException {
    if (tasks.isEmpty()) {
        return;
    }
    //获取一个任务. 例如，ClusterStateUpdateTask 对象
    //batchingKey为ClusterStateTaskExecutor
    //BatchedTask: getTask返回完整的task <T extends ClusterStateTaskConfig & ClusterStateTaskExecutor<T> & ClusterStateTas kListener>
    final BatchedTask firstTask = tasks.get(0);
    //如果一次提交多个任务，则必须有相同的batchingKey，这些任务将被批量执行
    assert tasks.stream().allMatch(t -> t.batchingKey == firstTask.batchingKey) :
        "tasks submitted in a batch should share the same batching key: " + tasks;
    // convert to an identity map to check for dups based on task identity
    //Task去重, 将tasks从List转换为Map, key为task对象，例如，ClusterStateUpdateTask,
    //如果提交的任务列表存在重复则抛出异常
    final Map<Object, BatchedTask> tasksIdentity = tasks.stream().collect(Collectors.toMap(
        BatchedTask::getTask,
        Function.identity(),
        (a, b) -> { throw new IllegalStateException("cannot add duplicate task: " + a); },
        IdentityHashMap::new));

    //task转换为LinkedHashSet类型
    //tasksPerBatchingKey在这里添加，在任务执行线程池中,在任务真正开始运行之前“remove"
    //key为batchingKey,值为tasks
    synchronized (tasksPerBatchingKey) {
        //batchingKey对应的是Executor，确保同一批次提交的任务由一个Executor完成
        //如果不存在batchingKey,则添加进去，如果存在则不操作
        //computeIfAbsent返回新添加的k对应的v，或者已存在的k对应的v
        LinkedHashSet<BatchedTask> existingTasks = tasksPerBatchingKey.computeIfAbsent(firstTask.batchingKey,
            k -> new LinkedHashSet<>(tasks.size()));
        for (BatchedTask existing : existingTasks) {
            // check that there won't be two tasks with the same identity for the same batching key
            //一个batchingKey对应的任务列表不可有相同的identity, identity是任务本身，例如，ClusterStateUpdateTask 对象
            BatchedTask duplicateTask = tasksIdentity.get(existing.getTask());
            if (duplicateTask != null) {
                throw new IllegalStateException("task [" + duplicateTask.describeTasks(
                    Collections.singletonList(existing)) + "] with source [" + duplicateTask.source + "] is already queued");
            }
        }
        //添加到tasksPerBatchingKey的value中。如果提交了相同的任务,
        //则新任务被追加到这里
        existingTasks.addAll(tasks);
    }

    //交给线程执行: threadExecutor：MasterService对应的是PrioritizedEsThreadPoolExecutor
    //这里只执行第一个任务firstTask
    if (timeout != null) {
        threadExecutor.execute(firstTask, timeout, () -> onTimeoutInternal(tasks, timeout));
    } else {
        threadExecutor.execute(firstTask);
    }
}
--------------------------------------------------------------------------------
//EsThreadPoolExecutor Override ThreadPoolExecutor 的execute()方法
@Override
public void execute(Runnable command) {
    command = wrapRunnable(command);
    try {
        //在这里面有任务拒绝策略的检查逻辑，如果任务被拒绝了，就会调用EsAbortPolicy的rejectedExecution()
        super.execute(command);
    }
```
最后，由EsThreadPoolExecutor 线程池去执行提交任务。

### BatchedTask#run
提交到线程池的任务，本质是线程，对应的是实现Runnable接口或继承Thread类重写run方法。在threadExecutor.execute(firstTask);中，firstTask是BatchedTask的子类，BatchedTask重写了run方法。
```
@Override
public void run() {
    // 创建索引、更新集群的状态信息是在Runnable#run()中，
    // 也就是TaskBatcher.BatchedTask#run方法中，这个方法继承了java类的Runnable接口
    runIfNotProcessed(this);
}
--------------------------------------------------------------------------------
void runIfNotProcessed(BatchedTask updateTask) {
    // if this task is already processed, it shouldn't execute other tasks with same batching key that arrived later,
    // to give other tasks with different batching key a chance to execute.
    if (updateTask.processed.get() == false) {
        final List<BatchedTask> toExecute = new ArrayList<>();
        final Map<String, List<BatchedTask>> processTasksBySource = new HashMap<>();
        synchronized (tasksPerBatchingKey) {
            //获取一个批次处理的BatchedTask
            LinkedHashSet<BatchedTask> pending = tasksPerBatchingKey.remove(updateTask.batchingKey);
            if (pending != null) {
                for (BatchedTask task : pending) {
                    if (task.processed.getAndSet(true) == false) {
                        logger.trace("will process {}", task);
                        toExecute.add(task);
                        processTasksBySource.computeIfAbsent(task.source, s -> new ArrayList<>()).add(task);
                    } else {
                        logger.trace("skipping {}, already processed", task);
                    }
                }
            }
        }

        //存在需要执行的BatchedTask
        if (toExecute.isEmpty() == false) {
            final String tasksSummary = processTasksBySource.entrySet().stream().map(entry -> {
                String tasks = updateTask.describeTasks(entry.getValue());
                return tasks.isEmpty() ? entry.getKey() : entry.getKey() + "[" + tasks + "]";
            }).reduce((s1, s2) -> s1 + ", " + s2).orElse("");

            // MasterService.Batcher#run实现了上面TaskBatcher#run
            run(updateTask.batchingKey, toExecute, tasksSummary);
        }
    }
}
```

### Batcher#run
run(updateTask.batchingKey, toExecute, tasksSummary);最后调用的是子类Batcher的run方法。
```
// MasterService.Batcher#run实现了上面TaskBatcher#run
@Override
protected void run(Object batchingKey, List<? extends BatchedTask> tasks, String tasksSummary) {
    ClusterStateTaskExecutor<Object> taskExecutor = (ClusterStateTaskExecutor<Object>) batchingKey;
    List<UpdateTask> updateTasks = (List<UpdateTask>) tasks;
    runTasks(new TaskInputs(taskExecutor, updateTasks, tasksSummary));
}
```
其中TaskInputs是Represents a set of tasks to be processed together with their executor，即同一批任务由同一个Executor处理。

### MasterService#runTasks
Master节点执行runTasks，核心是calculateTaskOutputs
```
//最终节点状态更新信息实现逻辑
private void runTasks(TaskInputs taskInputs) {
    final String summary = taskInputs.summary;
    if (!lifecycle.started()) {
        logger.debug("processing [{}]: ignoring, master service not started", summary);
        return;
    }

    logger.debug("executing cluster state update for [{}]", summary);
    //上一次的集群元数据由Coordinator或ZenDiscovery维护
    final ClusterState previousClusterState = state();

    if (!previousClusterState.nodes().isLocalNodeElectedMaster() && taskInputs.runOnlyWhenMaster()) {
        logger.debug("failing [{}]: local node is no longer master", summary);
        taskInputs.onNoLongerMaster();
        return;
    }

    final long computationStartTime = threadPool.relativeTimeInMillis();
    /**
     * 输入创建索引任务，输出集群状态变化结果
     *  改变集群的状态（各个分片的处理逻辑）
     */ 
    final TaskOutputs taskOutputs = calculateTaskOutputs(taskInputs, previousClusterState);
    .......................................................
    }
}
```

### MasterService#calculateTaskOutputs
```
private TaskOutputs calculateTaskOutputs(TaskInputs taskInputs, ClusterState previousClusterState) {
    // 抽象类ClusterStateUpdateTask实现了ClusterStateTaskExecutor接口
    ClusterTasksResult<Object> clusterTasksResult = executeTasks(taskInputs, previousClusterState);
    ClusterState newClusterState = patchVersions(previousClusterState, clusterTasksResult);
    return new TaskOutputs(taskInputs, previousClusterState, newClusterState, getNonFailedTasks(taskInputs, clusterTasksResult),
        clusterTasksResult.executionResults);
}
--------------------------------------------------------------------------------
private ClusterTasksResult<Object> executeTasks(TaskInputs taskInputs, ClusterState previousClusterState) {
    ClusterTasksResult<Object> clusterTasksResult;
    try {
        List<Object> inputs = taskInputs.updateTasks.stream().map(tUpdateTask -> tUpdateTask.task).collect(Collectors.toList());
        // 抽象类ClusterStateUpdateTask实现了ClusterStateTaskExecutor接口
        clusterTasksResult = taskInputs.executor.execute(previousClusterState, inputs);
        if (previousClusterState != clusterTasksResult.resultingState &&
            previousClusterState.nodes().isLocalNodeElectedMaster() &&
            (clusterTasksResult.resultingState.nodes().isLocalNodeElectedMaster() == false)) {
            throw new AssertionError("update task submitted to MasterService cannot remove master");
        }
    }
```

### ClusterStateUpdateTask#execute
execute方法最终调用ClusterStateUpdateTask，若某一个Task继承了ClusterStateUpdateTask，即调用子类的execute方法即可。
```
public final ClusterTasksResult<ClusterStateUpdateTask> execute(ClusterState currentState, List<ClusterStateUpdateTask> tasks)
        throws Exception {
    ClusterState result = execute(currentState);
    return ClusterTasksResult.<ClusterStateUpdateTask>builder().successes(tasks).build(result);
}
```
IndexCreationTask继承了AckedClusterStateUpdateTask，而AckedClusterStateUpdateTask继承了ClusterStateUpdateTask，最后execute调用的是具体子类IndexCreationTask的execute方法。这里回到最初任务提交的代码，形成一个闭环。
```
    clusterService.submitStateUpdateTask(
            "create-index [" + request.index() + "], cause [" + request.cause() + "]",
            new IndexCreationTask(
                    logger,
                    allocationService,
                    request,
                    listener,
                    indicesService,
                    aliasValidator,
                    xContentRegistry,
                    settings,
                    //索引名称/索引Settings校验
                    this::validate,
                    indexScopedSettings));
```
本文主要讨论的TransportCreateIndexAction把任务提交到Master节点，Master节点当前的处理过程。提交的任务可以说批量提交，但是要保证批量提交的任务由同一个Executor处理。































