---
layout: post
title:  "Beego 入门指南（一）如何新建项目"
date:   2017-06-24 17:00:00
categories: Beego 入门
tags: [Beego]
---

# 如何新建项目

你对beego一无所知？没关系，这篇文档会很好的详细介绍beego的各个方面，看这个文档之前首先确认你已经安装了beego，如果你没有安装的话，请看这篇[安装指南](https://github.com/astaxie/beego/blob/master/docs/zh/Install.md)

beego 是一个快速开发 Go 应用的 HTTP 框架，他可以用来快速开发 API、Web 及后端服务等各种应用，是一个 RESTful 的框架，主要设计灵感来源于 tornado、sinatra 和 flask 这三个框架，但是结合了 Go 本身的一些特性（interface、struct 嵌入等）而设计的一个框架。

<!--more-->

> 以下内容需要掌握 golang 的语法基础

## Hello world

首先我们可以看一下入门必学课程，编写一个Hello world 的应用

```
package main

import (
    "github.com/astaxie/beego"
)

type MainController struct {
    beego.Controller
}

func (this *MainController) Get() {
    this.Ctx.WriteString("hello world")
}

func main() {
    beego.Router("/", &MainController{})
    beego.Run()
}
```
把上面的代码保存为hello.go，然后通过命令行进行编译并执行：

```
$ go build hello.go
$ ./hello
```

![QQ20170507-084105@2x.png](http://upload-images.jianshu.io/upload_images/545755-436febc86fad703e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候你可以打开你的浏览器，通过这个地址浏览[http://127.0.0.1:8080](http://127.0.0.1:8080/)返回“hello world”

那么上面的代码到底做了些什么呢？

1、首先我们引入了包github.com/astaxie/beego,我们知道Go语言里面引入包会深度优先的去执行引入包的初始化(变量和init函数，[更多](https://github.com/astaxie/build-web-application-with-golang/blob/master/ebook/02.3.md#maininit))，beego包中会初始化一个BeeAPP的应用，初始化一些参数。

2、定义Controller，这里我们定义了一个struct为MainController，充分利用了Go语言的组合的概念，匿名包含了beego.Controller，这样我们的MainController就拥有了beego.Controller的所有方法。

3、定义RESTFul方法，通过匿名组合之后，其实目前的MainController已经拥有了Get、Post、Delete、Put等方法，这些方法是分别用来对应用户请求的Method函数，如果用户发起的是POST请求，那么就执行Post函数。所以这里我们定义了MainController的Get方法用来重写继承的Get函数，这样当用户GET请求的时候就会执行该函数。

4、定义main函数，所有的Go应用程序和C语言一样都是Main函数作为入口，所以我们这里定义了我们应用的入口。

5、Router注册路由，路由就是告诉beego，当用户来请求的时候，该如何去调用相应的Controller，这里我们注册了请求/的时候，请求到MainController。这里我们需要知道，Router函数的两个参数函数，第一个是路径，第二个是Controller的指针。

6、Run应用，最后一步就是把在1中初始化的BeeApp开启起来，其实就是内部监听了8080端口:Go默认情况会监听你本机所有的IP上面的8080端口

停止服务的话，请按ctrl+c

## 新建项目

首先进入goPath目录，通过如下命令创建beego项目

```
$ cd $GOPATH/src
$ bee new hello

// bee command
// 默认创建带 view 的工程
// bee new [appname] 
// 当然也可以直接快速创建 api 工程
// bee api [appserver]
```

这样就建立了一个项目hello，目录结构如下所示

```
quickstart
|-- conf
|   `-- app.conf
|-- controllers
|   `-- default.go
|-- main.go
|-- models
|-- routers
|   `-- router.go
|-- static
|   |-- css
|   |-- img
|   `-- js
|-- tests
|   `-- default_test.go
`-- views
    `-- index.tpl
```
从目录结构中我们也可以看出来这是一个典型的 MVC 架构的应用，main.go 是入口文件。


## 开发模式

通过bee创建的项目，beego默认情况下是开发模式。
我们可以打开app.conf 文件，通过如下的方式改变我们的模式：

```
beego.RunMode = "pro"
```

或者我们在conf/app.conf下面设置如下：

```
runmode = pro
```

以上两种效果一样。

开发模式中

* 开发模式下，如果你的目录不存在views目录，那么会出现类似下面的错误提示：

```
2017/05/07 9:36:17 [W] [stat views: no such file or directory]
```

* 模板会自动重新加载不缓存。

* 如果服务端出错，那么就会在浏览器端显示如下类似的截图：

![QQ20170507-090203@2x.png](http://upload-images.jianshu.io/upload_images/545755-cce31499ceaeda1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 编译运行

首先进入到hello 的根目录，执行以下命令行

```
yuhanledeMacBook-Pro:goProject yuhanle$ cd src/hello/
yuhanledeMacBook-Pro:hello yuhanle$ bee run
______
| ___ \
| |_/ /  ___   ___
| ___ \ / _ \ / _ \
| |_/ /|  __/|  __/
\____/  \___| \___| v1.8.3
2017/05/07 08:54:08 INFO     ▶ 0001 Using 'hello' as 'appname'
2017/05/07 08:54:08 INFO     ▶ 0002 Initializing watcher...
hello/controllers
hello/routers
hello
2017/05/07 08:54:10 SUCCESS  ▶ 0003 Built Successfully!
2017/05/07 08:54:10 INFO     ▶ 0004 Restarting 'hello'...
2017/05/07 08:54:10 SUCCESS  ▶ 0005 './hello' is running...
2017/05/07 08:54:10 [I] [asm_amd64.s:2197] http server Running on http://:8080
```

这样我们的应用已经在 8080 端口(beego 的默认端口)跑起来了.你是不是觉得很神奇，为什么没有 nginx 和 apache 居然可以自己干这个事情？是的，Go 其实已经做了网络层的东西，beego 只是封装了一下，所以可以做到不需要 nginx 和 apache。

然后就可以在浏览器中打开 [http://127.0.0.1:8080/](http://127.0.0.1:8080/) 预览我们创建的第一个项目。

![QQ20170507-090245@2x.png](http://upload-images.jianshu.io/upload_images/545755-8f52376c9d2c1e93.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下一节：路由设置（预计周末）

更多内容请参考：**[BeegoStudyNotes](https://github.com/yuhanle/BeegoStudyNotes)**

## 参考文章：

[beego 快速入门](https://beego.me/docs/quickstart/)