---
layout: post
comments: true
title: 基于openresty的后端应用健康检查-动态上下线
date: 2018-01-31 20:18:25
tags:
- openresty
categories:
- web
---

### 背景

为了保证在应用发布期间不间断用户的请求，我们需要实现后端服务的动态上下线。同时为了在OPS管理台能看到某个应用各个Server结点的健康状态，我们需要后端节点的健康检查功能。基于openresty，这些功能都可以实现。

<!-- more -->

### 健康检查

实现健康检查，可以通过`lua-resty-upstream-healthcheck`来实现。

具体配置如下：

```nginx nginx.conf
## 指定共享内存
lua_shared_dict healthcheck 1m;

## 在worker初始化过程中，启动定时器，进行后端结点的检查
init_worker_by_lua_block {
   local hc = require "resty.upstream.healthcheck"
   local ok, err = hc.spawn_checker {
       -- shm 表示共享内存区的名称，
       shm = "healthcheck",
       -- type 表示健康检查的类型， HTTP or TCP （目前只支持http）
       type = "http",    
       -- upstream 指定 upstream 配置的名称   
       upstream = "tomcat",
       -- 如果是http类型，指定健康检查发送的请求的URL
       http_req = "GET /health.txt HTTP/1.0\r\nHost: tomcat\r\n\r\n",
       -- 请求间隔时间，默认是 1 秒。最小值为 2毫秒
       interval = 2000,
       -- 请求的超时时间。 默认值为：1000 毫秒
       timeout = 5000,
       -- 失败多少次后，将节点标记为down。 默认值为 5
       fall = 3, 
       -- 成功多少次后，将节点标记为up。默认值为 2
       rise = 2,
       -- 返回的http状态码，表示应用正常
       valid_statuses = {200, 302},
       -- 并发度， 默认值为 1
       concurrency = 1,
   }
 
   if not ok then
       ngx.log(ngx.ERR, "=======> failed to spawn health checker: ", err)
       return
   end
}

# 配置监控检查访问页面
location /server/status {
  access_log off;
  default_type text/plain;
  content_by_lua_block {
      local hc = require "resty.upstream.healthcheck"
      ngx.say("Nginx Worker PID: ", ngx.worker.pid())
      ngx.print(hc.status_page())
  }
}
```

### 节点上下线

该功能利用了`lua-upstream-nginx-module`模块来实现，提供REST分割API来获取upstream的信息，上下线指定upstream下的server。具体看下面的代码：

```nginx
# 配置 upstream管理地址
location /upstreams {
  default_type text/plain;
  content_by_lua_file /Users/leo/workspace/lua/upstream.lua;

}
```

upstreams http API lua 代码:

