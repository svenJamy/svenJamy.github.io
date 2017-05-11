---
title: iOS Run Loop
date: 2017-05-10 19:00:00
tags: iOS
---

## Run Loop概念
runloop正如其名，运行时循环，和Windows的消息循环类似，用于事件的调度和分发<!-- more -->。RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行下面👇图1的步骤。线程执行了这个函数后，就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。
![runloop循环](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Art/runloop.jpg)

从上图我们可以看出，runloop的事件源有两种：

* input source：提供异步事件，通常来源于其他线程或者其他application。处理该事件时会调用`runUntilDate:`方法，处理完之后runloop退出。
* Timer sources：提供同步事件，通常来源于时间触发器。处理完该事件不会触发runloop退出。

## Run Loop和线程的关系

很多人都说run loop和线程是一一对应的关系，其实这种说法是不准确的。苹果不允许我们直接创建runloop，只提供了两个自动获取的函数：CFRunLoopGetMain() 和 CFRunLoopGetCurrent()，下面我们看看内部大致的实现：

```objc
static CFMutableDictionaryRef __CFRunLoops = NULL;
static CFLock_t loopsLock = CFLockInit;

// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) { // 空代表主线程
	t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) { // 第一次进入runloop
        __CFUnlock(&loopsLock);
        // 创建一个存放runloop和线程关系的字典
	CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
	// 创建主线程runloop
	CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
	// 设置runloop<-->mainthread映射
	CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
	if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
	    CFRelease(dict);
	}
	CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    // 不是第一次进来，先从全局字典里面获取
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    if (!loop) { // 全局字典里面没有，则去创建一个runloop，这个是非主线程才会进来⚠️
	CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
	loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
	if (!loop) {
	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
	    loop = newLoop;
	}
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```

主线程在创建的时候会主动的创建runloop，所以程序启动后就能处理各种事件，不然就直接exit了~ run loop里面保存了一个全局的字典，用来储存run loop和线程的映射关系。但是如果在非主线程不主动的获取它的话是不会有的。所以我认为应该是如下的关系：有runloop一定有一个线程和它对应，但是有线程（非主线程）的时候不一定有runloop。当一个线程结束的时候，runloop会自动的释放。

## Run Loop modes

我们可以把runloop mode看做是监控input source和timer source以及run loop observer的集合。每次我们创建run loop 的时候都会指定一种mode，只有加入到对应mode的事件或者observer才能得到对应的处理，举个粟子：我们把一个定时器事件加入到`default mode`的时候，假如这时候滑动列表控件，这时候我们会发现定时器停止了，当滑动结束后定时器又恢复运行。这个例子其实就和run loop mode的切换有关，这里我们会详细分析，先看看Run Loop Model的数据结构：

```objc
typedef struct __CFRunLoopMode *CFRunLoopModeRef;
struct __CFRunLoopMode {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;	/* must have the run loop locked before locking this */
    CFStringRef _name;
    Boolean _stopped;
    char _padding[3];
    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
    CFMutableDictionaryRef _portToV1SourceMap;
    __CFPortSet _portSet;
    CFIndex _observerMask;
    uint64_t _timerSoftDeadline; /* TSR */
    uint64_t _timerHardDeadline; /* TSR */
};
```

里面包含了source0，source1，timer，observer的数组，也就是说一个run loop mode可以对应多个source事件和time、observer事件。
run loop一共有5中mode，iOS开发中常用的有下面3中：

* `NSDefaultRunLoopMode(kCFRunLoopDefaultMode)`:默认模式。
* `NSEventTrackingRunLoopMode`:列表滑动时切换的模式 event track。
*  `NSRunLoopCommonModes`:在cocoa中是default，modal，event track模式的集合，在CF中只有default模式。在iOS中是default和event track的集合。

在前面的例子中，滑动列表的时候切换到track mode，定时器没有加入到这个模式里面，所以它停止了，滑动结束后切换到default。解决方法是创建定时器的时候把timer加入的common model里面。

## Input Source

input source异步分发事件到对应的线程。事件的来源取决已输入源的类型，主要有2中类型：

* souce0事件：只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
* source1事件：基于mach Port的事件，包含了回调指针，会唤醒run loop去处理该事件。

他们之间的主要区别是source1事件是由内核自动发出的signal，可以唤醒run loop处理事件，source0必须是由其他线程手动控制（标记）。

