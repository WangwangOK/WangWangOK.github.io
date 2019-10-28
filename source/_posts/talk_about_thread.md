---
title: 来唠嗑一下线程中的一些事儿
tags: 
    - 基础知识
categories: 线程
---
![](http://upload-images.jianshu.io/upload_images/619906-b76f26ef4b53a80a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
由于前段时间的工作中，在一个并发编程题中栽了跟头,也因此增加了我对这一方面的理解。下面我会结合例子的方式来阐述一下我的一点儿小理解。
<!-- more --> 

## 线程
  对于并发编程可能首先想到的便是和多线程有关，这又需要涉及到线程的概念。维基百科上关于[线程](https://zh.wikipedia.org/wiki/%E7%BA%BF%E7%A8%8B)的解释，我提取一个我认为比较关键的概念：
>它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

每个线程存在自己的栈和寄存器集合，所以在这种情况下去保证线程相对安全的时候应该要使用``violate``变量（防止编译器缓存了被多个线程寄存器并发用到的变量，让编译器直接去变量地址获取）。

上面的重点在于“线程是包含于进程中，并且一条线程实际上一段代码的执行流”（代码区是被进程内多个线程共享的）。这里也就不过多深入地去解释了，贴一个我翻译的官方的[多线程编程](https://www.gitbook.com/book/wangwangok/threading-programming-guide-will/details)文档，并配合有创建线程不同方式的[代码](https://github.com/wangwangok/Thread_programming)。
我之所以专门把这个提出来是因为我在这个知识基础上遇到过一个面试题——“我们使用异步的方式从服务端获取到了我们需要的数据，然后我们如何去更新对应视图？”。如果不去细想的话直接扔出一段代码：
```
    [self post:url aPara:nil completionBlock:^(id responseObject, NSError *error) {
#if defined(USE_GCD_UPDATE_UI) && USE_GCD_UPDATE_UI == 1
        dispatch_async(dispatch_get_main_queue(), ^{
           /// 更新UI
        });
#else
        /// updateView中更新UI
        [self performSelectorOnMainThread:@selector(updateView:) withObject:responseObject waitUntilDone:NO];
#endif
    }];
```
### 为什么在子线程中更新UI是不安全的
  他会继续追问你，我们为什么必须要把更新UI的任务放在主线程来做？放到子线程不可以吗？对于这个问题我只能用我浅显地认识来解释一下这个问题，因为操作系统相关的东西我几乎忘的差不多了。这里先抛出一个概念——__基于UIKit的控件是线程不安全的__。那么为什么苹果要把UIKit设计为线程不安全？

  最直观的来说，在牺牲性能为代价的前提下，使用同步能够确保代码的正确执行。在大多数情况下使用同步工具都会造成延迟。锁和原子操作通常会涉及到使用内存屏障和内核级的同步来确保代码正确执行。当出现锁竞争的情况下，线程可能会被阻塞从而导致体验上的延迟卡顿。所以我猜测有基于这个原因导致了苹果将UIKit设计成了线程不安全。这就回答了上面提出的“我们为什么必须要把更新UI的任务放在主线程来做？”

下面我通过画图的方式来表达一下在并行状态下UIKit不使用同步工具的情形：
![](http://upload-images.jianshu.io/upload_images/619906-3d6a928e19f97a18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
就如上图所示在主线程和子线程的消息队列中同时去修改同一内存空间中的值，如果不添加一个同步操作的话会发生意想不到的事情。Objective-C中的对象是存放在堆区，而堆区和前面我提过的代码区一样是线程共享的。现在我们知道为什么需要我们在主线程中去更新UI。而对于另一个问题：放到子线程不可以吗？或许在[AsyncDisplayKit](https://github.com/texturegroup/texture)中能够找到子线程处理视图的答案。

### 子线程是如何实现在主线程中更新UI
  现在回到最初的问题上来，子线程和主线程是怎样协调工作来更新视图的呢？更新的代码我在上面已经放出来了，但是面试官的目的是想要知道其底层实现，先来看一下官方文档中出现的一个图片：
![](http://upload-images.jianshu.io/upload_images/619906-687aad325b56e9ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果要完整的讲一下我的理解的话，需要以下这些假设。我们把右边绿色的这一部分当做是子线程，而左边紫色的部分当做是主线程；并且我们有自定义一个输入源（因为我想完整的模拟这个场景，而不是使用系统提供的封闭的输入源），我们要通过该输入源来给主线程的Runloop发送消息（你可能会问为什么要用输入源？我直接发消息不可以吗？不好意思，Runloop就人输入源或者定时器源）。

首先来创建一个自定的输入源，这个输入源负责从子线程给主线程的Runloop发送消息：
```
/// .h
@interface RunloopSource : NSObject
void	runloopsrc_schedule(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
void	runloopsrc_perform(void *info);
void	runloopsrc_cancel(void *info, CFRunLoopRef rl, CFRunLoopMode mode);
@end
```
上诉头文件中的三个函数分别是：  ``schedule``表示注册成功并提供外部给子线程传递数据；``perform``是子线程通过输入源想要给主线程传输的主要出口；``cancel``是在我们异步处理完了之后，调用``CFRunLoopSourceInvalidate``函数告知主线程Runloop该输入源已经完成其职责。
```
///.m
- (instancetype)init{
    self = [super init];
    if (self) {
        CFRunLoopSourceContext ctx = {
            .info = (__bridge void *)(self),
            .retain = NULL,
            .release = NULL,
            .copyDescription = NULL,
            .equal = NULL,
            .schedule = runloopsrc_schedule,
            .perform = runloopsrc_perform,
            .cancel = runloopsrc_cancel
        };
    }
    return self;
}
void runloopsrc_schedule(void *info, CFRunLoopRef rl, CFRunLoopMode mode){
    ...
}

void runloopsrc_perform(void *info){
    RunloopSource *src = (__bridge RunloopSource *)info;
    [src sourceFire];/// 接口用c，但是处理数据，看习惯，习惯用OC
}

void runloopsrc_cancel(void *info, CFRunLoopRef rl, CFRunLoopMode mode){
    ...
}
- (void)sourceFire{
    if (self.sourceFire_handle == nil) {
        return;
    }
    @synchronized (command_data) {
    /// 处理数据
    /// 回传给主线程数据
    /// 在输入源中，也就是这个函数中去给主线程的Runloop发送消息，让其更新界面
    /// - (void)performSelector:(SEL)aSelector onThread:waitUntilDone:modes:
    }
    CFRunLoopSourceInvalidate(runloop_src);
}
```
到这里假设我们已经从``- (void)URLSession: dataTask:didReceiveData:;``（NSURLSessionDataDelegate协议）获取到了数据，此时在runloopsrc_perform 中调用``performSelector ``方法给主线程Runloop发送消息。现在我们把注意力放在上图左边的紫色部分，可以看出它一直处于一个循环中。如果此时消息队列中存在消息，那么该Runloop会处理消息队列中的消息，如果消息队列为空，那么Runloop应该是处于一个休眠状态。当他收到了由我们从子线程的自定义输入源发来的消息时，他会被唤醒来处理该消息。此时在主线程中去执行更新UI的事件。

对于这一块儿我并没有十足的把握，如果有更好的理解麻烦告知与我，万分感谢。

## 我所了解的线程小常识
到这儿了主要就说一点儿我所了解的线程，其中主要包括了线程优先级和调度问题，线程和他寄存器之间的一点儿恩怨！
#### 关于线程优先级和调度
在多对一的线程模型中，一个内核线程对应了多个用户级线程，其实这时候的并发并不是真正意义上的并发，它应该是基于CPU轮转的方式来调度不同用户级线程，让他们每个都执行一小段时间，做到类似并发的效果。所以后面的线程模型都是基于多对多模型，它既可以实现真实的并发，又可以减少一对一模型中线程切换的消耗。

我们可以给线程设置不同的优先级来改变它们的先后执行顺序，除了我们指定的方式，系统会在以下两种情况下去更改线程优先级：
- I/O密集型线程会比CPU密集型线程更容易被系统提高优先级。
因为I/O密集型线程会经常进入waiting状态，而进入waiting状态说明它的任务花费时间短。而CPU密集型线程则是耗费完时间片之后进入ready状态。
- 对于I/O密集型线程来说，如果给它分配了较低优先级。而CPU密集型线程分配了较高线程优先级，那么就会造成I/O密集型线程处于“饿死”状态。所以系统会将长时间没有运行线程的优先级提高。

#### 编译器优化所带来的问题
编译器为了能够让CPU在获取数据更快速，它会把一些需要经常访问的数据读取到寄存器中。[为什么寄存器比内存快？](http://www.ruanyifeng.com/blog/2013/10/register.html)，我的理解是由于寄存器存在于CPU内，内存和CPU之间相隔了一个北桥芯片，而它们是通过PCI总线来连接的，就距离上来说这可能是一个原因（我瞎扯淡）。
因为这个优化会产生一些小问题，看下面一段代码：
```
NSThread thread1 = [NSThread detachNewThreadWithBlock:^{
        NSLock *lock = [[NSLock alloc] init];
        [lock lock];
        i++;
        [lock unlock];
}];
NSThread thread2 = [NSThread detachNewThreadWithBlock:^{
        NSLock *lock = [[NSLock alloc] init];
        [lock lock];
        i++;
        [lock unlock];
}];
```
这是线程安全的吗？我只能说不一定，因为这是一个偶然事件。下面我来说一下我理解的这个偶然事件是怎么发生的？
- 【thread1】读取i的值到线程1的寄存器集合R1（R1 = 0）；
- 【thread1】R1++（由于之后可能还要访问i，所以thread1暂时不会把R1写回到i）；
- 【thread2】读取i的值到线程2的寄存器集合R2（R2 = 0）；
- 【thread2】R2++（R1 = 1）；
- 【thread2】将R2写回i；
- 【thread1】过了很久之后，将R1写回i（i=1）；

很明显这并不是我们想要的结果，这就是由于编译器的优化把值读取到了线程响应的寄存器集合中，改变的根本不是同一块儿内存上的值。所以为了解决这个问题可以使用前面提到的``violate ``变量，以此来告诉编译器不要将该变量读取到寄存器中，而是直接在内存中进行操作。

## 并发与并行
文章的最后我们唠叨一下并发这个词儿，关于并发和并行知乎上有一个[回答](https://www.zhihu.com/question/33515481)解释的很通俗易懂。所以针对Apple的[并发编程指南](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)，指的是有能力同时去执行多个任务，但并不是指一定要同时执行多个任务。GCD就不说了，大部分时间都是在使用它（因为它在接口使用上易用）。重点来说一下NSOperation，它是即强大又难用。

在开始使用NSOperation之前，我们自己需要清楚“我们要干什么？Apple提供的NSOperation是否满足需求？我们自定义NSOperation子类是基于并行还是串行？等等”。这里提到的并行和串行就不解释了，再解释就是一篇科普文了。[NSHipster](http://nshipster.cn/nsoperation/)上提到了关于NSOperation：
>NSOperation表示了一个独立的计算单元。作为一个抽象类，它给了它的子类一个十分有用而且线程安全的方式来__建立状态、优先级、依赖性和取消等的模型__。

所以NSOperation和NSOpertaionQueue不仅仅只是用于网络的情况，当然与之对应的GCD同样可以用于其他事物。试想一下我们是否可以把NSOperation用于一个加载动画。





