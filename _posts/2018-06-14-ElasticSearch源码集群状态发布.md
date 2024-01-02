---
layout: post
categories: [ElasticSearch]
description: none
keywords: ElasticSearch
---
# ElasticSearch源码集群状态发布
Master节点处理完成之后需要通知其他节点，Master节点把元数据信息通知给其他节点是通过publishing ClusterState完成（In ES, the Master node informs other nodes by publishing ClusterState），这个过程认为是集群状态元数据的发布过程

## MasterService
MasterService#runTasks提到 final TaskOutputs taskOutputs = calculateTaskOutputs(taskInputs, previousClusterState);以创建索引为例， taskOutputs对应的是创建索引后的结果。

针对taskOutputs计算结果，首先是处理一下异常任务，判断当前集群状态newClusterState 与上一次的集群状态previousClusterState是否发生变化。
```
/**
 * 输入创建索引任务，输出集群状态变化结果
 *  改变集群的状态（各个分片的处理逻辑）
 */
final TaskOutputs taskOutputs = calculateTaskOutputs(taskInputs, previousClusterState);
//执行异常任务处理
taskOutputs.notifyFailedTasks();
final TimeValue computationTime = getTimeSince(computationStartTime);
logExecutionTime(computationTime, "compute cluster state update", summary);

//将变化了的状态同步给其他节点
if (taskOutputs.clusterStateUnchanged()) {
    //未检测到集群状态信息变化
    final long notificationStartTime = threadPool.relativeTimeInMillis();
    //触发监听器
    taskOutputs.notifySuccessfulTasksOnUnchangedClusterState();
    final TimeValue executionTime = getTimeSince(notificationStartTime);
    logExecutionTime(executionTime, "notify listeners on unchanged cluster state", summary);
} else {
    //集群状态发生变化
    final ClusterState newClusterState = taskOutputs.newClusterState;
    .....................
    final long publicationStartTime = threadPool.relativeTimeInMillis();
    try {
        //分片分配结束后，再由主节点调用相关服务将集群状态cluster state分发到集群的其他节点上，完成集群状态同步；
        // 主节点将集群状态分发到集群中的其他节点上
        ClusterChangedEvent clusterChangedEvent = new ClusterChangedEvent(summary, newClusterState, previousClusterState);
        .....................
        logger.debug("publishing cluster state version [{}]", newClusterState.version());
        //向其他节点 Publish ClusterChangedEvent
        publish(clusterChangedEvent, taskOutputs, publicationStartTime);
    }
}
```
calculateTaskOutputs方法中ClusterTasksResult<Object> clusterTasksResult =executeTasks(taskInputs, previousClusterState);patchVersions(previousClusterState, clusterTasksResult) 根据previousClusterState及任务执行结果clusterTasksResult，计算出newClusterState。
```
private TaskOutputs calculateTaskOutputs(TaskInputs taskInputs, ClusterState previousClusterState) {
    // 抽象类ClusterStateUpdateTask实现了ClusterStateTaskExecutor接口
    ClusterTasksResult<Object> clusterTasksResult = executeTasks(taskInputs, previousClusterState);
    //根据previousClusterState及任务执行结果clusterTasksResult，计算出newClusterState
    ClusterState newClusterState = patchVersions(previousClusterState, clusterTasksResult);
    return new TaskOutputs(taskInputs, previousClusterState, newClusterState, getNonFailedTasks(taskInputs, clusterTasksResult),
        clusterTasksResult.executionResults);
}
```

### notifySuccessfulTasksOnUnchangedClusterState
如果集群状态未发生变更，即当前Master节点操作后，集群元数据保持一致，则执行notifySuccessfulTasksOnUnchangedClusterState方法执行对应的监听器即可。
```
void notifySuccessfulTasksOnUnchangedClusterState() {
    nonFailedTasks.forEach(task -> {
        //ACK监听器
        if (task.listener instanceof AckedClusterStateTaskListener) {
            //no need to wait for ack if nothing changed, the update can be counted as acknowledged
            ((AckedClusterStateTaskListener) task.listener).onAllNodesAcked(null);
        }
        //触发监听器
        task.listener.clusterStateProcessed(task.source(), newClusterState, newClusterState);
    });
}
```

