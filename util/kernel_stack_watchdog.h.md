[TOC]

文件: `src/kudu/util/kernel_stack_watchdog.h`

# `class KernelStackWatchdog`

## 概述

是个**单例类**，用来管理一个线程集合，对其中的线程会进行“监视”。

“监视”的内容： 及时发现线程是否hang住了。

在这些操作之前，进程通过主动注册， 来表示希望自己被“监视”。在注册时会提供一个“时间长度”，作为阈值，如果发现该线程在经过这个阈值以后，仍然没有结束，那么就会打印“warning”日志，并且会把该线程的堆栈打印出来。

**针对的一些操作：**  
1. 对于一个可能会hang住的操作：比如IO操作。
2. 预期应该很快的操作：比如，在关键线程 中执行回调时不应该阻塞

**最常用的场景：**  

用来发现系统内核中IO hang住的场景。参见下面的“使用方法举例”部分。

**实现原理：**  
在后台，会常驻一个线程(`watchdog thread`)，定期的唤醒，然后对注册过的线程进行检查。

对于注册过的线程，如果(“当前时间” - “注册时间”) 大于 “线程注册时所提供的‘时间长度’”，那么说明该线程可能被hang住了，就触发打印"warning"日志，并打印堆栈（同时包括“系统级”和“用户级”的堆栈）。  

**注意：**  
`SCOPED_WATCH_STACK`宏，可以嵌套使用。但是嵌套层级有限制，目前硬编码为`8`层。

### **关于“时间长度”的注意事项：**  

这里的检测 是无法做到非常精准的。
1. 无法在超过 时间长度阈值时，立即打印堆栈。因为此时，后台检测线程可能正在休眠中。
2. 也无法做到 捕获到所有hang住时间 超过指定长度的线程。因为 在‘检测线程’唤醒时，之前耗时的IO操作可能已经结束了。

**根本原因是：后台线程 是定期唤醒的。**

```
// 假设：时间长度阈值是10ms, 但是一个IO操作需要15ms，‘后台检测线程’的唤醒周期是10ms

// 场景1：
               wake-up   wake-up    
                 |         |
                 |         |
-------|-------+-|---------|--+------------------> 时间线
       0       | 10        20 |    
               |              |
             op-begin(t=8)  op-end(t=23)
   
// 1. 后台线程第一次唤醒时，发现该线程已经耗时为2ms (小于阈值，即10ms)，不触发报警；
// 2. 在t=18时，操作的耗时已经超过了指定的阈值(10ms)，但是因为此时，后台检测线程正在休眠，所以也不会有报警被触发。
// 3. 在t=20时，后台检测线程再次唤醒，发现该线程已经耗时为12ms（大于阈值 10ms）,会触发报警。


// 场景2：
               wake-up   wake-up    
                 |         |
                 |         |
-------|--+------|-------+-|---------------------> 时间线
       0  |      10      | 20   25  
          |              |
        op-begin(t=3)  op-end(t=18)
   
// 1. 后台线程第一次唤醒时，发现该线程已经耗时为7ms (小于阈值，即10ms)，不触发报警；
// 2. 在t=13时，操作的耗时已经超过了指定的阈值(10ms)，但是因为此时，后台检测线程正在休眠，所以也不会有报警被触发。
// 3. 在t=20时，后台检测线程再次唤醒，但是这时，之前的线程已经结束了，所以也就不再当前的“需要监控的线程列表中”了。也就是说，“后台线程”无法发现这种情况下的 耗时的操作。

```

假定唤醒周期为`t`，操作的耗时为`T`，那么抛开其它的耗时因素：
**只有在 `T > 2 * t`的时候，才能保证被该耗时操作被检测到**。 

如果 `T <= 2 * t`，只能被部分检测到。
 
```
         wake-up   wake-up   wake-up    
           |         |         |
           |         |         |
-----------T---------T---------T---------------------> 时间线
           |                   |
        op-begin            op-end

```

