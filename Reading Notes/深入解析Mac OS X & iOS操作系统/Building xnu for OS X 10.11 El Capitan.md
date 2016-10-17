---
title: Building xnu for OS X 10.11 El Capitan
categories: 
 - Apple Development
 - 深入解析Mac OS X && iOS操作系统笔记
 - 内核
tags:
 - 内核
 - 编译
---


*此文只因为国内浏览[ssen's blog][1]需要翻墙，为了方便浏览从中拷贝了一份。*

The OS X kernel source (xnu) has been released for OS X 10.11 El Capitan: [here](https://opensource.apple.com/source/xnu/xnu-3247.1.106/)

Building xnu requires Xcode and some additional open-source (but not pre-installed) dependencies. You can build xnu manually by doing:

1. Install OS X El Capitan and Xcode 7.0, 7.1, or 7.2 from the Mac App Store, make sure the Xcode license has been agreed-to with "sudo xcodebuild -license"
2. Download the source for the dtrace and AvailabilityVersions projects, which are required dependencies, as well as xnu itself

	```
    $ curl -O https://opensource.apple.com/tarballs/dtrace/dtrace-168.tar.gz
    $ curl -O https://opensource.apple.com/tarballs/AvailabilityVersions/AvailabilityVersions-20.tar.gz
    $ curl -O https://opensource.apple.com/tarballs/xnu/xnu-3247.1.106.tar.gz
	```
    
3. Build and install CTF tools from dtrace

	```
    $ tar zxf dtrace-168.tar.gz
    $ cd dtrace-168
    $ mkdir -p obj sym dst
    $ xcodebuild install -target ctfconvert -target ctfdump -target ctfmerge ARCHS="x86_64" SRCROOT=$PWD OBJROOT=$PWD/obj SYMROOT=$PWD/sym DSTROOT=$PWD/dst
    ...
    $ sudo ditto $PWD/dst/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain
    Password:
    $ cd ..
	```    

4. Install AvailabilityVersions

	```
    $ tar zxf AvailabilityVersions-20.tar.gz 
    $ cd AvailabilityVersions-20
    $ mkdir -p dst
    $ make install SRCROOT=$PWD DSTROOT=$PWD/dst
    $ sudo ditto $PWD/dst/usr/local `xcrun -sdk macosx -show-sdk-path`/usr/local
    $ cd ..
	```

5. Build xnu

	```
    $ tar zxf xnu-3247.1.106.tar.gz
    $ cd xnu-3247.1.106
    $ make SDKROOT=macosx ARCH_CONFIGS=X86_64 KERNEL_CONFIGS=RELEASE
	```

**See xnu's top-level README for additional build variables that can be passed on the command-line, such as BUILD\_LTO=0 or KERNEL\_CONFIGS=DEVELOPMENT .**

Update: If you are attempting to add system calls, you may also need to build Libsyscall.

1. Download the Libsystem source

	```
	$ curl -O https://opensource.apple.com/tarballs/Libsystem/Libsystem-1225.1.1.tar.gz
	```

2. Install Libsystem headers

	```
    $ tar zxf Libsystem-1225.1.1.tar.gz
    $ cd Libsystem-1225.1.1
    $ xcodebuild installhdrs -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD OBJROOT=$PWD/obj SYMROOT=$PWD/sym DSTROOT=$PWD/dst
    $ sudo ditto $PWD/dst `xcrun -sdk macosx -show-sdk-path`
    $ cd ..
    ```
3. Install xnu and Libsyscall headers
	
	```
    $ cd xnu-3247.1.106
    $ mkdir -p BUILD.hdrs/obj BUILD.hdrs/sym BUILD.hdrs/dst
    $ make installhdrs SDKROOT=macosx ARCH_CONFIGS=X86_64 SRCROOT=$PWD OBJROOT=$PWD/BUILD.hdrs/obj SYMROOT=$PWD/BUILD.hdrs/sym DSTROOT=$PWD/BUILD.hdrs/dst
    $ sudo xcodebuild installhdrs -project libsyscall/Libsyscall.xcodeproj -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD/libsyscall OBJROOT=$PWD/BUILD.hdrs/obj SYMROOT=$PWD/BUILD.hdrs/sym DSTROOT=$PWD/BUILD.hdrs/dst
    $ sudo ditto BUILD.hdrs/dst `xcrun -sdk macosx -show-sdk-path`
    ```
4. Build Libsyscall

	```
    $ mkdir -p BUILD.libsyscall/obj BUILD.libsyscall/sym BUILD.libsyscall/dst
    $ sudo xcodebuild install -project libsyscall/Libsyscall.xcodeproj -sdk macosx ARCHS='x86_64 i386' SRCROOT=$PWD/libsyscall OBJROOT=$PWD/BUILD.libsyscall/obj SYMROOT=$PWD/BUILD.libsyscall/sym DSTROOT=$PWD/BUILD.libsyscall/dst
    ```
5. To install custom OS components, [System Integrity Protection must be disabled](https://developer.apple.com/library/mac/documentation/Security/Conceptual/System_Integrity_Protection_Guide/ConfiguringSystemIntegrityProtection/ConfiguringSystemIntegrityProtection.html#//apple_ref/doc/uid/TP40016462-CH5-SW1).

6. To install the resulting new binaries, execute:
  	1. xnu: 
	
		```
	    $ sudo cp BUILD/obj/RELEASE_X86_64/kernel /System/Library/Kernels/
	    $ sudo kextcache -invalidate /
	    / locked; waiting for lock.
	    Lock acquired; proceeding.
	    ...
	    $ sudo reboot
   		 ```
  	2. Libsyscall: 
		
		```
        $ sudo cp BUILD.libsyscall/dst/usr/lib/system/libsystem_kernel.dylib /usr/lib/system/
	    $ sudo update_dyld_shared_cache
	    ...
	    $ sudo reboot
    	```

---
[1]: http://shantonu.blogspot.co.uk