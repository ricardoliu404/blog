---
title: 建造者模式-Go
date: 2020-02-24 13:19:39
cover: https://gratisography.com/wp-content/uploads/2020/02/gratisography-loud-shirt-900x600.jpg
tags:
- Go设计模式
categories:
- 理论
---

# 建造者模式——接口多种实现的重用方法

## 描述

对象的创建可以简单到只有花括号`{}`，也可以复杂到需要调用API，检查状态，为属性创建对象。在Go语言中不支持继承，复用的特性则来自多个结构体的组合，所以一个对象的实例化需要多个对象组合在一起。

与此同时，你可能需要基本相同的技术去创建许多不同类型的对象。例如建造一辆轿车和一辆巴士的技术基本相同，不同的仅仅是车的大小，座位数等等，为了重用建造过程，就有了建造者模式。

## 预期目标

- 将复杂的创建抽象出来，把创建和使用分开
- 通过一步一步填充属性和内嵌对象来创建一个对象
- 在多个对象之间重用对象创建方法

## 例子

建造者模式基本上可以概括为**一个指导人**，**几个建造者**和他们产出的**产品**之间的关系。

下面介绍一个整车制造的例子

对于造一辆车来说，或多或少都有相同的工序，例如选择车辆的类型、整合车架、放置轮胎和放置座椅等等。

![本例中建造者模式UML图](UML.jpg)

首先我们定义需要产出的车辆的结构体，一个名为`VehicleProduct`的结构体，包括轮胎、座椅和车架三个部分。

``` Go
type VehicleProduct struct {
	Wheels    int
	Seats     int
	Structure string
}
```

然后我们定义一个产生`VechicleProduct`的建造工序`BuildProcess`，`BuildProcess`是一个接口，定义了三个装配方法返回中间产物和一个建造方法返回最终产物。
``` Go
type BuildProcess interface {
	SetWheels() BuildProcess
	SetSeats() BuildProcess
	SetStructure() BuildProcess
	Build() VehicleProduct
}
```


然后我们定义一个指导人`ManufacturingDirector`，指导人有一个属性保存了`BuilderProcess`，对应着本次生产应该基于哪一种具体要求。同时提供一个`SetBuilder()`方法，设置当前`BuilderProcess`，以及一个`Construct()`方法，依次调用`BuilderProcess`的各个方法，完成整车制造。
``` Go
type ManufacturingDirector struct {
	builder BuildProcess
}

func (f *ManufacturingDirector) Construct() {
	f.builder.SetSeats().SetStructure().SetWheels()
}

func (f *ManufacturingDirector) SetBuilder(b BuildProcess) {
	f.builder = b
}
```

最后我们定义一个具体的小轿车的制造方法，实现BuildProcess接口，定义一些具体的特定的工作。其中`SetWheels`、`SetSeats`、`SetStructure`三个方法是加工过程，可见返回类型为`BuildProcess`，最后的`Build`方法返回值则为真正出厂的产品。
``` Go
type CarBuilder struct {
	v VehicleProduct
}

func (c *CarBuilder) SetWheels() BuildProcess {
	c.v.Wheels = 4
	return c
}

func (c *CarBuilder) SetSeats() BuildProcess {
	c.v.Seats = 5
	return c
}

func (c *CarBuilder) SetStructure() BuildProcess {
	c.v.Structure = "Car"
	return c
}

func (c *CarBuilder) Build() VehicleProduct {
	return c.v
}
```

## 测试
``` Go
func TestBuilderPattern(t *testing.T) {
	manufacturingComplex := ManufacturingDirector{}

	carBuilder := &CarBuilder{}
	manufacturingComplex.SetBuilder(carBuilder)
	manufacturingComplex.Construct()

	car := carBuilder.Build()

	if car.Wheels != 4 {
		t.Errorf("Wheels on a car must be 4 and they were %d\n", car.Wheels)
	}

	if car.Structure != "Car" {
		t.Errorf("Structure on a car must be 'Car' and was %s\n", car.Structure)
	}

	if car.Seats != 5 {
		t.Errorf("Seats on a car must be 5 and they were %d\n", car.Seats)
	}

	bikeBuilder := &BikeBuilder{}
	manufacturingComplex.SetBuilder(bikeBuilder)
	manufacturingComplex.Construct()

	motorbike := bikeBuilder.Build()

	if motorbike.Wheels != 2 {
		t.Errorf("Wheels on a motorbike must be 2 and they were %d\n", motorbike.Wheels)
	}

	if motorbike.Structure != "Motorbike" {
		t.Errorf("Structure on a motorbike must be 'Motorbike' and was %s\n", motorbike.Structure)
	}

	if motorbike.Seats != 2 {
		t.Errorf("Seats on a motorbike must be 2 and they were %d\n", motorbike.Seats)
	}
}
```

## 运行结果
``` bash
$go test -v
=== RUN   TestBuilderPattern
--- PASS: TestBuilderPattern (0.00s)
PASS
ok      github.com/ricardoliu404/go-design-patterns/creational/builder  0.141s
```

> 源码见https://github.com/ricardoliu404/go-design-patterns/tree/master/creational/builder