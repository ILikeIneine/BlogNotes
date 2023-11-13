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

大致上，这里第一步的取任务是一个循环，条件是：

1. 如果指定了g_run_tasks_by_batches（批量执行任务）
2. batch_duration 没有超时（批量执行任务的最大时间）
3. work_batch_size 没有到达上限（可批量执行任务数量）

这个g_run_tasks_by_batches~~我不知道是个什么几把语义~~是允许批量执行任务，如果满足条件，就从`main_thread_only`里取可以执行的任务然后交给`task_annotator_`去RunTask。

批量任务执行完之后，返回下一次的取任务的等待信息给`DoWork`。

为了具体地分析以上三个流程，首先需要了解一些概念。

#### MainThreadOnly

MainThreadOnly是一个结构体，存在于很多类中定义为一个inner class。`MainThreadOnly`中存放的数据一般和当前线程绑定，并且只在当前线程中使用。可以不用加锁访问。

#### AnyThread

与MainThreadOnly语义上相对的是AnyThread。这个结构体可以被其他任意线程访问（需要持锁）。

#### [SequenceManager](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/base/task/sequence_manager/README.md)

`SequenceManager`的`MainThreadOnly`中有一个结构[`WakeUpQueue`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/wake_up_queue.h;l=109;)，而`WakeUpQueue`通过一个最小堆`IntrusiveHeap<ScheduledWakeUp, std::greater<>>`来保存多个`ScheduledWakeUp`，这确保delay时间最短的在最上面。

`ScheduledWakeUp`持有`TaskQueueImpl`对象。`TaskQueueImpl`中的`MainThreadOnly`和`Anythread`对象分别持有：

- [MainThreadOnly::delayed_work_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=424;)

- [MainThreadOnly::immediate_work_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=425;)

- [MainThreadOnly::delayed_incoming_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=426;)

- [Anythread::immediate_incoming_queue](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=575;)

也就是说一个`SequenceManagerImpl`可以管理多个`TaskQueueImpl`。

这里我们也可以看到AnyThread是通过编译器指令指定了[`GUARDED_BY(any_thread_lock)`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue_impl.h;l=589;)互斥锁。

当延迟任务提交后，会进入到`delayed_incoming_queue`，当到达可执行时间后提交到`delayed_work_queue`，然后去执行。立即执行的任务先提交到`immediate_incoming_queue`，然后由目标线程提交到`immediate_work_queue`，再去执行。


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

取任务大致流程如上，接下来分析annotator运行任务的逻辑。

具体代码可以看[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/thread_controller_with_message_pump_impl.cc;l=458;)，就是tracing 和 run。

最后再来分析`task_source->GetPendingWakeUp()`，这里实际上调用的是[`SequenceManagerImpl::GetPendingWakeUp()`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/sequence_manager_impl.cc;).

注释写的比较清晰了，跳转逻辑最后调用的是`main_thread_only().wake_up_queue->GetNextDelayedWakeUp()`

跳转到[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/wake_up_queue.cc;l=120;)，取最小堆顶部的WakeUp，获取下次唤醒时间。

这个是获取wake_up_queue_的堆顶taskqueue的唤醒时间，这个wake_up_queue_是怎么更新的呢？就是在我们每次取消息的时候，即：

`SequenceManagerImpl::SelectNextTaskImpl()` -> `SequenceManagerImpl::MoveReadyDelayedTasksToWorkQueues()` -> `WakeUpQueue::MoveReadyDelayedTasksToWorkQueues()`

循环中判断堆顶的任务是否已经可以执行，可执行的话调用`TaskQueueImpl::OnWakeUp()`。

然后是`TaskQueueImpl::MoveReadyDelayedTasksToWorkQueue()`。

> Enqueue all delayed tasks that should be running now, skipping any that have been canceled.

接着在最后调用

`TaskQueueImpl::UpdateWakeUp()`