## publish
如果集群状态发生变更，需要通过RPC通知同一集群内的其他节点，Master节点把集群元数据同步给其他Node。publish方法具体实现在Coordinator类（ES7.x以后默认的实现方式）
```
protected void publish(ClusterChangedEvent clusterChangedEvent, TaskOutputs taskOutputs, long startTimeMillis) {
    final PlainActionFuture<Void> fut = new PlainActionFuture<Void>() {
        @Override
        protected boolean blockingAllowed() {
            return isMasterUpdateThread() || super.blockingAllowed();
        }
    };
    //Master节点把数据同步给其他Node，new cluster state, notify all listeners
    clusterStatePublisher.publish(clusterChangedEvent, fut, taskOutputs.createAckListener(threadPool, clusterChangedEvent.state()));
```

## Coordinator
tePublisher接口的实现类，对应的类图如下：

publish方法先判断是否是Leader节点及Term是否匹配，构建publishRequest请求
```
public void publish(ClusterChangedEvent clusterChangedEvent, ActionListener<Void> publishListener, AckListener ackListener) {
    try {
        synchronized (mutex) {
            //不是Master节点或着Term不匹配
            //1. Master 开始发布集群状态时的前期验证，如果clusterState 中的 term不等于当前 term，则取消发布集群状态
            if (mode != Mode.LEADER || getCurrentTerm() != clusterChangedEvent.state().term()) {
                logger.debug(() -> new ParameterizedMessage("[{}] failed publication as node is no longer master for term {}",
                    clusterChangedEvent.source(), clusterChangedEvent.state().term()));
                publishListener.onFailure(new FailedToCommitClusterStateException("node is no longer master for term " +
                    clusterChangedEvent.state().term() + " while handling publication"));
                return;
            }
            .....................
            final ClusterState clusterState = clusterChangedEvent.state();

            final PublicationTransportHandler.PublicationContext publicationContext =
                //newPublicationContext
                publicationHandler.newPublicationContext(clusterChangedEvent);

            //1. 发布前参数校验（包含Term/Version等参数）
            final PublishRequest publishRequest = coordinationState.get().handleClientValue(clusterState);
            //构建协调者发布
            final CoordinatorPublication publication = new CoordinatorPublication(publishRequest, publicationContext,
                new ListenableFuture<>(), ackListener, publishListener);
            currentPublication = Optional.of(publication);

           // 获取目标发布节点
           final DiscoveryNodes publishNodes = publishRequest.getAcceptedState().nodes();
           //leader检测,在数据节点上用于追踪 master，和 master 保持心跳
           leaderChecker.setCurrentNodes(publishNodes);
           //fllower检测,设置当前 leader 需要追踪的 follower 节点，主要是用于心跳保持
           followersChecker.setCurrentNodes(publishNodes);
           //lag 探测器，如果某个节点超过指定时间默认 90s 没有 publish 成功则踢出
           lagDetector.setTrackedNodes(publishNodes);
           //开始发布任务,启动 publish 任务
           publication.start(followersChecker.getFaultyNodes());
        }
    }
```
publishNodes 是集群状态ClusterState的属性，ClusterState来自MasterService类runTasks方法内new ClusterChangedEvent(summary, newClusterState, previousClusterState);

## CoordinatorPublication
CoordinatorPublication继承抽象类Publication
```
public void start(Set<DiscoveryNode> faultyNodes) {
    logger.trace("publishing {} to {}", publishRequest, publicationTargets);

    //处理错误节点
    for (final DiscoveryNode faultyNode : faultyNodes) {
        onFaultyNode(faultyNode);
    }
    //commit请求处理
    onPossibleCommitFailure();
    //遍历正常节点并发送,依次发送给各节点
    publicationTargets.forEach(PublicationTarget::sendPublishRequest);
}
---------------------------------------------------------------------
void sendPublishRequest() {
    if (isFailed()) {
        return;
    }
    assert state == PublicationTargetState.NOT_STARTED : state + " -> " + PublicationTargetState.SENT_PUBLISH_REQUEST;
    state = PublicationTargetState.SENT_PUBLISH_REQUEST;
    //discoveryNode 发送节点；publishRequest 发送请求；PublishResponseHandler 发送结果处理
    Publication.this.sendPublishRequest(discoveryNode, publishRequest, new PublishResponseHandler());
```
sendPublishRequest方法参数，PublishResponseHandler对应的是监听返回结果的处理器。
```
protected void sendPublishRequest(DiscoveryNode destination, PublishRequest publishRequest,
                                  ActionListener<PublishWithJoinResponse> responseActionListener) {
    publicationContext.sendPublishRequest(destination, publishRequest, wrapWithMutex(responseActionListener));
}
```

