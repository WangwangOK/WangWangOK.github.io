---
title: 创建一个更轻的ViewController
tags: 
    - iOS
categories: 实战
---
现在的项目越来越臃肿，一个控制器的代码量越来越多，业务最繁重的一个控制器代码量已经达到了1000多行！这就导致给控制器瘦身是一定要做的。
<!-- more --> 
说实话这种的是很不好组织语言的，但是还是按照惯例还是在这里来记录一下自己想法经历和结果。
>注意:
>文中会同时出现Swift代码和objc代码，出现这个的原因是为了演示针对使用AF进行网络请求和数据处理，就用了其他工程的代码来进行演示。
>样例工程是用Swift写的没有集成AF库，只是模拟了一个网络请求，但是Swift的代码同样适用于objc，这里大多的管理类都是继承自`NSObject`，希望见谅吧。
<br>

## 目标
现在我们的目的是来分离一个登录页面的Controller。我先把这个界面的效果图放出来

![登录界面](/uploads/light_controller/light_controller_1.png)

## 分析
这个页面比较简单，一个简单的输入用户名称和密码，然后登录给出正确还是错误的结果！这里需要简单处理一下用户名密码的输入情况来绝对按钮的颜色的交互情况。
我将一个业务场景进行划分可以分为以下5个：
>1.控制器 -------Controller
>2.视图 ---------ViewOperator
>3.网络请求-----Request
>4.数据处理-----DataManager
>5.协议代理-----Delegate

### 职责划分
###### 1.控制器(Controller)
控制器的职责需要统调`视图(ViewOperator)`,`网络请求(Request)`,`数据处理(DataManager)`,`代理协议(Delegate)`以及不同控制器之间的联系。
###### 2.视图(ViewOperator)
ViewOperator的作用是将控制器里面与视图相关的操作剥离出来。同时也包括了按钮的点击。
###### 3.协议代理(Delegate)
剥离出controller中的delegate的回调，比如登录界面UITextField的相关代理操作，也可以是TableView的delegate和dataSource的分离。
###### 4.网络请求(Request)
分离控制器中的网络请求部分，比如在一个包含了tableView的界面，数据加载，下拉刷新，上拉加载的相关方法.
###### 5.数据处理(DataManager)
在网络请求完成的回调，回调回来的数据是不需要放在网络请求里面去做的，所以将网络请求数据回调放在`DataManager`中来。数据处理完成是需要给Controller一个数据回调的。

### 第一步来分离Controller中的视图
通常视图相关的操作如果都放在Controller会产生很多行的代码的，所以第一步我们需要将控制器和视图相关操作分离出来。
在使用StoryBoard来创建一个ViewController时，需要在控制器中拖一些`button`,`label`,`textfield `等等控件，这些控件的定制以及交互等等都要拖出代码进行处理。
所以这里我可以给当前控制器的Scene添加一个`object`

