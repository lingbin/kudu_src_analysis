[TOC]

文件: `src/kudu/util/threadpool.h`

## 概述

## `struct ThreadPoolMetrics`

线程池的`metric`信息。

任何一个“线程池”，都可以指定一个该类的对象。

3个都是`Histogram`类型的metric：
1. `queue_length_histogram`: 当一个task被添加进来时，队列的长度。
2. `queue_time_us_histogram`: task在队列内等待的长度（单位是`us`）;
3. `run_time_us_histogram`：task的执行时间（单位是`us`）;

```
struct ThreadPoolMetrics {
  // Measures the queue length seen by tasks when they enter the queue.
  scoped_refptr<Histogram> queue_length_histogram;

  // Measures the amount of time that tasks spend waiting in a queue.
  scoped_refptr<Histogram> queue_time_us_histogram;

  // Measures the amount of time that tasks spend running.
  scoped_refptr<Histogram> run_time_us_histogram;
};
```

## `ThreadPoolBuilder`

用来构建一个`ThreadPool`对象。

**为什么使用`ThreadPoolBuilder`来构建`ThreadPool`对象，而不是直接通过`ThreadPool`的构造函数进行构造？**  
答：主要是为了代码的简洁。  
1. 因为一个`ThreadPool`有很多属性，如果直接通过`ThreadPool`的构造函数进行构造，那么需要传入很多个参数。
2. 通过使用`ThreadPoolBuilder`，可以很轻松的在`ThreadPool`类的外部，为其指定一些默认值。

因为`ThreadPoolBulder`类存在的意义，就是指定`ThreadPool`的相关属性，所以该类提供了各种属性的`setter`方法。 

为了能够在编码时使用“流式风格”（类似于`object.setA().setB()`），这些`setter`方法，返回值都是都是`ThreadPoolBuilder`的引用。

### 线程池的属性

#### `name_`

必需的。主要是为了调试使用。

线程池的名字，会间接决定“线程”的名字。

注意：在Linux上，一个线程的名字，最长为`16`个字符，所以“线程池”的名字最好短一些。

#### `trace_metric_prefix_`  

作为该线程池相关的`TraceMetric`的前缀。

当一个需要进行`trace`的task，在一个线程池上运行时，那么这个“线程池”就会增加它的`TraceMetric`。 包括：在队内中等待的时间；运行消耗的cpu time和wall time。

默认值是：“线程池”的名字。

比如：一个“线程池”的名字是"apply"，那么在需要跟踪的task运行时，像`apply.queue_time_us`会被增加。

`TraceMetrics`的实现，要求 counter的名字集合应该比较小。因此，如果“线程池”的名字是自动生成的，那么按照上面的默认规则（使用“线程池”的名字作为前缀），会生成非常多的“counter name”。

为了防止出现这个情况，使用该属性(`trace_metric_prefix_`)来覆盖默认规则。

这样，多个“线程池”可以使用相同的`trace_metric_prefix_`属性。

例如：“Raft线程池”的名称，命名规则是“<tablet id>-raft”。因为一台机器上，可能有几千个tablet（也就是有几千个`tablet-id`）。在这种情况下，就可以所有“raft线程池”的`trace_metric_prefix_`属性都设置为'raft'。这样就极大的减少了名字的总数量。

#### `min_thread_`  

线程池中，最少的线程数量。 默认值为0；

#### `max_thread_`  

线程池中，最少的线程数量。 默认值为 当前机器cpu的数量；

#### `max_queue_size_`  

队内的最大长度。 如果队列满了，后续向其中提交作业(`Submit()`)时，会返回`Status::ServiceUnavailable`。

默认值是`INT_MAX`。

#### `idle_timeout_`  

如果一个线程闲置了，最大的存活时间。

注意：总是会保存`min_thread_`个线程。

#### `metrics_`  

统计信息。默认为空，即默认不进行统计。

```
 private:
  const std::string name_;
  std::string trace_metric_prefix_;
  int min_threads_;
  int max_threads_;
  int max_queue_size_;
  MonoDelta idle_timeout_;
  ThreadPoolMetrics metrics_;
```

