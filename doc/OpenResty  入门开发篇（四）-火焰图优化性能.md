## 说明

```
此篇文章主要针对OpenResty中根据火焰图做性能调优。火焰图主要用于显示CPU的调用栈，x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长，y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。注意，x 轴不代表时间，而是所有的调用栈合并后。此篇文档也是记录下相关操作，以及相关经验。
```

## 需要安装systemTap

```
sudo apt-get install elfutils
sudo apt-get install  libcap-dev
sudo apt-get install systemtap
```

##下载对应的kernel debug info包

###查看kernel版本

```
****@ubuntu:~$ uname -r
4.4.0-130-generic
```

###找到对应的debug info包

```
打开下面地址找到自己kernel版本所对应的包信息。
http://ddebs.ubuntu.com/pool/main/l/linux/
```

![](https://img.soaer.com/blog/47.png)

## 安装debug info包

```
sudo dpkg -i linux-image-4.4.0-130-generic-dbgsym_4.4.0-130.156_i386.ddeb
```

## 测试

```
****@ubuntu:~$ sudo stap -ve 'probe begin { log("hello world") exit() }'
[sudo] password for ****: 
Pass 1: parsed user script and 113 library scripts using 68608virt/37912res/5072shr/33120data kb, in 80usr/0sys/95real ms.
Pass 2: analyzed script: 1 probe, 2 functions, 0 embeds, 0 globals using 69268virt/39120res/5516shr/33780data kb, in 10usr/0sys/4real ms.
Pass 3: using cached /home/****/.systemtap/cache/6e/stap_6ef108742a1cf68c5f96d4ee1c0fa266_1104.c
Pass 4: using cached /home/****/.systemtap/cache/6e/stap_6ef108742a1cf68c5f96d4ee1c0fa266_1104.ko
Pass 5: starting run.
hello world
Pass 5: run completed in 0usr/10sys/384real ms.
```

## GitHub项目克隆

```
git clone https://github.com/openresty/openresty-systemtap-toolkit.git
git clone https://github.com/brendangregg/FlameGraph.git

可以看下https://github.com/openresty/openresty-systemtap-toolkit.git项目说明文档。其中
https://github.com/openresty/openresty-systemtap-toolkit#ngx-sample-bt是针对C级别函数，
https://github.com/openresty/openresty-systemtap-toolkit#ngx-sample-lua-bt是针对lua级别的函数打印。依情况可以选择。
```

## OpenResty调优实战

```
启动openresty后，可以看到work process和master process的pid。

备注：其中64696是work process的pid, 不要写master的pid

>>>>>openresty-systemtap-toolkit.git
****@ubuntu:~/openresty-systemtap-toolkit$ sudo ./sample-bt -p 64696 -t 10 -u > ~/nx_server.bt
symbolmap: 00000001: invalid section
WARNING: Tracing 64696 (/home/****/openresty-1.11.2.1/nginx/sbin/nginx) in user-space only...
WARNING: Missing unwind data for a module, rerun with 'stap -d (unknown; retry with -DDEBUG_UNWIND)'
WARNING: no or bad debug frame hdr
WARNING: No binary search table for eh frame, doing slow linear search for stap_9c06a36d86727c27ca7267a58004402_66293
WARNING: Missing unwind data for a module, rerun with 'stap -d kernel'
WARNING: Time's up. Quitting now...(it may take a while)
WARNING: Number of errors: 0, skipped probes: 13
```

```
>>>>>FlameGraph.git
./stackcollapse-stap.pl ~/nx_server.bt > ~/nx_server.cbt
./flamegraph.pl ~/nx_server.cbt > ~/nx_server.svg
```

![](https://img.soaer.com/blog/48.png)

## 调优说明

```
再次说明下x轴和y轴的含义：
x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长，y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。注意，x 轴不代表时间，而是所有的调用栈合并后。特别注意后者说需要特别对待的就是火焰图的平定的函数。以便针对性的调优。

由于做的广告的相关投放，后端会有很多的广告投放系统，甚至十几个二十几个，通过火焰图进行调优的方面一个是json的序列化问题（大部分json序列化性能都不是很好），另一个是DNS解析问题，json问题可以通过缓存部分对象而不是每次都需要序列化来进行调优，DNS解析可以通过定时解析缓存域名和IP的对应关系来调优。
```

