# 创建pod库的步骤

标签（空格分隔）： 开发 iOS
---
##注册Trunk
```
pod trunk register xxxxx@gmail.com 'xxxxx'
```
##验证邮箱
```
pod trunk me
```
##push源码到git仓库
##创建podspec文件
```
pod spec create https://github.com/Easence/EAMiniAudioPlayerView.git
```
##编辑podspec文件
##检查podspec文件格式是否符合规则
```
pod lib lint --no-clean
```
成功后的信息：`EAMiniAudioPlayerView passed validation.`
##使用pod trunk push命令把刚才创建的podspec文件推送到Github的specs repo远程库
```
pod trunk push EAMiniAudioPlayerView.podspec
```

##成功后`pod setup`
##验证是否成功：
```
pod search EAMiniAudioPlayerView
```



