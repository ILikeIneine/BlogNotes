# Chromium's Task Posting


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
    // 1
    std::unique_ptr<BrowserUIThreadScheduler> browser_ui_thread_scheduler,
    std::unique_ptr<BrowserIOThreadDelegate> browser_io_thread_delegate)
    // 2
    : ui_thread_executor_(std::make_unique<UIThreadExecutor>(
          std::move(browser_ui_thread_scheduler))),
      io_thread_executor_(std::make_unique<IOThreadExecutor>(
          std::move(browser_io_thread_delegate))) {
  // 3
  browser_ui_thread_handle_ = ui_thread_executor_->GetUIThreadHandle();
  browser_io_thread_handle_ = io_thread_executor_->GetIOThreadHandle();
  ui_thread_executor_->SetIOThreadHandle(browser_io_thread_handle_);
  io_thread_executor_->SetUIThreadHandle(browser_ui_thread_handle_);
}
```

具体分析一下【1】，`BrowserUIThreadScheduler`:

```c++
BrowserUIThreadScheduler::BrowserUIThreadScheduler()
    : owned_sequence_manager_(
          // a
          base::sequence_manager::CreateUnboundSequenceManager(
              base::sequence_manager::SequenceManager::Settings::Builder()
                  .SetMessagePumpType(base::MessagePumpType::UI)
                  .SetCanRunTasksByBatches(true)
                  .SetPrioritySettings(
                      internal::CreateBrowserTaskPrioritySettings())
                  .Build())),
      // b
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

【a】处`CreateUnboundSequenceManager()`最后调用的是

```c++
// static
std::unique_ptr<SequenceManagerImpl> SequenceManagerImpl::CreateUnbound(
    SequenceManager::Settings settings) {
  // 初始化thread_controller
  auto thread_controller =
      ThreadControllerWithMessagePumpImpl::CreateUnbound(settings);
  return WrapUnique(new SequenceManagerImpl(std::move(thread_controller),
                                            std::move(settings)));
}
```

这里我们看到之前分析获取任务和执行任务所熟悉的`ThreadControllerWithMessagePumpImpl`，在`SequenceManagerImpl`的构造函数中会将`controller_`的`SequencedTaskSource`设为`this`，经典的依赖注入，同时这也符合我们上面获取task的分析。

【b】处的`task_queues_`的创建

