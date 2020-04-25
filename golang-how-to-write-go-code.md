---
layout: post
comments: true
title: 如何开发GO代码
date: 2017-01-21 01:46:58
tags:
- GO
categories:
- golang
---

本文翻译自：[https://golang.org/doc/code.html](https://golang.org/doc/code.html)

### 简介

本文档演示了如何开发一个简单的Go包并介绍了Go自带的一些工具，包括标准的获取，构建和安装Go包的命令。

GO自带的工具要求你以特定方式组织代码。 请仔细阅读本文档。 它解释了入门GO开发最简单的方法。

<!-- more -->

### 代码组织

#### 概述

- GO程序员通常将它们所有的GO代码放在一个独立的工作区中。
- 工作区包含许多版本控制存储库（例如由Git管理）。
- 每个仓库包含一个或多个包。
- 每个包由位于同一个文件夹中的一个或多个GO源代码文件组成
- 包目录的路径决定其导入路径。

*请注意*，这不同于其他编程环境，其中每个项目都有一个单独的工作区，而工作区与版本控制库紧密相连。

#### 工作区

工作区是一个有层级的目录，它包含三个子目录

- src 包含GO源代码文件的目录
- pkg 包含包对象（编译后的包文件）
- bin 包含可执行的命令（生成的可执行文件）

go工具编译源码包并将生成的结果二进制文件安装到`pkg`目录和`bin`目录

src目录的子目录通常包含多个版本控制库，以此来跟踪一个或多个源码包的开发

下面是一个工作区的示例：

```golang
bin/
    hello                          # command executable
    outyet                         # command executable
pkg/
    linux_amd64/
        github.com/golang/example/
            stringutil.a           # package object
src/
    github.com/golang/example/
        .git/                      # Git repository metadata
	hello/
	    hello.go               # command source
	outyet/
	    main.go                # command source
	    main_test.go           # test source
	stringutil/
	    reverse.go             # package source
	    reverse_test.go        # test source
    golang.org/x/image/
        .git/                      # Git repository metadata
	bmp/
	    reader.go              # package source
	    writer.go              # package source
    ... (many more repositories and packages omitted) ...
```

上面的树结构显示了一个包含两个版本库的工作区（example和image）。 `example`版本库包含两个命令（hello和outyet）和一个库（stringutil）。 `image`版本库包含bmp包和[其他几个](https://godoc.org/golang.org/x/image)包。

一个典型的工作区包含许多个包含多个包和命令的源代码库。 大多数Go程序员将所有Go源代码和依赖关系保存在一个工作空间中。

命令和库是从不同类型的源包构建的。 我们稍后将讨论这种区别。

#### 环境变量 GOPATH 

环境变量GOPATH声明了你的工作区位置。这好像也是你开发go代码唯一需要配置的环境变量。

在开始开发之前，先创建一个工作空间目录并设置对应的环境变量`GOPATH`。你的工作空间可以在任何你喜欢的目录，但在该文档中我们使用`$HOME/work`。 需要注意的是该环境变量不是你安装GO开发环境时需要设置的环境变量`GOROOT`。

```
$ mkdir $HOME/work
$ export GOPATH=$HOME/work
```

为了方便起见，将工作空间下的`bin`目录添加到你的`PATH`环境变量中。    

```
export PATH=$PATH:$GOPATH/bin
```

想要学习更多关于如何设置`GOPATH`环境变量的知识，可以参考[https://golang.org/cmd/go/#hdr-GOPATH_environment_variable](https://golang.org/cmd/go/#hdr-GOPATH_environment_variable)    

#### import路径

import 路径是一个包唯一标识的字符串。一个包的import路径相对于它所在的工作空间中的位置或一个远程的仓库（下面解释）

标准库中的包都是通过短路径给出的， 例如`fmt`和`net/http`。对我们自己开发的包来说，你必须选择一个基准路径来和标准包或其它第三方包作区分。

如果你将代码保存在某个代码库中，那么你应该使用该代码库的根目录作为基准路径。 例如，如果你在`github.com/user`上有一个GitHub帐户，那么它应该是您的基准路径。

请注意，在构建之前，您不需要将代码发布到远程代码仓库。 这只是一个组织你代码的好习惯，如果有一天你会发布它。 在实践中，你可以选择任何任意的路径名称，只要它能保持在GO标准库和更大的Go生态系统的唯一性即可。

我们将使用`github.com/user`作为我们的基准路径。在你的工作空间建立一个目录来跟踪源代码。

```
mkdir -p $GOPATH/src/github.com/user
```

#### 第一个程序

为了编译和运行示例程序，首先需要选择一个包路径，(我们选择了 github.com/user/hello) 并在工作空间建立一个对应的包文件夹。    

```
$ mkdir $GOPATH/src/github.com/user/hello
```
    
下一步，创建一个名为`hello.go`的源文件，该文件包含下面的内容。

```
package main
import "fmt"
func main() {
	fmt.Printf("Hello, world.\n")
}
```

现在你可以通过go工具来编译和安装该程序：

```
go install github.com/user/hello
```

go工具会在工作空间的src目录下查找包`github.com/user/hello`的源代码

需要注意的是你可以在你系统的任何位置来运行该命令。go工具会基于环境变量`GOPATH`指定的路径来查找包`github.com/user/hello`下的源码。

如果你在包目录中运行`go install`，则可以省略包路径：

```
$ cd $GOPATH/src/github.com/user/hello
$ go install   
```

此命令构建结果是生成可执行二进制文件`hello`。 然后将该二进制文件（或在Windows下的hello.exe）安装到工作空间的bin目录。 在我们的例子中，这将是`$GOPATH/bin/hello`。

go工具只在出现错误时打印错误输出，因此如果这些命令没有产生输出，则表示它们已成功执行。

现在可以通过在命令行中键入其完整路径来运行程序：    

```
$ $GOPATH/bin/hello
Hello, world.
```

如过你将`$GOPATH/bin`添加到`PAHT`环境变量中，你也可以通过下面的命令来运行该程序。

```
$ hello
Hello, world.    
```
    
如果你正在使用一个版本管理系统，那现在应该是初始化该版本库的好时机。添加文件并提交你的第一次变更。 如果你没有版本控制系统则下面的步骤不是必须的：

```
$ cd $GOPATH/src/github.com/user/hello
$ git init
Initialized empty Git repository in /home/user/work/src/github.com/user/hello/.git/
$ git add hello.go
$ git commit -m "initial commit"
[master (root-commit) 0b4507d] initial commit
1 file changed, 1 insertion(+) create mode 100644 hello.go
```

推送代码到远程仓库作为给读者的练习。

#### 第一个库

让我们编写一个库并在hello中使用该库。

同样的，首先你需要选择一个包路径(我们选择了`github.com/user/stringutil`) 并在工作空间创建对应的目录。

    $ mkdir $GOPATH/src/github.com/user/stringutil
    
接下来在该目录创建一个名为`reverse.go`的源文件，内容如下：
```
// Package stringutil contains utility functions for working with strings.
package stringutil

// Reverse returns its argument string reversed rune-wise left to right.
func Reverse(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

现在通过命令`go build`来测试编译该包：

    go build github.com/user/stringutil
    
如果已经进入包的源代码目录你可以直接执行下面的命令：

    go build   

该命令不会有任何输出文件。如果需要生成文件你需要使用命令`go install`，该命令会把生成的文件放入`pkg`目录

在确认包`stringutil`能成功编译后，修改文件`hello.go`来使用该包：  

```golang
package main

import (
	"fmt"
	"github.com/user/stringutil"
)

func main() {
	fmt.Printf(stringutil.Reverse("!oG ,olleH"))
}
```

当go工具安装包或二进制可执行文件时会同时将依赖的包也进行安装。因此当你安装`hello`程序时：

    $ go install github.com/user/hello

包`stringutil`也会被自动安装。

执行新版本程序，你会看到一个新的反转的信息：

    $ hello
    Hello, Go!
    
在执行了上面的步骤后你的工作空间可能看起来是这样的:

```
bin/
    hello                 # command executable
pkg/
    linux_amd64/          # this will reflect your OS and architecture
        github.com/user/
            stringutil.a  # package object
src/
    github.com/user/
        hello/
            hello.go      # command source
        stringutil/
            reverse.go    # package source
```

注意，`go install`将`stringutil.a`对象放在`pkg/ linux_amd64`内的目录中，该目录反映其源目录。 这样在以后调用go工具时就可以找到包对象，并避免重新编译包。 `linux_amd64`部分有助于交叉编译，并且将反映系统的操作系统和体系结构。    

#### 包名

go源代码文件的第一行代码必须是：

    package name
    
where name is the package's default name for imports. (All files in a package must use the same name.)

Go's convention is that the package name is the last element of the import path: the package imported as "crypto/rot13" should be named rot13.

Executable commands must always use package main.

There is no requirement that package names be unique across all packages linked into a single binary, only that the import paths (their full file names) be unique.

See [Effective Go](https://golang.org/doc/effective_go.html#names) to learn more about Go's naming conventions.   

### 测试

Go has a lightweight test framework composed of the go test command and the testing package.

You write a test by creating a file with a name ending in _test.go that contains functions named TestXXX with signature func (t *testing.T). The test framework runs each such function; if the function calls a failure function such as t.Error or t.Fail, the test is considered to have failed.

Add a test to the stringutil package by creating the file $GOPATH/src/github.com/user/stringutil/reverse_test.go containing the following Go code.

```
package stringutil

import "testing"

func TestReverse(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"Hello, world", "dlrow ,olleH"},
		{"Hello, 世界", "界世 ,olleH"},
		{"", ""},
	}
	for _, c := range cases {
		got := Reverse(c.in)
		if got != c.want {
			t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
		}
	}
}
```

Then run the test with go test:

    $ go test github.com/user/stringutil
    ok  	github.com/user/stringutil 0.165s


As always, if you are running the go tool from the package directory, you can omit the package path:

    $ go test
    ok  	github.com/user/stringutil 0.165s
    
Run [go help test](https://golang.org/cmd/go/#hdr-Test_packages) and see the [testing package documentation](https://golang.org/pkg/testing/) for more detail. 

### Remote packages

An import path can describe how to obtain the package source code using a revision control system such as Git or Mercurial. The go tool uses this property to automatically fetch packages from remote repositories. For instance, the examples described in this document are also kept in a Git repository hosted at GitHub [github.com/golang/example](https://github.com/golang/example). If you include the repository URL in the package's import path, go get will fetch, build, and install it automatically:

```
$ go get github.com/golang/example/hello
$ $GOPATH/bin/hello
Hello, Go examples!
```

If the specified package is not present in a workspace, go get will place it inside the first workspace specified by GOPATH. (If the package does already exist, go get skips the remote fetch and behaves the same as go install.)

After issuing the above go get command, the workspace directory tree should now look like this:

```
bin/
    hello                           # command executable
pkg/
    linux_amd64/
        github.com/golang/example/
            stringutil.a            # package object
        github.com/user/
            stringutil.a            # package object
src/
    github.com/golang/example/
	.git/                       # Git repository metadata
        hello/
            hello.go                # command source
        stringutil/
            reverse.go              # package source
            reverse_test.go         # test source
    github.com/user/
        hello/
            hello.go                # command source
        stringutil/
            reverse.go              # package source
            reverse_test.go         # test source
```

The hello command hosted at GitHub depends on the stringutil package within the same repository. The imports in hello.go file use the same import path convention, so the go get command is able to locate and install the dependent package, too.

    import "github.com/golang/example/stringutil"            
    
This convention is the easiest way to make your Go packages available for others to use. The [Go Wiki](https://golang.org/wiki/Projects) and [godoc.org](https://godoc.org/) provide lists of external Go projects.

For more information on using remote repositories with the go tool, see [go help importpath](https://golang.org/cmd/go/#hdr-Remote_import_paths).

### What's next 

Subscribe to the [golang-announce](https://groups.google.com/group/golang-announce) mailing list to be notified when a new stable version of Go is released.

See [Effective Go](https://golang.org/doc/effective_go.html) for tips on writing clear, idiomatic Go code.

Take [A Tour of Go](https://tour.golang.org/) to learn the language proper.

Visit the [documentation page](https://golang.org/doc/#articles) for a set of in-depth articles about the Go language and its libraries and tools.

### Getting help

For real-time help, ask the helpful gophers in #go-nuts on the [Freenode](http://freenode.net/) IRC server.

The official mailing list for discussion of the Go language is [Go Nuts](https://groups.google.com/group/golang-nuts).

Report bugs using the [Go issue tracker](https://golang.org/issue).


    




