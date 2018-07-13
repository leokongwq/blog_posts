---
layout: post
comments: true
title: maven  命令行参数解释
date: 2017-02-28 23:08:31
tags:
- maven
---

## 前言

一直在使用maven管理项目，通常用的命令都是基于各个plugin提供的，maven提供的命令行参数接触的不算多。
本文就将maven提供的命令行参数进行一下总结，在该过程中你会有意外的惊喜。

文章使用的maven版本为: 3.5.2

<!--  more  -->

## mvn 命令用法

```shell
usage: mvn [options] [<goal(s)>] [<phase(s)>]
```

## 参数及解释

 -h,　--help                           显示帮助信息

-am,　--also-make   　                 构建指定模块，同时构建指定模块依赖的其他模块

-amd,　--also-make-dependents          构建指定模块,同时构建依赖于指定模块的其他模块;

-B,--batch-mode                        以批处理(batch)模式运行;

-C,--strict-checksums                  检查不通过,则构建失败;(严格检查)

-c,--lax-checksums                     检查不通过,则警告;(宽松检查)

-D,--define <arg>                      定义一个系统属性键值对

-e,--errors                            显示详细错误信息

-emp,--encrypt-master-password <arg>   Encrypt master security password

-ep,--encrypt-password <arg>           Encrypt server password

-f,--file <arg>                        使用指定的POM文件替换当前POM文件

-fae, --fail-at-end                     最后失败模式：Maven会在构建最后失败（停止）。如果Maven refactor中一个失败了，Maven会继续构建其它项目，并在构建最后报告失败。

-ff,--fail-fast                        最快失败模式： 多模块构建时,遇到第一个失败的构建时停止。

-fn,　--fail-never                       从不失败模式：Maven从来不会为一个失败停止，也不会报告失败。

-gs,　--global-settings <arg>            替换全局级别settings.xml文件(Alternate path for the global settings file)

-l,　--log-file <arg>                    指定输出日志文件

-N,　--non-recursive                     仅构建当前模块，而不构建子模块(即关闭Reactor功能)。

-nsu,　--no-snapshot-updates             强制不更新SNAPSHOT(Suppress SNAPSHOT updates)

-U,　--update-snapshots                  强制更新releases、snapshots类型的插件或依赖库(否则maven一天只会更新一次snapshot依赖)

-o,　--offline                           运行offline模式,不联网进行依赖更新

-P,　--activate-profiles <arg>           激活指定的profile文件列表(用逗号[,]隔开)

-pl,　--projects <arg>                   手动选择需要构建的项目,项目间以逗号分隔;A project can be specified by [groupId]:artifactId or by its relative path.

-q,　--quiet                             安静模式,只输出ERROR

-rf,　--resume-from <arg>                从指定的项目(或模块)开始继续构建

-s,　--settings <arg>                    替换用户级别settings.xml文件(Alternate path for the user settings file)

-T,　--threads <arg>                     Thread count, for instance 2.0C where C is core multiplied

-t,　--toolchains <arg>                  Alternate path for the user toolchains file

-V,　--show-version                      Display version information WITHOUT stopping build

-v,　--version                           Display version information

-X,　--debug                             输出详细信息，debug模式。

-cpu,　--check-plugin-updates            【废弃】,仅为了向后兼容

-npr,　--no-plugin-registry              【废弃】,仅为了向后兼容

-npu,　--no-plugin-updates               【废弃】,仅为了向后兼容

-up,　--update-plugins                   【废弃】,仅为了向后兼容


## 总结

maven提供和很多有用的命令行参数来定制构建行为。合理使用这些参数有助于提高maven的使用效率。

例如：一个多模块项目，只修改了其中一个小模块的代码，此时是不需要构建整个项目的，　只需要构建变化了的模块。这时参数`pl`就可以用来指定的模块。

当你想提高构建的速度是，可以通过参数`-T`来指定构建的线程数。