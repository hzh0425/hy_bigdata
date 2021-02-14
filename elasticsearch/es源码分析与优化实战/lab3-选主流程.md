![image-20210201203809665](https://gitee.com/zisuu/picture/raw/master/img/20210201203809.png)

## 1.设计思想

![image-20210201203902174](https://gitee.com/zisuu/picture/raw/master/img/20210201203902.png)

## 2.为什么使用主从模式

![image-20210201204025030](https://gitee.com/zisuu/picture/raw/master/img/20210201204025.png)

## 3.选举算法

![image-20210201204047888](https://gitee.com/zisuu/picture/raw/master/img/20210201204047.png)

## 4.相关配置

![image-20210201204915738](https://gitee.com/zisuu/picture/raw/master/img/20210201204915.png)

![image-20210201204943991](https://gitee.com/zisuu/picture/raw/master/img/20210201204944.png)

![image-20210201205236711](https://gitee.com/zisuu/picture/raw/master/img/20210201205236.png)



## 5.流程概述

![image-20210201205329642](https://gitee.com/zisuu/picture/raw/master/img/20210201205329.png)

## 6.流程分析

![image-20210201205417527](https://gitee.com/zisuu/picture/raw/master/img/20210201205417.png)

### 1.选举临时master

![image-20210201210114857](https://gitee.com/zisuu/picture/raw/master/img/20210201210115.png)



![image-20210201210127175](https://gitee.com/zisuu/picture/raw/master/img/20210201210127.png)

![image-20210201210138251](https://gitee.com/zisuu/picture/raw/master/img/20210201210138.png)

![image-20210201210145393](https://gitee.com/zisuu/picture/raw/master/img/20210201210145.png)



![image-20210201210154553](https://gitee.com/zisuu/picture/raw/master/img/20210201210154.png)

![image-20210201210206408](https://gitee.com/zisuu/picture/raw/master/img/20210201210206.png)

![image-20210201210220668](https://gitee.com/zisuu/picture/raw/master/img/20210201210220.png)

![image-20210201210226921](https://gitee.com/zisuu/picture/raw/master/img/20210201210227.png)

![image-20210201210232623](https://gitee.com/zisuu/picture/raw/master/img/20210201210232.png)

### 2.投票与得票的实现

![image-20210201211226328](https://gitee.com/zisuu/picture/raw/master/img/20210201211226.png)

### 3.确立master或加入集群

![image-20210201211244281](https://gitee.com/zisuu/picture/raw/master/img/20210201211244.png)

## 7.结点失效监测

![image-20210201211258126](https://gitee.com/zisuu/picture/raw/master/img/20210201211258.png)

### 1.NodesFD

![image-20210201211922213](https://gitee.com/zisuu/picture/raw/master/img/20210201211922.png)

![image-20210201211933874](https://gitee.com/zisuu/picture/raw/master/img/20210201211933.png)



### 2.MasterFD

![](https://gitee.com/zisuu/picture/raw/master/img/20210201212449.png)

## 8.源码分析

选主流程的主要方法都在zenDiscovery和NodeJoinController两个类中

开始于zenDiscovery的innerJoinCluster函数中

```
 /**
     * the main function of a join thread. This function is guaranteed to join the cluster
     * or spawn a new join thread upon failure to do so.
     */
    private void innerJoinCluster() 
```



### 1.临时选主

在innerJoinCluster中调用了findMaster()函数进行临时选主

```
while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
	masterNode = findMaster();
}
```

- 先通过ping获取所有能够ping通的结点,并加入自身

```
List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();



// add our selves
assert fullPingResponses.stream().map(ZenPing.PingResponse::node)
	.filter(n -> n.equals(localNode)).findAny().isPresent() == false;
fullPingResponses.add(new ZenPing.PingResponse(localNode, null, this.clusterState()));
```

- 接着过滤掉不是master 候选者(即没有设置master=true)的结点

```
    调用:
    final List<ZenPing.PingResponse> pingResponses = filterPingResponses(fullPingResponses, masterElectionIgnoreNonMasters, logger);
    
    
    static List<ZenPing.PingResponse> filterPingResponses(List<ZenPing.PingResponse> fullPingResponses,
                                                          boolean masterElectionIgnoreNonMasters, Logger logger) {
                                                          
                                                        
        List<ZenPing.PingResponse> pingResponses;
        //如果设置了忽略ignore_non_master=true,则过滤
        if (masterElectionIgnoreNonMasters) {
            pingResponses = fullPingResponses.stream().filter(ping -> ping.node().isMasterNode()).collect(Collectors.toList());
        } else {
            pingResponses = fullPingResponses;
        }

  
        return pingResponses;
    }
```

- 维护activeMaster列表

```
        List<DiscoveryNode> activeMasters = new ArrayList<>();
        for (ZenPing.PingResponse pingResponse : pingResponses) {
            if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
                activeMasters.add(pingResponse.master());
            }
        }
```

- 维护masterCandidates

```
        // nodes discovered during pinging
        List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
        for (ZenPing.PingResponse pingResponse : pingResponses) {
            if (pingResponse.node().isMasterNode()) {
                masterCandidates.add(...);
            }
        }

```

- 最后选择一个临时master

```
        if (activeMasters.isEmpty()) {
        	//判断是否过半
            if (electMaster.hasEnoughCandidates(masterCandidates)) {
            	//如之前的分析,排序,选版本最大/id最小的一个
                final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
                return winner.getNode();
            } else {
            
                return null;
            }
        } else {
        	//选id最小的一个
            return electMaster.tieBreakActiveMasters(activeMasters);
        }
        
        
    public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
        assert hasEnoughCandidates(candidates);
        List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
        sortedCandidates.sort(MasterCandidate::compare);
        return sortedCandidates.get(0);
    }
    
    
    public static int compare(MasterCandidate c1, MasterCandidate c2) {
            // we explicitly swap c1 and c2 here. the code expects "better" is lower in a sorted
            // list, so if c2 has a higher cluster state version, it needs to come first.
            int ret = Long.compare(c2.clusterStateVersion, c1.clusterStateVersion);
            if (ret == 0) {
                ret = compareNodes(c1.getNode(), c2.getNode());
            }
            return ret;
    }
```

### 2.joinRequest

#### 1.1 若临时master为当前结点

```
        if (transportService.getLocalNode().equals(masterNode)) {
        	//还需要接收的票数=minimumMasterNodes() - 1
            final int requiredJoins = Math.max(0, electMaster.minimumMasterNodes() - 1); 
            nodeJoinController.waitToBeElectedAsMaster(requiredJoins, masterElectionWaitForJoinsTimeout,
                    new NodeJoinController.ElectionCallback() {
                        @Override
                        public void onElectedAsMaster(ClusterState state) {
                            synchronized (stateMutex) {
                                joinThreadControl.markThreadAsDone(currentThread);
                            }
                        }

                        @Override
                        public void onFailure(Throwable t) {
                            logger.trace("failed while waiting for nodes to join, rejoining", t);
                            synchronized (stateMutex) {
                                joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
                            }
                        }
                    }

            );
```

这里调用了nodeJoinController.waitToBeElectedAsMaster,等待其他结点投票当前结点

这个方法理主要构建了一个回调函数,并构建ElectionContext上下文

```
        final CountDownLatch done = new CountDownLatch(1);
        final ElectionCallback wrapperCallback = new ElectionCallback() {
            @Override
            public void onElectedAsMaster(ClusterState state) {
                done.countDown();
                callback.onElectedAsMaster(state);
            }

            @Override
            public void onFailure(Throwable t) {
                done.countDown();
                callback.onFailure(t);
            }
        };
        
       ElectionContext myElectionContext = null;

        try {
            // check what we have so far..
            // capture the context we add the callback to make sure we fail our own
            synchronized (this) {
                myElectionContext = electionContext;
                electionContext.onAttemptToBeElected(requiredMasterJoins, wrapperCallback);
                checkPendingJoinsAndElectIfNeeded();
            }

            try {
                if (done.await(timeValue.millis(), TimeUnit.MILLISECONDS)) {
                    // callback handles everything
                    return;
                }
```

可以发现这里通过一个countDownLatch来控制选举的进度

并通过done.await来等待其他结点的投票

ElectionContext里的数据结构:

```
private ElectionCallback callback = null;
private int requiredMasterJoins = -1;
private final Map<DiscoveryNode, List<MembershipAction.JoinCallback>> joinRequestAccumulator = new HashMap<>();

```



- 当有结点投票当前结点时,会调用zenDiscovery的handleJoinRequest

```
    void handleJoinRequest(final DiscoveryNode node, final ClusterState state, final MembershipAction.JoinCallback callback) {
        if (nodeJoinController == null) {
            throw new IllegalStateException("discovery module is not yet started");
        } else {
            onJoinValidators.stream().forEach(a -> a.accept(node, state));

            transportService.connectToNode(node);

            nodeJoinController.handleJoinRequest(node, callback);
        }
    }
```

并接着调用了nodeJoinController.handleJoinRequest

```
    public synchronized void handleJoinRequest(final DiscoveryNode node, final MembershipAction.JoinCallback callback) {
        if (electionContext != null) {
            electionContext.addIncomingJoin(node, callback);
            checkPendingJoinsAndElectIfNeeded();
        } else {
            masterService.submitStateUpdateTask("zen-disco-node-join",
                node, ClusterStateTaskConfig.build(Priority.URGENT),
                joinTaskExecutor, new JoinTaskListener(callback, logger));
        }
    }
```

addIncomingJoin就是将结点加入到ElectionContext中

checkPendingJoinsAndElectIfNeeded则检查当前总的投票数是否过半了:

```
    private synchronized void checkPendingJoinsAndElectIfNeeded() {
        assert electionContext != null : "election check requested but no active context";
        final int pendingMasterJoins = electionContext.getPendingMasterJoinsCount();
        if (electionContext.isEnoughPendingJoins(pendingMasterJoins) == false) {
             
        } else {
            //过半就关闭选举
            electionContext.closeAndBecomeMaster();
            electionContext = null; // clear this out so future joins won't be accumulated
        }
    }
```

#### 1.2 若为其他结点

```
            // send join request
final boolean success = joinElectedMaster(masterNode);

```

先尝试连接

```
transportService.connectToNode(masterNode);
```

若连接失败,就进行有限次数的尝试,在尝试前会先等待一段时间

```
    private boolean joinElectedMaster(DiscoveryNode masterNode) {
        try {
            transportService.connectToNode(masterNode);
        } catch (Exception e) {
            return false;
        }
        int joinAttempt = 0; // we retry on illegal state if the master is not yet ready
        while (true) {
            try {
                membership.sendJoinRequestBlocking(masterNode, transportService.getLocalNode(), joinTimeout);
                return true;
            } catch (Exception e) {
                final Throwable unwrap = ExceptionsHelper.unwrapCause(e);
                if (unwrap instanceof NotMasterException) {
                    if (++joinAttempt == this.joinRetryAttempts) {
                        return false;
                    } 
                } else {
                    return false;
                }
            }

            try {
                Thread.sleep(this.joinRetryDelay.millis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
```

