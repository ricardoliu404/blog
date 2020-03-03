---
title: 适配器模式-Go
date: 2020-03-03 11:21:32
cover: https://images.unsplash.com/photo-1583165278902-412729c1e624?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=60
tags:
- Go设计模式
categories:
- 理论
---

# 适配器模式

最常用的结构模式之一是适配器模式。就像在现实生活中，你有插头适配器和螺栓适配器，在Go中，适配器将允许我们使用一些在开始时不是为特定任务而构建的东西。

## 描述

适配器模式非常有用，例如，当一个接口过时，无法轻松或快速地替换它时。相反，您可以创建一个新接口来处理应用程序的当前需求，该接口在幕后使用旧接口的实现。

适配器还帮助我们在应用程序中遵循开闭原则，使它们更加可预测。它们还允许我们编写使用一些我们无法修改的基础代码。

## 目标

适配器设计模式将帮助您满足最初不兼容的两部分代码的需要。在决定适配器模式是否适合您的问题时，要记住这一点——两个不兼容但必须协同工作的接口是适配器模式的最佳候选接口（但它们也可以使用Facade模式）。

### 使用与适配器对象不兼容的接口

例如，我们有一个老旧的`Printer`接口和一个新的。新接口的调用者不希望收到旧接口的签名，因此需要一个适配器，以便用户在必要时仍可以使用旧实现（例如，使用一些遗留代码）。

### 要求与验收标准

有一个名为`LegacyPrinter`的旧接口和一个名为`ModernPrinter`的新接口，创建一个实现`ModernPrinter`接口的结构，并且可以使用`LegacyPrinter`接口，如一下步骤所述：

1. 创建实现`ModernPrinter`接口的适配器对象
2. 新适配器对象必须包含`LegacyPrinter`接口的实例
3. 当使用`ModernPrinter`时，必须私下调用，并且在前面加上文本适配器

## 实现

首先是`LegacyPrinter`的实现：

``` Go
type LegacyPrinter interface { 
    Print(s string) string
} 

type MyLegacyPrinter struct {}

func (m *MyLegacyPrinter) Print(s string) (newMsg string) { //此处在方法定义处定义了返回值变量
    newMsg = fmt.Sprintf("Legacy Printer: %s\n", s) 
    println(newMsg) 
    return //直接返回无需传递变量，因为在函数头一行已经定义
}
```

首先`LegacyPrinter`接口有一个`Print`方法，接受一个字符串，返回一段消息。`MyLegacyPrinter`结构体实现了这接个口，修改了传递的字符串，打印并返回一个带前缀的字符串。

现在定义需要适配的新接口：

``` Go
type ModernPrinter interface {
    PrintStored() string
}
```

在这种情况下，新的`PrintStored()`方法不接受任何字符串作为参数，因此需要提前在实现类中储存起来。我们把适配器类型命名为`PrinterAdapter`接口：

``` Go
type PrinterAdapter struct {
    OldPrinter LegacyPrinter
    Msg string
}

func (p *PrinterAdapter) PrintStored() (newMsg string) {
    return
}
```

前面提到，`PrinterAdapter`适配器必须有一个存储待打印字符串的字段。也必须有一个字段存储`LegacyPrinter`实例。

现在我们写一个小测试：

``` Go
func TestAdapter(t *testing.T) {
    msg := "Hello World!"
    adapter := PrinterAdapter{
        OldPrinter: &MyLegacyPrinter{},
        //此处创建一个MyLegacyPrinter实例，赋给OldPrinter
        Msg: msg,
    }
    returnedMsg := adapter.PrintStored()
    if returnedMsg != "Legacy Printer: Adapter: Hello World!\n" {
        t.Errorf("Message didn't match: %s\n", returnedMsg)
    }
    adapter = PrinterAdapter{
        OldPrinter: nil, 
        //此处传递一个空值给OldPrinter，期望按原样输出
        Msg: msg,
    } 
    returnedMsg = adapter.PrintStored()
    if returnedMsg != "Hello World!" { 
        t.Errorf("Message didn't match: %s\n", returnedMsg) 
    }
}
```

接下来需要重用`PrinterAdapter`中存储的`MyLegacyPrinter`：

``` Go
type PrinterAdapter struct {
    OldPrinter LegacyPrinter
    Msg string
}

func(p *PrinterAdapter) PrintStored() (newMsg string) {
    if p.OldPrinter != nil{
        newMsg = fmt.Sprintf("Adapter: %s", p.Msg)
        newMsg = p.OldPrinter.Print(newMsg)
    } else {
        newMsg = p.Msg
    }
    return
}
```

在`PrintStored`方法中，首先检查是否有`LegacyPrinter`实例。在此例中，我们使用存储的消息和适配器前缀组成一个新字符串，将其存储在返回变量（称为`newMsg`）中。然后我们使用指向`MyLegacyPrinter`结构的指针来使用`LegacyPrinter`接口打印合成的消息。如果`LegacyPrinter`实例不存在，就直接返回原始值。

> 源码见 https://github.com/ricardoliu404/go-design-patterns/tree/master/structural/adapter

# Go源代码中的适配器模式举例

在Go语言的源代码中你可以找到很多适配器实现。著名的`http.Handler`接口就是一个非常有趣的适配器实现。Go语言中的一个非常简单的`Hello World`服务器是这么实现的：

