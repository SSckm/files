## 简介

```
本文主要介绍Openresy中使用ngx.balancer实现上游服务器的长链接，由于在做广告系统，每次都会大量的请求后端服务器，而且频次非常的高
```

## 准备

```
Ubuntu 16.04
OpenResty openresty-1.11.2.1
```

## http长链接和短链接

```
在HTTP/1.0中，默认使用的是短连接，浏览器和服务器每进行一次HTTP操作，就建立一次连接，任务结束就中断连接。如果客户端浏览器访问服务器每次连接都会创建一个会话。在HTTP/1.1起，默认使用长链接。
```

## nginx配置

```
在nginx中使用upstream模块可以使用长链接，但是如果直接配置upstream的话IP地址和端口都是写死的，但是我们的后台服务器都是不同的域名，这样就不能用写死IP这种方式来实现长链接了。注意nginx的长链接都是基于upstream的，脱离了upstream无法使用长链接，比如直接proxy_pass http://域名/路径，这种是不能使用长链接的。所以在这里主要讲当上游服务器多个地址的话怎么使用长链接。
```

## DNS解析域名

```
经过上面说明大家可能会说只有域名没有IP怎么办呢？此时我们需要用到resty.dns.resolver模块来定时解析所有的域名地址，这样就可以获取到IP地址，从而可以使用ngx.balancer来实现动态的upstream
```

#resty.dns.resolver的使用

直接上代码

```
local function dns_parse(url, id)
    -- 其实host和port是通过另一个程序根据URL获取到域名和port
    local host = "www.soaer.com"
    local port = 80
    if string_match(host, "^(%d%d?%d?)%.(%d%d?%d?)%.(%d%d?%d?)%.(%d%d?%d?)$") then
        write_ip_address_to_cache(url, host, port, id)
        return
    end

    local r, err = resolver:new {
        -- 这里我们用的自己云平台的DNS解析服务器，格式是：{"112.17.0.22","112.17.0.23"}
        nameservers = constant.DNS_SERVERS,
        retrans = 5,
        timeout = 2000,
    }
    if not r then
        ngx.log(l, "failed to instantiate the resolver: ", err, "URL: ", url)
        return
    end

    local answers, er = r:query(host)
    if not answers then
        ngx.log(l, "failed to query the DNS server: ", er, "URL: ", url)
        return
    end

    if answers.errcode then
        ngx.log(l, "server returned error code: ", answers.errcode, ": ", answers.errstr, "URL: ", url)
        return
    end
    -- answers可能会解析出来多个，可以选择一个
    select_ip(url, port, answers, id)
end
```

```
local function select_ip(url, port, answers, id)
    for _,v in ipairs(answers) do
        if v.address ~= nil then
            local ip_address = v.address
            --解析结果存放到缓存里面
            write_ip_address_to_cache(url, ip_address, port, id)
            break
        end
    end
end
```

到这里我们能得到所有的后台服务器的域名和端口号了。下面我们进入到动态的upstream的配置中

### location配置

```
location /reqs {
      internal;
      keepalive_timeout 150;
      #因为是通过IP访问，所以这里我们要在header中增加域名信息
      set_by_lua_file $REHOST conf/lua/resty/server/req/set_host.lua;
      #设置域名访问的uri信息
      set_by_lua_file $Path conf/lua/resty/server/req/set_path.lua;
      rewrite_by_lua_file conf/lua/resty/server/req/rewrite.lua;
      log_by_lua_file conf/lua/resty/server/req/log.lua;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header Host $REHOST;
      proxy_set_header Content-Type "application/json";
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://entrance$Path;
      #下面这两行很重要，必须添加
      proxy_http_version 1.1;
      proxy_set_header Connection "";
    }
```

### upstream配置

```
upstream entrance {
    ##下面server 0.0.0.0必须要有的。
    server 0.0.0.0;
    balancer_by_lua_file conf/lua/resty/server/engine/balance.lua;
    keepalive 32;
}
```

## balance.lua代码

```
-- ZhenxingLiu
local balancer = require "ngx.balancer"
-- 说明下 ngx.ctx.host 和 ngx.ctx.port是在rewrite.lua中存放在ngx.ctx当中的
if not ngx.ctx.host or not ngx.ctx.port then
	return ngx.exit(500)
end
local ok, err = balancer.set_current_peer(ngx.ctx.host, tonumber(ngx.ctx.port))
if not ok then
    ngx.log(ngx.ERR, "set upstream failed: ", err)
end
```

OK，这样我们就可以实现动态的upstream了。