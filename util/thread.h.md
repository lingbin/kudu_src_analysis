[TOC]

文件：`src/kudu/util/thread.h`

设置线程的名字：https://blog.csdn.net/lijzheng/article/details/39963331 

## `class ThreadJoiner`

工具类。 用来`join`一个线程。如果等待的时间过长，那么会打印`warning`日志。

有3种控制`warning`日志打印的参数：
1. 开始时间：默认为1秒。如果超过指定时间，开始打印日志。
2. 时间间隔：默认为1秒。在开始打印日志后，每隔指定的时间，打印一次日志：这个方法，只要线程不结束，就会一直打印日志。
3. 结束时间：默认为-1，表示不限制。如果超过这个时间，即使线程仍然没有结束，那么不再继续阻塞地等待（目标线程结束），而是直接返回失败(`Status::Aborted`)。（这个方法，主要为了防止一些线程永远不能结束，打印太多的日志）

```
                warn_every_ms_
                     ^
               /-----|-----\
     0         |     |     |  
   --+------+----T-----T-----T---+--------------+-->  时间线
     |      |                    |              |
     |    warn_after_ms_   give_up_after_ms_    |
     |                     (函数返回Aborted)    |
  开始join()                                 线程结束
   
说明：
1. warn_after_ms_ 和 give_up_after_ms_ 都是相对于开始“join()”的时间点；
2. warn_every_ms_ 是从 warn_after_ms_开始，每隔一段时间进行报警。
```

### 使用举例

```
    ThreadJoiner(&my_thread)
        .warn_after_ms(1000)
        .warn_every_ms(5000)
        .Join();
```

### `匿名的内部枚举类`

用来定义一些常量，指定控制参数的默认值，分别对应上述`3`种控制方法。

```
 enum {
    kDefaultWarnAfterMs = 1000,
    kDefaultWarnEveryMs = 1000,
    kDefaultGiveUpAfterMs = -1 // forever
  };
```

### 成员

因为该类就是要等待一个线程的结束，所以必然有一个成员是：要等待的目标线程（`thread_`）。

其余3个是控制等待行为的3个属性。

```
  Thread* thread_;

  int warn_after_ms_;
  int warn_every_ms_;
  int give_up_after_ms_;
```

### `setter`方法列表

分别对应上面提到的3种方法。

为了能够“流式地调用”，下面3个方法的返回值，都是当前对象（`ThreadJoiner`对象）的引用。

该类的3个控制等待行为的方法如下：
1. `warn_after_ms()`
2. `warn_every_ms()`
3. `give_up_after_ms()`

其实，这3个方法，都属于`setter`方法，就是给相应的 控制属性 赋值。

### `Join()`  

该类最核心的方法：就是要真正去等待(`join`)指定的线程。

流程：
1. 检查是否在等待自己（因为：一个线程，是不能join等待自己的）；
2. 一个线程，只能被别的线程等待一次。  
    如果一个线程，已经被join过了（说明那个线程已经结束了，所以这里可以直接返回了）
3. 下面就是根据3个控制属性，不断的进行wait和检查。

