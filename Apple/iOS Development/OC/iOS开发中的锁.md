# iOS开发中的锁

标签（空格分隔）： iOS 开发

---
`以下的锁耗时从低到高`

- **OSSpinLock（自旋锁）**
自旋锁的原理是，存在竞争关系的线程，在其中一个线程占用资源后，其他线程都处于忙等待状态（消耗CPU资源），直到有资源可用。
应用自旋锁的场景：锁等待时长小于线程切换寄存器时间。（线程执行的内容消耗时间很短的情况下？）
- **dispatch_semaphore**
GCD中使用的信号量机制
- **pthread_mutex**
- **NSLock**
- **NSCondition**
- **pthread_mutex(recursive)**
可递归互斥锁
- **NSRecursiveLock**
可递归锁
- **NSConditionLock**
- **@synchronized**





