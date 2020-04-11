---
title: Concurrency-in-Go(2)
date: 2020-04-11 17:03:48
cover: https://images.pexels.com/photos/3511104/pexels-photo-3511104.jpeg?cs=srgb&dl=pexels-3511104.jpg&fm=jpg
tags:
- Go 并发之道
categories:
- 理论
---

# Channels

Channel，即通道，衍生自CSP并发模型，是Go的并发原语，在Go中非常重要。可用于同步内存的访问，但是更适合用于goroutine之间的消息传递。

通道作为信息流的载体，数据可以沿着通道被传递，在下游被取出。使用通道时，把一个值传递给chan类型的变量，然后在程序的其他位置将这个值从通道中读取出来。该值被传递的出口和入口不需要彼此的存在，只需要使用通道的引用进行操作即可。

## 通道创建
创建一个通道非常简单：

``` Go
var dataStream chan interface{}     //声明一个通道，该通道的“类型”，即其中传递的值类型为interface{}
dataStream = make(chan interface{}) //使用make函数实例化一个通道
```

## 单向通道

在上述例子中，可以在通道中读写任意类型的值，因为我们使用了空接口。通道也可以声明为仅支持单向数据流（仅发送或仅接受数据的通道）。即：

``` Go
var dataStream <-chan interface{}       //声明一个只读的单向通道
dataStream := make(<-chan interface{})

var dataStream chan<- interface{}       //声明一个只写的单向通道
dataStream := make(chan<- interface{})
```

双向通道可以转化为单向通道，该转化不可逆；而单向通道不可以转化为双向通道；读写通道之间不能相互转化。

``` Go
var receiveChan <-chan interface{}
var sendChan chan<- interface{}
dataStream := make(chan interface{})

// 这样做是有效的
receiveChan = dataStream
sendChan = dataStream
```

## 通道操作

操作通道时仍旧使用`<-`：
``` Go
stringStream := make(chan string)
go func() {
    stringStream <- "Hello channels"    //将字符串放入通道中
}()
fmt.Println(<-stringStream)             //从通道中取出字符串并打印
```

Go中的channel是有阻塞机制的，意味着试图写入已满的通道的任何goroutine都会等待通道清空，并且任何尝试从空闲通道读取的goroutine将等待。在下面的例子中，fmt.Println包含一个对通道stringStream的读取，并且将阻塞知道通道中被放入一个值。同样，匿名goroutine试图在stringStream上放置一个字符，然后阻塞住等待被读取，因此goroutine在写入成功前不会退出。因此 main goroutine和匿名的goroutine发生阻塞是毫无疑问的：

``` Go
stringStream := make(chan string)
go func() {
    if 0 != 1 {
        return
    }
    stringStream <- "Hello channels"
}()
fmt.Println(<-stringStream)
```

## 通道的遍历及原理

`<-`运算符的接收形式可以选择返回两个值：

``` Go
stringStream := make(chan string)
go func() {
    stringStream <- "Hello channel"
}()
salutation, ok := <-stringStream
fmt.Println("(%v): %v", ok, salutation)
```

此处我们接受一个字符串`salutation`和一个bool值`ok`，此处会输出：

``` Bash
(true):Hello channel
```

此处的bool值是读取操作的一个标识，用于指示读取的通道时由过程中其他位置的写入生成的值or由已关闭通道生成的默认值。

这样的特性使得我们可以对通道进行遍历，即`range`操作，通过`range chan`对某通道进行遍历，在通道关闭时自动结束循环，例如：

``` Go
intStream := make(chan int)
go func() {
    defer close(intStream)
    for i := 1 ; i <= 5 ; i ++ {
        intStream <- i
    }
}()

for integer := range intStream {//简洁的遍历操作
    fmt.Printf("%v ", integer)
}
```

关闭某个通道同样可以被作为向多个goroutine同事发消息的方式之一。如果有多个goroutine在单通道上等待，可以简单的关闭通道，而不是循环解除每一个goroutine的阻塞。由于一个已关闭的通道可以被无限次读取，因此有多少个goroutine在阻塞状态不重要，关闭通道（解除所有阻塞）消耗的资源少，执行速度快。一下是一个示例：

``` Go
begin := make(chan interface{})
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
	wg.Add(1)
	go func(i int) {
		defer wg.Done()
		<-begin //读begin通道，由于里面没有内容，因此阻塞
		fmt.Printf("%v has begun\n", i)
	}(i)
}

fmt.Println("Unblocking goroutines...")
close(begin) //关闭begin，所有goroutine的阻塞被解除，在这之前没有任何协程开始打印hasbegun
wg.Wait()
```

## 缓冲通道

通道在实例化时可携带元素，意味着即使没有读操作，也可以进行n次写入操作，即通道的容量为n。无缓冲通道可以视为一个容量为0的缓冲通道。

``` Go
var dataStream chan interface{}
dataStream = make(chan interface{}, 4)// 创建一个容量为4的缓冲通道，可以容纳4个元素，在到达4个元素之前，所有写入操作不会被阻塞
```

## 通道的默认值：nil

从一个nil通道读取，会阻塞程序，若该操作在main goroutine中，会导致死锁，若在单个goroutine中，会导致阻塞。
向一个nil通道写入，同样会阻塞程序。关闭一个nil通道时会引发panic：close of nil channel。

