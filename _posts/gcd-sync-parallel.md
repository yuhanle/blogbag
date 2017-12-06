---
layout: post
title: 'GCD 同步异步以及串并行详解'
date: '2017-05-02'
categories: 移动技术
tags:
     - iOS
     - GCD
author: '煜寒了'
---

![浪起来](http://7xqhcq.com1.z0.glb.clouddn.com/gcd/wind-surfing-67627_1920.jpg)

GCD是iOS开发多线程中经常使用的技术，先看一下GCD中的常见的术语
<!--more-->

描述多个任务之间同一时刻的运行关系：

* serial（串行） 某一时刻，只执行一个任务
* concurrent（并行） 可以同时执行多个任务

侧重描述一个函数的执行完成，对其他任务的影响 (既 是否任务在等待某个函数完成，然后才可以运行)：

* synchronous（同步） 任务执行完成后reture，（阻塞）
* asynchronous（异步） 不等待任务执行完成，立即reture，（不阻塞当前）

在GCD中，我们用串行并行描述队列。这就是在描述，该队列里面的所有任务，相互之间在同一时刻，是怎样的运行关系。是指队列内本身的任务运行顺序。 

我们还用同步异步，描述某一个任务。比如说任务A是同步执行的。这就是在说，A任务，会阻塞当前任务，直到A结束。这是指不同任务之间的关系，与队列无关，可以是不同队列，也可以是相同队列。

接下来，我们先来看下，GCD里面的不同队列。

## Serial Queues

在串行队列里，同一时间只能执行一个任务。任务按照被添加进入队列的顺序依次执行。每一个任务只有在前面的任务完成后，才可以开始执行。

系统为我们提供的串行队列

* main queue ( dispatch_get_main_queue )

main queue是一个串行队列，有串行队列的一切特性。比较特殊的一点是加入这个队列的任务，都是在主线程执行的。

## Concurrent Queues

加入并行队列的任务，执行的顺序也是按照任务被加入队列的顺序执行，这是我们唯一可以保证的。每个任务都不用等待之前的任务完成，同一时刻可以多个任务同时执行。

系统同样有一个全局的并发队列

* global dispatch queue ( dispatch_get_global_queue )

这是另一个我们熟悉的并发队列，很多时候我们直接使用这个队列，可以简单处理一些我们需要并发执行的任务。

## Custom Queue

除了系统提供的全局队列之外，我们还可以自定义串行或者并行的队列。

```
dispatch_queue_t mySerialQueue = dispatch_queue_create("com.mySerialQueue", DISPATCH_QUEUE_SERIAL);
dispatch_queue_t myConcurrentQueue = dispatch_queue_create("com.myConcurrentQueue", DISPATCH_QUEUE_CONCURRENT);
```

上面是几种我们用GCD时，需要使用到的队列。

另外，使用GCD，除了选择正确的队列外，还要关注：我们要执行的任务是同步还是异步执行。

## dispatch_async 异步执行

dispatch_async 用来用异步的方式执行串行或者并行队列里面的任务，我们来看一下使用 dispatch_async 的几种常见情况：

* custom Serial Queue：当我们需要执行几个应该串行执行的任务，又不阻塞当前的时候。

```
// task1 task2 顺序依次执行，同时不阻塞others

dispatch_queue_t mySerialQueue = dispatch_queue_create("com.mySerialQueue", NULL);

dispatch_async(mySerialQueue, ^{
   ...task1
});

dispatch_async(mySerialQueue, ^{
  ...task2
});

...others
```

* main Queue：当我们执行并完成了一段异步的任务，需要回到主线程更新UI的时候，很常见的选择就是使用GCD的 main queue。

* custom or global concurrent Queue：这个是我们执行非UI任务的常见选择。要注意的是，加入队列的多个任务之间并发执行，我们无法知道那个任务先完成。

```
dispatch_queue_t myConcurrentQueue = dispatch_queue_create("com.myConcurrentQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(myConcurrentQueue, ^{
  
  ...task1    
    dispatch_async(dispatch_get_main_queue(), ^{
          Update UI 
    });
});
```

## dispatch_sync 同步执行

大部分时候我们执行dispatch_sync操作，都要格外小心些。

* custom or main Serial Queue： 同步执行串行队列时，要注意防止发生死锁，比如下面的代码:

```
/串行队列中，task2 等待 task1完成，所以不会开始。而task1又完成不了，因为task2还没有执行完(甚至都没有开始)。死锁。
  
dispatch_queue_t mySerialQueue = dispatch_queue_create("com.mySerialQueue", DISPATCH_QUEUE_SERIAL);
    
dispatch_sync(mySerialQueue, ^{
    
    ...task1
    dispatch_sync(mySerialQueue, ^{
       
       ...task2            
    });
});
```

* concurrent Queue：合理使用可以解决一些并发读写问题。例如

```
//task1 执行结束后，task2才会开始执行。
dispatch_queue_t myConcurrentQueue = dispatch_queue_create("com.myConcurrentQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_sync(myConcurrentQueue, ^{
    
  ...task1 读
});
    
dispatch_async(myConcurrentQueue, ^{
    
  ...task2 写
});
```

## dispatch_after

异步延迟操作。实际上 dispatch_after 就像一个延迟执行的 dispatch_async。

```
double delayInSeconds = 1.0;
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
 dispatch_after(popTime, dispatch_get_main_queue(), ^(void){

});
```
