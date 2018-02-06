---
title: ios-launch-time
date: 2017-08-28 12:30:40
categories:
- iOS
tags: iOS
---

## 程序和进程
广义上的程序就是一个静态的可执行文件，是由一个已经编译好的指令和数据集合的一个文件。就像是我们通过`Xcode`编译好的`macho`文件。而进程则是一个动态的概念，是程序的运行时的一个过程<!-- more -->。

## 虚拟地址空间
每个进程运行的时候都有自己独立的虚拟地址空间，这个空间的大小是由计算机的硬件决定的，比如在32位硬件平台上，它的寻址空间大小是2^32 - 1，现在iPhone都是64位的，寻址空间为2^64-1 。

## 冷启动和热启动
热启动是由于某种原因，APP的状态由`running `切换为`suspend `，但是此时APP并没有被系统kill掉，当我们再次把APP切换到前台的时候，APP会恢复之前的状态继续运行，这种就是热启动，我们平时所说的APP在后台的存活时间，其实就是APP能执行热启动的最大时间间隔。而冷启动则是APP从被加载到内存到运行的状态，下面我们要讲的主要是冷启动。

## 孤独的`main`函数
大概是从我们学习编程开始就知道`main`函数是程序的入口，但是真的是这样吗？在平时的面试过程中我也有问一些面试者这个问题，但是回答的都比较模糊。其实我们通过代码可以看出，在iOS里面 `main`只是简单的返回一个`UIApplicationMain `对象，里面的有一个重要的参数就是实现了`UIApplicationDelegate`代理的类。
```
// UIKIT_EXTERN int UIApplicationMain(int argc, char * _Nonnull * _Null_unspecified argv, NSString * _Nullable principalClassName, NSString * _Nullable delegateClassName);

int main(int argc, char *argv[])
{
@autoreleasepool {
return UIApplicationMain(argc, argv, nil, NSStringFromClass([UIAppDelegate class]));
}
}
```
APP启动流程时间主要包括两部分，`main`函数之前和`main`函数执行之后到`-(BOOL)Application:(UIApplication *)Application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions`方法执行完成。其中`main`函数执行之后优化主要是让上面的方法尽快执行完，不要有什么block主线程的操作。所以我们可以看出，其实在`main`里面处理的事情还是比较简单的。最重要的还是在`main`函数执行之前。

## 概述
从WWDC的视频我们可以得出简单的结论：系统先读取App的可执行文件，从里面获得dyld的路径，然后加载dyld，当所有依赖库的初始化后，轮到最后一位(程序可执行文件)进行初始化，在这时runtime会对项目中所有类进行类结构初始化，然后调用所有的load方法。最后dyld返回main函数地址，main函数被调用，我们便来到了熟悉的程序入口。

