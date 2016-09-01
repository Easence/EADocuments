# 引导过程：EFI和iBoot

标签（空格分隔）： 《OSX&iOS内核》 读书笔记

---
[TOC]

##什么是引导过程
引导过程指的是：从计算机通电的那一瞬间到CPU开始执行操作系统代码时的整个过程。整个过程大概是这样子的：

1. 刚通电的时候，BIOS或者固件会加载一些自举代码（bootstrap）给CPU，这些代码会探测各种存在设备。
2. 接着寻找引导磁盘，在引导磁盘的活动分区的第一个扇区找到`操作系统加载器`。由于BIOS的局限性，只支持一个操作系统的加载，如果需要安装多个操作系统，则要接住第三方的引导加载（如：GRUB），来提供操作系统的选择菜单。
3. 由操作系统加载器加载操作系统代码。

## EFI（Extensible Firmware Interface，可扩展固件接口）

### BIOS的局限性
- BIOS只能访问1M的内存。例如：在启动windows的时候，会发现开始的时候分辨率极低，在windows的logo出来以后才恢复正常。
- 只允许4个可引导分区（或称为主分区）。

### EFI的概念
正因为BIOS的局限性，EFI应运而生。EFI是一套接口，它规范了一组应用程序编程的接口。基于EFI开发的程序通常都是引导加载器（如：基于Linux的GRUB，苹果的boot.efi和Boot Camp），但也可以是一些诊断程序（如：苹果的硬件测试工具），甚至还可以是编译时链接了EFI API的普通程序。
> EFI自下而上的架构大概是：硬件->固件（包含：EFI引导服务和EFI运行时服务）->EFI二进制文件（EFI引导加载器）->软件

### EFI提供的服务

#### EFI引导服务
当系统仍然在EFI环境中，并且没有调用`ExitBootServices()`这个特殊的函数之前，可以访问引导服务。它提供了对内存、硬件的访问，还支持加载EFI程序，此时的资源都被认为归固件所有。一旦调用了`ExitBootServices()`,引导服务就无法访问了。

对于硬件的访问，EFI定义了协议的概念，每个协议都是一个唯一的128位的GUID，它封装了和某个特定设备或某一类设备相关的API。这些协议有：

- **控制台协议**（控制负责用户输入输出的设备，如：键盘、鼠标、串口、屏幕以及其他一些更复杂的设备）
- **媒介访问**（和文件系统打交道）
- **杂项协议**（）

#### EFI运行时服务
跟引导服务一样可以运行在EFI模式下，但是跟引导不同之处在于，运行时服务在退出EFI模式依然可以使用。不过运行时服务不能提供引导服务所提供的各种服务。只包含一下这些服务：

- 时间管理
- 闹钟
- 固件变量
- 其他杂项

## OS X的boot.efi
1. 调用EFI引导服务来完成获取设备树、画logo、加载链接好的内核（loadKernelCache）、加载RAMDisk到内存、跳转到内核入口点等工作。
2. 引导内核，EFI引导服务退出。
3. 内核回调EFI运行时服务。

## iOS的iBoot
iOS的引导是苹果独创的，构成如下图所示，起引导过程分两条主线：

- **普通引导、恢复模式引导**
- **DFU模式引导**

![图1][1]

整个引导过程大概是这样的：

1. 首先加载Boot ROM。(只有这一步骤是未加密的，其他的步骤是都是加密的)
2. 接着判断是否是DFU模式，是则跳到步骤4，否则跳到步骤3。
3. 普通引导或恢复模式引导：
	- 加载LLB（Low level Bootloader，底层引导加载器。
	- 加载iBoot这个主引导加载器，它负责定位、准备并加载kernelCache（链接好的内核）。 
4. DFU模式引导，使用了两个镜像iBSS和iBEC。
	- iBSS：负责底层初始化以及iBEC的加载。
	- iBEC：负责iTunes通过USB升级的过程。

### 普通引导或恢复模式引导
#### LLB
它是iOS镜像的一部分，可以在镜像中的Firmware/all_flash.xxxlp.production/下找到一个名为LLB.xxx.RELEASE.img3的文件。
#### iBoot
iBoot自带一个内建的HFS+的驱动程序，可以访问iOS的文件系统，iBoot是多线程的，通常至少派生两个线程：

- **“main”线程**，负责苹果的logo，以及系统的引导。
- **“uart reader”线程**，这个线程用户调试用。

正常情况下，iBoot会调用fsboot()函数，这个函数会挂在iOS文件系统分区，定位内核，准备设备树并引导系统。如果引导失败（或终止），iBoot进入**恢复模式**，main线程会派生几下几个任务：

- **idleoff任务**，当用户不操作时，关闭设备。
- **poweroff任务**，当电量不足的时候，关闭设备。
- **usb-req任务**，处理iTunes的USB请求。
- **usb-high-current和usb-no-current任务**，响应USB充电。（当设备充电或者断开充电，修改电池图标）
- **command任务**,启动命令行接口，即通过串口操作的控制台。

#### 恢复模式引导与普通引导的区别
恢复模式与普通引导的区别在于，恢复模式使用的文件系统是ramdisk，而不是使用包含了标准的iOS镜像的flash文件系统。它是一个完整的内存文件系统，可以用来替换根文件系统，并且flash文件系统可以挂载到这个文件系统，可以修改或更新文件系统。可以在iOS的镜像（ipsw）中找到。通常是iOS更新的第3个dmg。

### DFU模式引导
再这个模式中，NAND闪存中的固件本省会被更新。当iOS有新版本更新或者越狱的时候会使用到这个模式。

## OS X的安装过程
### 步骤1：installXXX.app
安装包包含的文件如下图所示：
![图2][2]
运行这个app之后，会展示一个GUI界面，收集一些用户的输入信息之后，将kernelcache、boot.efi和InstallESD.dmg这些文件拷贝到/Mac OS X Install Data这个特殊目录下，然后告诉内核挂载InstallESD.dmg作为容器镜像，其目的是为了找到用作`根文件系统`的镜像--BaseSystem.dmg，然后通过`bless`命令修改引导盘，使得系统从InstallESD.dmg引导。操作成功后，系统自动重启至新的镜像。

### 步骤2：OSInstaller
引导进入新系统，运行对应的kernelcache后，镜像会让luanchd运行OSInstaller，OSInstaller会从minstallconfig.xml获取安装数据，并执行diskmanagementd，重新将所有需要的磁盘分区，接着会准备一个恢复卷，其实就是：BaseSystem.dmg。
### 步骤3：安装.pkg文件
最后就是安装各种各样的软件包了。

## iOS文件系统镜像（.ipsw文件）


---
[1]: https://github.com/Easence/EADocuments/blob/master/Reading%20Notes/深入解析Mac%20OS%20X%20&%20iOS操作系统/Resources/Images/The%20iOS%20Boot%20Progress.png?raw=true
[2]: https://github.com/Easence/EADocuments/blob/master/Reading%20Notes/深入解析Mac%20OS%20X%20&%20iOS操作系统/Resources/Images/OSX_Installer_files.png?raw=true








