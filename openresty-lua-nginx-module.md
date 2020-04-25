---
layout: post
comments: true
title: openresty-lua-nginx-module
date: 2016-11-10 15:43:24
tags:
- nginx
- lua
categories:
- openresty
---

### 前言
偶然听见一个同事说[openresty](http://openresty.org)基于Lua操作redis时没有使用连接池，因此会不断的连接后端的redis服务，在高并发环境下会把redis搞垮。首先，redis单机肯定是有性能上限的，就算使用了连接池也会有被打垮的可能。这个和是否使用连接池没有必然的关系。当然了使用连接池肯定能大幅提高接口的QPS值。记忆中Lua中操作redis的模块`lua-resty-redis`是启用了连接池功能的。当时翻译的文档不小心丢了，所以重新整理如下。

<!-- more -->

### 先看代码

    server {
        location /test {
            content_by_lua_block {
                local redis = require "resty.redis"
                local red = redis:new()

                red:set_timeout(1000) -- 1 sec

                -- or connect to a unix domain socket file listened
                -- by a redis server:
                --     local ok, err = red:connect("unix:/path/to/redis.sock")

                local ok, err = red:connect("127.0.0.1", 6379)
                if not ok then
                    ngx.say("failed to connect: ", err)
                    return
                end

                ok, err = red:set("dog", "an animal")
                if not ok then
                    ngx.say("failed to set dog: ", err)
                    return
                end

                ngx.say("set result: ", ok)

                local res, err = red:get("dog")
                if not res then
                    ngx.say("failed to get dog: ", err)
                    return
                end

                if res == ngx.null then
                    ngx.say("dog not found.")
                    return
                end

                ngx.say("dog: ", res)

                red:init_pipeline()
                red:set("cat", "Marry")
                red:set("horse", "Bob")
                red:get("cat")
                red:get("horse")
                local results, err = red:commit_pipeline()
                if not results then
                    ngx.say("failed to commit the pipelined requests: ", err)
                    return
                end

                for i, res in ipairs(results) do
                    if type(res) == "table" then
                        if res[1] == false then
                            ngx.say("failed to run command ", i, ": ", res[2])
                        else
                            -- process the table value
                        end
                    else
                        -- process the scalar value
                    end
                end

                -- put it into the connection pool of size 100,
                -- with 10 seconds max idle time
                local ok, err = red:set_keepalive(10000, 100)
                if not ok then
                    ngx.say("failed to set keepalive: ", err)
                    return
                end

                -- or just close the connection right away:
                -- local ok, err = red:close()
                -- if not ok then
                --     ngx.say("failed to close: ", err)
                --     return
                -- end
            }
        }
    }
    
上面的代码比较容易理解，分别演示了：set, get, pipeline功能，唯一需要注意的是`red:set_keepalive`这个就表示将本次使用的连接放入连接池并指定连接的idle时间，超过idle时间的连接会被销毁。具体的信息可以参考[https://github.com/openresty/lua-nginx-module#tcpsocksetkeepalive](https://github.com/openresty/lua-nginx-module#tcpsocksetkeepalive), 不想看的同学继续往下看。


### tcpsock

`lua-nginx-module`模块提供了TCP功能，可以用来和后端的服务进行TCP通信。具体可以通过：

    tcpsock = ngx.socket.tcp()
    
来创建并返回一个TCP或面向流的`unix domain`socket对象。该对象有如下的方法：

- connect
- sslhandshake
- send
- receive
- close
- settimeout
- settimeouts
- setoption
- receiveuntil
- setkeepalive
- getreusedtimes

通过tcpsock对象提供的上述方法，我们可以和后端的服务机器建立TCP连接并通信。

### tcpsock:connect

通过`tcpsock = ngx.socket.tcp()`拿到的tcpsock对象并不能直接和后端进行TCP通信，必须调用`connect`方法建立连接才可以。但是该方法不光是建立连接这么简单。

该方法在建立连接前总是会检查连接池中是否有匹配的空闲连接，如果没有，则会真正和目前机器建立连接。
在建立连接的过程中如果需要进行域名解析，则会使用nginx核心功能中的动态域名解析进行（非阻塞的),此时必须在nginx.conf文件中配置：`resolver 8.8.8.8;` 来指定域名解析服务器（可以不适用Google）。如果域名解析返回多个IP地址，该方法会随机选一个IP进行连接建立。如果发生错误，该方法会返回`nil`，成功返回1。

    location /test {
        resolver 8.8.8.8;

         content_by_lua_block {
             local sock = ngx.socket.tcp()
             local ok, err = sock:connect("www.google.com", 80)
             if not ok then
                 ngx.say("failed to connect to google: ", err)
                 return
             end
             ngx.say("successfully connected to google!")
             sock:close()
        }
    }  
    
和 Unix 域 Socket 文件建立连接

    local sock = ngx.socket.tcp()
     local ok, err = sock:connect("unix:/tmp/memcached.sock")
     if not ok then
         ngx.say("failed to connect to the memcached unix domain socket: ", err)
         return
    end    

建立连接的超时时间可以通过指令`lua_socket_connect_timeout`进行配置或通过

    local sock = ngx.socket.tcp()
    sock:settimeout(1000)  -- one second timeout
    local ok, err = sock:connect(host, port)

代码进行配置，代码的优先级高于配置指令。 超时时间的设置比较在调用connect方法前。   

如果在一个已经建立连接的对象上调用该方法，则原来已经建立连接的socket会被关闭，这点一点要注意。

在connect方法的参数列表最后可以传递一个 `Lua table`,可以在该table中指定其它的一些配置参数。

- pool 可以用来自定义连接池的名字。缺省的连接池的名字格式是： "<host>:<port>" or "<unix-socket-path>"。

### tcpsock:setkeepalive

在tcpsock对象的所有方法中有一个方法setkeepalive方法需要注意，该方法和TCP本身机制的keepalive不同不可被字面意思迷惑。该方法的详细解释如下：

调用该方法会立刻将当前的socket连接对象放入cosocket内置的连接池中。

该方法的第一个可选参数是`timeout`, 可以用来指定该socket连接的最大idle时间（单位是毫秒）。缺省值是`lua_socket_keepalive_timeout`配置的值。如果timeout的值是0，则表示表示该连接始终存在。

第二个可选的参数值是`size`, 可以用来指定到后端服务器TCP连接池中连接数目的最大值。需要注意的是连接池的大小一旦被指定就不可被修改了。该参数的缺省值是配置指令`lua_socket_pool_size`指定的值。

### 关于cosocket 

lua-nginx-module中提供的socket API为什么称为`cosocket`呢? 这个是因为这些API的实现是基于Lua协程(coroutine)实现的,
并且是同步非阻塞的.参考[http://agentzh.org/misc/slides/ngx-openresty-ecosystem/#57](http://agentzh.org/misc/slides/ngx-openresty-ecosystem/#57)

### 参考

[https://github.com/openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module)
[https://github.com/openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module)