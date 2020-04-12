---
title: Flutter 状态管理 0x00 - 基础知识及 State.setState 背后逻辑
comments: true
date: 2020-03-21 15:28
updated: 2020-03-21 15:28
tags:
- state_management
- design_patterns
categories:
- flutter
---

{% asset_img flutter_state_management_0x00.png %}

## 前言

> Emmmmm...正式开始文章前先交代个事儿，2019 开年计划那篇文章被我删了。原因没啥特别的，就真的只是脸疼...

嘛～ 好久没更 Blog 了，最近笔者正在学习 Flutter 相关的知识（貌似现在学也不算特别晚），所以后续**可能**会有一波连续的 Flutter 相关的更新（关键字已加粗 😂）。

因为之前我的文章大多是源码剖析相关的，所以这次决定换种方式先从 Flutter 开发的状态管理聊起。由于个人能力不足、水平有限，文章中难免有纰漏和谬误，加上工作确实比较忙可能没有时间校验，希望发现的朋友能够在文章下方评论交流讨论（主要是避免误人子弟，不过评论应该需要科学上网）。

本文会简单介绍 Flutter 以及声明式编程思想和代码画风，对比 `StatelessWidget` & `StatefulWidget` 这两个重要的 Widget，再聊聊 `setState` 背后的那些事儿。

## 索引

- Flutter 简介
- 声明式 UI & 响应式编程
- Flutter 如何渲染 Widget
- `StatelessWidget` & `StatefulWidget`
- `State.setState` 背后的那些事儿

## Flutter 简介

