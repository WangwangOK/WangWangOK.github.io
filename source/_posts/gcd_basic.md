---
title: 从头认识GCD——派发队列基础
tags: 
    - GCD
categories: 实战
---
![](http://upload-images.jianshu.io/upload_images/619906-27e0e0084cc378cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  本文包括了从最基础的获取任务队列开始，配置任务队列，创建任务，提交任务一步一步地来复习GCD中所涉及到的知识。（建议在PC端浏览）
  <!-- more --> 
  包括使用较少的``dispatch_qos_class_t 、dispatch_block_t ``等等知识点。
  GCD任务队列能够让开发者能够更加专注于同步或者异步任务task，而不用把重点放在创建线程和具体同步和加锁等相关操作。但是如果我们想异步做更加灵活的任务的话（比如后台任务之类的），那选择线程肯定是更好的选择。毕竟操作简单带来就是灵活性的确实嘛！首先先来看看派发队列。
  当用户向某一线程提交一个task时，___dispatch_queue_t___作为任务队列以用户期望的方式来管理这些task。 管理的任务的方式有两种类型，分别是串行队列(``DISPATCH_QUEUE_SERIAL``)和并行队列(``DISPATCH_QUEUE_CONCURRENT ``)，它们两个是由宏定义的。 <br>

---
#### 一、获取任务队列
现在问题来了，我们既然知道有这么一个类型了，那我们总要有方式来得到它啊是吧。就目前而言，Apple给我们提供获取该类型变量的方式有三种，分别是：

- ***[dispatch_get_main_queue](https://developer.apple.com/documentation/dispatch/1452921-dispatch_get_main_queue?language=objc)***：程序主线程的任务队列，这是一个串行队列（``DISPATCH_QUEUE_SERIAL``）。在程序``main()``函数被调用之前由系统自动创建。在官方文档中还提到了，我们可以主动去执行被添加到``main_queue``的任务task（也就是说我们可以主动来调用添加到主线程队列的__block__）。三个方法分别是：``dispatch_main()、UIApplicationMain 、CFRunLoopRun()``，选用其中一个。我尝试了一下使用``dispatch_main()``会导致程序中断。

- ***[dispatch_get_global_queue](https://developer.apple.com/documentation/dispatch/1452927-dispatch_get_global_queue?language=objc)***：由系统定义并管理的一个全局并行队列。在获取时，我们需要指定任务队列的系统等级（DISPATCH_QUEUE_PRIORITY_HIGH、DISPATCH_QUEUE_PRIORITY_DEFAULT、DISPATCH_QUEUE_PRIORITY_LOW、DISPATCH_QUEUE_PRIORITY_BACKGROUND）。

    ```
dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
```
但是在iOS8以后，使用枚举``qos_class_t``的值，提供了细粒度更高的全局任务队列，关于QOS在后面统一梳理一下。

    ```
dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
```
>该函数的返回值上使用``dispatch_resume、dispatch_suspend``无效，

- ***dispatch_queue_create***：除了上诉系统提供的两种类型的任务队列之外，我们还可以自己去创建任务队列。我们可以自己创建串行（``DISPATCH_QUEUE_SERIAL``）和并行（``DISPATCH_QUEUE_CONCURRENT``）两种类型的队列，但是它们都有一个变种DISPATCH_QUEUE_SERIAL_INACTIVE, DISPATCH_QUEUE_CONCURRENT_INACTIVE。它们同样会有涉及到QOS的创建方法，后面一起记录一下

    ```
dispatch_queue_create("com.example.gcd", DISPATCH_QUEUE_CONCURRENT);
```

上诉三种就是获得任务队列方法，我们在设置``dispatch_ge_global_queue``的第二个参数时一般设置为***0***。上面这三种方式是我们在日常开发中，使用并发编程时通过GCD的方式来获取任务队列的方法。**在大部分时间使用并行的任务队列时，``global_queue``能够基本满足需求；对于我来说创建线程的场景，主要是当我们需要一个串行的任务，但是又不想在主线程去执行时使用**。既然我在上面提到了QOS，下面我们就系统的来认识一下QOS。

#### 二、通过QOS配置队列
  由于在我们的程序中，存在各种各样的场景，比如用户界面刷新，网络请求，资源下载，缓存存取之类的。为了能够保证程序的高效响应，需要对不同的任务对资源的消耗做出一些调整。
此时我们就可以使用QOS来解决不同任务的资源分配问题，QOS可以用于``dispatch_queue, NSOperation, NSOperationQueue, NSThread ,pthreads``中，这篇文章中主要讲一下在***dispatch_queue***中的使用场景。
> 在官方文档中也说，关于QOS的只能在iOS8以后使用

|QOS_CLASS|执行时机|相关使用场景|
|:---|:---|:---:|
|**USER_INTERACTIVE**|必须是要及时处理|等级最高。主要用户用户交互，比如主线程上的刷新用户界面等等。|
|**USER_INITIATED**|需要很快完成工作|它主要用于比如已经开了一个任务，此时需要立刻执行的场景。意思就是说需要瞬间完成的工作|
|❌ DEFAULT|——|这个我们一般不使用，``dispatch_get_global_queue``就是这一等级。|
|**UTILITY**|可能需要相当长一段时间|不需要及时响应，比如下载操作之类的，但是用户是可以看见进度之类的|
|**BACKGROUND**|长时间类型任务|完全是后台执行，用户不知道进度的|
|❌ UNSPECIFIED|——|开发人员没有指定，系统根据情况进行选定QOS等级|

上诉QOS对应OC参数如下：

|QOS-Class|对应OC|
|:---|:---|
|USER_INTERACTIVE|NSQualityOfServiceUserInteractive|
|USER_INITIATED|NSQualityOfServiceUserInitiated|
|UTILITY|NSQualityOfServiceUtility|
|BACKGROUND|NSQualityOfServiceBackground|

在dispatch_queue中，如果我们想要指定QOS的等级的话，我们可以使用函数[dispatch_queue_attr_make_with_qos_class](https://developer.apple.com/documentation/dispatch/1453028-dispatch_queue_attr_make_with_qo)。在创建任务队列时使用方法如下：

```
dispatch_queue_attr_t attr_qos = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INTERACTIVE, 0);
dispatch_queue_t queue = dispatch_queue_create("com.example.gcd", attr_qos);
```
因为 __QOS对于``dispatch_queue``来说是无法变更的属性__，以致于我们无法去更改已存在任务队列的QOS属性。但是我们可以使用``dispatch_queue_get_qos_class ``函数来获取任务队列的QOS：

```
dispatch_qos_class_t qos_class = dispatch_queue_get_qos_class(the_queue, 0);
/// 一般用于 根据已知队列来获取同qos等级的全局任务队列
dispatch_get_global_queue(dispatch_queue_get_qos_class(the_queue, nil), 0);
/// 或者是 根据已知的全局任务队列来创建与其qos相等的任务队列
dispatch_queue_t the_global = dispatch_get_global_queue(QOS_CLASS_UTILITY, 0);
dispatch_queue_t the_queue = dispatch_queue_create("com.example.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, dispatch_queue_get_qos_class(the_global, 0), 0));
```
当我们要获取全局队列时，在此之前可以使用``DISPATCH_QUEUE_PRIORITY_HIGH、DISPATCH_QUEUE_PRIORITY_DEFAULT、DISPATCH_QUEUE_PRIORITY_LOW、DISPATCH_QUEUE_PRIORITY_BACKGROUND``。现在我们可以使用QOS来获取一个全局的并发任务队列，因此我们有必要来了解一下它们之间的差异和共性：

|以前|现在QOS|
|:---|:---|
|Main Thread|QOS_CLASS_USER_INTERACTIVE|
| DISPATCH_QUEUE_PRIORITY_HIGH |QOS_CLASS_USER_INITIATED|
| DISPATCH_QUEUE_PRIORITY_DEFAULT |QOS_CLASS_DEFAULT|
| DISPATCH_QUEUE_PRIORITY_LOW |QOS_CLASS_UTILITY|
| DISPATCH_QUEUE_PRIORITY_BACKGROUND |QOS_CLASS_BACKGROUND|

具体使用方法如下：

```
dispatch_get_global_queue(QOS_CLASS_USER_INTERACTIVE, 0);
```
 除此之外，我们还可以在``dispatch_block``中对每一个人任务块来设置Qos等级，这里我先将``dispatch_block``提出来，后面我会对其进行较为详细的解释。

```
dispatch_block_t the_block = dispatch_block_create_with_qos_class(0, QOS_CLASS_UTILITY, -8, ^{
        ...
});
dispatch_async(the_queue, the_block);///dispatch_sync, dispatch_after等等需要用到dispatch_block的地方
```

#### 三、创建任务
  前面两点说了任务的执行地点和怎样来创建和配置任务的执行地点，但是我们必须得知道任务是什么？怎么创建任务？在GCD中使用block来作为任务提交给特定的任务队列，例如___dispatch_block_t___或者直接是一个简单的block。
对于``dispatch_block_t ``类型的变量，首先我们得要知道怎么去创建它。首先根据我们的尝试（下面的例子出自[Apple官方](https://developer.apple.com/documentation/dispatch/dispatch_block_t)），对block进行直接赋值：

```
dispatch_block_t error_block;
NSInteger x = 0;
if (x) {
        error_block = ^void(void){
            NSLog(@"TRUE");
        };
}else{
        error_block = ^void(void){
            NSLog(@"FALSE");
        };
}
error_block();/// unsafe
```
  官网中解释到：“ 由于该``dispatch_block_t``变量是在栈内存上声明的，如果执行过该变量作用域之后就有可能导致该变量被释放 ”。到这里我们还是不得不提一下``block``在MRC和ARC下的区别，我们先看一篇[测试](http://blog.parse.com/learn/engineering/objective-c-blocks-quiz/)，在这篇测试中很明显的一个点便是：“ MRC中有NSStackBlock类型，NSMallocBlock类型，NSGlobalBlock类型同时存在。但是在ARC中不再存在NSStackBlock类型，而是直接声明为NSMallocBlock类型” 。也就是说在ARC中就算是在函数方法中声明的block变量也是被声明为NSMallocBlock类型。
  NSMallocBlock类型就不存在上诉官网中提到的变量被提前释放的问题，这一步我并没有去实践，所以上诉结论是否为真，既然官方不建议这么做，那便放弃使用该方法。使用一下两种方式来创建``dispatch_block_t ``变量：
- ***dispatch_block_create***
- ***dispatch_block_create_with_qos_class***

当我们在使用上诉两种方法来创建``dispatch_block_t ``变量时，遇到的第一个便是``dispatch_block_flags_t``参数。它是一个枚举类型：

|枚举类型|作用|
|:---|:---|
|**DISPATCH_BLOCK_ASSIGN_CURRENT**|说明该块会被分配在创建该对象的上下文中（直接执行该块对象时推荐使用）|
|**DISPATCH_BLOCK_BARRIER**|类似于在做同步操作时的``barrier``|
|__DISPATCH_BLOCK_DETACHED__|表明dispatch_block与当前的执行环境属性无关|
|**DISPATCH_BLOCK_ENFORCE_QOS_CLASS**|当dispatch_block提交到队列或者直接提交执行做同步操作时，该值是默认值|
|**DISPATCH_BLOCK_INHERIT_QOS_CLASS**|异步执行的默认值，优先级低于``DISPATCH_BLOCK_ENFORCE_QOS_CLASS ``。可以用该值来覆盖原来QOS类|
|**DISPATCH_BLOCK_NO_QOS_CLASS**|表明dispatch_block不分配QOS类|
来创建``dispatch_block``变量：

```
/// 第一种使用QOS的方式来创建dispatch_block
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"RUN");
});
/// 直接创建dispatch_block
dispatch_block_create(DISPATCH_BLOCK_NO_QOS_CLASS, ^{
        ...
});
```
对于``dispatch_block_create_with_qos_class``方法中relative_priority的参数的规则是：``relative_priority``的值需要在0到``QOS_MIN_RELATIVE_PRIORITY``（-15）之间。
  我们**创建的block会被拷贝到堆上，并由``dispatch_block_t``类型的变量所持有**。创建完成之后，我们可以将其提交到对应的任务队列中（下一节提到的dispatch_async等等函数...），也可以直接去执行（比如：task_block()）。
  既然可以直接去输入一个block块，那为什么我们还需要去使用``dispatch_block_t``？存在即有价值，那么最明显的优势便是：我们可以对该任务块执行**取消操作**！例如：

```
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"RUN");
});
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5 * NSEC_PER_SEC)), dispatch_get_main_queue(), task_block);
dispatch_block_cancel(task_block);
```

但是**如果dispatch_block已经开始执行，便无法取消该任务的执行**。比如下面的例子中，我们对上面的代码进行一点小小的修改：

```
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"RUN");/// 成功执行
        /// 模拟一个长时间的耗时任务
        [NSThread sleepForTimeInterval:3];
        NSLog(@"End");/// 成功执行
});
dispatch_async(dispatch_get_main_queue(), task_block);
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
        /// 保证dispatch_block_t已经开始执行
        dispatch_block_cancel(task_block);
});
```
在这里我们是无法去取消已经执行的块。`` dispatch_block_testcancel ``函数的作用是让我们能够知道当前任务块是否已经被取消。
> 在调用``dispatch_block_cancel``函数时，我们必须要确定即将被cancle的块没有捕获任何其他外部变量，如果持有将会造成内存泄漏。

  除此之外我们来认识一下***dispatch_block_wait*** 函数，它的作用是以同步的方式执行并等待，得等待指定的任务块执行完成之后，抑或者是超时之后然后去执行当前线程的后续任务。如下：

```
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"Start");
        [NSThread sleepForTimeInterval:3];
        NSLog(@"End");
});
dispatch_async(dispatch_get_main_queue(), task_block);
NSLog(@"Before Wait");
dispatch_block_wait(task_block, DISPATCH_TIME_FOREVER);
NSLog(@"After Wait");
```
此时运行并不会得到``Start``。由于``dispatch_block_wait``函数是使用的同步的方式，只要是在该线程的执行流中，它不管你是同步提交还是异步提交（这两种提交方式在下面一节马上会讲）的方式，``dispatch_block_wait``函数如果是在被执行的block之前执行，后续的代码都会被挂起，并不仅仅是``dispatch_block_wait``函数后的代码，也包括block中的代码。因此也就导致了在同一个任务队列中（都处于main_queue中）的``dispatch_block_t ``永远不会执行。解决办法也很简单，第一种我们先让block执行起来；第二种我们让它们处在不同队列中即可：

```
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"Start");
        [NSThread sleepForTimeInterval:3];
        NSLog(@"End");
});
dispatch_queue_t block_queue = dispatch_queue_create("com.example.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_async(block_queue, task_block);
NSLog(@"Before Wait");
dispatch_block_wait(task_block, DISPATCH_TIME_FOREVER);
NSLog(@"After Wait");
```
我们可以利用这个方法来做由异步转同步的需求（后面还会介绍``dispatch_semaphore_t``，它同样可以达到这个效果）。
  最后来看一下函数___dispatch_block_notify___，它的作用是当指定的``dispatch_block_t``变量执行完了之后，通知到给特定的任务队列。在上面的例子中，我们在``block_queue ``中去执行了我们的任务块，但是我们想要在它执行完了以后在``main_queue``中来执行相关的操作，比如我们需要在``main_queue``中更新UI界面之类的：

```
dispatch_block_t task_block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_INHERIT_QOS_CLASS, QOS_CLASS_USER_INITIATED, -8, ^{
        NSLog(@"Start");
        [NSThread sleepForTimeInterval:3];
        NSLog(@"End");
});
dispatch_queue_t block_queue = dispatch_queue_create("com.example.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_async(block_queue, task_block);
dispatch_block_notify(task_block, dispatch_get_main_queue(), ^{
        NSLog(@"Notify");
});
```

#### 四、将任务提交到队列
  在文章的最后，我们来看看怎样把已经创建好的任务提交到特定的任务队列中去！对于提交操作主要涉及到的函数有：**dispatch_async、dispatch_sync、dispatch_block_perform、dispatch_group_async、dispatch_barrier_async、dispatch_barrier_sync**。在这篇文章中先讲前面三个。再后面文章中详解dispatch_group_t、dispatch_barrier时在进行对应的学习。
  ``dispatch_async``使用异步地方式去提交任务块，何为异步？异步方法调用它通过使用一种立即返回的异步的变体并提供额外的方法来支持接受完成通知以及完成等待改进长期运行的(同步)方法（出自[维基百科](https://zh.wikipedia.org/wiki/%E5%BC%82%E6%AD%A5%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)）。``dispatch_sync``使用同步的方式取提交任务块。下图是根据我自己的理解来解释了一下异步和同步的差异性。<br>

![异步和同步的对比](http://upload-images.jianshu.io/upload_images/619906-601bf2002a932ad4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上诉函数分别有对应的版本，分别是``dispatch_async_f、dispatch_sync_f``。它们两个和前面的区别在于，这两个函数不使用block的方式，而是使用C函数指针的方式来执行任务。它们中的__context__是以``void *``类型的变量作为参数，传递给函数指针指向的具体函数。

```
/// 异步使用block
dispatch_async(queue, ^{
        ...
});
/// 同步使用block
dispatch_sync(queue, ^{
        ...
});
/// 异步使用函数指针
dispatch_async_f(dispatch_get_main_queue(),  (__bridge void * _Nullable)(self), task_place);
void task_place(void *data){
    ...
}
/// 同步使用函数指针
dispatch_sync_f(block_queue, (__bridge void * _Nullable)(self), task_place);
void task_place(void *data){
    ...
}
```
最后我们来看看本应属于``dispatch_block_t``中应该讲解的函数``dispatch_block_perform``，但是它作为一个提交任务的函数，放在这里讲我觉得要更为合适一点。它会创建一个``dispatch_block_t``变量，并在该任务队列中以同步的方式来执行block中的内容。

```
dispatch_block_perform(DISPATCH_BLOCK_BARRIER, ^{
        NSLog(@"Start");
        [NSThread sleepForTimeInterval:3];
        NSLog(@"End");
});
```
上面的代码以下代码效果一样（取自[Apple官方文档](https://developer.apple.com/documentation/dispatch/1431048-dispatch_block_perform?language=objc)）:

```
dispatch_block_t b = dispatch_block_create(flags, block);
b();
Block_release(b);
```

但是``dispatch_block_perform ``方法可以以更加高效的方式来进行以上步骤，而不需要在对象分配时将block拷贝到指定堆中。
  到这里把最基础的部分算是走了一遍，可以说是走了最小的一步，但是本文的目的是力求以清晰地路线把每一步所涉及到的知识深挖严查。在后续的文章中会继续介绍GCD中的其他函数和相关的使用方法。


#### 相关链接
以文章中出现顺序：
- [Prioritize Work with Quality of Service Classes](https://developer.apple.com/library/content/documentation/Performance/Conceptual/EnergyGuide-iOS/PrioritizeWorkWithQoS.html#//apple_ref/doc/uid/TP40015243-CH39-SW1)
- [Concurrency Programming Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW17)
- [Objective-C Blocks Quiz](http://blog.parse.com/learn/engineering/objective-c-blocks-quiz/)
- [Transitioning to ARC Release Notes](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)
- [异步调用](https://zh.wikipedia.org/wiki/%E5%BC%82%E6%AD%A5%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8)

