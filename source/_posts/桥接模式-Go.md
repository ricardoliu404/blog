---
title: 桥接模式-Go
date: 2020-03-03 15:12:39
cover: https://images.unsplash.com/photo-1583165278902-412729c1e624?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=60
tags:
- Go设计模式
categories:
- 理论
---

# 桥接模式

从最初的*四人帮*开始，桥接就是一个有点神秘的定义。它将抽象与其实现分离，以便两者可以独立变化。这种神秘的解释只是意味着您甚至可以将最基本的功能形式分离：将对象与它所做的事情分离

## 描述

桥接模式尝试像往常一样将事物与设计模式分离。它将抽象（对象）与其实现（对象所做的事情）分离。这样，我们可以随心所欲地改变一个对象所做的事情。它还允许我们在重用相同实现的同时更改抽象对象。

## 目标

桥接模式的目标是为经常变化的结构带来灵活性。知道一个方法的输入和输出，我们就可以在不知道太多的情况下修改代码，并且让双方都有更容易修改的自由。

例如：我们将使用控制台输出抽象来保持简单，并将有两种实现。第一种向控制台写入，前面学过`io.Writer`接口，我们将对`io.Writer`接口进行第二次写入，以便为解决方案提供更大的灵活性。我们还将有两个实现的抽象对象用户：一个普通对象，它将以简单的方式使用每一个实现；另一个Packt实现，将Packt中的语句消息追加到打印消息中。

### 要求与验收标准

上面提到，我们会有两个对象（`Packt`和`Normal`）和两个实现（`PrinterImpl1`和`PrinterImpl2`），我们将使用桥接模式将他们连接在一起。如下要求与验收标准：

- 一个`PrinterAPI`接口，获取信息并打印
- 一个API实现，将信息打印到控制台
- 一个API实现，将信息输出到`io.Writer`
- 一个抽象的`Printer`，拥有`Print`方法实现具体打印类型
- 一个`normal`打印对象，实现`Printer`和`PrinterAPI`接口
- `normal`打印对象将把信息直接转发到实现
- `Packt`打印对象实现`Printer`和`PrinterAPI`接口
- `Packt`打印对象会为信息添加“Message from Packt：”前缀

### 测试用例

此接口定义了一个`PrintMessage(string)`方法，将输入的参数打印出来：

``` Go
type PrinterAPI interface {
    PrintMessage(string) error
}
```

``` Go
type PrinterImpl1 struct {}

func (p *PrinterImpl1) PrintMessage(msg string) error {
    fmt.Printf("%s\n", msg)
	return nil
}
```

`PrinterImpl1`是一个`PrinterAPI`实现类，`PrintMessage`尚未实现，仅返回一个错误，但是足够我们写单元测试了：

``` Go
func TestPrinAPI1(t *testing.T) {
    api1 := PrinterImpl1{}

    err := api1.PrintMessage("Hello")
    if err != nil {
        t.Errorf("Error trying to use the API1 implementation: Message: %s\n", err.Error())
    }
}
```

单元测试中创建了一个`PrinterImpl1`实例，用`PrintMessage`方法打印Hello到控制台，目前没有实现，所以返回错误。

`PrinterImpl2`实现了`PrinterAPI`接口，并存储了一个`io.Writer`的实现。

``` Go
type PrinterImpl2 struct {
    Writer io.Writer
}

func (d *PrinterImpl2) PrintMessage(msg stirng) error {
    if p.Writer == nil {
		return errors.New("You need to pass an io.Writer to PrinterImpl2")
	}
	_, _ = fmt.Fprintf(p.Writer, "%s", msg)
	return nil
}
```

我们创建一个`io.Writer`的实现，并在本地字段中保存写入的东西。这样就可以检查我们往`Writer`中写的是什么：

``` Go
type TestWriter struct {
    Msg string
}

func (t *TestWriter) Write(p []byte) (n int, err error) {
    n = len(p)
    if n > 0 {
        t.Msg = string(p)
        return n, nil
    }
    err = errors.New("Content received on Writer was empty")
    return
}
```