```
Status ThreadJoiner::Join() {
  if (Thread::current_thread() &&
      Thread::current_thread()->tid() == thread_->tid()) {
    return Status::InvalidArgument("Can't join on own thread", thread_->name_);
  }

  // Early exit: double join is a no-op.
  if (!thread_->joinable_) {
    return Status::OK();
  }

  int waited_ms = 0;
  bool keep_trying = true;
  while (keep_trying) {
    if (waited_ms >= warn_after_ms_) {
      LOG(WARNING) << Substitute("Waited for $0ms trying to join with $1 (tid $2)",
                                 waited_ms, thread_->name_, thread_->tid_);
    }

    int remaining_before_giveup = MathLimits<int>::kMax;
    if (give_up_after_ms_ != -1) {
      remaining_before_giveup = give_up_after_ms_ - waited_ms;
    }

    int remaining_before_next_warn = warn_every_ms_;
    if (waited_ms < warn_after_ms_) {
      remaining_before_next_warn = warn_after_ms_ - waited_ms;
    }

    if (remaining_before_giveup < remaining_before_next_warn) {
      keep_trying = false;
    }

    int wait_for = std::min(remaining_before_giveup, remaining_before_next_warn);

    if (thread_->done_.WaitFor(MonoDelta::FromMilliseconds(wait_for))) {
      // Unconditionally join before returning, to guarantee that any TLS
      // has been destroyed (pthread_key_create() destructors only run
      // after a pthread's user method has returned).
      int ret = pthread_join(thread_->thread_, nullptr);
      CHECK_EQ(ret, 0);
      thread_->joinable_ = false;
      return Status::OK();
    }
    waited_ms += wait_for;
  }
  return Status::Aborted(strings::Substitute("Timed out after $0ms joining on $1",
                                             waited_ms, thread_->name_));
}
```

>> 问题：这里的`pthread_join()`的注释，是什么意思？

## `class Thread`

该类是对`pthread`封装，从而可以通过`ThreadMgr`来进行管理。

说明1：`ThreadMgr`是一个`private`的类（实现在`thread.cc`中），用来跟踪记录所有 活着的线程，从而可以在 网页 上对这些线程进行监控。

说明2：`Thread`类的接口，是`boost::thread`接口的一个子集：
1. 构造函数是基本相同的，但是增加了两个参数：1) 类别; 2) 名称。利用这两个参数，可以在web界面上更方便的查看信息。
2. `boost::thread`仅仅支持`Join()`。

说明3：每个`Thread`对象都知道自己在操作系统中的id（即`thread_id`, `TID`，通过将`thread_id`保存在`tid_`属性中），可以有以下几个用处：
1. 可以将`debugger` attach 到指定的线程（通过`TID`）；
2. 可以从操作系统工具，来查看线程的资源使用情况('retrieve resource-usage statistics');
3. 将该线程添加到“资源控制组”中（'resource control group'）;

>> 问题：这里的`debugger`指的是什么？是`GDB`吗？

说明4：**`Thread`类是可以共享的，但是有 限制**  

说明5：因为继承了`RefCountedThreadSafe<Thread>`，所以可以使用`scoped_refptr`。

但是要注意：`Thread`对象的共享，但是和传统使用`scoped_refptr`的方式略有不同。

最多只会有两个引用：  
1. 创建该线程的调用者(parent)；
2. 线程本身(chile)；

说明6：只有两个方法能够修改该“线程对象”的状态，并且这两个函数的功能是有限制的：    
1. `Join()`：线程不能使用`Join()`等待自己。
2. 析构函数：只有当“引用计数”不为`0`的时候，“析构函数”中才会进行真正的析构。

**进行限制的目的：可以在不加锁的情况下，访问`Thread`的内部成员。**

**注意：**
这里虽然提供了对线程的封装（记为`kudu::Thread`），但一个进程中，并不是所有的线程都是通过`Kudu::Thread`启动的。比如 主线程(`man thread`)就不是。

>> 扩展：这可能也是很多系统实现中，只要进程起来后，“主线程”就不做任何工作（一般是进行循环的sleep）的原因之一。因为通过“特定封装”后启动的线程，都是可以通过类似`ThreadMgr`的结构来进行管理和调试，但是像主线程这种线程，是无法通过`ThreadMgr`进行管理的。  
>> 因为主线程是进程的第一个线程，主线程创建的时候，还不存在`ThreadMgr`对象。所以主线程，是无法像其它线程一样，封装在`kudu::Thread`中，并将进程启动的入口放在`kudu::Thread`中的。

### `enum Thread:CreateFlag`

默认创建的线程，flag的值都是`NO_FLAGS`（创建出来的线程，都是受"watchdog thread"监控的）。

