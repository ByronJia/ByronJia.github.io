---
layout: post
title: Flutter渲染机制—UI线程(四)
date: 2020-03-25
tags: Flutter
---

## 四、Framework层绘制
在引擎层的处理过程会调用到window.onBeginFrame()和onDrawFrame，回到framework层从这个两个方法开始说起。

### 4.1 SchedulerBinding.initInstances
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]

```
mixin SchedulerBinding on BindingBase, ServicesBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    ui.window.onBeginFrame = _handleBeginFrame; //[见小节4.2]
    ui.window.onDrawFrame = _handleDrawFrame;  //[见小节4.4]
    SystemChannels.lifecycle.setMessageHandler(_handleLifecycleMessage);
  }
}
```
可见，引擎层中的Window::BeginFrame()调用的两个方法，进入到dart层则分别是_handleBeginFrame()和_handleDrawFrame()方法

#### 4.1.1 Window初始化
[-> flutter/lib/ui/window.dart]
```
class Window {
    Window._()

    FrameCallback get onBeginFrame => _onBeginFrame;
    FrameCallback _onBeginFrame;

    VoidCallback get onDrawFrame => _onDrawFrame;
    VoidCallback _onDrawFrame;
    ...
}
```
Window初始化过程，可以知道onBeginFrame和onDrawFrame分别保存_onBeginFrame和_onDrawFrame方法。

### 4.2 _handleBeginFrame
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void _handleBeginFrame(Duration rawTimeStamp) {
  if (_warmUpFrame) {
    _ignoreNextEngineDrawFrame = true;
    return;
  }
  handleBeginFrame(rawTimeStamp);  //[见小节4.3]
}
```
### 4.3 handleBeginFrame
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void handleBeginFrame(Duration rawTimeStamp) {
  Timeline.startSync('Frame', arguments: timelineWhitelistArguments);
  _firstRawTimeStampInEpoch ??= rawTimeStamp;
  _currentFrameTimeStamp = _adjustForEpoch(rawTimeStamp ?? _lastRawTimeStamp);
  if (rawTimeStamp != null)
    _lastRawTimeStamp = rawTimeStamp;

  profile(() {
    _profileFrameNumber += 1;
    _profileFrameStopwatch.reset();
    _profileFrameStopwatch.start();
  });

  //此时阶段等于SchedulerPhase.idle;
  _hasScheduledFrame = false;
  try {
    Timeline.startSync('Animate', arguments: timelineWhitelistArguments);
    _schedulerPhase = SchedulerPhase.transientCallbacks;
    //执行动画的回调方法
    final Map<int, _FrameCallbackEntry> callbacks = _transientCallbacks;
    _transientCallbacks = <int, _FrameCallbackEntry>{};
    callbacks.forEach((int id, _FrameCallbackEntry callbackEntry) {
      if (!_removedIds.contains(id))
        _invokeFrameCallback(callbackEntry.callback, _currentFrameTimeStamp, callbackEntry.debugStack);
    });
    _removedIds.clear();
  } finally {
    _schedulerPhase = SchedulerPhase.midFrameMicrotasks;
  }
}
```
该方法主要功能是遍历_transientCallbacks，执行相应的Animate操作，可通过scheduleFrameCallback()/cancelFrameCallbackWithId()来完成添加和删除成员，再来简单看看这两个方法。

