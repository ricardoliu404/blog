---
title: beego源码阅读(1)
date: 2020-03-10 22:01:20
cover: https://images.unsplash.com/photo-1558980664-3a031cf67ea8?ixlib=rb-1.2.1&auto=format&fit=crop&w=1350&q=80
tags:
- beego源码
categories:
- 理论
---

# 万恶之源

万恶之源源自beego.me上的demo

``` Go
package main

import "github.com/astaxie/beego"

func main() {
    beego.Run()
}

```

# beego.go

该文件是整个框架的主入口，定义了如下几个东西

- 一些常量，例如版本信息，开发、生产模式的别名
- 一个键为`string`类型的map，简写为`M`
- 定义别名为`hookfunc`的函数类型，入参为空，返回error
- 包含元素为`hookfunc`函数的切片`hooks`

接下来是作为入口的`Run()`。首先运行了`initBeforeHTTPRun()`方法，该方法通过调用`AddAPPStartHook()`方法注册（向`hooks`切片中添加）了一些默认钩子函数：

- registerMime 该钩子函数检查`mime.go`文件中定义的类型和文件后缀之间的关联
- registerDefaultErrorHandler 将10种http错误响应码（`40x`和`50x`）和具体方法对应起来
- registerSession 注册Session组件（如果`BConfig`中确认开启`Session`的话），从`BConfig`中读取Session设置配置到真正的Session设置`session.ManagerConfig`中去。
- registerTemplate 注册默认模板
- registerAdmin 如果开启Admin，则会开启一个新的AdminApp（暂时不知道是干嘛的）
- registerGzip 初始化Gzip压缩

然后根据输入参数解析启动服务的IP和端口，并向相应的`BConfig`中修改参数，最后运行`BeeApp.Run()`。

值得注意的是，同时还有一个`RunWithMiddleWares()`方法存在，该方法同样经过上述过程，不同的是输入参数中有`MiddleWare`类型的参数`mws`，最后又把参数`mws`传递给`BeeApp.Run(mws...)`。