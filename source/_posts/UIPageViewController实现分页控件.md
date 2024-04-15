---
title: UIPageViewController实现分页控件
date: 2018-03-21 14:54:00
tags: "UIPageViewController"
categories: "Objective-C"
---

![屏幕截图](https://upload-images.jianshu.io/upload_images/1258499-92d39b4cf5356fce.gif?imageMogr2/auto-orient/strip)

使用UIPageViewController去实现这种滚动分页的控制，我们可以忽略比如view的复用，scroll的各种计算，我们只需要少量的代码就可以实现一个高性能的分页控件
我们只需要实现UIPageViewController的两个数据源代理即可
```
func pageViewController(_ pageViewController: UIPageViewController, viewControllerBefore viewController: UIViewController) -> UIViewController? {
    guard let index = (viewController as? DetailViewController)?.index else {
        return nil
    }
    if index == 0 {
        return nil
    }
    return self.dataSource?.previewController(formPage: index - 1)
}

func pageViewController(_ pageViewController: UIPageViewController, viewControllerAfter viewController: UIViewController) -> UIViewController? {
    guard let index = (viewController as? DetailViewController)?.index else {
        return nil
    }
    if index == number - 1 {
        return nil
    }
    return self.dataSource?.previewController(formPage: index + 1)
}
```
这两个代理，一个是向前翻页，一个是向后翻页
我们需要注意的是，我们不能用一个属性来计算将要展示的页面，因为有可能翻页时两个代理都会被调用，这样就很容易计算出错
所以我们把页面存储在显示的页面中，这样当需要翻页时，再取出当前页面的页码，再计算下一个界面的页码
那么我们怎么将当前的页码赋值给全局变量呢？可以通过下面的代理
```
func pageViewController(_ pageViewController: UIPageViewController, didFinishAnimating finished: Bool, previousViewControllers: [UIViewController], transitionCompleted completed: Bool) {
    self.index = (pageViewController.viewControllers?.first as? DetailViewController)?.index ?? 0
    if index < buttons.count {
        self.selected(buttons[index])
    }
}
```
该代理将会在翻页完成时调用，此时我们取出当前页面，就知道当前的页码了，然后再通过当前的页码来控制标题的变化，这样一个简单的分页控件就完成了

Demo地址：https://github.com/cdcyd/CommonControlsCollection
