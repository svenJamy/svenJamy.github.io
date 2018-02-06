---
title: NSProxy在iOS开发中的妙用
date: 2018-01-17 23:10:22
tags: iOS
categories: iOS
---

`NSObject`是OC中大多数类的基类，它定义了对象的一些基本方法，如对象的创建，消息的发送和转发等等。但是在iOS开发中还有另外一个和`NSObject`同级的类，它就是`NSProxy`,它们都是`Foundation`框架中的基类, 并且都实现了<NSObject>协议。<!--more-->

根据苹果的文档我们可以知道，`NSproxy`天生就是为消息转发而存在的，但是它并没有init方法，默认也米有实现copy方法。任何发送到`NSproxy`的消息都会被转发到真正的对象上，前提是要实现`- (void)forwardInvocation:(NSInvocation *)invocation;` 和 `
-(nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel` 这2个方法，关于这2个方法具体的使用大家可以自行Google。

## 多重代理
在平时的开发中我们可能会遇到这样的情况：在多个对象中都需要监听某个`delegate`方法，比如`UIScrollViewDelegate`，几个地方都需要用到`scrollviewDidScroll:`方法，比较传统的解决方案是在实现`UIScrollViewDelegate`的对象里面，将消息转发到其他对象上，但是这样会带来很多问题，代码的可维护性会很低。这里用`NSProxy`就很好解决：

```
@interface LGMutilDelegatesProxy ()
@property (nonatomic,strong) NSPointerArray *delegatesPointerArray;
@end

@implementation LGMutilDelegatesProxy
- (id)init {
  return self;
}
- (instancetype)initWithDelegateTagets:(NSArray *)Tagets {
  if (self) {
    self.delegatesPointerArray = [NSPointerArray weakObjectsPointerArray];
    [Tagets enumerateObjectsUsingBlock:^(id  _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
      [self.delegatesPointerArray addPointer:(void *)obj];
    }];
  }
  return self;
}
#pragma mark - forward
- (BOOL)respondsToSelector:(SEL)aSelector {
  for (id delegateObj in self.delegatesPointerArray.allObjects) {
    if ([delegateObj respondsToSelector:aSelector]) {
      return YES;
    }
  }
  return NO;
}
- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
  for (id delegateObj in self.delegatesPointerArray.allObjects) {
    if ([delegateObj respondsToSelector:sel]) {
      return [[delegateObj class] instanceMethodSignatureForSelector:sel];
    }
  }
  return [super methodSignatureForSelector:sel];
}
- (void)forwardInvocation:(NSInvocation *)invocation {
  for (id delegateObj in self.delegatesPointerArray.allObjects) {
    if ([delegateObj respondsToSelector:invocation.selector]) {
      [invocation invokeWithTarget:delegateObj];
    }
  }
}

@end

```
使用方法也很简单：

```
LGMutilDelegatesProxy *proxy = [LGMutilDelegatesProxy alloc] initWithDelegateTagets:@[ obj1, obj2, obj3... ]];
scrollView.delegate = (id<UIScrollViewDelegate>)proxy;
```
这样数组里面的类都能接收到`UIScrollViewDelegate`的消息了。

## 消息截断

开发中我们可能会遇到这样的情况，需要把特定的方法转发到另外一个对象里面，一般的做法就是判断对象是否实现了对应的方法，然后调用对象的方法。如果需要转发的方法的个数比较少这样做是可以的。但是比如类似`UICollectionViewDelegate`的方法，它是继承自`UITableViewDelegate`,我们需要把`UITableViewDelegate`的方法转发到其他的对象里。这种情况下可以使用`NSProxy`,`IGListKit`里面也有类似的实现：

```

static BOOL isInterceptedSelector(SEL sel) {
    return (
            sel == @selector(scrollViewDidScroll:) ||
            sel == @selector(scrollViewWillBeginDragging:) ||
            sel == @selector(scrollViewDidEndDragging:willDecelerate:)
            );
}

@implementation LGListAdapterProxy

- (instancetype)initWithCollectionViewTarget:(nullable id<UICollectionViewDelegate>)collectionViewTarget
                            scrollViewTarget:(nullable id<UIScrollViewDelegate>)scrollViewTarget
                                 interceptor:(IGListAdapter *)interceptor {
    if (self) {
        _collectionViewTarget = collectionViewTarget;
        _scrollViewTarget = scrollViewTarget;
        _interceptor = interceptor;
    }
    return self;
}

- (BOOL)respondsToSelector:(SEL)aSelector {
    return isInterceptedSelector(aSelector)
    || [_collectionViewTarget respondsToSelector:aSelector]
    || [_scrollViewTarget respondsToSelector:aSelector];
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (isInterceptedSelector(aSelector)) {
        return _interceptor;
    }

    // since UICollectionViewDelegate is a superset of UIScrollViewDelegate, first check if the method exists in
    // _scrollViewTarget, otherwise use the _collectionViewTarget
    return [_scrollViewTarget respondsToSelector:aSelector] ? _scrollViewTarget : _collectionViewTarget;
}

@end

```

## 循环引用

`NSTimer`可以实现定时任务，比如通过`timerWithTimeInterval:target:`方法，一般把target设置为视图或者VC，但是这样会出现循环引用：Timer->Target->Timer.即使在dealloc方法中写 `[_timer invalidate];`方法也是无效的，因为根本就执行不到这个方法。基于上面的分析，由于循环引用的存在，控制器永远也不会走dealloc方法，定时器会一直执行方法，造成内存泄露。当然解决的方法有很多种，这里也可以用`NSProxy`来解决：

```
@implementation LGWeakProxy

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sig = nil;
    sig = [self.obj methodSignatureForSelector:aSelector];
    return sig;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    [anInvocation invokeWithTarget:self.obj];
}

@end

self.timer = [NSTimer timerWithTimeInterval:1 target:[[LGWeakProxy alloc] init] selector:@selector(timerEvent) userInfo:nil repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];

```



