---
layout: post
comments: true
title: 基于Openresy的图片服务
date: 2018-02-12 14:43:12
tags:
- Openresy
- Nginx
categories:
- 架构
---

### 背景

有个项目的需要是：用户通过给好友分享带有二维码的图片，好友扫码或在微信中识别二维码来领取分享的礼物。要实现这个需要，能想到的解决方案有两个：

<!-- more -->

### 方案一

客户端实现。背景图片存在客户端，客户端动态生成二维码，并完成图片的合成（包括给背景图添加二维码，文字水印，用户头像等信息）。

优点：图片和合成处理分布在每个用户的手机上，图片合成的计算压力进行了分散。用户体验更好。服务端只需要存储合成后的图片，并进行图片的加速访问。

缺点：这个方案的可行的前提是客户端当前需要具备这样的能力，而且需要需要访问用户手机的相册（也可以不访问）。现实问题是：1.马上开发上线，开发，审核，推广，用户下载都需要时间。2.用户版本不一致，不能保证覆盖面。

<!-- more -->

由于以上的问题，才有了方案二。

### 方案二

服务端实现。背景图片存在服务端，服务端通过图片库来完成图片的合成（二维码，水印等）。

优点：不依赖客户端。更好的动态控制能力

缺点：图片处理的压力全部在服务端。


#### Java 实现