### 性能要求

在实现`SCOPED_WATCH_STACK`宏的时候，就已经尽量将开销降到了最低。

开销主要来自于以下两点：
1. 一个`clock_gettime()`；
2. 一个简单的`mfence`指令；
 
基准测试的结果是： 每次调用耗时大约为`50us`。

所以，即使在关键路径上，也是可以使用的。

## 使用方法和举例

用户应该使用“`SCOPED_WATCH_STACK`宏” 来进行注册。

在参数中指定一个“时间长度”，单位是`ms`。

```
   // We expect the Write() to return in <100ms. If it takes longer than that
   // we'll see warnings indicating why it is stalled.
   {
     SCOPED_WATCH_STACK(100);
     file->Write(...);
   }
```

在这个例子中，如果`Write()`操作耗时非常长，那么就会触发它打印一个"warning"级别的堆栈信息。

## `private struct KernelStackWatchdog::TLS`

定义了一个用来保存所有监控内容的“`thread-local`变量”。

在`KernelStackWatchdog`类中，有一个类型为`TLS`的“`static thread-local`成员属性”。   
因为每个线程在需要进行监控时（即调用`SCOPED_WATCH_STACK()宏`的时候），都会访问该成员，即所有注册操作都是“线程级别”的。 为了防止额外的并发控制，这里在实现时使用“`thread local`”变量。

即每个线程的`TLS`结构，都是在第一次使用的时候进行构造，在线程结束的时候进行析构。

一旦`TLS`对象被创建，立即会将该`TLS`结构注册到 "`watchdog thread`"中。

线程结束时自动进行析构：是通过添加一个'`thread-exit function`'来完成的。（通过`kudu::threadlocal::internal::AddDestructor()`注册一个“清理函数”）（参见`src/kudu/util/threadlocal.h`）

### `enum KernelStackWatchdog::TLS::Constants`

用来定义一个常量，`SCOPED_WATCH_STACK`宏的最高嵌套层级。

目前硬编码为`8`。

### `struct KernelStackWatchdog::TLS::Frame`

因为支持嵌套地使用`SCOPED_WATCH_STACK`宏，所以对于每一个层级，都需要保存起来。

一个`Frame`对象，表示一个嵌套层级。

#### 成员

##### `start_time_`属性

注册的时间，即调用`SCOPED_WATCH_STACK`宏的时间。

在后台检测线程中，在计算时间长度时，就是拿“检测的时间” 减去 “注册的时间”。

注意1：  
这里使用的`MicrosecondsInt64`类型，而不是`MonoTime`类型。 是因为`MicrosecondsInt64`的方法是`inline`的，所以性能会高一点。

注意2：  
这里的单位是`us`。 下面的`threshold_ms_`单位是`ms`，两者是不同的。

##### `threshold_ms_`属性

线程注册时所指定的时间长度阈值。单位是`ms`。

##### `status_`属性

一个字符串，用来描述当前线程的状态。

通常，这个字符串的内容是：文件名和行号（`file:line`）。

这里是一个`char*`指针，具体的数据应该保存在一个`static`的存储中，并且在运行期间，不应该被释放。

```
    struct Frame {
      MicrosecondsInt64 start_time_;
      int threshold_ms_;
      const char* status_;
    };
```

### `struct KernelStackWatchdog::TLS::Data`

在当前"thread local"变量中保存的数据。

**这时一个`POD`类型的结构**（通过默认的“赋值操作符”就可以进行复制），所以可以很容易的进行拷贝（主要是将该对象从 `TLS`中拷贝出来）。

#### 成员
##### `frames_`属性

一个“`Frame`数组”，容量是`kMaxDepth`（默认值为`8`）。

##### `depth_`属性

嵌套的层级。

##### `seq_lock_`属性

表面上是一个“计数器”，在“尝试修改”的时候先加`1`（变成一个“奇数”），然后修改完成后，再次加`1`（变成一个“偶数”）。

