---
title: 《C和指针Note》之指针
tags: 
    - C
    - 指针
categories: C语言
---
指针。
<!-- more --> 
当前C语言环境:
<pre>
Apple LLVM version 8.0.0 (clang-800.0.38)
Target: x86_64-apple-darwin15.6.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
</pre>

### 粗略的总结一下知识点
* 1.变量标识符与内存位置之间的关联并不是硬件提供的，而是编译器为我们提供的，__硬件仍然通过地址访问内存位置。__
* 2.数组中的元素存储于连续的内存地址中。
* 3.`NULL`指针作为一个不指向任何东西的特殊指针。__在对指针进行解引用操作之前，必须确保它不是`NULL`指针__(因为对一个NULL指针进行解引用是非法操作)。
* 4.指针的指针中，*操作符具有从右向左的结合性，所以这个表达式相当于`*(*c)`，所以可以从里向外求值。
对于如下代码中:
<pre>int a = 10;
int *b = &a;
int **c = &b;</pre>

| 表达式 | 相当的表达式 |
| ---- |:----:|
| a     | 10 |
| b    | &a  |
| *b 	| 12,a |
| c 	| &b   |
| *c 	| &a,b |
| **c | 12,a,*b |

>注意⚠️
在指针没有被初始化之前，一定不要对这个指针变量使用间接操作符。

### 习题练习纪录
* 1.字符串查找相关，两个字符串中找出第一个相同的字符串。
解答:
<pre>
char *find_char(char const *source,char const *chars){
    if (source == NULL || chars == NULL) {
        return NULL;
    }
    char const *f_p = chars;
    do {
        do {
            if (*source == *chars) {
                char *result = (char *)source;
                char *cp;
                *cp = *result;
                return cp;
            }
        } while (*chars++ != '\0');
        chars = f_p;
    } while (*source++ != '\0');
    return NULL;
}
</pre>
