# 编写 Qt 跨线程异步调用器



## 一、设计背景

众所周知，Qt 的信号槽系统提供了线程安全的跨线程异步执行代码的机制([Qt::QueuedConnection](https://doc.qt.io/qt-5/qt.html#ConnectionType-enum))。

使用该机制，可以让槽函数代码在另一个线程执行，并且可以携带参数，用户代码无需加锁，只要发射信号即可。

但很多时候，我们仅仅只想单次异步执行一段代码。若是通过信号槽机制执行，则就不得不声明一个信号函数，连接信号槽，再发射信号，这样显然很繁琐。

幸好，Qt 本身也知道这种需求的存在，提供了 [QTimer::singleShot()](https://doc.qt.io/qt-5/qtimer.html#singleShot) 函数，可以跨线程异步执行槽函数，甚至还可以延迟执行——然而该函数只能执行无参数槽函数，不能执行其它类型的回调（如 lambda）。

所以，最好能够有一个类似 [QTimer::singleShot()](https://doc.qt.io/qt-5/qtimer.html#singleShot)，但又可以接收任意参数个数的任意函数子的 API。

> 5.3的老代码写太久，思维定势了，刚查了下5.4的 singleShot 是支持 Functor 的……那这篇文章留作该机制的技术探讨吧……

考虑到异步执行时对执行结果的访问，可以参考 [std::async()](https://en.cppreference.com/w/cpp/thread/async)，返回一个 `future` 对象。但不能直接使用 [std::future](https://en.cppreference.com/w/cpp/thread/future)——因为它的 `get` 和 `wait` 会阻塞住线程，对于 Qt 而言就会阻塞事件循环。

即，我们还需要一个不会阻塞事件循环的等待机制。

综上所述，需求总结如下：

1. 提供跨线程异步执行代码的能力，让回调函数在目标线程执行；
2. 提供对任意函数子的异步执行接口，可以接受具备任意参数个数的任意函数子；
3. 提供延迟执行功能，以满足 [QTimer::singleShot()](https://doc.qt.io/qt-5/qtimer.html#singleShot) 的所有功能，便于替代前者；
4. 提供 `future` 返回对象，用于处理返回值和等待同步，接口与  [std::future](https://en.cppreference.com/w/cpp/thread/future) 类似；
5. 提供不阻塞 Qt 事件循环的等待机制，用于供 `future` 使用。



## 二、异步回调实现

跨线程异步回调的实现，可以参考 Qt 的元对象机制。

Qt 通过元对象系统进行异步执行时（信号槽、[QTimer::singleShot()](https://doc.qt.io/qt-5/qtimer.html#singleShot)、[QMetaMethod::invoke](https://www.qtdoc.cn/Src/M/QMetaMethod/QMetaMethod.html#bool-qmetamethodinvokea-hrefoqobjectqobjecthtmlqobjecta-object-a-hrefqqt_namespaceqt_namespacehtmlconnectiontype-enumqtconnectiontypea-connectiontype-a-hrefgqgenericreturnargumentqgenericreturnargumenthtmlqgenericreturnargumenta-returnvalue-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val0--qgenericargumentnullptr-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val1--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val2--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val3--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val4--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val5--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val6--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val7--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val8--qgenericargument-a-hrefgqgenericargumentqgenericargumenthtmlqgenericargumenta-val9--qgenericargument-const) 等），本质上是将回调函数封装为 `QMetaCallEvent` 对象，再通过 [QCoreApplication::postEvent()](https://doc.qt.io/qt-5/qcoreapplication.html#postEvent) 投送至目标对象。目标对象会在所属线程的事件循环中触发 [QObject::event()](https://doc.qt.io/qt-5/qobject.html#event) 事件处理函数，解析事件并执行回调函数。

然而 `QMetaCallEvent` 是非公开接口，Qt 不保证其接口的可用和稳定性，因此我们需要仿照此流程自行封装。

### 2.1 异步回调事件类

新建一个事件类，继承自 [QEvent](https://doc.qt.io/qt-5/qevent.html)，并注册获取事件类型编号：

```c++
class AsyncInvokeEvent : public QEvent {
 public:
  static const int kEventType;
  std::function<QVariant(void)> Function;
  std::promise<QVariant>;
  std::shared_future<QVariant>;
};
const int AsyncInvokeEvent::kEventType = QEvent::registerEventType();
AsyncInvokeEvent::AsyncInvokeEvent() : QEvent(QEvent::Type(kEventType)) {}
```

将用户通过 API 传入的回调函数封装为 [std::function\<QVariant(void)\>](https://en.cppreference.com/w/cpp/utility/functional/function) 对象，以擦除类型信息，便于封入事件类中。

考虑到需要获取返回值，此处使用 Qt 的万能动态类型 [QVariant](https://doc.qt.io/qt-5/qvariant.html) 存储返回类型，但代价是返回值必须注册至 Qt 元对象系统——也可将 `future` 实现为模板类型，但这会导致代码复杂度大幅增加，并且不得不将 `cpp` 中的大部分流程暴露至头文件。

### 2.2 异步事件过滤器

将异步回调事件发送至目标线程时，需要有一个重写了 [QObject::event()](https://doc.qt.io/qt-5/qobject.html#event) 函数的对象接受该事件。我们可以考虑为每个 Qt 线程建立一个事件过滤器，使用一个全局的字典保存，在使用时通过线程指针查询该字典，若未检索到则新建之，即惰性初始化：

```c++
AsyncInvokerEventFilter* filter;
{
  // Find event filter for given thread
  static std::atomic_flag flag = ATOMIC_FLAG_INIT;
  static QHash<QThread*, AsyncInvokerEventFilter*> filters;
  while (flag.test_and_set(std::memory_order_seq_cst)) { // Spin-lock
  }
  auto it = filters.find(thread);
  if (it == filters.end()) {
    it = filters.insert(thread, new AsyncInvokerEventFilter{thread});
  }
  filter = *it;
  flag.clear(std::memory_order_release);
}
```

拿到事件过滤器后，即可向其投送事件：

```c++
auto event = new AsyncInvokeEvent;
event->Function = function;
event->future = event->promise.get_future();
QCoreApplication::postEvent(filter, event);
return event->future;
```

该事件会通过 Qt 的事件循环机制，在目标线程中被传递至接收者的  [event()](https://doc.qt.io/qt-5/qobject.html#event)  函数：

```c++
bool AsyncInvokerEventFilter::event(QEvent* event) {
  bool ret = QObject::event(event);
  if (event->type() == AsyncInvokeEvent::kEventType) {
    AsyncInvokeEvent* e = static_cast<AsyncInvokeEvent*>(event);
    e->Invoke();
  }
  event->accept();
  return ret;
}
```

至此，跨线程异步执行代码的机制已经编写完毕，整体其实是非常简单的。而且也并非 Qt 专属，其实任意具备事件循环的框架，都可以使用相同逻辑实现。

### 2.3 生命周期控制

Qt 信号槽的接收者指针，除了指定槽函数执行的线程外，还负责了生命周期控制的作用——只要 `sender` 或者 `receiver` 对象被析构，则该信号槽便不会再执行。

由于上文的异步回调事件类是由事件过滤器执行，而非回调函数对应的逻辑意义上的接收者，因此存在回调函数与其依赖资源的生命周期不一致的风险——我们需要引入额外的信息来监测回调函数的生命周期。

虽然回调函数中，也可以通过各类智能指针来管理资源的生命周期，但这会强迫调用者编写更多的代码，而且无法让事件在执行回调前判断相关资源生命周期是否已结束。

因此，我们需要一个机制来判断依赖资源的生命周期。由于在接口层可以做各式封装，最终传递到执行点的判断方式，可通过  [std::function\<bool(void)\>](https://en.cppreference.com/w/cpp/utility/functional/function) 来表达：

```c++
void AsyncInvokeEvent::Invoke() {
  QVariant ret;
  if (!IsAlive || IsAlive()) {
    ret = Function();
  }
  promise.set_value(ret);
}
```

对外接口中，可以考虑提供如下几种使用方式：

- 最基础的方式，直接传递 [std::function\<bool(void)\>](https://en.cppreference.com/w/cpp/utility/functional/function) 回调函数，可在其中封装各类自定义判断；
- 仿信号槽方式，传递 `QObject*` 指针，接口层通过 [QPointer](https://doc.qt.io/qt-5/qpointer.html) 类监测其存活状态，并将其封装为回调函数；
- 无生命周期约束，则接口层封装默认实现的回调函数，自动返回 `true`。



## 三、异步回调接口封装

根据上文代码，此机制的接口需要提供 `(执行线程, 回调函数)` 二元组作为输入参数，以及一个可选参数 `[生命周期判断回调]`。

为方便使用，参考 Qt 的信号槽、 [QTimer::singleShot()](https://doc.qt.io/qt-5/qtimer.html#singleShot) 语法，也可直接提供一个 `QObject*` 对象指针作为逻辑意义上的接收者，则可通过 [QObject::thread()](https://doc.qt.io/qt-5/qobject.html#thread) 函数获取执行线程。

回调函数最终传递至内部实现的版本，便是上文所述的  [std::function\<QVariant(void)\>](https://en.cppreference.com/w/cpp/utility/functional/function) 对象。但为方便使用，我们可以提供 `Func function, Args&&... args` 形式的模板接口，用于承接任意类型的函数子和函数参数：

```c++
template <typename Func, typename... Args>
AsyncInvoker::Future AsyncInvoker::Invoke(QThread* thread, const Func& func,
                                          Args&&... args) {
  if (!thread) {
    thread = qApp->thread();
  }
  auto f = std::bind(func, std::forward<Args>(args)...);
  std::function<QVariant(void)> function = [f]{ return QVariant{f()}; };
  return Invoke(function, thread);
}
```

此处的封装返回值一句存在隐患，因为传入函数有可能无返回值，此时这行代码会无法编译。

针对此情况，我们可以去 Qt 源码中看看官方是如何处理的。顺着接收函数子作为槽函数的 [QObject::connect()](https://doc.qt.io/qt-5/qobject.html#connect-4) 源代码，可在 `qobjectdefs_impl.h` 中找到如下黑魔法：

```c++
/*
   trick to set the return value of a slot that works even if the signal or the slot returns void
   to be used like     function(), ApplyReturnValue<ReturnType>(&return_value)
   if function() returns a value, the operator,(T, ApplyReturnValue<ReturnType>) is called, but if it
   returns void, the builtin one is used without an error.
*/
template <typename T>
struct ApplyReturnValue {
    void *data;
    explicit ApplyReturnValue(void *data_) : data(data_) {}
};
template<typename T, typename U>
void operator,(T &&value, const ApplyReturnValue<U> &container) {
    if (container.data)
        *reinterpret_cast<U *>(container.data) = std::forward<T>(value);
    }
    template<typename T>
void operator,(T, const ApplyReturnValue<void> &) {}
```

该模板类重载了逗号运算符，然后再通过模板特化匹配到不同版本的实现，对于有返回值的版本，将返回值储存至构造时输入的对象指针中。

 仿写一下，就能得到我们想要的了：

```c++
namespace impl {
template <typename T>
struct ApplyReturnValue {
  mutable QVariant* data_;
  explicit ApplyReturnValue(QVariant* data) : data_(data) {}
};
template <typename T, typename U>
inline void operator,(T&& value, const ApplyReturnValue<U>& container) {
  container.data_->setValue(std::forward<T>(value));
}
template <typename T>
inline void operator,(T, const ApplyReturnValue<void>&) {}
}  // namespace impl

template <typename Func, typename... Args>
AsyncInvoker::Future AsyncInvoker::Invoke(QThread* thread, const Func& func,
                                          Args&&... args) {
  if (!thread) {
    thread = qApp->thread();
  }
  auto f = std::bind(func, std::forward<Args>(args)...);
  std::function<QVariant(void)> function = [f] {
    using return_t = decltype(func(std::forward<Args>(args)...));
    QVariant ret;
    f(), impl::ApplyReturnValue<return_t>(&ret);
    return ret;
  };
  return Invoke(function, thread);
}
```

**注意**：`lambda` 的返回类型无法通过 [std::result_of](https://en.cppreference.com/w/cpp/types/result_of) 获取，只能通过 [decltype](https://en.cppreference.com/w/cpp/language/decltype) 获取。



## 四、延迟执行

延迟执行原理上也很简单，将延迟事件一并封装入异步回调事件类中，投送至事件过滤器后，事件过滤器再启动一个定时器事件，在定时器事件中才实际执行回调。

考虑到性能问题，此处不应为了执行一个回调函数就创建一个 [QTimer](https://doc.qt.io/qt-5/qtimer.html) 定时器对象，并绑定信号槽。

好消息是，Qt 已经考虑到此类需求，提供了一个轻量级的定时器接口 [QObject::startTimer()](https://doc.qt.io/qt-5/qobject.html#startTimer)，无需额外新建任何对象以及信号槽。该接口会定时发起定时器事件，通过 [QObject::timerEvent](https://doc.qt.io/qt-5/qobject.html#timerEvent) 接收处理。

因此，将前文的 `AsyncInvokerEventFilter::event()` 代码进行改造如下：

```c++
// AsyncInvokeEvent 成员变量：
// QSharedPointer<AsyncInvokeData> d;

// AsyncInvokerEventFilter 成员变量：
// QHash<int, QSharedPointer<AsyncInvokeData>> events_;

bool AsyncInvokerEventFilter::event(QEvent* event) {
  bool ret = QObject::event(event);
  if (event->type() == AsyncInvokeEvent::kEventType) {
    AsyncInvokeEvent* e = static_cast<AsyncInvokeEvent*>(event);
    if (e->d->delay_ms > 0) {
      // Deferred event, invoke in timerEvent
      int id = startTimer(e->d->delay_ms);
      events_[id] = e->d;
    } else {
      e->d->Invoke();
    }
  }
  event->accept();
  return ret;
}

void AsyncInvokerEventFilter::timerEvent(QTimerEvent* event) {
  int id = event->timerId();
  killTimer(id);

  auto it = events_.find(id);
  if (it == events_.end()) {
    return;
  }
  it.value()->Invoke();
  events_.erase(it);
}
```

**注意**

对于自定义事件，无论 [QObject::event()](https://doc.qt.io/qt-5/qobject.html#event) 返回是 `true` 还是 `false`，或者通过 [QEvent::accept()](https://doc.qt.io/qt-5/qevent.html#accept) / [QEvent::ignore()](https://doc.qt.io/qt-5/qevent.html#ignore) 接受或者忽略事件，Qt 都会无视上述操作，在执行完 [QObject::event()](https://doc.qt.io/qt-5/qobject.html#event) 后，直接删除由 [QCoreApplication::postEvent()](https://doc.qt.io/qt-5/qcoreapplication.html#postEvent) 投送的异步事件对象。

因此，对于需要延迟执行的事件，直接将事件指针保存下来是无效的，该指针会成为悬空指针。

此处使用共享指针保存事件数据，而非直接与容器内的值进行 [std::swap()](https://en.cppreference.com/w/cpp/algorithm/swap)——因为这些数据在 `Future` 中也会被引用，需要进行共享。

此处不可使用 [QCoreApplication::processEvents()](https://doc.qt.io/qt-5/qcoreapplication.html#processEvents) 方式进行延时——因为若在延时过程中又接收到异步回调事件，则会递归进入此函数，以此类推，存在多次递归导致爆栈的风险。



## 五、Future 对象

其实，简单一点的话，在异步回调事件类中存储一个 [std::promise](https://en.cppreference.com/w/cpp/thread/promise) 对象，然后返回它的 [get_future()](https://en.cppreference.com/w/cpp/thread/promise/get_future) 即可。

但前文也提到了，[std::future](https://en.cppreference.com/w/cpp/thread/future) 等待操作会阻塞线程，导致 Qt 事件循环失去响应，因此我们需要编写一个不阻塞 Qt 事件循环的等待机制，并且基于它来封装我们的 `Future` 类。

### 5.1 不阻塞 Qt 事件循环的等待

这个等待机制，想必很多人都已经在自己的项目中广泛应用，即使用计时器配合 [QCoreApplication::processEvents()](https://doc.qt.io/qt-5/qcoreapplication.html#processEvents) 实现不阻塞事件循环的延时：

```c++
QElapsedTimer timer;
timer.start()
while (timer.elapsed() < timeout) {
  QCoreApplication::processEvents();
}
```

为方便定制化的使用，我们可以参考 [std::future](https://en.cppreference.com/w/cpp/thread/future) 的 [wait()](https://en.cppreference.com/w/cpp/thread/future/wait) / [wait_for()](https://en.cppreference.com/w/cpp/thread/future/wait_for) / [wait_until()](https://en.cppreference.com/w/cpp/thread/future/wait_until) 函数，做多个额外的封装，并提供 [QDateTime](https://doc.qt.io/qt-5/qdatetime.html) 和 [std::chrono](https://en.cppreference.com/w/cpp/header/chrono) 两套接口：

```c++
void Wait(const std::function<bool(void)>& isValid,
          QEventLoop::ProcessEventsFlags flags = QEventLoop::AllEvents);

bool WaitFor(int timeout_milliseconds,
             QEventLoop::ProcessEventsFlags flags = QEventLoop::AllEvents,
             const std::function<bool(void)>& isValid = {});

bool WaitUntil(const QDateTime& timeout_time,
               QEventLoop::ProcessEventsFlags flags = QEventLoop::AllEvents,
               const std::function<bool(void)>& isValid = {});

template <class Rep, class Period>
bool WaitFor(const std::chrono::duration<Rep, Period>& timeout_duration,
             QEventLoop::ProcessEventsFlags flags = QEventLoop::AllEvents,
             const std::function<bool(void)>& isValid = {});

template <class Clock, class Duration>
bool WaitUntil(const std::chrono::time_point<Clock, Duration>& timeout_time,
               QEventLoop::ProcessEventsFlags flags = QEventLoop::AllEvents,
               const std::function<bool(void)>& isValid = {});
```

具体实现不再赘述，本例思路如下：

- `WaitFor` 中，使用 `当前时间 + 延时` 方式转换为 `WaitUntil` 的调用。
- `WaitUntil` 中，将超时判断封装为回调函数，以转换为 `Wait` 的调用。

### 5.2 Future 对象的 wait  与 get

`Future` 对象的 `wait()/wait_for()/wait_until()` 可直接调用上述实现。

但 [wait_for()](https://en.cppreference.com/w/cpp/thread/future/wait_for) / [wait_until()](https://en.cppreference.com/w/cpp/thread/future/wait_until) 函数需要返回 [std::future_status](https://en.cppreference.com/w/cpp/thread/future_status) 状态值，因此我们还需要判断该异步事件当前的执行状态。

由于要避免阻塞事件循环，我们不能直接调用 [std::future](https://en.cppreference.com/w/cpp/thread/future) 的对应函数，因此需要自行封装执行状态。

可考虑在异步回调事件类对象中存储一个 [std::atomic_bool](https://en.cppreference.com/w/cpp/atomic/atomic) 标志位，用于标识异步执行状态，在回调执行后将其之为 `true`：

```c++
// Future 成员变量：
// QSharedPointer<AsyncInvokeData> d_;

std::future_status AsyncInvoker::Future::status() const {
  if (!d_->future.valid()) {
    return std::future_status::deferred;
  } else if (!d_->executed.load()) {
    return std::future_status::timeout;
  } else {
    return std::future_status::ready;
  }
}
```

则 `wait_for()` 和 `wait_until()` 函数在完成等待后，返回 `status()` 即可；`wait()` 则是将 `status()` 作为判断条件传给上一节的 `Wait()` 函数。

`get()` 函数同理，可以直接使用 `wait()` 完成等待，然后返回 [std::future::get()](https://en.cppreference.com/w/cpp/thread/future/get) 即可。

`valid()` 函数则是同时判断 [std::future::valid()](https://en.cppreference.com/w/cpp/thread/future/valid) 和 `executed` 状态，即 `status() == std::future_status::ready`。



## 六、范例代码

上文中的代码，已提交至 [GitHub: ZgblKylin/KtUtils](https://github.com/ZgblKylin/KtUtils) 仓库的 [AsyncInvoker 分支](https://github.com/ZgblKylin/KtUtils/tree/AsyncInvoker)。

该仓库提供 CMake 和 QMake 两种使用方式，支持静态链接和动态链接（QMake 还提供源码包含）。


库文件会生成至 `${CMAKE_SOURCE_DIR}/lib` 目录，`dll`文件(特例)和单元测试的`exe`文件会生成至 `${CMAKE_SOURCE_DIR}/bin` 目录，库文件名称为 `KtUtils`/`KtUtilsd`(Debug 后缀)。


**CMake 使用方式**

```cmake
# 启用动态链接。默认使用静态链接。
set(KT_UTILS_SHARED_LIBRARY ON)

# 编译单元测试
set(BUILD_TESTING ON)

# 链接目标
add_subdirectory(KtUtils)
target_link_libraries(TargetName KtUtils)
```



**QMake 使用方式**

```
# 源码包含
include(KtUtils/KtUtils.pri)


# 链接库
# 修改 KtUtilsconf.pri 以启用动态链接、启用单元测试
SUBDIRS += KtUtils
win32: {
  contains(KtUtils_CONFIG, KtUtils_Shared_Library) {
    LIBS += -LKtUtils/bin/
  } else {
    LIBS += -LKtUtils/lib/
  }
} else:unix: {
  LIBS += -LKtUtils/lib/
}
CONFIG(release, debug|release): LIBS += -lKtUtils
else:CONFIG(debug, debug|release): LIBS += -lKtUtilsd
DESTDIR = KtUtils/bin
INCLUDEPATH += KtUtils/include
```