如果指定了`NO_STACK_WATCHDOG`，那么创建出来的线程，是不受`watchdog thread`监控的。

注意：`watchdog thread`也是一个线程。创建`watchdog thread`的时候，一定要指定为`NO_STACK_WATCHDOG`。

如果使用`NO_FLAGS`去创建"watchdog thread"，因为要收到监控，所以创建该线程时需要依赖`KernelStackWatchdog`单例对象，就会去创建该对象，但是`KernelStackWatchdog`的构造函数中会直接创建需要的“后台监控线程”，这样调用关系就形成环了。

```
  // Flags passed to Thread::CreateWithFlags().
  enum CreateFlags {
    NO_FLAGS = 0,

    // Disable the use of KernelStackWatchdog to detect and log slow
    // thread creations. This is necessary when starting the kernel stack
    // watchdog thread itself to avoid reentrancy.
    NO_STACK_WATCHDOG = 1 << 0
  };
```

>> reentrancy: 可重入；可重入性；  

说明：这里的`NO_STACK_WATCHDOG`的值是通过“对`1`进行移位”来得到的。  
在代码中，是可以通过“按位相与”来判断是否设置了这个值。

### `匿名的enum`

匿名的`enum`，就相当于在当前类中声明了一组 常量。

```
  enum {
    INVALID_TID = -1,
    PARENT_WAITING_TID = -2,
  }
```

这两个变量，用来标识 新创建的子线程 的状态，参见`tid_`部分的说明。

### `ThreadFunctor`  -- 类型别名

封装该`Thread`要去执行的线程。

```
typedef boost::function<void ()> ThreadFunctor;
```

### 成员

#### `thread_`

`pthread_t`类型。  

因为在`pthread库`中，使用`pthread_t`作为一个线程的标识。所以这个成员，就是 `pthread库`中的“线程标识”。

#### `const category_`
当前线程所属的“类别”。

#### `cosnt name_`
当前线程的“名称”。

注意：`category_`和`name_`都是“常量”，不可以被修改。

#### `tid_`

在操作系统中的`thread_id`。

`tid_`的取值，有以下3种可能：  
1. `INVALID_TID`(`-1`)：线程尚未开始执行；或者已经运行结束；
2. `PARENT_WAITING_TID`(`-2`)：调用者已经启动了该线程，但是这个线程尚未真正开始执行。 （因为一个线程的`thread_id`，只有在操作系统真正启动运行线程后，才会知道，而这里因为线程尚未被真正执行起来，所以是不知道`thread_id`的）；
3. 一个正数：线程正在运行中；

也就是说，一个`Thread`对象的`tid_`属性，

```
              INVALID_TID         （对象刚被创建）
                   |
                   >
              PARENT_WAITING_TID  （父线程已经启动，子线程尚未启动）
                   |
                   >
                一个正数          （子线程真正启动）
                   |
                   >
                INVALID_TID        线程已经结束运行
                
```

#### `const functor_`
当前线程所要去执行的“函数”。

#### `done_`
用来通知所有正在`join`该`Thread`的所有线程：通知它们当前`Thread`已经结束运行。

#### `joinable_`
标识当前线程是否可以进行`join`。

说明：一个线程，是不需要被重复`join`的。

#### `static tls_`
`thread local`变量。

>> 它的值是？


```
  pthread_t thread_;

  const std::string category_;
  const std::string name_;

  int64_t tid_;

  const ThreadFunctor functor_;

  // Joiners wait on this latch to be notified if the thread is done.
  //
  // Note that Joiners must additionally pthread_join(), otherwise certain
  // resources that callers expect to be destroyed (like TLS) may still be
  // alive when a Joiner finishes.
  CountDownLatch done_;

  bool joinable_;

  // Thread local pointer to the current thread of execution. Will be NULL if the current
  // thread is not a Thread.
  static __thread Thread* tls_;
```

### 创建线程的多个方法

该类的创建线程的方法，是模拟的`boost::thread`。

