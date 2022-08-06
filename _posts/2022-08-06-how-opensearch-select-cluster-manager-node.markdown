---
layout: post
title: "OpenSearch是怎样选取主节点的"
date: 2022-08-06 16:52:14 +0800
categories: 原创
---

> OpenSearch是AWS在ElasticSearch 7.X的基础上开辟的分支。新分支延续了之前的Apache 2.0开源协议，并将独立演进，不再跟随ElasticSearch的节奏。
>
> 延续ElasticSearch的主体架构，OpenSearch依然是一个分布式搜索引擎。整个分布式系统仍然需要一个主节点，来负责管理集群整体的状态。
>
>本文将基于OpenSearch tag：2.1.0的代码（commitID：388c80ad945），简要介绍主节点的选举流程。

## 代码逻辑

OpenSearch的选主流程主要实现在`org.opensearch.cluster.coordination.Coordinator`类中。

当一个OpenSearch实例启动的时候，在`org.opensearch.node.Node`类的`start`方法中会调用`org.opensearch.discovery.Discovery`接口的`startInitialJoin`方法尝试加入集群。

此时，`Node`先将自己设置为`Candidate`角色，然后通过`Coordinator`里的`clusterBootstrapService`生成初次投票配置VotingConfiguration，调用自己的`setInitialConfiguration`方法，并最终触发`startElectionScheduler`方法启动选主流程。`electionScheduler`本质上是一个JVM线程对象，内部会周期性触发如下流程：

```java
@Override
public void run() {
    synchronized (mutex) {
        if (mode == Mode.CANDIDATE) {
            final ClusterState lastAcceptedState = coordinationState.get().getLastAcceptedState();

            if (localNodeMayWinElection(lastAcceptedState) == false) {
                return;
            }

            final StatusInfo statusInfo = nodeHealthService.getHealth();
            if (statusInfo.getStatus() == UNHEALTHY) {
                return;
            }

            if (prevotingRound != null) {
                prevotingRound.close();
            }
            final List<DiscoveryNode> discoveredNodes = getDiscoveredNodes().stream()
                .filter(n -> isZen1Node(n) == false)
                .collect(Collectors.toList());

            prevotingRound = preVoteCollector.start(lastAcceptedState, discoveredNodes);
        }
    }
}
```

其中，`localNodeMayWinElection`方法会先确认本节点有资格参与流程，否则直接退出。核心判断逻辑为，本node之前“在投票参与者中”或者“不在投票排除者中”。

最后一步实质上，是向所有广播节点发送preVoteRequest请求。响应会在`org.opensearch.cluster.coordination.PreVoteCollector`中的`handlePreVoteResponse`方法中进行处理：

```java
private void handlePreVoteResponse(final PreVoteResponse response, final DiscoveryNode sender) {
    ...
    updateMaxTermSeen.accept(response.getCurrentTerm());
    ...
    preVotesReceived.forEach(
        (node, preVoteResponse) -> voteCollection.addJoinVote(...)
    );
    if (electionStrategy.isElectionQuorum(...) == false) {
        ...
        return;
    }
    ...
    startElection.run();
}

```

1. 首先根据响应，不断调整当前所知道的最大term值。
2. 当响应节点数量超过所有节点总数的一般时，也即满足quorum条件，即可开始触发`Coordinator`里的`startElection`开始选举。

```
private void startElection() {
    synchronized (mutex) {
        if (mode == Mode.CANDIDATE) {
            if (localNodeMayWinElection(getLastAcceptedState()) == false) {
                ...
                return;
            }

            final StartJoinRequest startJoinRequest = new StartJoinRequest(getLocalNode(), Math.max(getCurrentTerm(), maxTermSeen) + 1);
            ...
            getDiscoveredNodes().forEach(node -> {
                if (isZen1Node(node) == false) {
                    joinHelper.sendStartJoinRequest(startJoinRequest, node);
                }
            });
        }
    }
}
```

这一段逻辑很简单就是，将已知最大的term值加一，向已知所有节点发送`StartJoinRequest`请求。

**请求终于发送出去了，那么是怎么被处理的呢？**

逻辑在这里`Coordinator`里的`processJoinRequest`方法里：

```java
private void processJoinRequest(JoinRequest joinRequest, JoinHelper.JoinCallback joinCallback) {
    final Optional<Join> optionalJoin = joinRequest.getOptionalJoin();
    synchronized (mutex) {
        updateMaxTermSeen(joinRequest.getTerm());

        final CoordinationState coordState = coordinationState.get();
        final boolean prevElectionWon = coordState.electionWon();

        optionalJoin.ifPresent(this::handleJoin);
        joinAccumulator.handleJoinRequest(joinRequest.getSourceNode(), joinCallback);

        if (prevElectionWon == false && coordState.electionWon()) {
            becomeLeader("handleJoinRequest");
        }
    }
}
```

其核心处理逻辑实现位于`optionalJoin.ifPresent(this::handleJoin)`。对比前后两次自选主请求是否成功，并在必要时调用`becomeLeader`调整自身状态，进入正式的`Cluster Manager`角色中。

而`handleJoin`的内部实质上在调用`org.opensearch.cluster.coordination.CoordinationState`的`handleJoin`方法。

```java
public boolean handleJoin(Join join) {
    if (join.getTerm() != getCurrentTerm()) {
        throw new CoordinationStateRejectedException(...);
    }

    if (startedJoinSinceLastReboot == false) {
        throw new CoordinationStateRejectedException(...);
    }

    final long lastAcceptedTerm = getLastAcceptedTerm();
    if (join.getLastAcceptedTerm() > lastAcceptedTerm) {
        throw new CoordinationStateRejectedException(...);
    }

    if (join.getLastAcceptedTerm() == lastAcceptedTerm && join.getLastAcceptedVersion() > getLastAcceptedVersionOrMetadataVersion()) {
        throw new CoordinationStateRejectedException(...);
    }

    if (getLastAcceptedConfiguration().isEmpty()) {
        throw new CoordinationStateRejectedException(...);
    }

    boolean added = joinVotes.addJoinVote(join);
    boolean prevElectionWon = electionWon;
    electionWon = isElectionQuorum(joinVotes);
    assert !prevElectionWon || electionWon;
    if (electionWon && prevElectionWon == false) {
        lastPublishedVersion = getLastAcceptedVersion();
    }
    return added;
}
```

如上逻辑可见，接受节点会根据请求和自身所知的term以及元数据的version来判断请求的合法性，只有合法请求数量满足quorum的时候，元数据的版本才会按请求更新。

## 总结

前文对OpenSearch的选主请求的发送与处理进行了基本介绍。

尽管网上对OpenSearch的介绍寥寥无几，但是如果去搜索ElasticSearch的相关博客，大多都介绍说ES的选主流程基于Bully算法。然而Bully算法的核心选主逻辑是“依照候选节点的ID排序并选择最大的”。这一点在前面的代码中并没有体现出来。

反而term、version、quorum这样的字眼不断的刺激着眼球，不禁使人联想到：这个算法更为接近**raft选主算法**。

ElasticSearch应该是对选主核心算法做过重大更新。

## 参考资料：

* [CS 551: Distributed Operating Systems - Bully Election Algorithm Example](https://www.cs.colostate.edu/~cs551/CourseNotes/Synchronization/BullyExample.html)
* [Raft算法详解](https://zhuanlan.zhihu.com/p/32052223)

