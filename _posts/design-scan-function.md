---
layout: post
title:  "如何设计扫一扫功能才能更优雅"
date:   2016-12-04 08:00:00
categories: 移动技术
tags:
 - 扫一扫
---

当前二维码在生活中随处可见，他的功能无需多言，彻底的改变了消费者的使用习惯，提升了用户的操作体验，同时也拉近了人与人，人与物乃至物与物之间的距离<!--more-->。

<center>
![Thematic](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/Thematic1.jpg)
</center>

## 写在前面

正如题图，当前二维码在生活中随处可见，他的功能无需多言，彻底的改变了消费者的使用习惯，提升了用户的操作体验，同时也拉近了人与人，人与物乃至物与物之间的距离。

二维码的使用场景很多，比如添加好友，关注公众号，移动支付，还有很多的某某活动入口，用户无需记住太多信息，拿出手机，轻轻的扫一扫对方的资料就能完成操作；移动支付时代，也是因为这个功能的便捷，才能发展的更普遍，剁起手来也就一瞬间；在商家举办的很多活动中，只需要提供一个二维码图片，再通过朋友圈等社交媒体的疯狂转发，亿万用户都能与商家零距离参与活动，接着二次分享，从而给商家带来蹭蹭蹭的流量。

然而，以上的种种便捷，都离不开扫一扫这个功能。下面，本人就站在一个刚刚接触此行业的基础上，跟大家聊一聊我脑袋里的扫一扫是如何设计的（如有非议，请艾特我）。

## 正正正文

### 市场上主导应用是如何做的

想必接触这个行业的产品和开发人员都了解，无论什么应用，这块的功能应该都是大差不差，条条大路通罗马，但其中的规则，应该都是统一的。

那我在这里就班门弄斧，顺便体验一下微信的扫一扫功能。

<center>
![该二维码图片由微信APP提供](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/wechat_qrcode_meitu_1.png "该二维码图片由微信APP提供")
</center>

<center>
该二维码图片由微信APP提供
</center>

<b>1. 先看一下微信扫描结果</b>

<center>
![该图片由微信APP提供](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/wechat_result_meitu_2.png "该图片由微信APP提供")
</center>

<center>
该图片由微信APP提供
</center>

这里因为是本人的二维码，所以下面显示的操作是`发消息`，如果是陌生人的二维码，就是`添加到通讯录`。这里可以知道，扫描这个二维码的操作，就是查看该用户的个人资料。那这个二维码里究竟藏有什么信息？我们可以通过其他工具扫描一下结果。

<b>2. 看一下草料扫描结果</b>

<center>
![该图片由草料二维码提供](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/cli_result.png "该图片由草料二维码提供")
</center>

<center>
该图片由草料二维码提供
</center>

很显然，这是一个http 的url 链接

```
http://weixin.qq.com/r/35zcxLnE_uSFrf2n98nN
```

我们先分析一下这url 的大概参数，下面用一个表格来说明：

| 参数 | 类型 | 说明 |
| ------------- 	|:-------------:	| -----:|
| http://weixin.qq.com | String | 域名 |
| r | String | 看起来是路由 表示要查看资料 |
| 35zcxLnE_uSFrf2n98nN | String | 看起来是加密后的能标识用户唯一属性 |

值得一提的是，这个url 是真实可以访问的，通过电脑打开会直接跳转到官方网站。可能会有很多人（包括我们的QA人员）提出疑问，如此简单的一个扫码查看资料的功能，为什么要做成一个复杂逻辑的url 超链接呢？

<b>3. UC 扫码结果</b>

<center>
![该图片由UC提供](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/uc_result.png "该图片由UC提供")
</center>

<center>
该图片由UC提供
</center>

以上是UC 浏览器扫码的结果，同样是跳转到微信官网，但是与之不同的是，紧接着直接打开微信APP（如果安装微信的话），否则就跳到应用市场告知用户下载微信APP。那这里就可以很好的解释一下，为什么二维码的扫描结果使用url 超链接，因为这样可以更好体验的引导用户，从而提升自身产品的流量，也就是说不管你通过何种方式，什么应用扫码，对于用户来讲，你可以很快速很直观的了解到你扫描的东西是什么，不再是二维码实际意义上的一段字符，从产品的角度来说，也能够保证这块的用户流量流失减少，操作体验上也有很大的提升。

### 我们的应用该如何设计

那这个，跟产品本身的需求有很大关系，但是开发的思路和逻辑都是统一的，类似微信这种超级APP，有各种牛人集思广益，项目中应该是会各种组件化，模块化的设计，拿一个简单的扫一扫功能来说，作为一个模块，在他的功能以及和其他模块之间的耦合性来说，肯定是值得我们借鉴和学些的，当然我没有找到任何相关资料，下面只能按照近期的项目来聊聊我是如何处理的。

<b>1. 二维码扫描的结果必定是一个可以访问的超链接</b>

