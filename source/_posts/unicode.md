---
title: Unicode和UTF-8、UTF-16以及UTF-32
tags: 
    - Unicode
categories: 编程
---
![](/uploads/unicode/unicode_1.png)
## 写在前面
如果你是iOS开发者，并且在处理NSString字符上遇到了一些问题，强烈建议去看看[Objc中国上关于 NSString 与 Unicode](https://www.objccn.io/issue-9-1/)。上面介绍了一些关于NSString相关的东西，比如``characterAtIndex:``返回的可能是包含组合序列（emoji最为常见）等等。
## 简介
Unicode对世界上大部分的文字系统进行了整理、编码，使得电脑可以用更为简单的方式来呈现和处理文字。这是[维基百科](https://zh.wikipedia.org/wiki/Unicode)对Unicode下的定义。
<!-- more --> 
Unicode的实现方式包含了UTF-8、UTF-16（字符用两个字节或者四个字节表示）和UTF-32（用四个字节来表示），下面对面一一进行介绍。
## UTF-8
UTF-8的最明显的一个特点是__它是变长的，它可以使用1到4个字节表示一个符号，根据不同的符号变化字节长度__。
先把[阮一峰在《字符编码笔记：ASCII，Unicode和UTF-8》](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)中对UTF-8的编写规则的一个总结放出来。
>️️️UTF-8的编码规则很简单，只有二条：
1、对于单字节的符号，字节的第一位设为0，后面7位为这个符号的unicode码。因此对于英语字母，UTF-8编码和ASCII码是相同的。
2、对于n字节的符号（n>1），第一个字节的前n位都设为1，第n+1位设为0，后面字节的前两位一律设为10。剩下的没有提及的二进制位，全部为这个符号的unicode码。

我的解释：第一个字节前面n为设1是为了知道当前字符占用多少字节，而后面字节的前两位为什么要设置为10下面会马上进行解释。
现在我们来具体的分析一下Unicode的不同范围：下面中前两个描述是否为ASCII，后两个描述多字节序列
- U+0000到U+007F（ASCII）
从U+0000到U+007F被编码为0x00~0x7F的单字节，这是[ASCII码](https://zh.wikipedia.org/wiki/ASCII)的所有字符，一共128个字符，所以Unicode是完全用来容纳ASCII的。
对于上面结论中提到的__后面字节的前两位一律设为10__因为必须要大于7F才和ASCII码分开。
- 大于 U+007F（非 ASCII）
所有大于 U+007F 的字符被编码为一串多字节序列，这样就可以区分一串多字节序列是多字节码还是 ASCII 码。
- 0xFE 和 0xFF 不会被用于 UTF-8 编码中。
- 多字节序列的第一个字节在0xC0~0xFD中，剩余字节在0x80~0xBF内。
这里解释一下为什么第一个是在0xC0~0xFD中，理解这里需要再回去看看上面注意中提到的Unicode编码规则。因为表示的是多字节就表明n是大于1的，所以第一个字节最小的值为：``11000000即C0``（每四位表示一个十六进制数，这也是为什么在编程的时候喜欢用十六进制数的原因），如果在没有限制的情况下，通过上面的结论我们可以得到第一个字节能表示的最大的数是0xFE（11111110），就是前面7位设置1最后一位设置为0，但是上面一条中提到__不包含FE__，所以第一个字节的最大值为``0xFD(11111101)``。
同理因为``后面字节的前两位一律设为10``所以多字节除了第一个字节的其他字节最大值为``10111111(BF)``。

>总结
UTF-8 编码字符最长可达六个字节


```
Unicode 字符:			    UTF-8 码:
U-00000000 - U-0000007F:	0xxxxxxx      ///表示ASCII
U-00000080 - U-000007FF:	110xxxxx 10xxxxxx      ///
U-00000800 - U-0000FFFF:	1110xxxx 10xxxxxx 10xxxxxx
U-00010000 - U-001FFFFF:	11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
U-00200000 - U-03FFFFFF:	111110xx 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx
U-04000000 - U-7FFFFFFF:	1111110x 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx 10xxxxxx
```
举个例子：汉字“王”的Unicode码为``U+73B8``转换为二进制为:``0111 0011 1010 1000``，"73b8"位于上面的第三类，把转换的16个二进制依次放入（一定是依次放入不管空格的）上诉的x中：
 ```
11100111 10001110 10101000
/// 同样我们来验证一下阮一峰举例的“严”字
/// 4E25 同样属于上诉的第三类 ，对应的二进制 => 0100 1110 0010 0101
/// 11100100 10111000 10100101 得到的结果和他文章中的一样。
```
到这里我自己觉得应该是把UTF-8的编码方式说清楚了，最后再来一个编码的顺序（很适合于我的方式）
>编码的顺序
对于单字节：
直接将其转换为八位的二进制就可以了；
对于多字节：
1.找到Unicode码对应的二进制数据
2.查看该Unicode码在分类中属于第几类
3.一次填入二进制码

## UTF-16
**UTF-16**是[Unicode](https://zh.wikipedia.org/wiki/Unicode)字符编码五层次模型的第三层：字符编码表（Character Encoding Form，也称为"storage format"）的一种实现方式。即把Unicode字符集的抽象[码位](https://zh.wikipedia.org/wiki/%E7%A0%81%E4%BD%8D)映射为__16位__长的整数（即[码元](https://zh.wikipedia.org/wiki/%E7%A0%81%E5%85%83)）的序列，用于数据存储或传递。
这里需要说明一下[基本多文种平面-BMP](https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84#.E5.9F.BA.E6.9C.AC.E5.A4.9A.E6.96.87.E7.A7.8D.E5.B9.B3.E9.9D.A2)和[辅助平面-SMP](https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84#.E7.AC.AC.E4.B8.80.E8.BC.94.E5.8A.A9.E5.B9.B3.E9.9D.A2)，在维基百科中每一个平面相关的图片下面都说了"每个写着数字的格子代表256个码点"，即00~FF。例如：位于BMP中``00``格子中的一个码点表示为:`0x00E5`。
下面同样用一下阮一峰的规则总结：
>基本平面的字符占用2个字节，辅助平面的字符占用4个字节。也就是说，UTF-16的编码长度要么是2个字节（U+0000到U+FFFF），要么是4个字节（U+010000到U+10FFFF）。

为了能够区分它本身是一个字符，还是需要跟其他两个字节放在一起解读。在BMP中，从**U+D800**到**U+DFFF**之间BMP的区段是永久保留不映射到字符（从维基百科的图中``D8~DF之间表示unallocated code points``）。
#### UTF-16结论(D800~DFFF)
具体来说，辅助平面的字符位共有``pow(2,20)``个，也就是说，对应这些字符至少需要20个二进制位。UTF-16将这20位拆成两半，前10位映射在U+D800到U+DBFF（空间大小``pow(2,10)``），称为高位（H），后10位映射在U+DC00到U+DFFF（空间大小``pow(2,10)``），称为低位（L）。这意味着，一个辅助平面的字符，被拆成两个基本平面的字符表示。
```
HHHH HHHH HHLL LLLL LLLL
```
>️注意-结论
高位：D800~DBFF;
低位：DC00~DFFF;

>所以，当我们遇到两个字节，发现它的码点在U+D800到U+DBFF之间，就可以断定，紧跟在后面的两个字节的码点，应该在U+DC00到U+DFFF之间，这四个字节必须放在一起解读。（这里我就直接引用阮一峰的解释，因为他解释很通俗易懂。）

解释一下这里为什么是``pow(2,20)``，在基本平面之外有16个辅助平面（即``pow(2,4)``），而每一个辅助平面``pow(2,16)``个码位（辅助平面和基本平面一样，每个码位里面都包含了256个码点）。
```
///辅助平面字符，转码公式。
H = Math.floor((c-0x10000) / 0x400)+0xD800
L = (c - 0x10000) % 0x400 + 0xDC00
```
H为上文提到的高位，L位上文提到的低位。
举例说明一下:
```
/// 对于小于0xFFFF的即基本平面的字符，为两个字节
U+8D9E = 0x8D9E  ///对应的二进制格式为:10001101 10011110

/// 对出于辅助平面的字符
/// 对于U+1D306
H = Math.floor((0x1D306-0x10000) / 0x400)+0xD800 = d834
L = (0x1D306 - 0x10000) % 0x400 + 0xDC00 = df06
```

## UTF-32
因为UTF-32对每个字符都使用4[字节](https://zh.wikipedia.org/wiki/%E5%AD%97%E8%8A%82)，就空间而言，是非常没有效率的。特别地，非[基本多文种平面](https://zh.wikipedia.org/wiki/%E5%9F%BA%E6%9C%AC%E5%A4%9A%E6%96%87%E7%A8%AE%E5%B9%B3%E9%9D%A2)的字符在大部分文件中通常很罕见，以致于它们通常被认为不存在占用空间大小的讨论，使得UTF-32通常会是其它编码的二到四倍。

## 结论
所以当我们在使用字符串的时候，通常使用length的时候，要看他的编码方式，并不是一个字符就代表了一个字节有可能是两个字节、四个字节甚至可能最多能到6个字节都是有可能的，这应该就能理解Swift中对于字符串的处理了。


## 参考文献
[Unicode代码图表](http://www.unicode.org/charts/)
[字符编码笔记：ASCII，Unicode和UTF-8](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
[什么是UTF-8](https://blog.igorw.org/2012/08/29/translate-what-is-utf-8/)
[Unicode与JavaScript详解](http://www.ruanyifeng.com/blog/2014/12/unicode.html)
[[Unicode编码及其实现：UTF-16、UTF-8，and more](http://blog.csdn.net/thl789/article/details/7506133)](http://blog.csdn.net/thl789/article/details/7506133)
[UTF-16](https://zh.wikipedia.org/wiki/UTF-16)
[UTF-32](https://zh.wikipedia.org/wiki/UTF-32)
