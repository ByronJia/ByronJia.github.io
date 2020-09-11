---
layout: post
title: Flutter渲染机制—UI线程(二)
date: 2020-03-12
tags: Flutter
---

## 二、 VSYNC注册流程
### 2.1 Engine::ScheduleFrame
[-> flutter/shell/common/engine.cc]
```
void Engine::ScheduleFrame(bool regenerate_layer_tree) {
    //[见小节2.2]
    animator_->RequestFrame(regenerate_layer_tree);
}
```
该方法说明：
- animator_的赋值过程是在Engine对象初始化过程完成，而Engine初始化过程在Shell创建过程，此处animator_便是Animator对象；
- ScheduleFrame的参数regenerate_layer_tree决定是否需要重新生成layer tree，还是直接复用上一次生成的layer tree；
- 绝大多数情况下，调用RequestFrame()时将regenerate_layer_tree_设置为true或者用默认值true，执行完Animator::BeginFrame()则设置该变量为false；
    - 当无参数调用该方法时，regenerate_layer_tree为默认值为true。
    - 特别的例子就是Shell::OnPlatformViewMarkTextureFrameAvailable()过程，设置参数为false，那么计划绘制一帧的时候就不需要重绘layer tree；

### 2.2 Animator::RequestFrame
[-> flutter/shell/common/animator.cc]

```
void Animator::RequestFrame(bool regenerate_layer_tree) {
  if (regenerate_layer_tree) {
    // regenerate_layer_tree_决定Vsync信号到来时，是否执行BeginFrame
    regenerate_layer_tree_ = true;
  }

  //当调用Animator::Stop()则会停止动画绘制
  if (paused_ && !dimension_change_pending_) {
    return;
  }

  //调用sem_trywait来保证不会同时有多个vsync请求
  if (!pending_frame_semaphore_.TryWait()) {
    return;
  }

  task_runners_.GetUITaskRunner()->PostTask([self = weak_factory_.GetWeakPtr(),
                                             frame_number = frame_number_]() {
    if (!self.get()) {
      return;
    }
    TRACE_EVENT_ASYNC_BEGIN0("flutter", "Frame Request Pending", frame_number);
    self->AwaitVSync();  // [见小节2.3]
  });
  frame_scheduled_ = true;  //标注已经schedule绘画帧
}
```
过程说明：

- pending_frame_semaphore_：非负信号量，初始值为1，第一次调用TryWait减1，而后再次调用则会失败直接返回。当消费了这次vsync回调，也就是调用了Animator的BeginFrame()或者DrawLastLayerTree()方法后，改信号量会加1[见小节3.6]，可以再次执行vysnc的注册；
- 通过Animator的Start()或者BeginFrame调用到的RequestFrame方法，则肯定需要重新生成layer tree；通过Engine的ScheduleFrame方法是否重建layer tree看小节2.1；
- 此处通过post把Animator::AwaitVSync任务放入到UI Task Runner来执行。

### 2.3 Animator::AwaitVSync
[-> flutter/shell/common/animator.cc]

```
void Animator::AwaitVSync() {
  // [见小节2.4]
  waiter_->AsyncWaitForVsync(
      [self = weak_factory_.GetWeakPtr()](fml::TimePoint frame_start_time,
                                          fml::TimePoint frame_target_time) {
        if (self) {
          //是否能复用上次layer树，取决于regenerate_layer_tree_
          if (self->CanReuseLastLayerTree()) {
            //复用上次layer树，直接把任务post到gpu线程做栅格化操作
            self->DrawLastLayerTree();
          } else {
            self->BeginFrame(frame_start_time, frame_target_time);
          }
        }
      });

  delegate_.OnAnimatorNotifyIdle(dart_frame_deadline_);
}
```
waiter_的赋值是在Animator初始化过程，取值为VsyncWaiterAndroid对象，当调用了RequestFrame()，默认参数regenerate_layer_tree_为true，意味着需要重新生成layer树，故不能重复使用上一次的layer树，接着来看一下AsyncWaitForVsync()方法的实现。

### 2.4 VsyncWaiter::AsyncWaitForVsync
[-> flutter/shell/common/vsync_waiter.cc]