获取下一次唤醒时间，然后设置下一次唤醒时间，在`WakeUpQueue::SetNextWakeUpForQueue()`里面涉及到调整堆结构。具体，把最新的`WakeUp`信息添加到`wake_up_queue_`。

在以上这些操作之后，这可能导致队列中的信息变乱，需要更新（update_needed）其他的wakeup信息。

如果更新前后的wake_up时间不一样，会通过`DefaultWakeUpQueue::OnNextWakeUpChanged()`通知观察者。

然后`SequenceManagerImpl::SetNextWakeUp()`调整任务调度,判断是否立刻唤醒

`ThreadControllerWithMessagePumpImpl::ScheduleWork()`

或者设置下次唤醒时间

`ThreadControllerWithMessagePumpImpl::SetNextDelayedDoWork()`

这些最后都会控制message_pump去执行对应的操作。

`MessagePumpForUI::ScheduleWork()`会Post一条message`kMsgHaveWork`，告诉有消息来了。

接下来看任务投递

为了将任务投递到主线程（UI）线程或者 IO 线程，使用`content::GetUIThreadTaskTunner({})` 或者`content::GetIOThreadTaskRunner({})`。你也可以提供额外的`BrowserTaskTraits`作为参数（这并不常见并且通常有特殊含义）

我们以通常用法`content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE, ...)`为例。

搜索一下源码，最后调用的都是：

```c++
// static
scoped_refptr<base::SingleThreadTaskRunner>
BrowserTaskExecutor::GetUIThreadTaskRunner(const BrowserTaskTraits& traits) {
  return Get()->GetTaskRunner(BrowserThread::UI, traits);
}
```

`GetTaskRunner()`的定义如下：

```c++
scoped_refptr<base::SingleThreadTaskRunner>
BaseBrowserTaskExecutor::GetTaskRunner(BrowserThread::ID identifier,
                                       const BrowserTaskTraits& traits) const {
  const QueueType queue_type = GetQueueType(traits);

  switch (identifier) {
    case BrowserThread::UI: {
      return browser_ui_thread_handle_->GetBrowserTaskRunner(queue_type);
    }
    case BrowserThread::IO:
      return browser_io_thread_handle_->GetBrowserTaskRunner(queue_type);
    case BrowserThread::ID_COUNT:
      NOTREACHED();
  }
  return nullptr;
}
```

这个`browser_ui_thread_handle_`是在`BrowserTaskExecutor::Create()`的时候构造出来的，具体就是`Create() -> CreateInternal() -> BrowserTaskExecutor()`

通过

```c++
BrowserTaskExecutor::BrowserTaskExecutor(
    std::unique_ptr<BrowserUIThreadScheduler> browser_ui_thread_scheduler,
    std::unique_ptr<BrowserIOThreadDelegate> browser_io_thread_delegate)
    // 1
    : ui_thread_executor_(std::make_unique<UIThreadExecutor>(
          std::move(browser_ui_thread_scheduler))),
      io_thread_executor_(std::make_unique<IOThreadExecutor>(
          std::move(browser_io_thread_delegate))) {
  // 2
  browser_ui_thread_handle_ = ui_thread_executor_->GetUIThreadHandle();
  browser_io_thread_handle_ = io_thread_executor_->GetIOThreadHandle();
  ui_thread_executor_->SetIOThreadHandle(browser_io_thread_handle_);
  io_thread_executor_->SetUIThreadHandle(browser_ui_thread_handle_);
}
```

具体分析一下，`BrowserUIThreadScheduler`:

