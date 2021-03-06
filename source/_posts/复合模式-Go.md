---
title: 复合模式-Go
date: 2020-03-02 10:50:01
cover: https://images.unsplash.com/photo-1583038152192-544136533345?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=500&q=60
tags:
- Go设计模式
categories:
- 理论
---

结构模式通过常用的结构和关系帮我们塑造应用，在Go语言中最常用的模式是**复合模式**，因为在Go语言中没有继承的概念，而鼓励**复合**的使用

# 复合模式

复合设计模式倾向于组合（通常定义为具有关系）而不是继承（即关系）。自90年代以来，组合继承方法一直是工程师们讨论的话题。我们将学习如何使用`has a`方法创建对象结构。总之，Go没有继承权，因为它不需要继承权！

## 描述
在复合设计模式中，您将创建对象的层次结构和树。对象有不同的对象，其中有自己的字段和方法。这种方法非常强大，解决了许多继承和多重继承的问题。例如，一个典型的继承问题是当一个实体从两个完全不同的类继承时，这两个类之间完全没有关系。想象一下训练的运动员和游泳的游泳运动员。

- `Athlete`类有`Train()`方法
- `Swimmer`类有`Swim()`方法

`Swimmer`类继承自`Athlete`，因此也继承了`Train`方法并且声明了自己的`Swim`方法。同时也可以有一个自行车运动员`Cyclist`声明一个`Ride`方法。但是如果现在有一个会吃东西的动物，像狗会叫一样：

- `Cyclist`有`Ride`方法
- `Animal`类有`Eat`和`Bark`方法

你也可以有一个鱼，当然可以游泳，但是如何解决呢？鱼不可以像运动员一样训练。你可以创建一个运动员接口，拥有一个swim方法，让游泳运动员和鱼实现它。这应该是最好的实现了，但你必须实现swim方法两次，代码重用性则会受到影响。如果是铁人三项呢？运动员需要游泳跑步和骑车。多重的接口实现可以暂时解决这个问题，但很快就没法维护了。

## 目标

复合模式就是为了避免这种会导致应用臃肿、代码不清洁的层级地狱。我们可以使用Go实现两种复合模式：直接复合**direct composition**和嵌入复合**embedding composition**。

## 示例

复合模式是一种纯纯结构模式，对于结构本身没什么可测试的。因此这次不会写单元测试，在此简单的描述复合模式的创建。

首先从`Athlete`结构和`Train()`方法开始

``` Go
type Athlete struct{}

func (a *Athlete) Train() {
    fmt.Println("Training)
}
```

上述代码很直接，`Train()`方法打印字符串，现在再创建一个游泳运动员A，将`Athlete`组合在里面。

``` Go
type CompositeSwimmerA struct {
    MyAthlete Athlete
    MySwim func()
}
```

组合类型`CompositeSwimmerA`有一个`Athlete`类型的数据域`MyAthlete`，同时还有一个`func()`。在Go中，函数是“一等公民”，他们可以像任何变量一样用作参数、字段。因此`CompositeSwimmerA`有一个`MySwim`字段，该字段存储一个闭包，不接受任何参数也不返回任何内容。我们怎么才能为他分配函数呢？我们创建一个与`func()`类型匹配的函数（没有参数， 没有返回值）。

``` Go
func Swim(){
    fmt.Println("Swimming!")
}
```

打完收工。`Swim()`方法没有参数没有返回值，因此可以用在`CompositeSwimmerA`结构体的`MySwim`字段上：

``` Go
swimmer := CompositeSwimmerA{ 
    MySwim: Swim,
}
swimmer.MyAthlete.Train() 
swimmer.MySwim()
```

现在有一个叫`Swim()`的函数，我们将该函数分配到`MySwim`字段上。注意，`Swim`类型没有将执行其内容的括号。这样我们把整个函数复制到`MySwim`方法。此时我们没有传递任何运动员到`MyAthlete`字段但是已经可以使用使用了。

``` Bash
$ go run main.go
Training
Swimming!
```

其实这是由于Go语言的自动初始化。如果你不给`CompositeSwimmerA`传递一个`Athlete`结构体，编译器会为他自动创建一个，并赋字段初值为0。再看看上面对`CompositeSwimmerA`的定义，再来解决鱼的问题就不难了。首先我们创建一个`Animal`结构体：

