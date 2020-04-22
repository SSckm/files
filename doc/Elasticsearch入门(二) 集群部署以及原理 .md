## 摘要

```
这篇文章主要讲一下Elasticsearch的工作原理，以及一些基本概念。本文主要针对ES6.x版本，
```

## 环境

```
ubuntu 16.04
elasticsearch 6.1.2
```

## 集群（CLuster）

```
ES集群是由多节点的共同组成的，所有的节点一起保存用户的文档数据。需要注意的是一个集群只有一个集群名称。开箱即用的集群名称为elasticsearch.只用相同集群名称的节点才能加入同一个集群。
```

## 集群原理简介

```
Elasticsearch集群可以按需扩容。其中包括水平扩容和垂直扩容，其中垂直扩容可以购买性能更强大的服务器。水平扩容（为集群增加更多的机器）可以将负载压力和数据处理分散到各个节点上去处理。作为用户你可以将请求发送到集群中任何一个节点，每个节点都知道任何一个文档所处的位置，而且每个节点都可以从其他节点收集数据供用户查询。
```

## 配置文件介绍

### 重要的配置

```
node.name: node-1
cluster.name: soaer
path.data
path.logs
discovery.zen.ping.unicast.hosts
discovery.zen.minimum_master_nodes
gateway.recover_after_nodes
```

### cluster.name

```
集群中的任何一个节点中的cluster.name必须保持一致
```

### path data & logs

```
数据目录和日志目录，不在介绍了
```

### discovery.zen.ping.unicast.hosts

```
这个值的默认设置是["127.0.0.1", "[::1]"]。
这个值主要是在ES节点启动的时候会寻找的主机列表。默认扫描9300到9305端口
注意：如果这里配置的是域名的话，会扫描该域名由DNS解析出来的所有IP地址。
如果节点在其他的服务器上，需要配置该服务器的ip和端口号
```

### discovery.zen.minimum_master_nodes

```
这个就比较重要了，这个配置意思就是当一个节点有资格成为主节点，那么这个节点必须可见的主合格节点的最小数目，以便组成集群。说白了就是为了防止‘脑裂’。官方的建议是 (有资格成为主节点的数目/2) + 1. 默认值为1。
如果设置的过小，日志会打印出如下信息：

[2018-01-24T10:06:16,652][WARN ][o.e.d.z.ElectMasterService] [node-3] value for setting "discovery.zen.minimum_master_nodes" is too low. This can result in data loss! Please set it to at least a quorum of master-eligible nodes (current value: [-1], total number of master-eligible nodes used for publishing in this round: [2])
```

### gateway.recover_after_nodes

```
这个值的意思是当集群中多少台机器成功加入集群后开始恢复数据。建议集群节点的一半节点
```

## 节点介绍

```
node.master: true  默认值：true
node.data: false 默认值：true
```

### node.master

```
设置为true，则表明此节点有条件被选举为主主节点，监控整个集群,主要针对节点的加入和删除，索引的创建和删除等功能。
```

### node.data

```
数据设置为true(默认)。此节点保存数据并执行诸如CRUD、搜索和聚合等相关操作。
```

## 客户端节点

```
当node.master设置为false, node.data也设置为false的情况下，那么这个节点可以叫做客户端节点。
```

## 集群分片

```
在创建一个索引后就会为这个索引创建5个分片，默认为五个，可以在配置文件里面修改。比如有两个索引，那么主分片就会有10个。
```

## 查看分片分配情况

测试用例

```
五个节点：node-1, node-2, node-3, node-4, node-5 ,
其中 node-1, node-2, node-3设置为  默认设置
node-4设置为：node.master: false, node.data: false
node-5设置为：node.master: true, node.data: false

注意：加入此时你设置了node-5为node.master: true, node.data: false， 他也不一定就是master节点，因为1,2,3也有资格成为master节点。是选举出来的。
创建了一个索引一共五个分片（0,1,2,3,4）
下面来看分片的分配情况：
```

```
cd /home/es/node/{1-5}/elasticsearch-6.1.2/data/nodes/0/indices， 可以看到如下信息：

node-1:存储的  1,3,4
node-2:存储的  0,1,2,3
node-3:存储的  0,2,4
node-4:是客户端节点，所以没有数据，只有一个_state目录，和一个node.lock文件
node-5:设置的是只能是master，所以在node-5的data目录里面也看不到分片。
共十个分片， 五个主分片五个副本，当然也可以安装head插件查看分片部署情况， 可以看到任何一个节点宕机后集群任然可以提供全部的搜索服务。避免单点故障。
注意：主分片在创建索引后不可修改，副本分片的数目可以修改。如果想修主分片只能reindex了。（后面章节会详细降到重建索引部分）
```

## 集群状态

```
五个节点启动成功后访问一下测试：
es@ubuntu:~$ curl localhost:9200/_cluster/health?pretty
{
  "cluster_name" : "soaer",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 5,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 5,
  "active_shards" : 10,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
状态有三种：green  yellow   red
green：全部分片都可以，主分片和副本分片
yello:全部主分片可以使用， 副本分片部分不可用
red:部分主分片不可以用
```

## 实战集群状态red

```
为了测试当集群为red的情况下是否可以建立索引，我们瞬间把node-2，和node-3暂定掉，查看集群状态为red，只有三个主分片可以用。
sunny@ubuntu:~$ curl localhost:9200/_cluster/health?pretty
{
  "cluster_name" : "soaer",
  "status" : "red",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 3,
  "active_shards" : 3,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 7,
  "delayed_unassigned_shards" : 7,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 30.0
}
```

## 测试

```
curl -H 'Content-Type:application/json' -XPOST http://localhost:9200/index/test/1 -d'
	{"content":"sunny is a good boy"}
'
结果成功：可以插入
curl -H 'Content-Type:application/json' -XPOST http://localhost:9200/index/test/2 -d'
	{"content":"sunny is a good boy"}
'
插入失败：不能插入。后台报错
org.elasticsearch.action.UnavailableShardsException: [index][2] primary shard is not active Timeout: [1m], request: [BulkShardRequest [[index][2]] containing [index {[index][test][2], source[


curl -H 'Content-Type:application/json' -XPOST http://localhost:9200/index/_search?pretty
也可以查询出来刚才插入的两条文档，能够提供搜索服务。
说明再插入ID为2的文档的时候找不到2所对应的主分片，所以插入失败。
所以当集群状态为red的情况下，是可以对外提供服务的 但是不能提供全部服务。
```

## 恢复node-2和node-3

```
集群状态恢复到green，插入id为2的文档也能插入了，能够提供全部的服务。
```