```c++
BrowserUIThreadScheduler::BrowserUIThreadScheduler()
    : owned_sequence_manager_(
          // 1 
          base::sequence_manager::CreateUnboundSequenceManager(
              base::sequence_manager::SequenceManager::Settings::Builder()
                  .SetMessagePumpType(base::MessagePumpType::UI)
                  .SetCanRunTasksByBatches(true)
                  .SetPrioritySettings(
                      internal::CreateBrowserTaskPrioritySettings())
                  .Build())),
      // 2
      task_queues_(BrowserThread::UI, owned_sequence_manager_.get()),
      queue_enabled_voters_(task_queues_.CreateQueueEnabledVoters()),
      handle_(task_queues_.GetHandle()) {
  CommonSequenceManagerSetup(owned_sequence_manager_.get());
  owned_sequence_manager_->SetDefaultTaskRunner(
      handle_->GetDefaultTaskRunner());

  owned_sequence_manager_->BindToMessagePump(
      base::MessagePump::Create(base::MessagePumpType::UI));
  g_browser_ui_thread_scheduler = this;
}
```

【1】处`CreateUnboundSequenceManager()`最后调用的是

```c++
// static
std::unique_ptr<SequenceManagerImpl> SequenceManagerImpl::CreateUnbound(
    SequenceManager::Settings settings) {
  auto thread_controller =
      ThreadControllerWithMessagePumpImpl::CreateUnbound(settings);
  return WrapUnique(new SequenceManagerImpl(std::move(thread_controller),
                                            std::move(settings)));
}
```

这里我们看到之前分析获取任务和执行任务所熟悉的`ThreadControllerWithMessagePumpImpl`，在`SequenceManagerImpl`的构造函数中会将`controller_`的`TaskSource`设为`this`，这也符合我们上面获取task的分析。

【2】处的`task_queues_`的创建

`task_queues_`的类型是`BrowserTaskQueues`，构造代码比较长，具体看[这里](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/scheduler/browser_task_queues.cc;l=162;drc=654daa129ebb42d20055ecacdbfc76b8eec050d8;bpv=1;bpt=1)。

`BrowserTaskQueues`构造时候会调用上面在【1】的`SequenceManager`里创建多个`Queue`，并为每个`Queue`设置不同优先级。

最后创建了`Handle`对象。

首先看`SequenceManager`里`TaskQueue`的创建，最终调用到[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/sequence_manager_impl.cc;l=402;drc=7172fffc3c545134d5c88af8ab07b04fcb1d628e):

```c++
std::unique_ptr<internal::TaskQueueImpl>
SequenceManagerImpl::CreateTaskQueueImpl(const TaskQueue::Spec& spec) {
  DCHECK_CALLED_ON_VALID_THREAD(associated_thread_->thread_checker);
  std::unique_ptr<internal::TaskQueueImpl> task_queue =
      std::make_unique<internal::TaskQueueImpl>(
          this,
          spec.non_waking ? main_thread_only().non_waking_wake_up_queue.get()
                          : main_thread_only().wake_up_queue.get(),
          spec);
  main_thread_only().active_queues.insert(task_queue.get());
  main_thread_only().selector.AddQueue(
      task_queue.get(), settings().priority_settings.default_priority());
  return task_queue;
}
```

在`TaskQueueImpl()`里创建了对应的`TaskRunner`，`TaskQueueImpl()`第二个参数是`WakeUpQueue`，负责管理该`TaskQueueImpl`。

```c++
TaskQueueImpl::TaskQueueImpl(SequenceManagerImpl* sequence_manager,
                             WakeUpQueue* wake_up_queue,
                             const TaskQueue::Spec& spec)
    : name_(spec.name),
      sequence_manager_(sequence_manager),
      associated_thread_(sequence_manager
                             ? sequence_manager->associated_thread()
                             : AssociatedThreadId::CreateBound()),
      task_poster_(MakeRefCounted<GuardedTaskPoster>(this)),
      main_thread_only_(this, wake_up_queue),
      empty_queues_to_reload_handle_(
          sequence_manager
              ? sequence_manager->GetFlagToRequestReloadForEmptyQueue(this)
              : AtomicFlagSet::AtomicFlag()),
      should_monitor_quiescence_(spec.should_monitor_quiescence),
      should_notify_observers_(spec.should_notify_observers),
      delayed_fence_allowed_(spec.delayed_fence_allowed),
      default_task_runner_(CreateTaskRunner(kTaskTypeNone)) {
  UpdateCrossThreadQueueStateLocked();
  // SequenceManager can't be set later, so we need to prevent task runners
  // from posting any tasks.
  if (sequence_manager_)
    task_poster_->StartAcceptingOperations();
}
```