### Port-Based Sources（source1）

cocoa和CF内建支持使用相关的对象和方法创建port-based source。我们不能直接创建这种source，可以使用`NSPort`或者`NSMachPort`类创建一个port，然后把port加入到run loop中就可以了。

举个粟子，在AFN2.6版本有个常驻线程接收网络请求回调消息，或者现在的RN，WEEX都有类似的实现，WEEX内部有一个常驻线程，用来处理组件的异步事件，RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 WEEX 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了：

```objc
+ (NSThread *)componentThread
{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        WXComponentThread = [[NSThread alloc] initWithTarget:[self sharedManager] selector:@selector(_runLoopThread) object:nil];
        [WXComponentThread start];
    });
    
    return WXComponentThread;
}

- (void)_runLoopThread
{
    [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    
    while (!_stopRunning) {
        @autoreleasepool {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
    }
}
```

### Custom Input Sources（source0）

苹果提供了几个方法支持创建自定义的source事件,这里就直接看例子吧：

```objc
static void sourceCallBack(void *info) {
 NSLog(@"source signal");
}

+ (void)load {
 CFRunLoopSourceContext  *sourceContext = calloc(1, sizeof(CFRunLoopSourceContext));
  sourceContext->perform = &sourceCallBack;
  
   _runLoopSource = CFRunLoopSourceCreate(NULL, 0, sourceContext);
  CFRunLoopAddSource(runLoop, _runLoopSource, kCFRunLoopCommonModes);
}

+ (void)signalSource {
  CFRunLoopSourceSignal(_runLoopSource);
}

```
### Cocoa Perform Selector Sources

除了以上两种在run loop中经常看到的source事件，cocoa定义了一个自定义的输入源事件允许我们在其他的线程执行一个selector。这种source会在执行完selector之后自动的移除。下面列出了`NSObject`类支持的performs方法[官方文档](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html):

| Methods       | Description    |
|:------------- |:---------------|
| `performSelectorOnMainThread:withObject:waitUntilDone:` `performSelectorOnMainThread:withObject:waitUntilDone:modes:`     | 在主线程下一个run loop cycle执行一个指定的selector |
| `performSelector:onThread:withObject:waitUntilDone:` `performSelector:onThread:withObject:waitUntilDone:modes:`      | 在指定的线程执行selector        |
| `performSelector:withObject:afterDelay:` `performSelector:withObject:afterDelay:inModes:` | 在主线程下一个run loop cycle执行一个指定的selector，并且可以指定延迟时间。        |
`cancelPreviousPerformRequestsWithTarget:` `cancelPreviousPerformRequestsWithTarget:selector:object:` | 取消一个发送到当前线程的消息 |

## Observers

在上面我们有提到，每种mode中可以有很多的observer。observer就是监听回调，我们可以监听run loop的几个特定的节点，根据苹果在文档里的说明，RunLoop 内部的逻辑大致如下（借用下YY大神的博文图片）：
![](http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png)

在CF头文件中我们可以看到如下定义：

```objc

/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),
    kCFRunLoopBeforeTimers = (1UL << 1),
    kCFRunLoopBeforeSources = (1UL << 2),
    kCFRunLoopBeforeWaiting = (1UL << 5),
    kCFRunLoopAfterWaiting = (1UL << 6),
    kCFRunLoopExit = (1UL << 7),
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};

struct __CFRunLoopObserver {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;
    CFRunLoopRef _runLoop;
    CFIndex _rlCount;
    CFOptionFlags _activities;		/* immutable */
    CFIndex _order;			/* immutable */
    CFRunLoopObserverCallBack _callout;	/* immutable */
    CFRunLoopObserverContext _context;	/* immutable, except invalidation */
};

```

这里我们可以根据上面定义结合apple提供的几个方法创建observer，在Run loop源码中我们可以看到苹果的Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。这个 Observer 的优先级是 2000000，低于常见的其他 Observer。

当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 捕获，并通过 CATransaction 提交到一个中间状态去（CATransaction 的文档略有提到这些内容，但并不完整）。

当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 CA 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，CA 会通过 DisplayLink 等机制多次触发相关流程。

又比如我们可以在测试阶段添加一个observer，监测run loop各个阶段，判断出一个循环的耗时时间：

