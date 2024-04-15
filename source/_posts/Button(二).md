---
title: UITableViewCell上的Button点击效果
date: 2016-10-21 18:00:00
tags: "UIButton"
categories: "Objective-C"
---

在iOS开发中，我曾遇到这样一个问题，很久都未能解决，就是在cell上添加一个button，当我们点击button时，它是没有高亮效果的，除非我们长按button，我这里整理一下解决这个问题的方法
原文链接: <http://stackoverflow.com/questions/19256996/uibutton-not-showing-highlight-on-tap-in-ios7>
解决方案一：
```
- (void)viewDidLoad {
    [super viewDidLoad];
    self.title = @"Button点击效果测试";
    
    self.tableView.delaysContentTouches = NO;
    
    // iOS7
    for (id view in self.tableView.subviews)
    {
        if ([NSStringFromClass([view class]) isEqualToString:@"UITableViewWrapperView"])
        {
            if([view isKindOfClass:[UIScrollView class]])
            {
                UIScrollView *scroll = (UIScrollView *) view;
                scroll.delaysContentTouches = NO;
            }
            break;
        }
    }
    
    // iOS8 注意，本人测试系统iOS10，没有走这个方法，走上面那个方法
    for (id view in self.tableView.subviews)
    {
        if ([NSStringFromClass([view class]) isEqualToString:@"UITableViewCellScrollView"])
        {
            if([view isKindOfClass:[UIScrollView class]])
            {
                UIScrollView *scroll = (UIScrollView *) view;
                scroll.delaysContentTouches = NO;
            }
            break;
        }
    }
    
    // 该方式相当于上面两个循环的合集，并且实现方式更加优雅，推荐使用它，而不是使用上面两个循环
    for (id obj in self.tableView.subviews) {
        if ([obj respondsToSelector:@selector(setDelaysContentTouches:)]) {
            [obj setDelaysContentTouches:NO];
        }
    }
}
```
解决方案二：
```
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesBegan:touches withEvent:event];
    [NSOperationQueue.mainQueue addOperationWithBlock:^{ self.highlighted = YES;}];
}

-(void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesCancelled:touches withEvent:event];
    [self performSelector:@selector(setDefault) withObject:nil afterDelay:0.1];
}

-(void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    [super touchesEnded:touches withEvent:event];
    [self performSelector:@selector(setDefault) withObject:nil afterDelay:0.1];
}

- (void)setDefault
{
    [NSOperationQueue.mainQueue addOperationWithBlock:^{ self.highlighted = NO; }];
}
```
该方案比较简单粗暴，我们创建一个UIButton的分类，然后将它导入pch文件中，就彻底解决了button的点击效果问题，比起方案一要简单一些
