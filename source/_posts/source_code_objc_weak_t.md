---
title: 细看objc-weak源码
tags: 
- 源码
categories: 编程
---
本文不看其他，只专注于weak的内部结构实现细节和源码解读，看了网上很多的文章都是贴上一篇[open source](https://opensource.apple.com/tarballs/objc4/)里面的代码，并没有对实现细节进行解释。所以在这篇文章中，主要分为
weak_entry_t、weak_table_t的源码解析，weak_entry_t和weak_table_t的相互关系，以及对应的操作函数。
>下文的主要是基于两个对象来说的，一个是被引用的对象，一个是弱引用变量（也就是源代码中大量出现的指向指针的指针）。

 我说一下我源码阅读的习惯，先把目光放在头文件中，因为头文件能够给我们一个整体基础结构。弄清楚具体的结构之后，然后再跳到实现文件中去看具体的实现细节。
先交代一下我的编译环境和源代码版本：
> 编译环境：
Apple LLVM version 9.1.0 (clang-902.0.39.1)
Target: x86_64-apple-darwin17.5.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
源代码版本：
 [objc4-723](https://opensource.apple.com/tarballs/objc4/objc4-723.tar.gz) 


## 头文件类关系和结构分析
我先根据头文件画一个基本的UML类图：
![UML类图](https://upload-images.jianshu.io/upload_images/619906-b599004ff3557156.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### DisguisedPtr模板类
先将视线放在weak_entry_t上面，结构weak_entry_t的第一个成员变量是referent，它是一个[DisguisedPtr](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-private.h)类模板实例化之后的变量（点开前面的链接吧，不然我讲不清楚，不然你会骂我的），这个成员其实就是保存被引用的对象。
DisguisedPtr类里面看起来这个类并不复杂，有一个uintptr_t类型的成员变量，由此DisguisedPtr类的对象所占用的内存空间大小也应该为8字节。
public下面主要是构造函数加三大函数中的两个：重载复制运算符，赋值构造函数；由于该类里面并没有涉及到动态new指针变量，所以其析构函数便使用了默认析构函数。除此之外还重载一些其他的操作符。主要看一下私有的两个成员函数：

```
static uintptr_t disguise(T* ptr) {
  return -(uintptr_t)ptr;
}
static T* undisguise(uintptr_t val) {
  return (T*)-val;
}
```
其中``disguise``函数是将指针变量强转为uintptr_t的整形变量，具体怎么伪装呢？就是把该指针指向的内存地址（16进制数据比如：0x7ffeefbff4e8）强制转换为无符号长整型的十进制数据。由于其类型是无符号长整型，因此取负数是数据溢出之后取该类型取值范围内较大的长整型值达到伪装的效果（也就是不好去找到原内存地址）。

```
unsigned long ul_val = 2;
unsigned long*bitl = &ul_val;
cout<<"ul_val address: "<<bitl<<endl;///0x7ffeefbff4e8
///140732920755432 取负数 -> 18446744069408184208
cout<<"disguise: "<<disguise(bitl)<<endl;
cout<<"undisguise: "<<undisguise(*bitl)<<endl;/// 0xfffffffffffffffe 1111...1110
```

其作用在源文件的注释中也说了，我通俗总结是：对那些比如leak这种内存检测工具进行伪装，然后这些检测工具可能就不好去跟踪被引用的对象。

#### weak_entry_t
现在来看一下union的具体内存分布细节，怎么来解释这个问题呢？奉上[objc-weak.h](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-weak.h.auto.html)的源码，打开源码配合文章来看。

```
union {
        struct {/// 为了方便说明问题，我将该结构取名为：struct1
            weak_referrer_t *referrers;
            uintptr_t        out_of_line : 2;
            uintptr_t        num_refs : PTR_MINUS_1;/// num_refs记录的是实际引用数量
            uintptr_t        mask;/// 记录当前referrers数组容器的大小
            uintptr_t        max_hash_displacement;/// 根据hash-key寻找index的最大移动数，这个在后面的append_referrer会讲
        };
        struct {/// 为了方便说明问题，我将该结构取名为：struct2
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
};
```
首先要有一个概念，union里面的多个成员是共享同一且相同大小的内存空间，在strcut1结构成员中算出其总共所占内存大小为64*4，也就是32个字节。其中我的机器是64位机，我的编译器对于指针类型所占内存大小的ABI实现为64位，而无符号长整型占用的内存大小也为64位。多说一句，在C++中结构和类的内存存储区域好像都是在堆上面，由低地址向高地址生长。
基于此来画出inline_referrers和上面第一个结构大致的内存分布样式（关于inline_referrers的元素类型模板类DisguisedPtr所占内存大小在上面讲DisguisedPtr类时提到了）：
![](https://upload-images.jianshu.io/upload_images/619906-5d957df3bbe69aa6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在源码中注释也说了：

```
// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
```
out_of_line_ness是和inline_referrers[1]的低2位是等同的，``out_of_line_ness``和``num_refs``使用了位段，一共占用64位（2位和62位）。由于此时已经是结构内存对齐了，所以下一个结构成员mask的内存地址就刚好换行。
上面还提到的0x0b10，它应该是经过DisguisedPtr伪装之后得到的值，并不是实际的等于0b10，一个只占两位内存空间的，怎么也存储不了16位的数据。__out_of_line_ness == 0b10__是标记使用out-of-line的状态。关于这个0b10我没有想清楚它的由来，有知道的同学麻烦告知于我！！！
继续来看该结构的构造函数：

```
weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
{
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
}
```
在创建weak_entry_t实例的时候，默认是使用inline_referrers的方式来管理对象引用的，并把其余的位上的数据清空。
``out_of_line_ness``用来判断使用out_of_line的方式来进行引用管理，这个out_of_line_ness的值主要是依据于被引用的对象，其引用变量的个数决定的，具体的逻辑在下文会讲到。
再看看struct1的referrers成员，看起来是一个指针变量，更具体的说是存储的引用变量数组的起始地址，而这些引用变量指针指向的地址被DisguisedPtr进行了伪装。

到这里我把weak_entry_t的内存分布讲了一遍（具体的含义在上面代码块中的注释里），然后下面来看一下``weak_table_t``。

#### weak_table_t
``weak_table_t``在头文件中看不出什么特别的内容，但是从源码中可以看出，应该是一个基于C的结构，没有使用C++中结构独有的特性。

```
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;/// 和weak_entry_t的num_refs概念类似
    uintptr_t mask;///和 weak_entry_t的mask概念类似
    uintptr_t max_hash_displacement;/// 和weak_entry_t的max_hash_displacement概念类似
};
```
同样的，其weak_entries成员也应该是一个数组，存储着weak_entry_t变量的指针。针对该结构头文件中公开的操作函数有：

```
id weak_register_no_lock(weak_table_t *weak_table, id referent, 
                         id *referrer, bool crashIfDeallocating);
void weak_unregister_no_lock(weak_table_t *weak_table, id referent, id *referrer);
#if DEBUG
bool weak_is_registered_no_lock(weak_table_t *weak_table, id referent);
#endif
void weak_clear_no_lock(weak_table_t *weak_table, id referent);
```
这看不了什么具体的内容，所以针对头文件的解读就到这里。下面去实现文件中看看具体的实现，看看网上为什么都在说的基于Hash表的一个存储结构。[源码地址](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-weak.mm.auto.html)，老规矩，打开这个网页对照着源码来看。

## objc-weak具体实现细节
首先看两个hash函数：

```
static inline uintptr_t hash_pointer(objc_object *key);
static inline uintptr_t w_hash_pointer(objc_object **key);
```
它们会根据对象的指针（不管是指针还是指向指针的指针）调用一个fast-hash函数来生成一个key，其原理是基于[fast_hash](http://locklessinc.com/articles/fast_hash/)，而这个key的作用目前我们无从得知。
#### grow_refs_and_insert函数
继续看源码，下面主要来看看一个很重要的函数：

```
__attribute__((noinline, used))
static void grow_refs_and_insert(weak_entry_t *entry, 
                                 objc_object **new_referrer)
{
    assert(entry->out_of_line);
    /**
      * #define TABLE_SIZE(entry) (entry->mask ? entry->mask + 1 : 0)
      * entry->mask用来记录referrers的数量
      */
    size_t old_size = TABLE_SIZE(entry);
    size_t new_size = old_size ? old_size * 2 : 8;/// 增长一倍的大小

    size_t num_refs = entry->num_refs;
    weak_referrer_t *old_refs = entry->referrers;
    entry->mask = new_size - 1;
    
    entry->referrers = (weak_referrer_t *)
        _calloc_internal(TABLE_SIZE(entry), sizeof(weak_referrer_t));
    entry->num_refs = 0;
    entry->max_hash_displacement = 0;
    /// 开始处理数据
    for (size_t i = 0; i < old_size && num_refs > 0; i++) {
        if (old_refs[i] != nil) {
            append_referrer(entry, old_refs[i]);/// 把老数据复制进新的entry里面
            num_refs--;
        }
    }
    // Insert
    append_referrer(entry, new_referrer);/// 给entry插入新的数据
    if (old_refs) _free_internal(old_refs);
}
```
由于基于C的数组其实都是定长的，为了能够动态地增加新元素就需要不断地去申请新的内存空间，并且还要是连续的内存地址（要是不连续的地址就去使用链表的方式，但是链表的索引明显弱于数组的）。正是因为新动态申请的连续内存空间，这就需要把老数据复制过来，并把需要新增的数据也追加进去，最后释放掉原内存空间：
![](https://upload-images.jianshu.io/upload_images/619906-e14c9c2269d6e13d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
它其实和C++里面的动态数组的原理一样，为了不频繁地去申请（``calloc``）新的空间和频繁地数据移动。所以每次2倍增长来增加weak_entry_t的长度。为什么说是C++里面动态数组的做法，在《数据结构与算法实现-C++描述》里有提及这些内容。

#### append_referrer和remove_referrer
在grow_refs_and_insert函数中调用了``append_referrer``函数，这个函数很明显是做插入操作的，默认使用inline的方式来增加新增的weak引用，如果使用inline的方式失败了，则是以outline的方式，并申请对应的存储空间，把entry->referrers指向新申请的内存地址，把inline_referrers数组里的数据拷贝到new_referrers中，其源码如下：

```
if (! entry->out_of_line) {
        // Try to insert inline.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }
        // Couldn't insert inline. Allocate out of line.
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            _calloc_internal(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // This constructed table is invalid, but grow_refs_and_insert
        // will fix it and rehash it.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[I];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
}
```
从这里就可以看出，当被引用对象的弱引用referrers个数小于WEAK_INLINE_COUNT时，其entry里面是以inline小数组方式来存储这些弱引用变量的，只有当inline_referrers全部装满之后，该entry out_of_line被设置为REFERRERS_OUT_OF_LINE，后续如若有变量继续引用该对象则是以outline的方式存储的。
> union是在被引用变量的referrers个数小于等于WEAK_INLINE_COUNT时，使用inline数组的内存表现形式；当referrers个数超过了WEAK_INLINE_COUNT则以struct1的内存表现形式！

由于使用inline的方式是使用小数组的方式，但是针对弱引用对象过多，那么它的存取性能就是考虑的一个重点。而散列是一种用于以常数平均时间执行插入、删除和查找的技术。
下面这个过程我不是很确定，如有不同的建议希望指出。

```
size_t index = w_hash_pointer(new_referrer) & (entry->mask);
size_t hash_displacement = 0;
while (entry->referrers[index] != NULL) {
        index = (index+1) & entry->mask;
        hash_displacement++;
}
if (hash_displacement > entry->max_hash_displacement) {
        entry->max_hash_displacement = hash_displacement;
}
weak_referrer_t &ref = entry->referrers[index];
ref = new_referrer;
entry->num_refs++;
```
begin是通过引用new_referrer调用散列函数获取一个散列值，这个散列值就是散列表中的元素查找自己所在散列槽的key。
从源码可以看出，通过散列值查找元素对应散列槽的方式好像是使用了__线性探测法__。简化上面的代码，配合下方这图来看一下把new_referrer指针查找正确index的过程：
![](https://upload-images.jianshu.io/upload_images/619906-3918183348f17cbd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如上图，假设使用``w_hash_pointer``获取到的key为2，obj1成功插入到散列槽2中，obj2使用``w_hash_pointer``获取到的key也为2，此时散列槽2已经放入了obj1，那么只有正向地去寻找下一个散列槽，如果为空则放入obj2。
回到源码中，在求begin值的时候。把hash值和entry->mask做了按位与的操作，但是这里为什么要对entry->mask做一次按位与操作呢？
entry->mask存储着weak_entry_t的referrers数组大小，这样做能保证所得的散列值是小于当前数组的出界的，因为大于referrers数组大小对应的二进制位的高位全部被置为0，从而避免出现数组越界带来的问题。
关于出界和入界的概念，可以在《C陷阱与缺陷》中关于一个介绍for循环越界导致的死循环一节，具体的记得不是很清楚了。针对出界这个概念还是蛮重要的，老板们可以去看一看。
``remove_referrer``和append_referrer在源码上来看基本没有什么区别，区别只不过是一个赋值，一个置空而已。

#### weak_table_t的扩容和减容
针对weak_table_t的扩容和减容源码相对来说比较简单，限于篇幅我没有提供对应的代码，所以在看的时候还麻烦自己打开上面提到的源码地址对照来看。
在源码中主要提供了如下函数：
- weak_entry_insert；
函数``weak_entry_insert ``和上一节提到的append_referrer是类似的，weak_table_t的内部实现同样也是使用散列表的方式来管理所有的entry变量的。只是weak_entry_insert没有去尝试inline的那一步。

- weak_resize；
函数``weak_resize``和上面提到的grow_refs_and_insert函数类似，在调整大小时，都是创建一个新尺寸大小的内存空间，然后将原内存空间的数据移动到新的内存空间。weak_resize只有移动老数据，没有新数据的添加！最后释放掉原内存。

- weak_grow_maybe；
函数``weak_grow_maybe ``是在原weak_table的entry数量大于了weak_table数组容量的3/4时，便调用weak_resize去扩充容量到原数组容量的2倍。

- weak_compact_maybe；
函数``weak_compact_maybe ``是用来收缩容量的，当数组的容量内大部分都为空的话，则减容。

```
if (old_size >= 1024  && old_size / 16 >= weak_table->num_entries) {
        weak_resize(weak_table, old_size / 8);
        // leaves new table no more than 1/2 full
}
```
- weak_entry_remove；
函数``weak_entry_remove ``用来weak_table_t的entries里对应的entry

```
if (entry->out_of_line()) free(entry->referrers);
bzero(entry, sizeof(*entry));
weak_table->num_entries--;
weak_compact_maybe(weak_table);
```
sizeof(*entry)获取到了weak_entry_t所占用的内存大小，使用__bzero__是将该内存段全部置为0，使用bzero而不使用memset影响并不大，使用memset需要多传入一个参数来确定需要重置的值。在《Unix网络编程》里创建sockaddr_in结构变量时，把对应内存空间数据清空用到了bzero，并讲了一下和memset的区别，具体内容可以去看看这本书。

#### 头文件暴露的四个函数
在头文件中暴露了四个外部可用的函数，分别是：__weak_register_no_lock、weak_unregister_no_lock、weak_is_registered_no_lock和weak_clear_no_lock__，根据注释来看主要是针对weak_table_t的添加、删除和清空数据等操作。在这里以下面的代码为基础来讲解：

```
__weak id refer = obj;
```
下面再来具体看看这几个函数在干什么？
``weak_register_no_lock``源代码中提出，注册一个新的键值对，如果新的弱对象不存在则去新创建一个对应的entry。

```
if (!referent  ||  referent->isTaggedPointer()) return referent_id;
```
如果被弱引用指向的对象（obj）是isTaggedPointer，这里便不做相关操作，直接返回弱引用指向的对象（obj）。
关于什么是Tagged Pointer，后面我再去细看一下里面的源码。从这里的源码可以看出，如果是TaggedPointer就不做后续操作，因为指针并没有指向真正的内存地址，返回的值则是被引用对象自身。

```
weak_entry_t *entry;
if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
}else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
}
```
如果存在对应的entry则直接调用append_referrer进行插入。如果不存在，则调用weak_entry_t的构造函数创建一个新的对象，并查看是否需要针对weak_table进行扩容，将新的entry插入到weak_table中。下图是一个为对象增加弱引用，并将引用添加到weak_table中的简易流程：
![](https://upload-images.jianshu.io/upload_images/619906-d185dc35ae44811e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在来看一下``weak_unregister_no_lock``函数，针对weak_table的移除，必须确保entry已经存在于weak_table中，才会去进行后续的操作，同样把对应的流程图画出来：
![](https://upload-images.jianshu.io/upload_images/619906-4892ddf9e99e5acf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最后两个函数是一个是debug状态下用于判断某一entry是否存在于weak_table中，另一个函数则是对特定的被弱引用的对象（obj）的所有引用做清楚操作。

## 结语
到这里objc-weak应该算是讲清楚了（天知道我的表达能力怎么样。。。），最后我从外层结构到内层结构来一一总结下：
1、weak_table可以存储多个entry，而且它会根据其散列表中entry的个数来伸缩其所占内存大小；
2、一个entry表示的是一个被弱引用的对象（上文提到的obj），该变量可以被多个变量弱引用（refer）。所以entry也存在一个散列表，其用来存储对应的弱引用变量的引用。也就是前面源码里面提到的指向指针的指针。
3、entry的out_of_line_ness只有在弱引用变量大于__WEAK_INLINE_COUNT__时才会置为__REFERRERS_OUT_OF_LINE__。也就是只有在这时候union才会使用struct1结构内存布局。
4、还有就是out_of_line_ness == 0b10没有看懂。。。
