实际上，是通过“计数”的方式，实现了一个“锁”。

**说明：**  
修改`TLS data`的操作，一定发生在 该`thread local`结构所对应的线程中。 后台监控线程（`watchdog thread`）只会进行读取，不会进行修改。

也就是说，对于任何一个`TLS data`结构，最多只会有“两个线程”会访问它。
1. 它所对应的线程（可能进行读写操作）；
2. 后台监控线程（只会进行读操作）；

那么，进行这里的“并发控制”，实际上就简化为：在这两个线程之间，进行并发控制。

另外，因为只有一个线程可以修改`TLS data`结构，那么就不会有`cpu cache-line bouncing`问题。

**这个“锁”的使用方式：**  

当后台监控线程("watchdog thread") 希望从一个线程中去读取 `TLS data`时，它首先要等待该变量 成为一个偶数（也就是说，没有其它的线程在对`TLS data`进行“写操作”）。

然后，监控线程会将`TLS data`拷贝到一个本地临时变量中。

最后，监控线程，会重新检查一下这个“计数器”的值。如果这个“计数器”的值，与读取`TLS data`之前的值是相等的（表示在读取过程中，没有并发的更改操作），那么就保证了 监控线程 本次读取`TLS data`是可以被使用的（即本次对`TLS data`的读取，是有效的）。

**为什么要用这样的锁，而不是使用一个真正的`mutex`:**  
答：为了达到极致性能。因为只有开销足够小，才可以在“关键路径”上使用。 

这种方式，可以让“后台监控线程”是`wait-free`的（因为它能够做到，获得锁的线程，在有限步骤内一定能够执行完毕）。

>> 问题：

有一段注释，是这么写的：
```
In particular, the watched thread is wait-free, since it doesn't need to loop or retry.
```
这里的原因是不是弄错了？ 不能说，因为没有“循环”和“重试”，就是"wait-free"的。 而且，在实现中，无论是`RunThread()`，还是`SnapshotCopy()`中，都是有循环的。

**注意事项：**  


**这个“锁”的缺点：**  
使用这种方式的“锁”，在获取对应的`TLS data`时，可能需要进行多次重试。

因为如下两个原因：
1. 该变量的值，必须是偶数时，才能进行读取；
2. 在读取`TLS data`前后，该变量的值 必须没有变化，才能返回读取的结果；

所以，
1. 如果当前值为一个奇数，那么就需要一直循环，直到它变为一个偶数；
2. 如果读取`TLS data`前后，该变量的值变化了，需要重试整个读取过程。

```
 struct Data {
      Frame frames_[kMaxDepth];
      Atomic32 depth_;

      Atomic32 seq_lock_;
    };
    Data data_;
  };
```

#### 方法
##### `SnapshotCopy()`方法

只有这一个方法，用来获取一个“有效的”快照（即不会读取到一个修改的中间状态）。

读取的结果，赋值到参数中，返回给调用者。

这里采用“乐观并发控制”：就是说，先获取到结果，然后再检查是否可以使用（判断依据是：获取前后`seq_lock_`的值是否发生修改）。

后台监控线程，会针对每个注册的线程，调用这个方法，从而判断是否应该打印该线程的“warning”日志。

### `TLS`的成员

#### `data_`
当前所有调用堆栈，以及对应的“监视内容”。

## 成员

### `static __thread tls_`
`thread-local`变量。

使用宏`DECLARE_STATIC_THREAD_LOCAL()`来进行声明的。(参见`src/kudu/util/threadlocal.h`)

### `tls_by_tid_`成员

记录需要被“监视”的线程。

### `pending_delete_`成员

需要延迟删除的`TLS`列表。 如果一个线程在结束的时候，“后台监控线程”正在读取该线程的`TLS data`，那么不可以立即从`tls_by_tid_`中立即删除该线程的结构。

相反，该线程应该将它的`TLS data`暂存在`pending_delete_`中，“后台监控线程”会定期删除这些`TLS`对象。

