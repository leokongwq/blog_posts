---
layout: post
comments: true
title: openresty动态upstream实现
date: 2018-01-27 20:38:12
tags:
- openresty
---

### 背景

公司现有应用的部署方式是所有的请求通过一个外网域名进行访问，在`openresty`内部通过`location`匹配来分离请求到不同的`upstream`。这样的配置方式带来的最大问题至少有两个：

1. 配置膨胀。随着子应用接口的增加，`location`的配置会很复杂，不易阅读和新增配置。
2. 每次新增和删除接口都需要修改所有`openresty`配置。这会带来额外的运维负担，而且会影响部分接口的稳定性。

那有没有一种方式可以动态的新增和删除需要跳转的`uri`呢？ 这个就是下面我要提供的解决方案。

<!-- more -->

先来看一个示例配置，如下所示：

```nginx
upstream www.a.com {
   server 127.0.0.1:7001;
}

upstream www.b.com {
    server 127.0.0.1:7002;
}

upstream www.abc.com {
    server 127.0.0.1:7003;
}

server {
    listen 80;
    server_name www.abc.com;

    charset utf-8;

    access_log  /data/logs/nginx/abc.access.log  main;
    
    location ~ (/a/hello.html)|(/a/world.html) {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass   http://www.a.com;
    }
    
    location ~ (/b/hello.html)|(/b/world.html) {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass   http://www.b.com;
    }
    
    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://www.abc.com;
    }
}
```

### set_by_lua_file

该解决方案的原理是通过`set_by_lua_file`指令计算需要动态upstream的名称，然后在`proxy_pass`中引用该变量。具体请看如下的配置和lua代码：

```nginx
### 生产环境需要开启：on.
lua_code_cache off;
lua_package_path '/path/to/lua/module/search/path';
lua_shared_dict rewrite_conf 10m;
init_by_lua_file init.lua;
```

init.lua 用来初始化全局配置。在这里我们用来读取动态配置的`uri`

```lua init.lua
local cjson = require "cjson"
local rewrite_conf = ngx.shared.rewrite_conf;

file = io.input("rewrite.json")    -- 使用 io.input() 函数打开文件

local jsonCfg = ""
repeat
	line = io.read()            -- 逐行读取内容，文件结束时返回nil
	if nil == line then
		break
	end
	jsonCfg = jsonCfg..line
until (false)

io.close(file)                  -- 关闭文件

rewrite_json = cjson.decode(jsonCfg)

for k,v in pairs(rewrite_json) do 
	rewrite_conf[k] = v 
end
```

rewrite.json文件的内容如下所示：

```json rewrite.json
{
	"www.abc.com" : {
		"rewrite_urls" : [
			{
				"uri" : "/a/hello.html",
				"rewrite_upstream" : "www.a.com"
			},
			{
				"uri" : "/b/hello.html",
				"rewrite_upstream" : "www.b.com"
			}
		]	
	}
}
```

下面就是我们简化后的location配置

```nginx
server {
    listen 80;
    server_name www.abc.com;

    charset utf-8;

    access_log  /data/logs/nginx/abc.access.log  main;
    
    location / {
        ## 动态计算upsteam的名字
        set_by_lua_file $my_upstream set.lua;
    
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header REMOTE-HOST $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://$my_upstream;
    }
}
```

```lua set.lua
-- 动态设置				
local cjson = require "cjson"

-- 执行 rewrite 判断
local rewrite_conf = ngx.shared.rewrite_conf
local host = ngx.var.http_host
local uri = ngx.var.uri

local default_upstream = host

if rewrite_conf[host] ~= nil then
	for i, elem in ipairs(rewrite_conf[host]['rewrite_urls']) do
		if uri == elem['uri'] then
			default_upstream = elem['rewrite_upstream']
			break
		end
	end
end

ngx.log(ngx.INFO, "default_upstream="..default_upstream)
return default_upstream
```

到此，整个方案就算完成了。但是还不够完美。因为没有实现动态的效果。

要实现动态的效果其实很简单，我这里有个建议：

1. 通过将动态配置的`uri`信息，也就是`rewrite.json`的内容放置的Redis中。
2. 提供一个管理API接口，用来读取Redis中的最新配置并更新`ngx.shared.rewrite_conf`的值。

当然了，如果配置更新了，但你不想主动调用该管理API接口。你也可以通过`ngx.timer.at`来设置定时任务来主动获取Redis的配置。

### 方案二 ngx.location.capture

先介绍一下指令`ngx.location.capture`的作用：

> 语法: res = ngx.location.capture(uri, options?)
> 上下文: rewrite_by_lua*, access_by_lua*, content_by_lua*
> 向 uri 发起一个同步非阻塞 Nginx 子请求。

Nginx 子请求是一种非常强有力的方式，它可以发起非阻塞的内部请求来访问目标`location`。目标 `location` 可以是配置文件中其它文件目录，或任何其它 `nginx C` 模块，包括 `ngx_proxy`、`ngx_fastcgi`、`ngx_memc`、`ngx_postgres`、`ngx_drizzle`，甚至 `ngx_lua` 自身等等 。

需要注意的是，子请求只是模拟 HTTP 接口的形式，没有额外的 `HTTP/TCP` 流量，也没有 `IPC` (进程间通信) 调用。所有工作在内部高效地在 C 语言级别完成。

子请求与 `HTTP 301/302` 重定向指令 (通过 `ngx.redirect`) 完全不同，也与内部重定向 ((通过 `ngx.exec`) 完全不同。

在发起子请求前，用户程序应总是读取完整的 HTTP 请求体 (通过调用 `ngx.req.read_body` 或设置 `lua_need_request_body` 指令为 on).

该 API 方法（`ngx.location.capture_multi` 也一样）总是缓冲整个请求体到内存中。因此，当需要处理一个大的子请求响应，用户程序应使用 `cosockets` 进行流式处理，

#### 实现方案

参考 [用openresty实现动态upstream](http://blog.csdn.net/daiyudong2020/article/details/53027934)

该方法的实现可以做的灵活，功能也可以更强大。当然缺点是你需要做非常多的工作。例如：获取元素请求的参数，请求的HTTP方法类型，然后设置请求的URI请求参数，请求体（可以通过`always_forward_body`避免）等。

### 参考文档

[lua-nginx-module](https://github.com/openresty/lua-nginx-module)


