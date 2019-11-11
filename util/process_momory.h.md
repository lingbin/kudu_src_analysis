[TOC]

文件： `src/kudu/util/process_memory.h`

## 概述

1. 该文件定义了几个GFALG变量，用来指定相关的内存限制。
2. 该文件在`kudu::process_memory`名称空间下，定义了几个全局函数，用来进行内存占用信息的判断。

**如何统计内存信息：**  
有两种方式，在编译的时候就会决定。实现见`CurrentConsumption()`方法。
1. 使用`tcmalloc`进行统计（开启`TCMALLOC_ENABLED`宏）。

## 相关的`GFLAG`

  gflags               | 默认值 |说明
---------------------- |------  | --
FLAGS_memory_limit_hard_bytes                | 0  | 硬限。如果值为`0`，那么默认是机器总内存的80%。如果值为`-1`，那么表示没有限制；
FLAGS_memory_pressure_percentage             | 60 | 当内存使用超过这个值，那么会提升`flush`操作的优先级。<br/> 默认是“硬限”的`60%`。
FLAGS_memory_limit_soft_percentage           | 80 | 软限。默认是“硬限”的`80%`。<br/> 超过的越多，那么写请求被拒绝的概率越高。 <br/>  通常情况下，降低该值，可以让 写操作的延迟更线性，但是吞吐会下降； 反之亦然（增大该值，写操作的延迟可能不再线性，吞吐会提高）
FLAGS_memory_limit_warn_threshold_percentage | 98 | 超过这个值，那么会周期的打印“内存警告”的日志。 默认是“硬限”的`98%`。
FLAGS_tcmalloc_max_free_bytes_percentage     | 10 | 在使用`tcmalloc`时（预设了`TCMALLOC_ENABLED`宏）使用。  <br/>  在系统的`RSS`中，允许`tcmalloc`占用但不进行回收的最大空闲值。 

说明：`tcmalloc`中释放的内存，不会理解返回给系统。

注意1：只有“硬限”是相对于“机器内存”的，其余都是相对于“硬限”的。

注意2： 在设置的时候，内存大小的“压力值”、“软限”，“警告值”，“硬限”，是要主键递增的。
```
FLAGS_memory_pressure_percentage 
< FLAGS_memory_limit_soft_percentage 
< FLAGS_memory_limit_warn_threshold_percentage 
< FLAGS_memory_limit_hard_bytes
```

## 几个全局变量（在“匿名名称空间”中）

在该文件中，使用了“匿名命名空间”，在其中声明了一些“全局变量”（因为是“匿名命名空间”，所以**这些“全局变量”的作用域只在当前`cpp`文件内有效**）

```
namespace {
int64_t g_hard_limit;
int64_t g_soft_limit;
int64_t g_pressure_threshold;

ThreadSafeRandom* g_rand = nullptr;

}
```

## 初始化函数 `DoInitLimit()`

这个函数不是公开的(定义在`cpp`文件中的匿名名称空间内，即作用域只限于当前`cpp`文件)。

在该文件提供的`public全局方法`中，每个方法都会调用`InitLimits()`，其中使用了 `GoogleOnceInit()` 接口，用来保证`DoInitLimit()`方法只会被调用一次，即这些“全局变量”仅会被初始化一次。

## `CurrentConsumption()`方法

获取当前已经消耗的内存；

有两种方法：
1. 使用`tcmalloc`时，那么通过`tcmalloc`的接口来获取；
2. 如果没有使用`tcmalloc`，那么只能通过 根节点的`MemTracker`来获取。

注意：在开启了`TCMALLOC_ENABLED`宏的场景下，该函数会定义几个`static`变量。因为是`static`变量，所以多个调用该方法的线程，可以通过这些static变量相互通信。

其中有一个`read_lock`的static变量，使用它可以在多个线程之间进行同步。

```
int64_t CurrentConsumption() {
#ifdef TCMALLOC_ENABLED
  const int64_t kReadIntervalMicros = 50000;
  static Atomic64 last_read_time = 0;
  static simple_spinlock read_lock;
  static Atomic64 consumption = 0;
  uint64_t time = GetMonoTimeMicros();
  if (time > last_read_time + kReadIntervalMicros && read_lock.try_lock()) {
    base::subtle::NoBarrier_Store(&consumption, GetTCMallocCurrentAllocatedBytes());
    // Re-fetch the time after getting the consumption. This way, in case fetching
    // consumption is extremely slow for some reason (eg due to lots of contention
    // in tcmalloc) we at least ensure that we wait at least another full interval
    // before fetching the information again.
    time = GetMonoTimeMicros();
    base::subtle::NoBarrier_Store(&last_read_time, time);
    read_lock.unlock();
  }

  return base::subtle::NoBarrier_Load(&consumption);
#else
  // Without tcmalloc, we have no reliable way of determining our own heap
  // size (e.g. mallinfo doesn't work in ASAN builds). So, we'll fall back
  // to just looking at the sum of our tracked memory.
  return MemTracker::GetRootTracker()->consumption();
#endif
}
```

