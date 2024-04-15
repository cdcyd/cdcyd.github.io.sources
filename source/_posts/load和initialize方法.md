---
title: load和initialize方法
date: 2018-07-24 14:29:00
tags: "Foundation"
categories: "Objective-C"
---

在iOS中，所有的类都继承自NSObject，我们来看一下初始化相关的几个方法
```
+ (void)load;
+ (void)initialize;
- (instancetype)init ;

+ (instancetype)new OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)allocWithZone:(struct _NSZone *)zone OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
+ (instancetype)alloc OBJC_SWIFT_UNAVAILABLE("use object initializers instead");
- (void)dealloc OBJC_SWIFT_UNAVAILABLE("use 'deinit' to define a de-initializer");
```

OBJC_SWIFT_UNAVAILABLE 宏表示只能在OC中使用，在Swift中不能使用
`+(instancetype)new` 可以看作是alloc与init方法的结合
`+(instancetype)allocWithZone` alloc也是调用该方法，参数传NULL
`+(instancetype)alloc` 为对象分配内存空间
`-(instancetype)init` 初始化变量
`-(void)dealloc` 销毁对象时调用的方法
上面的方法是我们开发时比较常用的，也很好理解，而load和initialize这两个方法并不常用，而且也有点特殊，下面我们就来详细说一下这两个方法

在介绍之前，我们首先来了解一下类的使用，我们要使用一个类，大概要经过以下步骤
1. 启动App，程序开始加载类到内存中(代码区)+(void)load
2. 首次使用该类时，创建类对象(我们可以把它看作是一个单例，它在整个程序中只有一份)+(void)initialize
3. 通过类对象创建实例对象+(instancetype)alloc、-(instancetype)init
4. 通过实例对象，我们就可使用实例方法、类属性了

从上面的步骤我们也大概了解到load和initialize的调用时机了，下面在来详细说一下

`+(void)load`
1. 在App启动后立即加载类，此时就会调用该函数，所以它的调用时机很早，甚至在main函数之前
2. 两个不相关的类的加载顺序是随机的
3. 如果一个类没有load方法，则该类就不会调用load方法，它不会去继承父类的load
4. 如果两个类有依赖关系，则优先加载被依赖的类
5. 如果两个类是继承关系，则优先加载父类，再加载子类
6. Category的load也会收到调用，但顺序上在主类的load调用之后

所以在load方法中，我们不需要调用super，因为在加载子类之前肯定加载完成父类了，即父类的load方法肯定已经执行过了，同时这里也有一个缺点，因为load方法执行时，运行环境还不确定，如果我们在load函数中使用了其它不相关的类，我们并不能保证该类已经加载完成且可用，甚至有时还要自己负责做auto release处理

`+(void)initialize`
1. 在首次使用类时，会生成类对象，该方法在此时调用，initialize其实就是一个懒加载方法
2. 如果两个类是继承关系，会先调用父类，再调用子类
2. 如果子类没有实现initialize，则会继承父类的initialize

在initialize方法中，我们同样不需要调用super，因为调用子类之前，父类已经调用一次了，而且如果子类没有实现initialize方法时，在首次使用子类时还会调用一次父类的方法，它与load方法还有不同的是，在initialize调用时，运行环境基本健全(在main函数之后，我们要保证在load方法中没有使用该类，不然initialize就没有该优势)，所以此时我们可以做更多的操作

上面原理讲完了，下面再写一个Demo来测试一下

##### 一、先来测试一下load方法，下面贴出源码，很简单
```
// Test继承自NSObject
@implementation Test
+ (void)load {
    NSLog(@"load Test");
}
@end

// Test1继承自Test类
@implementation Test1
+ (void)load {
    NSLog(@"load Test1");
}
@end

int main(int argc, char * argv[]) {
    @autoreleasepool {
        NSLog(@"main");
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```
下面是输出
```
2018-07-24 14:17:28.113416+0800 Test[1142:222096] load Test
2018-07-24 14:17:28.113760+0800 Test[1142:222096] load Test1
2018-07-24 14:17:28.113851+0800 Test[1142:222096] main
```
可以看出调用顺序 父类Text  -> 子类Text1  -> main函数
如果注释掉子类的load方法，则调用如下
```
2018-07-24 14:20:13.292581+0800 Test[1148:222715] load Test
2018-07-24 14:20:13.292943+0800 Test[1148:222715] main
```
这个测试没有覆盖load方法的所有特性，但可以测出上面所说的load方法的第1、3、5条特性

##### 二、再来测试一下initialize方法调用
```
@implementation Test
+ (void)initialize {
    NSLog(@"initialize Test");
}
@end

@implementation Test1
+ (void)initialize {
    NSLog(@"initialize Test1");
}
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];

    NSLog(@"首次使用Test");
    Test *obj = [[Test alloc] init];

    NSLog(@"首次使用Test1");
    Test1 *obj2 = [[Test1 alloc] init];
}
@end
```
输出如下
```
2018-07-24 14:24:34.439448+0800 Test[1152:223417] main
2018-07-24 14:24:34.490282+0800 Test[1152:223417] 首次使用Test
2018-07-24 14:24:34.490326+0800 Test[1152:223417] initialize Test
2018-07-24 14:24:34.490346+0800 Test[1152:223417] 首次使用Test1
2018-07-24 14:24:34.490361+0800 Test[1152:223417] initialize Test1
```
如果注释掉Text1的initialize方法后输出如下
```
2018-07-24 14:25:25.371237+0800 Test[1154:223686] main
2018-07-24 14:25:25.423578+0800 Test[1154:223686] 首次使用Test
2018-07-24 14:25:25.423622+0800 Test[1154:223686] initialize Test
2018-07-24 14:25:25.423642+0800 Test[1154:223686] 首次使用Test1
2018-07-24 14:25:25.423655+0800 Test[1154:223686] initialize Test
```
从输出可以看出上面说的initialize的3个特性