```
void VsyncWaiter::AsyncWaitForVsync(Callback callback) {
  {
    std::lock_guard<std::mutex> lock(callback_mutex_);
    //赋值callback_
    callback_ = std::move(callback);
  }
  TRACE_EVENT0("flutter", "AsyncWaitForVsync");
  AwaitVSync(); // [见小节2.5]
}
```
此次的callback_便是[小节2.3]方法中的参数，该方法根据regenerate_layer_tree_来决定执行流。
- 当regenerate_layer_tree_=false，则执行DrawLastLayerTree();
- 当regenerate_layer_tree_=false，则执行BeginFrame();

### 2.5 VsyncWaiterAndroid::AwaitVSync
[-> flutter/shell/platform/android/vsync_waiter_android.cc]

```
void VsyncWaiterAndroid::AwaitVSync() {
  std::weak_ptr<VsyncWaiter>* weak_this =
      new std::weak_ptr<VsyncWaiter>(shared_from_this());
  //获取VsyncWaiter的弱引用
  jlong java_baton = reinterpret_cast<jlong>(weak_this);

  JNIEnv* env = fml::jni::AttachCurrentThread();
  // 此次调用到Java层的asyncWaitForVsync方法，java_baton指向VsyncWaiter
  env->CallStaticVoidMethod(g_vsync_waiter_class->obj(),     //
                            g_async_wait_for_vsync_method_,  //
                            java_baton                       //
  );
}
```
此处g_vsync_waiter_class，g_async_wait_for_vsync_method_的赋值过程是由JNI_OnLoad完成，如下所示。

#### 2.5.1 JNI_OnLoad
[-> flutter/shell/platform/android/library_loader.cc]

```
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  // 初始化Java虚拟机
  fml::jni::InitJavaVM(vm);

  JNIEnv* env = fml::jni::AttachCurrentThread();
  bool result = false;

  // 注册FlutterMain.
  result = shell::FlutterMain::Register(env);

  // 注册PlatformView [见小节2.5.2]
  result = shell::PlatformViewAndroid::Register(env);

  // 注册VSyncWaiter [见小节2.5.3]
  result = shell::VsyncWaiterAndroid::Register(env);

  return JNI_VERSION_1_4;
}
```
首次加载共享库时虚拟机会调用此方法。

#### 2.5.2 Register
[-> flutter/shell/platform/android/platform_view_android_jni.cc]

```
bool PlatformViewAndroid::Register(JNIEnv* env) {
  //记录FlutterCallbackInformation类的全局引用
  g_flutter_callback_info_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/view/FlutterCallbackInformation"));
  //记录FlutterCallbackInformation构造函数
  g_flutter_callback_info_constructor = env->GetMethodID(
      g_flutter_callback_info_class->obj(), "<init>",
      "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V");
  //记录FlutterJNI类的全局引用
  g_flutter_jni_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("io/flutter/embedding/engine/FlutterJNI"));
  //记录SurfaceTexture类的全局引用
  g_surface_texture_class = new fml::jni::ScopedJavaGlobalRef<jclass>(
      env, env->FindClass("android/graphics/SurfaceTexture"));

  static const JNINativeMethod callback_info_methods[] = {
      {
          .name = "nativeLookupCallbackInformation",
          .signature = "(J)Lio/flutter/view/FlutterCallbackInformation;",
          .fnPtr = reinterpret_cast<void*>(&shell::LookupCallbackInformation),
      },
  };
  //注册FlutterCallbackInformation的nativeLookupCallbackInformation()方法
  env->RegisterNatives(g_flutter_callback_info_class->obj(),
                           callback_info_methods,
                           arraysize(callback_info_methods)) != 0);

  g_is_released_method =
      env->GetMethodID(g_surface_texture_class->obj(), "isReleased", "()Z");

  fml::jni::ClearException(env);

  g_attach_to_gl_context_method = env->GetMethodID(
      g_surface_texture_class->obj(), "attachToGLContext", "(I)V");

  g_update_tex_image_method =
      env->GetMethodID(g_surface_texture_class->obj(), "updateTexImage", "()V");

  g_get_transform_matrix_method = env->GetMethodID(
      g_surface_texture_class->obj(), "getTransformMatrix", "([F)V");

  g_detach_from_gl_context_method = env->GetMethodID(
      g_surface_texture_class->obj(), "detachFromGLContext", "()V");

  return RegisterApi(env);
}
```