> [Flutter](https://flutter.dev/) 是 Google 开源的 UI 工具包，帮助开发者通过一套代码库高效构建多平台精美应用，支持移动、Web、桌面和嵌入式平台。

「[Write once, run anywhere.](https://en.wikipedia.org/wiki/Write_once,_run_anywhere)」始终是大前端开发乃至整个程序界的一个永恒的话题。诚然 Flutter 的目标也是如此，不过它与之前业界已经存在的 React Native、Xamarin、Hybrid 等框架有何不同？

{% asset_img flutter_system_overview.png %}

从上图可以看到 Flutter 的设计整体上有三层抽象：

- 最上层 Framework 封装好了 Material 和 Cupertino 这两种分别对应 Android 和 iOS 官方设计风格的 UI 库，除此之外还包含了渲染、动画、绘制、手势等等，这一层主要是提供给开发者便捷构建 App 使用的，其在开发时为 [JIT](https://en.wikipedia.org/wiki/Just-in-time_compilation)、发布时为 [AOT](https://en.wikipedia.org/wiki/Ahead-of-time_compilation) 的特性兼顾了开发调试的便捷与线上运行的高性能；
- Engine 作为中间层，使用 C/C++ 实现了一套高性能、可移植的 Runtime，屏蔽了平台间差异的同时支撑了上层的 Flutter 应用。
- Embedder 由平台指定语言实现，提供了 Flutter 所需的事件循环、线程、渲染等基础 API。

Flutter 通过这套机制接管了最底层的系统接口，提供了一整套**从底层渲染逻辑到上层开发语言的完整解决方案**，使得它有着超越 React Native 的高保真、多端一致的体验，和超越 Web 容器的高性能渲染能力。

## 声明式 UI & 响应式编程

### 声明式 UI

{% asset_img ui-equals-function-of-state.png %}

这张图很好的描述了声明式 UI 的核心思想，简单来说就是通过 state 作为入参根据已经写好的构建 func 就能得到我们想要的 UI 效果。举个 🌰 更直观：

默认的 Flutter Demo 界面：

{% asset_img flutter_default_demo.png %}

对应的代码：

``` dart
return Scaffold(
  appBar: AppBar(
    title: Text(widget.title),
  ),
  body: Center(
    child: Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: <Widget>[
        Text(
          'You have pushed the button this many times:',
        ),
        Text(
          '$_counter',
          style: Theme.of(context).textTheme.display1,
        ),
      ],
    ),
  ),
  floatingActionButton: FloatingActionButton(
    // _incrementCounter 内部逻辑可忽略
    onPressed: _incrementCounter,
    tooltip: 'Increment',
    child: Icon(Icons.add),
  ),
);
```

> In flutter, everything is a widget.

Emmmmm...可以注意到上面的代码中 `children: <Widget>[...]` 一行，Flutter 开发中就是通过这种方式嵌套组合 Widget 来描述用户界面的。从这一角度讲，Widget 是对一部分用户界面的**不可变**描述。

默认的 Flutter Demo 跑起来很简单，推荐大家亲身体验。简单说一下我的感受，声明式 UI 的优势很大，具体表现为只需要在写 UI 时将不同 state 对应的 UI 展示考虑并描述清楚就可以省去后续 state 变更时命令式 UI 需要手动管理视图层级（新增或删除视图元素）和更新属性（颜色、字号等）等麻烦。

> 值得一提的是 SwiftUI 也使用了声明式 UI 的思想（声明式 UI 才是未来）。

### 响应式编程

> [响应式编程](https://en.wikipedia.org/wiki/Reactive_programming) ，在计算机领域，响应式编程是一个专注于数据流和变化传递的异步编程范式。
这意味着可以使用编程语言很容易地表示静态（例如数组）或动态（例如事件发射器）数据流，
并且在关联的执行模型中，存在着可推断的依赖关系，这个关系的存在有利于自动传播与数据流有关的更改。

响应式编程对于大家来说应该早已不是什么新鲜事物了，单在移动客户端领域就有诸如 [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)、[RxSwift](https://github.com/ReactiveX/RxSwift)、[RxJava](https://github.com/ReactiveX/RxJava)、RxXxx...简单来说这种编程思想或者说范式下开发者只需关注可能存在的数据状态以及与之对应的逻辑从而大大减轻了维护这些对应关系与状态细节的工作负担。因为用过的人都说好，所以目前渐渐被推为移动客户端乃至整个大前端的主流编程思想。

不难看出其实声明式 UI 和响应式编程从思想上是完全契合的，我们只需要将数据流对应到 `UI=f(state)` 公式中的 state 就可以了。

## Flutter 如何渲染 Widget

{% asset_img how_flutter_render_widget.png %}

在介绍 `StatelessWidget` 与 `StatefulWidget` 之前先要了解一下 Flutter 的渲染模型，简单来讲 Flutter 在渲染 Widget 时用到三棵树：

- Widget，负责描述 Element 的配置。
- Element，负责管理 Widget 和 RenderObject 的生命周期。
- RenderObject，负责控制尺寸，布局和绘制工作。

一言以蔽之，Element 会根据我们书写的 Widget 对其的配置描述来生成并管理对应的 Element 与 RenderObject，由 RenderObject 负责最终的绘制工作。 

> NOTE: Element 会根据 Widget 的描述最大程度复用现有 Element 与 RenderObject 以提升性能。

## `StatelessWidget` & `StatefulWidget`

### `StatelessWidget`

> `StatelessWidget`, A widget that does not require mutable state.

`StatelessWidget` 如其名 `Stateless`，它不需要追踪 State 并根据 State 更新 UI，所以一般 `StatelessWidget` 内部的属性用 `final` 修饰声明且构造方法一般以 `const` 修饰。

> NOTE: `const` 修饰的 Widget 构造方法使用时可以提效。

### `StatefulWidget`

> `StatefulWidget`, A widget that has mutable state.

`StatefulWidget` 与 `StatelessWidget` 一样，也可以通过一系列（组合嵌套）其他更具体描述 UI 的 Widget（比如 `Text`）来构建部分用户界面。区别在于 `Stateful`，即它需要追踪 State 并根据 State 更新 UI。

### 区别

`StatelessWidget` 与 `StatefulWidget` 大概是我们用 Flutter 技术栈开发 App 时最常打交道的两个 Widget 了。

共同点：

- 这两个 Widget 都可以用来通过对其他一系列 Widget 构建完成一部分用户界面的封装。

不同点：

- `StatelessWidget` 更适用于一些不需通过用户交互或其他原因通过 State 控制更新的 UI 封装。
- `StatefulWidget` 更适用于追踪某个会根据用户交互或其他因素影响的 State，并根据最新的 State 实时更新的 UI 封装。

此外，Flutter 对于 `StateLessWidget` 与 `StatefulWidget` 的绘制也有差异：

- `StatelessWidget` 通过 `StatelessElement.build` 触发 build
- `StatefulWidget` 通过 `StatefulElement.build` 触发 `State.build`

## `State.setState` 背后的那些事儿

> NOTE: 下面分析的 `State.setState` 源码版本为 Flutter SDK v1.9.1+hotfix.6。

我们还以 Flutter 默认 Demo 来分析，在 Demo 中我们点击界面右下角的 `FloatingActionButton` (就是蓝色带有 + 号的圆形按钮)之后会刷新界面，屏幕中间的 Text Widget 会显示按钮被按下的次数。

{% asset_img demo.gif %}

`FloatingActionButton` 点击之后调用了 `_incrementCounter` 方法：

``` dart
class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }
  ...
}
```

`_MyHomePageState._incrementCounter` 内部逻辑也仅仅是调用了 `State.setState` 而已。正如 Flutter 所宣传的公式 `UI = f(state)` 一样，开发者只需要调用 `State.setState` 传入 `VoidCallback` 内部写好相关数据更新逻辑即可更新 UI，那么 Flutter 是如何做到这一点的呢？

### `State.setState` 背后逻辑

{% asset_img state_dot_set_state.png %}

> `State.setState` 背后调用嵌套较多，实际所做的事情理解起来却很简单。如不喜在文内阅读源码可直接跳转至「总结 `State.setState` 背后逻辑」小节看相关逻辑总结。

#### `State.setState`

``` dart
@optionalTypeArgs
abstract class State<T extends StatefulWidget> extends Diagnosticable {
  ...
  @protected
  void setState(VoidCallback fn) {
    // assert ...
    final dynamic result = fn() as dynamic;
    // assert ...
    _element.markNeedsBuild();
  }
  ...
}
```
可见 `State.setState` 内除去断言的逻辑只有两行代码：

- 执行入参 `VoidCallback` 逻辑，Demo 中对应 `_counter++;`
- 执行 `_element.markNeedsBuild();`

#### `Element.markNeedsBuild`

``` dart
abstract class Element extends DiagnosticableTree implements BuildContext {
  ...
  void markNeedsBuild() {
    // assert ...
    if (!_active)
      return;
    // assert ...
    if (dirty)
      return;
    _dirty = true;
    owner.scheduleBuildFor(this);
  }
  ...
}
```

`Element.markNeedsBuild` 内部的逻辑也很简单明了：

- 如此 Element 不再活跃则直接 `return` 无需做任何事
- 如此 Element 已经被标记为 `dirty` 也无需重复标记直接 `return`
- 以上条件不满足时说明此 Element 是活跃且需要重新构建的，所以标记为 `dirty` 后调用 `owner.scheduleBuildFor(this);`

#### `BuildOwner.scheduleBuildFor`

``` dart
class BuildOwner {
  ...
  void scheduleBuildFor(Element element) {
    // assert ...
    if (!_scheduledFlushDirtyElements && onBuildScheduled != null) {
      _scheduledFlushDirtyElements = true;
      onBuildScheduled();
    }
    _dirtyElements.add(element);
    element._inDirtyList = true;
    // assert ...
  }
  ...
}
```

`BuildOwner.scheduleBuildFor(this);` 内部逻辑也不复杂：

- `!_scheduledFlushDirtyElements && onBuildScheduled != null` 可以理解为如果还没有安排过刷新此 BuildOwner 下被标记为 Dirty 的 Elements 且安排构建的逻辑不为空，满足以上条件则将此 BuildOwner 安排刷新 Dirty Elements 的标记置为 `true`
- 将传入的 Element 加入此 BuildOwner 管理的 `_dirtyElements` 并将此 Element `_inDirtyList` 标记为 `true`

#### `BuildOwner.onBuildScheduled`

其实 `onBuildScheduled();` 跟下去是 BuildOwner 内部属性 `onBuildScheduled`，类型为 `VoidCallback`，在 `WidgetsBinding.initInstances` 时被赋值为 `WidgetsBinding._handleBuildScheduled`:

``` dart
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  ...
  @override
  void initInstances() {
    ...
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    ...
  }
  ...
}
```

`WidgetsBinding` 可以理解为 Widget 层与 Flutter engine 之间的胶水层，篇幅原因不展开讲，我们直接看 `WidgetsBinding._handleBuildScheduled`。

``` dart
mixin WidgetsBinding on BindingBase, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  ...
  void _handleBuildScheduled() {
    // assert ...
    ensureVisualUpdate();
  }
  ...
}
```

#### `SchedulerBinding.ensureVisualUpdate`

``` dart
mixin SchedulerBinding on BindingBase, ServicesBinding {
  ...
  void ensureVisualUpdate() {
    switch (schedulerPhase) {
      case SchedulerPhase.idle:
      case SchedulerPhase.postFrameCallbacks:
        scheduleFrame();
        return;
      case SchedulerPhase.transientCallbacks:
      case SchedulerPhase.midFrameMicrotasks:
      case SchedulerPhase.persistentCallbacks:
        return;
    }
  }
  ...
}
```

这里通过当前 `SchedulerPhase` 的状态进行取舍，仅在 `SchedulerPhase.idle` & `SchedulerPhase.postFrameCallbacks` 时调用 `scheduleFrame();`。

#### `SchedulerPhase.scheduleFrame`

``` dart
mixin SchedulerBinding on BindingBase, ServicesBinding {
  ...
  void scheduleFrame() {
    if (_hasScheduledFrame || !_framesEnabled)
      return;
    // assert ...
    window.scheduleFrame();
    _hasScheduledFrame = true;
  }
  ...
}
```

这里的逻辑更简单：

- `_hasScheduledFrame || !_framesEnabled` 可理解为如果已经安排过 Frame 或当前 Frame 不可用，这种情况直接 `return`
- 不满足上述情况则调用 `window.scheduleFrame();` 然后标记 `_hasScheduledFrame` 为 `true`

#### `Window.scheduleFrame`

``` dart
void scheduleFrame() native 'Window_scheduleFrame';
```

这里的 `Window.scheduleFrame` 是 Native 方法，可以理解为通过 `Window` 与 Flutter engine 打交道，注册了 VSync 回调。

### 总结 `State.setState` 背后逻辑

{% asset_img state_set_state_logic.png %}

简单来说 `State.setState` 就干了这么几件事：

- 执行入参 `VoidCallback` 逻辑，即执行开发者写好的信息变更逻辑
- 尝试将当前 `StatefulWidget` 对应的 `StatefulElement` 标记为 `dirty`
- 通过 `BuildOwner.onBuildScheduled` 到 `SchedulerPhase.scheduleFrame` 再到 `Window.scheduleFrame` 一步步完成了 VSync 的回调注册

注意上面 `State.setState` 第二条主要逻辑就是把 State 关联的 Element 标记为 `dirty`。在 VSync 回调后会通过 Native 到 Flutter engine 调用 Flutter `_drawFrame` 方法，将之前标记为 `dirty` 的 Element 重新构建，最终会执行到开发者熟悉的 `State.build` 方法。

#### `State.build` 是如何被执行的

这里因为篇幅原因简单描述下后续逻辑，毕竟本文重点不在于 Flutter 如何渲染 Widget：

- State 是开发者重写 `StatefulWidget.createState` 返回的
- State 对应的 Element 是 `StatefulWidget.createElement` 返回的，类型为 `StatefulElement`
- `StatefulElement` 被标记为 `dirty` 后在 Flutter `_drawFrame` 时重新构建会调用 `StatefulElement.build`，其源码为 `Widget build() => state.build(this);`
- `state.build(this)` 中的 `state` 就是我们的 State，开发者写的 `State.build` 方法就这样被执行了

## 总结

本文是 Flutter 状态管理的开篇，为了照顾一些还没来得及学习 Flutter 或者刚入门 Flutter 的初心者所以文章从前到后做了一些铺垫介绍过渡。文章内容也比较浅显易懂，基本上是围绕 Flutter 创建 App 默认的 Demo 来展开讲解一些基本的 Flutter 知识点：

- Flutter 渲染 Widget 的三颗树概念
- `StatelessWidget` & `StatefulWidget`
- `State.setState` 背后逻辑

由于状态管理这个话题非常大且复杂，文章因为篇幅原因就到这里，后续的文章（如果有的话）应该不会再花大篇幅做 Flutter 基础知识的铺垫和过渡了（但是该有的前置知识点肯定还会有）。时间紧张，文章难免出现谬误，估计自己也没有时间做校对了，有问题还望在评论区提醒。

### 扩展补充

写到这里我意识到 Flutter 状态管理这个话题下可以引出很多值得挖掘的内容，而且笔者坚信声明式 UI + 响应式编程会是未来移动客户端乃至整个大前端的主流编程范式。鉴于此，后续的文章计划围绕 Flutter 状态管理逐步深挖，内容也将在 Flutter 技术栈的基础上做适度横向对比。

Emmmmm...后续的文章将会发布在我的《[Flutter 状态管理](https://xiaozhuanlan.com/flutter_state_management)》小专栏。至于专栏定价嘛，目前随便定了 ¥69（作为小透明的我瑟瑟发抖 ing~）。其实倒也不是很在意能卖出多少份，或者从中收到多少钱，毕竟自己写文章的精力拿去随便接个朋友的私活都应该比这次通过小专栏赚到的多。比起收入更多的是对自己的鞭策吧，毕竟是收费的东西，只要有一个人订阅了这个专栏就意味着对我的肯定，我就有责任把这个方向挖掘的更深入，同时文章质量也应该比博客内容更高。

> 其实早在去年就有幸被小专栏的开发者[@安卓大王子](https://weibo.com/apkbus?is_all=1)邀请去小专栏写文章，不过当时自己感觉没什么好的内容方向所以只是开通了专栏但没有任何输出。时至今日终于找到了一个值得深挖的方向，自然而然想到了这个平台。个人感觉[小专栏](https://xiaozhuanlan.com/)这个平台没有太重的商业化味道，这点很讨喜，在微信上面的阅读体验也比自己发公众号要好一些，更难能可贵的是省去了发公众号的繁琐操作流程。可能正是商业化氛围弱的原因，上面的文章质量还都挺不错的，推荐各位读者朋友可以上去试读或者写作。
