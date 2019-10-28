---
title: 动画中关于KVC官方文档翻译
tags: 
    - 动画
categories: iOS
---

CoreAnimation让CAAnimation和CALayer都遵守NSKeyValueCoding协议，因此为它们增加了一些默认的keys（对应的value），添加的keyPath中包含了了CGPoint,CGRect,CGSize和CATransform3D类型。
<!-- more --> 
## 1.键值编码兼容的容器类
CAAnimation和CALayer类就是作为键值编码兼容的容器类，我们可以根据任意的keys来设置对应的value，即便这个key不是CALayer公开的属性，比如：
```
[theLayer setValue:[NSNumber numberWithInteger:50] forKey:@"someKey"];
```
同样也可以通过任意已知的keys来查找对应的values，可以使用下面的代码通过预先设置好的somekey来检索values：
```
someKeyValue=[theLayer valueForKey:@"someKey"];
```
## 2.默认支持的value
CoreAnimation在键值编码时规定：一个类可以给没有value的key提供一个默认值。CAAnimation和CALayer类都提供了类方法``defaultValueForKey``。
>对于为key提供了默认value的类，在创建这个类的子类时必须要重写它的``defaultValueForKey``方法。

当你在实现这个方法的时候，需要检查key的参数列表，并且返回一个合适的value值，下面提供了一个例子，layer提供了``defaultValueForKey:``方法，为``maskToBounds``属性设置默认值：
```
+ (id)defaultValueForKey:(NSString *)key{
   if ([key isEqualToString:@"masksToBounds"])
      return [NSNumber numberWithBool:YES];
      return [super defaultValueForKey:key];
}
```

## 3.封装
当一个key的数据是由一个标量值或者一个C的数据结构时，**你必须要在其被分配到layer之前对其进行封装**。同样的，当要访问这些Type时，也必须检查对象，然后使用合适的方法来打开合适的值。下表显示了Objective-c和c类型封装

 | C type | 输入 | 
 | :---- :| :----: |
 | [CGPoint](https://developer.apple.com/reference/coregraphics/cgpoint) | [NSValue](https://developer.apple.com/reference/foundation/nsvalue) | 
 | [CGSize](https://developer.apple.com/reference/coregraphics/cgsize) | NSValue |
 | [CGRect](https://developer.apple.com/reference/coregraphics/cgrect) | NSValue |
 | [CATransform3D](https://developer.apple.com/reference/quartzcore/catransform3d) | NSValue |
 | [CGAffineTransform](https://developer.apple.com/reference/coregraphics/cgaffinetransform) | [NSAffineTransform](https://developer.apple.com/reference/foundation/nsaffinetransform) (OS X only) |
不同类型封装的类

## 4.为KeyPath的提供的结构
CAAnimation和CALayer类使用KeyPath来访问指定的字段，这功能可以让你在做动画时为特定的KeyPath提供数据。使用``setValue:forKeyPath和valueForKeyPath:``方法设置，然后用``valueForKeyPath:``获取相应的值。
#### (1)、CATransform3D KeyPaths
你可以使用更强大的KeyPath，查找包含了``CATransform3D``类型属性的值。在需要指定layer的``transforms``完整的KeyPath时，我们可以根据下表中提供的数据，使用``transform``和``sublayerTransform``的值。例如，我们需要制定绕着layer的z轴旋转时，我就需要指定KeyPath为``transform.rotation.z``。

 | Field Key Path | 描述 | 
 | :---- :| :----: |
 | rotation.x | 围绕X轴，旋转值为弧度，``NSNumber``类型 | 
 | rotation.y | 围绕y轴，旋转值为弧度，``NSNumber``类型 |
 | rotation.z | 围绕z轴，旋转值为弧度，``NSNumber``类型 |
 | rotation | 围绕z轴，旋转值为弧度，``NSNumber``类型，它和设置``rotation.z``一样 |
 | scale.x | x轴缩放，``NSNumber``类型 |
 | scale.y | y轴缩放，``NSNumber``类型  |
 | scale.z | z轴缩放，``NSNumber``类型  |
 | scale | 三个轴缩放的平均值，``NSNumber``类型 |
 | translation.x | x轴位移，``NSNumber``类型 |
 | translation.y | y轴位移，``NSNumber``类型 |
 | translation.z | z轴位移，``NSNumber``类型 |
 | translation | x，y上面位移，``NSSize`` 和``CGSize``|

下面展示了怎样通过setValue:forKeyPath方法来修改一个layer，这个例子设置了layer在x轴上位移了10个像素点，来显示layer在x轴上的移动:
```
[myLayer setValue:[NSNumber numberWithFloat:10.0] forKeyPath:@"transform.translation.x"];
```
>⚠注意：通过keyPath来设置value值的时候不能像Objective-C里面对属性的赋值，必须配合KeyPath字符串使用setValue:forKeyPath方法来进行赋值。

#### (2)、CGPoint KeyPath
如果当前给的是一个``CGPoint``类型，则可以根据下表进行设置。例如，当我们想要修改layer的``position``的x值时，可以在KeyPath中写``position.x``。

 | Structure Field | 描述 | 
 | :---- :| :----: |
 | x | x的分量 | 
 | y | y的分量 |

#### (3)、CGSize KeyPath
 | Structure Field | 描述 | 
 | :---- :| :----: |
 | width | size的width值 | 
 | height | size的height值 |

#### (4)、CGRect KeyPath
例如，要更改layer的``bounds``属性的width值，可以写入关键路径``bounds.size.width``

 | Structure Field | 描述 | 
 | :---- :| :----: |
 | origin | 坐标，类型``CGPoint`` | 
 | origin.x | 坐标的x值，类型``CGFloat`` |
 | origin.y | 坐标的y值，类型``CGFloat `` |
 | size | 大小，类型``CGSize`` |
 | size.width | size的width值 |
 | size.height | size的height值 |

## 结语
翻译这篇文章的目的因为我在做动画中需要每次都差到对应的KeyPath，很麻烦，索性我就将其翻译出来。
到目前为止，这片文章大部分翻译算是完成了，看起来很粗糙，能看懂就最好了。

原来地址：[Key-Value Coding Extensions](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Key-ValueCodingExtensions/Key-ValueCodingExtensions.html)
