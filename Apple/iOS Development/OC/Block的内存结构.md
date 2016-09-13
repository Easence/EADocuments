# Block的内存结构

原文：[《谈Objective-C block的实现》][1]

## 在 Objective-C 语言中，一共有 3 种类型的 block

- _NSConcreteGlobalBlock 全局的静态 block，不会访问任何外部变量。
- _NSConcreteStackBlock 保存在栈中的 block，当函数返回时会被销毁。
- _NSConcreteMallocBlock 保存在堆中的 block，当引用计数为 0 时会被销毁。
- 在ARC开启的情况下，将只会有 NSConcreteGlobalBlock 和 NSConcreteMallocBlock 类型的 block。原本的 NSConcreteStackBlock 的 block 会被 NSConcreteMallocBlock 类型的 block 替代。

### _NSConcreteGlobalBlock
	
- 例子：

	```
	#include <stdio.h>
	
	int main()
	{
	    ^{ printf("Hello, World!\n"); } ();
	    return 0;
	}

	```

### _NSConcreteStackBlock （非ARC情况下）
- 情况1：没有使用`__block`:

	```
	#include <stdio.h>
	
	int main() {
	    int a = 100;
	    void (^block2)(void) = ^{
	        printf("%d\n", a);
	    };
	    block2();
	
	    return 0;
	}
	```
	这时候，变量a将会值拷贝到表示block的结构体中，因此a的值在block中的变动，是不会影响到外部变量。
	
- 情况2：使用了`__block`:
	
	```
	#include <stdio.h>

	int main()
	{
	    __block int i = 1024;
	    void (^block1)(void) = ^{
	        printf("%d\n", i);
	        i = 1023;
	    };
	    block1();
	    return 0;
	}
	```
	
	转换到c++代码后:
	
	```
	struct __Block_byref_i_0 {
	    void *__isa;
	    __Block_byref_i_0 *__forwarding;
	    int __flags;
	    int __size;
	    int i;
	};
	
	struct __main_block_impl_0 {
	    struct __block_impl impl;
	    struct __main_block_desc_0* Desc;
	    __Block_byref_i_0 *i; // by ref
	    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_i_0 *_i, int flags=0) : i(_i->__forwarding) {
	        impl.isa = &_NSConcreteStackBlock;
	        impl.Flags = flags;
	        impl.FuncPtr = fp;
	        Desc = desc;
	    }
	};
	```
	这种情况下，变量i会被装换到一个结构体的一个指针，因此对这个指针指向的值做变动，会影响到外部变量。

### _NSConcreteMallocBlock(调用了copy以后)
- NSConcreteMallocBlock 类型的 block 通常不会在源码中直接出现，因为默认它是当一个 block 被 copy 的时候，才会将这个 block 复制到堆中。
- 在ARC开启的情况下，将只会有 NSConcreteGlobalBlock 和 NSConcreteMallocBlock 类型的 block。原本的 NSConcreteStackBlock 的 block 会被 NSConcreteMallocBlock 类型的 block 替代。




---
[1]: http://blog.devtang.com/2013/07/28/a-look-inside-blocks/