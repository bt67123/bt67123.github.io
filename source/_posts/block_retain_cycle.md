---
title: OC中block循环引用
date: 2016/7/11 12:25:25
---

## 什么是block循环引用

> 如果在Block中使用附有__strong修饰符的对象类型自动变量，那么当Block从栈复制到堆时，该对象为Block所有。这样容易引起循环引用。

举个简单的例子
``` objc
typedef void (^block1) (void);

@interface ViewController () {
    block1 _blk;
}
@end

@implement ViewController 
- (void)viewDidLoad {
    [super viewDidLoad];
    _blk = ^ {
    	NSLog(@"%@", self);
    };
}

- (void)dealloc {
}
@end
```
上面的例子中`ViewController`对象持有`_blk`，`_blk`持有`ViewController`对象，这样就形成了循环引用。
这样会导致`dealloc`无法被调用，也就是说`ViewContrller`对象无法被释放。 

<!--more-->

## 怎么解决循环引用

比较容易想到的是声明有`__weak`修饰符的变量，并将`self`赋值使用。
还是用上面那个例子

```objc
typedef void (^block1) (void);

@interface ViewController () {
    block1 _blk;
}

@implement ViewController 
- (void)viewDidLoad {
    [super viewDidLoad];
    __weak typeof(self) weakSelf = self;
    _blk = ^ {
    	NSLog(@"%@", weakSelf);
    };
}

- (void)dealloc {
}
@end
```

这样就能简单解决循环引用问题了。

但是有时候情况没那么简单，前些天去面试，遇到了一道循环引用的改错题，题目是这样的

```objc
@interface ViewController () {
    id _observer;
}

@implementation ViewController
- (void)viewDidLoad {
    [super viewDidLoad];
     _observer = [[NSNotificationCenter defaultCenter]
                 addObserverForName:@"name"
                 object:nil queue:nil
                 usingBlock:^(NSNotification * _Nonnull note) {
                         NSLog(@"%@", self);
                     });      
                 }];
}

- (void)dealloc {
    NSLog(@"dealloc");
    if (_observer != nil) {
        [[NSNotificationCenter defaultCenter] removeObserver:_observer];
    }
}
@end
```

这么显而易见的错误，不就是block的循环引用吗，`self` -> `_observer` -> `block` -> `self`
说时迟那时快，我已把答案填好了，头也不回交卷了。我当时用的就是上面用`__weak`的解法，我的答案是这样的

```objc
__weak typeof(self) weakSelf = self;
_observer = [[NSNotificationCenter defaultCenter]
            addObserverForName:@"name"
            object:nil queue:nil
            usingBlock:^(NSNotification * _Nonnull note) {
            		NSLog(@"%@", weakSelf);
            	});      
            }];
```




## 对于用__weak解决block引用循环的思考
上面的解决方案看上去没什么问题了，但是面试官看了答案后问，这样改会有什么问题？

what？exm？当时确实短路没能想到有什么问题。后来仔细想了下，循环引用确实解决了，但引入了个问题（我就想到了这个），就是如果程序走到`block`内后，如果还没完成`block`的任务`self`就被释放了，那么`block`里就没办法完成需要用到`self`的任务了。

假如产品需要把`block`内的任务执行完，那么上面的解法确实有点欠缺，反之，前面那种解法没什么问题。那怎么做到等到`block`走完后`self`再释放？

可以这样

```objc
__weak typeof(self) weakSelf = self;
_observer = [[NSNotificationCenter defaultCenter]
            addObserverForName:@"name"
            object:nil queue:nil
            usingBlock:^(NSNotification * _Nonnull note) { 
              		// 声明一个强引用指向self
            		typeof(weakSelf) strongSelf = weakSelf;
            		NSLog(@"%@", strongSelf);
            	});      
            }];
```

这样在`block`中声明一个强引用指向`self`，`self`就不会在`block`执行完之前释放，在此之前起码有一个强引用指向`self`。

这样会导致循环引用吗？
看上去有点像引用循环，`self` -> `_observer` -> `block` -> `strongSelf`
但由于`strongSelf`是`block`的局部变量，存储在栈内存中，所以当`block`执行完之后，超出作用域的`strongSelf`就会被释放，循环引用自然就不存在。


