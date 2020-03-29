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

接下来测试在一个线程上发送和接受消息所需的时间，此处构建一个类似Linux内核基准测试的Go例子，创建两个goroutine并在他们之间发送消息：

``` Go
func BenchmarkContextSwitch(b *testing.B) {
	var wg sync.WaitGroup
	begin := make(chan struct{})
	c := make(chan struct{})

	var token struct{}
	sender := func() {
		defer wg.Done()
		<- begin// 此处会阻塞，直到接收到数据
		for i := 0 ; i < b.N ; i ++ {
			c <- token// 接受发送者的数据，结构体不占空间，因此不会影响发送信息的时间
		}
	}

	receiver := func() {
		defer wg.Done()
		<- begin// 此处会阻塞，直到接收到数据
		for i := 0 ; i < b.N ; i ++ {
			<- c// 仅接受发送过来的数据，不作处理
		}
	}

	wg.Add(2)
	go sender()
	go receiver()
	b.StartTimer()
	close(begin)
	wg.Wait()
}
```

结果如下，可见线程间切换速度为159ns/op

``` 
goos: linux
goarch: amd64
BenchmarkContextSwitch
BenchmarkContextSwitch-8   	 9725066	       159 ns/op
PASS
```

# sync包

sync包包含对低级别内存访问同步最有用的并发原语。 如果你使用的是主要通过内存访问同步处理并发的语言，那么这些类型可能已经很熟悉了。Go与这些语言的区别在于，Go在内存访问同步基元的基础上构建了一组新的并发基元，并为使用者提供扩展的内容。 这些操作都有其用处——主要体现在小的作用域中，例如结构体。 你可以自行决定何时内存访问同步是最适当的。

## WaitGroup

若不关心并发操作的结果，或有其他方式收集结果，可以使用`WaitGroup`等待一组并发操作完成。`WaitGroup`类似于Java中的CountDownLatch，通过Add方法增加计数，通过Done方法减少计数。Wait方法会阻塞当前协程直到计数器归零。

**注意Add方法应在goroutine调用之前添加，否则无法保证Add方法在goroutine中的调用早于Wait的调用**

``` Go
var wg sync.WaitGroup

wg.Add(1) 
go func() {
	defer wg.Done() 
	fmt.Println("1st goroutine sleeping...")
	time.Sleep(1)
}()

wg.Add(1) 
go func() {
	defer wg.Done() 
	fmt.Println("2nd goroutine sleeping...")
	time.Sleep(2)
}()

wg.Wait() 
fmt.Println("All goroutines complete.")
```

## Mutex & RWMutex

### Mutex 互斥锁

Mutex提供了一种并发安全的方式来表示对共享资源访问的独占。下例有两个简单的goroutine，试图增加和减少一个公共值，病使用Mutex同步访问：

``` Go
var count int
var lock sync.Mutex

increment := func() {
    lock.Lock()
    defer lock.Unlock()
    count ++
    fmt.Println("Incrementing: %d\n", count)
}

decrement := func() {
    lock.Lock()
    defer lock.Unlock()
    count --
    fmt.Println("Decrementing: %d\n", count)
}

var arithmetic sync.WaitGroup
for i := 0 ; i <= 5 ; i ++ {
    arithmetic.Add(1)
    go func() {
        defer arithmetic.Done()
        increment()
    }()
}

for i := 0 ; i <= 5 ; i ++ {
    arithmetic.Add(1)
    go func() {
        defer arithmetic.Done()
        decrement()
    }()
}

arithmetic.Wait()
fmt.Println("Arithmetic complete.")
```

被锁定的部分时程序的性能瓶颈，进入和退出锁的成本较高，因此通常需要尽量减少锁定的范围。若多个并发进程之间共享的内存不是都要读和写的，可以考虑使用另一个类型的互斥锁：sync.RWMutex。

### RWMutex 读写锁