说明1：在真正调用`tcmalloc`的接口函数（去获取当前已经分配的总内存大小）之前，使用`read_lock`加锁，从而避免其它线程并发获取。

说明2：**这里的实现并不能保证，每次都获取到最新的信息**。`read_lock`加锁，使用的是`try_lock()`，也就是说，如果有其它线程正在获取的情况下（其它线程已经加上锁，正在获取中），那么当前线程`try_lock()`会失败，那么本次不进行真正的获取，而是直接返回`consumption()`的历史值。

说明3：因为有可能特定条件下，`tcmalloc`的接口函数可能会很慢。所以在锁内，会提升下`last_read_time`，从而尽最大可能的希望，会经过一个等待周期，才会进行下一次从`tcmalloc`获取内存占用信息。

说明4: **这里的实现也不能保证，两次使用`tcmalloc`的接口的间隔，一定会大于 `kReadIntervalMicros`（等待周期）**。只要上一次调用`tcmalloc`接口的线程，在更新`last_read_time`之前，下一个线程已经通过了`last_read_time`的检查，那么是可能会连续调用 `tcmalloc`的接口的。

```
    为方便起见，都以“秒”为单位，假定 read_interval=1，last_read_time=0
    
        thread_1                          thread_2
                                                    
                                        get current_time (假定为5)
                                                               
                                        check last_read_time
                                            （这时last_read_time=0，
                                              大于一个周期，进行获取）
                                             
                                             
                                        try_lock()
                                          
                                        invoke tcmalloc API
                                             （假定这一步非常慢，需要耗时5秒）
                                            
                                                                
    get current_time (为10)
    
    check last_read_time(满足)
       （这时last_read_time仍然为 0, 
       大于一个周期，也要进行获取）
                                          
                                        释放read_lock
                                        
                                        重新获取current_time，
                                        并update last_read_time（更新为5）
    
    try_lock()
    
    invoke tcmalloc API
    
    重新获取current_time，
    并update last_read_time
```

从上面的场景看， 第一次调用`tcmalloc`的接口时间区间是`[5, 10)`，第二次调用的开始时间是`10`，所以两次调用之间实际上是连续的，并没有保证一定会经过一个`kReadIntervalMicros`周期。


## `UnderMemoryPressure()`方法

检查当前内存使用量，是否超过了“警告值”(`g_pressure_threshold`)。

+ 如果没有达到阈值，那么返回`false`；
+ 否则（已经达到阈值），那么返回`true`，并且在参数（传入的指针）中，返回当前的使用百分比（相对于“硬限”）。

```
bool UnderMemoryPressure(double* current_capacity_pct) {
  InitLimits();
  int64_t consumption = CurrentConsumption();
  if (consumption < g_pressure_threshold) {
    return false;
  }
  if (current_capacity_pct) {
    *current_capacity_pct = static_cast<double>(consumption) / g_hard_limit * 100;
  }
  return true;
}
```

## `SoftLimitExceeded()`方法

检查是否超过了内存的“软限”(`g_soft_limit`)

如果超过了“软限”，就返回`true`；否则，返回false。

在超过“软限”的情况下，通过传入的参数，可以获取到 “当前内存使用量” 占“硬限”的百分比。

说明：
1. 如果`g_soft_limit == g_hard_limit`，那么说明当前并没有单独设置“软限”（即`FLAGS_memory_limit_soft_percentage == 100`），那么当前使用的内存重量，只要没有超过“硬限”的值，那么就认为没有超过“软限”。
2. 如果已经超过了“软限”，但尚未超过“硬限”：那么**使用“随机算法”**来决定是否允许执行当前请求。

**这里的随机方法，要做到：超过“软限”的越多，拒绝请求的概率越大。**  
（要做到这点很容易，比如说`g_hard_limit == 100; g_soft_limit=80`，只需要让随机算法在 `[0, 20]`内随机取值，这样在“当前内存”超过了`g_soft_limit`以后，只需要利用检查方法`current_consumption + random_value > g_hard_limit`，就可以有“当前内存 越大，该判断成立的可能性越大”的效果。因为`random_value`是完全随机的，当`current_consumption`无限接近`g_hard_limit`，那么只需要`random_value`为几乎任意值时，都会使该判断成立）

注意：在超过“软限”以后，返回的“百分比”，是相对于“硬限”的比例，不是相对于“机器的内存”。

```
bool SoftLimitExceeded(double* current_capacity_pct) {
  InitLimits();
  int64_t consumption = CurrentConsumption();
  // Did we exceed the actual limit?
  if (consumption > g_hard_limit) {
    if (current_capacity_pct) {
      *current_capacity_pct = static_cast<double>(consumption) / g_hard_limit * 100;
    }
    return true;
  }

  // No soft limit defined.
  if (g_hard_limit == g_soft_limit) {
    return false;
  }

  // Are we under the soft limit threshold?
  if (consumption < g_soft_limit) {
    return false;
  }

  // We're over the threshold; were we randomly chosen to be over the soft limit?
  if (consumption + g_rand->Uniform64(g_hard_limit - g_soft_limit) > g_hard_limit) {
    if (current_capacity_pct) {
      *current_capacity_pct = static_cast<double>(consumption) / g_hard_limit * 100;
    }
    return true;
  }
  return false;
}
```