### `Build()`  

根据设置的参数，创建出一个`ThreadPool`对象。


## `class Runnable`

类似于java中的`Interface`，提供一个公共接口：`Run()`。

所有子类都要实现这个方法。

```
class Runnable {
 public:
  virtual void Run() = 0;
  virtual ~Runnable() {}
};
```

## `class ThreadPool`

### 提交作业的方式

有两种提交作业的方法
1. 直接向“线程池”提交：和传统的线程池实现相同。作业会进入到一个FIFO的队列。当有空闲线程时，依次执行。
2. 使用`ThreadPoolToken`来进行提交。

不同的模块，可以使用“不同的token”向 “同一个线程池” 来提交task，这种场景下，实际上就是 在逻辑上 按照token 把task划分成了不同的“逻辑组”。

>> not unlike: 与……没有什么不同。

使用`ThreadPoolToken`提交作业，也有两种模式：
1. `SERIAL`: 可以提交多个，但在执行的时候，一次只执行1个task。即只有上一个task执行完毕，才会执行下一个task；
2. `CONCURRENT`：所有作业并行执行的。这种方式和“直接向线程池”提交task是类似的，但是使用`token`进行提交，即有了“逻辑组”的概念，在一些场景是比较有用的。比如：单独关闭掉一个逻辑组（仅仅等待该“逻辑组”中的所有task都执行完毕），按照逻辑组进行统计metric等。 （如果直接向“线程池”提交，那么无法完成这种“逻辑组”级别的操作）

>> “直接向线程池提交task”，也称为“不使用token来提交task”。

实际上，对于直接向“线程池”提交task的方式，在内部实现的时候，也是使用`Token`方式来实现的。在“线程池”内部，维护这一个“默认的Token”（即一个默认的“逻辑组”），对于所有直接向“线程池”提交的task，都会被加入到这个“默认的token”中。


>> 补充： 使用token来提交作业，参见kudu commit: 428a34ba0aecfb81178d821732feac694a9b35c1 

主要是为了解决的问题是：就是这里`SERIAL`的功能。最终能够做到`M` 个上下文（token）共用一个 有`N`个线程的“线程池”。 而且，能够保证：在每个token内部提交的作业，仍然是按照先后顺序进行执行的。

传统的线程池，如果要实现`SERIAL`方式，方法只能是：使用 只有一个线程的‘线程池’。 这样，整个系统中，可能会导致系统中存在非常多的线程池对象。

#### 3种方式并存时，如何相互影响

“直接向线程池提交”和“使用token在`CONCURRENT`模式下提交”的task，都是按照`FIFO`顺序进行处理的。 所以这两种方式不会让对方饿死。

“使用token在`SERIAL`模式提交task”，是按照"round-robin"模式的顺序执行的。其它两种方式（tokenless 和 token的`CONCURRENT`模式）可能会将这种作业（`SERIAL`）饿死。

>> 问题： "round-robin"方式，详细是什么，多个token的时候，也是一次只运行1个？画个图。


### 使用举例

### `enum ThreadPool::ExecutionMode`

只针对`token-based`的提交方式。

注意：所有`token`都必须 先于 `ThreadPool`对象进行析构。

对于`token`的总数量，是没有限制的。

```
  // Allocates a new token for use in token-based task submission. All tokens
  // must be destroyed before their ThreadPool is destroyed.
  //
  // There is no limit on the number of tokens that may be allocated.
  enum class ExecutionMode {
    // Tasks submitted via this token will be executed serially.
    SERIAL,

    // Tasks submitted via this token may be executed concurrently.
    CONCURRENT,
  };
```
### `struct ThreadPool::Task`  

### `struct ThreadPool::IdleThread`