#### 4.3.1 scheduleFrameCallback
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
int scheduleFrameCallback(FrameCallback callback, { bool rescheduling = false }) {
  scheduleFrame();  //触发帧绘制的调度
  _nextFrameCallbackId += 1;
  _transientCallbacks[_nextFrameCallbackId] = _FrameCallbackEntry(callback, rescheduling: rescheduling);
  return _nextFrameCallbackId;
}
```
callback保存在_FrameCallbackEntry对象里面

#### 4.3.2 cancelFrameCallbackWithId
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void cancelFrameCallbackWithId(int id) {
  assert(id > 0);
  _transientCallbacks.remove(id);
  _removedIds.add(id);
}
```
### 4.4 _handleDrawFrame
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void _handleDrawFrame() {
  if (_ignoreNextEngineDrawFrame) {
    _ignoreNextEngineDrawFrame = false;
    return;
  }
  handleDrawFrame();  //[见小节4.5]
}
```
### 4.5 handleDrawFrame
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  Timeline.finishSync(); // 标识结束"Animate"阶段
  try {
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
    //执行PERSISTENT FRAME回调
    for (FrameCallback callback in _persistentCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp); //[见小节4.5.1]

    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    // 执行POST-FRAME回调
    final List<FrameCallback> localPostFrameCallbacks = List<FrameCallback>.from(_postFrameCallbacks);
    _postFrameCallbacks.clear();
    for (FrameCallback callback in localPostFrameCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp);
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    Timeline.finishSync(); //标识结束”Frame“阶段
    profile(() {
      _profileFrameStopwatch.stop();
      _profileFramePostEvent();
    });
    _currentFrameTimeStamp = null;
  }
}
```
该方法主要功能：

- 遍历_persistentCallbacks，执行相应的回调方法，可通过addPersistentFrameCallback()注册，一旦注册后不可移除，后续每一次frame回调都会执行；
- 遍历_postFrameCallbacks，执行相应的回调方法，可通过addPostFrameCallback()注册，handleDrawFrame()执行完成后会清空_postFrameCallbacks内容。

#### 4.5.1 _invokeFrameCallback
[-> lib/src/scheduler/binding.dart:: SchedulerBinding]
```
void _invokeFrameCallback(FrameCallback callback, Duration timeStamp, [ StackTrace callbackStack ]) {
  try {
    callback(timeStamp);  //[见小节4.5.2]
  } catch (exception, exceptionStack) {
    FlutterError.reportError(FlutterErrorDetails(...));
  }
}
```
这里的callback是_persistentCallbacks列表中的成员，再来看看其成员是如何添加进去的。

#### 4.5.2 WidgetsBinding.initInstances
[-> lib/src/widgets/binding.dart]
```
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();  //[见小节4.5.3]
    _instance = this;
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    ui.window.onLocaleChanged = handleLocaleChanged;
    ui.window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
    SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
    SystemChannels.system.setMessageHandler(_handleSystemMessage);
  }
}
```
在flutter app启动过程，也就是执行runApp过程会有WidgetsFlutterBinding初始化过程，WidgetsBinding的initInstances()，根据mixin的顺序，可知此处的super.initInstances() 便是RendererBinding类。

#### 4.5.3 RendererBinding.initInstances
[-> lib/src/rendering/binding.dart]
```
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, SemanticsBinding, HitTestable {

  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    ui.window
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
      ..onSemanticsAction = _handleSemanticsAction;
    initRenderView();
    _handleSemanticsEnabledChanged();
    addPersistentFrameCallback(_handlePersistentFrameCallback); //[见小节4.5.4]
  }

  void _handlePersistentFrameCallback(Duration timeStamp) {
    drawFrame();  //[见小节4.6]
  }
}
```

#### 4.5.4 SchedulerBinding.addPersistentFrameCallback
[-> lib/src/scheduler/binding.dart]
```
mixin SchedulerBinding on BindingBase, ServicesBinding {

  void addPersistentFrameCallback(FrameCallback callback) {
    _persistentCallbacks.add(callback);
  }
}
```

