---
layout: post
title: NSNotification详解
date: 2020-12-29
tag: iOS
---

### 实现原理
参考GNUStep源码，了解其中实现原理，对于`addObserver:selector:name:object:`会创建一个`Observation`对象，是搜索和执行响应的核心对象， 包含observer和sel， 

```
typedef	struct	Obs {
  id		observer;	/* Object to receive message.	*/
  SEL		selector;	/* Method selector.		*/
  struct Obs	*next;		/* Next item in linked list.	*/
  int		retained;	/* Retain count for structure.	*/
  struct NCTbl	*link;		/* Pointer back to chunk table	*/
} Observation;
```

`NSNotification`维护了一个MapTable表，用于分类存储`Observation`，分为`nameless`,`named`,`cache`，代表无post名的通知，有名的通知，和快速缓存。

```
#define	CHUNKSIZE	128
#define	CACHESIZE	16
typedef struct NCTbl {
  Observation		*wildcard;	/* Get ALL messages.		*/
  GSIMapTable		nameless;	/* Get messages for any name.	*/
  GSIMapTable		named;		/* Getting named messages only.	*/
  unsigned		lockCount;	/* Count recursive operations.	*/
  NSRecursiveLock	*_lock;		/* Lock out other threads.	*/
  Observation		*freeList;
  Observation		**chunks;
  unsigned		numChunks;
  GSIMapTable		cache[CACHESIZE];
  unsigned short	chunkIndex;
  unsigned short	cacheIndex;
} NCTable;
```

但是`nameless`和`named`都是MapTable,但是存储的结构不同，即若有名字，则存储的是同名的table,其中分为不同的object对于不同的observation。若没有名字，则用object来区分不同的observation。
所以我们知道，即使不传name,传入指定的obj,也可以实现通知的功能。

```
在nameless表中：
GSIMapTable的结构如下
object : Observation
object : Observation
object : Observation

----------------------------
在named表中：
GSIMapTable结构如下:
name : maptable
name : maptable
name : maptable

maptable的结构如下
object : Observation
object : Observation
object : Observation
```

addObserver的逻辑如下：
- 根据传入的selector和observer创建Observation，并存入maptable中，如果已存在，则是从cache中取。
- 如果name存在，object存在，则向named的maptable表中插入元素，key为name,value为GSIMapTable，GSIMapTable中存key为object,value为Observation。
- 如果name不存在，object存在，则向nameless的maptable表中插入元素，key为object,value为Observation。
- 如果name和object都为空，则Observation->next=wildcard，将老的wildcard赋值为next指针,然后Observation对象置为wildcard，wildcard = Observation

postNotification的机制如下：
- 查找所有wildcard的Observation,加入数组
- 查找相同object和空name的Observation,加入数组
- 查找相同name和相同object的Observation,加入数组
- 查找相同name,非空object时,所有nil object的Observation加入数组
- 遍历数组,执行performSelector,情况数组

### 通知是同步or异步
`postNotificationName`底层实现是`performSeletor` 方法，所以一定是按顺序执行，是同步的。

### 如何修改发送时机
使用`NSNotificationQueue`, 顾名思义可以将NSNotification放入queue中，在适当时机再执行发送，虽然都是在主线程。

```
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1,      // 当runloop处于空闲状态时post
    NSPostASAP = 2,    // 当当前runloop完成之后立即post
    NSPostNow = 3    // 立即post
};
```

```
NSNotification *noti = [NSNotification notificationWithName:@"111" object:nil];
[[NSNotificationQueue defaultQueue] enqueueNotification:noti postingStyle:NSPostASAP];
```

### 和runloop的关系
与runloop的关系仅限当postingStyle不是`NSPostNow`，即需要在runloop一定的状态下再去发送，默认子线程是不启动runloop的，所以如果要在子线程使用`NSPostNow`以外的类型发送通知，需要启动runloop。

### 通知的线程切换
在上面也已经聊过，`NSNotification`执行是通过performSeletor方法，所以默认就是在当前线程，但是实际上我们会一个需求，在子线程发送通知，在主线程接收到通知并执行，虽然我们可以子线程接收通知后，切换到主线程执行相应的UI操作，但是如果多个接收者，都需要这种操作，就很不友好。下面来看一下，苹果官方给出的解决方案。


```
@interface NotificationVC ()<NSMachPortDelegate>

@property (nonatomic, strong)NSLock *notiLock;
@property (nonatomic, strong)NSThread *notiThead; // 期望线程
@property (nonatomic, strong)NSMutableArray *notiQueue;
@property (nonatomic, strong)NSMachPort *notiMach;

@end

@implementation NotificationVC
- (void)viewDidLoad {
    [super viewDidLoad];
    self.view.backgroundColor = [UIColor systemBackgroundColor];
    
    self.notiLock = [[NSLock alloc]init];
    self.notiQueue = [NSMutableArray array];
    self.notiMach = [[NSMachPort alloc] init];
    self.notiMach.delegate = self;
    self.notiThead = [NSThread currentThread];
    [[NSRunLoop currentRunLoop] addPort:self.notiMach forMode:NSRunLoopCommonModes];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(processNotification:) name:@"post" object:nil];

    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSLog(@"%@", [NSThread currentThread]);
        [[NSNotificationCenter defaultCenter] postNotificationName:@"post" object:nil];
    });
}


- (void)processNotification:(NSNotification *)noti
{
    if ([NSThread currentThread] != self.notiThead) {
        NSLog(@"不在主线程  %@",[NSThread currentThread]);
        [self.notiLock lock];
        [self.notiQueue addObject:noti];
        [self.notiLock unlock];
        [self.notiMach sendBeforeDate:[NSDate date] components:nil from:nil reserved:0];
    }else{
        NSLog(@"已经在主线程 %@",[NSThread currentThread]);
    }
}


#pragma mark - NSMachPortDelegate
- (void)handleMachMessage:(void *)msg
{
    NSLog(@"%s   %@",__func__,[NSThread currentThread]);
    
    [self.notiLock lock];
    while ([self.notiQueue count]) {
        NSNotification *noti = self.notiQueue[0];
        [self.notiQueue removeObjectAtIndex:0];
        [self.notiLock unlock];
        [self processNotification:noti];
        [self.notiLock lock];
    }
    
    [self.notiLock unlock];
}
```

以上方案有很大局限性，每个observer都要讲方法设为processNotification：，而且每个观察者都要实现这套逻辑。完美方案是实现NSNotificationCenter子类或者写单独的队列类来管理。


简单方案如下，在需要切换到主线程的观察者地方，使用`addObserverForName:object:queue:usingBlock:`方式，将响应的回调拉回到主线程。

```
[[NSNotificationCenter defaultCenter] addObserverForName:@"post" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
    NSLog(@"%@", [NSThread currentThread]);
}];
```