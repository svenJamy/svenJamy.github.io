---
title: autoreleasepool
date: 2018-01-08 22:29:42
tags:
---

## 简介
AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制。像在C++等语言中，手动malloc的对象是要通过free去释放的，如果没有释放，就会产生内存泄漏，在OC里面是通过ARC方式去管理内存的。AutoreleasePool提供了这样一种机制，它可以延迟对象的释放时机，加入到page中的对象是由AutoreleasePool统一释放的。

在公司平时的面试中，也会问面试者这个问题，`Autorelease`对象是什么时候被释放的？大多数人都没有回答上来，虽然不能以这个来衡量面试者的标准，但是了解`Autorelease`内部的实现，对于一个高级程序员来说是很有必要的。

## runloop 和AutoreleasePool
App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。这里我们可以添加这个symbol break point查看：

![runloop.png](http://upload-images.jianshu.io/upload_images/1743782-cdcda6fe5ae43313.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/600)

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

## Thread Local Storage（TLS）
线程局部存储，就是将一块内存作为某个线程专有的存储，以key-value的形式进行读写。🍎源码里面分为2中实现方式：arm和非arm，arm架构下实现是通过`tls_base（）`方法直接进行储存的。在非arm架构下是通过`pthread_getspecific`和`pthread_setspecific`函数实现的。

## 源码分析
先看下🍎的官方介绍：
```
/***********************************************************************
Autorelease pool implementation

A thread's autorelease pool is a stack of pointers.
Each pointer is either an object to release, or POOL_BOUNDARY which is
an autorelease pool boundary.
A pool token is a pointer to the POOL_BOUNDARY for that pool. When
the pool is popped, every object hotter than the sentinel is released.
The stack is divided into a doubly-linked list of pages. Pages are added
and deleted as necessary.
Thread-local storage points to the hot page, where newly autoreleased
objects are stored.

class AutoreleasePoolPage
{
static pthread_key_t const key = AUTORELEASE_POOL_KEY;
static uint8_t const SCRIBBLE = 0xA3;  // 0xA3A3A3A3 after releasing
static size_t const SIZE =
#if PROTECT_AUTORELEASEPOOL
PAGE_MAX_SIZE;  // must be multiple of vm page size
#else
PAGE_MAX_SIZE;  // size and alignment, power of 2
#endif
static size_t const COUNT = SIZE / sizeof(id);
// 用来检验结构是否完整
magic_t const magic;
// 栈顶指针
id *next;
// 当前线程
pthread_t const thread;
// 当前子节点的父节点
AutoreleasePoolPage * const parent;
// 子节点
AutoreleasePoolPage *child;
// 链表深度，也就是链表节点的个数
uint32_t const depth;
//  记录pool中最多存放对象的个数
uint32_t hiwat;
...
}
**********************************************************************/
```
从上面的说明中大概可以知道如下的结论：
* AutoreleasePool是线程相关的，不能跨线程使用，它实际上是一个指针栈，是一个双向链表结构，可以根据需要进行add和delete操作；
* 里面存放的是需要release的对象指针或者`POOL_BOUNDARY `（哨兵对象，相当于一个标志位）；
* 当autoreleasepool执行pop操作的时候，在哨兵对象后入栈的对象都将被释放；
* TSL存放的是当前正在使用的page，这里指的是`hot page`。

下面我们从主要的两个函数：
```
void *
objc_autoreleasePoolPush(void)
{
return AutoreleasePoolPage::push();
}

void
objc_autoreleasePoolPop(void *ctxt)
{
AutoreleasePoolPage::pop(ctxt);
}
```
分析AutoreleasePool的大致实现。

## push
入栈操作：
```
static inline void *push()
{
id *dest;
if (DebugPoolAllocation) {
// Each autorelease pool starts on a new pool page.
dest = autoreleaseNewPage(POOL_BOUNDARY);
} else {
dest = autoreleaseFast(POOL_BOUNDARY);
}
assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
return dest;
}
```
调用`autoreleaseFast `函数，参数是哨兵对象：
```
static inline id *autoreleaseFast(id obj)
{
AutoreleasePoolPage *page = hotPage();
if (page && !page->full()) {
return page->add(obj);
} else if (page) {
return autoreleaseFullPage(obj, page);
} else {
return autoreleaseNoPage(obj);
}
}
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
// The hot page is full.
// Step to the next non-full page, adding a new page if necessary.
// Then add the object to that page.
assert(page == hotPage());
assert(page->full()  ||  DebugPoolAllocation);

do {
if (page->child) page = page->child;
else page = new AutoreleasePoolPage(page);
} while (page->full());

setHotPage(page);
return page->add(obj);
}
```
`hotPage `返回的是当前正在使用的page，存放在TSL里面，上面有讲过，如果当前page没有超出最大对象限制，那边会把哨兵对象加入到栈顶，用来标识这是一次push操作的开始。如果当前的page满的话，那么会先在page的子节点查找没有满的page，最后如果没有找到会新建一个AutoreleasePoolPage对象，并设置为hotpage。

## pop
代码比较长，不是有意要贴这么多/(ㄒoㄒ)/~~
```
static inline void pop(void *token)
{
AutoreleasePoolPage *page;
id *stop;
// 1
if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
// Popping the top-level placeholder pool.
if (hotPage()) {
// Pool was used. Pop its contents normally.
// Pool pages remain allocated for re-use as usual.
pop(coldPage()->begin());
} else {
// Pool was never used. Clear the placeholder.
setHotPage(nil);
}
return;
}
// 2
page = pageForPointer(token);
stop = (id *)token;
if (*stop != POOL_BOUNDARY) {
if (stop == page->begin()  &&  !page->parent) {
// Start of coldest page may correctly not be POOL_BOUNDARY:
// 1. top-level pool is popped, leaving the cold page in place
// 2. an object is autoreleased with no pool
} else {
// Error. For bincompat purposes this is not
// fatal in executables built with old SDKs.
return badPop(token);
}
}

if (PrintPoolHiwat) printHiwat();

page->releaseUntil(stop);

// memory: delete empty children
if (DebugPoolAllocation  &&  page->empty()) {
// special case: delete everything during page-per-pool debugging
AutoreleasePoolPage *parent = page->parent;
page->kill();
setHotPage(parent);
} else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
// special case: delete everything for pop(top)
// when debugging missing autorelease pools
page->kill();
setHotPage(nil);
}
else if (page->child) {
// hysteresis: keep one empty child if page is more than half full
if (page->lessThanHalfFull()) {
page->child->kill();
}
else if (page->child->child) {
page->child->child->kill();
}
}
}
```
从上面可以看出pop分为2种：第一种是pop`EMPTY_POOL_PLACEHOLDER`类型的page，`EMPTY_POOL_PLACEHOLDER `标识的page存放在TSL，虽然page被入栈了，但是里面没有任何对象。
第二种是释放通用的对象，之前有讲过哨兵对象，它是入栈操作的标志，那么同样一次pop操作将会把入栈操作之后所有的对象都释放掉。

每次pop操作之后都会通过`printHiwat()`方法重新计算`hiwat`。


## refrence

https://blog.ibireme.com/2015/05/18/runloop/