### 4.6 WidgetsBinding.drawFrame
[-> lib/src/widgets/binding.dart]
```
void drawFrame() {
  try {
    if (renderViewElement != null)
      buildOwner.buildScope(renderViewElement);   //[见小节4.6.1]
    super.drawFrame();   //[见小节4.6.4]
    buildOwner.finalizeTree();  //[见小节4.12]
  } finally {
  }
}
```
#### 4.6.1 BuildOwner.buildScope
[-> lib/src/widgets/framework.dart]
```
void buildScope(Element context, [VoidCallback callback]) {
  if (callback == null && _dirtyElements.isEmpty)
    return;
  Timeline.startSync('Build', arguments: timelineWhitelistArguments);
  try {
    _scheduledFlushDirtyElements = true;
    if (callback != null) {
      _dirtyElementsNeedsResorting = false;
      callback();  //执行回调方法
    }
    _dirtyElements.sort(Element._sort); //排序
    _dirtyElementsNeedsResorting = false;
    int dirtyCount = _dirtyElements.length;
    int index = 0;
    while (index < dirtyCount) {
      try {
        //具体Element子类执行重建操作 [见小节4.6.2]
        _dirtyElements[index].rebuild();
      } catch (e, stack) {
      }
      index += 1;
      if (dirtyCount < _dirtyElements.length || _dirtyElementsNeedsResorting) {
        _dirtyElements.sort(Element._sort);
        _dirtyElementsNeedsResorting = false;
        dirtyCount = _dirtyElements.length;
        while (index > 0 && _dirtyElements[index - 1].dirty) {
          index -= 1;
        }
      }
    }
  } finally {
    for (Element element in _dirtyElements) {
      element._inDirtyList = false;
    }
    _dirtyElements.clear();
    _scheduledFlushDirtyElements = false;
    _dirtyElementsNeedsResorting = null;
    Timeline.finishSync();
  }
}
```
#### 4.6.2 Element.rebuild
[-> lib/src/widgets/framework.dart]
```
void rebuild() {
  if (!_active || !_dirty)
    return;
  performRebuild();
}
```
performRebuild具体执行方法，取决于相应的Element子类，这里以ComponentElement为例

#### 4.6.3 ComponentElement.performRebuild
[-> lib/src/widgets/framework.dart]
```
void performRebuild() {
  Widget built;
  try {
    built = build();  //执行build方法
  } catch (e, stack) {
    built = ErrorWidget.builder(_debugReportException('building $this', e, stack));
  } finally {
    _dirty = false;
  }
  try {
    _child = updateChild(_child, built, slot); //更新子元素
  } catch (e, stack) {
    built = ErrorWidget.builder(_debugReportException('building $this', e, stack));
    _child = updateChild(null, built, slot);
  }
}
```
#### 4.6.4 RendererBinding.drawFrame
[-> lib/src/rendering/binding.dart]
```
void drawFrame() {
  pipelineOwner.flushLayout();  //[见小节4.7]
  pipelineOwner.flushCompositingBits();  //[见小节4.8]
  pipelineOwner.flushPaint(); //[见小节4.9]
  renderView.compositeFrame();  //[见小节4.10]
  pipelineOwner.flushSemantics(); //[见小节4.11]
}
```

RendererBinding的initInstances()过程注册了一个Persistent的帧回调方法_handlePersistentFrameCallback()，故handleDrawFrame()过程会调用该方法。pipelineOwner管理渲染管道，提供了一个用于驱动渲染管道的接口，并存储了哪些渲染对象请求访问状态，要刷新管道，需要按顺序运行如下5个阶段：

- [flushLayout]：更新需要计算其布局的渲染对象，在此阶段计算每个渲染对象的大小和位置，渲染对象可能会弄脏其绘画或者合成状态，这个过程可能还会调用到build过程。
耗时对应timeline的‘Layout’过程
- [flushCompositingBits]：更新具有脏合成位的任何渲染对象，在此阶段每个渲染对象都会了解其子项是否需要合成。在绘制阶段使用此信息选择如何实现裁剪等视觉效果。如果渲染对象有一个自己合成的子项，它需要使用布局信息来创建裁剪，以便将裁剪应用于已合成的子项
耗时对应timeline的‘Compositing bits’过程
- [flushPaint]：访问需要绘制的任何渲染对象，在此阶段，渲染对象有机会将绘制命令记录到[PictureLayer]，并构建其他合成的[Layer]；
    - 耗时对应timeline的‘Paint’过程
- [compositeFrame]：将Compositing bits发送给GPU；
    - 耗时对应timeline的‘Compositing’过程