``` Go
package main
import ( 
    "fmt" 
    "log"
    "net/http" 
)
type MyServer struct{ 
    Msg string
}
func (m *MyServer) ServeHTTP(w http.ResponseWriter,r *http.Request){ 
    fmt.Fprintf(w, "Hello, World")
} 
func main() {
    server := &MyServer{ 
        Msg:"Hello, World",
    } 
    http.Handle("/", server)
    log.Fatal(http.ListenAndServe(":8080", nil)) 
}
```

HTTP包有一个函数`Handle`（类似Java中的静态方法）接受两个参数——一个字符串代表路径，还有一个`Handler`接口。`Handler`接口看起来像下面这样：

``` Go
type Handler interface {
    ServeHTTP (ResponseWriter, *Request)
}
```

我们需要实现一个`ServeHTTP`方法，服务端的HTTP链接会运行处理上下文。但是还有一个方法`HandlerFunc`允许你定义一些终端表现：

``` Go
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request){
        fmt.Fprintf(w, "Hello, World")
    })
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

HandleFunc函数实际上是适配器的一部分，用于将函数直接用作ServeHTTP实现。再慢慢读最后一句话-你能猜出是怎么做到的吗？

``` Go
type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

我们可以像定义结构一样定义一个函数类型。我们将此函数类型设置为实现`ServeHTTP`方法。最后，从`ServeHTTP`函数中，我们调用接收器本身`f（w，r）`。

您必须考虑Go的隐式接口实现。当我们定义像`func（ResponseWriter，*Request）`这样的函数时，它被隐式地识别为`HandlerFunc`。因为HandleFunc函数实现了处理程序接口，所以我们的函数也隐式地实现了处理程序接口。这听起来你熟悉吗？如果`A=B`和`B=C`，那么`A=C`.隐式实现提供了很大的灵活性和能力，但是您也必须小心，因为您不知道某个方法或函数是否正在实现某个接口，而该接口可能会引发不受欢迎的行为。

我们可以在Go的源代码中找到更多的例子。`io`包还有一个使用管道的强大示例。Linux中的管道是一种流机制，它接受输入上的某些内容，并在输出上输出其他内容。`io`包有两个接口，在Go的源代码中到处都使用—`io.Reader`和`io.Writer`接口：

``` Go
type Reader interface { 
    Read(p []byte) (n int, err error) 
} 

type Writer interface { 
    Write(p []byte) (n int, err error) 
}
```

我们在任何地方都使用`io.Reader`，例如，当您使用`os.open file`打开一个文件时，它返回一个文件，实际上，该文件实现了`io.Reader`接口。为什么有用？假设您编写了一个计数器结构，它会从您提供的数字到零计数：

``` Go
type Counter struct {}
func (f *Counter) Count(n uint64) uint64 { 
    if n == 0 {
        println(strconv.Itoa(0)) 
        return 0
    } 
    cur := n
    println(strconv.FormatUint(cur, 10)) 
    return f.Count(n - 1)
}
//如果输入为3
//输出为
/*
 *3
 *2
 *1
*/
```

简单的~~几何学~~递归。但如果我们想要写到文件里呢？则需要实现这个函数一遍。如果我们需要输出到控制台呢？还要实现一遍。我们需要用`io.Writer`吧这个函数结构化一些：

``` Go
type Counter struct {
    Writer io.Writer
}

func (f *Counter) count(n uint64) uint64{
    if n == 0 {
        f.Writer.Write([]byte(strconv.Itoa(0) + "\n"))
        return 0
    }
    cur := n
    f.Writer.Write([]byte(strconv.FormatUint(cur, 10) + "\n"))
    return f.Count(n - 1)
}
```

现在我们有了一个`io.Writer`在`Writer`字段中。这样一来，我们可以这样创建计数器`c := Counter{os.Stdout}`，我们就会获得控制台的`Writer`。但还是没有解决我们想要把倒数输出到许多控制台。我们可以写一个新的适配器，使用一个`Pipe()`对读方和写方建立连接。这样一来，就可以解决`Reader`和`Writer`接口不兼容的问题。

实际上，我们不需要编写适配器——Go的`io`库在`io.Pipe（）`中为我们提供了一个适配器。管道将允许我们将`Reader`转换为`Writer`。`Pipe（）`方法将提供一个`Writer`（管道的入口）和一个`Reader`（出口）供我们使用。因此，让我们创建一个管道，并将提供的`writer`分配给前面示例的计数器：

``` Go
pipeReader, pipeWriter := io.Pipe()
defer pw.Close()
defer pr.Close()

counter := Counter{
    Writer: pipeWriter,
}
```

现在我们有了一个`Reader`接口，在那里我们以前有一个`Writer`。我们在哪里可以使用`Reader`？`io.TeeReader`函数帮助我们将数据流从读接口复制到写接口，并返回一个新的`Reader`，您仍然可以使用该`Reader`再次将数据流传输到第二个`Writer`。因此，我们将数据从同一个`Reader`流到两个`Writer`-`file`和`StdOut`。

``` Go
tee := io.TeeReader(pipeReader, file)
```

现在我们将会往向`TeeReader`方法传递的文件中写入，我们还需要向控制台打印。`io.Copy`适配器可以向`TeeReader`一样使用，接受一个`Reader`，并向`Writer`写入。