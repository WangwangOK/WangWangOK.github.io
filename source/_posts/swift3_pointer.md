---
title: Swift3 Unsafe[Mutable]Pointer
tags: 
    - Swift
categories: Swift
---
这篇文章水准不高，可能因为我自己能力有限，英文水平也就这样，自己能看懂，可能存在误人子弟的可能性，所以如果有人有机会看到了这边文章就当是一个小白的入门级的笔记吧！如果需要更深入的了解请查看文末的参考链接<br>
<!-- more --> 
为了将我的PhotoCutter适配Swfit3一看到一大堆的Unsafe[Mutable]Pointer的错误就是脑壳疼！最头疼的事没有找到这方面的中文资料，只有自己来弄了，然后记录一下，现在项目还是用OC，一天不写就生疏,怕以后来自己又忘记了，然后自己纪录一下吧...

>这里我先粗略地介绍：
> 1.我的理解:`UnsafeMutablePointer`其实就是`UnsafePointer`的可以变化的类型，但是`UnsafePointer`又不允许你去改变指针元素<br>
> 
> 2.`Unsafe[Mutable]RawPointer`:在swift3以前为`Unsafe[Mutable]Pointer<Void>`,也就是c中的`void *`
> 
> 3.`Unsafe[Mutable]BufferPointer`表示一连串的数组指针

#### withUnsafePointer/withUnsafeMutablePointer ####
比如下面我用c的方式创建和销毁了一个`int`型的指针`a`:
<pre>
int \*a = malloc(sizeof(int));
\*a = 10;
free(a)
</pre>
假设在swift中`var a:Int = 10`,现在我们的目的是想创建一个指针a,我们需要将`a`转成`*a`，我们需要怎么做呢？这里可以用到`withUnsafePointer/withUnsafeMutablePointer`<br>
这两个函数会以swift类型和一个block为参数，而这个目的指针就是这个block的参数。也就是说你想将某一个swift类型的参数转换为一个指针result，这个result就是你想获得的指针，也就是下面两个例子中的ptr，希望我这个描述没有把你绕晕！<br>
这里我也以swift.org上面的socket例子来写吧!<br>
<pre>
var addrin = sockaddr_in()
</pre>

> 创建UnsafeMutablePointer

<pre>
withUnsafeMutablePointer(to: &addrin) { ptr in
	//ptr:UnsafeMutablePointer\<sockaddr_in>
}
</pre>

> 创建UnsafePointer

<pre>
withUnsafePointer(to: &addrin) { ptr in
	//ptr: UnsafePointer\<sockaddr_in>
}
</pre> 

#### withUnsafeBytes/withUnsafeMutableBytes ####
通过`withUnsafeBytes/withUnsafeMutableBytes`获得的`bytes`只能在在函数closure里面进行使用，这个函数只相对于`Data`类型来获取bytes使用！
<pre>
func unsafebytes() {
    guard let data = ".".data(using: .ascii) else{ return }
    data.withUnsafeBytes { (byte:UnsafePointer<CChar>) -> Void in
        print(byte) 
    }
}
unsafebytes()
</pre>

#### withMemoryRebound ####
我们可以使用`withMemoryRebound `函数，来将一个类型的指针转换为另外一个类型的指针，使用这个函数的时候也有一些需要注意点，在[UnsafeRawPointer Migration] (1)的介绍中说到:`Conversion from UnsafePointer<T> to UnsafePointer<U> has been disallowed`，所以只能将`UnsafePointer<Int>`转换为`UnsafeMutablePointer<UInt8>`.
>UnsafePointer<Int> -> UnsafeMutablePointer<UInt8>

<pre>
var a = 10
withUnsafePointer(to: &a) { a_pt in
	a_pt.withMemoryRebound(to: UInt8.self, capacity: 1, { a_pt_uint8 in
   		//a_pt_uint8:UnsafeMutablePointer           
	})
}
</pre>

具体的使用场景:
在使用socket的时候需要`bind `或者`connect `的时候
这个函数的具体使用场景在[UnsafeRawPointer Migration] (1)中也有提到。<br>
`sockaddr_in －> sockaddr`
<pre>
var addrin = sockaddr_in()
let sock = socket(PF_INET, SOCK_STREAM, 0)
let result = withUnsafePointer(to: &addrin) {
	$0.withMemoryRebound(to: sockaddr.self, capacity: 1) {
		connect(sock, $0, socklen_t(MemoryLayout<sockaddr_in>.stride))
	}
}
</pre>


#### assumingMemoryBound ####
将`UnsafeRawPointer`转换为`UnsafePointer<T>`类型，也就是swift3之前的`UnsafePointer<Void>`到`UnsafePointer<T>`。<br>
这个和前面提到的函数`withMemoryRebound `的区别就是:
>assumingMemoryBound可以看成是withMemoryRebound的一个特例，即:<br>
>`assumingMemoryBound`为`UnsafePointer<Void>`到`UnsafePointer<T>`，<br>
>`withMemoryRebound`为`UnsafePointer<U>`到`UnsafeMutablePointer<T>`

代码示例:<br>
<pre>
let strPtr = UnsafeMutablePointer\<CFString>.allocate(capacity: 1)
let rawPtr = UnsafeRawPointer(strPtr)
let intPtr = rawPtr.assumingMemoryBound(to: Int.self)
</pre>


