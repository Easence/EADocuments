#贯穿始终-launchd

## launchd
launchd（PID 1）有内核直接启动，是用户态的第一个进程，其他进程都是由它直接或者间接的启动的。其他的launchd（比如其他用户远程登录后会对应创建一个launchd）都是由launchd（PID 1）启动的。
launchd分为两种类型的后台作业：

- **守护程序**（daemon），不可和用户交互。
- **代理程序**（agent），特殊的守护程序，可以和用户交互。

## launchd的职责

### 运行定时作业
指定时间运行指定的命令。

### 启动网络服务
绑定一些端口（UDP端口或TCP端口），当网络请求到达的时候，根据需要启动相应的服务程序，并将服务程序的输入输出描述符（stdin、stderr和stdout）连接到对应的套接字。这样可以降低系统的负载。

### 提供自举服务<servers/bootstrap.h>
- launchd在启动的时候声明一个端口（**bootstrap_port**）,由于所有进程都是launchd的后代，所以所有进程都可以通过这个**bootstrap_port**来访问自举服务器来查询某个服务，并且匹配服务程序的端口。
- 如果想要在自举服务器中注册自己端口的服务程序也可以通过<servers/bootstrap.h>中定义的函数`bootstrap_check_in()`来实现。还可以通过服务程序自己的plist文件来向launcchd注册（可以想象一下Android的Serverice）。

### 事物支持
`vproc_transaction_begi`n和`vproc_transaction_end`之间的操作称为**未决事物**，当一个launchd有未决事物的时候会在系统关闭、用户退出，或超时被优雅的杀掉，否则强制杀掉。

### 资源限制和遏制
iOS Jetsam机制，可以强制施行虚拟内存使用率限制。

### Autorun模拟和文件系统观察
- launchd提供了startOnMount键，当一个文件系统挂载的时候会自动触发一个守护进程。
- 通过WatchPaths或QueueDirectories键，launchd还可以设置一个观察路径，不一定要求是挂载点。

### 整合了I/O Kit

## iOS的launchDeamon
iOS包含的launchDeamon列表如下图所示：
![launchDeamon][1]

**其中最重要的两个守护进程是lockdownd和SpringBoard**

### lockdownd
lockdownd有launchd启动，它负责处理设备激活、备份、崩溃报告、设备同步以及其他的服务。

### SpringBoard
- 创建GUI
- 处理UI，如果SpringBoard停止了所有UI事件都无法到达相应的应用，只有SpingBoard恢复执行的时候，才会将所有排队的UI事件投递到应用程序。
- SpringBoard包含大量的线程，比如：
	- 有Web相关的线程（WebCore和WebThread）
	- WiFiManager
	- CoreAnimation
- SpringBoard通过launchd注册了很多Mach端口，其中最重要的是`PurpleSystemEventPort`，这个端口通过GSEvent消息的方式处理UI事件。SPringBoard的主线程调用GSEventRun(),GSEventRun()是一个处理UI消息的CFRunloop。

## XPC
- XPC是Lion和iOS5以后引入的轻量级的进程间通信原语。XPC和GCD紧密结合在一起。XPC依赖两个私有的框架：XPCService和XPCObjects。前者负责处理XPC服务运行时相关的事务，后者为XPC对象提供编码和解码服务。iOS还包含一个私有框架：XPCKit。常用函数有：

```
xpc_connection_send_message(xpc_connection_t connection, xpc_object_t message); //Send message asynchronously on connection.
```
```
xpc_connection_send_barrier(xpc_connection_t connection, dispatch_block_t barrier); //Execute barrier block after last message is sent on connection.
```
```
xpc_connection_send_message_with_reply(xpc_connection_t connection, xpc_object_t message, dispatch_queue_t replyq, xpc_handler_t handler); //Send message, but also asynchronously execute handler in dispatch queue replyq when a reply is received.
```
```
xpc_object_txpc_connection_send_message_with_reply_sync(xpc_connection_t connection, xpc_object_t message); //Send message, blocking until a reply is received, and return reply as the xpc_ object_t return value
```
- XPC的例子可以参照：苹果官方的[SandboxedFetch][2]


---
[1]: https://github.com/Easence/EADocuments/blob/master/Reading%20Notes/深入解析Mac%20OS%20X%20&%20iOS操作系统/Resources/Images/iOSLaunchDeamon.png?raw=true
[2]: https://developer.apple.com/library/mac/samplecode/SandboxedFetch/Introduction/Intro.html#//apple_ref/doc/uid/DTS40011117-Intro-DontLinkElementID_2