该方法的主要工作：

记录和注册类FlutterCallbackInformation、FlutterJNI以及SurfaceTexture类的相关方法，用于Java和C++层方法的相互调用。
#### 2.5.3 Register
[-> flutter/shell/platform/android/vsync_waiter_android.cc]

```
bool VsyncWaiterAndroid::Register(JNIEnv* env) {
  static const JNINativeMethod methods[] = ;

  jclass clazz = env->FindClass("io/flutter/view/VsyncWaiter");

  g_vsync_waiter_class = new fml::jni::ScopedJavaGlobalRef<jclass>(env, clazz);

  g_async_wait_for_vsync_method_ = env->GetStaticMethodID(
      g_vsync_waiter_class->obj(), "asyncWaitForVsync", "(J)V");

  return env->RegisterNatives(clazz, methods, arraysize(methods)) == 0;
}
```

该注册过程主要工作：

- 将Java层的VsyncWaiter类的nativeOnVsync()方法，映射到C++层的OnNativeVsync()方法，用于该方法的Java调用C++的过程；
- 将Java层的VsyncWaiter类的asyncWaitForVsync()方法，保存到C++层的g_async_wait_for_vsync_method_变量，用于该方法C++调用Java的过程。

可见，将调用VsyncWaiter类的asyncWaitForVsync()方法

### 2.6 asyncWaitForVsync
[Java]
[-> flutter/shell/platform/android/io/flutter/view/VsyncWaiter.java]

```
public class VsyncWaiter {
    // FlutterView的刷新时间周期（16.7ms）
    public static long refreshPeriodNanos = 1000000000 / 60;

    private static HandlerThread handlerThread;
    private static Handler handler;

    static {
        handlerThread = new HandlerThread("FlutterVsyncThread");
        handlerThread.start();
    }

    public static void asyncWaitForVsync(final long cookie) {
        if (handler == null) {
            handler = new Handler(handlerThread.getLooper());
        }
        handler.post(new Runnable() {
            @Override
            public void run() {
                //注册帧回调方法，见小节[2.6.1]/[2.6.2]
                Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
                    @Override
                    public void doFrame(long frameTimeNanos) {
                        //frameTimeNanos是VYSNC触发的时间点，也就是计划绘制的时间点
                        nativeOnVsync(frameTimeNanos, frameTimeNanos + refreshPeriodNanos, cookie);
                    }
                });
            }
        });
    }
}
```
通过Handler将工作post到FlutterVsyncThread线程，具体的工作是通过Choreographer来注册回调方法doFrame()以监听系统VSYNC信号。

#### 2.6.1 Choreographer.getInstance
[-> Choreographer.java]

```public static Choreographer getInstance() {
    return sThreadInstance.get(); //单例模式
}

private static final ThreadLocal<Choreographer> sThreadInstance =
    new ThreadLocal<Choreographer>() {

    protected Choreographer initialValue() {
        //获取当前线程FlutterVsyncThread的Looper
        Looper looper = Looper.myLooper();
        // 初始化Choreographer对象
        return new Choreographer(looper);
    }
};

private Choreographer(Looper looper) {
    mLooper = looper;
    //创建Handler对象
    mHandler = new FrameHandler(looper);
    //创建用于接收VSync信号的对象
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;  //上一次帧绘制时间点
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];  
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```
此处Choreographer的mLooper和mHandler都运行在FlutterVsyncThread线程。

#### 2.6.2 postFrameCallback
[-> Choreographer.java]

```public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}

private void postCallbackDelayedInternal(int callbackType, Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        //添加到mCallbackQueues队列
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        if (dueTime <= now) {
          scheduleFrameLocked(now);
        } else {
          ...
        }
    }
}
```
将FrameCallback方法加入到mCallbackQueues[CALLBACK_ANIMATION]回调队列中。

