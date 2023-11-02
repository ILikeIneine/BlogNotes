# Chromium 中的 CHECK 宏

## 主旨

理解这段代码

```c++
// Macro which uses but does not evaluate expr and any stream parameters.
#define EAT_CHECK_STREAM_PARAMS(expr) \
  true ? (void)0                      \
       : ::logging::VoidifyStream(expr) & (*::logging::g_swallow_stream)
BASE_EXPORT extern std::ostream* g_swallow_stream;
```

## 什么是 CHECK

CHECK 是一个条件检查宏，类似于assert，但是提供了可选择的流输出（尽管这个是个dummy output）。

使用要检查的条件表达式，例如 `CHECK(object)` 或 `CHECK(!list.empty())`。有多种衍生方法，例如 `CHECK_EQ`、`CHECK_NE`、`DCHECK` 和 `DCHECK_CALLED_ON_VALID_SEQUENCE`。

如果条件不符合预期，它将立即崩溃（Crash），输出一条消息和堆栈跟踪，表明当时的条件检查失败，并倾向于退出（取决于标志）。消息可以这样确定：

```c++
CHECK(object) << "object must be set";
```

## CHECK 行为的差异

### When flagged OFFICIAL_BUILD && !DCHECK_IS_ON

当 `CHECK` 断言无论成功和失败，尾随的流输入是会被舍弃的，将不会打印，[Source Here](https://source.chromium.org/chromium/chromium/src/+/main:base/check.h;l=205;bpv=0;bpt=1)。

此时，`CHECK` 的宏实现为：

```c++
#define CHECK(condition) \
  UNLIKELY(!(condition)) ? logging::CheckFailure() : EAT_CHECK_STREAM_PARAMS()
```

`UNLIKELY` 是分支预测，在gcc上会使用`__builtin_expect`开洞优化分支选择，msvc就直接无视。

```c++
// Macro for hinting that an expression is likely to be false.
#if !defined(UNLIKELY)
#if defined(COMPILER_GCC) || defined(__clang__)
#define UNLIKELY(x) __builtin_expect(!!(x), 0)
#else
#define UNLIKELY(x) (x) // msvc
#endif  // defined(COMPILER_GCC)
#endif  // !defined(UNLIKELY)
```

可以看到，当condition 为 false，会直接 `CheckFailure()`，这是一个 *noreachable* 的 immediate crash。

```c++
[[noreturn]] IMMEDIATE_CRASH_ALWAYS_INLINE void CheckFailure() {
  base::ImmediateCrash();
}
```

当条件为真，意味着CHECK是符合操作的，我们需要抹掉 CHECK 后面可能携带的流操作。这时候会三目运算符会走到 `EAT_CHECK_STREAM_PARAMS`。

请注意，为了效率，我们同时不希望这个流在编译时被评估！就像他不存在一样。

仔细理解这段代码：

```c++
// Macro which uses but does not evaluate expr and any stream parameters.
#define EAT_CHECK_STREAM_PARAMS(expr) \
  true ? (void)0                      \
       : ::logging::VoidifyStream(expr) & (*::logging::g_swallow_stream)
BASE_EXPORT extern std::ostream* g_swallow_stream;
```

乍一看，既然恒为true，似乎和 `(void)0` 恒等，秉着 code clean 的行为，不如直接替换。这里是一个[最小实现](https://godbolt.org/z/GxobfbMdo)。

可以观察到，简单的 `(void)0` 在正常情况下是可以工作，但是，当你尝试在 CHECK 后面塞一个 `<< "what a stream?"`，编译失败了。并没有达到 `EAT_CHECK_STREAM_PARAMS` 的预期目的。

将宏展开观察：

```c++
UNLIKELY(!(condition)) ? logging::CheckFailure() : (void)0 << "what a stream?"  // Oops...

// invalid operand<< between 'void' and 'const char[14]' !
```

***noticing***: 条件运算符的优先级 `<<`(shift left) > `&`(bit operand) > `?:`。

导致, `operator<<` 在这里被当作位移操作，且优先级大于 `?:`，所以会优先评估 `(void)0 << "what a stream?"`。

所以，我们需要 `EAT_CHECK_STREAM_PARAMS(expr)` 同时：

- 恒评估为`(void)0`，（due to CheckFailure() is void， the third-operand must , to make program legal, be void or throw. check [this](https://en.cppreference.com/w/cpp/language/operator_other)）
- 且又能吸收掉后面的 `operator<<`

Now, consider this:

```c++
#define CHECK(condition) \
  UNLIKELY(!(condition)) ? logging::CheckFailure() : true ? (void)0 \
       : something & (*::logging::g_swallow_stream) << "what a stream"
```

在这里，`true ? (void)0 : expr` 保证了这段表达式恒为(void)0。

我们只需要一个 *dummy stream* `(*::logging::g_swallow_stream)` 用于吸收消息。此 `std::ostream` 的输出将按原样丢弃。

需要构造一个同样是 `void` 的 `expr`, 且执行优先级*晚于* `operator<<`， 这样`operator<<`就会先吞掉后面的流，然后再通过 *something* 的 `operator &` 操作, 把流产生的 `std::ostream` 消化掉。

考虑，当这个 *something* 是 `::logging::VoidifyStream(expr)` 的时候：

```c++
class VoidifyStream {
 public:
  VoidifyStream() = default;
  explicit VoidifyStream(bool) {}

  // This operator has lower precedence than << but higher than ?:
  void operator&(std::ostream&) {}
};
```

这个 `dummy stream` 先吃完先吃完后面的流，然后产生的 `std::ostream` 再被 `VoidifyStream` 的 `operator &`操作吃了。

> which uses but does not evaluate expr and any stream parameters.

### While not flagged OFFICIAL_BUILD && !DCHECK_IS_ON

会开启 `CHECK` 断言失败打印的流，[Source Here](https://source.chromium.org/chromium/chromium/src/+/main:base/check.h;l=213;bpv=0;bpt=1)。

此时，`CHECK` 的宏实现为:

```c++
#define CHECK(condition)                                                \
  LOGGING_CHECK_FUNCTION_IMPL(::logging::CheckError::Check(#condition), \
                              condition)
```

`::logging::CheckError::Check(#condition)`的实现如下：

```c++
// Declaration
static CheckError Check(
    const char* condition,
    const base::Location& location = base::Location::Current());

// Defination
CheckError CheckError::Check(const char* condition,
                             const base::Location& location) {
  auto* const log_message = new LogMessage(
      location.file_name(), location.line_number(), LOGGING_FATAL);
  log_message->stream() << "Check failed: " << condition << ". ";
  return CheckError(log_message);
}
```

`CheckError`是一个 *Factory Class*，构造出`#condition`的信息附加上代码的 *location info*, 返回携带这些info的一个 `CheckError`。

这是`LOGGING_CHECK_FUNCTION_IMPL`宏，

```c++
#define LOGGING_CHECK_FUNCTION_IMPL(check_stream, condition)              \
  switch (0)                                                              \
  case 0:                                                                 \
  default:                                                                \
    /* Hint to the optimizer that `condition` is unlikely to be false. */ \
    /* The optimizer can use this as a hint to place the failure path */  \
    /* out-of-line, e.g. at the tail of the function. */                  \
    if (const bool probably_true = static_cast<bool>(condition);          \
        LIKELY(ANALYZER_ASSUME_TRUE(probably_true)))                      \
      ;                                                                   \
    else                                                                  \
      (check_stream)
```

我们假设 `CHECK(1>2) << "one is greater than two"`，展开结果会是如下，

```c++
  switch (0)                                                              \
  case 0:                                                                 \
  default:                                                                \
    /* Hint to the optimizer that `condition` is unlikely to be false. */ \
    /* The optimizer can use this as a hint to place the failure path */  \
    /* out-of-line, e.g. at the tail of the function. */                  \
    if (const bool probably_true = static_cast<bool>(1>2);          \
        LIKELY(ANALYZER_ASSUME_TRUE(probably_true)))                      \
      ;                                                                   \
    else                                                                  \
      ::logging::CheckError << "one is greater than two"
```

可以看到，当condition成立的时候，分支会走到if里，因为绝大部分断言都倾向于成功的验证，这里依然进行了分支优化。

在condition不成立的时候，就走到了`CheckError`。

并且由于`CheckError` 重载了 `operator<<`[(Source Here)](https://source.chromium.org/chromium/chromium/src/+/main:base/check.h;l=120;bpv=0;bpt=1)，会把后面的Log写入到stream里。

在 `CheckError` 析构的时候：

```c++
CheckError::~CheckError() {
  // TODO(crbug.com/1409729): Consider splitting out CHECK from DCHECK so that
  // the destructor can be marked [[noreturn]] and we don't need to check
  // severity in the destructor.
  const bool is_fatal = log_message_->severity() == LOGGING_FATAL;
  // Note: This function ends up in crash stack traces. If its full name
  // changes, the crash server's magic signature logic needs to be updated.
  // See cl/306632920.
  delete log_message_;

  // Make sure we crash even if LOG(FATAL) has been overridden.
  // TODO(crbug.com/1409729): Remove severity checking in the destructor when
  // LOG(FATAL) is [[noreturn]] and can't be overridden.
  if (is_fatal) {
    base::ImmediateCrash();
  }
}
```

首先会判断内部`log_message_`的等级，如果是`fatal`就直接`immediate crash`。

在crash之前会先把`log_message_`释放。这个`::logging::LogMessage`在析构的时候会把之前的stream的log全部释放打印，the procedure is quite quite complex，[Source Here](https://source.chromium.org/chromium/chromium/src/+/main:base/logging.cc;l=720;drc=c177416839ec7a3fe2fe6e91a7e8e519248f96bd;bpv=0;bpt=1)

这样子，就会在触发_debug_point之前把堆栈信息和追加的流日志打印出来。

在老一点的版本中，还是使用了和三目运算符类似的技法（例如在106版本内核）而不是`switch`，分析原理和方式无异。