```objc
static void addObserverWithOrder(CFIndex order, CFRunLoopActivity activities) {
  CFRunLoopObserverContext *context = NULL;
  if (order < 0) {
    static BOOL isBegin = YES;
    context = &((CFRunLoopObserverContext){0, &isBegin, NULL, NULL, NULL});
  }
  CFRunLoopObserverRef observer = CFRunLoopObserverCreate(kCFAllocatorDefault, activities, YES,order,
&RunLoopObserverCallBack,context);
  CFRunLoopAddObserver(CFRunLoopGetMain(), observer, kCFRunLoopCommonModes);
}

static addObserver() {
	addObserverWithOrder(-0xffffff, kCFRunLoopAllActivities);
	addObserverWithOrder(0xffffff , kCFRunLoopAllActivities);
}

```

## AutoreleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop()和_objc_autoreleasePoolPush()释放旧的池并创建新池；Exit(即将退出Loop)时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

之前有问过几个面试者，在一个方法里面创建一个局部对象，这个对象是在什么时候被释放的？大多数回答都是在{}结束之前就释放了，因为是ARC管理内存的，其实这说的很肤浅，这个和上面说的run loop有关：

在一次run loop循环里面创建的局部对象都会注册到autorelease pool中，当run loop循环结束时，调用`_objc_autoreleasePoolPop` 方法释放这些对象。

## 源码分析