注意：数据类型是`std::vector<std::unique_ptr<TLS>>`。

说明：为什么成员是`std::unique_ptr<>`类型？
答：因为每个线程的`TLS`结构都是 动态`new`出来的，所以在线程结束的时候，是要将该结构`delete`掉的。

如果一个线程被延迟删除了，那么就是说要延迟删除它的`TLS`结构。

为了能够自动删除，所以在`pending_delete_`中使用`std::unique_ptr<>`结构。这样，当清空 `pending_delete_`数组时，会自动进行对象的析构。

### `log_collector_`成员

在单测中使用。是一个字符串的vector。

如果不为`nullptr`(通过`SaveLogsForTests(true)`来设置)，那么不再使用glog来向日志中打印数据，而是打印到 这个字符串数组中。

### `log_lock_`

用来保护`log_collecotr_`。

说明：向glog中打印日志时，不需要手动进行并发控制，所以不会使用到该变量。

### `tls_lock_`成员

用来保护`tls_by_tid_`和`pending_delete_`。

### `unregister_lock_`成员

用来防止当“后台监控线程”正在读取该线程的`TLS data`时，该线程因为执行结束而删除掉`TLS data`。

**为什么要使用这个锁：**  
如果不进行这个检查，可能会出现错误。 
原因是：那么在一个线程(thread_id)结束，它的`thread_id`可能会被复用。

如果在线程(`T-old`)结束时，直接删除了它的`TLS`，然后它又被复用了(新线程记为`T-new`)。而同时，"watchdog thread"可能针对该`T-old`生成了一个信号，就会错误的发给`T-new`。

**需要互斥的场景：**  
需要进行并发控制（需要获取`unregister_lock_`锁）的两个场景：
1. "watchdog thread"开始进行一轮检测时，先获取该锁；
2. 在一个线程结束，希望取消该线程的"TLS data"的注册时。通过`try_lock()`进行加锁
  + 如果`try_lock`加锁成功，说明“后台检测线程”没有正在运行，那么可以将该线程的`TLS data`删除掉；
  + 如果`try_lock`加锁失败，说明“后台检测线程”正在运行中，这时不能直接清理，而是将该线程的`TLS data`暂存到`pending_delete_`中。

说明：如果在`Unregister()`中，拿到了该锁，造成"watchdog thread"等待该锁，是没有问题的。因为"watchdog thread"是后台线程，阻塞一段时间是没有问题的。

**注意加锁顺序：**  
如果需要同时获取 `unregister_lock_`，(`tls_lock_`或`log_lock_`)，那么需要先获取`unregister_lock_`。

### `thread_`成员

当前“后台监控线程”自身。

### `finish_`成员

`CountDownLatch`类型，充当一个信号量。用来通知要将当前“watchdog thread”停掉。

```
  DECLARE_STATIC_THREAD_LOCAL(TLS, tls_);

  typedef std::unordered_map<pid_t, TLS*> TLSMap;
  TLSMap tls_by_tid_;

  // If a thread exits while the watchdog is in the middle of accessing the TLS
  // objects, we can't immediately delete the TLS struct. Instead, the thread
  // enqueues it here for later deletion by the watchdog thread within RunThread().
  std::vector<std::unique_ptr<TLS>> pending_delete_;

  // If non-NULL, warnings will be emitted into this vector instead of glog.
  // Used by tests.
  gscoped_ptr<std::vector<std::string> > log_collector_;

  // Lock protecting log_collector_.
  mutable simple_spinlock log_lock_;

  // Lock protecting tls_by_tid_ and pending_delete_.
  mutable simple_spinlock tls_lock_;

  // Lock which prevents threads from unregistering while the watchdog
  // sends signals.
  //
  // This is used to prevent the watchdog from sending a signal to a pid just
  // after the pid has actually exited and been reused. Sending a signal to
  // a non-Kudu thread could have unintended consequences.
  //
  // When this lock is held concurrently with 'tls_lock_' or 'log_lock_',
  // this lock must be acquired first.
  Mutex unregister_lock_;

  // The watchdog thread itself.
  scoped_refptr<Thread> thread_;

  // Signal to stop the watchdog.
  CountDownLatch finish_;
```

