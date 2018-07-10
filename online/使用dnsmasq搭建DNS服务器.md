## 准备工作

## 服务器

```
系统：Ubuntu
```

## 安装dnsmasq

```
sudo apt-get install dnsmasq
```

## 添加配置文件

```
1.在用户目录下面添加配置文件目录
mkdir dnsmasq
cd dnsmasq
2.新建dnsmasq.conf文件
touch dnsmasq.conf
3.新建dnsmasq-resolv.conf文件
touch dnsmasq-resolv.conf
4.新建dnsmasq.hosts文件
touch dnsmasq.hosts
```

## dnsmasq.conf文件内容

```
cache-size=10000
port=5353
resolv-file=/home/appops/dnsmasq/dnsmasq-resolv.conf
addn-hosts=/home/appops/dnsmasq/dnsmasq.hosts
log-queries
log-facility=/home/appops/dnsmasq/dnsmasq.log
local-ttl=600
conf-dir=/home/appops/dnsmasq/dnsmasq.d
```

说明：cache-size为缓存的条目数，port为dns服务器监听的端口号（普通用户不能使用53端口，受保护端口）

也可以直接拷贝/etc/dnsmasq.conf文件，这个文件里面有更为详细的配置。这里提供最简配置。

## dnsmasq-resolv.conf文件内容

```
nameserver 8.8.8.8
```

说明：8.8.8.8是谷歌提供的免费的DNS服务器

## dnsmasq.hosts文件内容

```
223.221.11.20  cc.soaer.com
223.221.11.20  bb.soaer.com
223.221.11.20  dd.soaer.com
```

## 新建文件夹

```
mkdir dnsmasq.d
```

## 文件列表

```
hadoop@soaer:~/dnsmasq$ ls
dnsmasq.conf  dnsmasq.d  dnsmasq.hosts  dnsmasq.log  dnsmasq-resolv.conf
```

## 启动dnsmasq

```
/usr/sbin/dnsmasq --conf-file=dnsmasq/dnsmasq.conf
```

## nginx配置DNS服务器

```
http {
  include mime.types;
  resolver 127.0.0.1:5353 valid=30s;
  include lua/resty/conf/*.conf;
}
```

这样就配置好了可以使用了。