---
layout: post
title: "RunLoop深度探究（四）"
date: 2016-01-08 18:45:07 +0800
comments: true
categories: ios
keywords: RunLoop, 官方译文, RunLoop深度探究
Description: 对RunLoop的官方文档进行翻译并加以解释

---

原文链接：[Run Loops](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1)

Run loops 是与线程相关联的基础设施的一部分。Run loop 是用来调度工作并且协调传入事件的时间处理循环。run loop 的目的是：让你的线程在有工作的任务的事后保持忙碌，并且在空闲的事后使线程保持休眠。

<!--more-->

Run loop 的管理并不是完全自动的，你仍然需要设计你的线程代码，并利用这些代码在合适的时机开启 run loop 并且相应传入的事件。Cocoa 和 Core Foundation 框架都提供了run loop 对象，以便于帮助你配置和管理你的线程 run loop。你的应用不需要显式的创建这些 run loop 对象。每条线程，包括应用的主线程，都会配有一个相关联的 run loop 对象。只有子线程们需要显式的（手动）运行它们的 run loop。然而，该 app 框架将会自动设置并且运行作为应用程序启动过程的一部分的处于 main thread （主线程）的 run loop。

接下来，我们一起看看关于 run loop 以及它们的配置相关的更多信息。参照[NSRunLoop Class Reference](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/doc/uid/TP40003725) 以及 [CFRunLoop Reference](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/doc/uid/20001441-CH1g-112707)。

### Run Loop 剖析

run loop 是一个名副其实的循环。他会是你的线程不停地循环工作，具体包括线程的进入并且用来响应输入源所运行的事件处理程序。你的代码提供的控制状态将会被用来实现run loop具体的循环部分。换句话说，你的代码需要提供 while 或 for 循环来驱动你的 run loop。在你的 run loop 中，你使用一个 run loop 对象去“执行”接收事件的事件处理代码和调用已经安装好的 handlers。

run loop 通过两种不同类型的 source 来接收事件。Input source 传递异步事件，通常包括从其他线程或者其他应用发来的 message 。Timer source 也传递异步事件，通常发生在一个被安排好的时间或者重复的时间间隔。这两种类型的 source 都会使用 application-specific 处理例程来处理到来的事件。

插图3-1展示了 run loop 各种 source 的概念结构。input source 传递异步事件到相对应的 handler（处理程序）并且引起 `runUtilDate:` 方法（调用线程相关的 NSRunLoop 对象）退出。

_插图3-1 run loop 和它的 source 结构_
![插图3-1](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/Art/runloop.jpg)

除了处理 input source，run loop 同样也会生成一些关于 run loop 的行为的通知。注册 run-loop observers（观察者）去接收这些通知并且使用他们去在线程上做额外的处理。你可以使用 Core Foundation 框架去在你的线程上安装 run-loop observer 。

下面这些段落提供了更多地关于 runloop的组成 和 它们可以操作的modes，它们还描述了在事件处理的不同时间点生成的通知。

### Run Loop Modes
一个 run loop mode 是一个包含 input sources 和 被监视的 timers 和 将要被通知的 run-loop 的observer 所组成的集合。每次你启动你的 run loop 时，你都会指定（隐式或者显式的）一种特定的“mode”来运行。在执行该运行循环的期间，只有与指定运行的 mode 相关联的 source 才会被检测和允许去发送他们的事件。（相同的，只有与指定运行的 mode 相关联的 observer 会被 run loop 的进度（progress）所通知）。关联到其他 modes 的 sources 直到 run loop 切换到对应的 mode 时才会继续让新的事件通行（只有到对应的mode下，相关的 source 才会继续其对应的功能）。

在你的代码中，可以使用名字来标记 modes，包括 Cocoa 和 Core Foundation 都定义了一些默认的mode和一些常用的mode，代码中你可以通过字符串来区别这些mode，你可以定义自定义的mode，仅仅通过指定简单的字符串来标记这些自定义的mode。尽管可以随意的对 mode 名字进行复制，但是这些 mode 的内容却不行。你必须确保你添加到得任何你创建的 mode 中的一个或多个input sources, timers, 或者 run-loop observer都是有用的。

你使用 modes 并通过你的 run loop 特定的扫描中过滤掉干扰的事件。大多数情况下，你都会将你的 run loop 运行在系统默认的 mode 中。模态面板可能会运行在“ modal ” mode 中。当处于这种 mode 下，只有和 source 关联的模态面板才可以发送事件到线程中去。对于子线程来说，你可能会使用自定义 modes 去防止优先级抵的 source 在时间要求严格的操作中发送事件。</br>
`
Note: modes 的区别取决于 event（事件） 的 source ，不是 event 的类型。比如，你可能不会用 mode 去仅仅匹配一个鼠标点击事件或者键盘点击事件。 你可能会用 mode 去监听一组不同的端口。暂停 timer ，或者改变当前监视的 source ，然后，run loop observer 会立即开始监视。
`
</br>
列表3-1列出了 Cocoa 和 Core Foundation 框架中你可以使用的具有官方文档描述的一些标准的 mode，列名称列出了你在代码中需要指定 mode 时需要使用的实际常量。     