在我们的测试对象中，写入字段前判断是否为空，如果是空则返回错误，如果不是，我们把输入字符保存到`Msg`字段中。我们利用这个做一个测试：

``` Go
func Test PrintAPI2(t *testing.T) {
    api2 := PrinterImpl2{}
    err := api2.PrintMessage("Hello")
    if err != nil{
        expectedErrorMessage := "You need to pass an io.Writer to PrinterImpl2"
        if !string.Contains(err.Error(), expectedErrorMessage) {
            t.Errorf("Error message was not correct.\n
            Actual: %s\nExpected: %s\n", err.Error(). exceptedErrorMessage)
        }
    }

    testWriter := TestWriter{}
    api2 = PrinterImpl2{
        Writer: &testWriter,
    }

    expectedMessage := "Hello"
    err = api2.PrintMessage(expectedMessage)
    if err != nil {
        t.Errorf("Error trying to use the API2 implementation: %s\n", err.Error())
    }

    if testWriter.Msg != expectedMessage {
        t.Fatalf("API2 did not write correctly on the io.Writer. \n Actual: %s\nExpected: %s\n", testWriter.Msg, expectedMessage)
    }
}
```

测试用例的上半部分，我们创建了一个叫做api2的PrinterImpl2实例。并没有给他传递一个io.Writer实例，因此我们首先检查我们会收到一个错误（"You need to pass an io.Writer to PrinterImpl2"），接着我们尝试调用PrintMessage方法。

测试用例的下班部分我们将io.Writer的实例传递给api2的Writer字段，看看我们会不会收到错误消息。然后检查testWriter.Msg域，如果没有问题，应该会包括`Hello`。 

接下来是桥接模块：

``` Go
type PrinterAbstraction interface {
	Print() error
}

type NormalPrinter struct {
	Msg     string
	Printer PrinterAPI
}

func (c *NormalPrinter) Print() error {
	c.Printer.PrintMessage(c.Msg)
	return nil
}

type PacktPrinter struct {
	Msg     string
	Printer PrinterAPI
}

func (c *PacktPrinter) Print() error {
	c.Printer.PrintMessage(fmt.Sprintf("Message from Packt: %s", c.Msg))
	return nil
}
```

具体测试设计如下：

``` Go
func TestNormalPrinter_Print(t *testing.T) {
	expectedMessage := "Hello io.Writer"

	normal := NormalPrinter{
		Msg:     expectedMessage,
		Printer: &PrinterImpl1{},
	}

	err := normal.Print()
	if err != nil {
		t.Errorf(err.Error())
	}

	testWriter := TestWriter{}
	normal = NormalPrinter{
		Msg: expectedMessage,
		Printer: &PrinterImpl2{
			Writer: &testWriter,
		},
	}
	err = normal.Print()
	if err != nil {
		t.Error(err.Error())
	}
	if testWriter.Msg != expectedMessage {
		t.Errorf("The expected message on the io.Writer doesn't match actual.\n Actual: %s\nExpected: %s\n", testWriter.Msg, expectedMessage)
	}
}

func TestPacktPrinter_Print(t *testing.T) {
	passedMessage := "Hello io.Writer"
	expectedMessage := "Message from Packt: Hello io.Writer"
	packt := PacktPrinter{
		Msg:     passedMessage,
		Printer: &PrinterImpl1{},
	}
	err := packt.Print()
	if err != nil {
		t.Errorf(err.Error())
	}
	testWriter := TestWriter{}
	packt = PacktPrinter{
		Msg: passedMessage,
		Printer: &PrinterImpl2{
			Writer: &testWriter,
		},
	}
	err = packt.Print()
	if err != nil {
		t.Error(err.Error())
	}
	if testWriter.Msg != expectedMessage {
		t.Errorf("The expected message on the io.Writer doesn't match actual.\n Actual: %s\nExpected: %s\n", testWriter.Msg, expectedMessage)
	}
}
```

>源码见 https://github.com/ricardoliu404/go-design-patterns/tree/master/structural/bridge