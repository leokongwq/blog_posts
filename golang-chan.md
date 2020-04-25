---
layout: post
comments: true
title: golang之chan简介
date: 2016-10-15 19:22:31
tags:
- go
categories:
- golang
---

                        
> Channel是Go中的一个核心类型，可以把它看成一个管道，通过它可以完成goroutine之间的通信。

<!-- more -->

### 基本操作
 
channel和map和slice一样，都需要先创建然后使用：

    ch := make(chan int) //创建了一个没有缓冲区，只能读写int数据类型的channel

channel的操作符是箭头 <- 。箭头的方向表示数据的流向，是往channel写数据还是读取数据。

    ch <- v    // 发送值v到Channel ch中
    v := <-ch  // 从Channel ch中接收数据，并将数据赋值给v

### Channel类型

Channel类型的定义格式如下：

    ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType .

它包括三种类型的定义。可选的<-代表channel的方向。如果没有指定方向，那么Channel就是双向的，既可以接收数据，也可以发送数据。

    chan T          // 可以接收和发送类型为 T 的数据
    chan<- float64  // 只可以用来发送 float64 类型的数据
    <-chan int      // 只可以用来接收 int 类型的数据

<-总是优先和最左边的类型结合。

### 创建的关闭

使用make初始化Channel,并且可以设置容量:

    ch := make(chan int, 100)
    
容量(capacity)代表Channel容纳的最多的元素的数量，代表Channel的缓存的大小。如果没有设置容量，或者容量设置为0, 说明Channel没有缓存，只有sender和receiver都准备好了后它们的通讯才会发生(和Java 的 SynchronousQueue 类似)。如果设置了缓存，就有可能不发生阻塞，只有buffer满了后send才会阻塞，而只有缓存空了后receive才会阻塞。一个nil channel不会通信。

可以通过内建的`close`方法可以关闭Channel。

多个goroutine可以同时读写channel不必考虑额外的同步措施。

Channel可以作为一个先入先出(FIFO)的队列，接收的数据和发送的数据的顺序是一致的。

channel的 receive 操作可以返回多个值，如

    v, ok := <-ch
    
它可以用来检查Channel是否已经被关闭了。如果ok是false,则表示channel已经关闭了。

**1.如果向一个已经关闭的channel写数据则会抛出运行时异常**
**2.如果读取一个已经关闭的channel，则会将缓冲区的数据读取完，而且可以一直读取该channel数据类型的零值**
**3.从一个nil channel中读取数据会一直被block**
**4.向一个nil channel写数据会一直被block**

### Range

for …… range语句可以处理Channel。

```golang
    func main() {
    	go func() {
    		time.Sleep(1 * time.Hour)
    	}()
    	c := make(chan int)
    	go func() {
    		for i := 0; i < 10; i = i + 1 {
    			c <- i
    		}
    		close(c)
    	}()
    	for i := range c {
    		fmt.Println(i)
    	}
    	fmt.Println("Finished")
    }
```
    
range c 产生的迭代值为Channel中发送的值，它会一直迭代直到channel被关闭。上面的例子中如果把close(c)注释掉，程序会一直阻塞在for …… range那一行。

### select

select语句选择一组就绪的send操作和receive操作去处理。它类似switch,但是只能用来处理channel操作。它的case可以是send语句，也可以是receive语句，亦或者default。receive语句可以将值赋值给一个或者两个变量。它必须是一个receive操作。最多允许有一个default case,它可以放在case列表的任何
位置，尽管我们大部分会将它放在最后。

```goalng
    import "fmt"
    
    func fibonacci(c, quit chan int) {
    	x, y := 0, 1
    	for {
    		select {
    		case c <- x:
    			x, y = y, x+y
    		case <-quit:
    			fmt.Println("quit")
    			return
    		}
    	}
    }
    func main() {
    	c := make(chan int)
    	quit := make(chan int)
    	go func() {
    		for i := 0; i < 10; i++ {
    			fmt.Println(<-c)
    		}
    		quit <- 0
    	}()
    	fibonacci(c, quit)
    }
```

如果有同时多个case去处理,比如同时有多个channel可以接收数据，那么Go会伪随机的选择一个case处理(pseudo-random)。如果没有case需要处理，则会选择default去处理，如果default case存在的
情况下。如果没有default case，则select语句会阻塞，直到某个case就绪。

需要注意的是，nil channel上的操作会一直被阻塞，如果没有default case,nil channel的select会一直被阻塞。select语句和switch语句一样，它不是循环，它只会选择一个case来处理，如果想一直处理channel，你可以在外面加一个无限的for循环：

```golang
    for {
    	select {
    	case c <- x:
    		x, y = y, x+y
    	case <-quit:
    		fmt.Println("quit")
    		return
    	}
    }
```

### timeout

select有很重要的一个应用就是超时处理。因为上面我们提到，如果没有case需要处理，select语句就会一直阻塞着。这时候我们可能就需要一个超时操作，用来处理超时的情况。下面这个例子我们会在2秒后往channel c1中发送一个数据，但是select设置为1秒超时,因此我们会打印出timeout 1,而不是result 1。

```golang
    import "time"
    import "fmt"
    func main() {
        c1 := make(chan string, 1)
        go func() {
            time.Sleep(time.Second * 2)
            c1 <- "result 1"
        }()
        select {
        case res := <-c1:
            fmt.Println(res)
        case <-time.After(time.Second * 1):
            fmt.Println("timeout 1")
        }
    }
```

其实它利用的是time.After方法，它返回一个类型为<-chan Time的单向的channel，在指定的时间
发送一个当前时间给返回的channel中。

### Timer和Ticker

我们看一下关于时间的两个Channel。timer是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间，它提供一个Channel，在将来的那个时间那个Channel提供了一个时间值。下面的例子中第二行会阻塞2秒钟左右的时间，直到时间到了才会继续执行。

    timer1 := time.NewTimer(time.Second * 2)
    <-timer1.C
    fmt.Println("Timer 1 expired")
    
当然如果你只是想单纯的等待的话，可以使用time.Sleep来实现。

你还可以使用timer.Stop来停止计时器。

```golang
    timer2 := time.NewTimer(time.Second)
    go func() {
    	<-timer2.C
    	fmt.Println("Timer 2 expired")
    }()
    stop2 := timer2.Stop()
    if stop2 {
    	fmt.Println("Timer 2 stopped")
    }
```
    
ticker是一个定时触发的计时器，它会以一个间隔(interval)往Channel发送一个事件(当前时间)，而Channel的接收者可以以固定的时间间隔从Channel中读取事件。下面的例子中ticker每500毫秒触发一次，你可以观察输出的时间。    

```golang
    ticker := time.NewTicker(time.Millisecond * 500)
    go func() {
    	for t := range ticker.C {
    		fmt.Println("Tick at", t)
    	}
    }()
```    

类似timer, ticker也可以通过Stop方法来停止。一旦它停止，接收者不再会从channel中接收数据了。

### 同步

channel可以用于goroutine之间的同步。

下面的例子中main goroutine通过done channel等待worker完成任务。
worker做完任务后只需往channel发送一个数据就可以通知main goroutine任务完成。

```golang
    import (
    	"fmt"
    	"time"
    )
    func worker(done chan bool) {
    	time.Sleep(time.Second)
    	// 通知任务已完成
    	done <- true
    }
    func main() {
    	done := make(chan bool, 1)
    	go worker(done)
    	// 等待任务完成
    	<-done
    }                        
```
                    
                    