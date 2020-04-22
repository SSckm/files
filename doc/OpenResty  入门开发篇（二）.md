## 简介

```
本文主要介绍OpenResty中定时任务和缓存的使用，在项目开发过程中有时候会用到定时任务，比如会定时的去Redis中拉数据，定时去http接口中获取数据等等。这些场景都需要用到定时任务。
```

## 代码

以定时写本地日志为例子

```
local log = ngx.log
local flush_time = 10
local timer = ngx.timer.at

local function writeLog()
    -----------
end


local handler
handler = function ()
    local ok, err = pcall(function () writeLog() end)
    if not ok then
        write_status = false
        log(l, err)
    end
    ok, err = timer(flush_time, handler)
    if not ok then
        log(l, "failed to create the timer: ", err)
        return
    end
end

function _M.timer()
    local ok, err = timer(flush_time, handler)
    if not ok then
        log(l, "failed to create the timer: ", err)
        return
    end
end
```

## 需要注意的地方

```
1.在使用timer的过程中比如上方的writeLog方法，最好能够捕获异常情况，如果不捕获异常下面的timer是不会执行的，这样定时任务也就不会继续执行了
2.也是最重要的地方，OpenResty有两个值可以配置timer的数量  lua_max_running_timers， lua_max_pending_timers， 在开发过程中如果在请求处理方面用到大量的timer，那么会后台会出现timer过多的错误，而且如果此时你的定时任务刚好执行而timer数量达到了 那么你的定时任务也就无法执行了。所以一定要控制好timer的数量限制。防止定时任务失效。如果有耗时的任务也会长期占用timer，尽量避免。
```

