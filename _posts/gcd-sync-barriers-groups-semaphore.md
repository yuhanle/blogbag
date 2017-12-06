---
layout: post
title: 'GCD 同步操作之 Resource Competition'
date: '2017-04-28'
categories: 移动技术
tags:
     - iOS
     - GCD
author: '煜寒了'
---

![更快更高更强](http://7xqhcq.com1.z0.glb.clouddn.com/gcd/surfer-2212948_1920.jpg)

在学会简单的使用GCD处理多线程之后，我们来再深入了解下GCD对多线程的一些同步控制。
<!--more-->

## dispatch barriers

在使用 Concurrent Queue 的时候，有时候我们希望队列中的某项任务，能够被串行执行，来避免资源竞争等多线程问题。比如遇到读写问题，这时候我们就需要使用 dispatch barriers。来保证即使在并行队列中，对某个对象的读和写操作，在同一时刻，只有一个可以被执行。这时候就可以用到 dispatch barriers了。下面我们来讨论一下，不同的队列中barriers的使用：

* Custom Serial Queue: 在串行队列中，队列都是顺序串行执行，使用barriers没有任何好处。一般来讲我们不需要这么做。

* Global Concurrent Queue: 这里虽然是并行队列，但这个队列是全局的，我们不能保证别人没有使用这个队列。对这个队列加barriers，可能会影响到其他模块的功能。所以不建议在这个队列中使用。

* Custom Concurrent Queue: 在自定义的并行队列中使用barriers，是比较合适的方式。

所以当我们要做的并行操作，可能存在线程安全问题的时候。我们最好考虑新建自定义并行队列，而不是简单地使用系统提供的 Global Queue。

举一个例子，假设某一个类要管理MyClass这个类型的读写，下面列举这个类的一些相关方法:

```
//初始化自定义并发队列
  - (instancetype)init{
      
      if(self = [super init]){
          customConcurrentQueue = dispatch_queue_create("com.customConcurrentQueue", DISPATCH_QUEUE_CONCURRENT); 
      }
  }

  //写方法
  - (void)write:(MyClass *)myClass {
  
      if( myClass ){
          
          //使用barrier，保证写方法，可以串行执行
            dispatch_barrier_async(self.customConcurrentQueue, ^{ 
              
              //写操作
              ...
          });
      }
  }

  //读方法
  - (MyClass *)read{
      
      //要保证，读和写方法不能同时执行，
      //首先，他们要在同一个队列中 ：self.customConcurrentQueue
      //其次，读方法要等待读出数据后返回，所以应该是同步操作 ：dispatch_sync
      
      __block MyClass *myClass = [[MyClass alloc] init];
      
      dispatch_sync(self.customConcurrentQueue, ^{
      
          //读操作
          myClass = ...
      });
      
      return myClass;
  }
```

## dispatch groups

有时候，我们需要在多个并行任务全部完成后，做一些操作，这时候就需要用到 group来管理了。

举一个简单的例子。我有4个任务要使用并发处理，任务4要等待，任务1、2、3完成后执行。同时，任务4不阻塞当前的线程：

```
- (void)testDispatchGroup{

  dispatch_queue_t concurrentQueue = dispatch_queue_create("com.test.testConcurrent", DISPATCH_QUEUE_CONCURRENT);

  dispatch_group_t group = dispatch_group_create();

  //异步操作
  dispatch_group_async(group, concurrentQueue, ^{
         
      任务1
  });

  dispatch_group_async(group, concurrentQueue, ^{
         
      任务2
  });

  dispatch_group_async(group, concurrentQueue, ^{
         
      任务3
  });

  //dispatch_group_notify 中的block执行的是我们最后要做的任务。同时，这里是异步操作，不会阻塞后面其他代码的执行。
  dispatch_group_notify(group, dispatch_get_main_queue(), ^{
      
      //前面3个任务，都执行完成后，执行里面的block
      任务4
  });

      ...
}
```

再看另一个需求，还是之前的4个任务。唯一的区别是，任务4除了要等待其他任务完成，还要阻塞当前线程：

```
- (void)testDispatchGroup{

  dispatch_group_t group = dispatch_group_create();

  //异步操作
  dispatch_group_async(group, concurrentQueue, ^{
         
      任务1
  });

  dispatch_group_async(group, concurrentQueue, ^{
         
      任务2
  });

  dispatch_group_async(group, concurrentQueue, ^{
         
      任务3
  });
  
  //dispatch_group_wait 等待上面任务全部完成，阻塞当前线程，直到超过设置的时间
  //使用时，要注意避免阻塞主线程等问题
  dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
  
  任务4
  
  ...
}
```

另外，除了使用`dispatch_group_async`管理要做的任务。还可以使用`dispatch_group_enter`、 `dispatch_group_leave` 组合的方式，手动通知任务完成。如果使用手动管理的话，我们要注意：`enter`和`leave`方法，应该是成对出现的。

dispatch_group_enter(customGroup) : 手动告知customGroup，表示一个任务已经开始执行。

dispatch_group_leave(customGroup) : 手动告知`customGroup`，表示一个任务已经完成。当所有`enter`对应的`leave`方法都执行过后。我们的`dispatch_group_notify()`或者`dispatch_group_wait()`，就可以接到任务完成的通知。

## dispatch semaphore 信号量

当有多个消费者，访问有限的资源的时候，[信号量](https://en.wikipedia.org/wiki/Semaphore_(programming)) 可以让我们更好的控制。简单来说，我们通过对信号个数的控制，来达到线程间的同步操作。当信号个数为0的时候，当前线程被阻塞，等待信号量增加，当信号量个数大于0的时候，则线程继续执行。

注意，同步的操作都要小心使用，避免死锁等问题。

另外，根据dispatch_semaphore_wait的返回值，可以用于判断某任务是否超时操作。

```
- (void)testSemaphore{

  //创建 信号量 参数代表初始个数
  dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);


  dispatch_async(concurrentQueue, ^{
  
      sleep(2);
  
      //发送一个信号，信号量个数 +1   
      dispatch_semaphore_signal(semaphore);
  });


  dispatch_async(concurrentQueue, ^{
  
      dispatch_time_t timeoutTime = dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC);
  
      //线程等待，当信号量大于0时 任务继续执行，信号量 -1
      //线程等待，超过预定的超时时间 任务继续执行 信号量不变
      //关于返回值：当返回值 不为0 的时候，说明超时
      if( dispatch_semaphore_wait(semaphore, timeoutTime) ){
          NSLog(@"time out");
      }
  });
}
```