- [flushSemantics]：编译渲染对象的语义，并将语义发送给操作系统；
    - 耗时对应timeline的‘Semantics’过程
    
packages/flutter/lib/src/rendering/debug.dart，这里面记录着关于render过程相关的调试开关，可以逐一实践。

### 4.7 PipelineOwner.flushLayout
[-> lib/src/rendering/object.dart]
```
void flushLayout() {
  profile(() {
    Timeline.startSync('Layout', arguments: timelineWhitelistArguments);
  });
  try {
    //遍历所有的渲染对象
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
        //如果渲染对象需要重新布局，则执行布局操作 [见小节4.7.1]
        if (node._needsLayout && node.owner == this)
          node._layoutWithoutResize();
      }
    }
  } finally {
    profile(() {
      Timeline.finishSync();
    });
  }
}
```
#### 4.7.1 _layoutWithoutResize
[-> lib/src/rendering/object.dart]
```
void _layoutWithoutResize() {
  try {
    performLayout();  //执行布局操作[]
    markNeedsSemanticsUpdate();  //[见小节4.7.2]
  } catch (e, stack) {
    _debugReportException('performLayout', e, stack);
  }
  _needsLayout = false; //完成layout操作
  markNeedsPaint(); // [见小节4.7.3]
}
```
该方法主要工作：

- performLayout操作：参数sizedByParent为false需要同时改变渲染对象和指导子项的布局，性能更慢；
- markNeedsSemanticsUpdate：标记需要更新语义；
- markNeedsPaint：标记需要绘制；

```
SchedulerBinding.scheduleWarmUpFrame
  RenderView.performLayout
    RenderObject.layout
      _RenderLayoutBuilder.performLayout
        _LayoutBuilderElement._layout
          BuildOwner.buildScope
```