>> mimics: 模拟  

#### `Create()`和`CreateWithFlags()`

根据“参数的个数”，区分为了多个`Create()`方法。

有以下几个参数
1. `category`： 指定线程的类别。一个字符串。在`debug web界面`，属于同一个类别的线程会放在一起。
2. `name`: 线程的名字。为了保证唯一性，会将`-$thread_id`添加到名称的末尾。
3. `F`: 一个“函数对象”，重载了`operator()`方法。在一个新的线程启动后，就会立即执行该“函数对象”。
4. `A1...An`: 函数指定具体的参数列表。
5. `holder`: 可选的共享指针，指向这个“刚被创建的线程”。

默认所有的`Create()`接口创建出来的线程，都是受“监视”的。（参见：`src/kudu/util/kernel_stack_watchdog.h`）

目前最多支持6个参数。如果需要增加参数的个数，那么在这里再添加一个Create()方法即可。

还有一个比较特殊的方法: `CreateWithFlags()`，相当于“没有 `A1...An`”参数列表 的`Create()`函数。区别是：它可以指定`flags`。

目前，`CreateWithFlags()`只有1个地方使用到了：在创建"watchdog thread"的时候使用(参见`src/kudu/util/kernel_stack_watchdog.h`)

上述多个创建线程的方法，都是通过调用`StartThread()`方法来进行真正的创建。

##### `private StartThread()`方法

新创建的线程，都是从`SuperviseThread()`开始执行。

在该方法中，会创建`Thread`对象，然后进行一些线程初始化工作。新创建的子线程的`TID`(`tid_`)是在`SuperviseThread()`中做的（在子线程中做的）。

如果参数`holder`不为空，那么会将“新创建的`Thread`对象”保存在其中。

如果创建过程大于500ms，会打印"warning"日志：提示新建线程比较慢。

**注意：父子线程不会进行握手**

即在父线程调用`StartThread()`去创建线程时，不会等待子线程真正运行起来以后，才返回。

这么做的原因是：为了提高创建线程的速度。参见`commit_id: 9fa9cf181ded1736c6014792fe619f5e0b9a94b4`

因此，在父线程中执行`Create()`结束时，可能子线程还没有对应的`thread_id`（因为尚未真正开始运行）。

**使用`GoogleOnceInit`来保证`ThreadMgr`对象只会被初始化一次**

可能会调用`GoogleOnceInit(&once, &InitThreading)`的有两个地方：
1. `StartThread()`；
2. `StartThreadInstrumentation()`；

**`tid_`的值**  
在该方法内部，在创建出来`Thread`对象以后，这个“新建子线程”的`tid_`会先被赋值为`PARENT_WAITING_TID`。表示已经开始创建线程，但是线程还没有真正被执行。 

新创建的子线程，会首先`SuperviseThread()`，在其中会获取到真正的`tid_`。

**资源限制--线程数量**  
如果创建失败，会检查当前进程中对资源的限制信息。

>> 问题：怎么对线程资源进行限制？

**注意：必须在“父线程”中设置“子线程的`joinable_`属性”**  

首先，因为只有“父线程”（或者“和父线程有通信的线程”）可以去join“新创建的 子线程”。(**一个线程不能 join 自己**)。

其次，在`StartThread()`方法返回之后，这个`joinable_`属性就一定要被设置好。因为在`StartThread()`返回后，“父线程”可能会在任意时刻去 join “子线程”。

所以，在父线程（即在`StartThread()`方法内）中设置该属性，就一定能够保证，只要`StartThread()`返回，子线程的`joinable_`属性一定被设置过了。

>> 扩展：在kudu的早期实现，`StartThread()`函数在返回之前，“父线程”和“子线程”会进行一些握手操作（其中，“父线程”上会等待“子线程”创建好`tid_`才会返回）。  
>> 如果有这种“握手规则”，那么也是可以在子线程中中设置`joinable_`的：只要在“子线程”中在设置`tid_`之前，就设置好`joinable_`属性，也是可以满足要求的。  
>> 只不过这种方式，对子线程的执行顺序就有要求了，容易留坑。

