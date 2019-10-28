---
title: BLUEPILL在项目中的实践
tags: 
    - BLUEPILL
categories: 实战
---
![](/uploads/bluepill/bluepill_1.png)
  Bluepill借助于[CoreSimulator](https://github.com/mmmulani/class-dump-o-tron/tree/master/Applications/Xcode.app/Contents/Developer/Library/PrivateFrameworks/CoreSimulator.framework/Versions/A/CoreSimulator)解决稳定性和可扩展性问题。使用CoreSimulator实现了将Bluepill从Xcode模拟器中隔离出来，并使Bluepill可并行使用多种模拟器运行测试。这里Xcode模拟器是一种随每次Xcode的更新而不断进化的黑盒。（来自[InfoQ介绍bluepill](http://www.infoq.com/cn/news/2017/01/linkedin-bluepill-ios-testing)时的一段话）
<!-- more --> 
  在bluepill的[Release](https://github.com/linkedin/bluepill/releases/)可以下载到的压缩包解压，然后在终端中敲入命令:
<pre>
./bluepill -a ./Sample.app -s ./SampleAppTestScheme.xcscheme -o ./output/
</pre>
很简单，对于这最基本的命令，我们只需要有`.app`和`.xcscheme`这两个文件，然后将上面下载下来的压缩包解压之后的两个文件放在恰当的位置，应该就可以了。如果真的是这样，这篇文章也就没有存在的价值了，在这篇文章中将要做下面这几件事：
- Xcode工程项目结构体系；
-  如何获取上面提到的两个文件并使用上面这个最简单命令跑起来（是否包含了Test，如果没有需要添加Test）；
- Xcode中Test的基本操作；
- 相关参数说明；

>当前环境:
macOS Sierra 版本 10.12.3
Xcode Version 8.2.1
在我使用bluepill时，它只能在iOS 10.2环境下运行，查看Xcode支持的模拟器版本:`xcrun simctl list runtimes`

## 实践前需要知道的概念
下面介绍的这几个概念在[Apple Developer](https://developer.apple.com/library/content/featuredarticles/XcodeConcepts/Concept-Targets.html#//apple_ref/doc/uid/TP40009328-CH4-SW1)都可以找到，我仅仅把我认为比较有用的信息罗列出来一下。更多关于设置和使用方面我在文末列出来了，可以自己查看！
#### Xcode Project
 `project`包含了需要构建一个应用程序需要的所有的文件，资源和相关信息，而`workspace`是能够包含多个project的。
>注意
一个`project`是可以包含多个`targets`的！

每个`project`都定义了一个默认的__build settings__。因为每个project包含了多个`targets`，所以我们可以为每个targets设置不同的build settings（在文档中特别说明了每个targets的build settings是重写了其所属的project的build settings）。
#### Xcode Workspace
`Workspace `是可以包含任意个__project __的，但是一个project又可以属于多个WorkSpace。对于处于同一个Workspace的不同project，由于workspace中所有project的所有文件都是位于同一个文件目录中，因此__并不需要拷贝它们到每一个project文件夹中__。对于Workspace的这个共同目录，我们是可以自己指定一个构建目录（build directory）的，但是如果指定了构建目录在build project时，这个project所在的所有workspace的构建目录都将覆盖这个指定的构建目录！
#### target
`target`指定了要构建的目标，在bluepill中会包含`samepleApp`、`samepleApp_Test`等targets。同时我们可以修改target的__build setting和build phases__，target会继承project的build setting，但是我们为target指定不同的build settings来覆盖project的build setting。

#### scheme
`scheme`是需要被build的target的集合。可以在Xcode中选择不同的__scheme__来指定当前需要build的target(在Xcode中同时只能选择一个scheme)。

## 实践
### 开荒阶段
在开荒阶段是没有使用WorkSpace的，一切的基础都是在Project上。

###### 1.创建一个不包含Test和UITest的空项目
  创建一个文件夹用来保存工程项目和下载的blupill解压出来的两个文件：__bluepill和bp__；
  学过C语言的都知道，在使用`cc xxx.c`文件之后会生成一个`.o`文件，当然也可以使用`cc xxx.c -o selfpill`来重命名这个-o文件，使用selfpill文件的命令是：`./selfpill`（因为c的main的参数就是接收的终端命令），所以我猜测这其实c的一个-o文件，为了使用这两个文件，就需要将上述提到的bluepill和bp和工程项目放在同一个文件夹下。
  这一步中我们创建不包含Test和UITest的空项目即不要勾选那两个和test相关的Checkbox，你肯定会疑问为什么要这么做，当我在看[BLUEPILL的README](https://github.com/linkedin/bluepill/blob/master/README.md)的时候特别指出如果是使用终端进行build(使用xcodebuild，后面我会把这一步放在脚本中)的话需要使用`build-for-testing`，我猜测应该和test是有关的。所以我先测试一下没有test的情况是否会出错，如果出错，错误是什么？

###### 2.编译这个空项目
  项目创建好之后编译一下之后第一目的就是在终端中输入上文提到的命令，但是这里有两个疑问：-a的参数__.app__的路径是什么？-s的__.xcscheme__的路径是什么？.app就是前面我们通过编译一次获得的一个文件夹，这个文件夹的地址是`/Users/xxxx/Library/Developer/Xcode/DerivedData/`在 DerivedData文件夹下找到刚刚创建的项目并点击进去，__.app__文件夹就在`Build/Products/Debug-iphonesimulator/`，如果你在创建好之后没有编译Products就是一个空文件夹；而__.xcscheme__在`工程项目所在文件夹/bluepill_sameple.xcodeproj/xcuserdata/xxx.xcuserdatad/xcschemes/bluepill_sameple.xcscheme`路径中提到的xxx是你在创建项目时的用户名。
好了，现在我们可以在终端中数据命令:
<pre><code>./bluepill -a /Users/xxx/Library/Developer/Xcode/DerivedData/bluepill_sameple-hdwawthuaefpnyecnudlxrpzozyq/Build/Products/Debug-iphonesimulator/bluepill_sameple.app -s /Users/xxx/Desktop/bluepill_new_build/bluepill_sameple/bluepill_sameple.xcodeproj/xcuserdata/xxx.xcuserdatad/xcschemes/bluepill_sameple.xcscheme -o output</code></pre>
敲回车之后我这里出现如下报错：
<pre>
ERROR: There is no 'Plugins' folder inside your app bundle at:
/Users/Wil/Library/Developer/Xcode/DerivedData/bluepill_sameple-hdwawthuaefpnyecnudlxrpzozyq/Build/Products/Debug-iphonesimulator/bluepill_sameple.app
Perhaps you forgot to 'build-for-testing'? (Cmd + Shift + U) in Xcode.
Also, if you are using XCUITest, check https://github.com/linkedin/bluepill/issues/16
</pre>
出现这个问题的原因是在.app文件夹下面没有__Plugins__文件夹。就算是按照提示说使用`Cmd + Shift + U`或者使用`xcodebuild build-for-testing`仍然是行不通的。走到这里我们有两条路，可以删除当前工程重新建一个包含了Test和UITest的项目，也可以在现有的Project中添加Test和UITest的Targets。我选择第二条，因为在我实际项目在这之前也是没有包含相关Test的。

###### 3.在包含Test的Project中实践
  现在我们需要在既有Project中添加一个Targets，`File/New/Target.../iOS Unit Testing Bundle`创建并在__Scheme Menu__中选择`New Scheme`选中刚刚我们添加的Targets。编译一下（这里可以使用Cmd + Shift + U或者选择Test的scheme并build，也可以使用`build-for-testing`命令:
<pre><code>xcodebuild build-for-testing -project bluepill_sameple/bluepill_sameple.xcodeproj/ -scheme bluepill_sameple -sdk iphonesimulator10.2 -derivedDataPath build/</code></pre>
进入.app文件夹下查看是否已经有了`PlugIns`文件夹，如果存在该文件夹则继续执行上一次出错的那个命令，运行成功，到这里已经能够使用最基础的命令。
  前面提到可以选择Test的scheme并build，可能会出现一点问题:`The scheme 'bluepill_samepleTests' is not configured for Running`。因为添加了这个Test的Target之后需要编辑一下对应的Scheme使它能够进行Running操作！具体的操作如下所示：

![](/uploads/bluepill/bluepill_2.png)
设置之后Cmd + B成功运行！
### 完善阶段
#### 基本参数解释和使用
- __-a/-s/-o__:这是三个最基本的参数，在前面使用的时候应该知道如何使用，这里不再进行解释！
- __-c__:读取一个自定义的`config.json`文件作为bluepill的运行参数，例如:
<pre><code>./bluepill -c config.json</code></pre>
- __-l__ :获取在当前project中存在的testcase，它的打印结果是存在于`.../bluepill_sameple.app/PlugIns`文件夹下面！现在我为了演示多建立了一个Test Target(添加Scheme)然后编译：
<pre><code>./bluepill -a /Users/用户名/Library/Developer/Xcode/DerivedData/bluepill_sameple-hdwawthuaefpnyecnudlxrpzozyq/Build/Products/Debug-iphonesimulator/bluepill_sameple.app -s /Users/用户名/Desktop/bluepill_new_build/bluepill_sameple/bluepill_sameple.xcodeproj/xcuserdata/Wil.xcuserdatad/xcschemes/bluepill_sameple.xcscheme -l</code></pre>打印结果:<pre>
bluepill_samepleTests.xctest
bluepill_samepleTests_target.xctest</pre>文件夹:
![](/uploads/bluepill/bluepill_3.png)
- __-x/-i__:`-x`在运行bluepill的时候不包含某一个testcase，`-i`和`-x`相反，它则是要包含某一个testcase，`-x`的优先级是大于`-i`的。
<pre>
./bluepill -a xxxx.app -s xxxx.xcscheme -x bluepill_samepleTests.xctest -r 'iOS 10.2';
./bluepill -a xxxx.app -s xxxx.xcscheme -i bluepill_samepleTests.xctest -r 'iOS 10.2';
</pre>其中关于`-x`和`-i`的参数可使用`-l`命令来获取！
- __-d/-n__:使用命令：`xcrun simctl list devices`来获取当前可用的模拟器设备，在得到这些设备之后可以使用`-d`来指定需要启动的设备！其中`-n`可以指定同时运行的模拟器数量。比如：
<pre>
./bluepill -a xxxx.app -s xxxx.xcscheme -r 'iOS 10.2'  -d 'iPhone 5s';
</pre>其中命令前面是一样的，这里先省略！
对于真机：在[bluepill的Issue页面](https://github.com/linkedin/bluepill/issues/61)指出blupill不支持真机！

目前对于bluepill的实践先到这里，后续还会继续更新遇到的相关问题！

__相关链接:__
- [Configuring Your Xcode Project](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/ConfiguringYourApp/ConfiguringYourApp.html)
- [Xcode Build Settings Reference](https://pewpewthespells.com/blog/buildsettings.html)
