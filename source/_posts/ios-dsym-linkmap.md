---
title: ios-dsym-linkmap
date: 2017-09-10 20:32:10
categories:
- iOS
tags: iOS
---

# dsym
当APP发生crash的时候，🍎会生成一个` crash report`文件保存在设备上，我们可以通过Xcode导出，但是这个文件是`unsymbolicated `，需要解析之后才能进行分析，这里就需要用到`dsym`文件<!-- more -->。

## dsym符号集
当编译器把目标代码编译成机器码的时候，它还生成调试符号，将编译二进制中的每个机器指令映射回源代码的对应的位置。在`Xcode`中设置`DEBUG_INFORMATION_FORMAT `属性可以调试符号是在二进制文件中还是单独的`dsym`文件，建议在`debug`模式选择`DWARF`，因为生成`dsym`文件对编译时间会有一定的影响。

## atos命令解析crash文件
atos是mac自带的命令，关于命令的相关使用，在term输入`man atos`即可。解析使用如下命令：
```bash
atos -arch arm64 -o TheElements.app.dSYM/Contents/Resources/DWARF/TheElements -l 0x1000e4000 0x00000001000effdc

-[AtomicElementViewController myTransitionDidStop:finished:context:]
```
* -arch:指令集，现在release的都包含arm64指令集架构。可以通过`atool -hv appname`查看相关的指令集；
* -o :目标文件：可执行文件。
* -l : `load address`，及发生crash对应的镜像起始地址；后面一个地址是`symblicate address`，及符号地址。

### 通过`symbolicatecrash `分析crash文件

Xcode有自带的`symbolicatecrash `工具,可以通过dSYM文件将crash文件中的16进制地址转换成可读的函数地址.
该文件是隐藏文件，可以通过如下命令查找并拷贝到系统目录下，并建立快捷方式。

* 查找`symbolicatecrash `目录

```bash
$ find /Applications/Xcode.app -name symbolicatecrash -type f

-> $symbolicatecrashpath = /Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash
```

* 拷贝`symbolicatecrash `到/usr/bin目录下（不行的话可以手动拷贝一下）

```bash
sudo cp $symbolicatecrashpath /usr/bin/symbolicatecrash
```

* 设置`DEVELOPER_DIR `系统变量

```bash
vi ~/.bash_profile
## 输入下面内容
export DEVELOPER_DIR="/Applications/Xcode.app/Contents/Developer"
## 保存后执行
source .bash_profile
```
* 重启终端，确认设置`DEVELOPER_DIR `系统变量

```bash
echo $DEVELOPER_DIR
-> /Applications/Xcode.app/Contents/Developer
```

* 使用如下命令，即可正确解析crash文件

```bash
symbolicatecrash appname.crash appname.app.dSYM > crash.txt
```
关于具体如何解析crash文件，下篇文章会有详细介绍。

# link map文件
`link map`文件是Xcode生成的链接映射文件，我们可以通过分析link map文件窥探二进制文件中布局及所有文件信息，符号表等信息。例如我们可以知道每个文件运行时所占用的空间，每个文件的方法等等。

## XCode开启编译选项Write Link Map File
XCode -> Project -> Build Settings -> link map -> 把Write Link Map File选项设为yes， 如下图所示。

