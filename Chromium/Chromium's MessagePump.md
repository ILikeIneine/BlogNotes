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
4. WaitForWork(next_work_info)

这里的逻辑大概就是，如果有可以执行的任务，就执行它并获取下一次任务的时间（NextTaskInfo），设置下一次任务的时间。当所有需要立即执行（immediate task）的任务执行完毕后（即获取到的下一次任务的属性*is not immediate*），开始执行idle任务。

idle任务全部执行完，并且没有更多的任务，就等待（WaitForWork）。然后重复上面的过程。

#### ProcessNextWindowsMessage

具体地，先处理windows的消息，peekmessage，translate message然后dispatch messasge之类的，过程中顺带通知观察者，这些设计里观察者确实蛮多的，有些操作需要被及时观测到以做出对应的处理，chromium的blogs里面有讲到这一部分。

#### DoWork

然后DoWork，这里的delegate是一个thread_local变量， 在这里[ThreadControllerWithMessagePumpImpl](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.h;drc=7fa0c25da15ae39bbd2fd720832ec4df4fee705a;l=108?q=main_thread_only&ss=chromium%2Fchromium%2Fsrc)
override 了`DoWork`。

`DoWork`通过`DoWorkImpl`获取下一次任务唤醒的信息。如果获取不到，意味着没有更多任务了。设置下一次唤醒时间为TimeTicks::Max()。

如果获取到的下一次的任务不是immediate，就`DoIdleWork`。

#### DoWorkImpl

`DoWorkImpl`大致上是一个循环取任务的过程，条件是：

- 指定g_run_tasks_by_batches（批量执行任务flag）
- batch_duration 没有超时（批量执行任务的最大时间）
- work_batch_size 没有到达上限（可批量执行任务数量）

这个g_run_tasks_by_batches~~我不知道是个什么几把语义~~是允许批量执行任务，大致上，如果以上条件满足，就从`main_thread_only`里取可以执行的任务然后交给`task_annotator_`去RunTask，直到循环条件结束，并且批量任务执行完之后，返回下一次的取任务的等待信息给`DoWork`。

具体上，循环重复以下三个过程：

1. 通过`main_thread_only().task_source->SelectNextTask()`选择一些可以执行的任务进行执行

2. 调用 `task_annotator_.RunTask()` 执行任务

3. 最后通过`main_thread_only().task_source->GetPendingWakeUp()` 获取下一次要执行的任务。返回给消息循环作为休眠时间的参考。



为了具体地分析以上三个流程，首先需要了解一些概念。

- MainThreadOnly

  MainThreadOnly是一个结构体，存在于很多类中定义为一个inner class。`MainThreadOnly`中存放的数据一般和当前线程绑定，并且只在当前线程中使用。可以不用加锁访问。

- AnyThread

  与MainThreadOnly语义上相对的是AnyThread。这个结构体可以被其他任意线程访问（需要持锁）。代码中也可以看到，AnyThread是通过编译器指令指定了[`GUARDED_BY(any_thread_lock)`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=589;)互斥锁。