#### **Table 3-1** [预定义的 run loop modes](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW18)
<!--
| Mode | Name |Description|
| :------ | :------ | :------ |
| Default | [NSDefaultRunLoopMode](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/c/data/NSDefaultRunLoopMode) (Cocoa)</br>[kCFRunLoopDefaultMode](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/data/kCFRunLoopDefaultMode) (Core Foundation) | Default Mode 是用的最多的 mode 操作。在大多数情况下，你应该使用此模式启动 Run Loop 和配置您的 input source。 |
|Connection|`NSConnectionReplyMode`(Cocoa)|Cocoa 使用这个 mode 来 `NSConnection` 对象共同监视 replies （回复响应），你一般不需要自己使用该 mode |
|Modal|`NSModalPanelRunLoopMode`(Cocoa)|Cocoa 使用这个 mode 来标记确认用于 modal panel（面板、控制器）的事件|
|Event tracking|`NSEventTrackingRunLoopMode`(Cocoa)|Cocoa 使用这个 mode 来约束鼠标拖动循环和其他类型的用户界面跟踪循环的输入事件|
|Common modes |[NSRunLoopCommonModes](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/c/data/NSRunLoopCommonModes)(Cocoa)</br>[kCFRunLoopCommonModes](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/data/kCFRunLoopCommonModes) (Core Foundation)|这是一个 common mode 的可配置组，用该 mode 关联的某个 input source 同时也被组中的这些 mode 所关联。对于 Cocoa 的应用，这一套包括 default modal， 和 默认的 event tracking 这些 mode。Core Foundation 只包括开始的 default mode。你可以使用[CFRunLoopAddCommonMode](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/func/CFRunLoopAddCommonMode) function（函数）来添加自定义的 mode 到 mode 组中。|-->

<!--内嵌官方HTML-->

-----

<div class="tableholder"><table class="graybox" border="0" cellspacing="0" cellpadding="5"><caption class="tablecaption"><strong class="caption_number"><!--Table 3-1--></strong>&nbsp;&nbsp;<!--Predefined run loop modes--></br></br></caption><tbody><tr><th scope="col" class="TableHeading_TableRow_TableCell"><p>Mode</p></th><th scope="col" class="TableHeading_TableRow_TableCell"><p>Name</p></th><th scope="col" class="TableHeading_TableRow_TableCell"><p>Description</p></th></tr>
<tr><td scope="row"><p>Default</p></td><td><p><code><a href="https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/c/data/NSDefaultRunLoopMode" data-renderer-version="2" target="_self" onclick="s_objectID=&quot;https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoo_4&quot;;return this.s_oc?this.s_oc(e):true">NSDefaultRunLoopMode</a></code> (Cocoa)</p><p><code><a href="https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/data/kCFRunLoopDefaultMode" data-renderer-version="2" target="_self" onclick="s_objectID=&quot;https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index_2&quot;;return this.s_oc?this.s_oc(e):true">kCFRunLoopDefaultMode</a></code> (Core Foundation)</p></td><td><p> Default Mode 是用的最多的 mode 操作。在大多数情况下，你应该使用此模式启动 Run Loop 和配置您的 input source。 </p></td></tr><tr><td scope="row"><p>Connection</p></td><td><p><code><!--a target="_self" -->NSConnectionReplyMode<!--/a--></code> (Cocoa)</p></td><td><p>Cocoa 使用这个 mode 和<code><!--a target="_self" -->NSConnection<!--/a--></code>  对象共同监视 replies （回复响应），你一般不需要自己使用该 mode 
</p></td></tr><tr><td scope="row"><p>Modal</p></td><td><p><code><!--a target="_self" -->NSModalPanelRunLoopMode<!--/a--></code> (Cocoa)</p></td><td><p>Cocoa 使用这个 mode 来标记确认用于 modal panel（面板、控制器）的事件</p></td></tr><tr><td scope="row"><p>Event tracking</p></td><td><p><code><!--a target="_self" -->NSEventTrackingRunLoopMode<!--/a--></code> (Cocoa)</p></td><td><p>Cocoa 使用这个 mode 来约束鼠标拖动循环和其他类型的用户界面跟踪循环的输入事件 </p></td></tr><tr><td scope="row"><p>Common modes</p></td><td><p><code><a href="https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoop_Class/index.html#//apple_ref/c/data/NSRunLoopCommonModes" data-renderer-version="2" target="_self" onclick="s_objectID=&quot;https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSRunLoo_5&quot;;return this.s_oc?this.s_oc(e):true">NSRunLoopCommonModes</a></code> (Cocoa)</p><p><code><a href="https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/data/kCFRunLoopCommonModes" data-renderer-version="2" target="_self" onclick="s_objectID=&quot;https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index_3&quot;;return this.s_oc?this.s_oc(e):true">kCFRunLoopCommonModes</a></code> (Core Foundation)</p></td><td><p>这是一个 common mode 的可配置组，用 mode 关联的某个 input source 同时也被组中的这些 mode 所关联。对于 Cocoa 的应用，这一套包括 default modal， 和 默认的 event tracking 这些 mode。Core Foundation 只包括 开始的 default mode。你可以使用 <code><a href="https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index.html#//apple_ref/c/func/CFRunLoopAddCommonMode" data-renderer-version="2" target="_self" onclick="s_objectID=&quot;https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopRef/index_4&quot;;return this.s_oc?this.s_oc(e):true">CFRunLoopAddCommonMode</a></code> function 来添加自定义的 mode 到 mode 组中。</p></td></tr></tbody></table></div>

