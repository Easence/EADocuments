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















