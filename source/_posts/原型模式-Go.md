---
title: 原型模式-Go
date: 2020-02-27 13:37:15
cover: https://images.unsplash.com/photo-1582464352942-e117a8d6bdc5?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1350&q=80
tags:
- Go设计模式
categories:
- 理论
---

# 原型模式

## 描述

原型模式的目标是在**编译期**就获得一个或一系列可以在**运行期**无限次克隆的对象。比如用户注册时的默认模板或者某些服务的默认价格方案。原型模式与建造者模式最大的不同点在于，原型模式获取的对象时克隆出来的而不是在运行时建造出来的。也可以使用一个类似缓存的解决方案，用原型存储信息。

## 目标

原型设计模式的主要目标是避免重复创建对象。 想象一下一个默认的对象，它由几十个字段和嵌入的类型组成。 我们不想每次使用对象时都编写这种类型所需的所有内容，特别是如果我们可以通过创建具有不同基础的实例来弄乱它的话：

- 维护一组将被克隆以创建新实例的对象
- 提供某种类型的默认值以在其上开始工作
- 释放复杂对象初始化的CPU以占用更多的内存资源

## 例子

实现一个衬衣商店，里面有一些衬衣（包括默认的颜色和价格）。所有衬衣都会有库存，还有一个系统去维护商品存放的地址。

我们使用衬衣的原型，每次我们需要一个新衬衣时，就将此原型拿出来，克隆和使用。

- 要具有衬衫克隆对象和接口来查询不同类型的衬衫（分别为15.00、16.00和17.00美元的白色，黑色和蓝色衬衫）
- 当您要求一件白衬衫时，必须制作一件白衬衫的副本，并且新实例必须与原始实例不同。
- 创建的对象的SKU不应影响新对象的创建
- 信息方法必须为我提供实例字段上的所有可用信息，包括更新的SKU

``` Go
type ItemInfoGetter interface {
	GetInfo() string
}

type ShirtColor byte
type Shirt struct {
	Price float32
	SKU   string
	Color ShirtColor
}

func (s *Shirt) GetInfo() string {
	return fmt.Sprintf("Shirt with SKU '%s' and Color id %d that costs %f\n", s.SKU, s.Color, s.Price)
}

func (s *Shirt) GetPrice() float32 {
	return s.Price
}

var whitePrototype *Shirt = &Shirt{
	Price: 15.00,
	SKU:   "empty",
	Color: White,
}
var blackPrototype *Shirt = &Shirt{
	Price: 16.00,
	SKU:   "empty",
	Color: Black,
}
var bluePrototype *Shirt = &Shirt{
	Price: 17.00,
	SKU:   "empty",
	Color: Blue,
}
```

`Shirt`结构体实现了`ItemInfoGetter`接口，具有打印具体信息的功能，同时记录了价格，库存和颜色。（强行实现一个接口可能是为了“面向接口编程”），同时定义了预设对象`whitePrototype`等，该对象在编译期就已经产生。

随后定义一个`ShirtCache`结构体，具有根据输入参数返回相应预设对象指针的作用

``` Go
const (
	White = 1
	Black = 2
	Blue  = 3
)
type ShirtCache struct{}

func (s *ShirtCache) GetClone(m int) (ItemInfoGetter, error) {
	switch m {
	case White:
		newItem := *whitePrototype
		return &newItem, nil
	case Black:
		newItem := *blackPrototype
		return &newItem, nil
	case Blue:
		newItem := *bluePrototype
		return &newItem, nil
	default:
		return nil, errors.New("Shirt model not recognized")
	}
}
```

注意，此处`newItem := *whitePrototype`以及返回时的`&newItem`的作用是拷贝一份`whitePrototype`到返回值中。

## 测试
测试中可以看到，首先获取shirtCache实例，调用GetClone方法获得颜色为White的whitePrototype的**拷贝**，确认该拷贝不是原型本型（即`item1 != whitePrototype`），再确认该拷贝确实属于`Shirt`类型。重复以上步骤得到`item2`，再断言`item1 != item2`，也就是根据同一原型产生的实例不应该相同，最后根据实例的内存地址确认两者确实不是同一对象。

``` Go
func TestClone(t *testing.T) {
	shirtCache := GetShirtCloner()
	if shirtCache == nil {
		t.Fatal("Received cache was nil")
	}

	item1, err := shirtCache.GetClone(White)
	if err != nil {
		t.Error(err)
	}
	if item1 == whitePrototype {
		t.Error("item1 cannot be equal to the white prototype")
	}
	shirt1, ok := item1.(*Shirt)
	if !ok {
		t.Fatal("Type assertion for shirt1 couldn't be done successfully")
	}
	shirt1.SKU = "abbcc"
	item2, err := shirtCache.GetClone(White)
	if err != nil {
		t.Fatal(err)
	}
	shirt2, ok := item2.(*Shirt)
	if !ok {
		t.Fatal("Type assertion for shirt1 couldn't be done successfully")
	}
	if shirt1.SKU == shirt2.SKU {
		t.Error("SKU's of shirt1 and shirt2 must be different")
	}
	if shirt1 == shirt2 {
		t.Error("Shirt 1 cannot be equal to Shirt 2")
	}
	t.Logf("LOG: %s", shirt1.GetInfo())
	t.Logf("LOG: %s", shirt2.GetInfo())
	t.Logf("LOG: The memory positions of the shirts are different %p != %p \n\n", &shirt1, &shirt2)
}
```

> 完整代码见 https://github.com/ricardoliu404/go-design-patterns/tree/master/creational/prototype