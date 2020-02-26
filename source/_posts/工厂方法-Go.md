---
title: 工厂方法-Go
cover: https://images.unsplash.com/photo-1582517339790-63168430ee86?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1951&q=80
date: 2020-02-24 19:15:53
tags:
- Go设计模式
categories:
- 理论
---

# 工厂方法——委托创建不同类型

工厂方法（或工厂）是工业实践中第二常用的设计模式。其目的是要达到特定的目标而需要实现的结构只是中将用户抽离出来（不是人话），例如从网络服务或者数据库中检索一些值。用户只需要一个获取值的接口。如果需要，还可以简化底层类型实现的降级或者升级过程（还不是人话）。

## 描述

使用Factory方法设计模式时，我们获得了额外的封装层，以便我们的程序可以在受控环境中增长。 使用Factory方法，我们将对象族的创建委派给不同的包或对象，以从我们可以使用的可能对象库的知识中抽象出来。 想象一下，您想使用旅行社安排假期。 您无需与酒店和旅行打交道，只需告诉旅行社您感兴趣的目的地，即可为您提供所需的一切。 旅行社代表旅行工厂。

## 目标

- 将创建新的结构实例委托给程序的不同部分
- 在接口级别工作，而不是具体的实现
- 对对象族进行分组以获得族对象创建者

## 例子

在我们的示例中，将实现工厂付款方式，这将为我们提供在商店中付款的不同方式。首先，我们将有两种付款方式——现金和信用卡。我们还将有一个与方法Pay的接口，每个要用作支付方法的结构都必须实现。

接受标准如下：
- 对称为“付款”的每种付款方式都有通用的方式
- 为了能够将付款方式的创建委托给工厂
- 只需将其添加到工厂方法即可将更多付款方法添加到库中


工厂方法非常简单，我们只要明确我们需要为接口存储多少实现，再提供一个可以传递支付方式类型的方法`GetPaymentMethod`即可。

``` Go
type PaymentMethod interface {
	Pay(amount float32) string
}
```

上述代码定义了支付方法的接口，工厂方法会返回实现该接口的实例。

``` Go
const (
	Cash      = 1
	DebitCard = 2
)
```

我们将工厂的支付方法定义成常量才能在包的外部调用、检查可能的支付方法。

为了完成工厂的定义，我们创建两个支付方法`Cash`和`DebitCard`。这两种支付方法结构体分别实现了`PaymentMethod`接口，返回包括支付内容的信息。

``` Go
type CashPM struct{}
type DebitCardPM struct{}

func (c *CashPM) Pay(amount float32) string {
	return fmt.Sprintf("%0.2f paid using cash\n", amount)
}

func (d *DebitCardPM) Pay(amount float32) string {
	return fmt.Sprintf("%0.2f paid using debit card\n", amount)
}
```

接下来是为我们创建对象的方法，它返回一个指针，指向一个实现了`PaymentMethod`接口的对象，如果请求了未注册的支付方法则会返回一个错误。

``` Go
func GetPaymentMethod(m int) (PaymentMethod, error) {
	switch m {
	case Cash:
		return new(CashPM), nil
	case DebitCard:
		return new(DebitCardPM), nil
	default:
		return nil, errors.New(fmt.Sprintf("Payment method %d not recognized\n", m))
	}
}
```

## 测试用例

首先创建测试用例，实现一个通用方法，取回实现`PaymentMethod`接口的对象。

`GetPaymentMethod`是一个取回支付方法的通用方法（不是人话），我们使用常量`Cash`作为参数（如果我们在包外使用该常量，我们需要带上包名`factory.Cash`）。检查`err`是否返回了错误，注意当我们获得了一个error的时候，调用了`t.Fatal`去停止测试运行，如果像之前几个测试中只使用`t.Error`的话，在下面尝试获取一个空对象的`Pay`方法时会出现问题，测试程序会崩溃退出。我们继续通过传递一个数值10.30给`Pay`方法，返回的信息仍旧包括`paid using cash`。`t.Log(string)`是`testing`中的一个特殊方法，如果我们传递了`-v`参数则可以打印一些日志。

``` Go
func TestCreatePaymentMethodCash(t *testing.T) {
	payment, err := GetPaymentMethod(Cash)
	if err != nil {
		t.Fatal("A payment method of type 'Cash' must exist")
	}

	msg := payment.Pay(10.30)
	if !strings.Contains(msg, "paid using cash") {
		t.Error("The cash payment method message was not correct")
	}
	t.Log("LOG: ", msg)
}

func TestCreatePaymentMethodDebitCard(t *testing.T) {
	payment, err := GetPaymentMethod(DebitCard)
	if err != nil {
		t.Fatal("A payment method of type 'DebitCard' must exist")
	}

	msg := payment.Pay(22.30)
	if !strings.Contains(msg, "paid using cash") {
		t.Error("The debit card payment method message was not correct")
	}
	t.Log("LOG: ", msg)
}

func TestGetPaymentMethodNonExistent(t *testing.T) {
	_, err := GetPaymentMethod(20)
	if err == nil {
		t.Error("A payment method with ID 20 must return an error")
	}
	t.Log("LOG: ", err)
}
```

测试结果如下：

``` bash
$go test -v
=== RUN   TestCreatePaymentMethodCash
--- PASS: TestCreatePaymentMethodCash (0.00s)
    factory_test.go:18: LOG:  10.30 paid using cash

=== RUN   TestCreatePaymentMethodDebitCard
--- PASS: TestCreatePaymentMethodDebitCard (0.00s)
    factory_test.go:31: LOG:  22.30 paid using debit card

=== RUN   TestGetPaymentMethodNonExistent
--- PASS: TestGetPaymentMethodNonExistent (0.00s)
    factory_test.go:39: LOG:  Payment method 20 not recognized

PASS
ok      github.com/ricardoliu404/go-design-patterns/creational/factory  0.143s

```

> 源码见https://github.com/ricardoliu404/go-design-patterns/tree/master/creational/factory