- [SequenceManager](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/base/task/sequence_manager/README.md)

  有关SequenceManager可以结合[有关分析](./Chromium's%20Sequence%20Manager.md)。

  `SequenceManager`的`MainThreadOnly`中有一个结构[`WakeUpQueue`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/wake_up_queue.h;l=109;)，而`WakeUpQueue`通过一个最小堆`IntrusiveHeap<ScheduledWakeUp, std::greater<>>`来保存多个`ScheduledWakeUp`，这确保delay时间最短的在最上面。

    `ScheduledWakeUp`持有`TaskQueueImpl`对象。`TaskQueueImpl`中的`MainThreadOnly`和`Anythread`对象分别持有：

  - [MainThreadOnly::delayed_work_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=424;)

  - [MainThreadOnly::immediate_work_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=425;)

  - [MainThreadOnly::delayed_incoming_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=426;)

  - [Anythread::immediate_incoming_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=575;)

  也就是说一个`SequenceManagerImpl`可以管理多个`TaskQueueImpl`。

    当延迟任务提交后，会进入到`delayed_incoming_queue`，当到达可执行时间后提交到`delayed_work_queue`，然后去执行。立即执行的任务先提交到`immediate_incoming_queue`，然后由目标线程提交到`immediate_work_queue`，再去执行。

##### 过程【1】：SelectNextTask

`main_thread_only().task_source->SelectNextTask()`这里的task_resource是被[sequence_manager_impl](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/sequence_manager_impl.cc;l=201;)通过SetSequencedTaskSource注入的，所以task_source 为SequenceManagerImpl。

```c++

absl::optional<SequenceManagerImpl::SelectedTask>
SequenceManagerImpl::SelectNextTaskImpl(LazyNow& lazy_now,
                                        SelectTaskOption option) {
  //...

  // 当MainThreadOnly-> immediate_work_queue为空时候尝试从AnyThread->immediate_incoming_queue获取任务
  ReloadEmptyWorkQueues();
  // 把到时间的delayed_incoming_queue的task移到delayed_work_queue
  MoveReadyDelayedTasksToWorkQueues(&lazy_now);

  // If we sampled now, check if it's time to reclaim memory next time we go
  // idle.

  // ...

  while (true) {
    // 选择一个WorkQueue
    internal::WorkQueue* work_queue =
        main_thread_only().selector.SelectWorkQueueToService(option);
    TRACE_EVENT_OBJECT_SNAPSHOT_WITH_ID(
        TRACE_DISABLED_BY_DEFAULT("sequence_manager.debug"), "SequenceManager",
        this,
        AsValueWithSelectorResultForTracing(work_queue,
                                            /* force_verbose */ false));


    if (!work_queue)
      return absl::nullopt;

    // If the head task was canceled, remove it and run the selector again.
    if (UNLIKELY(work_queue->RemoveAllCanceledTasksFromFront()))
      continue;

    if (UNLIKELY(work_queue->GetFrontTask()->nestable ==
                     Nestable::kNonNestable &&
                 main_thread_only().nesting_depth > 0)) {
      // Defer non-nestable work. NOTE these tasks can be arbitrarily delayed so
      // the additional delay should not be a problem.
      // Note because we don't delete queues while nested, it's perfectly OK to
      // store the raw pointer for |queue| here.
      internal::TaskQueueImpl::DeferredNonNestableTask deferred_task{
          work_queue->TakeTaskFromWorkQueue(), work_queue->task_queue(),
          work_queue->queue_type()};
      main_thread_only().non_nestable_task_queue.push_back(
          std::move(deferred_task));
      continue;
    }

    // ...
    main_thread_only().task_execution_stack.emplace_back(
        work_queue->TakeTaskFromWorkQueue(), work_queue->task_queue(),
        InitializeTaskTiming(work_queue->task_queue()));

    ExecutingTask& executing_task =
        *main_thread_only().task_execution_stack.rbegin();
    NotifyWillProcessTask(&executing_task, &lazy_now);

    // Maybe invalidate the delayed task handle. If already invalidated, then
    // don't run this task.
    if (!executing_task.pending_task.WillRunTask()) {
      executing_task.pending_task.task = DoNothing();
    }

    return SelectedTask(
        executing_task.pending_task,
        executing_task.task_queue->task_execution_trace_logger(),
        executing_task.priority, executing_task.task_queue_name);
  }
}
```

##### 过程【2】：task_annotator_.RunTask

取任务大致流程如上，接下来分析annotator运行任务的逻辑。

具体代码可以看[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=458;)，就是tracing 和 run。

##### 过程【3】：GetPendingWakeUp()

最后再来分析`task_source->GetPendingWakeUp()`，这里实际上调用的是[`SequenceManagerImpl::GetPendingWakeUp()`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/sequence_manager_impl.cc;).

注释写的比较清晰了，跳转逻辑最后调用的是`main_thread_only().wake_up_queue->GetNextDelayedWakeUp()`，跳转到[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/wake_up_queue.cc;l=120;)，取最小堆顶部的WakeUp，获取下次唤醒时间。

既然从获取wake_up_queue_的堆顶`TaskQueue`的唤醒时间，那么这个wake_up_queue_是怎么更新的呢？就是在我们每次取消息的时候，即：

调用栈：
`SequenceManagerImpl::SelectNextTaskImpl()` -> `SequenceManagerImpl::MoveReadyDelayedTasksToWorkQueues()` -> `WakeUpQueue::MoveReadyDelayedTasksToWorkQueues()` -> `TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue()`

```c++
void WakeUpQueue::MoveReadyDelayedTasksToWorkQueues(
    LazyNow* lazy_now,
    EnqueueOrder enqueue_order) {
  bool update_needed = false;
  while (!wake_up_queue_.empty() &&
         wake_up_queue_.top().wake_up.earliest_time() <= lazy_now->Now()) {
    internal::TaskQueueImpl* queue = wake_up_queue_.top().queue;
    // OnWakeUp() is expected to update the next wake-up for this queue with
    // SetNextWakeUpForQueue(), thus allowing us to make progress.
    // 这里唤醒队列
    queue->OnWakeUp(lazy_now, enqueue_order);
    update_needed = true;
  }

  if (!update_needed || wake_up_queue_.empty())
    return;
  // If any queue was notified, possibly update following queues. This ensures
  // the wake up is up to date, which is necessary because calling OnWakeUp() on
  // a throttled queue may affect state that is shared between other related
  // throttled queues. The wake up for an affected queue might be pushed back
  // and needs to be updated. This is done lazily only once the related queue
  // becomes the next one to wake up, since that wake up can't be moved up.
  // `wake_up_queue_` is non-empty here, per the condition above.
  internal::TaskQueueImpl* queue = wake_up_queue_.top().queue;
  queue->UpdateWakeUp(lazy_now);
  while (!wake_up_queue_.empty()) {
    internal::TaskQueueImpl* old_queue =
        std::exchange(queue, wake_up_queue_.top().queue);
    if (old_queue == queue)
      break;
    queue->UpdateWakeUp(lazy_now);
  }
}
```

循环中判断堆顶的任务队列是否已经可以唤醒，可执行的话调用`TaskQueueImpl::OnWakeUp()`。

然后是`TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue()`。

> Enqueue all delayed tasks that should be running now, skipping any that have been canceled.

接着在最后调用`TaskQueueImpl::UpdateWakeUp()`获取下一次唤醒时间并将该时间设置为下一次唤醒时间，在`WakeUpQueue::SetNextWakeUpForQueue()`里面涉及到调整堆结构。具体，把最新的`WakeUp`信息添加到`wake_up_queue_`。

在以上这些操作之后，这可能导致队列中的信息变乱，需要更新（update_needed）其他的wakeup信息。

如果更新前后的wake_up时间不一样，会通过`DefaultWakeUpQueue::OnNextWakeUpChanged()`通知观察者。

然后通过`SequenceManagerImpl::SetNextWakeUp()`调整任务调度,判断是否立刻唤醒`ThreadControllerWithMessagePumpImpl::ScheduleWork()`

或者设置下次唤醒时间`ThreadControllerWithMessagePumpImpl::SetNextDelayedDoWork()`

以上这些最后都会控制message_pump去执行对应的操作，通过`MessagePumpForUI::ScheduleWork()`Post一条windows message`kMsgHaveWork`，告诉有消息来了。

### MessagePumpForIO

`MessagePumpForIO` 是windows下IO线程的 message pump 的具体实例。

任务投递的逻辑就放到[下一篇](./Chromium's%20Task%20Posting.md)。