```
Status Thread::StartThread(const string& category, const string& name,
                           const ThreadFunctor& functor, uint64_t flags,
                           scoped_refptr<Thread> *holder) {
  TRACE_COUNTER_INCREMENT("threads_started", 1);
  TRACE_COUNTER_SCOPE_LATENCY_US("thread_start_us");
  GoogleOnceInit(&once, &InitThreading);

  const string log_prefix = Substitute("$0 ($1) ", name, category);
  SCOPED_LOG_SLOW_EXECUTION_PREFIX(WARNING, 500 /* ms */, log_prefix, "starting thread");

  // Temporary reference for the duration of this function.
  scoped_refptr<Thread> t(new Thread(category, name, functor));

  // Optional, and only set if the thread was successfully created.
  //
  // We have to set this before we even start the thread because it's
  // allowed for the thread functor to access 'holder'.
  if (holder) {
    *holder = t;
  }

  t->tid_ = PARENT_WAITING_TID;

  // Add a reference count to the thread since SuperviseThread() needs to
  // access the thread object, and we have no guarantee that our caller
  // won't drop the reference as soon as we return. This is dereferenced
  // in FinishThread().
  t->AddRef();

  auto cleanup = MakeScopedCleanup([&]() {
      // If we failed to create the thread, we need to undo all of our prep work.
      t->tid_ = INVALID_TID;
      t->Release();
    });

  if (PREDICT_FALSE(FLAGS_thread_inject_start_latency_ms > 0)) {
    LOG(INFO) << "Injecting " << FLAGS_thread_inject_start_latency_ms << "ms sleep on thread start";
    SleepFor(MonoDelta::FromMilliseconds(FLAGS_thread_inject_start_latency_ms));
  }

  {
    SCOPED_LOG_SLOW_EXECUTION_PREFIX(WARNING, 500 /* ms */, log_prefix, "creating pthread");
    SCOPED_WATCH_STACK((flags & NO_STACK_WATCHDOG) ? 0 : 250);
    int ret = pthread_create(&t->thread_, nullptr, &Thread::SuperviseThread, t.get());
    if (ret) {
      string msg = "";
      if (ret == EAGAIN) {
        uint64_t rlimit_nproc = Env::Default()->GetResourceLimit(
            Env::ResourceLimitType::RUNNING_THREADS_PER_EUID);
        uint64_t num_threads = thread_manager->ReadThreadsRunning();
        msg = Substitute(" ($0 Kudu-managed threads running in this process, "
                         "$1 max processes allowed for current user)",
                         num_threads, rlimit_nproc);
      }
      return Status::RuntimeError(Substitute("Could not create thread$0", msg), strerror(ret), ret);
    }
  }

  // The thread has been created and is now joinable.
  //
  // Why set this in the parent and not the child? Because only the parent
  // (or someone communicating with the parent) can join, so joinable must
  // be set before the parent returns.
  t->joinable_ = true;
  cleanup.cancel();

  VLOG(2) << "Started thread " << t->tid()<< " - " << category << ":" << name;
  return Status::OK();
}
```

##### `private SuperviseThread()`

新创建的子线程，入口就是这个函数。

参数是“新创建的`Thread`对象指针”。

