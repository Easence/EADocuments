# iOS签名

标签（空格分隔）： iOS 安全

---
[TOC]
## 签名证书（开发者证书）生成过程
- 本地生成CertificateSigningRequest.certSigningRequest（包含`用本地私钥加密的申请者信息`、`公钥`、`摘要算法、非对称加密算法`）。而私钥秘密的保存在本地。
- 苹果拿出CertificateSigningRequest.certSigningRequest里面的`公钥`,并将MC账号的用户信息封装到证书里面。

## 授权描述文件（provisioning profile）
- AppID
- 哪些证书合法
- 哪些设备(UUID)可以运行
- 拥有哪些特权
- 苹果的签名
> 查看mobileprovision文件的方法：
`security cms -D -i embedded.mobileprovision`

##授权文件（entitlements）
- 描述app有哪些功能（如：Push、iCloud等）的文件。
```
$ codesign -d --entitlements - Example.app
```
## app重签流程
- 首先解压ipa
- 如果mobileprovision需要替换，替换
- 
- 如果存在Frameworks子目录，则对.app文件夹下的所有Frameworks进行签名，在Frameworks文件夹下的.dylib或.framework
- 对xxx.app签名(实际上用的是证书对应的私钥进行签名)
- 重新打包

## 签名相关命令
- 解压ipa包
```
unzip -q xxx.ipa -d <destination>
```
- 找出本机可以用来签名的证书信息
```
security find-identity -v -p codesigning
```
- 列出app使用的签名信息
```
codesign -dvvv xxx.app
```
- 查看entitlement.plist
```
$ codesign -d --entitlements - Example.app
```
- 对app重签
```
codesign -fs "iPhone Developer: xxx@xxx.com (Z5YSB6PG5P)" --no-strict xxx.app
```
- 检验签名是否合法
```
codesign -v xxx.app
```
- 重新打包ipa包
```
zip -qry destination source
```
---

参考文章：

1. [漫谈iOS程序的证书和签名机制](https://segmentfault.com/a/1190000004144556)
2. [iReSign](https://github.com/maciekish/iReSign)
3. [re-sign-ios-app](https://gist.github.com/chaitanyagupta/9a2a13f0a3e6755192f7)
4. [iOS Code Signing 学习笔记](http://foggry.com/blog/2014/10/16/ios-code-signing-xue-xi-bi-ji/)






