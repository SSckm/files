## 摘要

```
本文主要讲解在eclipse环境中配置Elasticsearch源码环境，阅读Elasticsearch源码
```

## 环境介绍

```
操作系统  Mac os
Elasticsearch 版本：elasticsearch-6.1.2
java 版本：jdk-9.0.4.jdk
gradle版本：gradle-4.4.1
```

## 下载Elasticsearch源码

```
下载地址：https://github.com/elastic/elasticsearch
mkdir github
cd github
git clone https://github.com/elastic/elasticsearch.git
推荐使用git clone方式下载源码，因为克隆下来后可以查看任何一个版本的源码，不然还要下载其他版本源码进行对比，比较麻烦
```

## 切换版本

找到需要阅读源码的版本，然后checkout

![](http://img.soaer.com/blog/14.png)

```
找到这个版本的提交版本号， 然后到elasticsearch的git目录
比如我这里用的版本号是6.1.2的
git checkout 5b1fea58947fad5b19398a12c50b49fbeebfb722
这样就切换到这个版本的代码上。
```

## 下载gradle

```
很久以前ES还是maven管理的，现在改成了gradle，不是很适应
下载地址：https://gradle.org/install/
```

##配置gradle环境变量

```
export GRADLE_HOME="/Users/sunny/software/gradle-4.4.1"
export G2="$GRADLE_HOME/bin"
执行source .bash_profile
测试一下  gradle -v
```

## 配置Eclipse gradle插件

```
help->Eclipse Marketplace
输入buildship然后安装，如下图所示：
```

![](http://img.soaer.com/blog/15.png)

## eclipse配置gradle

```
在偏好设置里面配置一下gradle的安装路径
```

## eclipse支持

```
cd $es_git_home(克隆下来的项目根路径)
gradle eclipse (会很长时间，下载一些jar包)
```

## 导入Eclipse

```
在eclipse安装完成gradle插件后需要重新启动eclipse
注意不能直接导入克隆下来的项目，需要导入gradle项目。
import-->Gradle-->Existing Gradle project
然后next 选择git clone的项目路径 就可以了。
成功导入后如下图所示：
```

![](http://img.soaer.com/blog/19.png)

##配置Eclipse启动参数

![](http://img.soaer.com/blog/20.png)

```
-Des.path.conf="/Users/sunny/software/elasticsearch-6.1.2/config"
-Des.path.home="/Users/sunny/software/elasticsearch-6.1.2"
-Des.path.data="/Users/sunny/software/elasticsearch-6.1.2/data"
-Dlog4j2.disable.jmx=true
-Des.path.logs="/Users/sunny/software/elasticsearch-6.1.2/log"
```

## 启动Elasticsearch

直接点击Run

```
[2018-01-23T11:04:44,305][INFO ][o.e.n.Node               ] [] initializing ...
[2018-01-23T11:04:44,474][INFO ][o.e.e.NodeEnvironment    ] [_ICupdp] using [1] data paths, mounts [[/ (/dev/disk1s1)]], net usable_space [54.2gb], net total_space [112.8gb], types [apfs]
...
...
...
...
[2018-01-23T11:05:01,267][INFO ][o.e.h.n.Netty4HttpServerTransport] [_ICupdp] publish_address {127.0.0.1:9200}, bound_addresses {[::1]:9200}, {127.0.0.1:9200}
[2018-01-23T11:05:01,267][INFO ][o.e.n.Node               ] [_ICupdp] started
```

OK, 启动成功

## 测试

```
在浏览器中直接输入 localhost:9200就可以看到ES的输出信息了。
```

这样我们就可以在eclipse中debug调试Elasticsearch了。