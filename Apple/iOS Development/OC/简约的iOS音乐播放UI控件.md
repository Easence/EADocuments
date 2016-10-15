---
title: 简约的iOS音乐播放UI控件
date: 2016-05-24 18:46:59
categories: 
 - Apple Development
 - iOS开发笔记
tags:
---

## 这是一个什么样的控件
该控件不包含音乐播放的逻辑，单纯的只是UI层面的展示。但可以结合下载、音乐播放逻辑，拼接成一个简易的播放器。代码托管在[Github](https://github.com/Easence/EAMiniAudioPlayerView)上，并支持cocoapods安装。
![效果图][1]

## 主要功能介绍以及使用
- **支持隐藏播放按钮、隐藏声音的波浪图标、以及隐藏文字。**
  > 可以通过设置`EAMiniAudioPlayerStyleConfig`的`playerStyle`属性来实现只展示某部分元素, EAMiniPlayerStyle是一个枚举：

	```
		typedef NS_ENUM(NSUInteger, EAMiniPlayerStyle) {
		EAMiniPlayerNormal = 1 << 0,   //Has play button,sound icon
		EAMiniPlayerHidePlayButton = 1 << 1, //Hide play button
		EAMiniPlayerHideSoundIcon = 1 << 2, //Hide sound icon
		EAMiniPlayerHideText = 1 << 3, //Hide text label
		};
		 EAMiniAudioPlayerStyleConfig *config = [EAMiniAudioPlayerStyleConfig defaultConfig];
		 config.playerStyle |= EAMiniPlayerHidePlayButton;
	```
- **支持下载进度展示。**
实时的设置`EAMiniAudioPlayerView`的`downloadProgress`属性（取值在0和1之间）可以改变下载进度展示，结合下载逻辑可以实现音乐下载的效果。当`downloadProgress`的值达到1的时候会有调用`void(^downloadCompleted)(void)`这个block。

- **支持播放进度展示。**
设置`EAMiniAudioPlayerView`的`playProgress`属性（取值在0和1之间）可以改变播放进度展示，结合音乐播放逻辑可以实现音乐播放的效果。当`playProgress`的值达到1的时候会有调用`void(^playCompleted)(void)`这个block。

- **其他**
自定义圆角、内容的偏移、自定义颜色等。

## 怎么使用
- **使用cocoapods安装：**

	```
	pod install EAMiniAudioPlayerView
	```
## 结尾
这是一个纯粹的UI控件，查看完成的demo可以移步到[这里](https://github.com/Easence/EAMiniAudioPlayerView)。

---
[1]: https://github.com/Easence/EADocuments/blob/master/Apple/iOS%20Development/OC/images/EAMiniAudioPlayer/1.gif?raw=true