大致的看了下runloop的内部实现，具体[源码](https://opensource.apple.com/tarballs/CF/)可以看这里。

```objc
void CFRunLoopRun(void) {	/* DOES CALLOUT */
    int32_t result;
    do {
        result = CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
        CHECK_FOR_FORK();
    } while (kCFRunLoopRunStopped != result && kCFRunLoopRunFinished != result);
}
```
其实就是一个while无限循环，当runloop返回`kCFRunLoopRunStopped `或者`kCFRunLoopRunFinished `时退出循环；

```
SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    // 如果runloop释放直接返回
    if (__CFRunLoopIsDeallocating(rl)) return kCFRunLoopRunFinished;
    __CFRunLoopLock(rl);
    // 根据参数modename查找对应的mode
    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(rl, modeName, false);
    // 如果当前loop没有这个mode或者mode里没有source/timer/observer, 直接返回
    if (NULL == currentMode || __CFRunLoopModeIsEmpty(rl, currentMode, rl->_currentMode)) {
	Boolean did = false;
	if (currentMode) __CFRunLoopModeUnlock(currentMode);
	__CFRunLoopUnlock(rl);
	return did ? kCFRunLoopRunHandledSource : kCFRunLoopRunFinished;
    }
    // 储存当前loop信息
    volatile _per_run_data *previousPerRun = __CFRunLoopPushPerRunData(rl);
    CFRunLoopModeRef previousMode = rl->_currentMode;
    rl->_currentMode = currentMode;
    int32_t result = kCFRunLoopRunFinished;

	// 1. 通知 Observers: RunLoop 即将进入 loop
	if (currentMode->_observerMask & kCFRunLoopEntry ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
	// ❤ runloop主要函数，详见👇代码
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
	// 1. 通知 Observers: RunLoop 即将退出。
	if (currentMode->_observerMask & kCFRunLoopExit ) __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);

        __CFRunLoopModeUnlock(currentMode);
        __CFRunLoopPopPerRunData(rl, previousPerRun);
	rl->_currentMode = previousMode;
    __CFRunLoopUnlock(rl);
    return result;
}
```
__CFRunLoopRun: 
`USE_DISPATCH_SOURCE_FOR_TIMERS`flag的使用情况？


```objc
/* rl, rlm are locked on entrance and exit */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    uint64_t startTSR = mach_absolute_time(); // 获取loop开始的时间
    // 如果runloop停止了直接返回
    if (__CFRunLoopIsStopped(rl)) {
        __CFRunLoopUnsetStopped(rl);
	return kCFRunLoopRunStopped;
    } else if (rlm->_stopped) {
	rlm->_stopped = false;
	return kCFRunLoopRunStopped;
    }
    
    // 获取调度的mach_port
    mach_port_name_t dispatchPort = MACH_PORT_NULL;
    Boolean libdispatchQSafe = pthread_main_np() && ((HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && NULL == previousMode) || (!HANDLE_DISPATCH_ON_BASE_INVOCATION_ONLY && 0 == _CFGetTSD(__CFTSDKeyIsInGCDMainQ)));
    if (libdispatchQSafe && (CFRunLoopGetMain() == rl) && CFSetContainsValue(rl->_commonModes, rlm->_name)) dispatchPort = _dispatch_get_main_queue_port_4CF();
    
    // 启动一个超时定时器
    dispatch_source_t timeout_timer = NULL;
    struct __timeout_context *timeout_context = (struct __timeout_context *)malloc(sizeof(*timeout_context));
    if (seconds <= 0.0) { // instant timeout
        seconds = 0.0;
        timeout_context->termTSR = 0ULL;
    } else if (seconds <= TIMER_INTERVAL_LIMIT) {
	dispatch_queue_t queue = pthread_main_np() ? __CFDispatchQueueGetGenericMatchingMain() : __CFDispatchQueueGetGenericBackground();
	timeout_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
        dispatch_retain(timeout_timer);
	timeout_context->ds = timeout_timer;
	timeout_context->rl = (CFRunLoopRef)CFRetain(rl);
	timeout_context->termTSR = startTSR + __CFTimeIntervalToTSR(seconds);
	dispatch_set_context(timeout_timer, timeout_context); // source gets ownership of context
	dispatch_source_set_event_handler_f(timeout_timer, __CFRunLoopTimeout);
        dispatch_source_set_cancel_handler_f(timeout_timer, __CFRunLoopTimeoutCancel);
        uint64_t ns_at = (uint64_t)((__CFTSRToTimeInterval(startTSR) + seconds) * 1000000000ULL);
        dispatch_source_set_timer(timeout_timer, dispatch_time(1, ns_at), DISPATCH_TIME_FOREVER, 1000ULL);
        dispatch_resume(timeout_timer);
    } else { // infinite timeout
        seconds = 9999999999.0;
        timeout_context->termTSR = UINT64_MAX;
    }

    Boolean didDispatchPortLastTime = true;
    int32_t retVal = 0;
    do {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_state_t voucherState = VOUCHER_MACH_MSG_STATE_UNCHANGED;
        voucher_t voucherCopy = NULL;
#endif
        uint8_t msg_buffer[3 * 1024];
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        mach_msg_header_t *msg = NULL;
        mach_port_t livePort = MACH_PORT_NULL;
#endif
	__CFPortSet waitSet = rlm->_portSet;

        __CFRunLoopUnsetIgnoreWakeUps(rl);
		// 2. 通知Observers: RunLoop即将触发Timer回调
        if (rlm->_observerMask & kCFRunLoopBeforeTimers) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 3. 通知Observers: RunLoop即将触发source0（非port）回调
        if (rlm->_observerMask & kCFRunLoopBeforeSources) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
	// 执行加入的block
	__CFRunLoopDoBlocks(rl, rlm);
		// 处理source0
        Boolean sourceHandledThisLoop = __CFRunLoopDoSources0(rl, rlm, stopAfterHandle);
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks(rl, rlm);
	}

        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
	
        if (MACH_PORT_NULL != dispatchPort && !didDispatchPortLastTime) {
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
            msg = (mach_msg_header_t *)msg_buffer;
            // 如果有source1（基于port）消息，跳转到处理source1逻辑
            if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) {
                goto handle_msg;
            }
#elif DEPLOYMENT_TARGET_WINDOWS
            if (__CFRunLoopWaitForMultipleObjects(NULL, &dispatchPort, 0, 0, &livePort, NULL)) {
                goto handle_msg;
            }
#endif
        }

        didDispatchPortLastTime = false;
	2. 通知Observers: RunLoop即将进入sleep状态
	if (!poll && (rlm->_observerMask & kCFRunLoopBeforeWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
	// runloop进入休眠状态
	__CFRunLoopSetSleeping(rl);
	// do not do any user callouts after this point (after notifying of sleeping)

        // Must push the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced.

        __CFPortSetInsert(dispatchPort, waitSet);
        
	__CFRunLoopModeUnlock(rlm);
	__CFRunLoopUnlock(rl);

        CFAbsoluteTime sleepStart = poll ? 0.0 : CFAbsoluteTimeGetCurrent();

#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        if (kCFUseCollectableAllocator) {
            // objc_clear_stack(0);
            // <rdar://problem/16393959>
            memset(msg_buffer, 0, sizeof(msg_buffer));
        }
        msg = (mach_msg_header_t *)msg_buffer;
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
        
        __CFRunLoopLock(rl);
        __CFRunLoopModeLock(rlm);

        rl->_sleepTime += (poll ? 0.0 : (CFAbsoluteTimeGetCurrent() - sleepStart));

        // Must remove the local-to-this-activation ports in on every loop
        // iteration, as this mode could be run re-entrantly and we don't
        // want these ports to get serviced. Also, we don't want them left
        // in there if this function returns.

        __CFPortSetRemove(dispatchPort, waitSet);
        
        __CFRunLoopSetIgnoreWakeUps(rl);

        // user callouts now OK again
	__CFRunLoopUnsetSleeping(rl);
	if (!poll && (rlm->_observerMask & kCFRunLoopAfterWaiting)) __CFRunLoopDoObservers(rl, rlm, kCFRunLoopAfterWaiting);
		// 处理source1逻辑
        handle_msg:;
        __CFRunLoopSetIgnoreWakeUps(rl);
		 // 如果当前的port为空，不做任何处理
        if (MACH_PORT_NULL == livePort) {
            CFRUNLOOP_WAKEUP_FOR_NOTHING();
            // handle nothing
        } else if (livePort == rl->_wakeUpPort) {
            CFRUNLOOP_WAKEUP_FOR_WAKEUP();
            // do nothing on Mac OS
        }
        else if (livePort == dispatchPort) {
            CFRUNLOOP_WAKEUP_FOR_DISPATCH();
            __CFRunLoopModeUnlock(rlm);
            __CFRunLoopUnlock(rl);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)6, NULL);
            
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
            _CFSetTSD(__CFTSDKeyIsInGCDMainQ, (void *)0, NULL);
            __CFRunLoopLock(rl);
            __CFRunLoopModeLock(rlm);
            sourceHandledThisLoop = true;
            didDispatchPortLastTime = true;
        } else {
            CFRUNLOOP_WAKEUP_FOR_SOURCE();
            voucher_t previousVoucher = _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, (void *)voucherCopy, os_release);
				
            // 如果machport里面有source1事件，就进行处理
            CFRunLoopSourceRef rls = __CFRunLoopModeFindSourceForMachPort(rl, rlm, livePort);
            if (rls) {
		mach_msg_header_t *reply = NULL;
		// 处理source1事件
		sourceHandledThisLoop = __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
		if (NULL != reply) {
		    (void)mach_msg(reply, MACH_SEND_MSG, reply->msgh_size, 0, MACH_PORT_NULL, 0, MACH_PORT_NULL);
		    CFAllocatorDeallocate(kCFAllocatorSystemDefault, reply);
		}
	    }
            
            // Restore the previous voucher
            _CFSetTSD(__CFTSDKeyMachMessageHasVoucher, previousVoucher, os_release);
            
        } 
        if (msg && msg != (mach_msg_header_t *)msg_buffer) free(msg);
    // 处理加入的block
	__CFRunLoopDoBlocks(rl, rlm);
        
	// 判断下返回值
	if (sourceHandledThisLoop && stopAfterHandle) {
	  // 处理完source事件返回
	    retVal = kCFRunLoopRunHandledSource;
        } else if (timeout_context->termTSR < mach_absolute_time()) {
        // 超时返回
            retVal = kCFRunLoopRunTimedOut;
	} else if (__CFRunLoopIsStopped(rl)) {
	   // runloop被停止了
            __CFRunLoopUnsetStopped(rl);
	    retVal = kCFRunLoopRunStopped;
	} else if (rlm->_stopped) {
	    rlm->_stopped = false;
	    retVal = kCFRunLoopRunStopped;
	} else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
	    // runloop里面没有source、time、observe中的一个了
	    retVal = kCFRunLoopRunFinished;
	}
        
#if DEPLOYMENT_TARGET_MACOSX || DEPLOYMENT_TARGET_EMBEDDED || DEPLOYMENT_TARGET_EMBEDDED_MINI
        voucher_mach_msg_revert(voucherState);
        os_release(voucherCopy);
#endif

    } while (0 == retVal);

    if (timeout_timer) {
        dispatch_source_cancel(timeout_timer);
        dispatch_release(timeout_timer);
    } else {
        free(timeout_context);
    }

    return retVal;
}
```


文档更新中，写的不足的地方请谅解，您也可以留下您宝贵的意见，谢谢~~~

## 参考文献

[apple document](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)

[深入理解run loop](http://blog.ibireme.com/2015/05/18/runloop/)

[iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)