## 发送Publish请求sendPublishRequest
集群元数据发布是两阶段提交，核心是publish和commit两个请求，涉及流程如下图：

sendPublishRequest节点超过一半响应成功后，会进入Commit请求阶段，一旦进入Commit请求阶段即使异常也不会回滚。

## Master节点发送Publish请求
Publish请求对应流程如下图：
```
protected void sendPublishRequest(DiscoveryNode destination, PublishRequest publishRequest,
                                  ActionListener<PublishWithJoinResponse> responseActionListener) {
    publicationContext.sendPublishRequest(destination, publishRequest, wrapWithMutex(responseActionListener));
}
```
sendPublishRequest对应的是PublicationTransportHandler方法
```
public void sendPublishRequest(DiscoveryNode destination, PublishRequest publishRequest,
                               ActionListener<PublishWithJoinResponse> originalListener) {
    assert publishRequest.getAcceptedState() == clusterChangedEvent.state() : "state got switched on us";
    //
    final ActionListener<PublishWithJoinResponse> responseActionListener;
    if (destination.equals(nodes.getLocalNode())) {
        //发送给当前节点
        // if publishing to self, use original request instead (see currentPublishRequestToSelf for explanation)
        final PublishRequest previousRequest = currentPublishRequestToSelf.getAndSet(publishRequest);
        // we might override an in-flight publication to self in case where we failed as master and became master again,
        // and the new publication started before the previous one completed (which fails anyhow because of higher current term)
        assert previousRequest == null || previousRequest.getAcceptedState().term() < publishRequest.getAcceptedState().term();
        responseActionListener = new ActionListener<PublishWithJoinResponse>() {
            @Override
            public void onResponse(PublishWithJoinResponse publishWithJoinResponse) {
                currentPublishRequestToSelf.compareAndSet(publishRequest, null); // only clean-up our mess
                originalListener.onResponse(publishWithJoinResponse);
            }

            @Override
            public void onFailure(Exception e) {
                currentPublishRequestToSelf.compareAndSet(publishRequest, null); // only clean-up our mess
                originalListener.onFailure(e);
            }
        };
    } else {
        responseActionListener = originalListener;
    }

    if (sendFullVersion || !previousState.nodes().nodeExists(destination)) {
        //发送全部数据
        logger.trace("sending full cluster state version {} to {}", newState.version(), destination);
        PublicationTransportHandler.this.sendFullClusterState(newState, serializedStates, destination, responseActionListener);
    } else {
        //只发送变更的数据
        logger.trace("sending cluster state diff for version {} to {}", newState.version(), destination);
        PublicationTransportHandler.this.sendClusterStateDiff(newState, serializedDiffs, serializedStates, destination,
            responseActionListener);
    }
}
```
论发送全量还是增量内容，最终都通过PublishClusterStateAction#sendClusterStateToNode实现发送。
```
private void sendClusterStateToNode(ClusterState clusterState, BytesReference bytes, DiscoveryNode node,
                                    ActionListener<PublishWithJoinResponse> responseActionListener, boolean sendDiffs,
                                    Map<Version, BytesReference> serializedStates) {
    try {
        ..............................
        final String actionName;
        //RPC回调处理器
        final TransportResponseHandler<?> transportResponseHandler;
        //7.X之前
        if (Coordinator.isZen1Node(node)) {
            actionName = PublishClusterStateAction.SEND_ACTION_NAME;
            transportResponseHandler = publishWithJoinResponseHandler.wrap(empty -> new PublishWithJoinResponse(
                new PublishResponse(clusterState.term(), clusterState.version()),
                Optional.of(new Join(node, transportService.getLocalNode(), clusterState.term(), clusterState.term(),
                    clusterState.version()))), in -> TransportResponse.Empty.INSTANCE);
        } else {
            //7.X之后
            actionName = PUBLISH_STATE_ACTION_NAME;
            transportResponseHandler = publishWithJoinResponseHandler;
        }
        //actionName: PUBLISH_STATE_ACTION_NAME
        transportService.sendRequest(node, actionName, request, stateRequestOptions, transportResponseHandler);
    }
```
transportService.sendRequest涉及RPC请求发送