#### 4.7.2 markNeedsSemanticsUpdate
[-> lib/src/rendering/object.dart]
```
void markNeedsSemanticsUpdate() {
  if (!attached || owner._semanticsOwner == null) {
    _cachedSemanticsConfiguration = null;
    return;
  }

  final bool wasSemanticsBoundary = _semantics != null && _cachedSemanticsConfiguration?.isSemanticBoundary == true;
  _cachedSemanticsConfiguration = null;
  bool isEffectiveSemanticsBoundary = _semanticsConfiguration.isSemanticBoundary && wasSemanticsBoundary;
  RenderObject node = this;

  while (!isEffectiveSemanticsBoundary && node.parent is RenderObject) {
    if (node != this && node._needsSemanticsUpdate)
      break;
    node._needsSemanticsUpdate = true;

    node = node.parent;
    isEffectiveSemanticsBoundary = node._semanticsConfiguration.isSemanticBoundary;
    if (isEffectiveSemanticsBoundary && node._semantics == null) {
      return;
    }
  }
  if (node != this && _semantics != null && _needsSemanticsUpdate) {
    owner._nodesNeedingSemantics.remove(this);
  }
  if (!node._needsSemanticsUpdate) {
    node._needsSemanticsUpdate = true;
    if (owner != null) {
      //记录需要更新语义的渲染对象
      owner._nodesNeedingSemantics.add(node);
      owner.requestVisualUpdate();
    }
  }
}
```
#### 4.7.3 markNeedsPaint
[-> lib/src/rendering/object.dart]
```
void markNeedsPaint() {
  if (_needsPaint)
    return;
  _needsPaint = true;
  if (isRepaintBoundary) {
    if (owner != null) {
      //记录需要重新绘制的渲染对象
      owner._nodesNeedingPaint.add(this);
      owner.requestVisualUpdate();
    }
  } else if (parent is RenderObject) {
    final RenderObject parent = this.parent;
    parent.markNeedsPaint();
  } else {
    if (owner != null)
      owner.requestVisualUpdate();
  }
}
```
### 4.8 PipelineOwner.flushCompositingBits
[-> lib/src/rendering/object.dart]
```
void flushCompositingBits() {
  profile(() { Timeline.startSync('Compositing bits'); });
  _nodesNeedingCompositingBitsUpdate.sort((RenderObject a, RenderObject b) => a.depth - b.depth);
  for (RenderObject node in _nodesNeedingCompositingBitsUpdate) {
    //根据需要来决定是否更新位合成
    if (node._needsCompositingBitsUpdate && node.owner == this)
      node._updateCompositingBits(); // [见小节4.8.1]
  }
  _nodesNeedingCompositingBitsUpdate.clear();  //清空需要位合成的渲染对象
  profile(() { Timeline.finishSync(); });
}
```
#### 4.8.1 _updateCompositingBits
[-> lib/src/rendering/object.dart]
```
void _updateCompositingBits() {
  if (!_needsCompositingBitsUpdate)
    return;
  final bool oldNeedsCompositing = _needsCompositing;
  _needsCompositing = false;
  visitChildren((RenderObject child) {
    //遍历所有子项来更新位合成
    child._updateCompositingBits();
    if (child.needsCompositing)
      _needsCompositing = true;
  });
  if (isRepaintBoundary || alwaysNeedsCompositing)
    _needsCompositing = true;
  if (oldNeedsCompositing != _needsCompositing)
    markNeedsPaint();
  _needsCompositingBitsUpdate = false;
}
```
### 4.9 PipelineOwner.flushPaint
[-> lib/src/rendering/object.dart]
```
void flushPaint() {
  profile(() { Timeline.startSync('Paint', arguments: timelineWhitelistArguments); });
  try {
    final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
    _nodesNeedingPaint = <RenderObject>[];
    //排序脏节点，深度最大的节点排第一位
    for (RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
      if (node._needsPaint && node.owner == this) {
        //此节点是否连接到树中，如果连接则重绘，否则跳过
        if (node._layer.attached) {
          PaintingContext.repaintCompositedChild(node);  //[小节4.9.1]
        } else {
          node._skippedPaintingOnLayer();
        }
      }
    }
  } finally {
    profile(() { Timeline.finishSync(); });
  }
}
```
#### 4.9.1 repaintCompositedChild
[-> lib/src/rendering/object.dart]
```
static void repaintCompositedChild(RenderObject child, { bool debugAlsoPaintedParent = false }) {
  _repaintCompositedChild(
    child,
    debugAlsoPaintedParent: debugAlsoPaintedParent,
  );
}

static void _repaintCompositedChild(
  RenderObject child, {
  bool debugAlsoPaintedParent = false,
  PaintingContext childContext,
}) {
  if (child._layer == null) {
    child._layer = OffsetLayer();
  } else {
    child._layer.removeAllChildren();
  }
  childContext ??= PaintingContext(child._layer, child.paintBounds);
  child._paintWithContext(childContext, Offset.zero);
  childContext.stopRecordingIfNeeded();
}
```
### 4.10 RenderView.compositeFrame
[-> lib/src/rendering/view.dart]
```
void compositeFrame() {
  Timeline.startSync('Compositing', arguments: timelineWhitelistArguments);
  try {
    //创建SceneBuilder [见小节4.10.1]
    final ui.SceneBuilder builder = ui.SceneBuilder();
    //创建Scene [见小节4.10.2]
    final ui.Scene scene = layer.buildScene(builder);
    if (automaticSystemUiAdjustment)
      _updateSystemChrome();
    ui.window.render(scene); // [见小节4.10.3]
    scene.dispose();
  } finally {
    Timeline.finishSync();
  }
}
```
该方法主要工作：

- 分别创建Flutter框架(dart)和引擎层(C++)的两个SceneBuilder；
- 分别创建Flutter框架(dart)和引擎层(C++)的两个Scene；
- 执行render()将layer树发送给GPU线程；

#### 4.10.1 SceneBuilder初始化
[-> lib/ui/compositing.dart]
```
class SceneBuilder extends NativeFieldWrapperClass2 {
  @pragma('vm:entry-point')
  SceneBuilder() { _constructor(); }
  void _constructor() native 'SceneBuilder_constructor';
  ...
}
```
SceneBuilder_constructor这是native方法，最终调用到引擎中的lib/ui/compositing/scene_builder.h中的SceneBuilder::create()方法， 创建C++的SceneBuilder对象。

