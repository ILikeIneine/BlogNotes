# Chromium's MessagePump

在[讨论](https://groups.google.com/a/chromium.org/g/chromium-dev/c/Alycg9BUDEM)上看到这样一个解释:

> MessageLoop is a task runner. It keeps a queue of Tasks, but it may also have other messages (platform-specific UI events, or platform-specific IO events) that it handles. Those platform specific messages are handled by the MessagePumps. Therefore, MessageLoop depends on MessagePumps.

MessageLoop是一个`task runner`，保存了一个各种任务的队列，（基于平台的 UI events 或者基于平台的 IO events）。这些基于平台的消息是由MessagePumps管理的。因此`MessageLoop`依赖于`MessagePumps`。

由于这个回答的时间为2011年，考虑到时效性，不具有多大参考价值。

刚接触到这一块，我也不太明白其中的复杂的逻辑。所以这篇可能仅仅是一个繁琐注释的翻译解析（并且不那么准确）

## Background

每个`base::Thread`（除了主线程），在创建的时候都会拥有一个`MessageLoop`。每个进程至少有两个线程。

一个主线程，在浏览器进程中为`UI thread`，在`Blink`渲染进程中为`Blink main thread`，

和一个`IO thread`。

还会有根据其他需求创建的一些特殊用途的线程或者*线程池*。

大多数线程有一个`loop`，这个循环从队列中获取`tasks`然后执行。

TODO: add more explainations

## [message_pump.h](https://source.chromium.org/chromium/chromium/src/+/main:base/message_loop/message_pump.h)

Data: 2023/11/6

`MessagePump`的基类，inner class 有

- `MessagePump::Delegate`
- `MessagePump::ScopedDoworkItem`。

不知到谷狗为什么喜欢搞这么多inner class，基本每个坨坨里都要塞点。

先看`MessagePump`，主要有四种[`MessagePumpType`(click to view)](https://source.chromium.org/chromium/chromium/src/+/main:base/message_loop/message_pump_type.h;drc=4a0d413ada84db50379ab8f35ec2630cd16c800b;l=15):

- *DEFAULT*

    This type of pump only supports tasks and timers.

- *UI*

    This type of pump also supports native UI events (e.g Windows messages).

- *CUSTOM*

    User provided implementation of MessagePump interface.

- *IO*

    This type of pump also supports asynchronous IO.

并且提供了`Create`静态[Factory Implementation](https://source.chromium.org/chromium/chromium/src/+/main:base/message_loop/message_pump.h;l=37):

```c++
// Creates the default MessagePump based on |type|. Caller owns return value.
static std::unique_ptr<MessagePump> Create(MessagePumpType type);
```

根据`type`创建对应的`MessagePump`。

## [message_pump_win.h](https://source.chromium.org/chromium/chromium/src/+/main:base/message_loop/message_pump_win.h)

由于开发环境是Wendous，这里讲的是`MessagePump`在wendows下面的头文件。

`MEssagePumpWin`是继承自`MessagePump`的，作为windows下`MessagePump`的基础。

有一个inner class`RunState`记录了MessagePump的状态。

### MessagePumpForUI

`MessagePumpForUI` 是windows下UI线程的 message pump 的具体实例。
不过messagepump会在初始化的时候，create一个message windows，接受windows的系统窗口消息。

`DoRunLoop`里是处理消息的死循环，主要是：

1. ProcessNextWindowsMessage()
2. delegate->DoWork()
3. delegate->DoIdleWork()

这里的逻辑大概就是，如果有可以执行的任务，就执行它并获取下一次任务的时间（NextTaskInfo），设置下一次任务的时间。

当所有需要立即执行（immediate task）的任务执行完毕后（即获取到的下一次任务的属性*is not immediate*），开始执行idle任务。

idle任务全部执行完，并且没有更多的任务，就等待（WaitForWork）。然后重复上面的过程。

具体地：

先处理windows的消息，peekmessage，translate message然后dispatch messasge之类的，过程中顺带通知观察者，这些设计里观察者确实蛮多的，有些操作需要被及时观测到以做出对应的处理，chromium的blogs里面有讲到这一部分。

然后DoWork，这里的delegate是一个thread_local变量， 在这里[ThreadControllerWithMessagePumpImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.h;drc=7fa0c25da15ae39bbd2fd720832ec4df4fee705a;l=108?q=main_thread_only&ss=chromium%2Fchromium%2Fsrc)
override 了`DoWork`。

`DoWork`通过`DoWorkImpl`获取下一次任务唤醒的信息。如果获取不到，意味着没有更多任务了。设置下一次唤醒时间为TimeTicks::Max()。

如果获取到的下一次的任务不是immediate，就`DoIdleWork`。

`DoWorkImpl`主要三个工作：

1. 通过`main_thread_only().task_source->SelectNextTask()`选择一些可以执行的任务进行执行

2. 调用 `task_annotator_.RunTask()` 执行任务

3. 最后通过`main_thread_only().task_source->GetPendingWakeUp()` 获取下一次要执行的任务。返回给消息循环作为休眠时间的参考。

如果可以，就进入批量执行任务，批量执行任务是一个循环，会不停的从task_source中获取任务。

这个g_run_tasks_by_batches~~我不知道是个什么几把语义~~是允许批量执行任务，如果满足条件，就从`main_thread_only`里取任务然后交给`task_annotator_`去run。

条件是：

1. 如果指定了g_run_tasks_by_batches（批量执行任务）
2. batch_duration 没有超时（批量执行任务的最大时间）
3. work_batch_size 没有到达上限（可批量执行任务数量）

完了返回下一次的任务的信息到`DoWork`。这里如果NextTask不是immediate的话就delay，并设置一下delay的时间。

#### MainThreadOnly

main_thread_only() 获取到的MainThreadOnly代表只在当前线程中使用的数据， 可以不用加锁访问。

TODO: explaination

### MessagePumpForIO

`MessagePumpForIO` 是windows下IO线程的 message pump 的具体实例。
