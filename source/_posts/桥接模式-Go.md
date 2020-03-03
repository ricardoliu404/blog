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

#### 1. `PrinterAPI`接口

此接口定义了一个`PrintMessage(string)`方法，将输入的参数打印出来：

``` Go
type PrinterAPI interface {
    PrintMessage(string) error
}
```

#### 2. `PrinterImpl1`——简单API实现

``` Go
type PrinterImpl1 struct {}

func (p *PrinterImpl1) PrintMessage(msg string) error {
    return errors.New("Not implemented yet")
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

#### 3. `PrinterImpl1`实现向io.Writer写入

``` Go
type PrinterImpl2 struct {
    Writer io.Writer
}
```