#### 4.10.2 OffsetLayer.buildScene
[-> lib/src/rendering/layer.dart]
```
ui.Scene buildScene(ui.SceneBuilder builder) {
  updateSubtreeNeedsAddToScene();  //遍历layer树，将需要子树加入到scene
  addToScene(builder); //将layer添加到SceneBuilder
  return builder.build(); //调用C++层的build来构建Scene对象。
}
```
遍历layer树，将需要更新的全部都加入到SceneBuilder。再调用build()，同样也是native方法，执行SceneBuilder::build()来构建Scene对象。

#### 4.10.3 Window::Render
[-> flutter/lib/ui/window/window.cc]
```
void Render(Dart_NativeArguments args) {
  Dart_Handle exception = nullptr;
  Scene* scene = tonic::DartConverter<Scene*>::FromArguments(args, 1, exception);
  if (exception) {
    Dart_ThrowException(exception);
    return;
  }
  UIDartState::Current()->window()->client()->Render(scene);  // [4.10.4]
}
```
ui.window.render()位于window.dart文件，这是一个native方法，会调用到window.cc的Render()方法。

#### 4.10.4 RuntimeController::Render
[-> flutter/runtime/runtime_controller.cc]
```
void RuntimeController::Render(Scene* scene) {
  //从scene中取出layer树 [见小节4.10.5]
  client_.Render(scene->takeLayerTree());
}
```
#### 4.10.5 Engine::Render
[-> flutter/shell/common/engine.cc]
```
void Engine::Render(std::unique_ptr<flow::LayerTree> layer_tree) {
  if (!layer_tree)
    return;

  SkISize frame_size = SkISize::Make(viewport_metrics_.physical_width,
                                     viewport_metrics_.physical_height);
  if (frame_size.isEmpty())
    return;

  layer_tree->set_frame_size(frame_size);
  animator_->Render(std::move(layer_tree));  // [4.10.6]
}
```
#### 4.10.6 Animator::Render
[-> flutter/shell/common/animator.cc]
```
void Animator::Render(std::unique_ptr<flow::LayerTree> layer_tree) {
  if (dimension_change_pending_ &&
      layer_tree->frame_size() != last_layer_tree_size_) {
    dimension_change_pending_ = false;
  }
  last_layer_tree_size_ = layer_tree->frame_size();

  if (layer_tree) {
    layer_tree->set_construction_time(fml::TimePoint::Now() -
                                      last_begin_frame_time_);
  }

  //提交待处理的continuation，本次PipelineProduce完成 //[见小节4.10.7]
  producer_continuation_.Complete(std::move(layer_tree));

  delegate_.OnAnimatorDraw(layer_tree_pipeline_); //[见小节4.10.8]
}
```
UI线程的耗时从doFrame(frameTimeNanos)中的frameTimeNanos为起点，以Animator::Render()方法结束为终点， 并将结果保存到LayerTree的成员变量construction_time_，这便是UI线程的耗时时长。

#### 4.10.7 ProducerContinuation.Complete
[-> flutter/synchronization/pipeline.h]
```
class ProducerContinuation {
  void Complete(ResourcePtr resource) {
    if (continuation_) {
      continuation_(std::move(resource), trace_id_);
      continuation_ = nullptr;
      TRACE_EVENT_ASYNC_END0("flutter", "PipelineProduce", trace_id_);
      TRACE_FLOW_STEP("flutter", "PipelineItem", trace_id_);
    }
  }
  ```
#### 4.10.8 Shell::OnAnimatorDraw
[-> flutter/shell/common/shell.cc]
```
void Shell::OnAnimatorDraw(
    fml::RefPtr<flutter::Pipeline<flow::LayerTree>> pipeline) {

  //向GPU线程提交绘制任务
  task_runners_.GetGPUTaskRunner()->PostTask(
      [rasterizer = rasterizer_->GetWeakPtr(),
       pipeline = std::move(pipeline)]() {
        if (rasterizer) {
          //由GPU线程来负责栅格化操作
          rasterizer->Draw(pipeline);
        }
      });
}
```
这个方法主要是向GPU线程提交绘制任务。

