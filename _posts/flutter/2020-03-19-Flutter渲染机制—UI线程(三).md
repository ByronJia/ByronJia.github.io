---
layout: post
title: Flutter渲染机制—UI线程(三)
date: 2020-03-19
tags: Flutter
---

## 三、Engine层绘制
### 3.1 doFrame
[-> Choreographer.java]
```
public void doFrame(long frameTimeNanos) {
    //Android FW每次当vsync信号触发，则会调用该方法 [见下方]
    nativeOnVsync(frameTimeNanos, frameTimeNanos + refreshPeriodNanos, cookie);
}
```

Vsync注册过程见[小节2.6] Choreographer.FrameCallback()。注册了Vysnc信号后，一旦底层Vsync信号触发，经过层层调用回到FrameDisplayEventReceiver的过程，然后会有一个通过handler的方式post到线程”FlutterVsyncThread”来执行操作， 具体流程见Choreographer原理。紧接着再处理所有注册的doCallbacks方法，则会执行Choreographer.FrameCallback中的doFrame()方法，如下所示。
```
new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        //frameTimeNanos是VYSNC触发的时间点，也就是计划绘制的时间点 [见小节3.2]
        nativeOnVsync(frameTimeNanos, frameTimeNanos + refreshPeriodNanos, cookie);
    }
}
```

### 3.2 OnNativeVsync
[-> flutter/shell/platform/android/io/flutter/view/VsyncWaiter.java]
```
public class VsyncWaiter {
    ...
    // [见小节3.2.1]
    private static native void nativeOnVsync(long frameTimeNanos,
                                             long frameTargetTimeNanos,
                                             long cookie);
    ...
}
```

由[小节2.5.3]可知，VsyncWaiter.java中的nativeOnVsync对应于vsync_waiter_android.cc的OnNativeVsync()，具体过程在jni加载过程初始化，如下所示。

#### 3.2.1 OnNativeVsync[C++]
[-> flutter/shell/platform/android/vsync_waiter_android.cc]
```
static void OnNativeVsync(JNIEnv* env, jclass jcaller,
                          jlong frameTimeNanos,
                          jlong frameTargetTimeNanos,
                          jlong java_baton) {
  auto frame_time = fml::TimePoint::FromEpochDelta(
      fml::TimeDelta::FromNanoseconds(frameTimeNanos));
  auto target_time = fml::TimePoint::FromEpochDelta(
      fml::TimeDelta::FromNanoseconds(frameTargetTimeNanos));
  //消费pending回调[见小节3.3]
  ConsumePendingCallback(java_baton, frame_time, target_time);
}
```
### 3.3 ConsumePendingCallback
[-> flutter/shell/platform/android/vsync_waiter_android.cc]
```
static void ConsumePendingCallback(jlong java_baton,
                                   fml::TimePoint frame_start_time,
                                   fml::TimePoint frame_target_time) {
  auto* weak_this = reinterpret_cast<std::weak_ptr<VsyncWaiter>*>(java_baton);
  auto shared_this = weak_this->lock();
  delete weak_this;

  if (shared_this) {
    //shared_this指向VsyncWaiter的弱引用 [见小节3.4]
    shared_this->FireCallback(frame_start_time, frame_target_time);
  }
}
```
### 3.4 VsyncWaiter::FireCallback
[-> flutter/shell/common/vsync_waiter.cc]
```
void VsyncWaiter::FireCallback(fml::TimePoint frame_start_time,
                               fml::TimePoint frame_target_time) {
  Callback callback;
  {
    std::lock_guard<std::mutex> lock(callback_mutex_);
    callback = std::move(callback_);
  }
  if (!callback) {
    TRACE_EVENT_INSTANT0("flutter", "MismatchedFrameCallback");
    return;
  }

  TRACE_EVENT0("flutter", "VsyncFireCallback");
  //将任务放入task队列[见小节3.4.1]
  task_runners_.GetUITaskRunner()->PostTaskForTime(
    [callback, flow_identifier, frame_start_time, frame_target_time]() {
      FML_TRACE_EVENT("flutter", kVsyncTraceName, "StartTime",
                      frame_start_time, "TargetTime", frame_target_time);
      fml::tracing::TraceEventAsyncComplete(
          "flutter", "VsyncSchedulingOverhead", fml::TimePoint::Now(),
          frame_start_time);
      //开始执行vync [见小节3.5]
      callback(frame_start_time, frame_target_time);
      TRACE_FLOW_END("flutter", kVsyncFlowName, flow_identifier);
    },
    frame_start_time);
}
```
将任务闭包放入task队列，消息Loop一旦接受到消息则会读取出来。

