---
title: ios-machO
date: 2017-07-08 20:32:45
categories:
- iOS
tags: iOS
---

熟悉`Linux`和`windows`开发的同学都知道，`ELF`是`Linux`下可执行文件的格式，`PE32／PE32+`是`windows`下可执行文件的格式，下面我们要讲的就`OSX`和`iOS`环境下可执行文件的格式：`mach-o`<!-- more -->。

Mach-O 是 Mach object 文件格式的缩写，它是一种用于记录可执行文件、对象代码、共享库、动态加载代码和内存转储的文件格式。作为 .out 格式的替代品，Mach-O 提供了更好的扩展性，并提升了符号表中信息的访问速度（from wiki）。

## 分析工具
1. [MachOView](https://github.com/gdbinit/MachOView)工具可以在Mac平台上查看MachO文件格式信息，是一种可视化的方式，能更直观的方便我们查看；
2. [otool](http://www.manpagez.com/man/1/otool/)是命令行工具，该工具是`Xcode`自带的，路径是：`/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/otool`，具体操作可以使用`man otool`查看。

## 结构
从苹果的官方文档中我们可以知道，`mach-o`文件主要包括以下三个区域：

![apple-doc.jpg](http://upload-images.jianshu.io/upload_images/1743782-dc4d007313f0118a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. `Header`头部区域，主要包括`mach-o`文件的一些基本信息；
2. `LoadCommands`加载命令，是由多个`Segment command`组成的；
3. `Data`包含Segement的具体数据。

下面我们通过工具看下具体区域的结构。

## Header
`Header`位于`mach-o`文件的开始位置，不管是在32位架构还是64位架构。根据🍎开源的代码中我们可以看到以下数据结构（下面的代码默认都是以64位为例）：

```objc
/*
* The 64-bit mach header appears at the very beginning of object files for
* 64-bit architectures.
*/
struct mach_header_64 {
uint32_t  magic;    /* mach magic number identifier */
cpu_type_t  cputype;  /* cpu specifier */
cpu_subtype_t  cpusubtype;  /* machine specifier */
uint32_t  filetype;  /* type of file */
uint32_t  ncmds;    /* number of load commands */
uint32_t  sizeofcmds;  /* the size of all the load commands */
uint32_t  flags;    /* flags */
uint32_t  reserved;  /* reserved */
};
```

首先我们通过`otool`工具查看`Alipay `的`header`结构：

```shell
➜  ~ otool -hv /Alipay/Payload/AlipayWallet.app/AlipayWallet
Mach header
magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC     ARM         V7  0x00     EXECUTE    75       7524   NOUNDEFS DYLDLINK TWOLEVEL BINDS_TO_WEAK
Mach header
magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
MH_MAGIC_64   ARM64        ALL  0x00     EXECUTE    75       8240   NOUNDEFS DYLDLINK TWOLEVEL BINDS_TO_WEAK PIE
```
二进制文件包含`arm64`和`armv7s`两个指令集，就是我们通常听到的`胖二进制结构`或者多重指令集。下面我们对header的结构做一个大致的分析：

* `magic `: 表示的是`mach-o`文件的魔数`#define MH_MAGIC_64 0xfeedfacf`。
* `cputype `和`cupsubtype `: 标识的是cpu的类型和其子类型,具体定义可以查看`mach/machine.h`文件中的定义。

```objc
#define CPU_TYPE_I386    CPU_TYPE_X86    /* compatibility */
#define  CPU_TYPE_X86_64    (CPU_TYPE_X86 | CPU_ARCH_ABI64)
#define CPU_TYPE_MC98000  ((cpu_type_t) 10)
#define CPU_TYPE_HPPA           ((cpu_type_t) 11)
#define CPU_TYPE_ARM    ((cpu_type_t) 12)
#define CPU_TYPE_ARM64          (CPU_TYPE_ARM | CPU_ARCH_ABI64)
```

* `filetype`文件类型，比如可执行文件、库文件、Dsym文件等，定义如下所示：

```objc
* Constants for the filetype field of the mach_header
*/
#define  MH_OBJECT  0x1    /* relocatable object file */
#define  MH_EXECUTE  0x2    /* demand paged executable file */
#define  MH_FVMLIB  0x3    /* fixed VM shared library file */
#define  MH_CORE    0x4    /* core file */
#define  MH_PRELOAD  0x5    /* preloaded executable file */
#define  MH_DYLIB  0x6    /* dynamically bound shared library */
#define  MH_DYLINKER  0x7    /* dynamic link editor */
#define  MH_BUNDLE  0x8    /* dynamically bound bundle file */
#define  MH_DYLIB_STUB  0x9    /* shared library stub for static */
/*  linking only, no section contents */
#define  MH_DSYM    0xa    /* companion file with only debug */
/*  sections */
#define  MH_KEXT_BUNDLE  0xb    /* x86_64 kexts */
```

* `ncmds`：`load commands`的个数；
* `sizeofcmds`：所有的`load commands`的大小。
* `flags`:标志位，定义有很多，具体可以看下`mach-o/load.h`里面的定义，下面列出了上面`Alipay`打印出来的header的flag：

```objc
#define  MH_NOUNDEFS  0x1    /* 该文件没有未定义的符号引用 */
#define MH_DYLDLINK  0x4    /* 该文件是dyld的输入文件，无法被再次静态链接 */
#define MH_TWOLEVEL  0x80    /* 该镜像文件使用2级名称空间 */
#define MH_BINDS_TO_WEAK 0x10000  /* 最后链接的镜像文件使用弱符号 */
#define MH_PIE 0x200000 /* 加载程序在随机的地址空间，只在MH_EXECUTE中使用 */
```
### 二级名称空间

这是dyld的一个独有特性，说是符号空间中还包括所在库的信息，这样子就可以让两个不同的库导出相同的符号，与其对应的是平坦名称空间。

## Load Commands
`load command`结构紧跟着`header`结构的后面，并且它们在虚拟内存中指定文件的逻辑结构和文件的布局。 每个加载命令从指定命令类型和命令数据大小的字段开始。
`load command`的结构如下所示：

```objc
struct load_command {
uint32_t cmd;    /* type of load command */
uint32_t cmdsize;  /* total size of command in bytes */
};
```
* `cmd `: `load command`的类型，具体支持的类型如下图所示；
* `cmdsize `: 所有`command`的大小。

下面列举常见的command类型及用途：

| `命令类型`  | `用途` |
|:------------- | :------------- |
| LC_UUID | 为镜像文件或者`dSYM`文件指定一个128位的UUID  |
| LC_SEGMENT |定义该文件的一段，以映射到加载该文件的进程的地址空间中。 它还包括该段所包含的所有`sections`部分 |
| LC_SEGMENT_64 |   同上，映射到64位的`segment`      |
| LC_SYMTAB | 符号表，当链接文件时，静态和动态链接器都使用此信息，还可以通过调试器将符号映射到生成符号的原始源代码文件|
| LC_DYSYMTAB | 动态链接器使用的附加符号表信息 |
| LC_THREAD LC_UNIXTHREAD | 对于一个可执行文件，`LC_UNIXTHREAD `定义了主线程初始化的线程状态|
| LC_LOAD_DYLIB |定义此文件链接的动态共享库的名称 |
| LC_ID_DYLIB |指定安装的动态共享库名字 |
| LC_PREBOUND_DYLIB | 对于共享库，该可执行文件与Prebound对齐，指定共享库中使用的模块|
| LC_LOAD_DYLINKER |指定内核执行加载此文件的动态链接器 |
| LC_ID_DYLINKER |标识这个文件为动态链接器 |
| LC_ROUTINES |包含共享库初始化例程的地址 |
| LC_ROUTINES_64  |同上，64位 |
| LC_TWOLEVEL_HINTS  |包含两级命名空间查询提示表 |
| LC_SUB_FRAMEWORK  |标识这个文件为一个`umbrella framework`的`subframework `的实现，`umbrella framework`的名字存储在字符串参数中 |
| LC_SUB_UMBRELLA  |标识这个文件是`umbrella framework`的一个`subumbrella` |

这些加载命令在Mach-O文件加载解析时，被内核加载器或者动态链接器调用，指导如何设置加载对应的二进制数据段，具体定义可以查看`/usr/include/mach-o/loader.h`文件或者最后列出的🍎文档。

可以使用命令`otool -v -l AlipayWallet`查看APP当前的所有的`load command`。

## 段数据（Segments）

在上面`load command`中有定义`LC_SEGMENT `和`LC_SEGMENT_64`两种类型，它们实际上就是标识的`segments`，我们可以看下`Alipay`的结构：

![macho-segment.png](http://upload-images.jianshu.io/upload_images/1743782-18c5869a94631749.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数据结构定义：

```objc
struct segment_command_64 { /* for 64-bit architectures */
uint32_t  cmd;    /* LC_SEGMENT_64 */
uint32_t  cmdsize;  /* includes sizeof section_64 structs */
char    segname[16];  /* segment name */
uint64_t  vmaddr;    /* memory address of this segment */
uint64_t  vmsize;    /* memory size of this segment */
uint64_t  fileoff;  /* file offset of this segment */
uint64_t  filesize;  /* amount to map from the file */
vm_prot_t  maxprot;  /* maximum VM protection */
vm_prot_t  initprot;  /* initial VM protection */
uint32_t  nsects;    /* number of sections in segment */
uint32_t  flags;    /* flags */
};
```

每一个segment定义了一些Mach-O文件的数据、虚拟地址、文件偏移和内存保护等属性，这些数据在动态链接器加载程序时被映射到了虚拟内存中。每个段都有不同的功能，主要包括如下几种类型：

* __PAGEZERO: 空指针陷阱段，映射到虚拟内存空间的第一页，用于捕捉对NULL指针的引用，没有保护标志位；
* __TEXT: 包含了执行代码以及其他只读数据；
* __DATA: 包含了程序数据，该段可读写；
* __LINKEDIT: 含有为动态链接库使用的原始数据，比如符号，字符串，重定位表条目等等， 该段可读写。
* _RODATA：包含objc类名、方法等列表。

一般情况下`__TEXT `和`__DATA `段又分为几种`section`。
`section`括号前面大写的字母代码`segment`的名称，后面小写字母代表`section`名称。

下面列出`__TEXT`和`__DATA`下的section：

|__TEXT section| 用途 |
|:------------- | :------------- |
|__text|程序代码|
|__stubs|用于动态链接的存根|
|__stub_helper|用于动态链接的存根|
|__const|程序中const关键字修饰的常量变量|
|__objc_methname|objc方法名|
|__cstring|程序中硬编码的ANSI的字符串|
|__objc_classname | objc类名 |
|__objc_methtype | objc方法类型 |
|__gcc_except_tab | 异常处理相关 |
|__ustring |unicode字符串|
|__unwind_info | 异常处理 |
|__eh_frame | 异常处理|


|__DATA section| 用途 |
|:------------- | :------------- |
| __nl_symbol_ptr |动态符号链接相关,指针数组|
|__got|全局偏移表, Global Offset Table|
|__la_symbol_ptr|动态符号链接相关，也是指针数组，通过dyld_stub_binder辅助链接|
|__mod_init_func|初始化的全局函数地址，会在main之前被调用|
|__const|const修饰的常量|
|__cstring|程序中硬编码的ANSI的字符串|
|__cfstring | CF用到的字符串 |
|__objc_classlist | objc类列表 |
|__objc_nlclslist | objc`load`方法列表 |
|__objc_catlist |objc `category`列表|
|__objc_protolist | objc `protocol`列表 |
|__objc_imageinfo | 镜像信息 |
|__objc_const | objc的常量 |
|__objc_selrefs | objc引用的`SEL`列表 |
|__objc_protorefs |objc引用的`protocol`列表|
|__objc_classrefs | objc引用的`class`列表 |
|__objc_superrefs |objc父类的引用列表|
|__objc_ivar | objc`ivar`信息 |
|__objc_data | `class`信息 |
|__bss |未初始化的静态变量区|
|__data | 初始化的可变变量 |

###随机地址空间

进程每一次启动，地址空间都会简单地随机化。该方式对APP的安全性具有重大的意义。如果采用传统的方式，程序的每一次启动的虚拟内存镜像都是一致的，黑客很容易采取重写内存的方式来破解程序。采用`ASLR`可以有效的避免黑客的中间人攻击等。

###dyld

[dyld](http://opensource.apple.com/tarballs/dyld)动态链接器，全名是`dynamic link editer`，当内核执行LC_DYLINK时，连接器会启动，查找进程所依赖的所有的动态库，并加载到内存中，当然，🍎内部对这个步骤做了很多优化，有兴趣的可以查看`WWDC`相关的视频。

## 用途
上面我们列举了`mach-o`文件格式的组成部分及一些主要结构，通过分析我们的APP的可执行文件，可以帮助我们更深层次的了解APP的一些信息，也可以做一些hack的事情，比如微信之前有出一篇博文，讲的就是通过`LinkMap`文件进行iOS APP瘦身。这里注意，如果不使用`LinkMap `文件而是APP可执行文件的话，一定要是debug编译出来的，因为release编译的话是已经优化过的二进制文件，没有我们想要的一些信息。

### 查找无用selector
以往C++在链接时，没有被用到的类和方法是不会编进可执行文件里。但Objctive-C不同，由于它的动态性，它可以通过类名和方法名获取这个类和方法进行调用，所以编译器会把项目里所有OC源文件编进可执行文件里，哪怕该类和方法没有被使用到。

结合LinkMap文件的`__TEXT.__text`，通过正则表达式`([+|-][.+\s(.+)])`，我们可以提取当前可执行文件里所有objc类方法和实例方法（SelectorsAll）。再使用`otool`命令`otool -v -s __DATA __objc_selrefs`逆向`__DATA.__objc_selrefs`段，提取可执行文件里引用到的方法名（UsedSelectorsAll），我们可以大致分析出SelectorsAll里哪些方法是没有被引用的（SelectorsAll-UsedSelectorsAll）。注意，系统API的Protocol可能被列入无用方法名单里，如`UITableViewDelegate`的方法，我们只需要对这些Protocol里的方法加入白名单过滤即可。

另外第三方库的无用selector也可以这样扫出来的。

### 查找无用oc类
查找无用oc类有两种方式，一种是类似于查找无用资源，通过搜索"`[ClassName alloc/new"、"ClassName *"、"[ClassName class]`"等关键字在代码里是否出现。另一种是通过otool命令逆向`__DATA.__objc_classlist`段和`__DATA.__objc_classrefs`段来获取当前所有oc类和被引用的oc类，两个集合相减就是无用oc类，具体分析如下：
通过命令行：`otool -arch arm64 -o AlipayWallet > ~/Desktop/all.txt`保存所有的信息。

打开文件可以看到类似下面的数据：
```objc
/// 所有的信息
AlipayWallet:
Contents of (__DATA,__objc_classlist) section
00000001039d6d60 0x10442e250
isa 0x10442e228
superclass 0x10442e7a0
cache 0x0
vtable 0x0
data 0x1039f2378 (struct class_ro_t *)
flags 0x90
instanceStart 8
instanceSize 8
reserved 0x0
ivarLayout 0x0
name 0x104c5d014 APSettingsCanSchemeHandleParse
baseMethods 0x1039f2358 (struct method_list_t *)
entsize 24
count 1
name 0x10473c037 doParseURL:
types 0x104ca59c7 @24@0:8@16
imp
baseProtocols 0x0
ivars 0x0
weakIvarLayout 0x0
baseProperties 0x0
Meta Class
isa 0x0
superclass 0x10442e778
cache 0x0
vtable 0x0
data 0x1039f2310 (struct class_ro_t *)
flags 0x91 RO_META
instanceStart 40
instanceSize 40
reserved 0x0
ivarLayout 0x0
name 0x104c5d014 APSettingsCanSchemeHandleParse
baseMethods 0x0 (struct method_list_t *)
baseProtocols 0x0
ivars 0x0
weakIvarLayout 0x0
baseProperties 0x0
......
```

debug编译出来的可执行文件`objc_classrefs`段：

```objc
...
0000000101ba2340 0x101befcb0 _OBJC_CLASS_$_QQObjectPasteboard
0000000101ba2348 0x101befd28 _OBJC_CLASS_$_QQArrayPasteboard
0000000101ba2350 0x101befe68 _OBJC_CLASS_$_QQApiURLDecoder
0000000101ba2358 0x101bf0200 _OBJC_CLASS_$_QQWebViewKit
0000000101ba2360 0x101befe40 _OBJC_CLASS_$_QQApiURLEncoder
0000000101ba2368 0x101befff8 _OBJC_CLASS_$_GetMessageFromQQReq
0000000101ba2370 0x101bf0048 _OBJC_CLASS_$_GetMessageFromQQResp
0000000101ba2378 0x101bf0138 _OBJC_CLASS_$_ShowMessageFromQQReq
0000000101ba2380 0x101bf0188 _OBJC_CLASS_$_ShowMessageFromQQResp
...
```

release编译出来的可执行文件`objc_classrefs`段：

```bash
...
00000001043dc060 0x0 _OBJC_CLASS_$_NSDictionary
00000001043dc068 0x0 _OBJC_CLASS_$_NSNotificationCenter
00000001043dc070 0x10442e6b0
00000001043dc078 0x10442e340
00000001043dc080 0x10442e5c0
00000001043dc088 0x10442e610
00000001043dc090 0x10442e660
00000001043dc098 0x10442e520
00000001043dc0a0 0x0 _OBJC_CLASS_$_NSBundle
...
```

其中`__objc_classlist `section保存了所有的类信息，`objc_classrefs `section中保存了使用的类信息，`objc_classrefs `第二列是代表偏移量，和`__objc_classlist `中每个类开头`00000001039d6d60 0x10442e250`第二列的值是相同的。所以我们可以通过脚本找出所有的类和所有使用的类，两者相减就是没有使用的类。

针对上面两个小的功能，写了一个小的ruby脚本，自动的找出没有使用的`selector`和`class`，具体的使用及代码大家可以查看我的[GitHub](https://github.com/svenJamy/objcthin).

## References

1. [OS X ABI Mach-O File Format Reference](https://web.archive.org/web/20090901205800/http://developer.apple.com/mac/library/documentation/DeveloperTools/Conceptual/MachORuntime/Reference/reference.html)
2. [Mach-O file](https://en.wikipedia.org/wiki/Mach-O)
3. [mach-o格式浅析(二)](https://m.baidu.com/from=844b/bd_page_type=1/ssid=0/uid=0/pu=usm@2,sz@1320_2001,ta@iphone_1_10.3_3_603/t=iphone/l=3/tc?w=0_10_mach-o%20file%20type&ref=www_iphone&lid=13896337375742785616&fm=alop&m=8&srd=1&title=mach-o%E6%A0%BC%E5%BC%8F%E6%B5%85%E6%9E%90(%E4%B8%80)-tieyan-%E5%8D%9A%E5%AE%A2%E5%9B%AD&dict=30&w_qd=&eqid=c0d9b537a021f8001000000359e40dc7&ntc=1&bdenc=1&nsrc=IlPT2AEptyoA_yixCFOxXnANedT62v3IEQGG_ytK1DK6mlrte4viZQRAVjbqOXaMZpPPtCPQpxkJxnKh2mUskNYWgK&tcid=j8tkh5j8)
4. [iOS微信安装包瘦身](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207986417&idx=1&sn=77ea7d8e4f8ab7b59111e78c86ccfe66&scene=1&srcid=07272JPB19Oy82KZs95O1FeJ&key=8dcebf9e179c9f3af0b0e7ac9e093efd076ae58ebff8f4fe1b402ae7548d1f9059e999c09f0ef5686293ed92bf0c53a0&ascene=0&uin=MTYxMDI5ODg0MQ==&devicetype=iMac14,3+OSX+OSX+10.11.6+build(15G31)&version=11020201&pass_ticket=Ll/5zVyBxfLjPTkRqkFf07gehMwKTvNSMtkPVX2tqYgewVm9d6qWD54rlAqSoIvu)
5. [Executing Mach-O Files](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/MachOTopics/1-Articles/executing_files.html)
6. [Fat_binary](https://en.wikipedia.org/wiki/Fat_binary)
7. [loader.h](https://opensource.apple.com/source/xnu/xnu-2050.18.24/EXTERNAL_HEADERS/mach-o/loader.h)
