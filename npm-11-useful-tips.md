---
layout: post
comments: true
title: npm11个提供工作效率的用法
date: 2016-10-21 22:27:08
tags:
- nodejs
- npm
categories:
- nodejs
---

### 简介
> Npm 是开发NodeJS程序所需的包管理工具. 内置了非常多的功能. 我们可以通过在命令行直接输入`npm`, 不加任何参数就能查看npm
都有哪些功能或子命令.想要完整掌握每个功能用法有点困难,个人觉得也没有必要.只要掌握了常用的几个命令就OK了.
<!-- more -->

### 打开依赖包的项目主页

    npm home <package>
    
执行上面的命令我们可以通过浏览器打开`package`指定包的工程home页, 查询该包的介绍和使用说明.
    
### 打开模块的 GitHub 仓库
    
    npm repo <package>

执行上面的命令我们可以通过浏览器打开`package`指定包在github.com站点的地址.

### 列出需更新的依赖包

    npm outdated

在工程的更目录下执行上面的命令可以检测依赖的包哪些有了新版本.
    
### 检查未被 package.json 声明的包
    
    npm prune [[<@scope>/]<pkg>...] [--production]

在项目根目录下执行`npm prune`, 可以列出没有在文件package.json中声明但是在目录node_modules中存在的包.你可以评估一下哪些包是真正需要的(比如说通过 npm install 安装却未加 --save 参数的包), 不需要的包可以通过命令`npm prune <package>`进行删除.

### 锁定依赖版本

    npm shrinkwrap
    
执行 shrinkwrap 命令将生成 npm-shrinkwrap.json 文件。它能将安装在 node_modules 目录中的依赖锁定在一个特定的版本。当 npm-shrinkwrap.json 文件存在时，npm install 将遵守其中的版本号，而不再使用 package.json 里的更高版本号
    
### 使用 npm v3 和 Node.js v4 LTS
    
    npm install -g npm@3
    
用 npm 全局安装 npm@3 将把 v2 版的 npm 升级到 v3 版，甚至在 Node.js v4 LTS（Argon）上也可以这样做。这样，你的 v4 LTS 运行环境里就可以使用最新稳定版的 npm v3 了。
    
### 让 npm install -g 不再请求 sudo
    
    npm config set prefix <dir>

执行这个命令时，需要在 <dir> 参数指定新的全局安装模块的目录，这样你就可以避免全局安装时要输入密码的情况了。指定的目录将成为新的全局 bin 目录。需要注意的是，新指定的目录必须对你的当前用户提供写权限，可以使用例如 chown -R <USER> <dir> 的命令来修改权限.
    
### 修改工程里的保存前缀
    
    npm config set save-prefix ~
    
在安装新模块并用 --save 或 --save-dev 保存时，波浪号 ~ 比 npm 默认的 ^ 号保守。波浪号会把依赖锁定在当前小版本号，也就是说可以通过 npm update 来安装 patch 发行版。而 ^ 代表锁定在大版本号，通过 npm update 安装的是 minor 发行版。
    
### 上线前去掉工程里的 devDependencies
    
当你的项目准备上线进入生产环境时，请务必在安装模块时加上 --production 标记。这个标记将只安装 dependencies，忽略 devDependencies。这就保证了线上版本的代码库里不含有调试用的工具包等。
    
其实，你可以在 NODE_ENV 环境变量中设置 production，从而保障线上工程的 devDependencies 永远不被安装。

### 使用 .npmignore 时需谨慎

如果你尚未使用过 .npmignore，其默认值还是很安全的。

一旦你在工程里添加了 .npmignore 文件，那么 .gitignore 里的规则就会被无视了。结局就是你必须保持这两个文件内容的同步，否则有可能在发布时泄漏敏感信息。

### 设置 npm init 的默认值

当你在新工程里执行 npm init 时，会从头到尾走一遍设置流程，最后生成 package.json。如果你想要设定一些可以让 npm init 一直使用的默认值，可以执行 config set 命令，执行方法如下：

    npm config set init.author.name <name>
    npm config set init.author.email <email>
    
或者你想要直接修改初始化脚本文件，做一些自定的初始化逻辑，那么请执行下面的命令：

    npm config set init-module ~/.npm-init.js`
    
下面是个脚本范例，它会提问一些私人问题，并在需要时创建 GitHub 仓库。请修改默认的 GitHub 用户名 YOUR_GITHUB_USERNAME，作为 GitHub 用户名的默认参数。

```javascript
var cp = require('child_process');  
var priv;
var USER = process.env.GITHUB_USERNAME || 'YOUR_GITHUB_USERNAME';
module.exports = {
  name: prompt('name', basename || package.name),
  version: '0.0.1',
  
  private: prompt('private', 'true', function(val){
    return priv = (typeof val === 'boolean') ? val : !!val.match('true')
  }),
  create: prompt('create github repo', 'yes', function(val){
    val = val.indexOf('y') !== -1 ? true : false;
    if(val){
      console.log('enter github password:');
      cp.execSync("curl -u '"+USER+"' https://api.github.com/user/repos -d " +
        "'{\"name\": \""+basename+"\", \"private\": "+ ((priv) ? 'true' : 'false')  +"}' ");
      cp.execSync('git remote add origin '+ 'https://github.com/'+USER+'/' + basename + '.git');
    }
    return undefined;
  }),
  main: prompt('entry point', 'index.js'),
  repository: {
    type: 'git',
    url: 'git://github.com/'+USER+'/' + basename + '.git' },
  
  bugs: { url: 'https://github.com/'+USER'/' + basename + '/issues' },
  homepage: "https://github.com/"+USER+"/" + basename,
  keywords: prompt(function (s) { return s.split(/\s+/) }),
  license: 'MIT',
  cleanup: function(cb){
    cb(null, undefined)
  }
}
```    
    



    

