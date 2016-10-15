---
title: iOS引导页的镂空效果
date: 2016-04-29 22:06:55
categories: 
 - Apple Development
 - iOS开发笔记
tags:
---

## 初衷
最近项目新功能更改较大，产品童鞋要求加入新功能引导，于是一口气花了两天的时间做了一个引导页，当然加上后面的修修补补的时间，就不只两天了，不过这事情其实是一劳永逸的事情，值得做。同时为了能够更好的复用，我把它做成了pod库，项目地址在这里：[EAFeatureGuideView](https://github.com/Easence/EAFeatureGuideView)。
## EAFeatureGuideView能做什么
EAFeatureGuideView是UIView的一个扩展，用来做新功能引导提示，达到这样的效果：
- 局部区域高亮（可以设置圆角）
- 有箭头指向高亮区域
- 可以设置一段介绍文字（可以是图片、也可以是文字）
- 可以对应一个按钮，可以通过配置事件、标题。
最后的效果如下：
![效果图1][1]
![效果图2][2]

## 如何使用
如果安装了Cocoapods,可以在Podfile中加入如下代码：

```
pod 'EAFeatureGuideView
pod install
```
接着在需要展示提示的页面引入头文件：

```
#import "UIView+EAFeatureGuideView.h"
```
最后添加如下代码：
```
EAFeatureItem *item = [[EAFeatureItem alloc] initWithFocusView:self.exampleCell focusCornerRadius:0 focusInsets:UIEdgeInsetsZero];
item.introduce = @"txt_feature_post_activity_4.1.png";
item.actionTitle = @"太好了";
item.action = ^(id sender){
        NSLog(@"touched ..");  
    };

EAFeatureItem *recents = [[EAFeatureItem alloc] initWithFocusRect:CGRectMake(centerX - 25, centerY - 25, 50, 50) focusCornerRadius:25 focusInsets:UIEdgeInsetsZero];    
recents.introduce = @"recents";

[self.navigationController.view showWithFeatureItems:@[item, recents] saveKeyName:@"keyName" inVersion:nil];
```
## 可以优化的地方
- 介绍文案没有支持多颜色。
- 当高亮区域是圆形的时候，箭头的指向没有对中圆心。

---
[1]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/EAFeatureGuideView/1.png?raw=true
[2]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/EAFeatureGuideView/2.png?raw=true