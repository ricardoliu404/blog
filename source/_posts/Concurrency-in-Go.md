---
title: Concurrency-in-Go
date: 2020-03-19 14:21:50
cover: https://images.pexels.com/photos/3511104/pexels-photo-3511104.jpeg?cs=srgb&dl=pexels-3511104.jpg&fm=jpg
tags:
- Go 并发之道
categories:
- 理论
---

# Goroutines

Goroutine是Go中最基本的组织单位之一，每个Go程序最少拥有一个main goroutine，程序开始时自动创建。

## 启动Goroutine

Go中使用go关键字放在函数钱来启动一个并发的函数，可以有以下三种形式：

``` Go
func main() {
    go sayHello()    // 1 调用一个函数
    go func() {
        fmt.Println("anonymous")
    }()              // 2 调用匿名函数
    go sayGoodbye()  // 3 调用一个函数变量
}

func sayHello() {
    fmt.Println("hello")
}

sayGoodbye := func() {
    fmt.Println("Goodbye")
}
```

Go遵循称为fork-join模型的并发模型。fork这个词指的是在程序中的任何一点，它都可以将一个子执行的分支分离出来，以便与其父代同时运行。join这个词指的是这样一个事实，即在将来的某个时候，这些并发的执行分支将重新组合在一起。子分支重新加入的地方称为连接点。

## 创建连接点

考虑上例中的注释1，在创建新的goroutine后被交给Go运行时安排执行，但并没有办法保证在main结束之前sayHello函数一定会被执行，因此我们需要让main goroutine等待或休眠一段时间，但这样并不意味着创建了一个连接点，只是一个竞态条件。为了创建一个连接点，我们可以使用sync包中的WaitGroup，在两个goroutine之间创建连接点，示例如下：

``` Go
var wg sync.WatiGroup
func sayHello() {
    defer wg.Done()
    fmt.Println("hello")
}
func main() {
    wg.Add(1)
    go sayHello()
    wg.Wait()
}
```

此时则一定会输出`hello`字样，此例中明确阻塞了main goroutine，直到sayHello goroutine运行结束，才会真正退出main goroutine，否则持续等待。（此处用法类似Java CountDownLatch）

## Goroutine的变量引用

考察如下代码：

``` Go
var wg sync.WaitGroup
for _, greeting := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(greeting)
    }()
}
wg.Wait()
```

事实上该程序的输出为

```
good day
good day
good day
```

同样是goroutine运行时环境已经发生改变的情况，在迭代字符串数组并为每一个切片启动一个goroutine的过程中，goroutine真正运行之前可能循环已经结束，但`greeting`变量保留了最后一次切片操作值的引用，即`good day`，因此会三次获得该值的输出。

``` Go
var wg sync.WaitGroup
for _, greeting := range []string{"hello", "greetings", "good day"} {
    wg.Add(1)
    go func(greeting string) {
        defer wg.Done()
        fmt.Println(greeting)
    }(greeting)
}
wg.Wait()
```

上述写法则在每一次切片的时候向goroutine中传入了一个字符串副本，保证了三个字符串各出现一次（但是出现顺序依旧是不确定的）。

## 衡量Goroutine的大小

官方FAQ中提到

> 新建立一个goroutine有几千字节，这样的大小几乎总是够用的。 如果出现不够用的情况，运行时会自动增加（并缩小）用于存储堆栈的内存，从而允许许多goroutine存在适量的内存中。CPU开销平均每个函数调用大约三个廉价指令。 在相同的地址空间中创建数十万个goroutines是可以的。如果goroutines只是执行等同于线程的任务，那么系统资源的占用会更小。

如下代码粗略估计goroutine的大小：

``` Go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var c <- chan interface{}// chan，用于接受数据
var wg sync.WaitGroup
var noop = func() {      // 这个goroutine不会退出，读取管道数据时阻塞，因此永远存在
        wg.Done()        
        <-c
    }

const numGoroutines = 1e4 //goroutine数量，根据大数定律，渐进地逼近一个goroutine的大小

func main() {
	wg.Add(numGoroutines)
	before := memConsumed()//创建goroutine之前的内存消耗
	for i := numGoroutines; i > 0; i-- {
		go noop()
	}
	wg.Wait()
	after := memConsumed()//创建goroutine之后的内存消耗
	fmt.Printf("%.3fkb", float64(after-before)/numGoroutines/1000)
}

func memConsumed() uint64 {
	runtime.GC()                //执行一次垃圾回收
	var s runtime.MemStats      
	runtime.ReadMemStats(&s)
	return s.Sys                //返回当前系统内存状态
}

```

在Windows10 x64 & Go 1.14环境下，得到的数据为`8.692kb`。这个例子理想化地让我们了解了每个goroutine创建所需要的付出的空间代价。