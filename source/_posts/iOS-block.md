---
title: iOS block内部实现分析
date: 2017-05-08 21:40:16
tags:
---

# iOS block

## block常见类型
`block`和我们在其他语言常见的闭包类似，其实就是带有自动变量（局部变量）的匿名函数。
<!-- more -->
`block`是C语言级的语法，他们的使用和C语言类似，但是除了可执行代码之外，它们还可以包含自动（堆栈）或托管（堆）内存的变量绑定。因此，`block`维护了一组可用于在执行时影响行为的状态（数据）。

### block数据结构
`block`和大多数OC对象一样，内部实现也是结构体，关于`block`具体实现，可以参考苹果开源代码库([libclosure-65](https://opensource.apple.com/source/libclosure/libclosure-65/runtime.c))。以下是摘录部分主要部分：

```
struct Block_descriptor_1 {
    uintptr_t reserved;
    uintptr_t size;
};

struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
```
我们可以从上面block的内存布局中看出，`isa`指针会指向`block `所属的类型，用于帮助运行时系统进行处理。第二个变量`flag`的定义如下：

```
// Values for Block_layout->flags to describe block objects
enum {
    BLOCK_DEALLOCATING =      (0x0001),  // runtime
    BLOCK_REFCOUNT_MASK =     (0xfffe),  // runtime
    BLOCK_NEEDS_FREE =        (1 << 24), // runtime
    BLOCK_HAS_COPY_DISPOSE =  (1 << 25), // compiler
    BLOCK_HAS_CTOR =          (1 << 26), // compiler: helpers have C++ code
    BLOCK_IS_GC =             (1 << 27), // runtime
    BLOCK_IS_GLOBAL =         (1 << 28), // compiler
    BLOCK_USE_STRET =         (1 << 29), // compiler: undefined if !BLOCK_HAS_SIGNATURE
    BLOCK_HAS_SIGNATURE  =    (1 << 30), // compiler
    BLOCK_HAS_EXTENDED_LAYOUT=(1 << 31)  // compiler
};
```
从[源码](https://opensource.apple.com/source/libclosure/libclosure-65/data.c)我们可以看到`block`主要有以下几种类型定义：

```
void * _NSConcreteStackBlock[32] = { 0 };
void * _NSConcreteMallocBlock[32] = { 0 };
void * _NSConcreteAutoBlock[32] = { 0 };
void * _NSConcreteFinalizingBlock[32] = { 0 };
void * _NSConcreteGlobalBlock[32] = { 0 };
void * _NSConcreteWeakBlockVariable[32] = { 0 };
```

其中在iOS中常用的是`_NSConcreteStackBlock `，`_NSConcreteMallocBlock `，`_NSConcreteGlobalBlock `，另外几个是在GC环境中使用的，这里我们不讨论这几种类型。

捕获外部变量角度看他们之间的区别：

>* _NSConcreteStackBlock：
只用到外部局部变量、成员属性变量，且没有强指针引用的block都是StackBlock。
StackBlock的生命周期由系统控制的，在栈上创建，一旦返回之后，就被系统销毁了。

>* _NSConcreteMallocBlock：
有强指针引用或copy修饰的成员属性引用的block在真正发生copy的时候会复制一份到堆中成为MallocBlock，没有强指针引用即销毁，生命周期由代码控制。

>* _NSConcreteGlobalBlock：
没有用到外界变量或只用到全局变量、静态变量的block为_NSConcreteGlobalBlock，生命周期从创建到APP运行结束。

### _NSConcreteGlobalBlock
在以下情况中，block 会初始化为_NSConcreteGlobalBlock：

>* 未捕获外部变量(除了静态变量和全局变量)
>* 当需要布局（layout）的变量的数量为0

在[CGBlocks_8cpp_source](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html#l00326)源码中我们可以看到有如下定义：

```
static void computeBlockInfo(CodeGenModule &CGM, CodeGenFunction *CGF,                              CGBlockInfo &info) {
 ASTContext &C = CGM.getContext();
 const BlockDecl *block = info.getBlockDecl();
 SmallVector<llvm::Type*, 8> elementTypes;
 initializeForBlockHeader(CGM, info, elementTypes);

if (!block->hasCaptures()) { // 如果没有捕获外部变量
 info.StructureType =
 llvm::StructType::get(CGM.getLLVMContext(), elementTypes, true);
 info.CanBeGlobal = true; // _NSConcreteGlobalBlock
 return;
}
  ...
  
  // If that was everything, we're done here.
  if (layout.empty()) {
  	info.StructureType =
	llvm::StructType::get(CGM.getLLVMContext(), elementTypes, true);
	info.CanBeGlobal = true;
	return;
	}
  ...
  }
```

没有用到外部变量可以理解，但是全局和静态变量为什么也是global的呢？

```
#import <Foundation/Foundation.h>

int globle_int = 10;
static float globle_float = 11.f;

void globle_block() {
     static int function_int = 12;
    void (^testBlock)() = ^{
        NSLog(@"get the value:%@,%@,%@",@(globle_int),@(globle_float),@(function_int));
    };
    NSLog(@"block:%@", testBlock);
    testBlock();
}

int main(int argc, const char * argv[]) {
    
    globle_block();
    
    return 0;
}
```
输出如下结果：

```
2017-05-07 10:00:20.258943 block[34791:349008] block:<__NSGlobalBlock__: 0x100001060>
2017-05-07 10:00:20.259451 block[34791:349008] get the value:10,11,12
Program ended with exit code: 0
```
在`return 0`之前打断点或者加上watchpoint可以看到，函数执行之后`block`的内存区域还未释放。

### _NSConcreteStackBlock
只用到外部局部变量、成员属性变量，且没有强指针引用的block都是StackBlock。
StackBlock的生命周期由系统控制的，在栈上创建，一旦返回之后，就被系统销毁了。

### _NSConcreteMallocBlock
由于`_NSConcreteStackBlock `在函数返回后就自动释放了，在ARC环境下，编译器会自动判断，在符合以下情况下的`block`copy到堆上去：
>* 手动调用copy关键字；
>* block被强引用到或者赋给`copy`,`strong`修饰的property；
>* block作为方法返回值；

当block发生`copy`时，会调用到如下方法[libclosure-65](https://opensource.apple.com/source/libclosure/libclosure-65/runtime.c)，关键就是最后一步，把block的`isa`指针指向`_NSConcreteMallocBlock `。

`block_copy`具体实现：

```
// Copy, or bump refcount, of a block.  If really copying, call the copy helper if present.
void *_Block_copy(const void *arg) {
    struct Block_layout *aBlock;

    if (!arg) return NULL;
    
    // The following would be better done as a switch statement
    aBlock = (struct Block_layout *)arg;
    if (aBlock->flags & BLOCK_NEEDS_FREE) {
        // latches on high
        latching_incr_int(&aBlock->flags);
        return aBlock;
    }
    else if (aBlock->flags & BLOCK_IS_GLOBAL) {
        return aBlock;
    }
    else {
        // Its a stack block.  Make a copy.
        struct Block_layout *result = malloc(aBlock->descriptor->size);
        if (!result) return NULL;
        memmove(result, aBlock, aBlock->descriptor->size); // bitcopy first
        // reset refcount
        result->flags &= ~(BLOCK_REFCOUNT_MASK|BLOCK_DEALLOCATING);    // XXX not needed
        result->flags |= BLOCK_NEEDS_FREE | 2;  // logical refcount 1
        _Block_call_copy_helper(result, aBlock);
        // Set isa last so memory analysis tools see a fully-initialized object.
        result->isa = _NSConcreteMallocBlock;
        return result;
    }
}
```

## 变量捕获__block
在平时的使用中我们都知道如果不需要改变外部变量的值的话，在block中是可以直接使用外部变量的，但是如果在block中需要改变外部变量，如果不加上`__block`的话编译器会给我们一个错误的警告`Variable is not assignable(missing __block type specifier)`。我们通过下面2个例子看看其中的缘由，我们新建一个main.m的OC文件，在终端输入以下命令`clang -rewirte-objc main.m`，`clang`会为我们自动生成一个`mian.cpp`的文件：

```
#import <Foundation/Foundation.h>

int globle_int = 10;
int main(int argc, const char * argv[]) {
    
    static int local_int = 11;
    
    int number = 12;
    void (^testBlock)() = ^{
        NSLog(@"number incrment:%@-%@-%@", @(globle_int++), @(local_int++), @(number));
    };
    
    testBlock();
    
    NSLog(@"number incrment:%@-%@-%@", @(globle_int), @(local_int), @(number));
    
    return 0;
}
```
输出结果为：

```
2017-05-07 11:19:49.399983 block[37054:390808] number incrment:10-11-12
2017-05-07 11:19:49.401191 block[37054:390808] number incrment:11-12-12
```
从结果我们可以看到，全局变量和静态变量在`block`中修改值以后，他们的值确实发生了改变，但是局部变量却不行，我们打开`main.cpp`文件，大约有10W行左右，我们只需要关注最后面就行了：

```
int globle_int = 10;

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *local_int;
  int number;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_local_int, int _number, int flags=0) : local_int(_local_int), number(_number) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *local_int = __cself->local_int; // bound by copy
  int number = __cself->number; // bound by copy

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_r9_713pry0d4_x17tjbstr27syh0000gn_T_main_934a9a_mi_0, ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (globle_int++)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), ((*local_int)++)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(number)));
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {

    static int local_int = 11;

    int number = 12;
    void (*testBlock)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &local_int, number));

    ((void (*)(__block_impl *))((__block_impl *)testBlock)->FuncPtr)((__block_impl *)testBlock);

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_r9_713pry0d4_x17tjbstr27syh0000gn_T_main_934a9a_mi_1, ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(globle_int)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(local_int)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(number)));

    return 0;
}
```
我们可以看到，当`block`引用到外部变量的时候，会自动的追加在构造函数中：

```
__main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_local_int, int _number, int flags=0) : local_int(_local_int), number(_number) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
```
这样外部变量就被添加到block中，也就是说，外部变量成为了block结构的一部分。

有一点我们要注意：在上面的构造函数参数中，并没有全局变量。这是因为全局变量的作用域是全局的，所以在block中也无需再次去捕获，只有一个内存地址，在block中改变了值，那么外部同样也改变了。

另外在block_fun_0函数中我们可以看到，编译器自动的给我们加上了`// bound by copy`注释，捕获的值都是通过`__cself->local_int`这样来访问的，并不是对应变量的内存地址。所以在block中改变了相应的值，在外部也是没有变化的。所以如果我们在block中直接改变值，编译器会给我们上面的那个警告。

总结就是：block可以捕获外部变量，但是仅仅是变量的值而不是内存地址，如果需要改变操作在外部也有效，必须要加上`__block`关键字。

我们来看看加上__block之后的效果：

```
#import <Foundation/Foundation.h>

int globle_int = 10;
int main(int argc, const char * argv[]) {
    
    static int local_int = 11;
    
    __block int number = 12;
    void (^testBlock)() = ^{
        NSLog(@"number incrment:%@-%@-%@", @(globle_int++), @(local_int++), @(number++));
    };
    
    testBlock();
    
    NSLog(@"number incrment:%@-%@-%@", @(globle_int), @(local_int), @(number));
    
    return 0;
}
```
输出结果为：

```
2017-05-07 11:42:53.995190 block[37534:401137] number incrment:10-11-12
2017-05-07 11:42:53.996032 block[37534:401137] number incrment:11-12-13
```
可以看到局部变量`number`在外部的值也同样改变了，我们通过`clang`查看下具体内部的实现：

```
int globle_int = 10;
struct __Block_byref_number_0 {
  void *__isa;
__Block_byref_number_0 *__forwarding;
 int __flags;
 int __size;
 int number;
};

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *local_int;
  __Block_byref_number_0 *number; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_local_int, __Block_byref_number_0 *_number, int flags=0) : local_int(_local_int), number(_number->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_number_0 *number = __cself->number; // bound by ref
  int *local_int = __cself->local_int; // bound by copy

        NSLog((NSString *)&__NSConstantStringImpl__var_folders_r9_713pry0d4_x17tjbstr27syh0000gn_T_main_6ca483_mi_0, ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (globle_int++)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), ((*local_int)++)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), ((number->__forwarding->number)++)));
    }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->number, (void*)src->number, 8/*BLOCK_FIELD_IS_BYREF*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->number, 8/*BLOCK_FIELD_IS_BYREF*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, const char * argv[]) {

    static int local_int = 11;

    __attribute__((__blocks__(byref))) __Block_byref_number_0 number = {(void*)0,(__Block_byref_number_0 *)&number, 0, sizeof(__Block_byref_number_0), 12};
    void (*testBlock)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &local_int, (__Block_byref_number_0 *)&number, 570425344));

    ((void (*)(__block_impl *))((__block_impl *)testBlock)->FuncPtr)((__block_impl *)testBlock);

    NSLog((NSString *)&__NSConstantStringImpl__var_folders_r9_713pry0d4_x17tjbstr27syh0000gn_T_main_6ca483_mi_1, ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(globle_int)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)(local_int)), ((NSNumber *(*)(Class, SEL, int))(void *)objc_msgSend)(objc_getClass("NSNumber"), sel_registerName("numberWithInt:"), (int)((number.__forwarding->number))));

    return 0;
}

```
从上面源码可以看出带`__block`的变量被捕获后被转换成了一个`__Block_byref_number_0 `的结构体。这个结构体有5个变量，第一个是`isa`指针，第二个是指向自己的指针，第三个是flag，第四个是他的大小，第五个是和捕获前一样的变量，主要是用来保存值的。

`__block`被编译器转换为以下代码，可以看到第二个初始化为指向自己的指针：

```
__attribute__((__blocks__(byref))) __Block_byref_number_0 number = {(void*)0,(__Block_byref_number_0 *)&number, 0, sizeof(__Block_byref_number_0), 12};
```
这样做的好处是当block被copy到堆上时，我们依然可以通过`number.__forwarding->number`的方式获取到变量的值。

从上面的例子和编译源码分析我们可以得出以下结论：
>* 如果一个变量可以被block捕获，那么在`__main_block_impl_0 `结构体构造函数中就追加了对应的变量。全局变量和静态全局变量
不会被block捕获，因为他们的作用域为全局，block无须去捕获他们，所以block不会增加他们的retaincount。静态变量和普通的变量可以被捕获。
>* 带有__block的变量可以改变变量的值，因为他们是值引用的，没有__block修饰的话只能在block中访问，不能修改他们的值。

## Retain Circle

在平时的开发中，我们可能也会遇到循环引用，比如在block中，对象持有block，block里面又直接引用了对象，这样就造成了循环引用。解决的方法都是一样：打破他们之间的持有关系。先列一个大致的实现：

```
#define weakify(var) __weak typeof(var) weak_##var = var;

#define strongify(var) \
_Pragma("clang diagnostic push") \
_Pragma("clang diagnostic ignored \"-Wshadow\"") \
__strong typeof(var) var = weak_##var; \
_Pragma("clang diagnostic pop")
```
这个写法就是我们经常使用的形式，当然你也可以不写成宏的形式，和AFN一样在使用的时候手动去写也是可以的。

## 参考文献
[Blocks Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html)

[CGBlocks_8cpp_source](http://clang.llvm.org/doxygen/CGBlocks_8cpp_source.html)

[AutomaticReferenceCounting](http://clang.llvm.org/docs/AutomaticReferenceCounting.html)
