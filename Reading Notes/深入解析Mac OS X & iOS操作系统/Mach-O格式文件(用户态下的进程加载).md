---
title: Mach-O格式文件(用户态下的进程加载)
categories: 
 - Apple Development
 - 深入解析Mac OS X && iOS操作系统笔记
 - 内核
tags:
 - 内核
 - Mach
---

## Mach-O二进制文件
Mach-O的文件头包含的内容:

- 魔数
- CPU类型及其子类型
- 文件类型
- 用于加载器的“加载命令”的条数和大小
- 动态链接器的标志

> 使用`otool -h /bin/ls`来查看Mach-O的文件头。

## Mach-O的加载命令
内核加载器会在加载的过程中使用这些命令来对进程进行一些设置：包括分配虚拟内存、创建主线程、启动动态链接器以及处理代码签名等工作。重要的命令有：

- LC_SEGMENT或者LC_SEGMENT_64（设置进程的内存空间）
 - 代码段（__TEXT）、数据段(__DATA)、用户动态链接的桩(__stubs、__stub_helper)、主程序代码(__text)
- LC_LOAD_DYLINKER(内核加载器在执行该命令时启动动态链接器)
- LC_MAIN(设置进程的入口地址和栈大小，以及出程序计数器外的寄存器清零)
- LC_CODE_SIGNATURE(代码签名)

> `otool`可以用来可以用来分析加载命令和代码段，如：`otool -l /bin/ls`

## 动态库
### 动态链接
少量的进程只需要`内核加载器`就能完成加载，OSX中几乎所有的程序都是动态链接的--即填补对外部库和符号的引用。这个工作是由`动态链接器`来完成。该过程也被称为`符号绑定`。这个过程大概是这样的：
> 
如果二进制文件使用了外部定义的函数或符号，那么在他们的文本段中就会有一个名为__stubs的区，在这个区中存放的是本地未定义符号的占位符。编译器生成代码时会创建对符号桩的调用，链接器在运行的时候会解决对桩的这些调用--即在被调用的地址处放置一条JMP指令，并将控制权交给真实的函数体。但不会修改栈。因此真实的函数可以正常返回，就像直接调用函数一样。

链接一般都是递归的，因为库也有可能引用其他的库。

### 共享库缓存（shared library cache）
共享库缓存是dyld支持的的另一种机制。是指：一些库经过预先链接，然后保存在磁盘的一个文件中。
> 
在OS X中dyld共享缓存保存在`/private/var/db/dyld`目录下。在iOS中则保存在`/System/Library/Caches/com.apple.dyld`.

### 运行时加载
一般通过#include包含一些头文件，这种方式构建的可执行文件只有在解决了所有依赖条件之后才能加载执行。但是通过`<dlfcn.h>`头文件提供的函数就可以在运行时（runtime）加载库。这样函数有：

- dlopen(const char *path)
- dlopen_preflight(const char *path)
- dlsym(void *handle ,char *sym)
- dladdr(char *addr , DL_Info *info)
- dlerror()

Cocoa和Carbon为dl*系列提供了高层的封装，以及CFBundle和NSBundle对象，用于加载Mach-O bundle文件。

### 弱定义的符号
- 通常情况下符号都是被声明为强定义的，即文件在执行之前必须先解析这些符号，若发生解析失败，则程序运行失败，通常也会触发调试器陷阱。
- 可以使用`__attribute__(weak_import)`将符号声明为弱符号。这样则在解析符号错误的时候，不会触发链接错误，动态链接器会将这个符号设置为NULL，效果跟运行时加载动态库类似（如dlopen）。

> 使用`nm -m xxx.dylib`可以显示弱符号。

## dyld的特性
### 两级命名空间
- 通过将DYLD_FORCE_FLAT_NAMESPACE环境变量设置为非零即可禁用。
- 可执行文件也可以在文件头中设置MH_FORCE_FLAT标志，强制对其加载的所有库使用平坦命名空间。

### 函数拦截
- DYLD_INTERPOSE宏允许一个库将其函数替换为另一个函数。（跟iOS的swizzle类似）,例如：
```
DYLD_INTERPOSE(my_open ,open)
```
- dyld的函数拦截功能提供一个新的__DATA区，名为__interpose,在这个区中依次列出了替换的函数和被替换的函数，其他事情则交给dyld处理。例如：
```
static const interpose_t interposing_functions[] \
    __attribute__(section("__DATA,__interpose")) = {
        {(void *)my_free , (void *)free },
        {(void *)my_malloc , (void *) malloc },
    };
```
完整代码：
```
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <malloc/malloc.h> // for malloc_printf()

// Note: Compile with GCC, not cc (important)
//
//
// This is the expected interpose structure
 typedef struct interpose_s { void *new_func;
			       void *orig_func; } interpose_t;
// Our prototypes - requires since we are putting them in 
//  the interposing_functions, below

void *my_malloc(int size); // matches real malloc()
void my_free (void *); // matches real free()

// For clang, add attribute(used)
static const interpose_t interposing_functions[] \ 
    __attribute__ ((used, section("__DATA, __interpose"))) = {

 { (void *)my_free, (void *)free },
 { (void *)my_malloc, (void *)malloc } 

};

void *
my_malloc (int size) {
 // In our function we have access to the real malloc() -
 // and since we don’t want to mess with the heap ourselves,
 // just call it
 //
void *returned = malloc(size);
// call malloc_printf() because the real printf() calls malloc()
// // internally - and would end up calling us, recursing ad infinitum

  malloc_printf ( "+ %p %d\n", returned, size); return (returned);
}
void
my_free (void *freed) {
// Free - just print the address, then call the real free()


  malloc_printf ( "- %p\n", freed); free(freed);
}



#if 0
  From output 4-11:

 morpheus@Ergo(~)$ gcc -dynamiclib l.c -o libMTrace.dylib -Wall  // compile to dylib
 morpheus@Ergo(~)$ DYLD_INSERT_LIBRARIES=libMTrace.dylib ls     // force insert into ls
 ls(24346) malloc: + 0x100100020 88
 ls(24346) malloc: + 0x100800000 4096
 ls(24346) malloc: + 0x100801000 2160 
 ls(24346) malloc: - 0x100800000 
 ls(24346) malloc: + 0x100801a00 3312 ... // etc.

#endif
```
> 使用`pagestuff`命令可以显示文件逻辑页中的符号。如：`pagestuff /usr/lib/libgmalloc.dylib 6`,

## 进程的地址空间
- 每一个进程都有自己私有的虚拟地址空间。
- 32位地址空间，用户态可访问整个4G的内存空间。
- 64位的地址允许高达16EB（16GGB）
- 现代系统一般都会在每次启动进程的时候，将其地址空间随机化（随机的给每个段加上地址偏移）。
> 使用`vmmap`命令来查看内存的空间布局，可以加上参数`-interleaved`以清晰的方式导出地址空间。















