# Chromium's MessagePump

在[讨论](https://groups.google.com/a/chromium.org/g/chromium-dev/c/Alycg9BUDEM)上看到这样一个解释:

> MessageLoop is a task runner. It keeps a queue of Tasks, but it may also have other messages (platform-specific UI events, or platform-specific IO events) that it handles. Those platform specific messages are handled by the MessagePumps. Therefore, MessageLoop depends on MessagePumps.

MessageLoop是一个`task runner`，保存了一个各种任务的队列，（基于平台的 UI events 或者基于平台的 IO events）。这些基于平台的消息是由MessagePumps管理的。因此`MessageLoop`依赖于`MessagePumps`。

由于这个回答的时间为2011年，考虑到时效性，不具有多大参考价值。

刚接触到这一块，我也不太明白其中的复杂的逻辑。所以这篇可能仅仅是一个繁琐注释的翻译解析（并且不那么准确）

## Background

每个`base::Thread`（除了主线程），在创建的时候都会拥有一个`MessageLoop`。每个进程至少有两个线程，一个`UI thread`，一个`IO thread`。

TODO: add explainations

## [message_pump.h]

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
    User provided implementation of MessagePump interface
- *IO*
    This type of pump also supports asynchronous IO.

并且提供了`Create`静态[Factory Implementation](https://source.chromium.org/chromium/chromium/src/+/main:base/message_loop/message_pump.h;l=37):

```c++
// Creates the default MessagePump based on |type|. Caller owns return value.
static std::unique_ptr<MessagePump> Create(MessagePumpType type);
```

create `MessagePump for UI`