#### bindMemory ####
绑定一个类型\<T>到已经被分配的内存空间，返回一个绑定在`self`内存上`UnsafePointer<T>`的指针，需要注意的是这个函数是用于Unsafe[Mutable]RawPointer。<br>
<pre>
/// - Precondition: The memory is uninitialized.
/// - Postcondition: The memory is bound to 'T' starting at `self` continuing
///   through `self` + `count` * `MemoryLayout<T>.stride`
/// - Warning: Binding memory to a type is potentially undefined if the
///   memory is ever accessed as an unrelated type.
</pre>

操作 | 内存状态 | 类型
---- | --- | ---
`rawptr = allocate()` | uninitialized | None
`tptr = rawptr.bindMemory(T)` | uninitialized | bound to T
`tptr.initialize()` | initialized | bound to T
从上面的表格结合文档里面对于`bindMemory `的说明来看，我对于`bindMemory`的理解就是，使用函数之前这块内存空间是没有被初始化的，使用`bindMemory `的目的是将`T`绑定到`self`后面`self` + `count` * `MemoryLayout<T>.stride`长度的的这块内存空间上来。但是绑定上来并不代表初始化了，此时这个内存空间仍然是没有初始化的，所以最后需要调用函数`initialize`的函数来初始化!<br>
用这个函数同样可以把`void *`的C类型转换为Swift的类型。关于[Custom memory allocation](2)
这个函数的使用可能会有问题...先上一段我自己理解的代码吧
<pre>
let a = 100
let a_rawptr = UnsafeMutableRawPointer.allocate(bytes: MemoryLayout\<Int>.size, alignedTo: MemoryLayout\<Int>.alignment)
let bind_rawptr = a_rawptr.bindMemory(to: Int.self, capacity: MemoryLayout\<Int>.stride)
bind_rawptr.initialize(to: a)
</pre>


#### unsafeBitCast ####

返回一个翻译成某一特定类型的值！,`这个会破坏Swift的类型系统`！<br>

>特别注意️:
>不到万不得已不要使用这个函数


#### 实战 ####
PhotoCutter为了适配Swift3，这其中大部分和指针相关的东西需要适配，我开始看到这些也是懵逼的，根本不懂怎么改，只有自己去慢慢学。我的方法可能很差，就目前而言是适配了，下面贴上我的修改的代码吧!
>Swift2.x

<pre>
options = CFDictionaryCreate(kCFAllocatorDefault,
            UnsafeMutablePointer(UnsafePointer<Void>(keys)),
            UnsafeMutablePointer(UnsafePointer<Void>(values)),
            2,
            &kcKeysBack,
            &kcValuesBack)
</pre>

>Swift3.0

<pre>
fileprivate func buffer<T>(to type:T.Type, source:[T]) -> UnsafeMutablePointer<UnsafeRawPointer?>{
	var buffer = UnsafeMutablePointer<UnsafeRawPointer?>.allocate(capacity: source.count)
	for idx in 0..<source.count {
		let m_ptr = UnsafeMutableRawPointer.allocate(bytes: MemoryLayout<T>.size, alignedTo: MemoryLayout<T>.alignment)
		let bindptr = m_ptr.bindMemory(to: type, capacity: 1)
		bindptr.initialize(to: source[idx])
 		let pty = UnsafeRawPointer(m_ptr)
		buffer.advanced(by: idx).pointee = pty
	}
	return buffer
}
</pre>

调用:
<pre>
let keys:[CFString] = [
            kCGImageSourceCreateThumbnailWithTransform,
            kCGImageSourceCreateThumbnailFromImageIfAbsent,
            kCGImageSourceThumbnailMaxPixelSize]
var keybuffer = buffer(to: CFString.self, source: keys)
/\*
这之间做你的相关操作
*/
keybuffer.deallocate(capacity: keys.count)
</pre>

到这里我对于swift3指针相关的东西，就告一段落了，文章写的很粗糙，望见谅，明天开始公司要求做持续化集成了，如果你看到这个而且想和我交流的话可以在博客上的微博和我取得联系，因为我是真的不懂博客。。。以后有时间会去学习一下前端相关的东西，练手项目应该就是我这个博客！<br>
最后还是希望去看看[C 语言指针 5 分钟教程](3)。<br>
<br>

参考文献:<br>
[Use Swift With Cocoa](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html)<br>
[UnsafeRawPointer Migration Swift.org](https://swift.org/migration-guide/se-0107-migrate.html)<br>
[Swift 3.0 Unsafe World](http://technology.meronapps.com/2016/09/27/swift-3-0-unsafe-world-2/)<br>
[StackOverflow Question](http://stackoverflow.com/questions/39515173/how-to-use-unsafemutablepointer-in-swift-3)<br>
[Binding memory type](https://github.com/apple/swift-evolution/blob/master/proposals/0107-unsaferawpointer.md#memory-model-explanation)<br>
[Swift 中的指针使用](https://onevcat.com/2015/01/swift-pointer/)(这个是之前版本的介绍，就参考一下用unsafeBitCast，同时文章中提到的[C 语言指针 5 分钟教程](3))<br>
[指针](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%A8%99_(%E9%9B%BB%E8%85%A6%E7%A7%91%E5%AD%B8))

[1]: https://swift.org/migration-guide/se-0107-migrate.html
[2]: https://github.com/apple/swift-evolution/blob/master/proposals/0107-unsaferawpointer.md#custom-memory-allocation
[3]: http://blog.jobbole.com/25409/
