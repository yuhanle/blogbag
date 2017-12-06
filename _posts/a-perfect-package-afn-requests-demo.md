---
title: 一个完美封装AFNetworking的网络请求Demo
tags: [iOS学习]
categories: [iOS学习]
date: 2016-01-04 11:29:00
---

一个完美封装AFNetworking的网络请求Demo，其实我们的理想情况是希望API的数据下发之后就能够直接被View所展示。首先要说的是，这种情况非常少。另外，这种做法使得View和API联系紧密，也是我们不希望发生的。

<!--more-->

## 简介
- AFWSApiInvoker 主要负责调用AFN做网络请求以及返回错误信息和请求结果
- ApiInvoker
  所有api请求类的父类，用于填充请求相关信息以及统一接口
- ApiRequest
  网络请求实例 包含请求的所有信息
- ApiResponse
  服务器回应实例
- WSApi
  单例。继成ApiInvoker，负责提供请求相关接口
- WSDevApi
  分类api，继承WSApi，负责提供分类里的请求接口

## 目录
![Screen.png](http://7xqhcq.com1.z0.glb.clouddn.com/yi-ge-wan-mei-feng-zhuang.png)

## 其他探讨
1. 交付什么样的数据给业务层？
我见过非常多的App的网络层在拿到JSON数据之后，会将数据转变成对应的对象原型。注意，我这里指的不是NSDictionary，而是类似Item这样的对象。这种做法是能够提高后续操作代码的可读性的。在比较直觉的思路里面，是需要这部分转化过程的，但这部分转化过程的成本是很大的，主要成本在于：
```
1.数组内容的转化成本较高：数组里面每项都要转化成Item对象，如果Item对象中还有类似数组，就很头疼。
2.转化之后的数据在大部分情况是不能直接被展示的，为了能够被展示，还需要第二次转化。
3.只有在API返回的数据高度标准化时，这些对象原型（Item）的可复用程度才高，否则容易出现类型爆炸，提高维护成本。
4.调试时通过对象原型查看数据内容不如直接通过NSDictionary/NSArray直观。
5.同一API的数据被不同View展示时，难以控制数据转化的代码，它们有可能会散落在任何需要的地方。
```
其实我们的理想情况是希望API的数据下发之后就能够直接被View所展示。首先要说的是，这种情况非常少。另外，这种做法使得View和API联系紧密，也是我们不希望发生的。

有问题，欢迎提issue，或者微博探讨

下载地址：[一个完美封装AFNetworking的网络请求Demo](https://github.com/yuhanle/WSApiInvoker)

本站文章如无其他特殊说明，均为原创，转载请注明出处。