```
void* Thread::SuperviseThread(void* arg) {
  Thread* t = static_cast<Thread*>(arg);
  int64_t system_tid = Thread::CurrentThreadId();
  PCHECK(system_tid != -1);

  // Take an additional reference to the thread manager, which we'll need below.
  ANNOTATE_IGNORE_SYNC_BEGIN();
  shared_ptr<ThreadMgr> thread_mgr_ref = thread_manager;
  ANNOTATE_IGNORE_SYNC_END();

  // Set up the TLS.
  //
  // We could store a scoped_refptr in the TLS itself, but as its
  // lifecycle is poorly defined, we'll use a bare pointer. We
  // already incremented the reference count in StartThread.
  Thread::tls_ = t;

  // Publish our tid to 'tid_', which unblocks any callers waiting in
  // WaitForTid().
  Release_Store(&t->tid_, system_tid);

  string name = strings::Substitute("$0-$1", t->name(), system_tid);
  thread_manager->SetThreadName(name, t->tid_);
  thread_manager->AddThread(pthread_self(), name, t->category(), t->tid_);

  // FinishThread() is guaranteed to run (even if functor_ throws an
  // exception) because pthread_cleanup_push() creates a scoped object
  // whose destructor invokes the provided callback.
  pthread_cleanup_push(&Thread::FinishThread, t);
  t->functor_();
  pthread_cleanup_pop(true);

  return nullptr;
}
```

**子线程会向`ThreadMgr`注册**  

该方法会向`ThreadMgr`注册自己，并且向操作系统获取“操作系统上的thread_id”（保存到`tid_`中）。

该函数最终会去执行（创建线程时所指定的）`functor_`，并且在执行结束后，从`ThreadMgr`中删除掉自己。

**参数是裸指针，在父线程删掉引用后，为什么在子线程仍然可以安全引用**
虽然`arg`参数是一个裸指针，但是`Thread`类继承自`RefCountedThreadSafe`（只有在它类内部维护的 引用计数 为`0`的时候，才会进行析构）。在`StartThread()`中已经增加了 它的引用计数，所以这里即使父线程中的`StartThread()`结束了，并且释放了它的引用，因为它的引用计数不为`0`，所以该对象不会进行析构。

### 获取属性的方法

#### `tid()`方法

获取该线程在操作系统中的`thread_id`。

注意：
如果线程还没有开始执行，那么会进行自旋，直到子线程设置好`tid_`。也就是说，如果线程正在创建过程中，那么调用该方法，可能会阻塞一小段时间。

```
  int64_t tid() const {
    int64_t t = base::subtle::Acquire_Load(&tid_);
    if (t != PARENT_WAITING_TID) {
      return tid_;
    }
    return WaitForTid();
  }
```

#### `static current_thread()`方法

获取当前`Thread`对象的指针。

这个方法是`static`方法。所以对于任何线程，直接使用`Thread::current_thread()`方法就可以获取到 自己 所对应的`Thread`对象。

注意：  
一些常规的线程管理中，都必须先拿到相应的`Thread`对象（或者`pthread_t`），才能调用一些放来 获取该线程的属性，或者管理该线程。

在这里，在任何一个`kudu::Thread`中，都会在"thread local"中保存一个指向 自己所处线程的指针，从而可以快速获取自己所对应的`Thread`对象(因为是静态方法，所以可以直接在当前线程中调用`Thread::current_thread()`)。所以，在需要操作线程的地方，就不再需要显式的传递“线程的标识符”。

**注意：如果不是`kudu::Thread`，那么调用该方法，返回`nullptr`。**  
因为只有 `kudu::Thread`在启动的时候，会向它的"thread local"变量中(`tls_`)，当前的`Thread`对象。

而非`kudu::Thread`线程，比如说“主线程”，它不是从`kudu::Thread`创建出来的，所以也更没有相应的"thread local"变量。

#### `static UniqueThreadId()`方法

获取当前线程所对应的唯一的、稳定的标识符。

这是一个`static`方法，所以可以在任何线程中使用，包括`main线程`。

注意：`main线程`，并没有使用`kudu::Thread`封装。

**通常的使用场景：**  
当需要一个 全局唯一的值，并且能够在所有线程（包括“main线程”）中都可用时。

**注意：这个方法的返回值，不是`tid_`。** 只是依赖于不同的实现，所对应的一个唯一值。 所以，这个值不应该在log中进行打印（因为根据这个值，是找不到相应的线程的，无法帮助调试）。

