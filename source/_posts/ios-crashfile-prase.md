---
title: ios-crashfile-prase
date: 2017-10-04 22:10:15
categories:
- iOS
tags: iOS
---

# crash文件
当运行的APP发生crash的时候，如果代码里面增加对应的handler或者有第三方的crash SDK，他们会采集相关的运行堆栈，发送到对应的服务器上，然后通过开发者上传的dsym文件进行解析，得到符号化的堆栈信息，我们可以通过分析这个知道crash的原因。另外，当发生crash的时候，相应的设备上也会生成一个crash文件。我们可以通过Xcode导出crash文件<!-- more -->。

Window->Devices and Simulator->选择对应的设备->view device log:

![crash-list.png](http://upload-images.jianshu.io/upload_images/1743782-45e95c506ac0b501.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于如何解析符号化crash文件，可以查看我的上一篇[文章](http://www.jianshu.com/p/16b680d45e09)。这里我们分析下文件的基本结构。下面是crash文件的大致预览：

```
Incident Identifier: B9C9CA64-DB6D-4EA7-AE86-A7BD841E2A50
CrashReporter Key:   6b0e0da01ba5183a74d6fb37e0ebd2b46540e89d
Hardware Model:      iPhone7,2
Process:             AlipayWallet [221]
Path:                /private/var/containers/Bundle/Application/8A3BF1BC-06B9-481F-8B33-5F51CCA1D764/AlipayWallet.app/AlipayWallet
Identifier:          com.alipay.iphoneclient
Version:             10.1.5.102407 (10.1.5)
Code Type:           ARM-64 (Native)
Role:                Non UI
Parent Process:      launchd [1]
Coalition:           com.alipay.iphoneclient [359]


Date/Time:           2017-10-28 18:27:52.9845 +0800
Launch Time:         2017-10-28 18:27:26.7691 +0800
OS Version:          iPhone OS 11.0.3 (15A432)
Baseband Version:    6.17.00
Report Version:      104

Exception Type:  EXC_CRASH (SIGKILL)
Exception Codes: 0x0000000000000000, 0x0000000000000000
Exception Note:  EXC_CORPSE_NOTIFY
Termination Reason: Namespace SPRINGBOARD, Code 0x8badf00d
Termination Description: SPRINGBOARD, process-exit watchdog transgression: com.alipay.iphoneclient exhausted real (wall clock) time allowance of 5.00 seconds |  | Elapsed total CPU time (seconds): 10.020 (user 10.020, system 0.000), 100% CPU | Elapsed application CPU time (seconds): 1.194, 12% CPU |
Triggered by Thread:  0

Filtered syslog:
None found

Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   CoreFoundation                  0x00000001818a057c CFAllocatorAllocate + 0
1   CoreFoundation                  0x00000001818a09bc _CFRuntimeCreateInstance + 348
2   CoreFoundation                  0x00000001818a1610 CFBasicHashCreate + 108
3   CoreFoundation                  0x0000000181961c44 __CFDictionaryCreateTransfer + 156
4   CoreFoundation                  0x00000001819462b0 __CFBinaryPlistCreateObjectFiltered + 5952
5   CoreFoundation                  0x0000000181945ed0 __CFBinaryPlistCreateObjectFiltered + 4960
6   CoreFoundation                  0x00000001819469e4 __CFTryParseBinaryPlist + 200
7   CoreFoundation                  0x00000001818a0d54 _CFPropertyListCreateWithData + 192
8   CoreFoundation                  0x00000001818ec178 _CFPropertyListCreateFromXMLData + 116
9   Foundation                      0x00000001822fa68c +[NSPropertyListSerialization propertyListFromData:mutabilityOption:format:errorDescription:] + 52
10  AlipayWallet                    0x00000001023f2058 0x100480000 + 32972888
11  AlipayWallet                    0x00000001023f0ad4 0x100480000 + 32967380
```
## header
* Incident Identifier： 事件标识符，每个crash文件对应一个唯一的标识符。
* CrashReporter Key： 匿名设备标识符。
* Hardware Model：设备型号；
* Process： 进程名；
* Identifier：app Identifier；
* Exception Type： 异常类型；
* Exception Codes： 异常代码；
* Termination Reason：进程被结束的原因

## 线程堆栈信息

下面是各个线程的堆栈信息，有部分是符号化的，有的是没有符号化的，一般系统动态库相关的都已经符号化的，没有符号化的是运行APP里面的符号，我们可以通过`atos`或者`symbolicatecrash `命令逐行解析。

```bash
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   CoreFoundation                  0x00000001818a057c CFAllocatorAllocate + 0
.....
9   Foundation                      0x00000001822fa68c +[NSPropertyListSerialization propertyListFromData:mutabilityOption:format:errorDescription:] + 52
10  AlipayWallet                    0x00000001023f2058 0x100480000 + 32972888
11  AlipayWallet                    0x00000001023f0ad4 0x100480000 + 32967380

```
* 第一行的数字表示对应线程的调用栈顺序；
* 第二行表示对应的镜像文件名称，如系统的foundation等等；
* 第三行表示当前行对应的符号地址；
* 第四行： 如果是已经符号化过的，则表示对应的符号，没有符号化的是镜像的起始地址+文件偏移地址，这2个值相加实际上就是第三行的地址。

# Exception

从上面的分析我们可以知道在crash文件的头部包含的Exception相关信息。下面我们来看看我们常见的几种Exception类型：

### Bad Memory Access [EXC_BAD_ACCESS // SIGSEGV // SIGBUS]

当进程尝试的去访问一个不可用或者不允许访问的内存空间时，会发生野指针异常，`Exception Subtype`中包含了错误的描述及想要访问的地址。我们可以通过以下方法来发生我们APP里面的野指针问题：
* 如果`objc_msgSend `或者`objc_release `符号信息位于crash线程的最上面，说明该进程可能尝试访问已经释放的对象。我们可以使用[Zombies instrument](https://developer.apple.com/library/ios/documentation/DeveloperTools/Conceptual/InstrumentsUserGuide/EradicatingZombies.html)更好地了解此次崩溃的情况。
* 如果`gpus_ReturnNotPermittedKillClient`符号在crash线程的最上面，说明进程试图在后台使用OpenGL ES或Metal进行渲染。
* 在debug模式下打开[Address Sanitizer](https://developer.apple.com/videos/play/wwdc2015/413/)，它会在编译的时候自动添加一些关于内存访问的工具，在crash发生的时候，Xcode会给出对应的详细信息。

## Abnormal Exit [EXC_CRASH // SIGABRT]
非正常退出，大多数发生这种crash的原因是因为未捕获的`Objective-C/C++`异常，导致进程调用`abort()`方法退出。上面的crash文件产生的原因就是这个。
如果APP消耗了太多的时间在初始化，`watchdog `（看门狗定时器）就会终止程序运行。

## Trace Trap [EXC_BREAKPOINT // SIGTRAP]
这个异常是为了让一个附加的调试器有机会在执行过程中的某个特定时刻中断进程，我们可以在代码里面添加`__builtin_trap()`方法手动触发这个异常。如果没有附加调试器，则会生成crash文件。
较低级别的库（例如libdispatch）会在遇到致命错误时捕获该进程。
swift代码在遇到下面2种情况的时候也会抛出这个异常：
* 一个非可选类型的值为nil
* 错误的强制类型转换，如把NSString转换为NSDate等等。

## Illegal Instruction [EXC_BAD_INSTRUCTION // SIGILL]
程序尝试的去执行一个非法的或者没有定义的指令。如果配置的函数地址有问题，进程在执行函数跳转的时候，就会发生这个crash。

## Quit [SIGQUIT]
该进程在具有管理其生命周期的权限的另一进程的请求下终止。 SIGQUIT并不意味着进程崩溃，但是可以说明该进程存在一些问题。
比如在iOS中，第三方键盘应用可能在在其他APP中被唤起，但是如果键盘应用需要很长的时间去加载，则会被强制退出。

## Killed [SIGKILL]
进程被系统强制结束，通过查看`Termination Reason`找到crash信息。

## Guarded Resource Violation
## Resource Limit
进程使用的资源超出的限制。这个一个系统的通知，我们可以在` Exception Subtype`里面查看相关信息。

## Other Exception Types
一些crash report记录的exception信息可能并不是描述型的，可能是一个十六进制的代码，比如`00000020 `，下面列出了几种常见的异常代码：

* `0xbaaaaaad `：该code表示这个crash文件是系统的stackshot，并不是crash report，可以通过按住`home`+`voice`按键生成；
* `0xbad22222 `：一个`VoIP `应用启动恢复的次数太频繁；
* `0x8badf00d `： 看门狗定时器超时，上面有讲到，一般是因为APP启动的时间过长或者响应系统事件事件超时导致；比如在主线程进行网络请求，主线程会一直卡住知道网络回调回来；
* `0xc00010ff ` ： 当操作系统响应`thermal `事件的时候，会强制的kill进程。
* `0xdead10cc `： 进程在suspend期间保持在文件锁或sqlite数据库锁。

# reference

[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/content/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-APPINFO)
