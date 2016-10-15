---
title: iOS中的MAX(A,B)，需要注意的点
date: 2016-04-19 11:22:30
categories: 
 - Apple Development
 - iOS开发笔记
tags:
---

## 问题由来
今天有朋友在使用MAX(A,B)的时候出现了一个诡异的问题,代码是这样的：
![代码1][1]

而执行的结果竟然是这样的：
![结果1][2]

“我是不是眼花了？max(-1,0)返回了-1？”我的朋友惊讶到。这不科学啊，怎么会负数比0大呢？于是我查看了MAX(A,B)的源码：
![MAX源码][3]

## 验证过程
然后我做了如下两个实验（请注意调试区a的类型）：

### 实验1：（a的类型为unsigned long）
![实验1][4]

### 实验2：（a的类型为int）
![实验2][5]

通过这两个实验我们可以发现：由于NSArray的count属性是个NSUInteger类型，因此__typeof_(array.count - 1) 会得到一个无符号的类型，当array.count - 1为负数的时候，就相当于(NSUInteger)(负数) = 正数，因此会有MAX(array.count - 1, 0) = (NSUInteger)（array.count - 1）。如下图所示：
![结果][6]

## 结论：
当使用MAX(A,B)取大值的时候要注意A、B是否是无符号型，可以统一将它们转换成有符号型来进行比较。如下图：
![结论][7]

---
[1]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/1.jpeg?raw=true
[2]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/2.jpeg?raw=true
[3]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/3.png?raw=true
[4]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/4.jpeg?raw=true
[5]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/5.png?raw=true
[6]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/6.png?raw=true
[7]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/Max(A,B)/7.png?raw=true