----

<!--内嵌官方HTML-->

### Input Sources

input source 异步传递事件到你的线程中。event 的source 取决于 input source 的类型，通常是两个类型中的一个。`基于 port 的 input source` 监视着你的应用中的 Mach Port 。`自定义的 input source` 监视着你的自定义的事件 source 。至于你的run loop 来讲，他不应该在是否是基于 port 的 input source 还是 自定义 input source 出现问题。系统通常会实现这两种全部允许程序员自己使用类型的 input source 。这两种 source 之间唯一的不同在于它们是如何被发送信号的。基于 port 的 source 可以通过 kernel 内核接收发送的信号，而自定义的 source 必须手动从另一个线程发送信号过来。

当你创建一个 input source 时，你可以给它分配一个或多个你的 run loop 的 mode 。受 mode 影响的 input source 会在任何给定的时刻受到监视，在大多数情况下，你的 run loop 会运行在 default mode 下，但是你也可以自己指派自定义的mode给它。如果一个 input source 没有处于当前受监视的 mode 下，那么任何它产生的事件都会被挂起，除非你的 run loop 在正确的 mode 下运行（就是让你的 run loop 以你需要监视的 input source 所属的 mode 开始运行，这样 run loop 才能监视到你的 input source 产生的事件）。


下面介绍一些 input source 。。。

