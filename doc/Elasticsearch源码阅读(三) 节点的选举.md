## 简介

```
继上篇文章调试源码环境以后，本文主要讲解Elasticsearch在启动过程中进行节点选举部分。
```

## 环境介绍

```
 系统：Mac os
 Elasticsearch6.1.2
```

##Discovery实现类

```
...
org.elasticsearch.node.Node#start

此处的Discovery实例主要是根据配置文件生成相应的Discovery。
org.elasticsearch.discovery.DiscoveryModule中会根据配置文件的配置生成具体的Discovery的实现类。
在此处我的配置为：discovery.type： zen.

Discovery discovery = injector.getInstance(Discovery.class);//此处根据配置生成的实现类为：org.elasticsearch.discovery.zen.ZenDiscovery
```

## Discovery#startInitialJoin->innerJoinCluster 方法

```
 while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
      masterNode = findMaster();//寻找master节点。入口方法，直到找到masterNode。
 }
```

## ZenDiscovery#findMaster()方法

```
//ping所有的节点地址，在上文中已经说到支持的形式比如域名等的支持。得到返回结果
List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();
if (fullPingResponses == null) {
   logger.trace("No full ping responses");
   return null;
}


//当前可能的master节点（注意这里的activeMasters并不是在配置文件中配置的master:true, 而是其他节点选举出来的master节点，其中可能会有不同的节点选举不同的master节点，所以这里是列表）。
List<DiscoveryNode> activeMasters = new ArrayList<>();
for (ZenPing.PingResponse pingResponse : pingResponses) {         
   if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
          activeMasters.add(pingResponse.master());
   }
}

//这里的masterCandidates指的是在配置文件中有资格成为master节点的候选节点，下面会用到。
List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
for (ZenPing.PingResponse pingResponse : pingResponses) {
    if (pingResponse.node().isMasterNode()) {
    masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
    }
}

//这里有两种情况，response中没有发现活跃的master节点和有活跃的被选举出来的master节点。
if (activeMasters.isEmpty()) {
   
   //如果活跃的或者说已经被选举出来的master列表为空，此处判断是否有足够的候选者节点。配置项：discovery.zen.minimum_master_nodes
   if (electMaster.hasEnoughCandidates(masterCandidates)) {
   	  //如果有多个则进入electMaster方法。下面有关于electMaster方法。
      final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
       logger.trace("candidate {} won election", winner);
       return winner.getNode();
   } else {
       ....
       如果没有则直接返回null， 之后还会回到上层的while 循环里面继续找master节点。
       return null;
            }
   } else {
       .....
       //如果存在活跃的被选举出来的master节点则进入tieBreakActiveMasters方法。请查看下面的tieBreakActiveMasters简介。
       return electMaster.tieBreakActiveMasters(activeMasters);
}
```

###electMaster方法

```
public MasterCandidate electMaster(Collection<MasterCandidate> candidates) {
    assert hasEnoughCandidates(candidates);
    List<MasterCandidate> sortedCandidates = new ArrayList<>(candidates);
    sortedCandidates.sort(MasterCandidate::compare);
    return sortedCandidates.get(0);
}

要注意的是此刻的compare方法其实并不是对比的各个节点的ID大小，而是对比的是clusterStateVersion。
在初始化version的时候是通过UnicastPingRequest.response初始化进去的。大家可以看下org.elasticsearch.discovery.zen.UnicastZenPing#sendPings方法。

备注：在下面的ClusterState中会进行详细的说明。


//根据每个节点的clusterStateVersion状态来决策master节点的选举。后面说明。
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

####ZenDiscovery#doStart()->ClusterState

```
此方法主要作用是初始化ClusterState，用于在没有被选举的master节点的情况下根据clasterStateVersion来选举主节点。

ClusterState initialState = builder.blocks(ClusterBlocks.builder().addGlobalBlock(STATE_NOT_RECOVERED_BLOCK).addGlobalBlock(discoverySettings.getNoMasterBlock())).nodes(DiscoveryNodes.builder().add(localNode).localNodeId(localNode.getId())).build();

其中的STATE_NOT_RECOVERED_BLOCK， discoverySettings.getNoMasterBlock()的意思分别为：

大家注意看最后一个参数ClusterBlockLevel.ALL
这药说明一下：

当集群不能产生一个master的时候，discovery.zen.no_master_block 用来配置操作的策略，它有两个配置
all (节点上的所有读写操作都被拒绝，这也包括了集群api状态的读写操作)
write(写操作会被拒绝，读操作能成功)
discovery.zen.no_master_block 配置的设置不影响节点基本的api,如节点信息和节点统计api


STATE_NOT_RECOVERED_BLOCK = ClusterBlock(1, "state not recovered / initialized", true, true, false, RestStatus.SERVICE_UNAVAILABLE, ClusterBlockLevel.ALL);

ClusterBlock NO_MASTER_BLOCK_ALL = new ClusterBlock(NO_MASTER_BLOCK_ID, "no master", true, true, false, RestStatus.SERVICE_UNAVAILABLE, ClusterBlockLevel.ALL);

ClusterBlock NO_MASTER_BLOCK_WRITES = new ClusterBlock(NO_MASTER_BLOCK_ID, "no master", true, false, false, RestStatus.SERVICE_UNAVAILABLE, EnumSet.of(ClusterBlockLevel.WRITE, ClusterBlockLevel.METADATA_WRITE));

这个时候可以看出来原来version的stage的状态有1和2.

committedState.set(initialState);
```

##ElectMasterService#tieBreakActiveMasters

```
//此时才会根据每个节点的ID比较大小， 然后取ID最小的。
public DiscoveryNode tieBreakActiveMasters(Collection<DiscoveryNode> activeMasters) {
    return activeMasters.stream().min(ElectMasterService::compareNodes).get();
}

private static int compareNodes(DiscoveryNode o1, DiscoveryNode o2) {
    if (o1.isMasterNode() && !o2.isMasterNode()) {
        return -1;
    }
    if (!o1.isMasterNode() && o2.isMasterNode()) {
        return 1;
    }
    return o1.getId().compareTo(o2.getId());
}
```

### 