### 4.11 PipelineOwner.flushSemantics
[-> lib/src/rendering/view.dart]
```
void flushSemantics() {
  if (_semanticsOwner == null)
    return;
  profile(() { Timeline.startSync('Semantics'); });
  try {
    final List<RenderObject> nodesToProcess = _nodesNeedingSemantics.toList()
      ..sort((RenderObject a, RenderObject b) => a.depth - b.depth);
    _nodesNeedingSemantics.clear();
    //遍历_nodesNeedingSemantics，更新需要更新语义的渲染对象
    for (RenderObject node in nodesToProcess) {
      if (node._needsSemanticsUpdate && node.owner == this)
        node._updateSemantics(); // [见小节4.11.1]
    }
    _semanticsOwner.sendSemanticsUpdate(); // 发送语义更新[见小节4.11.2]
  } finally {
    profile(() { Timeline.finishSync(); });
  }
}
```
#### 4.11.1 _updateSemantics
[-> lib/src/rendering/object.dart]
```
void _updateSemantics() {
  if (_needsLayout) {
    //此子树中没有足够的信息来计算语义，子树可能被视图窗口保持活着但没有布局
    return;
  }
  final _SemanticsFragment fragment = _getSemanticsForParent(
    mergeIntoParent: _semantics?.parent?.isPartOfNodeMerging ?? false,
  );
  final _InterestingSemanticsFragment interestingFragment = fragment;
  final SemanticsNode node = interestingFragment.compileChildren(
    parentSemanticsClipRect: _semantics?.parentSemanticsClipRect,
    parentPaintClipRect: _semantics?.parentPaintClipRect,
  ).single;
}
```
#### 4.11.2 sendSemanticsUpdate
[-> lib/src/semantics/semantics.dart]
```
void sendSemanticsUpdate() {
  if (_dirtyNodes.isEmpty)
    return;
  final Set<int> customSemanticsActionIds = Set<int>();
  final List<SemanticsNode> visitedNodes = <SemanticsNode>[];
  while (_dirtyNodes.isNotEmpty) {
    final List<SemanticsNode> localDirtyNodes = _dirtyNodes.where((SemanticsNode node) => !_detachedNodes.contains(node)).toList();
    _dirtyNodes.clear();
    _detachedNodes.clear();
    localDirtyNodes.sort((SemanticsNode a, SemanticsNode b) => a.depth - b.depth);
    visitedNodes.addAll(localDirtyNodes);
    for (SemanticsNode node in localDirtyNodes) {
      if (node.isPartOfNodeMerging) {
        //如果合并到父节点，确保父节点已被添加到脏列表
        if (node.parent != null && node.parent.isPartOfNodeMerging)
          node.parent._markDirty(); //将节点添加到脏列表
      }
    }
  }
  visitedNodes.sort((SemanticsNode a, SemanticsNode b) => a.depth - b.depth);
  final ui.SemanticsUpdateBuilder builder = ui.SemanticsUpdateBuilder();
  for (SemanticsNode node in visitedNodes) {
    if (node._dirty && node.attached)
      node._addToUpdate(builder, customSemanticsActionIds);
  }
  _dirtyNodes.clear();
  for (int actionId in customSemanticsActionIds) {
    final CustomSemanticsAction action = CustomSemanticsAction.getAction(actionId);
    builder.updateCustomAction(id: actionId, label: action.label, hint: action.hint, overrideId: action.action?.index ?? -1);
  }
  ui.window.updateSemantics(builder.build());  // [见小节4.11.3]
  notifyListeners(); //通知已注册的监听器
}
```
可以看看监听器的数据，是否影响性能。

updateSemantics这是window.dart中的一个native方法，调用到如下方法。

