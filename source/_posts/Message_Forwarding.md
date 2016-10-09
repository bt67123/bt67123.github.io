---
title: Objective-C - 利用动态消息转发实现代理转发
date: 2016/7/25 11:17:30
---

## Runtime ?

Runtime 是 Objective-C 区别于 C 语言这样的静态语言的一个非常重要的特性。对于 C 语言，函数的调用会在编译期就已经决定好，在编译完成后直接顺序执行。但是 OC 是一门动态语言，函数调用变成了消息发送，在编译期不能知道要调用哪个函数。所以 Runtime 无非就是去解决如何在运行时期找到调用方法这样的问题。

## 动态消息转发

由于OC有Runtime这一重要特性，使得动态消息转发(Message Forwarding)成为了可能。

<!--more-->

最近我在试着实现`爱范儿app`的卡片菜单，其中菜单控件是继承`UIScrollView`的，在实现控件过程中，我需要在控件内监听`UIScrollView`是否在滚动。想过用KVO监听`decelerating`，但是无法实时获取。想过用回调方法`scrollViewDidEndDecelerating:`，但是如果我占用了`delegate`，那么外面使用这个控件的 ViewController 就无法再监听 ScrollView 了。当然也可以自己定义一个 myDelegate 什么的去回调`delegate`的方法，但我希望的是外面在使用菜单控件的时候能更方便。

在网上翻查了一下解决方案，发现了使用动态消息解析的野路子，比较接近的就是这个blog

[在 OC 中实现消息的一箭双雕](http://kittenyang.com/forwardinvocation/)

我们先来看看消息转发的流程图

![](http://static.oschina.net/uploads/space/2016/0227/211220_TxEr_580523.jpg)

上图大概分为3步

1. Method resolution (`resolveInstanceMethod:`)
2. Fast forwarding (`forwardingTargetForSelector:`)
3. Normal forwarding (`methodSignatureForSelector:` -> `forwardInvocation:`)

当程序在运行时没找到某个方法，在抛出unrecognized selector sent to的异常之前，会先经过上面的这些方法，换言之，OC 的运行时会给你三次拯救程序的机会。

参考 blog 就是使用 Normal forwarding 这一步进行动态消息解析转发的。

我尝试了一下这个方法，奏效，代码如下
``` objc
// LCScrollViewDelegateContainer.h
@interface LCScrollViewDelegateContainer : NSObject
@property (nonatomic, assign) id<UIScrollViewDelegate> firstDelegate;
@property (nonatomic, assign) id<UIScrollViewDelegate> secondDelegate;
@end

// LCScrollViewDelegateContainer.m
// 转发器
@implementation LCScrollViewDelegateContainer

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    SEL aSelector = [anInvocation selector];
    // 以下起到代理分发效果
    // 如果firstDelegate实现了代理方法，则调用
    if([self.firstDelegate respondsToSelector:aSelector]){
        [anInvocation invokeWithTarget:self.firstDelegate];
    }
    // 如果secondDelegate实现了代理方法，则调用
    if([self.secondDelegate respondsToSelector:aSelector]){
        [anInvocation invokeWithTarget:self.secondDelegate];
    }
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    NSMethodSignature *first = [(NSObject *)self.firstDelegate methodSignatureForSelector:aSelector];
    NSMethodSignature *second = [(NSObject *)self.secondDelegate methodSignatureForSelector:aSelector];
    if(first){
        return first;
    } else if(second) {
        return second;
    }
    return nil;
}

- (BOOL)respondsToSelector:(SEL)aSelector{
    if([self.firstDelegate respondsToSelector:aSelector] || 
       [self.secondDelegate respondsToSelector:aSelector]){
        return YES;
    } else {
        return NO;
    }
}

@end
```

使用

``` objc
// ViewController.m
@interface ViewController () <UIScrollViewDelegate>
@property (nonatomic, strong) LCScrollView *scrollView;
@property (nonatomic, strong) LCScrollViewDelegateContainer *scrollViewDelegateContainer;
@end

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
    self.scrollView = [[LCScrollView alloc] initWithChildViews:views];
    self.scrollViewDelegateContainer = [[LCScrollViewDelegateContainer alloc] init];

    _scrollViewDelegateContainer.firstDelegate = _scrollView;
    _scrollViewDelegateContainer.secondDelegate = self;

    _scrollView.delegate = (id)_scrollViewDelegateContainer;
    [self.view addSubview:_scrollView];
}

#pragma mark - UIScrollViewDelegate
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    NSLog(@"view controller - scrollViewDidScroll");
}
@end
```

``` objc
// LCScrollView.m

...
...

#pragma mark - UIScrollViewDelegate
- (void)scrollViewDidScroll:(UIScrollView *)scrollView {
    NSLog(@"scroll View - scrollViewDidScroll");
}
```

滚动 ScrollView 打印：

```
...
2016-07-25 12:18:49.868 ifanr_menu_demo[877:438227] scroll View - scrollViewDidScroll
2016-07-25 12:18:49.868 ifanr_menu_demo[877:438227] view controller - scrollViewDidScroll
2016-07-25 12:18:49.886 ifanr_menu_demo[877:438227] scroll View - scrollViewDidScroll
2016-07-25 12:18:49.886 ifanr_menu_demo[877:438227] view controller - scrollViewDidScroll
...
```

成功分发。

像上面的代码`scrollView`在尝试找`delegate`中的`scrollViewDidScroll:`方法时，会调用`respondsToSelector:`方法询问能否响应，由于实现代理的是`LCScrollViewDelegateContainer`中的`firstDelegate`或`secondDelegate`，所以需要重写它的`respondsToSelector:`方法，否则直接跳出无法进行下一步。

虽然`respondsToSelector:`会返回`true`，但是实际上这个转发器并没有实现代理的方法啊！所以程序就会进入消息转发，希望能抓到什么方法执行不至于崩溃。

所以上面程序顺序大概这样

调用`scrollViewDidScroll:` -> 判断能否响应`respondsToSelector:` -> 获取方法签名`methodSignatureForSelector:` -> 改变调用目标`forwardInvocation:`

## 小坑
这里有一个不小心就会犯的错误 - **循环引用**

如果想用`NSArray`等容器去得到代理集合，并使用动态消息解析分发到每个代理的话，那么`NSArray`，`NSSet`等不能添加弱引用对象的容器就很容易造成引用循环，可以使用`NSPointerArray`，`NSHashTable`等可以添加弱引用对象的容器。

以上。