![object](/uploads/light_controller/light_controller_2.png)
然后自定义个`LoginViewOperator`来继承自`NSObject`，将这个object的CustomClass改为`LoginViewOperator `。此时你可以将上诉的控件都拖到这个class里面，比如:
<pre>
public class LoginViewOperator:NSObject{
    @IBOutlet var fields: [UITextField]!
    @IBOutlet weak var login_btn: UIButton!
    @IBOutlet weak var indicator: UIActivityIndicatorView!
    @IBAction func login(_ sender: UIButton) {
    }
}
</pre>同时我也可以直接将`LoginViewOperator `拖到Controller中，来让Controller来管理LoginViewOperator `。
<pre>
class LoginViewController {
    @IBOutlet var view_operator: LoginViewOperator!
}
</pre>
如果有自定义视图的话，自定义视图的相关操作可以放在这个类中去做，或者可以给这个视图添加一个`helper`(`CustomViewHelper`)通过他来做自定义视图的相关操作

### 第二步来分离Controller中的代理协议
加入在一个业务场景中有一个tableView，或者textfield，或者其他的系统的控件等等，和他们打交道肯定需要回调的，把这些放在控制器中也是很占位置的，必须将他们分离出来。
在当前场景主要是`UITextFieldDelegate`的分离，我们需要创建一个类继承自`NSObject`,并让他遵守`UITextFieldDelegate`。
<pre>
public class EDLoginTextFieldDelegate:NSObject{
   public var textdidChange:((UITextField)->Void)?
}
extension EDLoginTextFieldDelegate:UITextFieldDelegate{
    public func textFieldDidChange(_ textField: UITextField){
        textdidChange?(textField)
    }
    public func textFieldShouldReturn(_ textField: UITextField) -> Bool {
        return true
    }
    public func textField(_ textField: UITextField, shouldChangeCharactersIn range: NSRange, replacementString string: String) -> Bool {
        return true
    }
}
</pre>上面方法中`textFieldDidChange `是通过在`LoginViewOperator`中的UITextField添加的target-action的，这个会在接下来讲到。
此时在`LoginViewOperator`新建一个变量`delegate`:
<pre>
lazy var delegate:EDLoginTextFieldDelegate = EDLoginTextFieldDelegate().then{
   //text did change
  $0.textdidChange = {[unowned self] in
    //在这里处理当textfield在发生变化时，视图需要做出的改变，也就是上面gif中通过判断输入框中是否有值来改变button的alpha和enable的操作
  }
  //text did end
  //text did begin
}
</pre>同时将`delegate`赋值给UITextField的delegate，修改上面outlet出来的`fields `变量：
<pre>
@IBOutlet var fields: [UITextField]!{
      didSet{
        fields.forEach{
          $0.delegate = delegate
          //给每一个field都添加一个target-action，这个action不要写在`LoginViewOperator`中，因为这个实际上和delegate是类似的，不是一个视图操作，所以我这里绕了一圈在delegate的textdidChange进行了回调。
          $0.addTarget(delegate, action: #selector(delegate.textFieldDidChange(_:)), for: .editingChanged)
      }
      if let username = User.mine.name{
         fields.first?.text = username
      }
    }
  }
</pre>上面就是我在分离一个`UITextFieldDelegate`所做的操作。
在这里看来额外添加的这个`delegate`可能多此一举，因为操作相对比较少，但是如果是在tableView中的话，你需要在`numberOfSections`,`numberOfRowsInSection`,`cellForRowAt`以及`willDisplayCell`中需要写一大堆的逻辑，在这里就会体现出协议代理的分离带来的好处。
>在这里建议：如果在使用数据回调的时候可以使用delegate而不是block，这样就可以把放在controller中的block代码分离到CustomDelegate中来，后面在分离数据处理部分会用到delgate。

### 第三步来分离Controller中的网络请求
这一步我用objc的代码来进行讲解，因为在样例工程是没有涉及到网络请求的，所以我将我们公司项目的中的网络请求部分的分离代码贴出来。
同样我们需要创建一个类`DailyProhRequst`来继承于`NSObject`，该类分离控制器的网络请求。
<pre>
@interface DailyProhRequst : NSObject
@end
</pre>
并为`DaulyProhRequst`添加三个方法:
>下拉加载

`+ (NSInteger)headerRefresh:(NSDictionary *)para currentPage:(NSInteger)page dataOperator:(DailyProhDataOperator *)operators;`
>上拉刷新的网络操作

`+ (NSInteger)footerRefresh:(NSDictionary *)para currentPage:(NSInteger)page dataOperator:(DailyProhDataOperator *)operators;`
>简单的数据加载

`+ (NSInteger)reloadData:(NSDictionary *)para currentPage:(NSInteger)page dataOperator:(DailyProhDataOperator *)operators;`
如果需要上拉加载下拉刷新数据请求之类的，可以直接调用上面上个方法中的一个，在上面方法中还提到了`dataOperator `，它是用来整理数据的，这部分放到后面来讲。这三个方法只进行网络并把数据传给`dataOperator `，request对controller是没有回调的！<br>
在上面的分离视图部分，我们将登录按钮的`@IBAction`放在了`LoginViewOperator`中，我们需要将这个按钮事件回调给控制器。
其实严谨来讲，我们是没有必要将按钮的`IBAction `放在视图中来的，这里是我的失误。
所以在控制器中点击按钮进行一个网络请求的操作就是`[DailyProhRequst headerRefresh:@{} currentPage:weakSelf.currentPage dataOperator:weakSelf.dataOperator]`，网络请求的分离相对来说比较简单。

### 第四步创建数据操作类来管理数据
同上面一样适用objc的代码进行讲解。这里数据可以分为网络请求返回的数据和抽取本地db的数据，我主要讲一下通过网络获取数据进行数据处理的数据管理类。
首先我们同样需要创建一个类来继承自`NSObject`，并添加一个方法从`Requst`中获取原始的网络数据:
<pre>
@interface DailyProhDataOperator : NSObject
@end
</pre>同时我们给DailyProhDataOperator类添加一个获取数据的方法
`- (void)fetchData:(id)responseobject requstFailed:(NSError *)error;`
而且我们的目的是必须要将处理好的数据（可能是你自己定一个数据模型）返回给Controller来更新View（由ViewOperator来更新操作）,所以需要创建一个delegate：
<pre>
@protocol DailyProhDataOperatorHandler <NSObject>
@end
</pre>
并给协议添加两个回调方法 
`- (void)dataOperatorSuccess:(DailyProhDataModel *)dataSource;`
` - (void)dataOperatorFailed:(NSError *)error;`
然后给`DailyProhDataOperator`类添加一个代理属性
`@property (weak ) id<DailyProhDataOperatorHandler> handler;`
并在网络请求分离中的方法中进行修改:
<pre> +(NSInteger)reloadData:(NSDictionary *)para currentPage:(NSInteger)page dataOperator:(DailyProhDataOperator *)operators{
      //在网络请求或者失败的回调中添加数据处理类的入口
      //[operators fetchData:responseObject requstFailed:nil];
      return 0;
}
</pre>在DataOperator中进行数据处理完成的回调
`[self.handler dataOperatorSuccess:data];`
这个delegate可以通过上面已经讲过的协议代理分离放在CustomDelegate的类中，由CustomDelegate和ViewOperator进行通信来达到根据数据来修改视图的效果。
<br>
### 结果
虽然在样例工程中没有网络请求的部分，但是实际上网络请求在控制器中所占的代码就仅仅一个跳转相关的逻辑的代码而已，所以经过上诉的方式对controller进行瘦身之后，一个完整的功能的登录界面代码量为：

![瘦身的结果](/uploads/light_controller/light_controller_3.png)
图中的request是我模拟的一个假的网络请求。
### 总结
通过上诉的方式可以很大程度上将控制器代码进行缩减，而且指责划分也很明确。最后来说一下他们之间的关系：
>###### Controller <-> ViewOperator
通过ViewOperator修改视图，Controller相关交互之后通过ViewOperator来进行界面更行

>###### Controller  <-> Delegate
由delegate来分担Controller中的相关代理协议的工作，包括从delegate拿取数据等等，

>###### Controller <-> Requst
由Controller来发起Requst

>###### Requst <-> DataOperator
DataOperator从Request获取数据，并对数据进行整理

>###### Delegate <-> DataOperator
DataOperator把整理好的数据返回给Delegate

>###### Delegate <-> ViewOperator
Delegate把数据传给ViewOperator之后，由ViewOperator更新视图

这个只是罗列出了在当前业务逻辑下面他们之间的相互关系，只是为了梳理一下思路，在不同的逻辑下有不同的组合，但是一个业务场景在代码实现层面可以大体分成五种或者更多的。