#### 4.11.3 Window::updateSemantics
[-> flutter/lib/ui/window/window.cc]
```
void UpdateSemantics(Dart_NativeArguments args) {
  Dart_Handle exception = nullptr;
  SemanticsUpdate* update =
      tonic::DartConverter<SemanticsUpdate*>::FromArguments(args, 1, exception);
  if (exception) {
    Dart_ThrowException(exception);
    return;
  }
  UIDartState::Current()->window()->client()->UpdateSemantics(update); // [见小节4.11.4]
}
```
#### 4.11.4 RuntimeController::UpdateSemantics
[-> flutter/runtime/runtime_controller.cc]
```
void RuntimeController::UpdateSemantics(SemanticsUpdate* update) {
  if (window_data_.semantics_enabled) {
    client_.UpdateSemantics(update->takeNodes(), update->takeActions()); // [见小节4.11.5]
  }
}
```
#### 4.11.5 Engine::UpdateSemantics
[-> flutter/shell/common/engine.cc]
```
void Engine::UpdateSemantics(blink::SemanticsNodeUpdates update,
                             blink::CustomAccessibilityActionUpdates actions) {
  delegate_.OnEngineUpdateSemantics(std::move(update), std::move(actions)); // [见小节4.11.6]
}
```
#### 4.11.6 Shell::OnAnimatorDraw
[-> flutter/shell/common/shell.cc]
```
void Shell::OnEngineUpdateSemantics(
    blink::SemanticsNodeUpdates update,
    blink::CustomAccessibilityActionUpdates actions) {

  task_runners_.GetPlatformTaskRunner()->PostTask(
      [view = platform_view_->GetWeakPtr(), update = std::move(update),
       actions = std::move(actions)] {
        if (view) {
          view->UpdateSemantics(std::move(update), std::move(actions));
        }
      });
}
```
这个方法主要是向平台线程提交Semantic任务。

再回到小节4.6，可知接下来再执行finalizeTree()操作；

### 4.12 BuildOwner.finalizeTree
[-> lib/src/widgets/framework.dart]
```
void finalizeTree() {
  Timeline.startSync('Finalize tree', arguments: timelineWhitelistArguments);
  try {
    lockState(() {
      //遍历所有的Element，执行unmount()动作，且取消GlobalKeys的注册
      _inactiveElements._unmountAll();
    });
  } catch (e, stack) {
    _debugReportException('while finalizing the widget tree', e, stack);
  } finally {
    Timeline.finishSync();
  }
}
```
遍历所有的Element，执行相应具体Element子类的unmount()操作，下面以常见的StatefulElement为例来说明。

#### 4.12.1 StatefulElement.unmount
[-> lib/src/widgets/framework.dart]
```
void unmount() {
  super.unmount(); //[见小节4.12.2]
  _state.dispose(); //执行State的dispose()方法
  _state._element = null;
  _state = null;
}
```
#### 4.12.2 Element.unmount
[-> lib/src/widgets/framework.dart]
```
void unmount() {
  if (widget.key is GlobalKey) {
    final GlobalKey key = widget.key;
    key._unregister(this);  //取消GlobalKey的注册
  }
}
```

## 附录
本文涉及到相关源码文件
```
flutter/shell/common/
    - vsync_waiter.cc
    - engine.cc
    - animator.cc
    - shell.cc
    - rasterizer.cc

flutter/shell/platform/android/
    - vsync_waiter_android.cc
    - platform_view_android_jni.cc
    - library_loader.cc
    - io/flutter/view/VsyncWaiter.java

flutter/runtime/runtime_controller.cc
flutter/synchronization/pipeline.h
flutter/fml/message_loop_impl.cc
flutter/lib/ui/window/window.cc
flutter/lib/ui/window.dart
flutter/lib/ui/hooks.dart

lib/src/widgets/framework.dart
lib/src/widgets/binding.dart
lib/src/scheduler/binding.dart
lib/src/semantics/semantics.dart

lib/src/rendering/
    - binding.dart
    - object.dart
    - view.dart
```