## 远程节点处理Publish请求
RequestHandlerRegistry类的processMessageReceived方法
```
public void processMessageReceived(Request request, TransportChannel channel) throws Exception {
    //本地Node任务暂存
    final Task task = taskManager.register(channel.getChannelType(), action, request);
    boolean success = false;
    try {
        handler.messageReceived(request, new TaskTransportChannel(taskManager, task, channel), task);
        success = true;
    }
```
远程节点返回内容处理逻辑：

messageReceived调用RPC请求对应的Handler，对应Action是PUBLISH_STATE_ACTION_NAME，调用channel.sendResponse方法
```
(request, channel, task) -> channel.sendResponse(handleIncomingPublishRequest(request))
```
handleIncomingPublishRequest(request)返回的是远程节点处理PUBLISH_STATE_ACTION_NAME（Action名称）的结果。
```
public void sendResponse(TransportResponse response) throws IOException {
    endTask();
    channel.sendResponse(response);
}
----------------------------------------------
private void endTask() {
    taskManager.unregister(task);
}
```
远程节点具体处理逻辑方法是handleIncomingPublishRequest
```
private PublishWithJoinResponse handleIncomingPublishRequest(BytesTransportRequest request) throws IOException {
    final Compressor compressor = CompressorFactory.compressor(request.bytes());
    StreamInput in = request.bytes().streamInput();
    try {
        if (compressor != null) {
            in = compressor.streamInput(in);
        }
        in = new NamedWriteableAwareStreamInput(in, namedWriteableRegistry);
        in.setVersion(request.version());
        // If true we received full cluster state - otherwise diffs
        if (in.readBoolean()) {
            //接收全部集群状态
            final ClusterState incomingState;
            try {
                incomingState = ClusterState.readFrom(in, transportService.getLocalNode());
            } catch (Exception e){
                logger.warn("unexpected error while deserializing an incoming cluster state", e);
                throw e;
            }
            fullClusterStateReceivedCount.incrementAndGet();
            logger.debug("received full cluster state version [{}] with size [{}]", incomingState.version(),
                request.bytes().length());
            final PublishWithJoinResponse response = acceptState(incomingState);
            lastSeenClusterState.set(incomingState);
            return response;
        } else {
            //接收变更的集群状态
            final ClusterState lastSeen = lastSeenClusterState.get();
            if (lastSeen == null) {
                logger.debug("received diff for but don't have any local cluster state - requesting full state");
                incompatibleClusterStateDiffReceivedCount.incrementAndGet();
                throw new IncompatibleClusterStateVersionException("have no local cluster state");
            } else {
                ClusterState incomingState;
                try {
                    Diff<ClusterState> diff = ClusterState.readDiffFrom(in, lastSeen.nodes().getLocalNode());
                    incomingState = diff.apply(lastSeen); // might throw IncompatibleClusterStateVersionException
                } catch (IncompatibleClusterStateVersionException e) {
                .................................
                compatibleClusterStateDiffReceivedCount.incrementAndGet();
                logger.debug("received diff cluster state version [{}] with uuid [{}], diff size [{}]",
                    incomingState.version(), incomingState.stateUUID(), request.bytes().length());
                final PublishWithJoinResponse response = acceptState(incomingState);
                lastSeenClusterState.compareAndSet(lastSeen, incomingState);
                return response;
}
```