## 接口列表

该类没有任何`public`的接口。

用户代码应该使用`SCOPED_WATCH_STACK`宏来使用它。

### 构造函数

会创建“后台监控线程("watchdog thread")”。

注意：因为当前线程是"watchdog thread"，所以创建的新线程，不能开启“监视”(指定`Thread::No_STACK_WATCHDOG`)。

```
KernelStackWatchdog::KernelStackWatchdog()
  : log_collector_(nullptr),
    finish_(1) {

  // During creation of the stack watchdog thread, we need to disable using
  // the stack watchdog itself. Otherwise, the 'StartThread' function will
  // try to call back into initializing the stack watchdog, and will self-deadlock.
  CHECK_OK(Thread::CreateWithFlags(
      "kernel-watchdog", "kernel-watcher",
      boost::bind(&KernelStackWatchdog::RunThread, this),
      Thread::NO_STACK_WATCHDOG,
      &thread_));
}
```
### `private Register()`

```
void KernelStackWatchdog::Register(TLS* tls) {
  int64_t tid = Thread::CurrentThreadId();
  lock_guard<simple_spinlock> l(tls_lock_);
  InsertOrDie(&tls_by_tid_, tid, tls);
}
```

### `private Unregister()`

线程退出的时候，会触发调用该方法。

触发的原理是：在线程创建`TLS`的时候，会通过`kudu::threadlocal::internal::AddDestructor()`注册一个“清理函数”，来保证在线程退出的时候，会调用该“清理函数”（参见`src/kudu/util/threadlocal.h`）.

```
void KernelStackWatchdog::Unregister() {
  int64_t tid = Thread::CurrentThreadId();

  std::unique_ptr<TLS> tls(tls_);
  {
    std::unique_lock<Mutex> l(unregister_lock_, std::try_to_lock);
    lock_guard<simple_spinlock> l2(tls_lock_);
    CHECK(tls_by_tid_.erase(tid));
    if (!l.owns_lock()) {
      // The watchdog is in the middle of running and might be accessing
      // 'tls', so just enqueue it for later deletion. Otherwise it
      // will go out of scope at the end of this function and get
      // deleted here.
      pending_delete_.emplace_back(std::move(tls));
    }
  }
  tls_ = nullptr;
}
```

这里先使用一个`std::unique_ptr<>`来封装`tls_`，是为了不进行显式的调用 `delete`。

这里的实现能够保证：如果需要析构`tls_`（在加入`pending_delete_`时不会进行析构），一定是在锁外(`unregister_lock_`锁和`tls_lock_`锁)进行的。

注意：这里通过`try_lock()`进行加锁的。

1. 如果`try_lock`加锁成功，说明“后台检测线程”没有正在运行，那么可以将该线程的`TLS data`删除掉；
 2. 如果`try_lock`加锁失败，说明“后台检测线程”正在运行中，这时不能直接清理，而是将该线程的`TLS data`暂存到`pending_delete_`中。

注意1：放到`pending_delete_`时，是使用`std::unique_ptr<>`封装的。所以，后续将元素从`pending_delete_`中移除时，会自动析构对应的`TLS`结构。

注意2：无论是否虽然放到`pending_delete_`中，都会将`tls_by_tid_`中的映射关系删除。

### `static ThreadExiting()`
在线程创建自己的`TLS`的时候，会将当前函数作为“清理函数”，注册进去。在线程退出的时候，会调用该函数。

```
void KernelStackWatchdog::ThreadExiting(void* /* unused */) {
  KernelStackWatchdog::GetInstance()->Unregister();
}
```

### `CreateAndRegisterTLS()`