``` Go
type Animal struct{}

func (a *Animal) Eat () {
    fmt.Println("Eating")
}
```

再创建一个`Shark`对象嵌入一个`Animal`对象：

``` Go
type Shark struct {
    Animal
    Swim func()
}
```

等下，`Animal`类型字段的名字去哪了？还记得**嵌入**这个词吗？在Go里，你可以把对象嵌入对象里，看起来就像是继承。也就是说，我们不必显式调用字段名来访问其字段和方法，因为它们将是我们的一部分。这样说的话下面的代码就很ok了：

``` Go
fish := Shark{
    Swim: Swim,
}
fish.Eat()
fish.Swim()
```

现在我们有了一个有初始值的、嵌入的`Animal`类型。这就是为什么可以在不创建或者使用内部字段名的情况下调用`Eat()`方法。

还有第三个方法使用复合模式。我们可以创建一个`Swimmer`接口，定义一个`Swim()`方法和一个`SwimmerImpl`类型嵌入到游泳运动员中。

``` Go
type Swimmer interface {
    Swim()
}

type Trainer interface {
    Train()
}

type SwimmerImpl struct {}
func (s *SwimmerImpl) Swim() {
    fmt.Println("Swimming!")
}

type CompositeSwimmerB struct {
    Trainer
    Swimmer
}
```

用这种方法可以更明确的控制对象的创建。`Swimmer`字段是嵌入的，但是不会被自动初始化，因为它指向一个接口。正确的使用方法是：

``` Go
swimmer := CompositeSwimmerB {
    &Athlete{},
    &SwimmerImpl{},
}

swimmer.Train()
swimmer.Swim()
```

哪种方法更妙？个人来讲，接口方法是最好的。首先，面向接口编程、其次，你不会把代码的一部分留给编译器的零初始化特性。这是一个非常强大的特性，但必须小心使用，因为它会导致运行时问题，在编译时使用接口时会发现这些问题。实际上，在不同的情况下，零初始化将在运行时为您节省时间！但我更喜欢尽可能多地使用接口，所以这实际上不是一个选项。

## 二叉树复合（Binary Tree Compositions）

另一种常见的符合模式是在模拟二叉树时，需要在字段中存储本身类型（左右孩子）：

``` Go
type Tree struct {
    LeafValue int
    Right *Tree
    left *Tree
}
```

这是一种递归复合，并且由于递归的特性，我们必须用指针才能让编译器知道需要留多少内存。`Tree`这个结构体为每个实例存储了`LeafValue`对象，还有`Left`和`Right`中的两个新`Tree`。

我们可以这样创建一个对象：

``` Go
root := Tree{
    LeafValue: 0,
    Right: &Tree{
        LeafValue: 5,
        Right: &Tree{
            6, nil, nil,
        }
        Left: nil,
    },
    Left: &Tree{
        4, nil, nil,
    }
}
//          0
//        /   \
//       4     5
//      / \   / \
//     n   n n   6
//              / \
//             n   n
```

# 复合模式和继承

我们在Go中使用组合设计模式时，必须明确与继承区分开。例如当我们把`Parent`结构嵌入到`Son`结构中：

``` Go
type Parent struct { 
    SomeField int
}
type Son struct { 
    Parent
}
```

我们不能把`Son`当做一个`Parent`，也就是说当一个函数需要一个`Parent`实例时不能把`Son`传递给他：

``` Go
func GetParentField(p *Parent) int {
    fmt.Println(p.SomeField)
}
```

如果试图传递一个`Son`实例给`GetParentField`，就会得到一个错误信息

> **cannot use son (type Son) as type Parent in argument to GetParentField**

如果非要这样的话，可以吧`Parent`作为一个字段而不是嵌入进`Son`中去：

``` Go
type Son struct {
    P Parent
}
```

现在可以通过`P`这个字段传递`Parent`给`GetParentField`了：

``` Go
son := Son{}
GetParentField(son.P)
```

> 源码见 https://github.com/ricardoliu404/go-design-patterns/tree/master/structural/composite