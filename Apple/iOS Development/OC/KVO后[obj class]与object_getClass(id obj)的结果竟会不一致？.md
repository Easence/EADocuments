# KVO后[obj class]与object_getClass(id obj)的结果竟会不一致？

标签（空格分隔）： iOS 开发

---
[TOC]

## 遇到的问题
在做iOS项目过程中，一次偶然的机会发现`object_getClass(id obj)`返回的结果是`NSKVONotifing_ObjectClass`,`[obj class]`返回的结果却是`ObjectClass`,它们的结果竟会不一致？      
这不科学啊。因为实例的`class方法`，底层实际上就是调用runtime的`object_getClass(id obj)`方法实现的，所以正常的情况下这两个方法返回的结果应该是一致的。那是什么原因导致的呢？

##KVO是元凶
-  从`object_getClass(id obj)`返回的结果`NSKVONotifing_ObjectClass`我们可以猜测，obj可能使用了KVO。那么我们要知道问题出在什么地方，就可以从KVO的实现原理入手。
-  KVO的原理
> 其实也简单，就是利用OC的runtime特性，更改了isa只能只指向类，以下三步是必不可少的：
1. 比如原先实例obj的isa指针指向的是ObjectClass，那么当你在第一次调用过`[obj addObserver:self forKeyPath:@“propertyA” options:context:]`方法后，runtime会创建一个新的类，类名以NSKVONotifing开头叫NSKVONotifing_ObjectClass，同时更改实例obj的isa的指针，将其指向NSKVONotifing_ObjectClass。
2. 在NSKVONotifing_ObjectClass中重写观察的属性propertyA的setter：`setProperA`，并在它里面调用 `- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context`方法。这样当改变属性propertyA的值时，外面就会得到通知。
3. 在NSKVONotifing_ObjectClass中重写`- (Class) class`方法，返回原先的isa指向的类（在这个例子中就是ObjectClass）。这就是为什么`[obj class]`与`object_getClass(id obj)`返回的结果不一致的原因。可以通过下面的的方法打印出NSKVONotifing_ObjectClass的方法列表加以证明：
```
- (void)printMethodList
{
    Class cls =  object_getClass(self);
    unsigned int outCount;
    Method* methods = class_copyMethodList(cls,&outCount);
    
    for (int i = 0; i < outCount ; i++)
    {
        SEL name = method_getName(methods[i]);
        NSString *strName = [NSString  stringWithCString:sel_getName(name) encoding:NSUTF8StringEncoding];
        NSLog(@"selName : %@",strName);
    }

}
```
## 总结
经过研究这个问题，可以得到以下几个要点：
1. 查看OC Runtime可以知道，`[obj class]`的底层实现实际是： 
```
- (Class) class {
    return object_getClass(self);
}
```
,因此正常情况下`[obj class]`与`object_getClass(obj)`返回的结果应该是一致的。
2. 当使用KVO时，OC Runtime会改变isa，并重写了class方法。
3. 当发现`[obj class]`，`object_getClass(obj)`,两者结果不一致的时候，就要想到是不是有地方更改了`- (Class) class`的实现。









