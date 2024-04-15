---
title: MJRefresh 源码阅读
date: 2018-06-04 18:00:00
tags: "Refresh"
categories: "源码阅读"
---

MJRefresh项目地址 https://github.com/CoderMJLee/MJRefresh
下载下来后我们打开项目可以看到下面的目录
![MJ项目结构](https://upload-images.jianshu.io/upload_images/1258499-7bd09c5e9a636edd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
1. MJRefresh目录下就是下拉刷新的实现，其中
Base：是实现刷新的核心代码，里面实现了刷新的基础控件(Header/Footer)
Custom：是一些自定义的刷新控件，比如自动刷新、Gif动画刷新等
MJRefresh.bundle：多语言处理
其它的还有`MJRefreshConst`常量定义，还有一些扩展(通过runtime增加mj需要的属性)
2. Classes目录下是MJ官方文档中示例的实现，我们阅读源码可以忽略它

虽然MJRefresh里面有很多的类，咋一看好像很复杂，其实它实现的核心只有一个类，其它的都是对它进行一层一层的封装，我们可以用官网的一张图来表示MJ的结构
![MJ结构图.png](https://upload-images.jianshu.io/upload_images/1258499-ff9984995da3bf37.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图可以看出，最基础的类就是MJRefreshComonent，我们就来看一下里面都做了什么
MJRefreshComonent继承自UIView，它就是刷新时展示的自定义View
```
- (instancetype)initWithFrame:(CGRect)frame
{
    if (self = [super initWithFrame:frame]) {
        // 准备工作
        [self prepare];
        // 默认是普通状态
        self.state = MJRefreshStateIdle;
    }
    return self;
}
- (void)prepare
{
    // 基本属性
    self.autoresizingMask = UIViewAutoresizingFlexibleWidth;
    self.backgroundColor = [UIColor clearColor];
}
- (void)layoutSubviews
{
    [self placeSubviews];
    [super layoutSubviews];
}
- (void)placeSubviews{}
- (void)willMoveToSuperview:(UIView *)newSuperview
{
    [super willMoveToSuperview:newSuperview];
    // 如果不是UIScrollView，不做任何事情
    if (newSuperview && ![newSuperview isKindOfClass:[UIScrollView class]]) return;
    // 旧的父控件移除监听
    [self removeObservers];
    if (newSuperview) { // 新的父控件
        // 设置宽度
        self.mj_w = newSuperview.mj_w;
        // 设置位置
        self.mj_x = -_scrollView.mj_insetL;
        // 记录UIScrollView
        _scrollView = (UIScrollView *)newSuperview;
        // 设置永远支持垂直弹簧效果
        _scrollView.alwaysBounceVertical = YES;
        // 记录UIScrollView最开始的contentInset
        _scrollViewOriginalInset = _scrollView.mj_inset;
        // 添加监听
        [self addObservers];
    }
}
- (void)drawRect:(CGRect)rect
{
    [super drawRect:rect];
    if (self.state == MJRefreshStateWillRefresh) {
        // 预防view还没显示出来就调用了beginRefreshing
        self.state = MJRefreshStateRefreshing;
    }
}
```
代码注释原作者已经写的很详细了，总结一下就是
1. 设置view的基本属性(自动布局)autoresizingMask、背景(backgroundColor)
2. 重新定义了初始化相关的接口 `- (void)prepare` 和 `- (void)layoutSubviews` 它们分别是初始化函数和开始加载UI的函数，子类继承时需要实现它们
3. 弱引用父视图，并设置对父视图的监听(这里有个细节是`- (void)willMoveToSuperview:(UIView *)newSuperview`函数在view添加和移除时都会调用，所以只要调用该函数，就移除一次监听，然后再添加监听，这样就不会出现忘记移除监听而出现的Crash)

再来看一下监听
```
- (void)addObservers
{
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.scrollView addObserver:self forKeyPath:MJRefreshKeyPathContentOffset options:options context:nil];
    [self.scrollView addObserver:self forKeyPath:MJRefreshKeyPathContentSize options:options context:nil];
    self.pan = self.scrollView.panGestureRecognizer;
    [self.pan addObserver:self forKeyPath:MJRefreshKeyPathPanState options:options context:nil];
}

- (void)removeObservers
{
    [self.superview removeObserver:self forKeyPath:MJRefreshKeyPathContentOffset];
    [self.superview removeObserver:self forKeyPath:MJRefreshKeyPathContentSize];
    [self.pan removeObserver:self forKeyPath:MJRefreshKeyPathPanState];
    self.pan = nil;
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    // 遇到这些情况就直接返回
    if (!self.userInteractionEnabled) return;
    // 这个就算看不见也需要处理
    if ([keyPath isEqualToString:MJRefreshKeyPathContentSize]) {
        [self scrollViewContentSizeDidChange:change];
    }
    // 看不见
    if (self.hidden) return;
    if ([keyPath isEqualToString:MJRefreshKeyPathContentOffset]) {
        [self scrollViewContentOffsetDidChange:change];
    } else if ([keyPath isEqualToString:MJRefreshKeyPathPanState]) {
        [self scrollViewPanStateDidChange:change];
    }
}

- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change{}
- (void)scrollViewContentSizeDidChange:(NSDictionary *)change{}
- (void)scrollViewPanStateDidChange:(NSDictionary *)change{}
```
从上面可以看出一共有3个监听
1. ContentOffset：监听scrollview的滑动情况
2. ContentSize：监听scrollview的contentSize的大小变换
3. PanGesture：监听scrollview上pan手势的状态变化

监听的方法都没有具体的实现，说明它是需要子类去实现，所以MJRefreshComonent是一个抽象类，直接使用它是没有意义的，我们需要创建子类来继承它，下面再来看一下继承它的子类
1. MJRefreshHeader，下面是该类的核心函数
```
- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change
{
    [super scrollViewContentOffsetDidChange:change];
    // 在刷新的refreshing状态
    if (self.state == MJRefreshStateRefreshing) {
        // 暂时保留
        if (self.window == nil) return;
        // sectionheader停留解决
        CGFloat insetT = - self.scrollView.mj_offsetY > _scrollViewOriginalInset.top ? - self.scrollView.mj_offsetY : _scrollViewOriginalInset.top;
        insetT = insetT > self.mj_h + _scrollViewOriginalInset.top ? self.mj_h + _scrollViewOriginalInset.top : insetT;
        self.scrollView.mj_insetT = insetT;
        self.insetTDelta = _scrollViewOriginalInset.top - insetT;
        return;
    }
    
    // 跳转到下一个控制器时，contentInset可能会变
     _scrollViewOriginalInset = self.scrollView.mj_inset;
    // 当前的contentOffset
    CGFloat offsetY = self.scrollView.mj_offsetY;
    // 头部控件刚好出现的offsetY
    CGFloat happenOffsetY = - self.scrollViewOriginalInset.top;
    // 如果是向上滚动到看不见头部控件，直接返回
    // >= -> >
    if (offsetY > happenOffsetY) return;
    // 普通 和 即将刷新 的临界点
    CGFloat normal2pullingOffsetY = happenOffsetY - self.mj_h;
    CGFloat pullingPercent = (happenOffsetY - offsetY) / self.mj_h;
    if (self.scrollView.isDragging) { // 如果正在拖拽
        self.pullingPercent = pullingPercent;
        if (self.state == MJRefreshStateIdle && offsetY < normal2pullingOffsetY) {
            // 转为即将刷新状态
            self.state = MJRefreshStatePulling;
        } else if (self.state == MJRefreshStatePulling && offsetY >= normal2pullingOffsetY) {
            // 转为普通状态
            self.state = MJRefreshStateIdle;
        }
    } else if (self.state == MJRefreshStatePulling) {// 即将刷新 && 手松开
        // 开始刷新
        [self beginRefreshing];
    } else if (pullingPercent < 1) {
        self.pullingPercent = pullingPercent;
    }
}

- (void)setState:(MJRefreshState)state
{
    MJRefreshCheckState
    // 根据状态做事情
    if (state == MJRefreshStateIdle) {
        if (oldState != MJRefreshStateRefreshing) return;
        // 保存刷新时间
        [[NSUserDefaults standardUserDefaults] setObject:[NSDate date] forKey:self.lastUpdatedTimeKey];
        [[NSUserDefaults standardUserDefaults] synchronize];
        // 恢复inset和offset
        [UIView animateWithDuration:MJRefreshSlowAnimationDuration animations:^{
            self.scrollView.mj_insetT += self.insetTDelta;
            // 自动调整透明度
            if (self.isAutomaticallyChangeAlpha) self.alpha = 0.0;
        } completion:^(BOOL finished) {
            self.pullingPercent = 0.0;
            if (self.endRefreshingCompletionBlock) {
                self.endRefreshingCompletionBlock();
            }
        }];
    } else if (state == MJRefreshStateRefreshing) {
         dispatch_async(dispatch_get_main_queue(), ^{
            [UIView animateWithDuration:MJRefreshFastAnimationDuration animations:^{
                CGFloat top = self.scrollViewOriginalInset.top + self.mj_h;
                // 增加滚动区域top
                self.scrollView.mj_insetT = top;
                // 设置滚动位置
                CGPoint offset = self.scrollView.contentOffset;
                offset.y = -top;
                [self.scrollView setContentOffset:offset animated:NO];
            } completion:^(BOOL finished) {
                [self executeRefreshingCallback];
            }];
         });
    }
}
```
MJRefreshHeader通过继承实现父类`- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change`函数，在该函数中做了一件事就是，当scrollView滑动时，判断当前scrollView应该处于什么状态，然后再通过`- (void)setState:(MJRefreshState)state`函数来更新UI，这样一个简单的下拉刷新就实现了
2. MJRefreshAutoFooter、MJRefreshBackFooter(auto和back两个模式区别是，一个自适应尾部刷新控件位置，一个刷新控件位置始终在底部)

auto的核心函数
```
- (void)scrollViewContentSizeDidChange:(NSDictionary *)change
{
    [super scrollViewContentSizeDidChange:change];
    // 设置位置
    self.mj_y = self.scrollView.mj_contentH;
}

- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change
{
    [super scrollViewContentOffsetDidChange:change];
    if (self.state != MJRefreshStateIdle || !self.automaticallyRefresh || self.mj_y == 0) return;
    if (_scrollView.mj_insetT + _scrollView.mj_contentH > _scrollView.mj_h) { // 内容超过一个屏幕
        // 这里的_scrollView.mj_contentH替换掉self.mj_y更为合理
        if (_scrollView.mj_offsetY >= _scrollView.mj_contentH - _scrollView.mj_h + self.mj_h * self.triggerAutomaticallyRefreshPercent + _scrollView.mj_insetB - self.mj_h) {
            // 防止手松开时连续调用
            CGPoint old = [change[@"old"] CGPointValue];
            CGPoint new = [change[@"new"] CGPointValue];
            if (new.y <= old.y) return;
            // 当底部刷新控件完全出现时，才刷新
            [self beginRefreshing];
        }
    }
}

- (void)scrollViewPanStateDidChange:(NSDictionary *)change
{
    [super scrollViewPanStateDidChange:change];
    if (self.state != MJRefreshStateIdle) return;
    UIGestureRecognizerState panState = _scrollView.panGestureRecognizer.state;
    if (panState == UIGestureRecognizerStateEnded) {// 手松开
        if (_scrollView.mj_insetT + _scrollView.mj_contentH <= _scrollView.mj_h) {  // 不够一个屏幕
            if (_scrollView.mj_offsetY >= - _scrollView.mj_insetT) { // 向上拽
                [self beginRefreshing];
            }
        } else { // 超出一个屏幕
            if (_scrollView.mj_offsetY >= _scrollView.mj_contentH + _scrollView.mj_insetB - _scrollView.mj_h) {
                [self beginRefreshing];
            }
        }
    } else if (panState == UIGestureRecognizerStateBegan) {
        self.oneNewPan = YES;
    }
}

- (void)setState:(MJRefreshState)state
{
    MJRefreshCheckState
    if (state == MJRefreshStateRefreshing) {
        [self executeRefreshingCallback];
    } else if (state == MJRefreshStateNoMoreData || state == MJRefreshStateIdle) {
        if (MJRefreshStateRefreshing == oldState) {
            if (self.endRefreshingCompletionBlock) {
                self.endRefreshingCompletionBlock();
            }
        }
    }
}
```
back的核心函数
```
- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change
{
    [super scrollViewContentOffsetDidChange:change];
    // 如果正在刷新，直接返回
    if (self.state == MJRefreshStateRefreshing) return;
    _scrollViewOriginalInset = self.scrollView.mj_inset;
    // 当前的contentOffset
    CGFloat currentOffsetY = self.scrollView.mj_offsetY;
    // 尾部控件刚好出现的offsetY
    CGFloat happenOffsetY = [self happenOffsetY];
    // 如果是向下滚动到看不见尾部控件，直接返回
    if (currentOffsetY <= happenOffsetY) return;
    CGFloat pullingPercent = (currentOffsetY - happenOffsetY) / self.mj_h;
    // 如果已全部加载，仅设置pullingPercent，然后返回
    if (self.state == MJRefreshStateNoMoreData) {
        self.pullingPercent = pullingPercent;
        return;
    }
    if (self.scrollView.isDragging) {
        self.pullingPercent = pullingPercent;
        // 普通 和 即将刷新 的临界点
        CGFloat normal2pullingOffsetY = happenOffsetY + self.mj_h;
        if (self.state == MJRefreshStateIdle && currentOffsetY > normal2pullingOffsetY) {
            // 转为即将刷新状态
            self.state = MJRefreshStatePulling;
        } else if (self.state == MJRefreshStatePulling && currentOffsetY <= normal2pullingOffsetY) {
            // 转为普通状态
            self.state = MJRefreshStateIdle;
        }
    } else if (self.state == MJRefreshStatePulling) {// 即将刷新 && 手松开
        // 开始刷新
        [self beginRefreshing];
    } else if (pullingPercent < 1) {
        self.pullingPercent = pullingPercent;
    }
}

- (void)scrollViewContentSizeDidChange:(NSDictionary *)change
{
    [super scrollViewContentSizeDidChange:change];
    // 内容的高度
    CGFloat contentHeight = self.scrollView.mj_contentH + self.ignoredScrollViewContentInsetBottom;
    // 表格的高度
    CGFloat scrollHeight = self.scrollView.mj_h - self.scrollViewOriginalInset.top - self.scrollViewOriginalInset.bottom + self.ignoredScrollViewContentInsetBottom;
    // 设置位置和尺寸
    self.mj_y = MAX(contentHeight, scrollHeight);
}

- (void)setState:(MJRefreshState)state
{
    MJRefreshCheckState
    // 根据状态来设置属性
    if (state == MJRefreshStateNoMoreData || state == MJRefreshStateIdle) {
        // 刷新完毕
        if (MJRefreshStateRefreshing == oldState) {
            [UIView animateWithDuration:MJRefreshSlowAnimationDuration animations:^{
                self.scrollView.mj_insetB -= self.lastBottomDelta;
                // 自动调整透明度
                if (self.isAutomaticallyChangeAlpha) self.alpha = 0.0;
            } completion:^(BOOL finished) {
                self.pullingPercent = 0.0;
                if (self.endRefreshingCompletionBlock) {
                    self.endRefreshingCompletionBlock();
                }
            }];
        }
        CGFloat deltaH = [self heightForContentBreakView];
        // 刚刷新完毕
        if (MJRefreshStateRefreshing == oldState && deltaH > 0 && self.scrollView.mj_totalDataCount != self.lastRefreshCount) {
            self.scrollView.mj_offsetY = self.scrollView.mj_offsetY;
        }
    } else if (state == MJRefreshStateRefreshing) {
        // 记录刷新前的数量
        self.lastRefreshCount = self.scrollView.mj_totalDataCount;
        [UIView animateWithDuration:MJRefreshFastAnimationDuration animations:^{
            CGFloat bottom = self.mj_h + self.scrollViewOriginalInset.bottom;
            CGFloat deltaH = [self heightForContentBreakView];
            if (deltaH < 0) { // 如果内容高度小于view的高度
                bottom -= deltaH;
            }
            self.lastBottomDelta = bottom - self.scrollView.mj_insetB;
            self.scrollView.mj_insetB = bottom;
            self.scrollView.mj_offsetY = [self happenOffsetY] + self.mj_h;
        } completion:^(BOOL finished) {
            [self executeRefreshingCallback];
        }];
    }
}
```
Footer相对于Header来说要相对复杂一些
MJRefreshAutoFooter通过`- (void)scrollViewContentSizeDidChange:(NSDictionary *)change`监听到contentSize改变来改变Footer的y值，以达到自适应位置，通过`- (void)scrollViewContentOffsetDidChange:(NSDictionary *)change`和`- (void)scrollViewPanStateDidChange:(NSDictionary *)change`监听来改变刷新状态，通过`- (void)setState:(MJRefreshState)state`来更新UI，MJRefreshBackFooter也一样，不过不同的是MJRefreshBackFooter的footer的y值最小是scrollView的高度

总结：我们参照MJ实现下拉刷新大概需要以下步骤
1. 自定义一个View
2. 将view加载到scrollView上，并在此时对scrollView的offset、contentSize、panGesture.state进行监听，在移除view时，需要移除监听
3. 通过上面的监听来修改view的位置、动画等自定的内容(这一步也是自定义刷新的难点，然而像这种对UI的操作，如果不能满足项目的需求，我们去阅读对我们的参考价值也不大)