![xcode setting.png](http://upload-images.jianshu.io/upload_images/1743782-9ea6447d761453c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后找到上面默认的文件位置（或者自定义位置），我们通过AFN的demo来查看下大致的结构：
```bash
# Path: /Users/jamy/Library/Developer/Xcode/DerivedData/AFNetworking-cmhenvqhrddnjtcnbkcxlqypctnc/Build/Products/Debug-iphonesimulator/AFNetworking.framework/AFNetworking
# Arch: x86_64
# Object files:
[  0] linker synthesized
[  1] /Users/jamy/Library/Developer/Xcode/DerivedData/AFNetworking-cmhenvqhrddnjtcnbkcxlqypctnc/Build/Intermediates.noindex/AFNetworking.build/Debug-iphonesimulator/AFNetworking iOS.build/Objects-normal/x86_64/UIProgressView+AFNetworking.o
[  2] /Users/jamy/Library/Developer/Xcode/DerivedData/AFNetworking-cmhenvqhrddnjtcnbkcxlqypctnc/Build/Intermediates.noindex/AFNetworking.build/Debug-iphonesimulator/AFNetworking iOS.build/Objects-normal/x86_64/AFNetworkReachabilityManager.o
[  3] /Users/jamy/Library/Developer/Xcode/DerivedData/AFNetworking-cmhenvqhrddnjtcnbkcxlqypctnc/Build/Intermediates.noindex/AFNetworking.build/Debug-iphonesimulator/AFNetworking iOS.build/Objects-normal/x86_64/UIRefreshControl+AFNetworking.o

.......

# Sections:
# Address  Size      Segment  Section
0x000012C0  0x0003ABEC  __TEXT  __text
0x0003BEAC  0x0000033C  __TEXT  __stubs
0x0003C1E8  0x00000574  __TEXT  __stub_helper
0x0003C75C  0x00000F1C  __TEXT  __gcc_except_tab
0x0003D680  0x000000B8  __TEXT  __const
0x0003D738  0x000052A6  __TEXT  __objc_methname
0x000429DE  0x00002DEA  __TEXT  __cstring
0x000457C8  0x00000475  __TEXT  __objc_classname
0x00045C3D  0x0000106C  __TEXT  __objc_methtype
0x00046CAC  0x00000348  __TEXT  __unwind_info
0x00047000  0x00000010  __DATA  __nl_symbol_ptr
0x00047010  0x00000150  __DATA  __got
0x00047160  0x00000450  __DATA  __la_symbol_ptr
0x000475B0  0x00001180  __DATA  __const
0x00048730  0x00001260  __DATA  __cfstring
0x00049990  0x000000E0  __DATA  __objc_classlist
0x00049A70  0x00000010  __DATA  __objc_nlclslist
0x00049A80  0x00000050  __DATA  __objc_catlist
0x00049AD0  0x00000078  __DATA  __objc_protolist
0x00049B48  0x00000008  __DATA  __objc_imageinfo
0x00049B50  0x000078E8  __DATA  __objc_const
0x00051438  0x000014B0  __DATA  __objc_selrefs
0x000528E8  0x00000018  __DATA  __objc_protorefs
0x00052900  0x00000210  __DATA  __objc_classrefs
0x00052B10  0x000000C8  __DATA  __objc_superrefs
0x00052BD8  0x00000450  __DATA  __objc_ivar
0x00053028  0x00000910  __DATA  __objc_data
0x00053938  0x000005F8  __DATA  __data
0x00053F30  0x00000240  __DATA  __bss

.......

# Symbols:
# Address  Size      File  Name
0x000012C0  0x00000070  [  1] -[UIProgressView(AFNetworking) af_uploadProgressAnimated]
0x00001330  0x00000090  [  1] -[UIProgressView(AFNetworking) af_setUploadProgressAnimated:]

....

0x00002040  0x00000290  [  2] _AFStringFromNetworkReachabilityStatus
0x000022D0  0x000000B0  [  2] +[AFNetworkReachabilityManager sharedManager]
0x00002380  0x00000050  [  2] ___45+[AFNetworkReachabilityManager sharedManager]_block_invoke

.....

0x00003560  0x000000D0  [  3] -[UIRefreshControl(AFNetworking) af_notificationObserver]
0x00003630  0x00000080  [  3] -[UIRefreshControl(AFNetworking) setRefreshingWithStateOfTask:]

......
```
* Object files: 编译后的目标文件.o
* Sections：machO对应的各个段，如__TEXT,__DATA,__LINK_EDITD等等。
* Symbols：符号相关信息，第一列是在文件中的偏移位置，第二列是大小，第三列是对应的文件名[1.2.3]代表上面`Object files`对应的编号。如[1]代表`[  1] /Users/jamy/Library/Developer/Xcode/DerivedData/AFNetworking-cmhenvqhrddnjtcnbkcxlqypctnc/Build/Intermediates.noindex/AFNetworking.build/Debug-iphonesimulator/AFNetworking iOS.build/Objects-normal/x86_64/UIProgressView+AFNetworking.o`文件。

每一行的数据都紧跟在上一行后面，如`_AFStringFromNetworkReachabilityStatus `的文件偏移位置是`0x00002040 `，大小是`0x00000290 `，相加就是下一列`0x000022D0 `。

我们可以通过统计对应的大小进而知道使用的静态库之类的运行时大小，从而做一些优化。

# reference

[Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/content/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-APPINFO)
[iOS APP可执行文件的组成](http://blog.cnbang.net/tech/2296/)