## 启动时间
在Xcode中可以通过设置`DYLD_PRINT_STATISTICS `环境变量来查看APP的启动时间详细信息：
![statistics.png](http://upload-images.jianshu.io/upload_images/1743782-0d75d324d0637fae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后就可以在控制台看到如下信息：
```
Total pre-main time: 282.69 milliseconds (100.0%)
dylib loading time: 107.37 milliseconds (37.9%)
rebase/binding time:  44.92 milliseconds (15.8%)
ObjC setup time:  64.72 milliseconds (22.8%)
initializer time:  65.56 milliseconds (23.1%)
slowest intializers :
libSystem.dylib :   7.98 milliseconds (2.8%)
libMainThreadChecker.dylib :  23.55 milliseconds (8.3%)
AFNetworking :  19.46 milliseconds (6.8%)
```
从上面可以看出时间区域主要分为下面几个部分：
* dylib loading time
* rebase/binding time
* ObjC setup time
* initializer time

## dyld
(the dynamic link editor)动态链接器，是一个专门用来加载动态链接库的库，它是开源的，源码在[这里](https://github.com/opensource-apple/dyld)。在 xnu 内核为程序启动做好准备后，执行由内核态切换到用户态，由dyld完成后面的加载工作，dyld的主要是初始化运行环境，开启缓存策略，加载程序依赖的动态库(其中也包含我们的可执行文件)，并对这些库进行链接（主要是rebaseing和binding），最后调用每个依赖库的初始化方法，在这一步，runtime被初始化。

![obj_init.png](http://upload-images.jianshu.io/upload_images/1743782-98f085db2ddd88ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`ImageLoader `是用于加载可执行文件格式的类，程序中对应实例可简称为image(如程序可执行文件macho，Framework，bundle等)。

## Rebasing 和 Binding
ASLR（Address Space Layout Randomization），[地址空间布局随机化](https://en.wikipedia.org/wiki/Address_space_layout_randomization)。在ASLR技术出现之前，程序都是在固定的地址加载的，这样hacker可以知道程序里面某个函数的具体地址，植入某些恶意代码，修改函数的地址等，带来了很多的危险性。ASLR就是为了解决这个的，程序每次启动后地址都会随机变化，这样程序里所有的代码地址都需要需要重新对进行计算修复才能正常访问。rebasing这一步主要就是调整镜像内部指针的指向。

Binding：将指针指向镜像外部的内容。

## ObjC setup
上面最后一步调用的`objc_init`方法，这个事runtime的初始化方法，在这个方法里面主要的操作就是加载类：
```
/***********************************************************************
* _objc_init
* Bootstrap initialization. Registers our image notifier with dyld.
* Called by libSystem BEFORE library initialization time
**********************************************************************/

void _objc_init(void)
{
static bool initialized = false;
if (initialized) return;
initialized = true;

// fixme defer initialization until an objc-using image is found?
environ_init();
tls_init();
static_init();
lock_init();
exception_init();

_dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
```
` _dyld_objc_notify_register(&map_images, load_images, unmap_image);`向dyld注册了一个通知事件，当有新的image加载到内存的时候，就会触发`load_images `方法，这个方法里面就是加载对应image里面的类，并调用`load`方法。
```
load_images(const char *path __unused, const struct mach_header *mh)
{
// Return without taking locks if there are no +load methods here.
if (!hasLoadMethods((const headerType *)mh)) return;

recursive_mutex_locker_t lock(loadMethodLock);

// Discover load methods
{
rwlock_writer_t lock2(runtimeLock);
prepare_load_methods((const headerType *)mh);
}

// Call +load methods (without runtimeLock - re-entrant)
call_load_methods();
}

/***********************************************************************
* call_load_methods
* Call all pending class and category +load methods.
* Class +load methods are called superclass-first.
* Category +load methods are not called until after the parent class's +load.
*
* This method must be RE-ENTRANT, because a +load could trigger
* more image mapping. In addition, the superclass-first ordering
* must be preserved in the face of re-entrant calls. Therefore,
* only the OUTERMOST call of this function will do anything, and
* that call will handle all loadable classes, even those generated
* while it was running.
*
* The sequence below preserves +load ordering in the face of
* image loading during a +load, and make sure that no
* +load method is forgotten because it was added during
* a +load call.
* Sequence:
* 1. Repeatedly call class +loads until there aren't any more
* 2. Call category +loads ONCE.
* 3. Run more +loads if:
*    (a) there are more classes to load, OR
*    (b) there are some potential category +loads that have
*        still never been attempted.
* Category +loads are only run once to ensure "parent class first"
* ordering, even if a category +load triggers a new loadable class
* and a new loadable category attached to that class.
*
* Locking: loadMethodLock must be held by the caller
*   All other locks must not be held.
**********************************************************************/
void call_load_methods(void)
{
static bool loading = NO;
bool more_categories;

loadMethodLock.assertLocked();

// Re-entrant calls do nothing; the outermost call will finish the job.
if (loading) return;
loading = YES;

void *pool = objc_autoreleasePoolPush();

do {
// 1. Repeatedly call class +loads until there aren't any more
while (loadable_classes_used > 0) {
call_class_loads();
}

// 2. Call category +loads ONCE
more_categories = call_category_loads();

// 3. Run more +loads if there are classes OR more untried categories
} while (loadable_classes_used > 0  ||  more_categories);

objc_autoreleasePoolPop(pool);

loading = NO;
}
```
如果有继承的类，那么会先调用父类的`load`方法，然后调用子类的，但是在`load`里面不能调用`[super load]`。最后才是调用category的`load`方法。所以在这一步，所有的`load`都会被调用到。

## C++ initializer
在这一步，如果我们代码里面使用了clang的`__attribute__((constructor))`构造方法，都会调用到。

## 优化点

那么如何尽可能的减少pre-main花费的时间呢,主要就从上面给出的几个阶段下手:

* 动态库加载的时间优化。每个App都进行动态库加载,其中系统级别的动态库占据了绝大数,而针对系统级别的动态库都是经过系统高度优化的,不用担心时间的花费。开发者应该关注于自己集成到App的那些动态库,这也是最能消耗加载时间的地方。对此Apple建议减少在App里开发者的动态库集成或者有可能地将其多个动态库最终集成一个动态库后进行导入, 尽量保证将App现有的非系统级的动态库个数保证在6个以内；

* (Rebase/binding)时间优化。减少App的Objective-C类,分类和Selector的个数。这样做主要是为了加快程序的整个动态链接, 在进行动态库的重定位和绑定(Rebase/binding)过程中减少指针修正的使用,加快程序机器码的生成；

* objc init 优化。用+initialize方法替换+load方法,从而加快所有类文件的加载速度。

# refrence

* [WWDC2016 section 406](https://developer.apple.com/videos/play/wwdc2016/406/)

* [dyld](https://github.com/opensource-apple/dyld)
