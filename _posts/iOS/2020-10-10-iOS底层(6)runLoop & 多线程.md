---
layout: post
title: iOS底层(6)- runLoop & 多线程
date: 2020-10-10
tag: iOS
---

## runloop
### 常见类型
**runloop** 和线程一一对应，子线程默认不开启runloop,当第一次获取runloop时就会创建runloop。存在全局map中，key是线程，value是runloop

runloopModel结构体如下
![1](http://image.smartjames.cn/mweb/20201010/16023184791593.png)

常见2种Mode:
* `kCFRunLoopDefaultMode`（`NSDefaultRunLoopMode`）：App的默认Mode，通常主线程是在这个Mode下运行
* `UITrackingRunLoopMode`：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响

目的是不同组的`Source0`/`Source1`/`Timer`/`Observer`能分隔开来，互不影响，处理不同的事件，只能同一时间运行一种模式

### 线程保活

一般线程启动后，如果事情做完会直接销毁，因为启动runloop 如果Mode里没有任何`Source0`/`Source1`/`Timer`/`Observer`，RunLoop会立马退出。 唯一办法是在runloop中添加前面所说的几个事件。
做法：
在当前线程中调用以下代码
```
[[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init]   forMode:NSDefaultRunLoopMode];
while (1){
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
}
```
这样添加一个port的 `source1` 事件，并启动本次runMode, 一旦响应一次事件后本次会结束，所以需要while 运行runModel 。其实就是阻塞了当前runloop的一个事件，导致后续事件无法运行，以此保证runloop和线程的存活


## 多线程
![2](http://image.smartjames.cn/mweb/20201010/16023185008204.png)

### 线程死锁
**使用`sync`函数往当前串行队列中添加任务，会卡住当前的串行队列（产生死锁)**
`performSelector:withObject:afterDelay: `方法是用你NSTimer实现定时，且是加在当前线程runloop中，但是如果当前是`asyn`产生的线程runloop没启动的话这种方法无效。需要手动启动
```
[[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
```
### 线程锁

* `OSSpinLock` 自旋锁，一直处于忙等状态
* `os_unfair_lock` 互斥锁，等待锁的线程处于休眠状态，节省资源

![3](http://image.smartjames.cn/mweb/20201010/16023185133777.jpg)

* `pthread_mutex` 互斥锁，等待锁的线程处于休眠状态
![4](http://image.smartjames.cn/mweb/20201010/16023185345005.jpg)

* `pthread_mutex`的条件锁
![5](http://image.smartjames.cn/mweb/20201010/16023185451423.jpg)

* `NSLock`  对于pthread_mutex的OC封装
* `NSRecursiveLock`  对于pthread_mutex的OC封装
* `NSCondition`  对于pthread_mutex 和 condition的OC封装
* `dispatch_queue(DISPATCH_QUEUE_SERIAL)` 把不同线程的任务放在同步队列中执行，能达到锁的效果
* `dispatch_semaphore` 信号量， 初始化赋一个最多同时执行线程数量，如果是1就实现了锁的效果

锁都是由于不同线程的问题导致的，如果条件锁在一个线程有wait，没有其他线程signal,就会永远睡眠下去。wait后的代码不会执行。


**队列和线程是2个概念，队列中的任务可以在不同的线程中去执行。例如用globalQueue来创建不同的线程任务，所以可以使用串行队列来实现线程锁的问题，只要保证任务都加入到队列中执行即可**

atomic 用于保证setter、 getter的操作原子性，在方法内部加上线程同步锁

### 读写安全方案 ,保证同一时间只有一个线程写，可以多个线程读，不能同时读写
* `pthread_rwlock` 读写锁
* `dispatch_barrier_async` 异步栅栏调用
    `dispatch_asyn(que)` 用于读操作
    `diapatch_barrier_async(que)` 用于写操作，可以保证当前队列中只有栅栏中的任务在执行，其他全部暂停。
    