---
title: CALayer的圆角、边框和阴影详解
date: 2017-07-04 18:00:00
tags: "CALayer"
categories: "Objective-C"
---

1. 圆角
给view设置圆角，只需要设置view的layer属性的conrnerRadius，它表示图层角的曲率，默认值是0
圆角还可以用贝塞尔曲线来切，这样还可以实现单切某一个角，其它角不切的效果，我的demo中就是用该方法实现的，有兴趣的可以下下来看一看
conrnerRadius只影响背景颜色不影响背景图和子图层，所以往往我们在设置圆角时还会开启view的masksToBounds(剪裁属性)，当设置成YES时，图层里面所有东西都会被截取

2. 边框
边框需要设置layer的两个属性，borderWidth和borderColor，并且边框是沿着图层bounds绘制，同时包含图层的角
borderWidth边框的宽度，以点为单位，默认是0；borderColor边框的颜色，默认是黑色

3. 阴影
阴影一般需要设置layer的四个属性，shadowOpacity、shadowColor、shadowOffset和shadowRadius，阴影是绘制在layer的边界之外的，所以当我们设置masksToBounds属性为YES
时，阴影就会被裁剪掉
1）shadowOpacity是(0，1]之间的值，默认是0，当它大于0时，阴影就会显示，并且，值越大，阴影透明度越低
2）shadowColor 阴影的颜色，默认是黑色
3）shadowOffset 阴影的方向和距离，默认是(0, -3)，即阴影相对于Y轴有3个点的向上位移
4）shadowRadius 阴影的模糊度，当它的值是0的时候，阴影就和视图一样有一个非常确定的边界线，当值越来越大的时候，边界线看上去就会越来越模糊和自然
5）shadowPath 可以通过这个属性单独于图层形状之外指定阴影的形状

4. 圆角+阴影
从上面我们可以得出，因为对裁剪属性不同需求，在一个view上，圆角和阴影一般是不可并存的，那么我们需要怎么办呢？
在解决这个问题之前，我们还需要了解阴影的另一个特性：阴影是依据view内容的外形确定的，而不是根据边界和角半径来确定，下面放张图来解释一下
![阴影是通过里面的飞机来计算](http://upload-images.jianshu.io/upload_images/1258499-cf219a2d28defec3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以，我们圆角加阴影的实现方案就出来了，我们可以用两个视图来实现，一个只画阴影的空的外图层，和一个经过裁剪的内图层，这样外图层的阴影会根据裁剪过后的内图层来计算，这样看起来就即有阴影又有圆角了

5. 代码
因为圆角、边框、阴影每个效果的设置都需要设置2~4个属性，再加上它们可以两两组合，如果用方法传不同参数来写的化，只方法名都要写半天，所以我机智的用了链式编程的思想来写了一个分类，下面来看一下.h里面的内容
```
typedef UIView *(^ConrnerCorner) (UIRectCorner corner);
typedef UIView *(^ConrnerBounds) (CGRect bounds);
typedef UIView *(^ConrnerRadius) (CGFloat radius);
typedef UIView *(^BorderColor)   (UIColor* color);
typedef UIView *(^BorderWidth)   (CGFloat width);
typedef UIView *(^ShadowColor)   (UIColor* color);
typedef UIView *(^ShadowOffset)  (CGSize size);
typedef UIView *(^ShadowRadius)  (CGFloat radius);
typedef UIView *(^ShadowOpacity) (CGFloat opacity);
typedef UIView *(^ShowVisual) (void);
typedef UIView *(^ClerVisual) (void);
@interface UIView (Animation)
// 圆角
@property(nonatomic, strong, readonly)ConrnerCorner conrnerCorner;  // UIRectCorner 默认UIRectCornerAllCorners
@property(nonatomic, strong, readonly)ConrnerBounds conrnerBounds;  // 在使用约束布局时必传 默认CGSizeZero
@property(nonatomic, strong, readonly)ConrnerRadius conrnerRadius;  // 圆角半径 默认0
// 边框
@property(nonatomic, strong, readonly)BorderColor borderColor;      // 边框颜色 默认black
@property(nonatomic, strong, readonly)BorderWidth borderWidth;      // 边框宽度 默认0
// 阴影
@property(nonatomic, strong, readonly)ShadowColor  shadowColor;     // 阴影颜色 默认black
@property(nonatomic, strong, readonly)ShadowOffset shadowOffset;    // 阴影偏移方向和距离 默认{0，0}
@property(nonatomic, strong, readonly)ShadowRadius shadowRadius;    // 阴影模糊度 默认0
@property(nonatomic, strong, readonly)ShadowOpacity shadowOpacity;  // (0~1] 默认0
@property(nonatomic, strong, readonly)ShowVisual showVisual; // 展示
@property(nonatomic, strong, readonly)ClerVisual clerVisual; // 隐藏
@end
```

 上面的属性代表什么，以及默认值，都注释的很详细的，我就不再多讲了，这里在特别说明一下这个属性：
```
@property(nonatomic, strong, readonly)ConrnerBounds conrnerBounds;
```

 这个conrnerBounds是需要我们传入切圆角view的bounds属性，为什么需要传这个值呢？因为我切圆角的实现方法
```
UIBezierPath *radiusPath = [UIBezierPath bezierPathWithRoundedRect:CGRectEqualToRect(CGRectZero, self.cBounds)?self.bounds:self.cBounds
                                                             byRoundingCorners:self.cCorner
                                                                   cornerRadii:CGSizeMake(self.cRadius, self.cRadius)];
CAShapeLayer *maskLayer = [CAShapeLayer layer];
maskLayer.frame = view.bounds;
maskLayer.path = radiusPath.CGPath;
view.layer.mask = maskLayer;
```

 其中self.cBounds就是通过conrnerBounds赋值的，self.cCorner是通过conrnerCorner赋值的，self.cRadius是通过conrnerRadius赋值的，所以，在切圆角时，我们需要知道view的大小，如果我们用了约束或者切圆角时没有设置view的大小，这样就会吧整个view都切没了，所以在这两种情况，我们需要传一个bounds值进来，如果在切角时已经设置了view的大小，conrnerBounds就不用传了
 下面再来一个具体的用法：
```
// 圆角+边框+阴影
UIView *view = [[UIView alloc] initWithFrame:CGRectMake(50, 50, 100, 100)];
view.backgroundColor = [UIColor grayColor];
[self.view addSubview:view];
// 属性分别是：阴影透明度0.7，阴影颜色红色，阴影模糊度5，阴影方向和距离(5，5)，边框粗细2，边框颜色蓝色，圆角曲率10
// 最后设置完属性后，调用.showVisual()来展示效果，如果想清除效果，可以调用.clerVisual()来清除之前设置的效果
// 如果连续设置两次，后面的值将会覆盖前面的值
view.shadowOpacity(0.7).shadowColor([UIColor redColor]).shadowRadius(5).shadowOffset(CGSizeMake(5, 5)).borderWidth(2).borderColor([UIColor blueColor]).conrnerRadius(10).showVisual();
```

 通过上面的方法，我们就可以设置圆角、边框、阴影随意组合的效果，下面是效果图：
 ![效果图](http://upload-images.jianshu.io/upload_images/1258499-0e48bcb61e5564c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

6. Demo地址：https://github.com/cdcyd/CCViewEffects
