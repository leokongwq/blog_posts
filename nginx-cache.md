---
layout: post
comments: true
title: nginx缓存功能介绍
date: 2016-11-25 11:30:57
tags:
- nginx
- web
categories:
- web
---

### 介绍

当启用了Nginx的缓存功能时，Nginx会将后端服务的响应保存在本地磁盘上，在后续的请求中只有请求满足缓存的条件就会命中缓存，Nginx不会再将请求转发到后端的服务上。

### 启用响应缓存

要开始Nginx响应缓存，我们需要在`http`配置块中添加配置指令`proxy_cache_path`。该指令的第一个参数必须是一个本机文件系统路径。参数`keys_zone`也是必须的，用来设置共享内存缓存区的名称和大小。 该内存区域是用来保存缓存条目的原信息的。例如：

    http {
        ...
        proxy_cache_path /path/to/cache levels=1:2 keys_zone=one:10m max_size=10g
                 inactive=60m use_temp_path=off;
    }

配置好`proxy_cache_path`后，我们还需要配置`proxy_cache`指令，语法如下：

    Syntax:	proxy_cache zone | off;
    Default:	
    proxy_cache off;
    Context:	http, server, location    

例子：
    
    http {
        ...
        proxy_cache_path /data/nginx/cache keys_zone=one:10m;

        server {
            proxy_cache one;
            location / {
                proxy_pass http://localhost:8000;
            }
        }
    }
    
`proxy_cache`指令有两个参数`zone`和`off`， `zone`的取值是`proxy_cache_path`指令中指定的共享内存区域名称的值（上例中的one）， 如果取值为`off`则表示禁用缓存功能（注意限制范围）。

proxy_cache_path指令的完整语法如下：

    Syntax:	proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time] [purger=on|off] [purger_files=number] [purger_sleep=time] [purger_threshold=time];
    Default:	-
    Context:	http
    
### Nginx对缓存的处理

Nginx针对响应缓存有两个处理动作

- 缓存管理器会定时检查缓存的状态。如果缓存的内容大小达到了指令`proxy_cache_path`的参数`max-size`指定的值,则缓存管理器会根据LRU算法删除缓存的内容。 在检查的间隔时间内，总的缓存内容大小可以临时超过设定的大小阈值。
- 缓存加载器只在Nginx启动的时候执行一次，将缓存内容的`原信息`加载到指定的共享内存区内。一次将所有的缓存内容加载到内存中会耗费大量的资源，并且会影响Nginx启动后几分钟内的性能。为了避免这种问题可以通过在指令`proxy_cache_path`后添加下面的参数：
    - loader_threshold – 缓存加载器加载缓存内容的最大执行时间（单位是毫秒，默认值是200毫秒）。
    - loader_files – 在缓存加载器加载缓存内容的执行时间间隔内，最多能加载多少个缓存条目，默认100。
    - loader_sleeps – 每两次执行的时间间隔, 单位是毫秒 (默认50毫秒)

例子：

    proxy_cache_path /data/nginx/cache keys_zone=one:10m loader_threshold=300 loader_files=200;
    
### 指定缓存条件

默认情况下，Nginx会缓存所有第一次通过HTTP GET 和 HEAD 方法发起的请求。Nginx将请求的完整字符串作为缓存内容的`KEY`。如果后面的请求和缓存的内容拥有相同的KEY，则Nginx会将缓存的内容发送给客户端。我们也可以在`http`, `server`,`location`块内包含各种指令来控制哪些响应可以被缓存。

可以通过使用指令`proxy_cache_key`来控制缓存KEY的生成规则：

    proxy_cache_key "$host$request_uri$cookie_user";
    
可以通过指令`proxy_cache_min_uses` 指定一个请求的响应被缓存前的最小访问次数：

    proxy_cache_min_uses 5;

可以通过指令`proxy_cache_methods` 指定只有哪些HTTP请求类型才能被缓存：

    proxy_cache_methods GET HEAD POST;    

### 限制或旁路缓存

默认情况下响应内容会拥有被缓存起来，除非缓存内容的大小超过了指定的配置大小，超过以后会根据LRU算法进行删除。单我们也可以通过在`http`,`server`,`location`中配置统一的过期时间：

    proxy_cache_valid 200 302 10m;
    proxy_cache_valid 404      1m;    
    
上面的配置指定了当后端服务的响应是200，302时，缓存的有效期是10分钟；404 缓存有效期为1分钟。

    proxy_cache_valid any 5m;

也可以不区分后端的响应码，此时可以用`any`来指代。

可以通过指令`proxy_cache_bypass`指定Nginx使用缓存的条件，如下：

    proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;    

该指令的每个参数都指定了一个条件，只有请求满足其中的任何一个条件，并且参数的值不是0，则Nginx会把请求转发到后端的服务而不会使用缓存。

可以通过指令`proxy_no_cache`指定哪些请求不需要Nginx来缓存，配置格式和`proxy_cache_bypass` 类似：

    proxy_no_cache $http_pragma $http_authorization;
    
### 清除缓存内容

