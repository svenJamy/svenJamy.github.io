---
title: iOS isEqual和hash杂谈
date: 2017-06-22 21:45:03
tags: iOS
categories: iOS
---

在日常开发中我们可能经常接触到比较两个对象是否相等的情况，如果你对下面几个问题比较了解，说明你已经了解其中的技巧了：<!---more--->

* `==`和`isEqual`之间有什么区别？
* `NSString`的两个不同对象内容相同时为什么`isEqual`相同？
*  `NSArray`等类簇怎么比较是否相等？
* `isEqual`和`hash`之间有什么关系，为什么实现了`isEqual`方法的同时也要实现`hash`？

针对于上面的问题，我们来一一解答：


## `==`和`isEqual`
`==`这个很好理解，就是比较两个对象的内存地址是否相等，和其他语言一样，大部分情况下，两个对象的内存地址相同，那么通过`isEqual`比较的时候也相等，但是反过来不然。一般我们实现`isEqual`的思路如下：

```
- (BOOL)isEqual:(id)object {
  if (object == self) {
    return YES;
  }
  
  if (![object isKindOfClass:Person.class]) {
    return NO;
  }
  
  // property compare
  // ....
  
  return YES;
}
```

## 特别的`NSString`
首先我们来看一个demo：

```
NSString *a = @"HelloviewDidLoadviewDidLoadviewDidLoadview";
NSString *b = @"HelloviewDidLoadviewDidLoadviewDidLoadview";
BOOL wtf = (a == b);
NSLog(@"a:%p, b:%p--result:%@",a, b, @(wtf));

// a:0x10ddee068, b:0x10ddee068--result:1(yes)
```

首先我们要明确一点，比较 `NSString` 对象正确的方法是 `-isEqualToString:`。任何情况下都不要直接使用 == 来对 `NSString` 进行比较。

现在来看，到底发生了什么？为什么这样做结果是正确的，同样的代码对于 `NSArray` 和 `NSDictionary` 就不好使？

所有这些行为，都来源于一种称为[字符串驻留](https://en.wikipedia.org/wiki/String_interning)的优化技术，它把一个不可变字符串对象的值拷贝给各个不同的指针。NSString *a 和 *b都指向同样一个驻留字符串值 @"Hello"。 注意所有这些针对的都是静态定义的不可变字符串。


## `NSArray`等比较相等

细心的同学可能发现`NSArray`有两个比较相等的方法：`isEqualToArray:`和`isEqual:`,这里要说明的是：`isEqual:`是`NSObject``protocol`定义的协议，继承自`NSObject`的类都实现了这个方法，`NSArray`也不例外。而`isEqualToArray:`则是`NSArray`实现的一个深层次比较相等的方法，猜测`isEqual:`内部是会调用到`isEqualToArray:`方法的，大致的实现如下：

```
@implementation NSArray (Approximate)
- (BOOL)isEqualToArray:(NSArray *)array {
  if (!array || [self count] != [array count]) {
    return NO;
  }

  for (NSUInteger idx = 0; idx < [array count]; idx++) {
      if (![self[idx] isEqual:array[idx]]) {
          return NO;
      }
  }

  return YES;
}

- (BOOL)isEqual:(id)object {
  if (self == object) {
    return YES;
  }

  if (![object isKindOfClass:[NSArray class]]) {
    return NO;
  }

  return [self isEqualToArray:(NSArray *)object];
}
@end
```

## `isEqual`和`hash`的关系

通常我们实现`isEqual`方法的时候，同样也会实现`hash`方法，但是比较两个对象的时候，发现并不会调用到hash这个方法，为什么还要实现呢？但是当我们在NSDictionary等类簇中使用到这个对象的时候，发现就会调用到这个方法。这是因为 NSSet 和 NSDictionary是基于散列表实现的，通过hash可以实现(O(1))的查找。

对于面向对象编程来说，对象相等性检查的主要用例，就是确定一个对象是不是一个集合的成员。为了加快这个过程，子类当中需要实现 hash 方法：

* 对象相等具有 交换性 （[a isEqual:b] ⇒ [b isEqual:a])；
* 如果两个对象相等，它们的 hash 值也一定是相等的 ([a isEqual:b] ⇒ [a hash] == [b hash])；
* 反过来则不然，两个对象的散列值相等不一定意味着它们就是相等的 ([a hash] == [b hash] ¬⇒ [a isEqual:b])。


## reference

[nshipster.cn](http://nshipster.cn/equality/)

