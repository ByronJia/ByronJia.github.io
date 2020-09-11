---
layout: post
title: Flutter渲染机制—UI线程(一)
date: 2020-03-02
tags: Flutter
---


## 一、UI线程渲染
Flutter是谷歌开源的移动UI框架，可以快速在Android和iOS上构建出高质量的原生用户界面，目前全世界越来越多的开发者加入到Flutter的队伍。 Flutter相比RN性能更好，由于Flutter自己实现了一套UI框架，丢弃了原生的UI框架，非常接近原生的体验。

为了揭秘Flutter高性能，本文从源码角度来看看Flutter的渲染绘制机制，跟渲染直接相关的两个线程是UI线程和GPU线程：

- UI线程：运行着UI Task Runner，是Flutter Engine用于执行Dart root isolate代码，将其转换为layer tree视图结构；
- GPU线程：该线程依然是在CPU上执行，运行着GPU Task Runner，处理layer tree，将其转换成为GPU命令并发送到GPU。

通过VSYNC信号使UI线程和GPU线程有条不紊的周期性的渲染界面，本文介绍VSYNC的产生过程、UI线程在引擎和框架的绘制工作，下一篇文章会介绍GPU线程的绘制工作。

### 1.1 UI渲染原理
#### 1.1.1 UI渲染概览
通过VSYNC信号使UI线程和GPU线程有条不紊的周期性的渲染界面，如下图所示：
![](http://gityuan.com/img/flutter_ui/flutter_draw.png)

- 当需要渲染则会调用到Engine的ScheduleFrame()来注册VSYNC信号回调，一旦触发回调doFrame()执行完成后，便会移除回调方法，也就是说一次注册一次回调；
- 当需要再次绘制则需要重新调用到ScheduleFrame()方法，该方法的唯一重要参数regenerate_layer_tree决定在帧绘制过程是否需要重新生成layer tree，还是直接复用上一次的layer tree；
- UI线程的绘制过程，最核心的是执行WidgetsBinding的drawFrame()方法，然后会创建layer tree视图树
- 再交由GPU Task Runner将layer tree提供的信息转化为平台可执行的GPU指令。

#### 1.1.2 UI绘制核心工作
1）Vsync单注册模式：保证在一帧的时间窗口里UI线程只会生成一个layer tree发送给GPU线程，原理如下：

Animator中的信号量pending_frame_semaphore_用于控制不能连续频繁地调用Vsync请求，一次只能存在Vsync注册。 pending_frame_semaphore_初始值为1，在Animator::RequestFrame()消费信号会减1，当而后再次调用则会失败直接返回； Animator的BeginFrame()或者DrawLastLayerTree()方法会执行信号加1操作。

2）UI绘制最核心的方法是drawFrame()，包含以下几个过程：

- Animate: 遍历_transientCallbacks，执行动画回调方法；
- Build: 对于dirty的元素会执行build构造，没有dirty元素则不会执行，对应于buildScope()
- Layout: 计算渲染对象的大小和位置，对应于flushLayout()，这个过程可能会嵌套再调用build操作；
- Compositing bits: 更新具有脏合成位的任何渲染对象， 对应于flushCompositingBits()；
- Paint: 将绘制命令记录到Layer， 对应于flushPaint()；
- Compositing: 将Compositing bits发送给GPU， 对应于compositeFrame()；
- Semantics: 编译渲染对象的语义，并将语义发送给操作系统， 对应于flushSemantics()。

UI线程的耗时从doFrame(frameTimeNanos)中的frameTimeNanos为起点，以小节[4.10.6]Animator::Render()方法结束为终点， 并将结果保存到LayerTree的成员变量construction_time_，这便是UI线程的耗时时长。

#### 1.1.3 Timeline说明
3）以上几个过程在Timeline中ui线程中都有体现，如下图所示：
![](http://gityuan.com/img/flutter_ui/timeline_ui_draw.png)
另外Timeline中还有两个比较常见的标签项

- “Frame Request Pending”：从Animator::RequestFrame 到Animator::BeginFrame()结束；
- ”PipelineProduce“： 从Animator::BeginFrame()到Animator::Render()结束。

### 1.2 UI线程渲染流程图
#### 1.2.1 VSYNC注册流程
![](http://gityuan.com/img/flutter_ui/Vsync.jpg)
当调用到引擎Engine的ScheduleFrame()方法过程则会注册VSYNC信号回调，一旦Vsync信号达到，则会调用到doFrame()方法。 对于调用ScheduleFrame()的场景有多种，比如动画的执行AnimationController.forward()，再比如比如surface创建的时候shell::SurfaceCreated()。
#### 1.2.2 Engine层绘制
![](http://gityuan.com/img/flutter_ui/UIDraw_engine.jpg)

doFrame()经过多层调用后通过PostTask将任务异步post到UI TaskRunner线程来执行，最后调用到Window的BeginFrame()方法。

#### 1.2.3 Framework层绘制
![](http://gityuan.com/img/flutter_ui/UIDraw_fwk.jpg)

其中window.cc中的一个BeginFrame()方法，会调用到window.dart中的onBeginFrame()和onDrawFrame()两个方法。

### 1.3 核心类图
![](http://gityuan.com/img/flutter_ui/ClassEngine.jpg)

为了让大家更容易理解源码，先看一张关于Shell、Engine、Animator等Flutter等Flutter引擎中核心类的类图。

- Window类：是连接Flutter框架层(Dart)与引擎层(C++)的关键类，在框架层中window.dart文件里的一些方法在引擎层的window.cc文件有相对应的方法，比如scheduleFrame()方法。 在window.cc里面通过Window::RegisterNatives()注册了一些框架层与引擎层的方法对应关系；
- RuntimeController类：可通过其成员root_isolate_找到Window类；
- Shell类：同时继承了PlatformView::Delegate，Animator::Delegate，Engine::Delegate，所以在Engine，Animator，PlatformView中的成员变量delegate_都是指Shell对象， 从图中也能看出其中心地位，代理多项业务，该类是由AndroidShellHolder过程中初始化创建的；另外Shell类还继承了ServiceProtocol::Handler，图中省略而已。
- PlatformViewAndroid类：在Android平台上PlatformView的实例采用的便是PlatformViewAndroid类。
- Dart层与C层之间可以相互调用，从Window一路能调用到Shell类，也能从Shell类一路调用回Window。


