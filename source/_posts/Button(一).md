---
title: 自定义Button详解
date: 2016-09-18 18:00:00
tags: "UIButton"
categories: "Objective-C"
---

在开发中经常会遇到一种情况，就是按钮的UI布局(上图下文、左文右图等)和系统自带的布局(左图右文)不一样
这种情况：一种解决办法是创建一个button并在上面加一个imageView和一个label，但是这样遇到图片的位置会根据文字的长度变化的情况，会相当麻烦；另一种解决办法就是自定义一个button，这种方法更加简洁，同时处理点击事件的逻辑也更方便

1. 首先创建一个类，继承自UIButton
2. 初始化方法
```
+(instancetype)buttonWithType:(UIButtonType)buttonType{
    CCButton *ccButton = [super buttonWithType:buttonType];
    if (ccButton) {
        ccButton.titleLabel.textAlignment = NSTextAlignmentCenter;
        // 开启button的imageView的剪裁属性，根据imageView的contentMode属性选择是否开启
        ccButton.imageView.layer.masksToBounds = YES;
        ccButton.layer.masksToBounds = YES;
    }
    return ccButton;
}
```

3. 重写button的几个边界方法，return语句后面的数字，是我打断点得到这几个函数的执行顺序
```
// 返回背景边界（background）
-(CGRect)backgroundRectForBounds:(CGRect)bounds{
    NSLog(@"背景边界:%@",NSStringFromCGRect(bounds));
    return CGRectMake(0, 0, 200, 10);// 6 7  14
}

// 返回内容边界 标题+图片+标题与图片之间的间隔(title + image + the image and title separately)
-(CGRect)contentRectForBounds:(CGRect)bounds{
    NSLog(@"内容边界:%@",NSStringFromCGRect(bounds));
    return CGRectMake(0, 0, 10, 10); // 1 3 5 8 10 12 15 17 19
}

// 返回标题边界
-(CGRect)titleRectForContentRect:(CGRect)contentRect{
    NSLog(@"标题边界:%@",NSStringFromCGRect(contentRect));
    return CGRectMake(0, 0, 200, 30); // 2 11 18
}

// 返回图片边界
-(CGRect)imageRectForContentRect:(CGRect)contentRect{
    NSLog(@"图片边界:%@",NSStringFromCGRect(contentRect));
    return CGRectMake(0, 0, 100, 60); // 4 9 13 16 20
}
```
 输出如下图：
 ![](http://upload-images.jianshu.io/upload_images/1258499-6d49feb24d8181fc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 刚开始对这几个函数不是很明白，经过多次调试，才明白各自的功能：
```
// 该函数返回的是背景view的大小，参数bounds是button的大小，即button.frame
-(CGRect)backgroundRectForBounds:(CGRect)bounds {
    1.return bounds
    // 此时背景view和button的大小相同，是默认的大小

    2.return  CGRectMake(0, 0, 50, 50)，且button.frame = CGRectMake(0, 0, 100, 100)
    // 此时背景view的大小是{{0,0},{50,50}}，而button的大小是{{0,0},{100,100}}，在背景view之外的button是透明的且不能改变颜色，它可以响应点击事件

    3.return  CGRectMake(0, 0, 100, 100)，且button.frame = CGRectMake(0, 0, 50, 50)
    // 此时分两种情况，一种是layer.masksToBounds = YES，button的背景view和frame都是{{0,0},{50,50}}；另一种layer.masksToBounds = NO，button的背景view的大小是{{0,0},{100,100}}，button.frame大小是{{0,0},{50,50}}，此时界面显示是一个{{0,0},{100,100}}的button，但是只有button的{{0,0},{50,50}}范围内才会响应点击事件
}

// 该函数返回内容view的大小，内容view包括title view 、image view 和二者之间的间隔，参数bounds是button的大小，即button.frame
-(CGRect)contentRectForBounds:(CGRect)bounds {
    1.return bounds
    // 此时在返回title view边界和image view边界函数中的contentRect参数的值为button.bounds
    2.：return CGRectMake(0, 0, 100, 100)
    // 此时在返回title view边界和image view边界函数中的contentRect参数的值为{{0,0},{100，100}}
}

// 该函数返回标题view的大小，参数contentRect由函数-(CGRect)contentRectForBounds:(CGRect)bounds确定
-(CGRect)titleRectForContentRect:(CGRect)contentRect

// 该函数返回图片view的大小，参数contentRect由函数-(CGRect)contentRectForBounds:(CGRect)bounds确定
-(CGRect)imageRectForContentRect:(CGRect)contentRect
```

4. 最后写一个上图下字的示例，这只是一个简单的例子，具体情况可以根据使用场景调整
```
// 该自定义button的背景和内容view都和它的frame一样大，所以可以不用重写-(CGRect)backgroundRectForBounds:(CGRect)bounds和-(CGRect)contentRectForBounds:(CGRect)bounds这两个函数
// 返回标题边界
-(CGRect)titleRectForContentRect:(CGRect)contentRect{
    // 这contentRect就是button的frame，我们返回标题view宽和button相同，高为20，在button的底部
    return CGRectMake(0, contentRect.size.height-20, contentRect.size.width, 20);
}

// 返回图片边界
-(CGRect)imageRectForContentRect:(CGRect)contentRect{
    // button的image view 是正方形，由于title占了20的高了，所以取height - 20和width中最小的值作为image view的边长
    // 如果图片的位置是根据文字来布局的，这里可以通过self.titleLabel.text拿到title，再计算出图片的位置
    CGFloat imageWidth = MIN(contentRect.size.height - 20, contentRect.size.width);
    return CGRectMake(contentRect.size.width/2 - imageWidth/2, 0, imageWidth, imageWidth);
}
```

5. demo地址：https://github.com/cdcyd/CommonControlsCollection