初始化完毕后，`return task_queue` 返回到上一层：

```c++
TaskQueue::Handle SequenceManagerImpl::CreateTaskQueue(
    const TaskQueue::Spec& spec) {
  return TaskQueue::Handle(CreateTaskQueueImpl(spec));
}
```

这里，代码发生过一些变动:
> [scheduler] Introduce TaskQueue::Handle which owns TaskQueue

最新的chromium引进了[`Handle`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue.h;l=105;drc=903cd05b772e5868c14f57ec3766b6192fc0104a;bpv=1;bpt=1)来管理`TaskQueue`。

`SequenceManager::Handle`内部持有`TaskQueueImpl`和一个`sequence_manager`的引用。

在没有这个`Handle`之前，`TaskRunner`是在这里创建的（通过impl->CreateTaskRunner()等等...）。

感觉这一系列都是依赖注入，一层一层的。

最后来看`BrowserTaskQueues::Handle`:

```c++
BrowserTaskQueues::Handle::Handle(BrowserTaskQueues* outer)
    : outer_(outer),
      control_task_runner_(outer_->control_queue_->task_runner()),
      default_task_runner_(outer_->GetDefaultTaskQueue()->task_runner()),
      browser_task_runners_(outer_->CreateBrowserTaskRunners()) {}
```

初始化了三个`TaskRunner`，分别从三个不同的`TaskQueueImpl获取`。`CreateBrowserTaskRunner`会初始化并获取上面通过`SeqenceManager::CreateTaskQueue()`创建的`TaskQueue`的里的`TaskRunner`:

```c++
std::array<scoped_refptr<base::SingleThreadTaskRunner>,
           BrowserTaskQueues::kNumQueueTypes>
BrowserTaskQueues::CreateBrowserTaskRunners() const {
  std::array<scoped_refptr<base::SingleThreadTaskRunner>, kNumQueueTypes>
      task_runners;
  for (size_t i = 0; i < queue_data_.size(); ++i) {
    task_runners[i] = queue_data_[i].task_queue->task_runner();
  }
  return task_runners;
}
```

这里`task_queue->task_runner()`的结果是：

```c++
const scoped_refptr<SingleThreadTaskRunner>& TaskQueueImpl::task_runner() const {
  return default_task_runner_;
}
```

这个`default_task_runner_`就是前面`TaskQueueImpl::TaskQueueImpl()`里通过`TaskQueueImpl::CreateTaskRunner(kTaskTypeNone)`初始化的。

```c++
TaskQueueImpl::TaskRunner::TaskRunner(
    scoped_refptr<GuardedTaskPoster> task_poster,
    scoped_refptr<const AssociatedThreadId> associated_thread,
    TaskType task_type)
    : task_poster_(std::move(task_poster)),
      associated_thread_(std::move(associated_thread)),
      task_type_(task_type) {}
```

回到`Handler`部分，最后通过`BrowserTaskQueue::GetBrowserTaskRunner()`，根据不同的`QueueType`选择对应的`TaskRunner`：

```c++
const scoped_refptr<base::SingleThreadTaskRunner>& GetBrowserTaskRunner(
        QueueType queue_type) const {
      return browser_task_runners_[static_cast<size_t>(queue_type)];
    }
```

以上是*任务投递*的前情提要，准备工作都分析到位，最后来看任务投递的具体逻辑。

搜索一下相关用法，大部分都是：`content::GetUIThreadTaskRunner({})->PostTask(FROM_HERE, ...)`