```
  // List of all threads currently waiting for work.
  //
  // A thread is added to the front of the list when it goes idle and is
  // removed from the front and signaled when new work arrives. This produces a
  // LIFO usage pattern that is more efficient than idling on a single
  // ConditionVariable (which yields FIFO semantics).
  //
  // Protected by lock_.
  struct IdleThread : public boost::intrusive::list_base_hook<> {
    explicit IdleThread(Mutex* m)
        : not_empty(m) {}

    // Condition variable for "queue is not empty". Waiters wake up when a new
    // task is queued.
    ConditionVariable not_empty;

    DISALLOW_COPY_AND_ASSIGN(IdleThread);
  };
```

用来封装一个空闲的线程。

### 成员

大部分成员，都是使用`lock_`进行保护。

#### 几个基本属性

这几个属性都是常量。即只要`ThreadPool`对象建立，这几个值都不可以再修改。

```
  const std::string name_;
  const int min_threads_;
  const int max_threads_;
  const int max_queue_size_;
  const MonoDelta idle_timeout_;
```

#### `lock_`成员

保护大部分成员的并发访问。

内部的“条件变量(condition variable)”也使用它。

#### `pool_status_`成员

线程池的整体状态。

在“线程池”被关闭的时候，也会设置成一个状态(`Status::ServiceUnavailable`)。

受`lock_`保护。

#### `idle_cond_`成员  

条件变量。当“线程池”所有线程都空闲(`active_threads_`的值变为`0`)时，会发出信号。

#### `no_threads_cond_`成员  

条件变量。当“线程池”没有空闲线程（`num_threads_`和`num_pending_threads_`的值都为`0`）时，发出信号。

#### `num_threads_`成员  

当前的线程总数。

受`lock_`保护。

#### `num_threads_pending_start_`成员  

正在启动的线程数量。

在这些线程启动以后，会 降低 `num_threads_pending_start_`，并且增大`num_threads_`。  

受`lock_`保护。

#### `active_threads_`成员  

当前“正在处理task”状态的线程数量。 受`lock_`保护。

**`active_threads_`和`num_threads_`**：
一个线程被启动以后，就会计入`num_threads_`。 而只有正在处理task的过程中，才会被计入`active_threads_`。

#### `total_queued_tasks_`成员  

当前在队列中等待的task总数量。  受`lock_`保护。

#### `tokens_`成员  

当前已经创建的所有`ThreadPoolToken`集合。受`lock_`保护。

注意：这里存储的只是指针。**`ThreadPoolToken`对象的析构是由客户端负责的**。

在析构`ThreadPool`对象析构的时候，并不会去析构这些`ThreadPoolToken`对象。

#### `queue_`成员  

不同的token，提交作业的先后顺序。受`lock_`保护。


#### `threads_`成员  

当前的线程对象(`kudu::Thread`对象)列表。受`lock_`保护。

注意这里保存的是`Thread`的裸指针。   因为一个`Thread`对象，只有在被从这个集合中删除的时候，才会被析构，所以这里可以直接使用裸指针。

#### `idle_threads_`成员  

当前正在空闲的、等待task的线程列表。受`lock_`保护。

注意：这里采用的是`LIFO`模式，而不是`FIFO`模式。

1. 在一个线程空闲以后，它会被添加到队列的头部；
2. 在一个task到来以后，它会从 头部 取走一个线程，触发信号量通知。

**为什么不采用`FIFO`模式**  
答：从功能上讲，使用哪种模式都行。 但是使用`LIFO`模式的效率更好些。

>> 为什么`LIFO`模式的性能好？

#### `tokenless_`成员  

一个默认的“token”（在`CONCURRENT`模式），所有不使用token提交的作业，都相当于是从这个token进行提交的。

### `metrics_`成员

“线程池”级别的`metric`。

#### 几个“线程池”级别的汇总`metric`名字

使用`trace_metric_prefix_`（如果为空时，使用“线程池”的名字）作为前缀。

1. 所有task在在队列中排队的时间；
2. 所有task执行的wall time：单位是us。
3. 所有task执行的cpu time：单位是us。