RWMutex在控制方式上更加灵活，可以请求读锁，此时你可以读内存，除非有其他goroutine在进行写操作，也就是说，只要没有goroutine在进行写操作，可以允许任意数量的并发读。如下时一个实例：

``` Go
package main

import (
	"fmt"
	"math"
	"os"
	"sync"
	"text/tabwriter"
	"time"
)

var producer = func(wg *sync.WaitGroup, l sync.Locker) {
	defer wg.Done()
	for i := 5 ; i > 0 ; i -- {
		l.Lock()
		l.Unlock()
		time.Sleep(1)
	}
}

var observer = func(wg *sync.WaitGroup, l sync.Locker) {
	defer wg.Done()
	l.Lock()
	defer l.Unlock()
}

var test = func(count int, mutex, rwMutex sync.Locker) time.Duration {
	var wg sync.WaitGroup
	wg.Add(count + 1)
	beginTestTime := time.Now()
	go producer(&wg, mutex)
	for i := count ; i > 0 ; i -- {
		go observer(&wg, rwMutex)
	}
	wg.Wait()
	return time.Since(beginTestTime)
}

func main() {
	tw := tabwriter.NewWriter(os.Stdout, 0, 1, 2, ' ', 0)
	defer tw.Flush()

	var m sync.RWMutex
	fmt.Fprintf(tw, "Readers\tRWMutex\tMutex\n")
	for i := 0 ; i < 20 ; i ++ {
		count := int(math.Pow(2, float64(i)))
		fmt.Fprintf(
			tw, "%d\t%v\t%v\n", count,
			test(count, &m, m.RLocker()), test(count, &m, &m),
			)
	}
}
```

本机运行环境为Ubunut 18.04 with Go 1.14，运行结果如下，可以观察到在某些情况下RWMutex耗时仍超过Mutex。（僵硬）
```
Readers  RWMutex       Mutex
1        13.604µs      1.749µs
2        2.063µs       1.675µs
4        3.376µs       2.13µs
8        9.271µs       5.18µs
16       5.468µs       12.849µs
32       18.164µs      29.581µs
64       32.949µs      30.475µs
128      30.385µs      32.923µs
256      50.322µs      112.662µs
512      98.378µs      117.562µs
1024     248.239µs     244.162µs
2048     428.428µs     421.438µs
4096     735.572µs     859.512µs
8192     1.645508ms    1.396318ms
16384    3.436893ms    3.579149ms
32768    6.182269ms    6.854283ms
65536    12.855801ms   13.686491ms
131072   25.55003ms    27.068042ms
262144   54.767586ms   60.825475ms
524288   103.354592ms  103.018981ms
```

## Cond

> Cond实现了一个条件变量，用于等待或者宣布时间发生时goroutine的交汇点。

有时需要在收到某个goroutine信号之前令自己处于等待状态，最初的暴力解决办法是使用无限循环`for conditionTrue() == false {/*巴拉巴拉*/}`，但是这样会消耗一个内核的所有周期。同事我们也可以用sleep的方法暂停当前协程，即`for conditionTrue() == false { time.Sleep(1 * time.Millisecond); /*巴拉巴拉*/}`，但是执行效率非常低，明确标示需要休眠多久，这样过长或过短都会浪费CPU时间。

此时我们即可考虑`Cond`。

``` Go
c := sync.NewCond(&sync.Mutex{})
c.L.Lock()
for conditionTrue() == false {
	c.Wait()
}
c.L.UnLock()
```

在该段代码中，首先实例化一个新的Cond，将互斥锁传入；接着锁定Cond中的Locker，因为在Wait调用时，此goroutine会被暂停并解锁该Locker；在循环中进入暂停状态，此处被阻塞，直到接收到通知；最后在循环外部解锁Locker。

