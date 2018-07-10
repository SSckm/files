## 准备工作

### 环境介绍

```
系统：ubuntu
```

### 下载

```
下载地址：https://openresty.org/cn/download.html
或者执行：wget https://openresty.org/download/openresty-1.11.2.1.tar.gz
```

### 解压缩

```
tar -zxf openresty-1.11.2.1.tar.gz
```

### 安装依赖组件

```
apt-get install libreadline-dev libncurses5-dev libpcre3-dev \
    libssl-dev perl make build-essential cmake openssl uuid uuid-dev zlib1g zlib1g.dev
```

### 安装

```
cd 解压路径
./configure --prefix=/home/hadoop/openresty-1.11.2.1 --with-debug --with-http_ssl_module --user=hadoop
#--prefix 为安装目录
#--with-debug debug模式安装
#如果编译不带--user的话 也可以在nginx的配置文件里面指定启动用户是那个用户，如果不指定则user为 nobody(你懂的， 启动nginx后可以用   ps -aux|grep nginx 来查看进程是那个用户启动的)
make -j2
make install
至此安装成功（如果安装过程中缺少依赖包可自行下载安装依赖）
```

### 测试

```
修改nginx配置文件  $OR_HOME/nginx/conf/nginx.conf
listen       80  改成你需要监听的端口    listen       9090(80端口需要sudo权限)
启动nginx
cd $OR_HOME/nginx/sbin
./nginx

curl localhost:9090
如果出现Welcome to OpenResty!页面则安装成功
```

### 开发接口

```
修改nginx配置文件nginx.conf，增加location
必须指定lua_package_path
http {
  include mime.types;
  server_tokens off;
  reset_timedout_connection on;
  lua_package_path '$prefix/conf/lua/resty/?.lua;;';
  lua_code_cache on;
  lua_need_request_body on;
  init_by_lua_file conf/lua/resty/init/init.lua;
  init_worker_by_lua_file conf/lua/resty/init/init_work.lua;
  lua_socket_pool_size 1024;
  lua_shared_dict cache 100m;
  lua_max_running_timers 1024;
  lua_max_pending_timers 2048;
  resolver 8.8.8.8;
  include lua/resty/conf/*.conf;
}

说明:
1.$prefix为你安装openresty指定的--prefix的路径，可以使用$prefix来寻找lua文件
2.lua_code_cache要打开，不然性能非常的低，每次都回去读取lua文件，并发很低，所以要打开（调试的时候可以放开）
3.lua_need_request_body 要打开（读取request pauload的数据）
4.lua_shared_dict 这个是放全局缓存的数据可以实现key，value形式的存储，也可以实现队列。（所有nginx进程使用同一个cache）
5.lua_max_running_timers 如果在项目中有很多地方用到timer接口，这个值最好设置的大一点，特别是有定时任务定时抓取数据的timer的时候，因为在运行过程中一旦timer超多设置的值，可能会导致某些定时任务得不到timer，从而定时任务失效（这个一定要注意）
6.init_by_lua_file在master进程启动时执行
7.init_worker_by_lua_file 在worker进程启动初始化执行，每个worker都会执行一次
8.resolver 这个后续文章会提到，设计到ipv4和ipv6的解析问题


location /interface {
      default_type application/json;
      content_by_lua_block {
        local res = {}
        res.code = 0
        local cjson = require "cjson"
        ngx.say(cjson.encode(res))
      }
}

也可以使用content_by_lua_file指令 指定lua文件。
curl localhost:9090/interface 就可以访问接口了。
```

更多API请查看：https://github.com/openresty/lua-nginx-module

后续会有更多关于OpenResty的文章，希望一起学习一起进步