```
  Status pool_status_;

  mutable Mutex lock_;

  ConditionVariable idle_cond_;
  ConditionVariable no_threads_cond_;

  int num_threads_;
  int num_threads_pending_start_;
  int active_threads_;

  int total_queued_tasks_;

  std::unordered_set<ThreadPoolToken*> tokens_;
  std::deque<ThreadPoolToken*> queue_;

  std::unordered_set<Thread*> threads_;
  
  boost::intrusive::list<IdleThread> idle_threads_; // NOLINT(build/include_what_you_use)

  // ExecutionMode::CONCURRENT token used by the pool for tokenless submission.
  std::unique_ptr<ThreadPoolToken> tokenless_;

  // Metrics for the entire thread pool.
  const ThreadPoolMetrics metrics_;

  const char* queue_time_trace_metric_name_;
  const char* run_wall_time_trace_metric_name_;
  const char* run_cpu_time_trace_metric_name_;
```

>> 问题： `NOLINT`是什么意思？

### `private DispatchThread()` -- 子线程的入口

**什么是permanent线程**  

因为“线程池”中最少有`min_thread_`个数的线程。所以对于该线程池中，最早创建的线程，都是“永久的”("permanent")。

“permanent”的线程的特点是：只要被创建出来，就永远不会被销毁。所以在处于idle状态时，就会一直阻塞等待task的到来。

而非永久的线程，当处于idle状态的时间，超过了`idle_timeout_`，那么就会被销毁。

>> spurious wake-up: 虚假的唤醒；  
>> investigation: 调研；  
>> apparently: 表面上; 似乎； 

**对于非永久线程，在因为超时而被唤醒时，要重新检查队列是否为空**  

在kudu的注释说：在返回`ETIMEOUT`时有一些奇怪的行为。

看起来是这样的：当出现超时而被唤醒时，有一小段时间，可能会有其它线程获取到了这个（保护这个条件变量的）锁，然后会发出信号（因为添加了task），然后释放锁。这时，等待的地方（即调用`waitfor`超时的地方）才会重新获得锁。

对于上面这种情况，因为已经有task到来了，该线程就不需要被销毁了。所以，在被唤醒以后（这时会加上锁），要重新检查下队列是否为空。如果仍然为空，那么就销毁线程。

**什么时候触发进行线程销毁**  
注意：只有“非永久”线程，才会被销毁。

当一个线程处于空闲状态的时间，超过`idle_timeout_`。

实现方式：当线程循环时，如果整个“线程池”没有待做的任务，那么就在“条件变量”上wait。这里wait是有超时时间的，超时时间为`idle_timeout_`。如果`wait()`返回超时错误，那么就说明该线程应该被销毁了。

**如何销毁线程**  
因为对于一个线程，执行完毕后，就会自动被销毁。所以对于要销毁的线程，只要退出“工作的循环”，就会被销毁。

在退出循环外，还会进行资源清理的操作，才会真正退出`DispatchThread()`函数。

**Task对象的析构要放在锁外**

参见commit: 2840a586d57012fc4bd805f6aca810e4f48dd37b

因为当前`ThreadPool`的实现中，所有地方都依赖同一把锁进行保护。

另外，有些Task的析构本身是比较重的。

还有一种情况是：有些Task相关的类，会在析构函数中获取锁（这虽然不是很好的编程实践，但确实存在这种情况）。如果这些类在析构函数中，也会向“线程池”提交task，那么就会产生死锁（因为析构未完成，所以锁还没释放，而提交task又需要加锁）。

**执行外一个task后，要设置对应token的状态**

因为自身正在执行一个token的task，所以这个token的状态一定不会是idle。

检查条件：如果“当前task”是“该token最后一个正在运行的task”。

1. 如果token当前状态为`QUIESCING`，说明该token已经被关闭，这个token不会再接受新的任务。因为当前task执行完毕后，该token就可以彻底关闭了。所以要将token的状态修改`QUIESCED`。
2. 否则，运行到这里，说明token的状态一定为`RUNNING`
    + 2.1 如果该token的task队列为空，就将它的状态修改为`IDLE`；
    + 2.2 否则（即，还有正在等待的task）：
      - 2.2.1 如果该token的提交模式是`SERIAL`：因为该模式下的作业是串行执行的。因为这时还有等待的task，所以这时要将当前token加入到“线程池”类的token队列中。
      - 2.2.2 当前token的提交模式是`CONCURRENT`：啥也不需要做。 因为对于`CONCURRENT`模式下提交的task，**在提交task的时候，已经将当前token添加到“线程池”的列表中了，所以这里不需要再次添加**。