实现：
1. 在linux系统上，就是`pthread_self()`的值。
2. 在apple系统，通过`pthread_threadid_np()`获取的值。

说明：几种id的关系：
1. 进程pid: `getpid()`                 
2. 线程tid: `pthread_self()`     //进程内唯一，但是在不同进程则不唯一。
    就是这里 `UniqueThreadId()`的返回值。
3. 线程pid: `syscall(SYS_gettid)`     //系统内是唯一的
    即`tid_`，是`tid()`方法的返回值。

#### `static CurrentThreadId()`

返回该线程在操作系统的 thread_id。
1. 在linux上，就是`tid_`;
2. 在mac上，等于`UniqueThreadId()`；

```
  static int64_t CurrentThreadId() {
#if defined(__linux__)
    return syscall(SYS_gettid);
#else
    return UniqueThreadId();
#endif
  }
```

**对比： `CurrentThreadId()` VS  `tid()`**

在linux操作系统上，两者返回的值是相同的。

1. 能调用的对象不同：
  + `CurrentThreadId()`是`static`方法，所有线程都可以调用（包括主线程）。
  +`tid()`是普通方法，只能通过`kudu::Thread`对象进行调用。

2. `tid()`的性能会高一些。 因为`tid()`的值会缓存起来(在变量`tid_`中)，多次调用时，只有第一次是真正的进行系统调用。

**对比： `CurrentThreadId()` VS  `UniqueThreadId()`**  

在linux系统上，这两个函数返回的值是不同的。

另外，`UniqueThreadId()`的效率会略高一些。

注意：在性能要求非常高的代码路径上，应该优先使用`Thread::UniqueThreadId()`或`Thread::tid()`。 （当前函数的实现是利用`syscall(SYS_gettid)`，这是一个系统调用，性能会差一些。）

但是要注意的就是`UniqueThreadId()`返回的并不是操作系统的`thread_id`。

所以，在对性能有要求的代码路径上，有以下推荐：
+ 如果需要操作系统的"thread_id"，那么优先使用`tid()`。
+ 如果不需要操作系统的"thread_id"，那么优先使用`UniqueThreadId()`。

```
  static int64_t CurrentThreadId() {
#if defined(__linux__)
    return syscall(SYS_gettid);
#else
    return UniqueThreadId();
#endif
  }
```
### 有关行为的接口方法

#### `Join()`方法

阻塞等待该线程对象执行结束。

当该方法返回，该线程对象会从`ThreadMgr`中删除，并且在"debug web UI"中不再显示。

```
void Join() { 
  ThreadJoiner(this).Join(); 
}
```

## 一些`metric`信息

### 线程池的`metric`
有2个。
1. 累积启动了多少个线程；
2. 当前有多少个线程正在运行；

### 线程自身的`metric`
目前主要有4个metric信息，都是进程级别的：  
1. 进程的`CPU user time`;
2. 进程的`CPU system time`;
3. 进程的`voluntary context switch`：主动上下文切换；
4. 进程的`involuntary context switch`：被动上下文切换；

获取进程的统计信息：使用`getrusage()`函数，参见： http://man7.org/linux/man-pages/man2/getrusage.2.html 

说明：通过`getrusage()`可以获取当前线程的很多统计信息，只是在kudu中，只关注了这4个。

## `class ThreadMgr`

是一个单例类。用来跟踪所有“活着的”线程。

因为在`thread.cc`中实现，所以只能由`Thread`类使用。

### `private class ThreadMgr::ThreadDescriptor`

对于每个要管理的线程，希望看到的属性。

```
  {
    string name_;
    string category_;
    int64_t thread_id_;
  }
```

### `class ThreadMgr::ThreadIdHash`

重载了`operator()`方法。用在`unordered_map`的模板参数中，作为hash函数。

### `class ThreadMgr::ThreadIdEqual`

重载了`operator()`方法。用在`unordered_map`的模板参数中，用于判断是否相等。