# select

select是用来处理通道的“东西”，下面是select使用示例：

``` Go
var c1, c2 <-chan interface{}
var c3 chan<- interface{}
select {
    case <- c1:
    //do something
    case <- c2:
    //do something
    case c3 <- struct{}{}:
    //do something
}
```

select和switch有点像，将代码块分解为case分支。不同的是，case不会被顺序测试，如果没有任何分支的条件满足，会一直等到某一个case完成。
所有通道的读写都会被考虑到，以查看她们中的任意一个是否准备好：如果没有任意一个准备好，select语句将被阻塞。有一个准备好时，该操作将继续，并执行响应语句：

``` Go
start := time.Now()
c := make(chan interface{})
go func() {
    time.Sleep(5 * time.Second)
    close(c)// 5秒后关闭通道
}()
fmt.Println("Blocking on read...")
select {
    case <- c://尝试读取通道
    fmt.Printf("Unblocked %v later.\n", time.Since(start))
}
```

本例将会输出

``` Bash
Blocking on read...
Unblocked 5s later.
```

## 多个通道同时读取会发生什么？

``` Go
c1 := make(chan interface{})
close(c1)
c2 := make(chan interface{})
close(c2)

var c1Count, c2Count int
for i := 1000; i >= 0; i-- {
	select {
	case <-c1:
		c1Count++
	case <-c2:
		c2Count++
	}
}

fmt.Printf("c1Count: %d\nc2Count: %d\n", c1Count, c2Count)
```

该例中的输出可能如下

``` Go
c1Count: 505
c2Count: 496
```

之所以可能是上述结果，事实上是因为Go的运行时，Go运行时对一组case语句执行伪随机统一选择，也就是每个case被选中的机会几乎是一样的。乍一看这似乎并不重要，但其背后的理由是令人难以置信的有意思。我们先陈述一个事实：Go运行时无法知道你的select语句的意图；也就是说，它不能推断出你的问题所在，或者你为什么将一组通道放在一个select语句中。正因为如此，Go运行时所能做的最好的事情就是在任何情况下运行良好。一个好的方法是在你的程序中中引入一个随机变量——以决定选择哪个case执行。通过加权平均使用每个通道的机会，使得所有使用select语句的Go程序表现良好。

## 如果所有通道都未初始化怎么办？

``` Go
var c <-chan int
select {
case <-c: //1
case <-time.After(1 * time.Second):
	fmt.Println("Timed out.")
}
```

Go的time包提供了一个很好的方式防止所有通道持续阻塞，上述程序会输出Time out，`time.After`接收一个类型为`time.After`的参数，并返回一个通道，该通道将在你提供该通道的持续时间后发送当前时间。这提供了在选择语句中超市的简洁方法

## 如果我们想做点什么，但当前通道还没准备好呢？

这时可以使用default条件，在所有分支都不符合执行条件时执行：

``` Go
start := time.Now()
var c1, c2 <-chan int
select {
case <-c1:
case <-c2:
default:
	fmt.Printf("In default after %v\n\n", time.Since(start))
}
```

会输出：

``` bash
In default after 1.231µs
```

这几乎就是瞬间运行到默认语句，我们可以在不阻塞的情况下退出选择快。通常我们使用for-select循环组合使用，使得goroutine可以在等待另一个goroutine报告结果的同时做一些事，例如：

``` Go
done := make(chan interface{})
go func() {
	time.Sleep(5 * time.Second)
	close(done)
}()

workCounter := 0
loop:
for {
	select {
	case <-done:
		break loop
	default:
	}

	// Simulate work
	workCounter++
	time.Sleep(1 * time.Second)
}

fmt.Printf("Achieved %v cycles of work before signalled to stop.\n", workCounter)
```

输出结果为：

``` bash
Achieved 5 cycles of work before signalled to stop.
```

该例中，有一个循环正在做某种工作，偶尔检查它是否停止工作。

若select语句没有case子句，那么这条语句将永久阻塞。

# GOMAXPROCS

这个函数控制着将要托管所谓的“工作队列”的操作系统线程数量。

几乎所有的开发人员都希望利用其进程能够使用计算机上的所有内核。因此，在随后的Go版本中，它被自动设置为主机上的逻辑CPU数量。

那么，如果你想调整这个值呢？ 大多数时候你尽量别这么想。Go的调度算法在大多数情况下足够好，即增加或减少工作队列和线程的数量可能会造成更多的伤害，但仍然有些情况下可能会更改此值。

例如，我曾经参与过一个项目，该项目有一个受竞争条件困扰的测试套件。事实上，该团队有一些测试包有时会失败。我们运行测试的基础架构只有四个逻辑CPU，因此在任何一个时间点，我们都有四个goroutines同时执行。 通过增加GOMAXPROCS超过我们所拥有的逻辑CPU数量，我们能够更频繁地触发竞态条件，从而更快地纠正它们。

有人可能会通过实验发现，他们的程序在一定数量的工作队列和线程下运行得更好，但我敦促谨慎行事。如果你通过调整CPU来压缩性能，请务必在每次提交后，使用不同硬件以及使用不同版本的Go时执行此操作。调整这个值是以更高的抽象和长期性能稳定性的降低为代价的。
