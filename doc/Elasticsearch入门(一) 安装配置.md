## 环境说明

```
Ubuntu 16.04
java 1.8
Elasticsearch 6.1
注意6.1版本的Elasticsearch需要java最低的版本位1.8，请注意查看java版本
```

## 下载安装

```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.1.2.tar.gz
tar -zxf elasticsearch-6.1.2.tar.gz
```

## 运行

```
cd $ES_HOME
bin/elasticsearch
如果想后台运行
bin/elasticsearch -d
```

## 修改java配置文件

```
cd $ES_HOME/config
vi jvm.options  //这个是修改java参数设置的配置文件，包括启动内存等
```

## 修改ES配置文件

```
cd $ES_HOME/config
vi elasticsearch.yml
```

## ES配置文件说明

```
cluster.name: 集群名称，这个建议最好设置一下
node.name: 节点名称，如果想用本机名称也可以用${HOSTNAME}
node.attr.rack: r1
path.data: /path/to/data 存放数据目录
path.logs: /path/to/logs 日志目录
network.host: 设置IP地址
http.port: http的端口号
gateway.recover_after_nodes: 这个值很重要，如果集群的话最好设置成集群中节点个数的一半（建议设置），
action.destructive_requires_name: true
```

## 启动可能会遇到的错误

### 以root用户启动错误

如果以root用户启动回报错，如下

```
java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:104) ~[elasticsearch-6.1.1.jar:6.1.1]
```

## 系统参数错误

```
[3] bootstrap checks failed
[1]: max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]
[2]: max number of threads [3826] for user [sunny] is too low, increase to at least [4096]
[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2017-12-28T14:16:08,037][INFO ][o.e.n.Node               ] [soaer-1] stopping ...
[2017-12-28T14:16:08,104][INFO ][o.e.n.Node               ] [soaer-1] stopped
[2017-12-28T14:16:08,104][INFO ][o.e.n.Node               ] [soaer-1] closing ...
[2017-12-28T14:16:08,121][INFO ][o.e.n.Node               ] [soaer-1] closed
```

如果出现上面的错误，那么就是系统的limit参数设置过小引起的。

解决方法：(设置成65536或者更大 都可以)

```
vi /etc/security/limits.conf,添加下面的内容
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

```
vi /etc/sysctl.conf
添加这行数据：vm.max_map_count=262144
保存后执行：sysctl -p
```

到此就配置完成了，大家可以启动Elasticsearch服务器了

## 中文分词

```
下载地址：https://github.com/medcl/elasticsearch-analysis-ik
也可以直接用ES自带的安装插件功能，命令如下：
./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.1.1/elasticsearch-analysis-ik-6.1.1.zip
这里大家要注意的是ES的版本跟ik的版本，很重要，安装好插件后再启动的时候也可以查看插件是否安装成功。
```

```
如果在启动的时候出现这个就说明安装成功了。
[2017-12-28T15:38:06,308][INFO ][o.e.p.PluginsService     ] [soaer-1] loaded plugin [analysis-ik]
```

到此 ES服务安装就成功了，已经插件的安装方式等等。后续还有更多的关于ES的文章。包括集群，数据迁移，数据备份等等。