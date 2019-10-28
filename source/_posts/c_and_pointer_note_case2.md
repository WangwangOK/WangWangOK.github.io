---
title: 《C和指针Note》之操作符和表达式
tags: 
    - C
categories: C语言
---
这篇主要记录操作符和表达式相关的只是！
<!-- more --> 
当前C语言环境:
```
Apple LLVM version 8.0.0 (clang-800.0.38)
Target: x86_64-apple-darwin15.6.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
```

### 移位操作符
左移操作符`<<`,右移操作符`>>`;而且位移的操作数必须是***整数类型***。其中右移操作符存在两种情况：算术移位和逻辑移位；这里主要来说一说算术移位。
###### 逻辑移位
左边移入的位用0填充
###### 算术移位
算术移位也就是说会考虑到符号位，即左边移入的位由原值的符号位决定，如果符号位为1则移入的位为1，符号位为0则移入的位为0；
对于移位操作，还必须知道数字在计算机中是以二进制的形式存在的，而且负数的表示形式还稍有不同。
>负数在计算机中的二进制表示形式

>假设在当前计算机中，int型占8位。比如要知道数字`-10`在计算机的表现形式，第一步：数字`10`的二进制值为00001010，第二步用100000000来减去00001010。得到的就是`-10`在二进制中的表现形式为:11110110。更多关于负数补码的介绍[阮一峰的网络日志－－关于2的补码](http://www.ruanyifeng.com/blog/2009/08/twos_complement.html)

这里我举例说明一下移位操作:
<pre><code>
int uint = 10; //00001010
int sint = -10;//11110110
printf("%d\n",uint >> 2);//00000010
printf("%d\n",sint >> 2);//11111101
</code></pre>在当前环境下：
<pre><code>
Apple LLVM version 8.0.0 (clang-800.0.38)
Target: x86_64-apple-darwin15.6.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
</code></pre>打印的结果分别为`2`和`-3`,所以从这里看出来编译器对于左移操作是***算术移位***。

### ++和--操作符
对于这两个操作符的计算就很简单了，还是来举例说明一下:
<pre>
int a,b,c,d;
a = b = 10;
c = ++a;
d = b++;
</pre>但是需要来理解这其中的原理:
>++和--操作符原理
抽象的说，前缀和后缀形式的增值操作符都复制一份变量值的拷贝，这些操作符的结果不是被他们所修改的变量，而是变量值的拷贝。

### 逗号操作符
逗号操作符将多个表达式分隔开，这些表达式从左向右逐个进行求值，__整个逗号表达式的值就是最后那个表达式的值。
<pre>
int (^f1)(int) = ^(int value){
  return value + 1;
};
int (^f2)(int) = ^(int value){
  return value + 2;
};
int (^f3)(int,int) = ^(int a,int b){
  return a + b;
};
int x,a,b,c;
x = 0;
//从这里开始
a = f1(x);
b = f2(x + a);
for (c = f3(a,b); c < 10; c = f3(a,b)) {
  printf("while statements c is:%d\n",c);
  a = f1(++x);
  b = f2(x + a);
}
//到这里结束，这其中的代码将会被修改
</pre>上面为原始的代码片段，现在我们需要使用逗号操作符来简化上面的代码，同时我选择用`while`循环来代替`for`循环。
<pre>
while (a = f1(x),b = f2(x + a),c = f3(a,b),++x,c < 10) {
  printf("while statements c is:%d\n",c);
}
</pre>现在，循环中用于获得下一个值的语句只需要出现一次，逗号操作符使源码更易于维护。