```
void KernelStackWatchdog::CreateAndRegisterTLS() {
  DCHECK(!tls_);
  // Disable leak check. LSAN sometimes gets false positives on thread locals.
  // See: https://github.com/google/sanitizers/issues/757
  debug::ScopedLeakCheckDisabler d;
  auto* tls = new TLS();
  KernelStackWatchdog::GetInstance()->Register(tls);
  tls_ = tls;
  kudu::threadlocal::internal::AddDestructor(&ThreadExiting, nullptr);
}
```
>> 问题：`ScopedLeakCheckDisabler`的工作原理是？

### `private RunThread()`  -- 线程具体工作的函数

只要`finish_`没有被标记，该线程就一直工作。

因为是单例类，所以只有进程结束的时候，才会析构该对象（从而把`finish_`信号量进行标记）。

堆栈分为两种：
1. 内核态的堆栈：通过读取文件`/proc/pid/stack`文件；
2. 用户态的堆栈：调用`DumpThreadStack()`进行打印；


>> 问题：  
```
这段注释： 
// NOTE: it's still possible that the thread will have exited in between grabbing its pointer
// and sending a signal, but DumpThreadStack() already is safe about not sending a signal
// to some other non-Kudu thread.
```
这个场景会在什么场景下发生？

```
void KernelStackWatchdog::RunThread() {
  while (true) {
    MonoDelta delta = MonoDelta::FromMilliseconds(FLAGS_hung_task_check_interval_ms);
    if (finish_.WaitFor(delta)) {
      // Watchdog exiting.
      break;
    }

    // Don't send signals while the debugger is running, since it makes it hard to
    // use.
    if (IsBeingDebugged()) {
      continue;
    }

    // Prevent threads from deleting their TLS objects between the snapshot loop and the sending of
    // signals. This makes it safe for us to access their TLS.
    //
    // NOTE: it's still possible that the thread will have exited in between grabbing its pointer
    // and sending a signal, but DumpThreadStack() already is safe about not sending a signal
    // to some other non-Kudu thread.
    MutexLock l(unregister_lock_);

    // Take the snapshot of the thread information under a short lock.
    //
    // 'tls_lock_' prevents new threads from starting, so we don't want to do any lengthy work
    // (such as gathering stack traces) under this lock.
    TLSMap tls_map_copy;
    vector<unique_ptr<TLS>> to_delete;
    {
      lock_guard<simple_spinlock> l(tls_lock_);
      to_delete.swap(pending_delete_);
      tls_map_copy = tls_by_tid_;
    }
    // Actually delete the no-longer-used TLS entries outside of the lock.
    to_delete.clear();

    MicrosecondsInt64 now = GetMonoTimeMicros();
    for (const auto& entry : tls_map_copy) {
      pid_t p = entry.first;
      TLS::Data* tls = &entry.second->data_;
      TLS::Data tls_copy;
      tls->SnapshotCopy(&tls_copy);
      for (int i = 0; i < tls_copy.depth_; i++) {
        const TLS::Frame* frame = &tls_copy.frames_[i];

        int paused_ms = (now - frame->start_time_) / 1000;
        if (paused_ms > frame->threshold_ms_) {
          string kernel_stack;
          Status s = GetKernelStack(p, &kernel_stack);
          if (!s.ok()) {
            // Can't read the kernel stack of the pid, just ignore it.
            kernel_stack = "(could not read kernel stack)";
          }

          string user_stack = DumpThreadStack(p);

          // If the thread exited the frame we're looking at in between when we started
          // grabbing the stack and now, then our stack isn't correct. We shouldn't log it.
          //
          // We just use unprotected reads here since this is a somewhat best-effort
          // check.
          if (ANNOTATE_UNPROTECTED_READ(tls->depth_) < tls_copy.depth_ ||
              ANNOTATE_UNPROTECTED_READ(tls->frames_[i].start_time_) != frame->start_time_) {
            break;
          }

          lock_guard<simple_spinlock> l(log_lock_);
          LOG_STRING(WARNING, log_collector_.get())
              << "Thread " << p << " stuck at " << frame->status_
              << " for " << paused_ms << "ms" << ":\n"
              << "Kernel stack:\n" << kernel_stack << "\n"
              << "User stack:\n" << user_stack;
        }
      }
    }
  }
}
```