**流程：**  
1. 加锁
2. 将自己添加到`threads_`中。增加`num_threads_`，并减小`num_threads_pending_start_`；
3. 检查自己的`permanent`属性。
4. 开启循环，等待task的到来
5. 在循环退出后，该线程即将被销魂

循环内的流程：
1. 检查“线程池”的状态，如果线程池被关闭了，就退出循环。
2. 如果`queue_`为空（即当前没有待做的task），那么当前线程进入idle状态。等待最长`idle_timeout_`的时间长度，如果没有新的task到来，那么就退出循环（最终会销毁线程）。
3. （走到这里，说明`queue_`不为空，即有task需要去做）从`queue_`的首部取出一个token，并从该token内部的task队列中取出一个task。
4. **临时解锁**；
5. 更新metric：task在队列中等待时间；
6. 执行task，会记录响应的walltime和cputime。
7. 析构task.
8. **重新加锁**；
9. 设置state的状态。并根据需要（`SERIAL`模式），添加相应的task。

循环结束后的流程（线程即将结束）：  
1. 减小中的线程数，即`num_thread_--`；
2. 如果当前没有线程，那么就发出通知（因为`Shutdown()`会等待所有线程退出）；

### 提交task

提供了几个重载的`Submit()`方法。

底层都是通过调用`DoSubmit()`来实现的。

#### `private DoSumbit()`方法

**如果不清理现有线程，那么一段时间后线程池中的总线程数**

等于： `“当前存在的线程数” + “正在创建的线程数”`。

即`num_threads_ + num_threads_pending_start_`。 

因为“线程池”最有`max_threads_`个线程，所以如下不等式一定要成立：  
`num_threads_ + num_threads_pending_start_ <= max_threads_`

**线程池最大的容量(最多同时提交的task数)**  
等于 `“最大的线程数” + “队列的最大长度”`。 即`max_threads_ + max_queue_size_`。

**当前剩余容量（即剩余的task座位数、还可以提交多少个task）**  
等于 `“线程池最大容量” - “正在运行的线程数” - 正在排队的task数`。

即`(max_threads_ + max_queue_size_) - active_threads_ - total_queued_tasks_`

注意：因为`max_queue_size_`的默认值是`INT_MAX`。因为它和任何一个整数相加，都会造成`int32_t`类型的溢出。所以在计算上面的等式时，应该写为:  
`(static_cast<int64_t>(max_threads_) + max_queue_size_) - active_threads_ - total_queued_tasks_`。 即将表达式中的第1个元素，强制转换为`int64_t`类型，从而避免溢出。

**当前处于“非活跃”状态的线程数**  
“非活跃”：只要一个线程属于该“线程池”，但是却没有“正在处理task”，那么久认为是“非活跃”的。 包括：正在创建过程中的线程，和 已经创建的但处于idle状态的。

等于 `“当前总的线程数” + “正在启动的线程数” - “正在处理task的线程数”`。

即 `num_threads_ + num_threads_pending_start_ - active_threads_`.

**是否需要新创建一个线程来处理‘新到来的task’**  
主要取决于“当前task的总数”、“当前的线程数”。

这里有个假设：**如果处于`inactive`的线程，都会最终取到一个task。**  
即如果当前有4个待执行的task，但是有5个`inactive`的线程，那么一定不需要创建新线程。（即，这些`inactive`的线程，很快就会去获取task并进行处理，数量已经足够，所以不需要再去创建新线程）。  

注意：如果‘当前task’是在`SERIAL`模式下的，那么当前task不会让需要的线程数增加1个。

