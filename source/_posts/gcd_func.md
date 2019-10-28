---
title: 从头认识GCD——相关函数的使用
tags: 
    - GCD
categories: 实战
---
![](http://upload-images.jianshu.io/upload_images/619906-d284587e19456111.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  在上一篇文章中，我们对GCD有了基本的认知，知道其中一些简单的类型，和一些简单函数。这本篇文章中，我们将继续学习GCD中我们在日常开发中使用较多的函数，及其使用方法。在本篇会介绍**dispatch_after、dispatch_apply、dispatch_group_t、dispatch_semaphore_t和dispatch_barrier**等相关函数。
<!-- more --> 
---
### dispatch_after／dispatch_time_t
我先来说说``dispatch_after``，从某种意义上来说，它属于任务提交的一种方式。在刚刚接触iOS开发的时候，我一直在想“ 对于``dispatch_after ``它是同步提交代码块还是异步提交的代码块的呢？ ”。后来看到Apple的文档中说到"This function waits until the specified time and then asynchronously adds block to the specified queue"，也就是说它的延迟执行，并不是马上就将代码块就提交到指定的队列中，而是__等到指定的时间通过异步的方式将提其提交到指定的队列中去__。因此从这段话中也可以看出它仅仅是``dispatch_async``的一种。该函数的声明如下：

```
void dispatch_after(dispatch_time_t when, dispatch_queue_t queue, dispatch_block_t block);
```
到这里就需要来系统地说一说``dispatch_after``函数的第一个参数，一个``dispatch_time_t``类型的变量。``dispatch_time_t``实际是``uint64_t``类型。系统为该类型定义了两个特殊值，分别是**DISPATCH_TIME_NOW、DISPATCH_TIME_FOREVER**，其中``DISPATCH_TIME_NOW ``表示值为0，而``DISPATCH_TIME_FOREVER ``表示为无穷大（infinity）。除了这两个特殊值之外，我们可以使用函数``dispatch_time()``来创建相对于默认时钟的时间；或者使用``dispatch_walltime()``函数获取绝对时间。
对于``dispatch_time()``函数，第一个参数我们传入DISPATCH_TIME_NOW或者DISPATCH_TIME_FOREVER值。
>``dispatch_time()``函数第二个参数接受的是__ 基于纳秒级别的数值 __。

这时候就需要将具体的数字乘以一个常数，在[官方文档](https://developer.apple.com/documentation/dispatch/dispatch_time_multiplier_constants?language=objc)中列出了相关的常数。

|常数|意义|具体数值|
|:---|:---|:---|
| NSEC_PER_SEC |表示一秒能转换成多少纳秒|``1000000000ull``|
|USEC_PER_SEC|表示一秒能转换成多少微秒|``1000000ull``|
|NSEC_PER_USEC|表示一微秒转换成多少纳秒|``1000ull``|

```
/// 使用相对时间，相对于现在延迟五秒
dispatch_time_t time_t = dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC);
dispatch_after(time_t, dispatch_get_main_queue(), ^{
        NSLog(@"Run");
});
```
如果我们想要该代码块延迟到某一指定时刻去执行，我们只需要去修改``dispatch_after``中的``dispatch_time_t``类型中值，在这里我们使用函数``dispatch_walltime ``来获取绝对的时间戳值。``dispatch_walltime()``函数的一个参数是``struct timespec``类型的一个变量，它是一个结构：

```
_STRUCT_TIMESPEC
{
	__darwin_time_t  tv_sec;
	long  tv_nsec;
};
```
分别为秒和纳秒。**timespec是基于纳秒级别的数值**，关于``dispatch_walltime ``具体是方式之一如下：

```
/// 延迟到某一绝对时刻执行
struct timespec __tp;
double sec, n_sec;
n_sec = modf(1500794750.797543543, &sec);
__tp.tv_sec = sec;
__tp.tv_nsec = n_sec;
dispatch_after(dispatch_walltime(&__tp, 0), dispatch_get_main_queue(), ^{
        ...
});
```
上诉代码要等到时间戳为``1500794750 ``时才会将代码块提交到指定的事件队列中。

### dispatch_apply
***dispatch_apply***是``dispatch_sync``函数配合不同的的``dispatch_queue_t``队列，来循环执行任务。
>如果在``dispatch_apply ``函数中传入的是一个并发队列，那么block中的任务就可以被并发的调用！相对于一般的for循环来说要高效许多。

```
dispatch_queue_t apply_queue = dispatch_queue_create("com.example.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, apply_queue, ^(size_t index) {
        NSLog(@"%zd",index);
});
NSLog(@"End");
```
结果如下``0, 2, 3, 1, 4, End``。但是我们将上面的并发队列改成串行队列之后：

```
dispatch_queue_t apply_queue = dispatch_queue_create("com.example.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, apply_queue, ^(size_t index) {
        NSLog(@"%zd",index);
});
NSLog(@"End");
```
返回的结果``0, 1, 2, 3, 4, End``和正常的for循环没有什么差距。但是不管是在并发的队列还是在串行的队列中，``End``总是最后才打印的。

### dispatch_group_t相关函数
使用dispatch_group可以把许多操作进行合并。在将多个任务block提交之后，我们可以在dispatch_group中获取到这些操作全部完成的时间（不管是串行执行还是并行执行）。
现在我们有一个场景：第一步，我们需要将多个本地资源传递给服务器。我们用``dispatch_group``相关的技术来实现这个需求。创建一个``dispatch_group_t``类型的变量实现非常简单，不像其他GCD函数需要一些其他的参数：

```
dispatch_group_t upload_group = dispatch_group_create();
```
当创建好了dispatch_group之后，我们需要将这些任务进行提交，这里我使用上一节的``dispatch_apply``来将多个任务放在并发的队列中：

```
dispatch_queue_t upload_queue = dispatch_queue_create("com.example.upload.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^(size_t index) {
  dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), upload_queue, ^{
  /// 模拟网络请求
    NSLog(@"Upload %zd",index);
  });
});
```
在大部分的应用中的上传请求，都有一个上传完成的标志。第二步，那么在这个场景中我们如何知道所有图片已经上传成功呢？我们使用同步的方式，用户的交互不起作用，静静地等待上传完成：

```
dispatch_group_t upload_group = dispatch_group_create();
dispatch_queue_t upload_queue = dispatch_queue_create("com.example.upload.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^(size_t index) {
        dispatch_group_enter(upload_group);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), upload_queue, ^{/// 模拟网络请求
            NSLog(@"Upload %zd",index);
            dispatch_group_leave(upload_group);
        });
});
dispatch_group_wait(upload_group, DISPATCH_TIME_FOREVER);
NSLog(@"Upload Complete");
```
``dispatch_group ``的管理是基于计数来做的。``dispatch_group_enter ``会增加该Group内部的任务计数，``dispatch_group_leave ``会减少该Group中未完成的计数，它们两个函数必须配对使用。
``dispatch_group_wait ``函数和我们在上一篇文中讲到的``dispatch_block_wait ``函数功能类似，只不过``dispatch_group_wait ``是针对多个block的**同步方法**，它会等到Group中所有的任务执行完毕之后才会去继续执行后面的内容。
  既然上面提到了``dispatch_group_wait``函数对应``dispatch_block_wait ``函数，那么很明显应该存在``dispatch_block_notify``函数对应的Group函数。我们将上面的函数进行稍加改动，将同步的方式改为异步的方式，让用户能够做其他的操作：

```
dispatch_group_t upload_group = dispatch_group_create();
dispatch_queue_t upload_queue = dispatch_queue_create("com.example.upload.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^(size_t index) {
        dispatch_group_enter(upload_group);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), upload_queue, ^{/// 模拟网络请求
            NSLog(@"Upload %zd",index);
            dispatch_group_leave(upload_group);
        });
});
dispatch_group_notify(upload_group, dispatch_get_main_queue(), ^{
        NSLog(@"Upload Complete");
});
```
  其实相对于使用繁琐的``dispatch_group_enter、dispatch_group_leave``，Apple给我们提供了更为简单的函数``dispatch_group_async``。我这样做的目的是为了在一开始就能让我们清楚，在Group内部是什么在决定着``dispatch_group_wait 、dispatch_group_notify ``的触发时机，我们还是对上面的例子进行稍加修改：

```
dispatch_group_t upload_group = dispatch_group_create();
dispatch_queue_t upload_queue = dispatch_queue_create("com.example.upload.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
dispatch_apply(5, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^(size_t index) {
        dispatch_group_async(upload_group, upload_queue, ^{
            NSLog(@"Upload %zd",index);
        });
});
dispatch_group_notify(upload_group, dispatch_get_main_queue(), ^{
        NSLog(@"Upload Complete");
});
```
很明显对于使用``dispatch_group_async ``给我们带来便利的同时，在灵活性上也就出现缺失，再者就是在用Group做同步的时候使用``dispatch_group_enter、dispatch_group_leave``是更好的选择！

### dispatch_semaphore_t相关函数
在系统中，给予每一个进程一个信号量，代表每个进程目前的状态，未得到控制权的进程会在特定地方被强迫停下来，等待可以继续进行的信号到来（来自[维基百科](https://zh.wikipedia.org/wiki/%E4%BF%A1%E8%99%9F%E6%A8%99)）。通俗一点儿讲就是说在进程内部有一原子递增和递减的计数器（也就是该数据变量**具有原子性**）。
如果触发了某个操作使得信号量小于等于0，那么该操作将会被阻塞，直到其信号量大于0。上面提到过，信号量是基于进程的。所以：
> 信号量不依赖于任何队列，它可以在任何线程中使用。

在GCD中，函数``dispatch_semaphore_signal``增加信号量计数，如果之前信号量计数小于等于0，该函数会唤醒当前正在等待的线程。相反，函数``dispatch_semaphore_wait``会减少信号量计数，如果当该信号量计数小于或者等于0之后，会阻塞当前线程，等待其他操作来增加信号量计数。

```
- (NSArray *)downloadSync{
    NSMutableArray *contents = [NSMutableArray array];
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    
    dispatch_group_t upload_group = dispatch_group_create();
    dispatch_queue_t upload_queue = dispatch_queue_create("com.example.download.gcd", dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_CONCURRENT, QOS_CLASS_USER_INITIATED, 0));
    dispatch_apply(5, dispatch_get_global_queue(QOS_CLASS_UTILITY, 0), ^(size_t index) {
        dispatch_group_enter(upload_group);
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), upload_queue, ^{
            NSString *cts = [NSString stringWithFormat:@"%zd",index];
            NSLog(@"~ %@ ~",cts);
            [contents addObject:cts];
            dispatch_group_leave(upload_group);
        });
    });
    dispatch_group_notify(upload_group, dispatch_get_main_queue(), ^{
        dispatch_semaphore_signal(semaphore);
    });
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    return contents;
}
```
我们现在来看看上面这个方法可以正常的返回吗？除了``dispatch_semaphore_t ``相关的代码，我都是直接从上面拷贝下来，没有做任何修改。当我跑起来之后，始终方法``downloadSync ``不会返回，这里很明显的是造成了死锁的问题！由于``dispatch_semaphore_wait ``函数会阻塞当前线程（它此时是处于主线程中），``dispatch_group_notify ``函数的任务线程即为主线程对应的主任务队列。``dispatch_semaphore_wait ``需要等到函数``dispatch_semaphore_signal ``来增加信号量计数之后才会继续执行主线程，而``dispatch_group_notify ``又要在主线程中执行（由于主线程被阻塞）之后才能去调用``dispatch_semaphore_signal ``函数，因此就造成了死锁，程序永远不会继续执行！。
解决办法也很简单，将``dispatch_semaphore_signal ``放在一个并行的任务队列中进行：

```
dispatch_group_notify(upload_group, dispatch_get_global_queue(QOS_CLASS_USER_INITIATED, 0), ^{
        dispatch_semaphore_signal(semaphore);
});
```
上面使用信号量的相关函数，实现了异步转同步的需求。

### dispatch_barrier
``dispatch_barrier ``的作用是在并发队列中实现同步操作。在并发队列中，任务的提交顺序会影响到执行顺序，当异步提交的任务在``dispatch_barrier ``之后，该任务需要等到``dispatch_barrier ``提交的任务执行完成之后才会开始执行。
把上面的话用下面的图通俗的来解释一下：
![](http://upload-images.jianshu.io/upload_images/619906-c458370900934975.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用下面的伪代码来实现一下上图中的相关任务：

```
dispatch_async(task_queue, task_1);
dispatch_async(task_queue, task_2);
dispatch_async(task_queue, task_3);
dispatch_barrier_async(task_queue, task_4);
dispatch_async(task_queue, task_5);
```
**函数``dispatch_barrier_async ``中block参数，会被目标队列复制并持有，直到任务完成时被释放**。官方文档中提到：
>目标队列必须是用户手动创建的并发队列，如果传入的是串行队列或者是全局并发队列，那么这个函数就和``dispatch_async``类似。

``dispatch_barrier_sync ``在做同步操作时和``dispatch_barrier_async ``效果类似，但是它必须得等到block任务完成之后才会返回！而且``dispatch_barrier_sync ``函数的目标线程不会复制和持有block。

### dispatch_once
在这篇文章的最后以``dispatch_once``来做一个结尾，对于``dispatch_once``我们iOS开发者用的太多了。**该函数在多线程环境下同样也是安全的**，如果是在多线程中进行调用，它会同步地等待block任务执行完成！官方文档中提出：对于``dispatch_once``函数的
>第一个参数必须是存储在全局区或者静态区的变量

```
static dispatch_once_t predicate;
dispatch_once(&predicate, ^{
  ...        
});
```
关于dispatch_once更多的文章见[dispatch_once](https://www.mikeash.com/pyblog/friday-qa-2014-06-06-secrets-of-dispatch_once.html)，以及对应的源码[once.c](https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/once.c)。第三篇文章会在后面放出来，我准备写关于``dispatch_source``和``dispatch_data``以及``dispatch_io``等相关知识。


### 相关链接
根据文中出现顺序
- [Apple Dispatch Github](https://apple.github.io/swift-corelibs-libdispatch/)
- [Dispatch Time Multiplier Constants](https://developer.apple.com/documentation/dispatch/dispatch_time_multiplier_constants?language=objc)
- [Elapsed Time](http://www.gnu.org/software/libc/manual/html_node/Elapsed-Time.html)
- [信号量](https://zh.wikipedia.org/wiki/%E4%BF%A1%E8%99%9F%E6%A8%99)

