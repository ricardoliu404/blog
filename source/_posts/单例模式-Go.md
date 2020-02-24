---
title: 单例模式-Go
date: 2020-02-24 09:55:09
tags:
- Go设计模式
categories:
- 理论
---

# 单例模式——在整个程序中某类型只有唯一实例

## 描述

单例模式提供某类型的唯一实例，并且保证没有其他副本。在第一次调用实例的时候创建实例，该实例后续在程序其他地方被复用。

## 预期目标

- 我们需要某种特定类型的，单一的、共享的值。
- 在整个程序中某些类型的对象数量限制为一。

## 例子

下述例子中实现了一个具有唯一计数变量`count`的`singleton`。

其中`Singleton`为接口，所有实现了接口中定义的`AddOne() int`方法的`struct`默认实现了该接口，`singleton struct`中定义了一个指向`singleton`类型的指针`instance`。`singleton`结构体对外唯一暴露了一个`GetInstance`方法，该方法默认返回一个实现了`Singleton`接口的结构体。
``` Go
package singleton

//Singleton为接口，定义了一个返回值为int的AddOne()方法，所有实现了该方法的结构体默认为该接口的“实现类”
type Singleton interface {
	AddOne() int
}

//定义singleton结构体，具有一个int型的变量count
type singleton struct {
	count int
}

//定义一个指向singleton的指针instance，默认为nil类型
var instance *singleton

//GetInstance()暴露了一个返回Singleton实现类的方法
func GetInstance() Singleton {
	if instance == nil {
		instance = new(singleton)
	}
	return instance
}

//AddOne方法传入一个指向singleton的指针，对该struct的count字段进行自增操作，并返回该值
func (s *singleton) AddOne() int {
	s.count++
	return s.count
}
```

## 测试用例

``` Go
package singleton

import (
	"testing"
)

func TestGetInstance(t *testing.T) {
    //counter1为该测试用例中的第一个singleton实例
	counter1 := GetInstance()

    //如果该实例为空，则获取实例失败，有内鬼，终止交易
	if counter1 == nil {
		t.Error("Expected pointer to Singleton after calling GetInstance(), not nil")
	}

    //对当前实例的count进行自增，并获取当前实例的count，如果值不为1，有内鬼，终止交易
	expectedCounter := counter1
	currentCount := counter1.AddOne()
	if currentCount != 1 {
		t.Errorf("After calling for the first time to count, the count must be 1 but it is %d\n", currentCount)
    }
    
    //counter2为该测试用例中的第二个singleton实例，如果counter2和expectedCounter不一致，则获取的实例不唯一，有内鬼，终止交易
	counter2 := GetInstance()
	if counter2 != expectedCounter {
		t.Error("Expected same instance in counter2 but it got a different instance")
	}

    //counter2的count自增，如果不是2，有内鬼，终止交易
	currentCount = counter2.AddOne()
	if currentCount != 2 {
		t.Errorf("After calling 'AddOne' using the second counter, the current count must be 2 but was %d\n", currentCount)
	}
}
```

## 运行结果

``` Shell
$go test -v
=== RUN   TestGetInstance
--- PASS: TestGetInstance (0.00s)
PASS
ok      github.com/ricardoliu404/go-design-patterns/creational/singleton        0.151s
```