原因是：
1. 对于“非`SERIAL`模式”提交的task，都是要并发执行的，所以在当前没有剩余线程的时候，是需要创建新线程去处理的。
2. 对于`SERIAL`模式下提交的task，因为是要串行执行的，所以实际上线程池中只需要存在1个线程，就可以满足`SERIAL`模式。而实际上，任何一个正确设置的线程池，线程数一定是大于1的，所以这里如果是`SERIAL`模式时，当前task不会让需要的线程数加1.

等于： `“之前提交的task总数” + “当前task是否应该被计入（值为1或0）” - “非活跃的线程数”`

**线程要在“锁”外创建**
因为“线程创建过程”可能会非常耗时（在某些情况下，可能需要几百毫秒）。

**lazy方式创建线程**  

另外，即使需要创建新线程，kudu是采用lazy的方式。即 在需要创建新线程的时候，kudu会向将当前task压入到队列中，然后由线程起来后自己去检查队内是否仍然有需要做的事情。而不是，直接创建一个线程，直接去做当前task。

注意：对于这种“lazy创建线程”的方式，可能会创建多余的线程。  
因为，在理论上，当前正在处理task的线程，可能很快就结束了（也就可以开始处理当前task），而且这时，新的线程可能还在创建过程中。 因为task已经被处理了，所以这个新创建的线程实际上是没有用到的（即创建了一个实际上不需要的线程）。   
但是，这种情况没法避免，而是是无害的。

当然，无论怎样，创建的线程总数量都不会超过`max_threads_`。

**task队列的组织形式**  

整体上task队列的组织，分为两个层级：

1. 第1级：在`ThreadPool`中，保持着有提交task的“`token`队列”，FIFO顺序。
2. 第2级：在每个`Token`内部，维护着该token所有已提交的task队列。也是FIFO顺序。

“工作线程”会不断的检查“线程池的 token队列”，如果不为空，那么就将队列首部的token 移除队列 ，然后从对应的token内部的“task队列”上取出task进行处理。

对于“`token-less`方式”和“token在`CONCURRENT`模式”下提交的task，会进行如下操作：
1. 将task放到“token自己的队列”中；
2. 将自己（token）放到“线程池”的“token队列”中。

如果是“token在`SERIAL`模式”下提交的task：
1. 将task放在“token自己的队列”中；
2. 只有在token的状态为`IDLE`的时候，才会将自己（token）放到“线程池”的“token队列”中。 

另外，无论以任何方式提交，只要token当前的状态为`IDLE`，这时还需要把token的状态修改为`RUNNING`。

说明：对于`SERIAL`模式的token，在工作线程执行它的task结束的时候，也会进行检查。如果发现“该token自己的队列”上仍然有排队的task，那么会将token放入到“线程池”的“token队列”中，从而让空闲的工作线程能够检查到，并进行处理。

```
----------token1 ----- token2 ----- token1 ----- xxxx ----
            |           |            /
            | <---------+-----------/       
            |           |
            v           v              
       +--------+    +--------+    
       | task_2 |    | task_1 |
       | task_3 |    | task_4 |
       | task_5 |    |  ...   |
       |  ...   |    +--------+
       +--------+
        (token1)      (token2)
```

**工作线程以`LIFO`的顺序被唤醒**

传统的线程池，都是在内部简单维护一个队列，然后有一个“条件变量”（记为`not_empty_cv`）来表示当前队列是否为空：
1. 提交task：就是向队列中加入一个元素，然后向`not_empty_cv`发出信号(`notify_one()`)；
2. 工作线程：就是不断检查队列的容量，如果为空，那么都阻塞等待`not_empty_cv`发出通知。

这种方式的特点是：所有线程都会wait在同一个“条件变量”上（在上面例子中，都wait在`not_empty_cv`上）。 这种实现方式，实际上会产生一个`FIFO`的顺序。也就是说：“谁先进行wait，谁就先会被唤醒”。 这种算法是比较公平的。

