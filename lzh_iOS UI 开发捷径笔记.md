# iOS UI 开发捷径
全书一共分为7章节
IB: Interface Builder
- IB概要
- IB使用
- IB中的XIB
- IB中的Auto Layout
- IB中StoryBoard
- IB进阶
- 在IB开发中技巧与bug

## IB概要:
1. IB是苹果公司提供给开发者的可视化UI工具,随着Xcode版本的更新,功能越完善,以后更倾向于IB开发
2. IB有两种文件XIB跟SB,都是以XML的形式存储
- XIB : 编译后,生成对应的xxxx.nib,属于二进制plist字典
- SB : 编译后,生成对应的xxxx.storyboardc,属于二进制plist字典
3.IB优点: 开发、维护效率高;快速学习控件;减少UI代码;
4.缺点: 团队开发容易冲突;执行效率没有代码快,但是现在硬件功能提升,影响不大

## IB使用: 

+ 1.xib的加载方法
```         

UINib *nib = [UINib nibWithNibName:@"XibView" bundle:nil];
NSArray *objs = [nib instantiateWithOwner:nil options:nil];
[self.view addSubview:objs[0]];

```
+ 2.SB的加载方法
 ```
 SecVC *secVC = [[UIStoryboard storyboardWithName:@"Demo" bundle:[NSBundle mainBundle]] instantiateViewControllerWithIdentifier:@"SecVC"];
 
 ```
+ 3.操作就是拖拽,上手就知道
+ 4.UIView创建的时候,不带xib文件, 需要单独创建,创建后关联"源文件" (在Custom Class中设置)
+ 5.IBOutlet: 链接属性 ; IBAction: 链接AddTarget,拖拽至关联的文件中
+ 6.UIVIewController创建的时候带xib,自动关联
+ 7.File's Owner:  可以理解为代理,谁拥有就可以直接在.h/.m中使用
+ 8.Is Initial View Controller:  勾选后,默认加载这个SB
+ 9.UINIb加载会缓存,  bundle每次都会加载,所以推荐使用INIb加载

##  IB中的XIB: 
#1.常用UILabel/UIbutton/UIView/UItableview/UIcollection 

