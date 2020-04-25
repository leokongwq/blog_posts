---
layout: post
comments: true
title: go自带工具build的用法
date: 2016-10-14 16:41:10
tags:
    - go
categories:
    - golang
---

### 用法：
    
    usage: build [-o output] [-i] [build flags] [packages]
    
Build 会编译哪些通过`import`导入的包和那些包依赖的包，但是不会`install` 编译的结果。如果`build`的参数是一系列以`.go`结尾的文件,则`build`会将它们每一个看做是一个单独的包。当编译一个单独的`main`包时，`build`会将结果写入一个可执行的文件，文件名称默认是main方法所在的文件名。当编译多个包或一个非`main`包时，`build`会编译这些包，但是会丢掉结果对象。该操作仅用来检查这些包是否可以被编译。

<!-- more -->

`-o` 选项仅在编译单独的一个包时可用。该选项告诉`build`将编译的结果强制写入`-o`选项指定名称的文件中或可执行文件中。
`-i` 选项告诉`build`安装哪些依赖的包。
`build`子命令的选项可以被这些命令共享：build, clean, get, install, list, run,和 test commands。

 - a 强制重新编译需要编译的包。
 - n 打印`build`命令的执行过程使用的命令但是不执行它们
 - -p n 指定能并行执行的`build`个数。默认和可用的CPU个数相同。     在`darwin/arm`平台上，默认值为1。
	
 - race 启用数据种类监测。只在`linux/amd64`,`freebsd/amd64`, `darwin/amd64` 和 `windows/amd64`这些平台上可用。
	
 - v 输出被编译的包的名称。
	
 - work 输出临时工作文件夹的名称并且在build结束后不删除该目录。
	
 - x 打印执行的命令

 - asmflags 'flag list' arguments to pass on each go tool asm invocation.
	
 - buildmode mode 指定使用的编译模式。参考`go help buildmode`获取更多信息
	
 - compiler name 指定要使用的编译器(gccgo or gc)。
	
 - gccgoflags 'arg list' 指定传递给gccgo compiler/linker 的调用参数.
	
 - gcflags 'arg list' 指定传递给每个go工具的编译参数
	
 - installsuffix suffix 指定安装包文件夹名称的后缀，以此来将编译结果输出到不同的文件夹。如果使用了`-race`选项, 则安装目录后缀就是`_race`。如果同时使用了`-race 和 -installsuffix` 两个选项，则同时使用这个的组合。 a similar 
 - ldflags 'flag list' 指定传递给每个go链接工具的参数

 - linkshared link against shared libraries previously created with -buildmode=shared
	
 - pkgdir dir 指定安装和加载包的目录，用次目录来代替默认的pkg目录
	
 - tags 'tag list' 提供给build命令的编译参数

### gcflags的参数列表

    go tool compile -h
    usage: 6g [options] file.go...
      -%	debug non-static initializers
      -+	compiling runtime
      -A	for bootstrapping, allow 'any' type
      -B	disable bounds checking
      -D path
        	set relative path for local imports
      -E	debug symbol export
      -I directory
        	add directory to import search path
      -K	debug missing line numbers
      -L	use full (long) path in error messages
      -M	debug move generation
      -N	禁用优化
      -P	debug peephole optimizer
      -R	debug register optimizer
      -S	print assembly listing
      -V	print compiler version
      -W	debug parse tree after type checking
      -asmhdr file
        	write assembly header to file
      -buildid id
        	record id as the build id in the export metadata
      -complete
        	compiling complete package (no C or assembly)
      -cpuprofile file
        	write cpu profile to file
      -d list
        	print debug information about items in list
      -dynlink
        	support references to Go symbols defined in other shared libraries
      -e	no limit on number of errors reported
      -f	debug stack frames
      -g	debug code generation
      -h	halt on error
      -i	debug line number stack
      -importmap definition
        	add definition of the form source=actual to import map
      -installsuffix suffix
        	set pkg directory suffix
      -j	debug runtime-initialized variables
      -l	禁用内联
      -largemodel
        	generate code that assumes a large memory model
      -live
        	debug liveness analysis
      -m	print optimization decisions
      -memprofile file
        	write memory profile to file
      -memprofilerate rate
        	set runtime.MemProfileRate to rate
      -nolocalimports
        	reject local (relative) imports
      -o file
        	write output to file
      -p path
        	set expected package import path
      -pack
        	write package file instead of object file
      -r	debug generated wrappers
      -race
        	enable race detector
      -s	warn about composite literals that can be simplified
      -shared
        	generate code that can be linked into a shared library
      -trimpath prefix
        	remove prefix from recorded source file paths
      -u	reject unsafe code
      -v	increase debug verbosity
      -w	debug type checking
      -wb
        	enable write barrier (default 1)
      -x	debug lexer
      -y	debug declarations in canned imports (with -d)

### ldflags 参数列表

    go tool link -h
    usage: link [options] main.o
      -B note
        	add an ELF NT_GNU_BUILD_ID note when using ELF
      -C	check Go calls to C code
      -D address
        	set data segment address (default -1)
      -E entry
        	set entry symbol name
      -H type
        	set header type
      -I linker
        	use linker as ELF dynamic linker
      -L directory
        	add specified directory to library path
      -R quantum
        	set address rounding quantum (default -1)
      -T address
        	set text segment address (default -1)
      -V	print version and exit
      -W	disassemble input
      -X definition
        	add string value definition of the form importpath.name=value
      -a	disassemble output
      -buildid id
        	record id as Go toolchain build id
      -buildmode mode
        	set build mode
      -c	dump call graph
      -cpuprofile file
        	write cpu profile to file
      -d	disable dynamic executable
      -extld linker
        	use linker when linking in external mode
      -extldflags flags
        	pass flags to external linker
      -f	ignore version mismatch
      -g	disable go package data checks
      -h	halt on error
      -installsuffix suffix
        	set package directory suffix
      -k symbol
        	set field tracking symbol
      -linkmode mode
        	set link mode (internal, external, auto)
      -linkshared
        	link against installed Go shared libraries
      -memprofile file
        	write memory profile to file
      -memprofilerate rate
        	set runtime.MemProfileRate to rate
      -n	dump symbol table
      -o file
        	write output to file
      -r path
        	set the ELF dynamic linker search path to dir1:dir2:...
      -race
        	enable race detector
      -s	disable symbol table
      -shared
        	generate shared object (implies -linkmode external)
      -tmpdir directory
        	use directory for temporary files
      -u	reject unsafe packages
      -v	print link trace
      -w	disable DWARF generation	

                        
                    
                    
                    