一开始的方案是通过Java开发一个图片服务，通过Java库[zxing](https://github.com/zxing/zxing)来生成二维码。[Thumbnails](https://github.com/coobird/thumbnailator) 来进行图片的合成，水印等功能。

马上开发并进行压测后发现效率一般。决定这个方案作为备选方案，抗不住就加机器。

#### Openresty + Lua + ImageMagic

Openresty，Lua，ImageMagic 的作用就不解释了。下面直接上配置和代码。

Openresty 配置

```nginx main block
# 二维码文件名缓存
lua_shared_dict qr_cache 10m;
# 合成后的图片缓存，生产环境可以调大
lua_shared_dict share_img_file_cache 10m;
```

```nginx server

server {
    listen    80;
    server_name localhost;
    
    access_log  /data/logs/nginx/image.access.log  main;
    
    location /images {
       set $image_root "/data/static";
       set $file "$image_root$uri";
       set $convert_bin "/usr/local/bin/convert";
       rewrite_by_lua_file "/Users/leo/workspace/lua/nginx-imagemagick.lua";
    }
}
```

Lua合成图片代码

```lua nginx-imagemagick.lua
local qr = require("qrencode")
local qr_cache = ngx.shared.qr_cache;
local share_img_file_cache = ngx.shared.share_img_file_cache
local imgages_root_dir = ngx.var.image_root .. "/images/"
local img_file_suffix = ".png"

-- config
local image_sizes = { "640x640", "320x320", "124x124", "140x140", "64x64", "60x60", "32x32", "0x0" }

-- 字符串分隔方法
function string:split(sep)
	local sep, fields = sep or ":", {}
	local pattern = string.format("([^%s]+)", sep)
	self:gsub(pattern, function (c) fields[#fields + 1] = c end)
	return fields
end

-- parse uri

function parseUri(uri)

	local _, _, name, ext, size = string.find(uri, "(.+)(%..+)!(%d+x%d+)")

	--ngx.header.content_type = "text/plain";

	--ngx.say(name,size);

	if name and size and ext then
		return ngx.var.image_root .. name .. ext, size
	else
		return "", ""
	end
end

function fileExists(name)
	local f, msg = io.open(name, "r")
	if f ~= nil then
		io.close(f)
		return true
	else
		ngx.log(ngx.ERR, msg)
		return false
	end
end

function sizeExists(size)
	for _, value in pairs(image_sizes) do
		if value == size then
			return true
		end
	end
	return false
end

-- 返回图片
function response_image(img_file_path)
	ngx.header.content_type = "image/jpg";

	local f = io.open(img_file_path, "rb")
	local content = f:read("*all")
	ngx.print(content)
	f:close()
	return ngx.exit(ngx.OK)
end

-- 返回图片, 是二进制
function response_image_bin(img_bin)
	ngx.header.content_type = "image/jpg";
	ngx.print(img_bin)
	-- ngx.flush()
	return ngx.exit(ngx.HTTP_OK)
end

-- 缓存生成的图片
function cache_img_bin(cache_key, img_file_path)
	local f = io.open(img_file_path, "rb")
	local content = f:read("*all")
	share_img_file_cache[cache_key] = content
	f:close()
end

-- 图片大小转化
function resize_imgage(image_file, width_height)
	local src_image_path = imgages_root_dir .. image_file
	local target_image_path = imgages_root_dir .. image_file:split(".")[1] .. "_" .. img_file_suffix
	local command = table.concat(
		{
			ngx.var.convert_bin,
			"-resize",
			width_height,
			src_image_path,
			target_image_path
		},
		" "
	)
	-- 进行图片处理
	os.execute(command)
	return target_image_path
end

-- 生成二维码图片 - 二进制流
function gen_qr_code_bin(content)
	local qr_code_bin = qr {
		text=content,
		level="L",
		kanji=false,
		ansi=false,
		size=4,
		margin=2,
		symversion=0,
		dpi=78,
		casesensitive=true,
		foreground="000000",
		background="FFFFFF"
	}
	return qr_code_bin
end

-- 生成二维码图片文件
function gen_qr_code_file(qr_file_name, qr_code_bin)
	local full_qr_file_path = imgages_root_dir .. qr_file_name
	file = io.open(full_qr_file_path, "wb")
	io.output(file)
	io.write(qr_code_bin)
	io.close(file)
	-- 缩放到固定大小
	-- resize_imgage(qr_file_name, "120x120")
end

-- 返回指定内容的二维码文件
function get_qr_file(content)
	local qr_md5 = ngx.md5(content)

	if qr_cache[qr_md5] ~= nil
	then
		return qr_cache[qr_md5]
	else
		local qr_file_name = qr_md5 .. img_file_suffix
		local qr_code_bin = gen_qr_code_bin(content)
		gen_qr_code_file(qr_file_name, qr_code_bin)

		qr_cache[qr_md5] = qr_file_name
		return qr_file_name
	end
end

-- 获取生成二维码的内容
function get_qr_code_content()
	-- 解析 body 参数之前一定要先读取 body
	--	ngx.req.read_body()
	--	local arg = ngx.req.get_post_args()
	--	for k,v in pairs(arg) do
	--		ngx.say("[POST] key:", k, " v:", v)
	--	end

	local args = ngx.req.get_uri_args()
	local text = args['url']
	return text
end

--图片合成
function compose_img(bg_img, fg_img, result_img)
	local command = table.concat(
		{
			ngx.var.convert_bin,
			bg_img,
			"+profile '*'",
			"-quality 75%",
			"-strip",
			"-gravity northwest",
			"-compose over " .. fg_img,
			"-geometry +500+525",
			"-composite -compress BZip",
			result_img
		},
		" "
	)
	-- ngx.log(ngx.ERR, command)
	-- 进行图片处理
	os.execute(command)
end

-- 图片添加二维码，文字水印
function compose_img()
	local text = get_qr_code_content()
	if text == nil or text == "" then
		ngx.header.content_type = "text/plain"
		ngx.say('need a text param')
		return ngx.exit(ngx.OK)
	end
	--ngx.log(ngx.ERR, text)

	local sub_uris = ngx.var.uri:split("/")
	local file_name = sub_uris[table.getn(sub_uris)]

	ori_filename = ngx.var.image_root .. ngx.var.uri
	if fileExists(ori_filename) == false then
		return ngx.exit(404)
	end

	ngx.header.content_type = "image/png"

	-- 生成二维码
	local qr_code_file = imgages_root_dir .. get_qr_file(text)
	local result_img = ngx.md5(text) .. "_res_" .. img_file_suffix
	local result_file = imgages_root_dir .. result_img

	if share_img_file_cache[result_img] ~= nil
	then
			-- ngx.log(ngx.ERR, "from cache")
			return response_image_bin(share_img_file_cache[result_img])
	end

	-- ngx.exit(ngx.OK)
	compose_img(ori_filename, qr_code_file, result_file)
	cache_img_bin(result_img, result_file)

	response_image(result_file)
end

compose_img()
```

### 方案二的优化和扩展

1.生成图片的缓存可以通过在openresty前面添加专门的缓存服务器(Openresty，varnish)。
2.生成的分析图片可以通过CDN进行分发，这样用户访问的速度更快。
3.合成图片的大小可以进一步压缩。目前合成的图片偏大。
4.通过这个demo，我们可以进一步丰富功能，只要是ImageMagic支持的功能，理论上都可以实现。

### 环境的安装配置

#### Openresy安装

请参考:[http://openresty.org/en/installation.html](http://openresty.org/en/installation.html)

#### 二维码生成库libqrencode安装

ubuntu： 

```shell
sudo apt-get install libqrencode-dev libpng12-dev
```

CentOS

```shell
yum install libpng-devel
wget http://ftp.riken.jp/Linux/centos/7/os/x86_64/Packages/qrencode-devel-3.4.1-3.el7.x86_64.rpm
rpm -ivh qrencode-devel-3.4.1-3.el7.x86_64.rpm 
```

MacOS

```shell
brew install libqrencode
```

安装Lua扩展库

```shell
git clone https://github.com/vincascm/qrencode
make && make install
```

`make test`如果能成功，控制台会显示一个二维码。


下面是一个生成二维码的配置

```shell
server {

   listen 8080;
   server_name img.papa.com.cn;

   location / {
        default_type image/png;
        content_by_lua_block {
              local qr = require("qrencode")
              local args = ngx.req.get_uri_args()
              local text = args.text

              if text == nil or text== "" then
                      ngx.say('need a text param')
                      ngx.exit(404)
              end

              ngx.say(qr {
                      text=text,
                      level="L",
                      kanji=false,
                      ansi=false,
                      size=4,
                      margin=2,
                      symversion=0,
                      dpi=78,
                      casesensitive=true,
                      foreground="000000",
                      background="FFFFFF"
              })
           }
           add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";    
           add_header Pragma "no-cache";    
           add_header Cache-Control "no-cache, max-age=0, must-revalidate";
           #add_header Content-Type image/png;
  }
}
```

### 问题

在配置真个环境的过程中，有一下几点需要注意：

1.lua的版本最好使用5.1，否则会有各种问题。
2.二维码生成的配置中`ansi=false`，否则在浏览器不能显示。


### 参考

[https://github.com/vincascm/qrencode](https://github.com/vincascm/qrencode)

[基于 OpenResty 的二维码生成方案](http://blog.csdn.net/orangleliu/article/details/64912578)

[Lua包管理工具Luarocks详解](https://www.jianshu.com/p/62dc0b601e91)

[OpenResty(Nginx)+Lua+GraphicsMagick实现缩略图功能](http://www.hopesoft.org/blog/?p=1188)







