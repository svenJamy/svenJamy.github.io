---
title: libffi简介及在iOS中的使用技巧
date: 2018-01-13 14:55:39
categories:
- iOS
tags: iOS
---

## 什么是Libffi

高级语言编译器产生代码时都会依据一系列的规则，这些规则是十分必要的，特别是对独立的编译器来说。“调用约定” （Calling Convention），它包含了编译器关于函数入口处的函数参数、函数返回值的一系列假设。常被称作“ABI”（Application Binary Interface）<!-- more -->。调用约定（Calling Conventions）定义了程序中调用函数的方式，它决定了在函数调用的时候数据（比如说参数）在堆栈中的组织方式。有些程序在编译的时候可能不知道什么参数要传递给一个函数，但是可以在运行时告诉解释器用于调用给定函数的参数的数量和类型。`Libffi`可以用来提供从解释程序到编译代码的桥梁。

FFI(Foreign Function Interface),允许以一种语言编写的代码调用另一种语言的代码。而`Libffi`库只提供了最底层的、与架构相关的、完整的”FFI”，因此在它之上必须有一层来负责管理两种语言之间参数的格式转换。

`LibFFI`针对这些不同的调用约定，提供一个高层次的可移植的API，只需调用这些API就可以在运行时进行动态的函数调用。

## 使用方法

### 基本使用
`Libffi`假定你有一个指向你希望调用的函数的指针，并且你知道传递参数的数量和类型，以及函数的返回类型。首先我们需要创建一个`ffi_cif`对象来匹配调用的函数的签名（methodSignature）。要准备调用接口对象，请使用函数`ffi_prep_cif`。

```
typedef struct {
  ffi_abi abi;
  unsigned nargs;
  ffi_type **arg_types;
  ffi_type *rtype;
  unsigned bytes;
  unsigned flags;
#ifdef FFI_EXTRA_CIF_FIELDS
  FFI_EXTRA_CIF_FIELDS;
#endif
} ffi_cif;

ffi_status ffi_prep_cif(ffi_cif *cif,
			ffi_abi abi,	// ABI
			unsigned int nargs, // 参数个数
			ffi_type *rtype, // 函数返回值指针
			ffi_type **atypes); // 函数参数指针
```
当使用`ffi_prep_cif`函数创建好`ffi_cif`结构体之后，我们就可以使用`ffi_call`函数来调用对应的函数：

```
#include <stdio.h>
#include <ffi.h>
     
 int main()
 {
   ffi_cif cif;
   ffi_type *args[1];
   void *values[1];
   char *s;
   int rc;
 
   // 初始化参数指针
   args[0] = &ffi_type_pointer;
   values[0] = &s;
 
   // 初始化cif
   if (ffi_prep_cif(&cif, FFI_DEFAULT_ABI, 1,
 		       &ffi_type_uint, args) == FFI_OK)
     {
       s = "Hello World!";
       ffi_call(&cif, puts, &rc, values);
       // rc存储着puts函数的返回值
     }
 
   return 0;
 }
```

### Closure API

`libffi`还提供了一个通用函数的方法，可以接受和解码任何参数组合的函数。这个很有用，比如在OC中，我们可以用这种方式去hock任一方法的调用。

```
// size : sizeof(ffi_closure)
// code: 新函数的地址，对应OC中的IMP
void *ffi_closure_alloc (size_t size, void **code);

ffi_status
ffi_prep_closure_loc (ffi_closure*,
		      ffi_cif *,
		      void (*fun)(ffi_cif*,void*,void**,void*),
		      void *user_data,
		      void*codeloc);
```

closure API的使用方法：

```
#include <stdio.h>
#include <ffi.h>
 
/* Acts like puts with the file given at time of enclosure. */
void puts_binding(ffi_cif *cif, unsigned int *ret, void* args[],
               FILE *stream)
{
*ret = fputs(*(char **)args[0], stream);
}
 
int main()
{
	ffi_cif cif;
	ffi_type *args[1];
	ffi_closure *closure;
	 
	int (*bound_puts)(char *);
	int rc;
	 
	/* Allocate closure and bound_puts */
	closure = ffi_closure_alloc(sizeof(ffi_closure), &bound_puts);
	 
	if (closure)
	 {
	   /* Initialize the argument info vectors */
	   args[0] = &ffi_type_pointer;
	 
	   /* Initialize the cif */
	   if (ffi_prep_cif(&cif, FFI_DEFAULT_ABI, 1,
	                    &ffi_type_uint, args) == FFI_OK)
	     {
	       /* Initialize the closure, setting stream to stdout */
	       if (ffi_prep_closure_loc(closure, &cif, puts_binding,
	                                stdout, bound_puts) == FFI_OK)
	         {
	           rc = bound_puts("Hello World!");
	           /* rc now holds the result of the call to fputs */
	         }
	     }
	 }
	 
	/* Deallocate both closure, and bound_puts */
	ffi_closure_free(closure);
	 
	return 0;
}
```


### 内建type

FFI提供了一些内建的type用来描述函数的参数和返回值。

```
ffi_type_void
The type void. This cannot be used for argument types, only for return values. 
ffi_type_uint8
An unsigned, 8-bit integer type. 
ffi_type_sint8
A signed, 8-bit integer type. 
ffi_type_uint16
An unsigned, 16-bit integer type. 
ffi_type_sint16
A signed, 16-bit integer type. 
ffi_type_uint32
An unsigned, 32-bit integer type. 
ffi_type_sint32
A signed, 32-bit integer type. 
ffi_type_uint64
An unsigned, 64-bit integer type. 
ffi_type_sint64
A signed, 64-bit integer type. 
ffi_type_float
The C float type. 
ffi_type_double
The C double type. 
ffi_type_uchar
The C unsigned char type. 
ffi_type_schar
The C signed char type. (Note that there is not an exact equivalent to the C char type in libffi; ordinarily you should either use ffi_type_schar or ffi_type_uchar depending on whether char is signed.) 
ffi_type_ushort
The C unsigned short type. 
ffi_type_sshort
The C short type. 
ffi_type_uint
The C unsigned int type. 
ffi_type_sint
The C int type. 
ffi_type_ulong
The C unsigned long type. 
ffi_type_slong
The C long type. 
ffi_type_longdouble
On platforms that have a C long double type, this is defined. On other platforms, it is not. 
ffi_type_pointer
A generic void * pointer. You should use this for all pointers, regardless of their real type.
```



## reference

* [GitHub libFFi](https://github.com/libffi/libffi)
* [atmark-techno.com](https://www.atmark-techno.com/~yashi/libffi.html#Types)
