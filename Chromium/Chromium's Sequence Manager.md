# Sequence Manager

`SequenceManager`提供了一系列`FIFO`优先队列，它允许了*立即*任务(immediate tasks)序列和*延时*任务(delayed tasks)序列等多个序列汇集在单个底层序列。

## Work Queue 和 Task Selection

即时任务(immediate tasks)和延时任务(delayed tasks)都会被投递到一个任务队列(`TaskQueue`)，通过一个与之相关联的`TaskQueueImpl::TaskRunner`。

`TaskQueue`使用独立的原始FIFO队列`WorkQueue`管理这些即时任务(immediate tasks)和延时任务(delayed tasks)。任务最终会到达他们所指定的`WorkQueue`，这些`WorkQueue`通过`TaskQueueSelector`对于`SequenceManager`来说直接可见。

具体来说，`SequenceManagerImpl::SelectNextTask()`使用`TaskQueueSelector::SelectWorkQueueToService()`去选择下一个工作队列`WorkQueue`，这其中有各种策略，例如根据优先级，从`WorkQueue`一次弹出一个任务。

理解上来说，**TaskQueue** 包含多个 **WorkQueue**（准确的说是4个，3个在MainThreadOnly，1个在AnyThread），**WorkQueue** 实际上内部维护的是一个队列 **TaskQueueImpl::TaskDeque**

## Journey of a Task

任务队列(Task queues)有一个特别的机制，可以允许跨线程且高效的投送(posting)。

原理是使用了两个工作队列(work queue)，任务(tasks)在投送时使用的是`immediate_incoming_queue`，而在弹出任务时使用的是`immediate_work_queue`。

一个即时任务(immediate tasks)从主线程被发布，会在`TaskQueueImpl::PostImmediateTaskImpl()`里面被推到`immediate_incoming_queue`里。

如果这个工作队列(work queue)是空的就会通知`SequenceManager`，并且注册该`TaskQueue`。然后`SequenceManger`选择一个任务之前，会执行`ReloadEmptyImmediateWorkQueue()`。这个操作是为所有注册过的`TaskQueue`*批量地*(in batch)将任务从`immediate_incoming_queue`移动到`immediate_work_queue`。

一个延时任务(delayed tasks)从主线程发布，会在`TaskQueueImpl::PostImmediateTaskImpl()`里面被推到`delayed_incoming_queue`里面，然后更新下一次WakeUp。

一个延时任务(delayed tasks)从其他线程发布的话，会有一些额外逻辑，但最终也会被推到`delayed_incoming_queue`里。

这些任务之后遵从上面常规的工作队列选择机制(work queue selection mechanism)。

## Journey of a WakeUp

一个`WakeUp`代表了一个延迟任务(delayed task)想要运行的时间。

每个`TaskQueueImpl`都有属于的各自的下一次唤醒时间，保存在`main_thread_only().scheduled_wake_up`，并且与最早的待处理延时任务(the earliest pending delayed task)相关联。

它通过`WakeUpQueue::SetNextWakeUpForQueue()`将它的下次唤醒信息传递给`WakeUpQueue`。这个`WakeUpQueue`负责确定线程下一次的唤醒时间，并且是通过`SequenceManagerImpl`访问。假如当前没有*立即*工作(immediate work)，则可以决定下一个运行时间，并最终传递给`MessagePump`。

通常是通过`MessagePump::Delegate::NextWorkInfo`（由`ThreadControllerWithMessagePumpImpl::DoWork()`返回）或通过`MessagePump::ScheduleDelayedWork()`（在极少数情况下，下一次唤醒是在`DoWork()`以外的主线程上安排的）。

当到达与唤醒相关联的延迟运行时间时，`WakeUpQueue`会通过`WakeUpQueue::MoveReadyDelayedTasksToWorkQueues()`方式被通知，然后反过来通知所有能够解决这次唤醒(wake-up)问题的任务队列(`TaskQueue`)。

这样，每个任务队列都能处理到期的延时任务。

## Journey of a delayed Task

*跨线程*发布的延时任务会生成一个即时任务，以运行`TaskQueueImpl::ScheduleDelayedWorkTask()`并且最终调用`TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread()`，如此便可以在主线程上把即时任务加入队列。

从*主线程*发布的延迟任务会跳过这一步，直接调用`TaskQueueImpl::PushOntoDelayedIncomingQueueFromMainThread()`。然后，任务会被推送到`main_thread_only().delayed_incoming_queue`，并可能更新下一次任务队列的唤醒(wake-up)。

一旦到达延迟运行时间，可能因为唤醒已经被解决，延迟的任务会被移动到`main_thread_only().delayed_work_queue`，并遵循常规的工作队列选择机制。