### 成员

#### `thread_categories_`成员

是一个 两级的map: ` category => ( pthread_t => ThreadDescriptor) `。

#### `lock_`成员

用来保护`thread_categories_`和相关的metric成员（即`threads_started_metric_`和`threads_running_metric_`）

#### `threads_started_metric_`成员

累积启动过多少个线程。

#### `threads_running_metric_`成员

当前正在运行的线程个数。

```
 // A ThreadCategory is a set of threads that are logically related.
  typedef unordered_map<const pthread_t, ThreadDescriptor,
                        ThreadIdHash, ThreadIdEqual> ThreadCategory;

  // All thread categories, keyed on the category name.
  typedef unordered_map<string, ThreadCategory> ThreadCategoryMap;

  // Protects thread_categories_ and thread metrics.
  mutable rw_spinlock lock_;

  // All thread categories that ever contained a thread, even if empty.
  ThreadCategoryMap thread_categories_;

  // Counters to track all-time total number of threads, and the
  // current number of running threads.
  uint64_t threads_started_metric_;
  uint64_t threads_running_metric_;
```

### 方法

#### `AddThread()`

将一个线程注册进来。

>> 问题：这里有使用`ANNOTATE_IGNORE_SYNC_XXXXXX`宏，这个宏的作用是？为什么这里要用。 代码中有一段注释，没看明白。

注意2： 在向map中添加元素时，不能使用`EmplaceOrDie()`。 

因为如果使用`fork()`创建了“子进程”，那么在“子进程”中会有`thread_categories_`的一份拷贝。
在这里的`tid`，是通过`pthread_self()`获得的（这种方式，只能保证“进程内唯一”，但不同的进程中，该值是可能重复的）。所以，如果在“子进程”中，调用`AddThread()`的时候，新产生的`tid`(使用`pthread_self()`产生的)，是可能和“父进程中已经添加的`tid`”重复的。 

在有重复时，使用`EmplaceOrDie()`，就会造成进程挂掉。

#### `RemoveThread()`

删除一个线程。

### `static SetThreadName()`
设置当前线程的“名字”。

注意：这是一个`static`方法，所有的线程都可以调用该方法，设置“当前线程”的名字。

注意：不设置“主线程”的名字。在`linux`中，如果设置了“主线程”的名字，那么就相当于修改了“进程”的名字。所以这里会检查“当前线程”是否为“子线程”，如果是“主线程”，直接返回。

>> 备注：在`linux`上，给线程设置一个名字，是通过`prctl()`来实现的。  
>> 参见：https://blog.csdn.net/jasonchen_gbd/article/details/51308638  
linux编程 - 给线程起名字  

```
void ThreadMgr::SetThreadName(const string& name, int64_t tid) {
  // On linux we can get the thread names to show up in the debugger by setting
  // the process name for the LWP.  We don't want to do this for the main
  // thread because that would rename the process, causing tools like killall
  // to stop working.
  if (tid == getpid()) {
    return;
  }

#if defined(__linux__)
  // http://0pointer.de/blog/projects/name-your-threads.html
  // Set the name for the LWP (which gets truncated to 15 characters).
  // Note that glibc also has a 'pthread_setname_np' api, but it may not be
  // available everywhere and it's only benefit over using prctl directly is
  // that it can set the name of threads other than the current thread.
  int err = prctl(PR_SET_NAME, name.c_str());
#else
  int err = pthread_setname_np(name.c_str());
#endif // defined(__linux__)
  // We expect EPERM failures in sandboxed processes, just ignore those.
  if (err < 0 && errno != EPERM) {
    PLOG(ERROR) << "SetThreadName";
  }
}
```

## `GFLAGS`变量

### `FLAGS_thread_inject_start_latency_ms`

在测试时，指定一个时间段，表示在启动一个新线程时，先`sleep`一段时间。

默认值为`0`，即直接启动“新线程”。