```lua
local cjson = require "cjson"
local hc = require "resty.upstream.healthcheck"

local concat = table.concat
local upstream = require "ngx.upstream"
local get_servers = upstream.get_servers
local get_upstreams = upstream.get_upstreams
local get_primary_peers = upstream.get_primary_peers
local get_backup_peers = upstream.get_backup_peers
local set_peer_down = upstream.set_peer_down


-- 字符串分隔方法
function string:split(sep)  
	local sep, fields = sep or ":", {}  
	local pattern = string.format("([^%s]+)", sep)  
	self:gsub(pattern, function (c) fields[#fields + 1] = c end)  
	return fields  
end 

-- get all upstream config block 
local function get_all_upstream()
	local us = get_upstreams()
	local upstreams = {}
	
	for _, u in ipairs(us) do
		local srvs = get_servers(u)
		upstreams[u] = srvs
	end		
	
	return upstreams	
end

-- 获取某个upstream下的所有Server
local function get_servers_by_upstream_name(upstream_name)
	local up = get_all_upstream()[upstream_name]
	if not up then
		return {}
	end

	return up	
end

-- 获取所有的 peer
local function get_all_peers(upstream_name)
	local primary_peers = get_primary_peers(upstream_name)
--	local backup_peers = get_backup_peers(upstream_name)
--	
--	local primary_cnt = table.getn(primary_peers)
--	local backup_cnt = table.getn(backup_peers)
--		
--	local total = table.getn(primary_peers) + table.getn(backup_peers)
--	local all_peers = {}
--	
--	for i = primary_cnt + 1, total do
--		backup_peers[i - primary_cnt]["backup"] = true
--		table.insert(primary_peers, i, backup_peers[i - primary_cnt])
--	end
--	
	return primary_peers
end

-- 操作Server节点上下线
local function op_server(upstream_name, server_name, op)
	-- ngx.say(cjson.encode(get_all_peers(upstream_name)))
	local all_peers = get_all_peers(upstream_name)
	for i, peer in ipairs(all_peers) do 
		if peer["name"] == server_name then
			target_peer = peer
			break
		end
	end 
	
	if target_peer == nil then
		ngx.say(cjson.encode({code = "E00001", msg = "error peer name"}))	
	else
		if op == "down" then
			down_value = true
		else
			down_value = false
		end
		
		local is_back_up = target_peer["backup"] or false
		set_peer_down(upstream_name, is_back_up, target_peer["id"], down_value)		
		ngx.say(cjson.encode({code = "A00000", msg = "Success"}))	
	end
end

local http_method = ngx.req.get_method()
local sub_uris = ngx.var.uri:split("/")

-- 节点上下线
if table.getn(sub_uris) == 4 then
	op_server(sub_uris[2], sub_uris[3], sub_uris[4])
end

-- 获取master or backup nodes
if table.getn(sub_uris) == 3 then
	if sub_uris[3] == "primary" then
		ngx.say(cjson.encode(get_primary_peers(sub_uris[2])))
	else
		ngx.say(cjson.encode(get_backup_peers(sub_uris[2])))
	end
	ngx.exit(ngx.HTTP_OK)
end

-- 获取所有upstream or 指定名称的upstream
if sub_uris[2] then
	ngx.say(cjson.encode(get_all_peers(sub_uris[2])))	
else
	ngx.say(cjson.encode(get_all_upstream()))
end
```

提供的API有：

#### 获取所有的upstream的信息

http://localhost/upstreams 

#### 获取指定upstream的信息

http://localhost/upstreams/{upstream_name}

####  获取指定upstream下指定server_name的节点信息

http://localhost/upstreams/{upstream_name}/{server_name}

#### 指定upstream下指定server_name节点上下线

http://localhost/upstreams/{upstream_name}/{server_name}/down

http://localhost/upstreams/{upstream_name}/{server_name}/up

有了这些个API，就有了在OPS控制台远程控制节点上下线的能力。通过和发布系统的配合，在节点发布时，先进行下线操作，指定时间间隔（处理完目前的请求，无流量后进行发布）后再发布。

### 测试

能不能实现节点平滑的上下线， 一定要用测试结果说话。 下面是一段[gatling](http://gatling.io)的压测脚本。

以`50req/s`的速率发送请求，持续时间为30秒。如果整个压测过程中，在动态下线某个节点后，没有出现失败的请求，那就可以说明平滑的上下线是成功的。

```scala
package qiyueorder

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class AccessLimit extends Simulation {

	val httpConf = http
		.baseURL("http://www.google.com/")
		.acceptHeader("text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8")
		.doNotTrackHeader("1")
		.acceptLanguageHeader("zh-CN,en;q=0.5")
		.acceptEncodingHeader("gzip, deflate")
		.userAgentHeader("Mozilla/5.0 (Macintosh; Intel Mac OS X 10.8; rv:16.0) Gecko/20100101 Firefox/16.0")

	 val uri = "/pay/h5/paytype.action"
	 val scn = scenario("qiyue-api限流组件压测").exec(http("queryOrderStat").get(uri))

	setUp(
		scn.inject(constantUsersPerSec(50) during(30 seconds))
		//scn.inject(atOnceUsers(50))
	).protocols(httpConf)
}
```

### 参考资料

https://github.com/cubicdaiya/ngx_dynamic_upstream
https://github.com/openresty/lua-upstream-nginx-module
https://github.com/openresty/lua-resty-upstream-healthcheck