## `tcmalloc`相关函数

### 相关全局变量（也是匿名名称空间）

```
namespace {

#ifdef TCMALLOC_ENABLED
// Total amount of memory released since the last GC. If this
// is greater than GC_RELEASE_SIZE, this will trigger a tcmalloc gc.
Atomic64 g_released_memory_since_gc;

// Size, in bytes, that is considered a large value for Release() (or Consume() with
// a negative value). If tcmalloc is used, this can trigger it to GC.
// A higher value will make us call into tcmalloc less often (and therefore more
// efficient). A lower value will mean our memory overhead is lower.
// TODO(todd): this is a stopgap.
const int64_t kGcReleaseSize = 128 * 1024L * 1024L;

#endif // TCMALLOC_ENABLED

} // anonymous namespace
```

`g_released_memory_since_gc`表示从上次调用`tcmalloc`的垃圾回收函数(`MallocExtension::instance()->ReleaseToSystem(int num)`)之后，已经累计释放的内存总量。

如果累积释放的内存数，超过了`kGcReleaseSize`(默认值是`128MB`)，那么就会触发`GcTcmalloc()`函数。

### `GetTCMallocCurrentAllocatedBytes()`方法

获取当前`tcmalloc`中已经分配的内存中大小。

```
int64_t GetTCMallocCurrentAllocatedBytes() {
  return GetTCMallocProperty("generic.current_allocated_bytes");
}
```

### `MaybeGCAfterRelease()`方法

每次调用`MemTracker::Release()`时，都会调用该方法。

>> 说明：如果使用了`tcmalloc`时，默认情况下，析构对象（`free`或`delete`)的内存是不会立即返回给系统的。

如果多次`Release()`的内存（或者使用负数调用`Consume()`）累积到一定阈值，那么就手动调用`tcmalloc`的内存回收。

当前的实现，这个阈值是写死的，为 `kGcReleaseSize = 128MB`。

```
void MaybeGCAfterRelease(int64_t released_bytes) {
#ifdef TCMALLOC_ENABLED
  int64_t now_released = base::subtle::NoBarrier_AtomicIncrement(
      &g_released_memory_since_gc, released_bytes);
  if (PREDICT_FALSE(now_released > kGcReleaseSize)) {
    base::subtle::NoBarrier_Store(&g_released_memory_since_gc, 0);
    GcTcmalloc();
  }
#endif
}
```

>> 说明：这里在`released_bytes`一定是个正数，但是在之前的代码中有个bug，累加`g_released_memory_since_gc`时在`released_bytes`前面有个负号，这就导致不是累加了。 向Kudu提了PR，已经被合入，PR链接为：：https://gerrit.cloudera.org/c/14244/  

### `GcTcmalloc()`方法

有两个地方会调用该函数：
1. 后台线程：定期检查tcmalloc的内存使用；(见`ServerBase::TcmallocMemoryGcThread()`)
2. 每次调用`MemTracker::Release()`都会被记录，当累计释放的内存超过一定阈值（当前是128MB），会强制调用一次该函数（参见：`MaybGCAfterRelease()`）。

**原理：**  
如果当前在`tcmalloc`内部的空闲列表的内存大小（在操作系统看来，是被进程占用的）超过一定阈值，那么会触发调用`tcmalloc`的内存回收函数，将内存返回给操作系统。

阈值通过`FLAGS_tcmalloc_max_free_bytes_percentage`指定，默认是10%。

`tcmalloc`中将内存返回给操作系统的接口是：`MallocExtension::instance()->ReleaseToSystem(int num)`

注意：在调用 `ReleaseToSystem()`时，如果一次性返回较大的内存，那么可能会加锁阻塞较长的时间，所以这里是每次只返回 1MB 的内存。

```
void GcTcmalloc() {
  TRACE_EVENT0("process", "GcTcmalloc");

  // Number of bytes in the 'NORMAL' free list (i.e reserved by tcmalloc but
  // not in use).
  int64_t bytes_overhead = GetTCMallocProperty("tcmalloc.pageheap_free_bytes");
  // Bytes allocated by the application.
  int64_t bytes_used = GetTCMallocCurrentAllocatedBytes();

  int64_t max_overhead = bytes_used * FLAGS_tcmalloc_max_free_bytes_percentage / 100.0;
  if (bytes_overhead > max_overhead) {
    int64_t extra = bytes_overhead - max_overhead;
    while (extra > 0) {
      // Release 1MB at a time, so that tcmalloc releases its page heap lock
      // allowing other threads to make progress. This still disrupts the current
      // thread, but is better than disrupting all.
      MallocExtension::instance()->ReleaseToSystem(1024 * 1024);
      extra -= 1024 * 1024;
    }
  }
}
```