调用逻辑是`TaskQueueImpl::TaskRunner::PostTask()` -> `TaskRunner::PostDelayedTask()` -> `TaskQueueImpl::TaskRunner::PostDelayedTask()`

```c++
bool TaskQueueImpl::TaskRunner::PostDelayedTask(const Location& location,
                                                OnceClosure callback,
                                                TimeDelta delay) {
  return task_poster_->PostTask(PostedTask(this, std::move(callback), location,
                                           delay, Nestable::kNestable,
                                           task_type_));
}
```

`task_poster_`是一个`TaskQueueImpl::GuardTaskPoster`。

```c++
bool TaskQueueImpl::GuardedTaskPoster::PostTask(PostedTask task) {
  // Do not process new PostTasks while we are handling a PostTask (tracing
  // has to do this) as it can lead to a deadlock and defer it instead.
  ScopedDeferTaskPosting disallow_task_posting;

  auto token = operations_controller_.TryBeginOperation();
  if (!token)
    return false;

  outer_->PostTask(std::move(task));
  return true;
}
```

主要是对PostTask的前后做一些状态检测，然后最后task给outer_，也就是外面的`TaskQueueImpl::PostTask()`处理

```c++
void TaskQueueImpl::PostTask(PostedTask task) {
  CurrentThread current_thread =
      associated_thread_->IsBoundToCurrentThread()
          ? TaskQueueImpl::CurrentThread::kMainThread
          : TaskQueueImpl::CurrentThread::kNotMainThread;

// some checks...

  if (!task.is_delayed()) {
    PostImmediateTaskImpl(std::move(task), current_thread);
  } else {
    PostDelayedTaskImpl(std::move(task), current_thread);
  }
}
```

获取当前*相关线程*，判断任务是否延时，然后推送到对应队列。

首先来看添加延迟任务，这个比较简短。

```c++
void TaskQueueImpl::PostDelayedTaskImpl(PostedTask posted_task,
                                        CurrentThread current_thread) {
  // Use CHECK instead of DCHECK to crash earlier. See http://crbug.com/711167
  // for details.
  if (current_thread == CurrentThread::kMainThread) {
    // ...
    PushOntoDelayedIncomingQueueFromMainThread(
        std::move(pending_task), &lazy_now,
        /* notify_task_annotator */ true);
  } else {
    // ...
    PushOntoDelayedIncomingQueue(
        MakeDelayedTask(std::move(posted_task), &lazy_now));
  }
}
```

如果当前线程是主线程，直接塞到`delayed_incoming_queue`里：

```c++
void TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread(
    Task pending_task,
    LazyNow* lazy_now,
    bool notify_task_annotator) {

  // 一些 tracing
  if (notify_task_annotator) {
    sequence_manager_->WillQueueTask(&pending_task);
    MaybeReportIpcTaskQueuedFromMainThread(pending_task);
  }
  // 塞入队列
  main_thread_only().delayed_incoming_queue.push(std::move(pending_task));
  // 设置下次唤醒时间
  UpdateWakeUp(lazy_now);

  TraceQueueSize();
}
```

如果当前是其他线程：

```c++
void TaskQueueImpl::PushOntoDelayedIncomingQueue(Task pending_task) {
  sequence_manager_->WillQueueTask(&pending_task);
  MaybeReportIpcTaskQueuedFromAnyThreadUnlocked(pending_task);

  // TODO(altimin): Add a copy method to Task to capture metadata here.
  auto task_runner = pending_task.task_runner;
  const auto task_type = pending_task.task_type;
  PostImmediateTaskImpl(
      PostedTask(std::move(task_runner),
                 BindOnce(&TaskQueueImpl::ScheduleDelayedWorkTask,
                          Unretained(this), std::move(pending_task)),
                 FROM_HERE, TimeDelta(), Nestable::kNonNestable, task_type),
      CurrentThread::kNotMainThread);
}
```