#### 3.4.1 MessageLoopImpl::RunExpiredTasks
[-> flutter/fml/message_loop_impl.cc]
```
void MessageLoopImpl::RunExpiredTasks() {
  TRACE_EVENT0("fml", "MessageLoop::RunExpiredTasks");
  std::vector<fml::closure> invocations;

  {
    std::lock_guard<std::mutex> lock(delayed_tasks_mutex_);
    //当没有待处理的task则直接返回
    if (delayed_tasks_.empty()) {
      return;
    }

    auto now = fml::TimePoint::Now();
    while (!delayed_tasks_.empty()) {
      const auto& top = delayed_tasks_.top();
      if (top.target_time > now) {
        break;
      }
      invocations.emplace_back(std::move(top.task));
      delayed_tasks_.pop();
    }
    WakeUp(delayed_tasks_.empty() ? fml::TimePoint::Max()
                                  : delayed_tasks_.top().target_time);
  }

  for (const auto& invocation : invocations) {
    invocation();  // [见小节3.5]
    for (const auto& observer : task_observers_) {
      observer.second();
    }
  }
}
```
对于ui线程处于消息loop状态，一旦有时间到达的任务则开始执行，否则处于空闲等等状态。前面[小节3.4] VsyncWaiter::FireCallback过程已经向该ui线程postTask。 对于不可复用layer tree的情况则调用Animator::BeginFrame()方法。

### 3.5 callback
[-> flutter/shell/common/animator.cc]
```
[self = weak_factory_.GetWeakPtr()](fml::TimePoint frame_start_time,
                                    fml::TimePoint frame_target_time) {
  if (self) {
    if (self->CanReuseLastLayerTree()) {
      self->DrawLastLayerTree();
    } else {
      //根据默认参数regenerate_layer_tree_为true，则执行该分支 [见小节3.6]
      self->BeginFrame(frame_start_time, frame_target_time);
    }
  }
}
```
此次的callback赋值过程位于[小节2.3]Animator::AwaitVSync()方法的闭包参数，相关说明：

- frame_start_time：计划开始绘制时间点，来源于doFrame()方法中的参数；
- frame_target_time：从frame_start_time加上一帧时间(16.7ms)的时间，作为本次绘制的deadline。

### 3.6 Animator::BeginFrame
[-> flutter/shell/common/animator.cc]
```
void Animator::BeginFrame(fml::TimePoint frame_start_time,
                          fml::TimePoint frame_target_time) {
  TRACE_EVENT_ASYNC_END0("flutter", "Frame Request Pending", frame_number_++);
  TRACE_EVENT0("flutter", "Animator::BeginFrame");

  frame_scheduled_ = false;
  notify_idle_task_id_++;
  regenerate_layer_tree_ = false;
  //信号量加1，可以注册新的vsync信号，也就是能执行Animator::RequestFrame()
  pending_frame_semaphore_.Signal();

  if (!producer_continuation_) {
    //[小节3.6.1]/[小节3.6.2]
    producer_continuation_ = layer_tree_pipeline_->Produce();
    //pipeline已满，说明GPU线程繁忙，则结束本次UI绘制，重新注册Vsync
    if (!producer_continuation_) {
      RequestFrame();
      return;
    }
  }

  //从pipeline中获取有效的continuation，并准备为可能的frame服务
  last_begin_frame_time_ = frame_start_time;
  //获取当前帧绘制截止时间，用于告知可GC的空闲时长
  dart_frame_deadline_ = FxlToDartOrEarlier(frame_target_time);
  {
    TRACE_EVENT2("flutter", "Framework Workload", "mode", "basic", "frame",
                 FrameParity());
    //此处delegate_为Shell [小节3.7]
    delegate_.OnAnimatorBeginFrame(last_begin_frame_time_);
  }

  if (!frame_scheduled_) {
    task_runners_.GetUITaskRunner()->PostDelayedTask(
        [self = weak_factory_.GetWeakPtr(),
         notify_idle_task_id = notify_idle_task_id_]() {
          if (!self.get()) {
            return;
          }
          // 该任务id和当前任务id一致，则不再需要审查frame，可以通知引擎当前处于空闲状态，100ms
          if (notify_idle_task_id == self->notify_idle_task_id_) {
            self->delegate_.OnAnimatorNotifyIdle(Dart_TimelineGetMicros() +
                                                 100000);
          }
        },
        kNotifyIdleTaskWaitTime); //延迟51ms再通知引擎空闲状态
  }
}
```
该方法主要功能说明：

- layer_tree_pipeline_是在Animator对象初始化的过程中创建的LayerTreePipeline，其类型为Pipeline
- 此处kNotifyIdleTaskWaitTime等于51ms，等于3帧的时间+1ms，之所以这样设计是由于在某些工作负载下（比如父视图调整大小，通过viewport metrics事件传达给子视图）实际上还没有schedule帧，尽管在下一个vsync会生成一帧(将在收到viewport事件后schedule)，因此推迟调用OnAnimatorNotifyIdle一点点，从而避免可能垃圾回收在不希望的时间触发。

