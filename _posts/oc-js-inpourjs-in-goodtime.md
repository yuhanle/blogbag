---
layout: post
title:  "ObjC&JavaScript 交互，在恰当的时机注入对象"
date:   2016-11-30 08:00:00
categories: 移动技术
tags:
    - 移动技术
---

警告：文章中提到的

```
- (void)webView:(id)unuse didCreateJavaScriptContext:(JSContext *)ctx forFrame:(id)frame;
```

方法涉及私有API，有网友反馈说审核会被拒绝，希望看到的朋友们慎用

移动端项目开发中，免不了出现 Native App （以下简称Native）和 H5 页面（以下简称H5）的交互，网络上有很多第三方框架，比如[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)，对于一些小的项目需求来说，其实不用那么麻烦，我们还是<!--more-->先从基础着手。

![图文无关](http://7xqhcq.com1.z0.glb.clouddn.com/inpurejs-in-goodtime/16.jpg)

## 先了解几个基础方法

* 网页即将加载（最先执行的代理方法），在每次load 页面的时候都会先走这个回调，可以在此做一些自己的操作，经常会在这儿拦截协议

```
- (BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:(NSURLRequest *)request navigationType:(UIWebViewNavigationType)navigationType { 
    // do something...
    
    return YES;
}
```

* 网页已经加载完成（最后执行的代理方法），执行到这个地方，web 页面已经加载完成，相关代码也都执行完毕

```
- (void)webViewDidFinishLoad:(UIWebView *)webView {
    // 加载完成 隐藏HUD
    
}
```

## 根据不同的场景，找一个最合适的方法

### 场景1 
>（H5 通信 Native，告知Native 要做的事儿）

```
H5 页面在某个标签点击后，要关闭当前加载网页的控制器VC
```

需求分析：
> 这应该不是最简单的一个需求，最简单的是Native 通过url 给H5 页面传参数，告知H5 要做的事儿。
> 
> 这个需求中，H5 页面已经加载完毕，此时可以说H5 页面相关的Bug 和UI 缺陷都与Native 无关，我每次都是这么跟测试人员讲，类似问题直接assign 给他们。
> 

功能实现：
> 对于这类比较简单的需求，最常用的做法就是，通过拦截协议的方法，在点击标签的时候，可以调用自定义协议的超链接，比如定义一个 `yuhanle://action/close `的链接，在页面即将load 的时候，判断url 的协议，如果协议是 `yuhanle`，就拦截掉这个请求，做自己的处理。
> 
> 

图解：

![](http://7xqhcq.com1.z0.glb.clouddn.com/inpurejs-in-goodtime/brokenrule.001.png)


### 场景2
>（H5 调用 Native App 的JS 方法，包括同步和异步操作）

```
H5 页面在加载过程中，需要从Native 中取得部分数据，或调用某个功能，均包含同步
操作或异步操作，比如只是简单的获取token，则直接同步返回，如果需要Native 异
步拿到结果，Native 则需要考虑 JSExport 中的线程问题
```

需求分析：

> 这个需求中肯定需要Native 注入JS 方法，H5 通过调用JS 和Native 通信，其中包括同步和异步两种情况下的处理，需要注意的就是异步操作时，H5 需要在调用 App 时传入一个 JS 方法名，App 在拿到数据后可以回调 H5 的JS 方法，在调用这个回调的时候，需要使用webView 的currentThread，不然就会出现页面卡死。
> 

功能实现：

> 1- 定义一个类，用于注入这个对象

```
// 此模型用于注入JS的模型，这样就可以通过模型来调用方法。
@interface QWSJsObjCModel : NSObject <JavaScriptObjectiveCDelegate>

@property (nonatomic, weak) JSContext *jsContext;
@property (nonatomic, weak) UIWebView *webView;
@property (nonatomic, weak) G100WebViewController * webVc;

@end

```

> 2- 声明协议，实现和JS 对应的方法

```
#import <JavaScriptCore/JavaScriptCore.h>

@protocol JavaScriptObjectiveCDelegate <JSExport>

/**
 *  获取客户端的token
 *
 *  @param qwsKey 客户端生成的密码key
 *
 *  @return 返回值token
 */
- (NSString *)getToken:(NSString *)qwsKey;

/**
 *  H5 传递key 获取newToken 在调用其 callback 方法
 *
 *  @param key      qwskey
 *  @param callback 回调方法名
 *  @param property 方法参数
 */
- (void)getNewToken:(NSString *)key callback:(NSString *)callback property:(NSString *)property;

/**
 *  H5 在加载完成后 告诉客户端在返回的时候调用该方法
 *
 *  @param callback js 方法名
 */
- (void)getExitMsgCallback:(NSString *)callback;

```

> 3- 我们需要在打开webView 的时候，找到一个好的时机注入 JS

```
	// 首先拿到JSContext
	self.jsContext = [_jsWebView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    // 通过模型调用方法，这种方式更好些。
    QWSJsObjCModel *model  = [[QWSJsObjCModel alloc] init];
    self.jsContext[@"nativeObj"] = model;
    model.jsContext = self.jsContext;
    model.webView = _jsWebView;
    
    self.jsContext[@"getUserinfo"] = ^(){
        return @"1234";
    };
    
    self.jsContext.exceptionHandler = ^(JSContext *context, JSValue *exceptionValue) {
        context.exception = exceptionValue;
        NSLog(@"异常信息：%@", exceptionValue);
    };
```

> 4- 对应H5 页面的JS 定义及调用

```
<!DOCTYPE html>
<html>
<head>
	<title>测试IOS与JS之前的互调</title>
	<style type="text/css">
	  * {
	  	font-size: 40px;
	  }
	</style>
  <script type="text/javascript">

  var jsFunc = function() {
    alert('Objective-C call js to show alert');
  }
  
  var jsParamFunc = function(argument) {
    document.getElementById('jsParamFuncSpan').innerHTML
    = argument['name'];
  }
  
  </script>
  
</head>

<body>
  
<div style="margin-top: 100px">
	<h1>Test how to use objective-c call js</h1>
	<input type="button" value="getToken" onclick="alert(nativeObj.getToken())">
	<input type="button" value="Call ObjC system alert" onclick="nativeObj.showAlertMsg('js title', 'js message')">
</div>

<div>
	<input type="button" value="Call ObjC func with JSON " onclick="nativeObj.callWithDict({'name': 'testname', 'age': 10, 'height': 170})">
	<input type="button" value="Call ObjC func with JSON and ObjC call js func to pass args." onclick="nativeObj.jsCallObjcAndObjcCallJsWithDict({'name': 'testname', 'age': 10, 'height': 170})">
</div>
<div>
  <a href="test1.html">Click to next page</a>
</div>

<div>
	<span id="jsParamFuncSpan" style="color: red; font-size: 50px;"></span>
</div>


</body>
</html>
```

按照以上的做法，就能达到Native 和H5 之间的相互通信，现在的问题是，在什么时候注入JS 对象，才能满足H5 页面的需求，因为实际情况中，H5 页面可能会随时调用你的JS。

### 需要注意的几个问题

1- 场景2 中我们提到的，异步调用时的线程问题 首先看下下面的代码

```
- (void)getNewToken:(NSString *)key callback:(NSString *)callback property:(NSString *)property {
    if (_webVc) {
        if ([_webVc.qwsKey isEqualToString:key]) {
            
            __block NSString * newToken = @"";
            __block NSInteger result = 0;
            
            [[UserManager shareManager] autoLoginWithComplete:^(NSInteger statusCode, ApiResponse *response, BOOL requestSuccess) {
                if (requestSuccess) {
                    newToken = [[G100InfoHelper shareInstance] token];
                }else{
                    newToken = @"error";
                }
                
                result = requestSuccess ? response.errCode : statusCode;
                
                JSValue * function = self.jsContext[callback];
                NSArray * params = @[@(result), newToken, property];
                [function callWithArguments:params];
            }];
        }
    }
}

```

这段代码，就是想在H5 页面调用的时候，App 这边自动登陆，重新获取到最新的token，拿到结果以后并回调H5，整个过程上是异步的，看起来是没问题的，但是一旦实际操作起来，会在这里卡死。具体原因，我也不好解释，解决办法是有的，只能通过webView 的currentThread 来执行perform 操作。

示例如下：

```
- (void)getNewToken:(NSString *)key callback:(NSString *)callback property:(NSString *)property {
    if (_webVc) {
        if ([_webVc.qwsKey isEqualToString:key]) {
            
            __block NSString * newToken = @"";
            __block NSInteger result = 0;
            NSThread * webThread = [NSThread currentThread];
            
            [[UserManager shareManager] autoLoginWithComplete:^(NSInteger statusCode, ApiResponse *response, BOOL requestSuccess) {
                if (requestSuccess) {
                    newToken = [[G100InfoHelper shareInstance] token];
                }else{
                    newToken = @"error";
                }
                
                result = requestSuccess ? response.errCode : statusCode;
                // 这里通过此方法 在当前线程操作才不会造成卡死的现象
                [self performSelector:@selector(callQWSJSWithArgument:) onThread:webThread withObject:@[callback, @(result), newToken, property] waitUntilDone:NO];
            }];
        }
    }
}

- (void)callQWSJSWithArgument:(NSArray *)argument {
    NSString * callback = argument[0];
    JSValue * function = self.jsContext[callback];
    
    NSMutableArray * params = [NSMutableArray arrayWithArray:argument];
    // 移除第一个 方法名
    [params removeObjectAtIndex:0];
    [function callWithArguments:params];
}
```

2- 同样是场景2 中的一个问题，什么时候注入对象

需求总是虚无缥缈的，对于H5 结合 Native 的开发结构中，Native 始终扮演着服务和入口的角色，H5 可能随时都会主动和Native 通信，但是Native 应该在什么时候准备好这些服务呢？

看了很多网上的资料，几乎全部都是在页面加载完成 webViewDidFinishLoad 这个回调中注入方法，但实际开发中，很多页面在加载的时候就需要和Native 通信，比如说拿到token，如果在这个时候才注入，肯定是来不及的，只能无功而返。

相信大多数人都没太在意这个问题，当然，如果强制让H5 的开发人员修改逻辑，将所有的通信都放在页面加载完成以后在做，也没问题，只不过对于用户的体验会变得糟糕。

深入研究官方文档，就会发现，webView 在加载过程中，会执行这么一个方法，他的作用是

```
_:didCreateJavaScriptContext:for:

Notifies the delegate that a new JavaScript context has been created created.
```

![](http://7xqhcq.com1.z0.glb.clouddn.com/inpurejs-in-goodtime/webView_didCreateJavaScriptContext.png)

具体参见官方文档说明 [didCreateJavaScriptContext](https://developer.apple.com/reference/webkit/webframeloaddelegate/1501462-webview?language=objc)

看到这里，我们就能在收到这个消息的时候，拿到JSContext，然后注入我们的Model。

首先，新建一个NSObject 的Catagory，在这个代理方法中发送一个通知

```
@implementation NSObject (JSTest)

- (void)webView:(id)unuse didCreateJavaScriptContext:(JSContext *)ctx forFrame:(id)frame {
    [[NSNotificationCenter defaultCenter] postNotificationName:@"DidCreateContextNotification" object:ctx];
}

@end
```

然后，在webView 的控制器中监听这个消息

```
- (void)viewDidLoad {
    [super viewDidLoad];
    // 监听可以注入js 方法的通知
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(didCreateJSContext:) name:@"DidCreateContextNotification" object:nil];
}
```

实现@selector 方法

```
#pragma mark - 可以注入js 的监听
- (void)didCreateJSContext:(NSNotification *)notification {
    NSString *indentifier = [NSString stringWithFormat:@"indentifier%lud", (unsigned long)self.webView.hash];
    NSString *indentifierJS = [NSString stringWithFormat:@"var %@ = '%@'", indentifier, indentifier];
    [self.webView stringByEvaluatingJavaScriptFromString:indentifierJS];
    
    JSContext *context = notification.object;
    
    if (![context[indentifier].toString isEqualToString:indentifier]) return;
    
    self.jsContext = context;
    // 通过模型调用方法，这种方式更好些。
    QWSJsObjCModel *model  = [[QWSJsObjCModel alloc] init];
    self.jsContext[@"nativeObj"] = model;
    model.jsContext = self.jsContext;
    model.webView   = self.webView;
    model.webVc     = self;
    
    self.jsContext.exceptionHandler = ^(JSContext *context, JSValue *exceptionValue) {
        context.exception = exceptionValue;
        DLog(@"异常信息：%@", exceptionValue);
    };
}
```

如此，应该是一次比较完美的注入了~