注意1：整个检查期间，是一直加着`unregister_lock_`锁的。就会为了防止在，读取`tls_`期间，某个线程退出，然后导致其对应的`TLS`结构被析构，从而导致在这个“watch dog 线程”出现非法访问。

这里加了`unregister_lock_`锁以后，如果有线程运行完毕退出，那么它对应的`TLS`结构，不会立即被析构，而是保存在`pending_delete_`中。然后对象的线程就直接退出了（即对应的线程，并不是等待这里释放`unregister_lock_`锁。）在退出的线程中，通过`try_lock`来检查是否已经被加锁（即`watch dog thread`是否正在读取）。

也正是因为“退出的线程”会将对应的`TLS`放到`pending_delete_`中，所以在该`watch dog`线程中，需要检查并删除`pending_delete_`的内容（否则，就没有人来删除这其中的`TLS`了）。 说明：在访问`pending_delete_`的时候，需要在`tls_lock_`锁的保护下。

注意2：因为拥有`tls_lock_`锁而不释放，会导致新的线程无法进行注册。所以，在实现时，应尽量让锁内的工作越少越好。
这里，
1. 先在锁外实例化`to_delete`，然后在锁内调用`swap()`来快速交换数据；
2. 不是在锁内去遍历`tls_by_tid_`，而是拷贝一遍(拷贝到临时变量`tls_map_copy`)，然后就可以释放`tls_lock_`锁了。

还有：因为`to_delete.clear()`，会触发对`TLS`对象的析构，这个也在“锁外”进行。

注意3： 在遍历拷贝出来的`tls_map_copy`时，如果需要报警，在获取完堆栈的过程中，有可能其中的元素已经被修改了（必须线程已经离开了之前的堆栈）。所以对于每个元素，在获取完堆栈后，都会进行重新判断下，是否已经不需要再报警了（如果程序已经离开了之前需要报警的堆栈，而且，因为不知道什么时间离开的，可能当前获取的堆栈已经是不准确的了，所以也就不进行再报警了）。  

判断方法：用拷贝出来的`tls_copy`和“当前的`tls_`”进行比较（通过比较 “嵌套深度”、以及相应`Frame`的开始时间），如果不同，那么不进行输出该`Frame`。

注意4：这里在获取完堆栈后的“重新比较”时，并没有重新拷贝出来一个新的`tls_`，而是直接访问的`tls_`。  
在代码中，有使用`ANNOTATE_UNPROTECTED_READ()`宏。

>> 问题：`ANNOTATE_UNPROTECTED_READ`宏是干什么用的？

# `class ScopeWatchKernelStack`

工具类。 类似于`lock_guard`。
1. 将一些操作 封装在“构造函数”中。 比如：构建和修改 `TLS data`。
2. 将资源的释放等操作，封装在“析构函数”中。

该对象所在的`scope`，如果执行时间超过了指定的“时间长度”（单位是`ms`），那么就触发打印"warning"日志。

## 构造函数

在构造含对象时，可以传入一个`label`。 在需要打印"warning"日志的时候，也会打印这个`label`。

所以可以在`label`中，保存一些“帮助调试的信息”。如果通过`SCOPED_WATCH_STACK`宏来声明时，相应的`label`为 “文件名:行号”信息。

这个`label`是一个`char*`类型，具体的数据由调用者维护，并且**它的内容，就不会被拷贝，也不会被析构**（即在该类析构的时候，不会释放该指针指向的数据）。