但是，如果使用“线程池”的环境中，task都是非常轻量的，这时如果希望使用尽量少的线程，来完成那些非常短的工作。那么`FIFO`的工作方式就做不到了。 因为`FIFO`方式会通知到所有的线程。  

比如一个轻量task的流（比如 RPC），这个使用应该只会使用1个线程，或者接近1个线程就可以完成。 而不需要所有线程都参与。

参见kudu commit: bec5a200c0b7e5349a488a71985547820feebbdc 

还要补充一点：  
在这个提交中，多个处于idle状态的工作线程，不再是wait在同一个“全局的条件变量”上，而是每个线程都wait在自己的条件变量上（当然，用来保护的锁还是同一个）。 在新的task到达、进行通知的时候，也是只通知1个线程（因为通知是通过条件变量实现的，只需要先拿到一个线程的条件变量，然后发出的通知，就只有这一个线程能够收到）。

>> 问题：“所有线程wait在同一个条件变量”，与“所有线程都wait在不同的条件变量”上，有性能差异吗？ 注意：这两种场景，用来保护的锁都是同一个。

**处理流程：**    
1. 加锁；
2. 检查“线程池”的状态，是否有错误（包括被关闭了）；
3. 相应的`Token`的状态，是否可以提交作业；
4. 容量检查：当前线程必须还有剩余容量，当前task才能被提交；
5. 判断是否需要创建新线程；
6. 封装当前task，并根据当前的提交方式，放入到对应的队列中；
7. 如果有空闲的thread(空闲线程队列 不为空)，那么从中取出来一个线程，并激活该线程。
8. 解锁；
9. 更新metric
10. 根据前面的检查，如果需要创建新线程，那么在这里创建。
11. 结束；

### `private CheckNotPoolThreadUnlocked()`

调用`Shutdown()`, `Wait()`方法的线程，一定不能是 “线程池”内部的线程。否则就会产生死锁。

因为，因为这些方法实际上是要在内部阻塞等待 所有task都执行结束，才会返回。 “线程池”内部的工作线程，如果调用这些方法，就会陷入这些阻塞，导致工作线程一直不能结束。 而因为工作线程阻塞，导致`Shutdown()`等永远不会结束，最终出现死锁。

所以在调用这些方法的地方，都要执行`CheckNotPoolThreadUnlocked()`进行检查。

## `class ThreadPoolToken`

所有"token-based"方式提交的入口类。

可以提交作业，也可以阻塞等待特定的事件。

**注意1:** 只有一种方式可以创建token对象，即通过`ThreadPool::NewToken()`方法。

**注意2：** 所有的方法都是线程安全的。都是由`ThreadPool::lock_`来进行保护。

**注意3：** 所有的`ThreadPoolToken`对象，都必须在`ThreadPool`对象析构之前进行析构。

### 成员

#### `mode_`成员

该属性指定，当前`Token`对象提交的多个作业，如何执行。

是一次只能执行1个，还是可以多个并发执行。

在构建`Token`对象时，就要指定该属性。

#### `metrics_`成员

当前`Token`对象所对应的`metric`信息。

#### `pool_`成员

指向当前所属的“线程池”对象。

#### `state_`成员

当前`Token`的状态。

#### `entries_`成员

当前`Token`提交的作业列表。

所有提交的作业，都会先存放到这里。工作线程在执行的时候，才会把它该队内中取出。

#### `not_running_cond_`成员

表示当前`Token`没有正在运行的作业。当`Token`的状态变为 `IDLE`或`QUIESCED`时，会发出信号。

#### `active_threads_`成员

由当前token提交的、正在运行的task数量。

```
  const ThreadPool::ExecutionMode mode_;

  const ThreadPoolMetrics metrics_;

  ThreadPool* pool_;

  State state_;

  // Queued client tasks.
  std::deque<ThreadPool::Task> entries_;

  // Condition variable for "token is idle". Waiters wake up when the token
  // transitions to IDLE or QUIESCED.
  ConditionVariable not_running_cond_;

  // Number of worker threads currently executing tasks belonging to this
  // token.
  int active_threads_;
```

### 方法列表



































