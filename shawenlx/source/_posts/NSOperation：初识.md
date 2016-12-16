---
title: NSOperation：初识
date: 2016-08-09 19:02:00
tags: iOS
categories: iOS
---
### NSOperation的简介

NSOperation的抽象程度高于NSThread，它是苹果对线程的一个面向对象封装。NSOperation表示一个独立的计算单元，作为一个抽象类，你需要实例化他的子类 : NSInvocationOperation / NSBlockOperation 来进行具体操作。实例化之后，调用start方法或者加入到一个NSOperationQueue 操作队列中，就可以开始执行。

------
<!--more-->
### NSOperation的使用

- 启动NSInvocationOperation

```objc
// 如果直接调用operation的start方法，是在主线程上运行，不会开启新的线程
NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(threadLoadImage:) object:imageView];
[operation start];
```

- 使用NSOperationQueue管理NSOperation并开启一个异步线程

```objc
// 定义操作队列属性
@property (strong, nonatomic) NSOperationQueue *queue;
```

```objc
// 实例化操作队列
self.queue = [[NSOperationQueue alloc] init];
```

```objc
// 初始化一个NSInvocationOperation
NSInvocationOperation *operation = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(threadLoadImage:) object:imageView];
```

```objc
// 将NSInvocationOperation添加到队列，一添加到队列，就会开启新线程执行任务，不可以同时使用start
[self.queue addOperation:operation];
```

- 使用NSOperationQueue管理并NSBlockOperation开启一个线程
  - 更偏向于使用NSBlockOperation，原因在于通过使用代码块会更方便一些，本质上和NSInvocationOperation没有区别。

```objc
// 初始化一个NSBlockOperation
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{ // 异步操作 [self operationLoadImage:imageView];}];
// 添加操作到队列中
[self.queue addOperation:operation];
```

- 在主线程中执行操作

```objc
//可通过该＋方法 获取系统主线程，通常我们刷新UI界面进行绘制的时候，必须在主线程下完成。
+ (NSOperationQueue *)mainQueue NS_AVAILABLE(10_6, 4_0);

//该＋号方法可以获取当前使用线程
+ (nullable NSOperationQueue *)currentQueue NS_AVAILABLE(10_6, 4_0);
```

```objc
// 在主线程队列上更新UI
[[NSOperationQueue mainQueue] addOperationWithBlock:^{
   [imageView setImage:image];
}];
```

- 通过 addDependency：添加线程之间的依赖关系
  - Tip: 直接在队列中添加操作会并发执行，执行顺序是系统调用决定的，在特定的时候我们需要控制操作的执行顺序，就会使用到addDependency操作。addDependency: 是NSOperation的成员方法，调用该方法的NSOperation对象将在参数执行完成之后执行。<strong>需要先添加依赖关系，再将操作添加到队列中。</strong>

```objc
  // 初始化三个块操作
    NSBlockOperation *op1 =[NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"下载 %@", [NSThread currentThread]);
    }];
    
    NSBlockOperation *op2 =[NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"美化 %@", [NSThread currentThread]);
    }];
    
    NSBlockOperation *op3 =[NSBlockOperation blockOperationWithBlock:^{
        NSLog(@"更新 %@", [NSThread currentThread]);
    }];
    
    // 通过添加依赖可以控制线程执行顺序，依赖关系可以多重依赖
    // 注意：不要建立循环依赖，会造成死锁
    [op2 addDependency:op1];
    [op3 addDependency:op2];
    
    // 直接加到队列里面会并发执行，谁先先后是系统调用决定
    [self.operationQueue addOperation:op3];
    [self.operationQueue addOperation:op1];
    [self.operationQueue addOperation:op2];
```

- 控制线程并发数

```objc
// 并发的线程越多越耗资源，队列可以设置同时并发线程的数量，来进行控制
self.queue.maxConcurrentOperationCount = 2;
```

- 取消线程操作
  - NSOperation里有一系列的属性用来标明自身状态的，isReady → isExecuting →isFinish。线程start后并不是立即执行，而是进入一个就绪的状态（isReady），由系统调度执行。

```objc
//有时可能需要进行取消操作，可以调用– (void)cancel; 来停止一些还未执行的不必要的线程。
[operation cancel];
```

- 为NSOperation添加完成代码块

```objc
operation.completionBlock = ^{
    NSLog(@"completion!");
};
```

- NSOperation的优先级

```objc
//同NSThread一样，NSOperation可以通过threadPriority属性来指定优先级。
//但是在IOS8，线程这个概念已经被苹果框架系统性的忽略了，threadPriority已由NSQualityOfService属性替代.

typedef NS_ENUM(NSInteger, NSQualityOfService) {

   /* 和图形处理相关的任务，比如滚动和动画 */ 
   NSQualityOfServiceUserInteractive = 0x21, 
   /* 用户请求的任务，但是不需要精确到毫秒级。例如如果用户请求打开电子邮件App来查看邮件 */ 
   NSQualityOfServiceUserInitiated = 0x19, 
   /* 周期性的用户请求任务。比如，电子邮件App可能被设置成每5分钟自动检测新邮件。但是在系统资源极度匮乏的时候，将这个周期性的任务推迟几分钟也没有大碍*/ 
   NSQualityOfServiceUtility = 0x11,
   /* 后台任务，对这些任务用户可能并不会察觉，比如电子邮件App对邮件进行索引以方便搜索 */ 
   NSQualityOfServiceBackground = 0x09,

   /* 默认的优先级 */ 
   NSQualityOfServiceDefault = -1

} NS_ENUM_AVAILABLE(10_10, 8_0);
```

------

### NSOperation优势

- **NSOperation方便控制线程执行顺序**
- **使用NSBlockOperation可以使用块代码，不必单写线程方法，便于传递多个参数**
- **可以控制线程并发数，有效地对线程进行控制**
- **可以添加线程完成代码块，执行需要的操作**

------

### 对比GCD 和 NSOperation

虽然GCD已经成为流行， 但是在某些框架中，例如AFNetworking还是使用的NSOperation来完成线程相关的操作，除了使用框架的NSInvocationOperation / NSBlockOperation 来处理线程操作，你也可以通过自定义集成来完成你需要的操作。

------

### 关于NSOperation博客的传送门

[http://www.cnblogs.com/kenshincui/p/3983982.html#NSOperation](http://www.cnblogs.com/kenshincui/p/3983982.html#NSOperation)
[http://nshipster.cn/ios8/](http://nshipster.cn/ios8/)
[http://nshipster.cn/nsoperation/](http://nshipster.cn/nsoperation/)