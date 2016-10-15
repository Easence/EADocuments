---
title: Thrift的二进制数据编解码--以OC为例
date: 2016-04-16 22:56:14
categories: 
 - Apple Development
 - iOS开发笔记
tags:
---

## 什么是Thrift
Thrift是一种接口描述语言和二进制通讯协议，它被用来定义和创建跨语言的服务，这是维基百科的描述。简单来说就是你可以按照Thrift定义语法编写.thrift,然后用Thrift命令行生成各种语言的代码，比如OC、Java、C++、JS，调用这些代码就可以完成客户端与服务器的通信了，不需要自己去写网络请求、数据解析等接口。更多详情可以通过[这里](https://zh.wikipedia.org/wiki/Thrift)了解。

## 为什么使用Thrift
在本人的实际项目中主要考虑到这两个优点：
* RPC。通过简单定义Thrift描述语言文件，使用Thrift -gen命令可以生成多种语言的代码，这些代码包含了网络通信,数据编解码的功能。这就免去了前后台编写这部分繁琐的代码，同时也统一了前后台的实现逻辑。
* Thrift的二进制数据的编码比json更加紧凑、减少了无用的数据的传输。这也是本文讨论的重点。

## Thrift的数据编解码
我们知道json中一个对象类似于这样的：{"key":"content"},但实际上这个对象只有“content”才是我们真正想要的数据，而“key”这个字符串并不是我们实际需要的，只是为了做一个标记，方便我们查找“content”。而Thrift则可以省去“key”这个多余的字符串。这是怎么做到的呢？首先我们在Example.thrift文件中定义了这么一个结构：
```struct ExamConfig
{
	1:required	i64	actionId	 
	2:required  string	value
}```
然后用命令thrift -gen cocoa编译出来的OC代码，我们随便打开一个Thrift编出来的对象可以看到都会有一个write:方法和一个read:方法，例如：
![write:方法][1]
![read:方法][2]

我们看到1中`[outProtocol writeFieldBeginWithName: @"actionId" type: TType_I64 fieldID: 1];`的实现是这样的：
```- (void) writeFieldBeginWithName: (NSString *) name
                            type: (int) fieldType
                         fieldID: (int) fieldID
{
  [self writeByte: CheckedCastIntToUInt8(fieldType)];
  [self writeI16: CheckedCastIntToUInt8(fieldID)];
}```
注意到参数name实际中并没有被使用，只使用了fieldType（数据类型）、fieldID（协议定义中的序号,比如1:required	i64	actionId	 中的1）。再看看2中```[inProtocol readFieldBeginReturningName: &fieldName type: &fieldType fieldID: &fieldID];```的实现：
```- (void) readFieldBeginReturningName: (NSString **) name
                                type: (int *) fieldType
                             fieldID: (int *) fieldID
{
  if (name != NULL) {
    *name = nil;
  }
  int ft = [self readByte];
  if (fieldType != NULL) {
    *fieldType = ft;
  }
  if (ft != TType_STOP) {
    int fid = [self readI16];
    if (fieldID != NULL) {
      *fieldID = fid;
    }
  }
}```
同样的也没有使用到参数name。
我们再看下面这个例子：
假如服务器使用的ExamConfig定义是这样的：
```struct ExamConfig
{
	1:required	i64	actionId	 
	2:required  string	value
}```
而客户端使用的ExamConfig定义是这样的：
```struct ExamConfig
{
	1:required	i64	actionId2	 
	2:required  string	value2
}```
请问服务器发送ExamConfig数据到客户端客户端能正确的解析出数据吗？答案是YES。
## 结论
这就说明了我们定义thrift的结构里的属性名称实际上在thrift数据二进制编解码是被忽略的（thrift的json编解码未验证），这个名称的作用只是作为生成的OC代码类的属性名称。这也解释了为什么Thrift的二进制编码会比平时使用的json更省流量。同时也说明了只要我们在.thrift文件中定义struct的时候保证struct的属性的顺序不变，即使通信双方使用了各自使用不同的属性名称也不会有问题。

---
[1]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Thrift_protocol/1.png?raw=true
[2]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Thrift_protocol/2.png?raw=true