`task_queues_`的类型是`BrowserTaskQueues`，构造代码比较长，具体看[这里](https://source.chromium.org/chromium/chromium/src/+/main:content/browser/scheduler/browser_task_queues.cc;l=162;drc=654daa129ebb42d20055ecacdbfc76b8eec050d8;bpv=1;bpt=1)。

```c++
BrowserTaskQueues::BrowserTaskQueues(
    BrowserThread::ID thread_id,
    base::sequence_manager::SequenceManager* sequence_manager) {
  for (size_t i = 0; i < queue_data_.size(); ++i) {
    // 创建TaskQueue
    queue_data_[i].task_queue = sequence_manager->CreateTaskQueue(
        base::sequence_manager::TaskQueue::Spec(
            GetTaskQueueName(thread_id, static_cast<QueueType>(i))));
    queue_data_[i].voter = queue_data_[i].task_queue->CreateQueueEnabledVoter();
    if (static_cast<QueueType>(i) != QueueType::kDefault) {
      queue_data_[i].voter->SetVoteToEnable(false);
    }
  }

  // 设置优先级
  GetBrowserTaskQueue(QueueType::kUserVisible)
      ->SetQueuePriority(BrowserTaskPriority::kLowPriority);

  // ...

  // Control queue
  control_queue_ =
      sequence_manager->CreateTaskQueue(base::sequence_manager::TaskQueue::Spec(
          GetControlTaskQueueName(thread_id)));
  control_queue_->SetQueuePriority(BrowserTaskPriority::kControlPriority);

  // Run all pending queue
  run_all_pending_tasks_queue_ =
      sequence_manager->CreateTaskQueue(base::sequence_manager::TaskQueue::Spec(
          GetRunAllPendingTaskQueueName(thread_id)));
  run_all_pending_tasks_queue_->SetQueuePriority(
      BrowserTaskPriority::kBestEffortPriority);

  handle_ = base::AdoptRef(new Handle(this));
}
```

`BrowserTaskQueues`构造时候会调用上面在【a】的`SequenceManager`里创建多个`Queue`，并为每个`Queue`设置不同优先级，最后创建了一个自己的`Handle`对象。

首先看*TaskQueue*的创建，`SequenceManager->CreateTaskQueue()`，最终会调用到[这里](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/sequence_manager_impl.cc;l=402;drc=7172fffc3c545134d5c88af8ab07b04fcb1d628e):

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

然后`TaskQueueImpl`:

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
      // 创建 Default TaskRunner
      default_task_runner_(CreateTaskRunner(kTaskTypeNone)) {
  UpdateCrossThreadQueueStateLocked();
  // SequenceManager can't be set later, so we need to prevent task runners
  // from posting any tasks.
  if (sequence_manager_)
    task_poster_->StartAcceptingOperations();
}
```

在`TaskQueueImpl()`里创建了对应的`TaskRunner`，`TaskQueueImpl()`第二个参数是`WakeUpQueue`，负责管理（唤醒相关）该`TaskQueueImpl`。

初始化完毕后，`return task_queue` 返回到上一层：

```c++
TaskQueue::Handle SequenceManagerImpl::CreateTaskQueue(
    const TaskQueue::Spec& spec) {
  return TaskQueue::Handle(CreateTaskQueueImpl(spec));
}
```

这里，代码发生过一些变动:
> [scheduler] Introduce TaskQueue::Handle which owns TaskQueue

最新的chromium引进了[`SequenceManager::Handle`](https://source.chromium.org/chromium/chromium/src/+/main:base/task/sequence_manager/task_queue.h;l=105;drc=903cd05b772e5868c14f57ec3766b6192fc0104a;bpv=1;bpt=1)来管理`TaskQueue`。

`SequenceManager::Handle`内部持有`TaskQueueImpl`和一个`sequence_manager`的引用。

在没有这个`Handle`之前，`TaskRunner`是在这里创建的（通过SequenceManagerImpl->CreateTaskRunner()等等...）。

（这一系列都是依赖注入，一层一层的）

任务队列创建完毕后，`BrowserTaskQueues()`最后把他放到handle里面：

```c++
handle_ = base::AdoptRef(new Handle(this));
```

这个 handle 类型是`BrowserTaskQueues::Handle`:

```c++
BrowserTaskQueues::Handle::Handle(BrowserTaskQueues* outer)
    : outer_(outer),
      control_task_runner_(outer_->control_queue_->task_runner()),
      default_task_runner_(outer_->GetDefaultTaskQueue()->task_runner()),
      browser_task_runners_(outer_->CreateBrowserTaskRunners()) {}
```

初始化了三个`TaskRunner`，分别从三个不同的`TaskQueue`获取。`CreateBrowserTaskRunner`会初始化并获取上面通过`SeqenceManager::CreateTaskQueue()`创建的`TaskQueue`的里的`TaskRunner`:

```c++
std::array<scoped_refptr<base::SingleThreadTaskRunner>,
           BrowserTaskQueues::kNumQueueTypes>
BrowserTaskQueues::CreateBrowserTaskRunners() const {
  std::array<scoped_refptr<base::SingleThreadTaskRunner>, kNumQueueTypes>
      task_runners;
  for (size_t i = 0; i < queue_data_.size(); ++i) {
    // 这里task
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

所以看来，这个handle顺带帮`BrowserTaskQueues`绑定了一下对应的`task_runner`，以便外部通过调用`BrowserTaskQueues::GetBrowserTaskRunner(QueueType)`来根据不同的`QueueType`选择对应的`TaskRunner`：

```c++
const scoped_refptr<base::SingleThreadTaskRunner>& GetBrowserTaskRunner(
        QueueType queue_type) const {
      return browser_task_runners_[static_cast<size_t>(queue_type)];
    }
```

以上是*任务投递*的前情提要，正片开始，来看投递的具体逻辑。

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

主要是对PostTask的前后做一些状态检测，最后把task还给给outer_，也就是外面的`TaskQueueImpl::PostTask()`处理

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

【1】处最后也是把任务推到到immediate_incoming_queue。