#### 3.6.1 LayerTreePipeline初始化
[-> flutter/shell/common/animator.cc]
```
Animator::Animator(Delegate& delegate,
                   TaskRunners task_runners,
                   std::unique_ptr<VsyncWaiter> waiter)
    : delegate_(delegate),
      task_runners_(std::move(task_runners)),
      waiter_(std::move(waiter)),
      last_begin_frame_time_(),
      dart_frame_deadline_(0),
      layer_tree_pipeline_(fml::MakeRefCounted<LayerTreePipeline>(2)),
      ... {}
```
此处LayerTreePipeline的初始化过程如下：

```
using LayerTreePipeline = Pipeline<flutter::LayerTree>;
```
在pipeline.h的过程会初始化Pipeline，可见初始值empty_ = 2，available_ = 0；
```
Pipeline(uint32_t depth) : empty_(depth), available_(0) {}
```
#### 3.6.2 Pipeline::Produce
[-> flutter/synchronization/pipeline.h]
```
ProducerContinuation Produce() {
  //当管道不为空，则不允许再次向管道加入数据
  if (!empty_.TryWait()) {
    return {};
  }

  //[见小节3.6.3]
  return ProducerContinuation{
      std::bind(&Pipeline::ProducerCommit, this, std::placeholders::_1,
                std::placeholders::_2),  // continuation
      GetNextPipelineTraceID()};  
}
```
通过信号量empty_的初始值为depth(默认等于2)，来保证同一个管道的任务最多不超过depth个，每次UI线程执行Produce()会减1，当GPU线程执行完成Consume()方法后才会执行加1操作。

#### 3.6.3 ProducerContinuation初始化
[-> flutter/synchronization/pipeline.h]
```
ProducerContinuation(Continuation continuation, size_t trace_id)
    : continuation_(continuation), trace_id_(trace_id) {
  TRACE_FLOW_BEGIN("flutter", "PipelineItem", trace_id_);
  TRACE_EVENT_ASYNC_BEGIN0("flutter", "PipelineProduce", trace_id_);
}
3.6.3 Pipeline.ProducerCommit
[-> flutter/synchronization/pipeline.h]

void ProducerCommit(ResourcePtr resource, size_t trace_id) {
  {
    std::lock_guard<std::mutex> lock(queue_mutex_);
    queue_.emplace(std::move(resource), trace_id);
  }

  available_.Signal();
}
```

### 3.7 Shell::OnAnimatorBeginFrame
[-> flutter/shell/common/shell.cc]
```
void Shell::OnAnimatorBeginFrame(fml::TimePoint frame_time) {
  if (engine_) {
    engine_->BeginFrame(frame_time);  // [小节3.8]
  }
}
```
### 3.8 Engine::BeginFrame
[-> flutter/shell/common/engine.cc]
```
void Engine::BeginFrame(fml::TimePoint frame_time) {
  TRACE_EVENT0("flutter", "Engine::BeginFrame");
  runtime_controller_->BeginFrame(frame_time);  // [小节3.9]
}
```
### 3.9 RuntimeController::BeginFrame
[-> flutter/runtime/runtime_controller.cc]
```
bool RuntimeController::BeginFrame(fml::TimePoint frame_time) {
  if (auto* window = GetWindowIfAvailable()) {
    window->BeginFrame(frame_time);  // [小节3.10]
    return true;
  }
  return false;
}
```

### 3.10 Window::BeginFrame
[-> flutter/lib/ui/window/window.cc]
```
void Window::BeginFrame(fml::TimePoint frameTime) {
  std::shared_ptr<tonic::DartState> dart_state = library_.dart_state().lock();
  if (!dart_state)
    return;
  tonic::DartState::Scope scope(dart_state);
  //注意此处的frameTime便是前面小节3.1中doFrame方法中的参数frameTimeNanos
  int64_t microseconds = (frameTime - fml::TimePoint()).ToMicroseconds();

  // [见小节4.2]
  DartInvokeField(library_.value(), "_beginFrame",
                  {Dart_NewInteger(microseconds)});

  //执行MicroTask
  UIDartState::Current()->FlushMicrotasksNow();

  // [见小节4.4]
  DartInvokeField(library_.value(), "_drawFrame", {});
}
```

Window::BeginFrame()过程主要工作：

- 执行_beginFrame
- 执行FlushMicrotasksNow
- 执行_drawFrame

可见，Microtask位于beginFrame和drawFrame之间，那么Microtask的耗时会影响ui绘制过程。

DartInvokeField()通过dart虚拟机调用了window.onBeginFrame()和onDrawFrame方法，见hooks.dart文件中如下过程：
```
@pragma('vm:entry-point')
void _beginFrame(int microseconds) {
  _invoke1<Duration>(window.onBeginFrame, window._onBeginFrameZone, new Duration(microseconds: microseconds));
}

@pragma('vm:entry-point')
void _drawFrame() {
  _invoke(window.onDrawFrame, window._onDrawFrameZone);
}
```