## Master节点监听远程节点Publish返回结果并处理
transportResponseHandler是监听结果处理器
```
private class PublishResponseHandler implements ActionListener<PublishWithJoinResponse> {

    @Override
    public void onResponse(PublishWithJoinResponse response) {
        //这个 response 中可能含有 join 请求 PublishResponseHandler：
        if (response.getJoin().isPresent()) {
            final Join join = response.getJoin().get();
            assert discoveryNode.equals(join.getSourceNode());
            assert join.getTerm() == response.getPublishResponse().getTerm() : response;
            logger.trace("handling join within publish response: {}", join);
            onJoin(join);
        } else {
            logger.trace("publish response from {} contained no join", discoveryNode);
            onMissingJoin(discoveryNode);
        }

        assert state == PublicationTargetState.SENT_PUBLISH_REQUEST : state + " -> " + PublicationTargetState.WAITING_FOR_QUORUM;
        state = PublicationTargetState.WAITING_FOR_QUORUM;
        handlePublishResponse(response.getPublishResponse());
    }
```
handlePublishResponse方法内，如果sendPublishRequest节点超过一半完成并成功则返回Optional.of(newApplyCommitRequest）对象，然后调用sendApplyCommit 方法。
```
void handlePublishResponse(PublishResponse publishResponse) {
    assert isWaitingForQuorum() : this;
    logger.trace("handlePublishResponse: handling [{}] from [{}])", publishResponse, discoveryNode);
    if (applyCommitRequest.isPresent()) {
        sendApplyCommit();
    } else {
        try {
            Publication.this.handlePublishResponse(discoveryNode, publishResponse).ifPresent(applyCommit -> {
                assert applyCommitRequest.isPresent() == false;
                applyCommitRequest = Optional.of(applyCommit);
                ackListener.onCommit(TimeValue.timeValueMillis(currentTimeSupplier.getAsLong() - startTime));
                publicationTargets.stream().filter(PublicationTarget::isWaitingForQuorum)
                    //发送ApplyCommit
                    .forEach(PublicationTarget::sendApplyCommit);
            });
        }
```
handlePublishResponse方法对应源码如下：
```
protected Optional<ApplyCommitRequest> handlePublishResponse(DiscoveryNode sourceNode,
                                                             PublishResponse publishResponse) {
    assert Thread.holdsLock(mutex) : "Coordinator mutex not held";
    assert getCurrentTerm() >= publishResponse.getTerm();
    return coordinationState.get().handlePublishResponse(sourceNode, publishResponse);
}
---------------------------------------------------------
public Optional<ApplyCommitRequest> handlePublishResponse(DiscoveryNode sourceNode, PublishResponse publishResponse) {
    ...........................
    publishVotes.addVote(sourceNode);
    //判断是否超过一半的节点已经完成
    if (isPublishQuorum(publishVotes)) {
        logger.trace("handlePublishResponse: value committed for version [{}] and term [{}]",
            publishResponse.getVersion(), publishResponse.getTerm());
        return Optional.of(new ApplyCommitRequest(localNode, publishResponse.getTerm(), publishResponse.getVersion()));
    }

    return Optional.empty();
}
```

## 发送Commit请求sendApplyCommit
发送commit请求对应流程图
```
void sendApplyCommit() {
    assert state == PublicationTargetState.WAITING_FOR_QUORUM : state + " -> " + PublicationTargetState.SENT_APPLY_COMMIT;
    state = PublicationTargetState.SENT_APPLY_COMMIT;
    assert applyCommitRequest.isPresent();
    Publication.this.sendApplyCommit(discoveryNode, applyCommitRequest.get(), new ApplyCommitResponseHandler());
    assert publicationCompletedIffAllTargetsInactiveOrCancelled();
}
```

Master节点发送Commit请求
```
protected void sendApplyCommit(DiscoveryNode destination, ApplyCommitRequest applyCommit,
                               ActionListener<Empty> responseActionListener) {
    publicationContext.sendApplyCommit(destination, applyCommit, wrapWithMutex(responseActionListener));
}
```
调用接口如下：
```
public void sendApplyCommit(DiscoveryNode destination, ApplyCommitRequest applyCommitRequest,
                            ActionListener<TransportResponse.Empty> responseActionListener) {
    final String actionName;
    final TransportRequest transportRequest;
    //7.X之前
    if (Coordinator.isZen1Node(destination)) {
        actionName = PublishClusterStateAction.COMMIT_ACTION_NAME;
        transportRequest = new PublishClusterStateAction.CommitClusterStateRequest(newState.stateUUID());
    } else {
        //7.X之后
        actionName = COMMIT_STATE_ACTION_NAME;
        transportRequest = applyCommitRequest;
    }
    //actionName：COMMIT_STATE_ACTION_NAME
    transportService.sendRequest(destination, actionName, transportRequest, stateRequestOptions,
        new TransportResponseHandler<TransportResponse.Empty>() {

            @Override
            public TransportResponse.Empty read(StreamInput in) {
                return TransportResponse.Empty.INSTANCE;
            }

            @Override
            public void handleResponse(TransportResponse.Empty response) {
                responseActionListener.onResponse(response);
            }

            @Override
            public void handleException(TransportException exp) {
                responseActionListener.onFailure(exp);
            }

            @Override
            public String executor() {
                return ThreadPool.Names.GENERIC;
            }
        });
}
```

## 远程节点处理Commit请求
RequestHandlerRegistry类的processMessageReceived方法
```
public void processMessageReceived(Request request, TransportChannel channel) throws Exception {
    //本地Node任务暂存
    final Task task = taskManager.register(channel.getChannelType(), action, request);
    boolean success = false;
    try {
        handler.messageReceived(request, new TaskTransportChannel(taskManager, task, channel), task);
        success = true;
    } finally {
}
```
远程节点返回内容处理逻辑：

messageReceived调用RPC请求对应的Handler，对应Action是COMMIT_STATE_ACTION_NAME，handleApplyCommit的accept调用Coordinator#handleApplyCommit
```
(request, channel, task) -> handleApplyCommit.accept(request, transportCommitCallback(channel)));
```
远程节点具体处理逻辑方法是transportCommitCallback，逻辑比较简单只是增加一个监听器，把handleApplyCommit.accept结果通过channel返回给Master节点。
```
private ActionListener<Void> transportCommitCallback(TransportChannel channel) {
    return new ActionListener<Void>() {

        @Override
        public void onResponse(Void aVoid) {
            try {
                //commit 成功只用返回一个TransportResponse空对象
                channel.sendResponse(TransportResponse.Empty.INSTANCE);
            } catch (IOException e) {
                logger.debug("failed to send response on commit", e);
            }
        }

        @Override
        public void onFailure(Exception e) {
            try {
                channel.sendResponse(e);
            } catch (IOException ie) {
                e.addSuppressed(ie);
                logger.debug("failed to send response on commit", e);
            }
    };
}
```
handleApplyCommit的accept调用Coordinator#handleApplyCommit
```
private void handleApplyCommit(ApplyCommitRequest applyCommitRequest, ActionListener<Void> applyListener) {
    synchronized (mutex) {
        logger.trace("handleApplyCommit: applying commit {}", applyCommitRequest);
        // 将我们之前接收到的 acceptedState 的元数据改为 commit 状态
        coordinationState.get().handleCommit(applyCommitRequest);
        final ClusterState committedState = hideStateIfNotRecovered(coordinationState.get().getLastAcceptedState());
        applierState = mode == Mode.CANDIDATE ? clusterStateWithNoMasterBlock(committedState) : committedState;
        if (applyCommitRequest.getSourceNode().equals(getLocalNode())) {
            // master node applies the committed state at the end of the publication process, not here.
            // Master节点最后commit
            applyListener.onResponse(null);
        } else {
            // 元数据接收完毕，进入应用环节,数据节点 commit 元数据之后，元数据的发布流程就完毕了，之后数据节点进入异步应用环节
            clusterApplier.onNewClusterState(applyCommitRequest.toString(), () -> applierState,
                new ClusterApplyListener() {

                    @Override
                    public void onFailure(String source, Exception e) {
                        applyListener.onFailure(e);
                    }

                    @Override
                    public void onSuccess(String source) {
                        applyListener.onResponse(null);
                    }
                });
        }
    }
```
handleCommit方法是数据节点处理commit请求
```
/**数据节点处理 commit 请求
 * May be called on receipt of an ApplyCommitRequest. Updates the committed configuration accordingly.
 *
 * @param applyCommit The ApplyCommitRequest received.
 * @throws CoordinationStateRejectedException if the arguments were incompatible with the current state of this object.
 */
public void handleCommit(ApplyCommitRequest applyCommit) {
    ...................

    persistedState.markLastAcceptedStateAsCommitted();
    assert getLastCommittedConfiguration().equals(getLastAcceptedConfiguration());
}
```

Master节点监听远程节点Commit请求返回结果并处理
ApplyCommitResponseHandler
```
private class ApplyCommitResponseHandler implements ActionListener<TransportResponse.Empty> {

    @Override
    public void onResponse(TransportResponse.Empty ignored) {
        if (isFailed()) {
            logger.debug("ApplyCommitResponseHandler.handleResponse: already failed, ignoring response from [{}]",
                discoveryNode);
            return;
        }
        setAppliedCommit();
        onPossibleCompletion();
        assert publicationCompletedIffAllTargetsInactiveOrCancelled();
    }
```