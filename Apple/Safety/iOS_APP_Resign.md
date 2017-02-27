# 重签iOS应用的方法

## 使用fastlane的sigh重签

安装好brew，先用brew安装ruby，然后用gem安装sigh。

1. `brew install ruby`
2. `sudo gem install sigh`

使用就非常简单了：

1. 输入sigh resign，回车
2. 把要签名的ipa文件拖到窗口上，回车
3. 填写用来签名的证书，回车
4. 把embedded.mobileprovision文件拖到窗口上，回车
5. 好了，resign脚本会自动更改bundel id，签名并重新打包。

如果像是微信那种带多targets的应用，可以直接调用resigh.sh进行签名：
`./resign.sh YourApp.ipa "iPhone Distribution: YourCompanyOrDeveloperName" -p "bundel id"=<path_to_provisioning_profile_for_app>.mobileprovision -p "bundel id"=<path_to_provisioning_profile_for_watchkitextension>.mobileprovision -p "bundel id"=<path_to_provisioning_profile_for_watchkitapp>.mobileprovision -p "bundel id"=<path_to_provisioning_profile_for_todayextension>.mobileprovision  resignedYourApp.ipa`

我详细举个例子说明吧，重签名一个叫乐动力的应用，里面包含一个XQTodayExtension.appex的通知栏插件，我们来看怎么签名：

1. 先去导出两个mobileprovision文件，分别是应用和Plugin的，这里我导出了1. mobileprovision和2. mobileprovision，分别对应com.fenzi.xiaoqin和com.fenzi.xiaoqin.XQTodayExtension。
2. 在1.4这个版本的sigh里，resigh.sh的位置是/usr/local/lib/ruby/gems/2.3.0/gems/sigh-1.4.0/lib/assets/resign.sh，运行resign.sh进行签名：

`resign.sh /Users/Dylan/Code/LDL/xiaoqin.ipa "iPhone Distribution: YourCompanyOrDeveloperName" -p com.fenzi.xiaoqin=/Users/Dylan/Code/LDL/1.mobileprovision -p com.fenzi.xiaoqin.XQTodayExtension=/Users/Dylan/Code/LDL/2.mobileprovision /Users/Dylan/Code/LDL/xiaoqin2.ipa
`
保存下来的xiaoqin2.ipa就是重签之后的文件。如果有苹果手表的文件，也同理处理。

sign脚本还有很多实用的功能，比如直接申请ADHOC签名证书，申请Developent签名证书等等。
而sign脚本是fast lane系列工具中的一个，有兴趣可以研究下，功能非常强大。

![fastlane](http://7xibfi.com1.z0.glb.clouddn.com/uploads/default/original/2X/b/b39d6a1fe64032a75182fdded355c9694f122772.png)

## 利用 iOS App Signer重签
1. 原理参见：[非越狱环境下从应用重签名到微信上加载Cycript][1]
2. [ios-app-signer][2]

[1]: http://www.jianshu.com/p/262b9849fa10
[2]: https://github.com/Easence/ios-app-signer


