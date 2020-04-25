---
layout: post
comments: true
title: hexo使用post_asset_folder功能文章链接不能以html结尾
date: 2016-10-14 16:12:49
tags:
    - hexo
categories:
    - web
---

### 解决使用hexo时启用post_asset_folder功能时,文章的perma_link不能以`.html`结尾bug

> 原来的blog是我用Node.JS + Mysql开发的, 一方面重新复习下Node.JS的使用,一方面想开发一个blog将好的技术文章,自己工作中遇到的问题,知识点进行记录. 开发完成后又觉得还是静态bolg好.找了一圈选了Hexo.搭建的过程中遇到了一些问题,本篇就是其中一个.

<!-- more -->

### permalink

`permalink` 的默认值是:`:year/:month/:day/:title/`, 这样每篇文章的url就是以`/`结尾的. 这样对搜索引擎是不友好的(没有试验).所以我就讲`permalink`的值改为`:year/:month/:day/:title.html`这中格式.改成这样格式本身并没有问题,但是如果你的文章中需要引用一些图片资源这样就需要使用`post_asset_folder`功能.

### post_asset_folder
 
`post_asset_folder`:该功能会在文章的同级目录建立一个名为`:title`的目录,这样在该文章中就可以通过如下的方式引用图片资源

```js
{% asset_img 图片路径(相对于_post目录) %}
```
问题来了: hexo在保存资源的时候`public`下对应的目录已经存在了文件`test.html`, 这样创建对应的资源目录`test.html`就会失败.其实文章对应的资源目录应该是`test`,而不是`test.html`. 


### 解决办法

既然已经找到了问题之所在,解决的办法就是在构造文章对应的资源目录是去掉`.html`, 通过跟踪hexo的代码找到了在文件`hexo/lib/models/post_asset.js` 中实现了文章对应资源的保存代码(看文件名也能猜到,哈哈).修改后的代码如下:

```js
  PostAsset.virtual('path').get(function() {
    var Post = ctx.model('Post');
    var post = Post.findById(this.post);
    if (!post) return;
    // PostAsset.path is file path relative to `public_dir`
    // no need to urlescape, #1562
      //如果生成的文章路径是以html结尾的, 如:  2016/10/13/byte-order.html,
      // 则对应的资源路径应该是: 2016/10/13/byte-order + this.slug
      var reg = new RegExp("html" + "$");
      if(reg.test(post.path)) {
          var assetPath = post.path.substr(0, post.path.lastIndexOf("."));
          return pathFn.join(assetPath, this.slug);
      }
      return pathFn.join(post.path, this.slug);
  });
```
经测试,可以正确实现我所需要的功能. 解决的办法也放到[https://github.com/hexojs/hexo/issues/2134](https://github.com/hexojs/hexo/issues/2134)
上了,遇到该问题的同学可以使用了.希望作者能提供更好的解决办法.这样就不用每个人修改自己的代码.