#### 基于 port 的 source
Cocoa 和 Core Foundation 框架都提供了内置的支持--使用 port 相关的对象和函数来创建基于 port 的input source。例如，在 Cocoa 中，你根本不用自己直接创建一个 input source，你只需要创建一个 port 对象，并使用 [NSPort](https://developer.apple.com/library/ios/documentation/Cocoa/Reference/Foundation/Classes/NSPort_Class/index.html#//apple_ref/occ/cl/NSPort) 的方法去将 port 添加到 run loop 中去。port 对象会为你处理好创建和配置一个你需要的 input source 这些底层的事情。

在 Core Foundation 中， 你必须手动的两者都创建，其中包括 port 和 它的 run loop source ，在两种情况下， 你可以使用 port 不透明类型的 ([CFMachPortRef](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFMachPortRef/index.html#//apple_ref/c/tdef/CFMachPortRef), [CFMessagePortRef](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFMessagePortRef/index.html#//apple_ref/c/tdef/CFMessagePortRef), or [CFSocketRef](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFSocketRef/index.html#//apple_ref/c/tdef/CFSocketRef)) 相关的函数来创建合适的对象（you use the functions associated with the port opaque type (CFMachPortRef, CFMessagePortRef, or CFSocketRef) to create the appropriate objects）。

如何创建和配置自定义的基于 port 的源，可以参照[创建一个基于 port 的 input source](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-131281)
</br>
</br>
####自定义 input source
为了自定义一个 input source，你必须使用 Core Foundation 框架中 [CFRunLoopSourceRef](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopSourceRef/index.html#//apple_ref/c/tdef/CFRunLoopSourceRef)的不透明类型相关联的函数。你可以使用多个回调函数来配置你的自定义 input source ，Core Foundation 框架会配置你的  source 在不同的点去调用这些函数，处理将要发生的事件，并在 source 被移出 run loop 的时候销毁 source。

除了定义在事件到来时自定义 source 的行为，你还必须定义事件传递机制。source 的这部分运行在一个单独的线程中，并且负责 当数据已经准备好去被处理的时候，这部分会去提供一个拥有数据的 input source，并且发信号给 input source。这个事件传递机制取决于你，但是也不用过于复杂。

有关如何创建一个自定义 input source 的例子，可以参考[定义一个自定义 input source](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW3)， 关于自定义 input source 的详细信息， 可以参考 [CFRunLoopSourceReference](https://developer.apple.com/library/ios/documentation/CoreFoundation/Reference/CFRunLoopSourceRef/index.html#//apple_ref/doc/uid/20001443)

### Run Loop 的时间执行次序

每一次你运行 run loop 时，你的线程的 run loop 都会处理未完成的事件，并且为已经注册的任何观察者生成通知信息。下面是它执行的具体步骤：

1. 通知观察者 runloop 已经进入。
* 通知观察者 任何就绪的 timer 即将触发。
* 通知观察者 任何 不是基于端口的 input source （source0）即将触发
* 触发任何不是基于端口的并且准备就绪的 input source。
* 假如存在已经准备好的基于端口的 input source （source1），将会等待被触发。并立即开始处理事件。跳转至步骤 9 。
* 通知观察者 线程即将睡眠。
* 设置让线程在符合如下条件之一时从睡眠状态唤醒：
	* 一个基于端口的 input source（source1）产生的事件到来。 	
	* timer 触发了。
	* 超出 runloop 预定的时间。
	* run loop 明确唤醒
* 通知观察者，线程刚刚唤醒。
* 处理剩余的事件：
	*	如果一个用户定义的 timer 触发，处理 timer 的事件并且重启 ru loop ，跳转值步骤 2 。
	* 如果一个 input source 触发，传递这个事件。
	* run loop 已经明确地被唤醒但是还没有超时。重启 run loop，并且跳转至 第二步。
* 通知观察者，run loop 已经退出。
因为 timer 和 input source 发出的 observer 通知是在这些事件确实是发生时被发送的，那么在事件发生的时间和 observer 收到的通知的时间会存在时间间隙。如果对事件发生时间的准确性要求非常严格，那么你需要采取 `睡眠` 和 `唤醒睡眠` 的通知方式来帮助你纠正事件实际发生的时间。

由于定时器等周期性事件是在 run loop 运行的时候发送的，需要避免在发送这些事件时被打断。典型的例子比如：只要进入运行循环，并且反复实时地从应用中请求事件来实现对鼠标跟踪的常规动作。由于你的代码是直接抓取事件的，而不是像平时一样让应用分发这些事件的，所以 timer 是不能被触发的，除非鼠标追踪程序退出并返回到应用程序控制。

一个 run loop 可以使用 run loop 对象显示唤醒，其他的事件可能也会引起 run loop 的唤醒。比如，添加另一个 `不是基于端口的 input source`（source1） 唤醒 run loop 从而使 input source 可以被立即处理（相比于一直处于等待状态直到其他时间发生才唤醒）。

### 在什么时候适合使用 RunLoop ？
**`唯一需要你显示的使用 run loop 的时候是：当你为你的应用创建了子线程的时候。`**在你的应用中，主线程的 run loop 是基础设施中至关重要的一部分。所以，app 的框架都会提供运行主线程 loop 的代码并且自动开启 loop。iOS中[UIApplication](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIApplication_Class/index.html#//apple_ref/occ/cl/UIApplication)（或者OSX中`NSApplication`）的`run`方法会开启一个应用中的主 loop 作为应用程序启动步骤的一部分。如果你使用 Xcode 的项目模板来创建你的应用，你完全不用自已去显示的调用这些常规方法。

对子线程来讲，你需要决定是否一个 run loop 对它来说是必要的，如果是，需要你自己配置并且启动它。你并不需要在任何情况下都开启线程的 run loop。比如：你使用线程去执行一些长期的并且预先决定的任务的时候，你可能需要避免开启 run loop 。Run loop 通常服务于一些需要你和线程之间互动性更强的特殊场景。比如，你需要在以下这些场景中开启你的 run loop：

*	使用 port 或者自定义 input source 来和其他线程进行通信。
*	在线程中使用 timer 。
*	在 Cocoa 的应用中使用任何与 `performSelector…`相关的方法。
*	让你的线程继续执行周期性的任务。

如果你选择使用 run loop，配置和建立它是非常简单的。如同所有的多线程编程一样，你需要有计划的在合适的情况下退出子线程。让它（run loop）更好的退出而不是迫使它终止永远都是更好的更干净的结束线程的方法。关于如何配置并且退出的方法将在[使用Run Loop对象](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW5)中向大家展示。