Nginx提供了清除过期缓存内容的功能。 这个功能是非常有必要的，一方面我们可以手动删除过期的缓存内容；另一方面我们当我们后端的内容已经改变了，我们就可以通过删除老的缓存来使用新的内容。缓存的删除是通过特殊的HTTP请求方法`purge`或自定义的请求头来实现的。

下面的配置演示了通过HTTP方法`purge`来删除缓存内容

1. 在HTTP配合块内定义了一个新的变量`$purge_method`, 改变量的值依赖变量`$request_method`的值

    ```
    http {
        ...
        map $request_method $purge_method {
            PURGE 1;
            default 0;
        }
    }
    ```
    
2. 在配置缓存功能的`location`中, 通过包含指令`proxy_cache_purge`来指定清除缓存请求的条件。在下面的配置例子中是通过变量`$purge_method`来控制的：

   ```
   server {
    listen      80;
    server_name www.example.com;

        location / {
            proxy_pass  https://localhost:8002;
            proxy_cache mycache;
    
            proxy_cache_purge $purge_method;
        }
    }
    ``` 

#### 发送清除命令

当配置了`proxy_cache_purge`指令后，我们就可以通过发送缓存清除请求来删除缓存了。例如：

    $ curl -X PURGE -D – "https://www.example.com/*"
    //响应
    HTTP/1.1 204 No Content
    Server: nginx/1.5.7
    Date: Sat, 01 Dec 2015 16:33:04 GMT
    Connection: keep-alive
    
在上面的例子中, 所有拥有相同URL部分的资源的缓存都会被删除（通过`*`号来指定的）。然而需要注意的是这些被删除的缓存条目不会被马上从磁盘上删除。它们或者被`purger`进程删除，或者在之后的访问中被删除掉或者在超过指令`proxy_cache_path`的参数`inactive`指定的过期时间后被删除。（这个很容易理解，如果删除的内容过大，就会造成Nginx性能的临时下降；其次是尽量不影响其它正常请求的处理）

#### 限制缓存清除请求

通常推荐配置可以发起缓存删除请求的客户端IP地址，来防止意外的缓存失效。

    geo $purge_allowed {
       default         0;  # deny from other
       10.0.0.1        1;  # allow from localhost
       192.168.0.0/24  1;  # allow from 10.0.0.0/24
    }
    
    map $request_method $purge_method {
       PURGE   $purge_allowed;
       default 0;
    }

### 完全删除缓存文件

为了完全删除匹配`*`号的缓存文件, 我们需要启动一个指定的缓存清除进程。可以通过在指令`proxy_cache_path` 后面添加参数`purger=on`来启动该进程。

    proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:10m purger=on;
    
### 缓存清除配置示例

    http {
        ...
        proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=mycache:10m purger=on;
    
        map $request_method $purge_method {
            PURGE 1;
            default 0;
        }
    
        server {
            listen      80;
            server_name www.example.com;
    
                location / {
                    proxy_pass        https://localhost:8002;
                    proxy_cache       mycache;
                    proxy_cache_purge $purge_method;
                }
        }
    
        geo $purge_allowed {
           default         0;
           10.0.0.1        1;
           192.168.0.0/24  1;
        }
        map $request_method $purge_method {
               PURGE   $purge_allowed;
               default 0;
        }
    }

### Byte-Range

Sometimes, the initial cache fill operation may take some time, especially for large files. When the first request starts downloading a part of a video file, next requests will have to wait for the entire file to be downloaded and put into the cache.

NGINX makes it possible cache such range requests and gradually fill the cache with the cache slice module. The file is divided into smaller “slices”. Each range request chooses particular slices that would cover the requested range and, if this range is still not cached, put it into the cache. All other requests to these slices will take the response from the cache.

To enable byte-range caching:

1. Make sure your NGINX is compiled with the slice module.
2. Specify the size of the slice with the slice directive:
    
    ```
    location / {
        slice  1m;
    }
    ```
    
The slice size should be adjusted reasonably enough to make slice download fast. A too small size may result in excessive memory usage and a large number of opened file descriptors while processing the request, a too large value may result in latency.

3. Include the $slice_range variable to the cache key:

        proxy_cache_key $uri$is_args$args$slice_range;
    
4. Enable caching of responses with 206 status code:
    
        proxy_cache_valid 200 206 1h;
    
5. Enable passing range requests to the proxied server by passing the $slice_range variable in the Range header field:
    
        proxy_set_header  Range $slice_range;


    
### 完整配置

    http {
        ...
        proxy_cache_path /data/nginx/cache keys_zone=one:10m loader_threshold=300 
                         loader_files=200 max_size=200m;
    
        server {
            listen 8080;
            proxy_cache one;
    
            location / {
                proxy_pass http://backend1;
            }
    
            location /some/path {
                proxy_pass http://backend2;
                proxy_cache_valid any 1m;
                proxy_cache_min_uses 3;
                proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;
            }
        }
    }

### 参考

[https://www.nginx.com/blog/nginx-caching-guide/](https://www.nginx.com/blog/nginx-caching-guide/)
[https://www.nginx.com/resources/admin-guide/content-caching/](https://www.nginx.com/resources/admin-guide/content-caching/)
[http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)

    
    

