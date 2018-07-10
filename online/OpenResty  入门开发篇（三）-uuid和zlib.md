## 说明

```
此篇文章主要针对OpenResty中uuid的使用以及zlib使用，uuid会有场景用到，比如生成唯一标示，广告系统中生成竞价ID等等都会用到uuid的接口。
zlib主要用于在上游服务器返回到OpenResty的时候 如果是压缩的数据，那么需要解压缩，这个时候就会用到zlib
```

## 需要安装uuid-dev

```
sudo apt-get install uuid-dev
```

## uuid生成

直接上代码

```
local ffi = require 'ffi'
local uuid_key = "uuid"
local _M = {}

ffi.cdef[[
    typedef unsigned char uuid_t[16];
    void uuid_generate(uuid_t out);
    void uuid_unparse(const uuid_t uu, char *out);
]]

local uuid = ffi.load('libuuid')
function _M.get_uuid()
    local result
    if uuid then
        local uuid_t   = ffi.new("uuid_t")
        local uuid_out = ffi.new("char[64]")
        uuid.uuid_generate(uuid_t)
        uuid.uuid_unparse(uuid_t, uuid_out)
        result = ffi.string(uuid_out)
    end
    return string.gsub(result, "-", "");
end

return _M
```



## 下载lua-zlib

```
下载地址：https://github.com/brimworks/lua-zlib
```

## 安装cmake

```
sudo apt-get install cmake
```

## 编译安装

```
cmake -DLUA_INCLUDE_DIR=$OR_HOME/luajit/include/luajit-2.1 -DLUA_LIBRARIES=$OR_HOME/luajit/lib -DUSE_LUAJIT=ON -DUSE_LUA=OF

make && make install
```

注意：$OR_HOME为OpenResty的安装目录， 编译安装需要安装cmake依赖

把目录下面编译好的zlib.so拷贝到 $OR_HOME/lualib/zlib.so

## 使用方式

```
local function unzip_data(resp)
    local is_gzip = resp.header['Content-Encoding']
    local body = resp.body
    if nil == body or "" == body then
        return nil
    end
    if "gzip" == is_gzip then
        local gzip = zlib.inflate()
        return gzip(body)
    else
        return body
    end
end
```

这样上游服务器返回的数据如果是压缩的情况下就可以得到明文数据了。