**注意：不是每次访问都调用`GetTLS()`**  
`GetTLS()`是一个 延迟初始化的方法。 为了提高效率，这里先手动直接获取`tls`成员（一个线程，只有第一次调用是不成功的，其余都是成功的）。只有在确实没有尚未被初始化的时候，才会通过调用`GetTLS()`进行获取（这时会在内部初始化`thread local`变量）。

也就是说，这里将`GetTLS()`当做`createTLS()`来使用的。

这样的好处是：不许哟将`TLS`的构造部分都进行`inline`。



注意：这里如果传入的参数`threadhold_ms`的值 小于等于0，那么就不会进行监视。

```
  ScopedWatchKernelStack(const char* label, int threshold_ms) {
    if (threshold_ms <= 0) return;

    KernelStackWatchdog::TLS* tls = KernelStackWatchdog::tls_;
    if (PREDICT_FALSE(tls == NULL)) {
      tls = KernelStackWatchdog::GetTLS();
    }
    KernelStackWatchdog::TLS::Data* tls_data = &tls->data_;

    // "Acquire" the sequence lock. While the lock value is odd, readers will block.
    // TODO: technically this barrier is stronger than we need: we are the only writer
    // to this data, so it's OK to allow loads from within the critical section to
    // reorder above this next line. All we need is a "StoreStore" barrier (i.e.
    // prevent any stores in the critical section from getting reordered above the
    // increment of the counter). However, atomicops.h doesn't provide such a barrier
    // as of yet, so we'll do the slightly more expensive one for now.
    base::subtle::Acquire_Store(&tls_data->seq_lock_, tls_data->seq_lock_ + 1);

    KernelStackWatchdog::TLS::Frame* frame = &tls_data->frames_[tls_data->depth_++];
    DCHECK_LE(tls_data->depth_, KernelStackWatchdog::TLS::kMaxDepth);
    frame->start_time_ = GetMonoTimeMicros();
    frame->threshold_ms_ = threshold_ms;
    frame->status_ = label;

    // "Release" the sequence lock. This resets the lock value to be even, so readers
    // will proceed.
    base::subtle::Release_Store(&tls_data->seq_lock_, tls_data->seq_lock_ + 1);
  }
```

## 析构方法

**不需要任何的加锁操作**  
这里只会修改`depth_`并且修改过程是“原子的”。

注意：不会对相应的`Frame`进行任何修改。

即使有"watchdog thread"并发的读取该线程的`TLS data`，那么只有可能读取到两个结果。
1. 修改深度前的`TLS data`；
2. 修改深度后的`TLS data`；

因为对应的`Frame`数组是定长的，所以无论读取到哪个深度，都能够读取到一个`Frame`对象。

也就是说：并发的"watchdog thread"读取到的数据，可能是“过时的”，但一定不会是错乱的数据。

```
  ~ScopedWatchKernelStack() {
    if (!KernelStackWatchdog::tls_) return;

    KernelStackWatchdog::TLS::Data* tls = &KernelStackWatchdog::tls_->data_;
    int d = tls->depth_;
    DCHECK_GT(d, 0);

    // We don't bother with a lock/unlock, because the change we're making here is atomic.
    // If we race with the watchdog, either they'll see the old depth_ or the new depth_,
    // but in either case the underlying data is perfectly valid.
    base::subtle::NoBarrier_Store(&tls->depth_, d - 1);
  }
```

# `SCOPED_WATCH_STACK`宏

```
#define SCOPED_WATCH_STACK(threshold_ms) \
  ScopedWatchKernelStack _stack_watcher(__FILE__ ":" AS_STRING(__LINE__), threshold_ms)
```

# `GFLAGS`变量
# `FLAGS_hung_task_check_interval_ms`

默认值为`200ms`。表示"watchdog thread"每轮的检查间隔。


# `FLAGS_inject_latency_on_kernel_stack_lookup_ms`
在测试中，在获取一个线程的“内核堆栈”时，延迟的时间（单位是毫秒）。

默认是`0ms`，即不延迟。