调用`PostImmediateTaskImpl`添加一个立即执行任务。

首先看该任务回调`TaskQueueImpl::ScheduleDelayedWorkTask()`。

在`TaskQueueImpl::ScheduleDelayedWorkTask()`中，可以看到：

> If |delayed_run_time| is in the past then push it onto the work queue immediately. To ensure the right task ordering we need to temporarily push it onto the |delayed_incoming_queue|.

和

> If |delayed_run_time| is in the future we can queue it as normal.

意味着，最终也是在目标线程推入了`delayed_incoming_queue`。

再来看`PostImmediateTaskImpl`：

```c++
void TaskQueueImpl::PostImmediateTaskImpl(PostedTask task,
                                          CurrentThread current_thread) {
  // checks...

  bool should_schedule_work = false;
  {
    // TODO(alexclarke): Maybe add a main thread only immediate_incoming_queue
    // See https://crbug.com/901800

    // 使用anythread需要加锁
    base::internal::CheckedAutoLock lock(any_thread_lock_);
    bool add_queue_time_to_tasks = sequence_manager_->GetAddQueueTimeToTasks();
    TimeTicks queue_time;
    if (add_queue_time_to_tasks || delayed_fence_allowed_)
      queue_time = sequence_manager_->any_thread_clock()->NowTicks();

    // The sequence number must be incremented atomically with pushing onto the
    // incoming queue. Otherwise if there are several threads posting task we
    // risk breaking the assumption that sequence numbers increase monotonically
    // within a queue.
    EnqueueOrder sequence_number = sequence_manager_->GetNextSequenceNumber();
    bool was_immediate_incoming_queue_empty =
        any_thread_.immediate_incoming_queue.empty();

    // 序号在推入前就要指定，不然并发时候会乱（后来的有可能在前面）
    // 1
    any_thread_.immediate_incoming_queue.push_back(
        Task(std::move(task), sequence_number, sequence_number, queue_time));

    // 一些tracing，back就是我们刚放进去的task
    sequence_manager_->WillQueueTask(
        &any_thread_.immediate_incoming_queue.back());
    MaybeReportIpcTaskQueuedFromAnyThreadLocked(
        any_thread_.immediate_incoming_queue.back());

    for (auto& handler : any_thread_.on_task_posted_handlers) {
      DCHECK(!handler.second.is_null());
      handler.second.Run(any_thread_.immediate_incoming_queue.back());
    }

    // If this queue was completely empty, then the SequenceManager needs to be
    // informed so it can reload the work queue and add us to the
    // TaskQueueSelector which can only be done from the main thread. In
    // addition it may need to schedule a DoWork if this queue isn't blocked.
    if (was_immediate_incoming_queue_empty &&
        any_thread_.immediate_work_queue_empty) {
      empty_queues_to_reload_handle_.SetActive(true);
      should_schedule_work =
          any_thread_.post_immediate_task_should_schedule_work;
    }
  }

  // On windows it's important to call this outside of a lock because calling a
  // pump while holding a lock can result in priority inversions. See
  // http://shortn/_ntnKNqjDQT for a discussion.
  //
  // Calling ScheduleWork outside the lock should be safe, only the main thread
  // can mutate |any_thread_.post_immediate_task_should_schedule_work|. If it
  // transitions to false we call ScheduleWork redundantly that's harmless. If
  // it transitions to true, the side effect of
  // |empty_queues_to_reload_handle_SetActive(true)| is guaranteed to be picked
  // up by the ThreadController's call to SequenceManagerImpl::DelayTillNextTask
  // when it computes what continuation (if any) is needed.
  if (should_schedule_work)
    sequence_manager_->ScheduleWork();

  TraceQueueSize();
}
```

【1】处添加到immediate_incoming_queue

### MessagePumpForIO

`MessagePumpForIO` 是windows下IO线程的 message pump 的具体实例。
