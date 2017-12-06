---
layout: post
title:  "iOS-自定义导航栏导致侧滑返回功能失效"
date:   2015-06-22 10:00:00
categories: iOS学习
tags: [iOS学习]
---

iPhone有一个回退按钮在所有的导航条上.这是一个简单的没有文字箭头。

在一开始写项目的时候，就要做好一个准备，导航栏是自定义还是使用系统的，后期有什么改动，有什么比较特殊的需求、当然这些在更改需求的同时，很多东西都已经被改得面目全非了。<!--more-->

完全自定义导航栏，在实际开发中，并不能满足特殊需求，因此更多情况下，还是需要配合系统导航栏自定义，从而达到我们想要的效果。当我们自定义返回按钮之后，就会出现系统的右滑Pop功能就失效了，这是其中的一个小问题，下面就跟大家分享一下我所了解到的：

实现一个自定义按钮是简单的.类似这个设置controller 的navigationItem一个leftBarButtonItem.


```
- (void)viewDidLoad
{
    self.navigationItem.leftBarButtonItem = [self backButton];
}

- (UIBarButtonItem *)backButton
{
    UIImage *image = [UIImage imageNamed:@"back_button"];
    CGRect buttonFrame = CGRectMake(0, 0, image.size.width, image.size.height);

    UIButton *button = [[UIButton alloc] initWithFrame:buttonFrame];
    [button addTarget:self action:@selector(backButtonPressed) forControlEvents:UIControlEventTouchUpInside];
    [button setImage:[UIImage imageNamed:normalImage] forState:UIControlStateNormal];

    UIBarButtonItem *item; = [[UIBarButtonItem alloc] initWithCustomView:button];

    return item;
}
```

但是这样在iOS7上 pop手势交互就不好使了.我发现了一个轻松解决的办法.通过我的beta测试者,我收到了很多关于pop手势的崩溃日志.我发现在栈中推入一个controller后,快速向左平滑,将会引起崩溃.换句话说,如果用户在推入还在进行的时候立即去点击返回.那么导航控制器就秀逗了.我在调试日志里面发现这些:


```
nested pop animation can result in corrupted navigation bar
```

经过几个小时的奋斗和尝试,我发现可以缓解这个错误:设置手势的delegate为这个导航控制器就像Stuart Hall在他的帖子说的那样,分配了一个手势交互行为的委托在自定义按钮显示的时候.然后,当用户快速点击退出的时候,控制器因为手势发送了一个消息在本身已经被销毁的时候.我的解决方案是简单的让NavigationController自己成为响应的接受者.最好用一个UINavigationController的子类.

```
@interface CBNavigationController : UINavigationController <UIGestureRecognizerDelegate>
@end

@implementation CBNavigationController

- (void)viewDidLoad
{
    __weak CBNavigationController *weakSelf = self;

    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
    {
        self.interactivePopGestureRecognizer.delegate = weakSelf;
    }
}

@end
```

在转场/过渡的时候禁用 interactivePopGestureRecognizer当用户在转场的时候触发一个后退手势,则各种事件又凑一块了.导航栈内又成了混乱的.我的解决办法是,转场效果的过程中禁用手势识别,当新的视图控制器加载完成后再启用.再次建议使用UINavigationController的子类操作.

```
@interface CBNavigationController : UINavigationController <UINavigationControllerDelegate, UIGestureRecognizerDelegate>
@end

@implementation CBNavigationController

- (void)viewDidLoad
{
    __weak CBNavigationController *weakSelf = self;

    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
    {
        self.interactivePopGestureRecognizer.delegate = weakSelf;
        self.delegate = weakSelf;
    }
}

// Hijack the push method to disable the gesture

- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated
{
    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
        self.interactivePopGestureRecognizer.enabled = NO;

    [super pushViewController:viewController animated:animated];
}

#pragma mark UINavigationControllerDelegate

- (void)navigationController:(UINavigationController *)navigationController
didShowViewController:(UIViewController *)viewController
animated:(BOOL)animate
{
    // Enable the gesture again once the new controller is shown

    if ([self respondsToSelector:@selector(interactivePopGestureRecognizer)])
        self.interactivePopGestureRecognizer.enabled = YES;
}


@end
```

### 2015-07-21 更新 解决左滑手势冲突和不灵敏的问题

```
-(UIViewController *)popViewControllerAnimated:(BOOL)animated {
    
    return [super popViewControllerAnimated:YES];
}

#pragma mark UINavigationControllerDelegate
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer {
    if ([self.childViewControllers count] == 1) {
        return NO;
    }
    return YES;
}

// 我们差不多能猜到是因为手势冲突导致的，那我们就先让 ViewController 同时接受多个手势吧。
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    return YES;
}
//运行试一试，两个问题同时解决，不过又发现了新问题，手指在滑动的时候，被 pop 的 ViewController 中的 UIScrollView 会跟着一起滚动，这个效果看起来就很怪（知乎日报现在就是这样的效果），而且也不是原始的滑动返回应有的效果，那么就让我们继续用代码来解决吧
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldBeRequiredToFailByGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    return [gestureRecognizer isKindOfClass:UIScreenEdgePanGestureRecognizer.class];
}

-(UIViewController *)popViewControllerAnimated:(BOOL)animated {
    
    return [super popViewControllerAnimated:YES];
}

#pragma mark UINavigationControllerDelegate
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer {
    if ([self.childViewControllers count] == 1) {
        return NO;
    }
    return YES;
}

// 我们差不多能猜到是因为手势冲突导致的，那我们就先让 ViewController 同时接受多个手势吧。
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    return YES;
}
//运行试一试，两个问题同时解决，不过又发现了新问题，手指在滑动的时候，被 pop 的 ViewController 中的 UIScrollView 会跟着一起滚动，这个效果看起来就很怪（知乎日报现在就是这样的效果），而且也不是原始的滑动返回应有的效果，那么就让我们继续用代码来解决吧
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldBeRequiredToFailByGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
    return [gestureRecognizer isKindOfClass:UIScreenEdgePanGestureRecognizer.class];
}
```

在这里还是要感谢万能的Google！

本站文章如无其他特殊说明，均为原创，转载请注明出处。