这个应该是没人反驳的。

之所以这么定义，有两点优势。第一，用户体验和导流上，可以做到更好，具体上面的分析中也提到，就不再赘述；第二，在研发上，通过超链接，可以很方便的将功能集成到我们的路由模块，当然这里也有其他的方法，不过也同样是为了处理扫码结果来定义的。

<center>
![路由模块的耦合设计](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/router.001.png "路由模块的耦合设计")

</center>

<center>
路由模块的耦合设计
</center>

上图中，扫一扫功能通过超链接的形式，可以直接整合到路由模块中，<b>通过scheme 的方式，先将域名 替换成 自定义的scheme，如果路由模块可以处理就丢过去处理，不能处理的情况下，就通过APP 内部的浏览器打开该链接，其他逻辑的操作就丢给web 页面处理，比如引导用户到官网，提醒用户下载应用</b>，等一些错误的处理。

<b>2.为了更好地兼容扫码功能，做一些优化 </b>

<center>
![路由模块的扫码功能优化](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/router.002.png "路由模块的扫码功能优化")
</center>

<center>路由模块的扫码功能优化</center>

这里，我们把一些通用的处理操作，通过类别或者代理协议的方法，按功能分别添加到路由模块中，这种优化不仅优化了扫一扫的功能，也同时优化了其他各模块使用路由模块的逻辑。

### 那我们可以开始着手研发了

#### 适用环境

Web或其他应用调用XXX应用特定功能，也可以用于应用内部功能跳转

#### 支持系统

iOS，Android，H5

如有因系统特性导致的不一致，在协议中详细列出

#### 调用方式

##### Web端

http://appwebv2.xxx.cn/<action>/?parms (开发测试环境为appwebv2test.xxx.cn)

##### App端

xxxapp://<action>/?params

Web 调用端需要先判断XXX应用是否已经安装
如已安装，则通过xxxapp://调用
如未安装，视实际定义及功能，通过web调用或者提示用户下载应用


App 端需要先通过xxxapp:// 路由

如果本地已经注册，继续执行

没有注册，直接通过应用内浏览器打开 Web 页面（需要匹配模板填充必要参数）

#### 示例

http://appwebv2.xxx.cn/action/bind/bike/{bikeid}/?<font color=red>appversion={version}&platform={platform}&xxxkey={xxxkey}</font>

链接前半部分，由服务器生成，后面红色部分参数需要客户端在不能正常通过路由打开该功能，调用浏览器时拼接在 url 里。

后面红色部分参数，根据服务器的模板链接匹配填充，然后生成二维码展示。

##### 必要参数

参数 | 类型 | 说明
----|------| ----
action/ | Router | 此路由是action 表示该链接是要实现某功能
bind | String | 表示绑定的操作
bike | String | 表示要绑定的是车
bikeid | String | 需要处理的参数

### 功能已经实现

按照上面的定义和逻辑，扫一扫的功能已经可以据此定义砌砖实现。对于路由的功能，可以本地实现，也可以像蘑菇街那样，通过后台的配置可以随时更改处理操作。

于是，web 端准备了一个页面，超链接是 

```
http://appwebv2.xxx.cn/action/bind/bike
```

生成二维码之前，服务端给了你一个字符串，内容是

```
http://appwebv2.xxx.cn/action/bind/bike/11234/?appversion={version}&platform={platform}&xxxkey={xxxkey}
```

客户端根据字符串，通过匹配和填充，生成一个二维码

<center>
![生成二维码](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/qr_result.png "生成二维码")
</center>

<center>生成二维码</center>

二维码的实际结果就是

```
http://appwebv2.xxx.cn/action/bind/bike/11234/?appversion=V1.0&platform=iOS&xxxkey=3a10362d077cd27b5f2c537e1ff2fc48
```

至此，通过XXX 应用扫码，就会调用添加车辆的功能。
通过第三方APP 扫码的话，会先打开浏览器，web 页面会再通过系统方法先尝试调用XXX 应用，如果不能调起，就提醒用户去商店下载，如果能调起应用，XXX 应用在启动以后，通过路由模块处理，首先跳转到扫一扫界面，然后调用添加车辆的功能，如果路由无法处理这个http 超链接，就会调用浏览器再加载这个web 页面。

到这里，我脑海里能想到的基本分析完了，如果有更好的想法或建议，请艾特我~ 新浪微博：[煜寒了](http://weibo.com/2621837475/profile?topnav=1&wvr=6&is_all=1)

附：
文中插入的微信是本人的，处于部分原因，加了马赛克，不能扫描添加。如有兴趣添加好友的朋友，请关注微信公众号Neanother，回复```我从掘金来```即可收到我的所有联系方式哦~

<center>
![恁说二维码](http://7xqhcq.com1.z0.glb.clouddn.com/Elegant-design-scan-function/nenshuo.jpeg "恁说二维码")
</center>

<center>恁说二维码</center>