**在这个例子里，调用`Wait`时不仅仅阻塞当前goroutine，同时会调用`c.L.Unlock()`，并在退出`Wait`时会调用`c.L.Lock()`。也就是说，在等待条件发生的过程中我们并非一直持有这个锁。**

``` Go
c := sync.NewCond(&sync.Mutex{})//使用一个标准的sync.Mutex作为Locker来创建Cond
queue := make([]interface{}, 0, 10)//创建一个长度为零的切片。 由于我们知道最终会添加10个元素，因此我们将其容量设为10

removeFromQueue := func(delay time.Duration) {
	time.Sleep(delay)
	c.L.Lock()//再次进入该并发条件下的关键部分，以修改与并发条件判断直接相关的数据
	queue = queue[1:]//移除切片的头部并重新分配给第二个元素，这一步模拟了元素出列
	fmt.Println("Removed from Queue")
	c.L.Unlock()//退出操作关键部分，因为我们已经成功移除了一个元素
	c.Signal()//发出信号，通知处于等待状态的goroutine可以进行下一步了
}

for i := 0 ; i < 10 ; i ++ {
	c.L.Lock()//进入关键的部分前调用Lock来锁定c.L。
	for len(queue) == 2 {//在这里我们检查队列的长度，以确认什么时候需要等待。由于removeFromQueue是异步的，for不满足时才会跳出，而if做不到重复判断，这一点很重要。
		c.Wait()//调用Wait，这将阻塞main goroutine，直到接受到信号。
	}
	fmt.Println("Adding to Queue")
	queue = append(queue, struct{}{})
	go removeFromQueue(1 * time.Second)//创建一个新的goroutine，它会在1秒后将元素移出队列。
	c.L.Unlock()//这里我们退出条件的关键部分，因为我们已经成功加入了一个元素。
}
```

上述代码片段输出为：

```
Adding to queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
Removed from queue 
Adding to queue 
```

程序成功将所有十个元素添加到队列中，并有机会在最后两项出队之前退出。也会持续等待，直到至少有一个元素在放入另一个元素之前出列。
`Signal`是`Cond`类型提供的两种通知方法之一，用于通知在等待调用上阻塞的goroutines条件已经被触发。另一种是`Broadcast`。在`Cond`内部，`runtime`维护了一个等待信号发送的goroutines的FIFO队列，`Signal`寻找等待时间最长的goroutine并通知，而`Broadcast`向所有处在等待状态的goroutines发送信号。

关于`Broadcast`可以有如下示例，该示例中创建一个带有按钮的程序，该程序注册任意数量的函数，点击按钮时运行这些函数，用`Broadcast`通知：

``` Go
type Button struct {
	//定义一个Button类型，包含了sync.Cond指针类型的Clicked属性，这是goroutine接收通知的关键条件。
	Clicked *sync.Cond
}
button := Button{Clicked: sync.NewCond(&sync.Mutex{})}

subscribe := func(c *sync.Cond, fn func()) { //定义了简单的函数，允许我们注册函数来处理信号。每个注册的函数都在自己的goroutine上运行，并且在该goroutine不会退出，直到接收到通知。
	var tempwg sync.WaitGroup
	tempwg.Add(1)
	go func() {
		tempwg.Done()
		c.L.Lock()
		defer c.L.Unlock()
		c.Wait()
		fn()
	}()
	tempwg.Wait()
}

var wg sync.WaitGroup //创建WaitGroup，等待所有子协程完成后再结束。
wg.Add(3)
subscribe(button.Clicked, func() { //注册了最大化窗口函数
	fmt.Println("Maximizing window.")
	wg.Done()
})
subscribe(button.Clicked, func() { //注册了对话框函数
	fmt.Println("Displaying annoying dialog box!")
	wg.Done()
})
subscribe(button.Clicked, func() { //注册了鼠标点击函数
	fmt.Println("Mouse clicked.")
	wg.Done()
})

button.Clicked.Broadcast() //按钮在这里被点击

wg.Wait()
```