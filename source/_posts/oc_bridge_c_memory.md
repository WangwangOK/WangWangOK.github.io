---
title: OC与C的交互及其内存管理
tags: 
    - 内存管理
categories: C、OC
---
![](http://upload-images.jianshu.io/upload_images/619906-9ef7407a879ddf09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  首先我们从最基本的C中三种链接属性，分别是：外部(external)、内部(internal)、无(none)。我们可以通过关键字``extern、static``来修改变量的链接属性。
  <!-- more --> 
  **extern**关键将一个变量声明为外部的链接属性之后，便可以去访问其他文件中同名该变量。**static**关键字在用于代码块外部的变量时是将其设置为内部链接属性，如果是在代码块内部则将该变量声明为静态变量。
  然后再来看看C中变量的存储类型。存储类型决定了变量的创建、销毁时机。存储变量的位置一共三个地方：**普通内存、运行时堆栈、硬件寄存器**。结合C中的三种链接属性，具体可以分为：
- __栈区__：代码块中的变量在一般情况下为自动变量（由高地址向低地址生长）
- __堆区__：由``malloc、realloc、calloc``等函数动态生成的变量。这些变量我们只能访问其地址，而且当我们不再使用之后需要收到去free掉（由低地址向高地址生长）。
- __全局区／静态区__：代码块之外声明的变量总是存储于静态内存中（默认的链接属性为external）。未初始化的变量放在一起，已经初始化的紧挨地放着。
由于函数实参总是在堆栈中进行传递，所以函数的形参不能设置为static。
- __常量区__：常量字符串
- __代码区__

> 在代码块内部声明的变量的缺省存储类型是自动的，即它存储于堆栈中，称为__自动变量__。
>如果代码块被多次执行，那么自动变量将会重复创建，每一次创建时，它们在内存中的位置可能会不同。

至于上面提到的寄存器中的变量，因为CPU对于寄存器的读取速度非常快，通常编译器会将使用频率很高的变量将其移到寄存器中。如果寄存器变量在多线程编程时出现了问题，我们可能需要显式将该变量声明为``volatile``，让编译器不对该变量进行优化。

```
/**
 全局静态区
 */
int a = 10;  /// external
extern int b;/// external
static int c;/// internal
int d(int e){/// 函数d 默认为external
    int f = 15;/// auto 栈区
    static int g = 20;/// 静态变量 静态区
    return 0;
}

static int h(int i){/// 函数h 修改为static，internal
    register int j;/// 寄存器类型，但是不一定起作用
    int *k = malloc(sizeof(int));/// 堆区
    free(k);
    const int m = 25;/// 常量区
    return 1;
}
```
#### C中各个类型变量的内存管理
C语言中的内存管理与链接属性和所在内存区域都有直接关系。**栈区**的自动变量会在其作用域之后自动进行销毁；**堆区**的中由用户动态的创建的内存，需要手动调用``free``函数来释放（否则会造成内存泄漏）； **全局区／静态区**中的变量由系统创建和销毁，它们在程序开始运行之前就创建好，静态区的变量在程序运行过程中我们不能去修改； **常量区**程序结束后由系统释放。

> 关于堆的一点儿说明：
> 如果我们在使用``malloc``和``free``时是无序的话，最终会产生堆碎片。
>而且被分配的内存是经过对齐的，一般为2的乘方。

堆的末端由一个称为**break**的指针来标识，当堆管理器需要更多内存时，它可以通过系统调用``brk``和``sbrk``来移动break指针。

***

## OC与C的交互(__bridge)
![](http://upload-images.jianshu.io/upload_images/619906-d75d32dc36154952.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当oc在和c相关的函数（CoreFoundataion、Runtime）进行交互时，我们需要将OC的类型传递到C中，也需要将C中的数据返回给OC使用。这其中就需要使用它们类型之间的转换。在``id``类型或者对象变量赋值给``void *``或者逆向赋值时都需要进行特定的转换，单纯的赋值我们可以使用 ``__bridge`` 。

```
NSObject *obj = [[NSObject alloc] init];
void *p_obj = (__bridge void *)(obj);
NSObject * r_obj = (__bridge NSObject *)(p_obj);
```
相对于``__bridge ``，我们可以使用__``__bridge_retained``修饰符，它即可以进行转换，也能持有被转换的对象__（上例中的``obj ``），因此该对象不会被废弃。其语法形式如下：

```
__bridge_retained <#CF type#>)<#expression#>
```
__bridge中还有个``__bridge_transfer ``，它的作用和__bridge_retained相反，被转换的变量（上例中的``p_obj ``）所持有的对象（上例中的``obj ``）会在``r_obj ``被赋值之后释放掉，其语法形式如下：

```
__bridge_transfer <#Objective-C type#>)<#expression#>
```
把上诉例子进行修改：

```
NSObject *obj = [[NSObject alloc] init];
void *p_obj = (__bridge_retained void *)(obj);
NSObject * r_obj = (__bridge_transfer NSObject *)(p_obj);
```
当我们在C语言的结构中，需要使用OC的类型作为结构成员，除了将OC的类型转换为``void *``之外，我们可以使用``__unsafe_unretained``修饰符（这个修饰符会在后面介绍）。

```
/// 在C中使用OC的对象方式
typedef struct rls_temp_ctx{
    NSObject __unsafe_unretained *obj;
    void *target;
} rls_temp_context;
/// 在C中传入OC对象
rls_temp_context tmp_ctxs = {
    .obj = [NSObject new],
    .target = (__bridge void *)(self)
};
```
但是在使用``obj ``时，由于``__unsafe_unretained ``存在悬浮指针的问题，必须要判断该值是否存在。

***
## OC内存管理
前面看了C的内存管理，还看了C和OC的交互，最后就来看看在OC中内存管理应该注意的事项。
  现在我们讨论OC的内存管理是基于ARC的，其中对象变量的创建和释放问题和C的内存管理有点儿相似。大多数情况下系统会帮我们进行内存管理，我们只需要明确自己所声明的对象或者变量存在于什么区域（上面提到的内存区域），给它们添加合适的修饰符等等。
  大部分情况下，对于栈区、堆区、全局静态区的变量对象和C是相同的，我们可以类比来分析OC中对象或者变量的创建和释放时机。ARC中栈区用autoreleasepool管理的变量和C中的自动变量的内存管理时机很相似。
  在OC中使用基于C的函数时，通过``malloc``等函数声明的变量，都需要我们明确地调用``free``函数进行释放！抑或在使用CoreFondation、Runtime时，基本上如果遇到了包含有Copy， Create等关键字函数，在使用完成之后都需要手动释放内存。
2017-09-13更新：
当我们使用Runtime时，运用下面的方法来动态创建一个对象时，被创建的对象不会被释放，但是对应的``release``方法又是MRC时代的。所以我们可以使用如下方法来解决：
```
/// 创建对象
id obj = ((id(*)(id,SEL))objc_msgSend)(((id(*)(id,SEL))objc_msgSend)([self class],@selector(alloc)),@selector(init));
...
...
...
/// 释放对象
((id(*)(id,SEL))objc_msgSend)(obj,NSSelectorFromString(@"release"));
```

#### 内存管理关键字
下面来介绍一下，在Objective-C的ARC中所涉及到的关键字。

- 1、``__strong``为默认值，在声明成员变量和方法参数时也可以使用！

    ```
__strong id obj_var = [[NSObject alloc] init];
```
__作用：默认的行为。__


- 2、``__weak``是不会持有对象实例，__weak修饰符可以避免循环引用

    ```
__weak id obj2 = nil;
{
        __strong id obj_var = [[NSObject alloc] init];/// 自己生成对象并持有
        obj2 = obj_var;/// obj2持有对象的弱引用
        NSLog(@"__weak %@",obj2);/// 此时由于在obj_var变量可用域中，obj2此时有值
}
NSLog(@"__weak %@",obj2);/// 由于不在obj_var作用域之外，obj_var被释放。而且obj2是弱引用于obj_var的，所以此时obj2值为空
```
__作用：避免循环引用，不持有对象实例__

- 3、``__unsafe_unretained``修饰符的变量不属于编译器的内存管理对象。它和__weak类似，不会持有对象实例；

    ```
__unsafe_unretained id obj1 = nil;
{
/// 在obj_var作用域内，__unsafe_unretained和__weak是一样的
        __strong id obj_var = [[NSObject alloc] init];
        obj1 = obj_var;
        NSLog(@"__unsafe_unretained %@",obj1);
}
NSLog(@"__unsafe_unretained %@",obj1);/// 此时变量已经被遗弃，成为悬浮指针
```
> 在使用``__unsafe_unretained``修饰符时，赋值给__strong修饰符的变量时，需要检查被赋值的对象是否存在（也就是被__unsafe_unretained修饰的变量）
 
__作用：在iOS4之前\_\_weak的替代品，但是在将其赋值给其他时，最好做非空判断__

- 4、``__autoreleasing``修饰符的变量替代调用MRC时代的``autorelease``方法，该对象会被注册到autoreleasepool中。以下是__autoreleasing修饰符的使用场景：
  
    1）、在生成对象时，编译器会检查方法名是否是以alloc/new/copy/mutablcopy开始（自己生成自由持有）。如果不是自己生成的则自动将返回值注册到autoreleasepool中。
2）、对象作为返回值时，编译器会自动将其注册到autoreleasing中。
3）、在使用__weak修饰符的变量时就必定要使用注册到autoreleasepool中的对象。
4）、__id的指针或者对象的指针(NSObject **/NSError **)在没有显示指定时会被附加上``__autoreleasing``修饰符__。

    ```
NSError *error = nil;
BOOL result = [self performOperationWithError:&error];
```

最后还是去看看[这套题](http://blog.parse.com/learn/engineering/objective-c-blocks-quiz/)，它的解释对于理解内存的释放很有益处。对于这套题我已经推荐了几次了，哈哈哈。

#### 相关引用
- [ARC](https://developer.apple.com/library/content/releasenotes/ObjectiveC/RN-TransitioningToARC/Introduction/Introduction.html)

