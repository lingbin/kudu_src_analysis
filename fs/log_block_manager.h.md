[TOC]

文件：`src/kudu/fs/log_block_manager.h`

# `LogBlockManager`

## 概述

“基于log”的`block storage`。 

> 说明： “基于log”的含义是：顺序分配文件。

当前实现的主要目的：  
减少总的文件数。方法是：将多个`block`放到一个“大文件”中。(即这个“大文件”，会作为一个“容器”) 。

一个“容器”，在开始的时候，是一个空文件，所有的数据，都是顺序写入的。（即逐个将多个`Block`写入进去）。

如果一个“容器”写满了，那么它会被放在旁边，然后新建一个“容器”继续接受写入。

### 实现细节

一个“容器”中，包含两个文件：
1. “元数据”文件；
2. 数据文件；

注意：这两个文件，都只能“顺序写入”。

**说明：写入流程是这样的：**  
1. 将一个`Block`写入到“数据文件”中；
2. 对“数据文件”进行`fsync`，然后在“元数据文件”中添加一条记录（其中包含这个`Block`的`block_id`和“它在数据文件中的‘位置信息’”）。

>> as-is: 照原来的样子；不予改变的；  

**说明：删除一个`Block`的流程：**  
和写入是类似的。
1. 在“元数据文件”中写入一条记录（描述本次的删除动作）。

注意：因为要删除的`Block`的数据，在“容器”文件中的一部分，所以不能立即被删除。

也就是说，这个被删除的`Block`的数据，在“数据文件”中就成为了“孤儿”。

**说明：这个被删除的`Block`的数据，所占用的空间，如何被立即释放：**  
有两种方式：
1. 通过`hole punching`：这种方式可以做到立即删除；
2. 通过后期的“垃圾清理”流程：  
   + 在不支持`hole punching`的文件系统上，会使用该方式。
   + 或者在进程启动过程中。因为之前的进程退出，可能是因为“在删除之后，在进行`hole punching`之前”，进程崩溃了；
   
**说明：“元数据文件”不会进行压缩**  
“元数据文件”不会进行压缩，因为期望是：即使在进行了非常多的`create/delete`操作周期后，仍然希望“文数据文件”比较小。

>> “不压缩” 和 “希望保持较小”有啥关系？

**说明：对于“数据文件”和“元数据文件”的操作，是经过小心的安排顺序的**，从来保证在任何时候，在磁盘上结构都是正确的。

1. 在一个`WritableBlock`的生命周期内（就是说，新创建了一个`Block`），所有的操作都是：先操作“数据文件”，后操作“元数据文件”。  
    如果一个“元数据文件”的操作失败了：那么该操作的结果就是：在“数据文件”中，遗留了一个“孤儿”的`Block`。（在下一轮的“垃圾回收”时，会被检测到、并被删除掉）。
2. 在删除一个`Block`的时候：与上面相反，操作顺序是：先操作“元数据文件”，后操作“数据文件”。  
    这时，如果第2步执行失败（即在“数据文件”中删除`Block`失败了），那么最坏的情况就是：产生了垃圾数据，在“垃圾回收”时会被清理。

> 备注： 一个`Block`，如果**在“元数据”中不存在，但是在“数据文件”中存在**，那么这个“Block”在“数据文件”中的数据，最终会在“垃圾回收”的时候被清理掉。

> 说明: 因为“操作数据文件”和“操作元数据文件”，是两个操作，是没法做到原子的。  
>    无论谁先随后，两个操作都有失败的可能。   
>    但是这里只关心的是：如果只有一个成功，那么“残留的部分修改”，应该如何处理。就像“事务”一样，如何回滚，要做到最终像“两个操作”都没有进行一样。
>
> 1. 如果是第一个操作失败，那么因为“第2个”操作尚未执行，所以就不会有“残留部分修改”的问题。
> 2. 所以，上面讨论的，都是“第一个操作成功了，但是第2个操作失败了”的场景下，如何进行处理的。

**说明：`LogBlockManager`的内存结构也是经过经过仔细处理的，为的是能够和“磁盘上的持久化的内容”保持一致**  
>> to wit: 和"that is to say"是相同意思；

对于一个`Block`来说，只有它的所有“磁盘操作”都是成功的，这个在内存数据结构中的`Block`才是可用的。

这里的“磁盘操作”，是包含所有的“`fsync()`”操作的.

**说明：向一个“容器”中写数据，是使用`block transaction`来打包在一起的**。

在写数据的时候，每个`writer`会拥有一个“可用的‘容器’”，然后将`Block`的内容写入到“容器”中，最后（在写完数据后，即`finalize`的时候）释放这个“容器”。

一个容器，只有在被上一个`Block`释放之后，才能被其它的`Block`使用，用来写数据。

**注意：这个是会并发的。**  

也就是说，多个事务 是可以 “交错的” 向一个“容器”中写入数据的。只要上一个`Block`进行了`Finalize`，那么下一个`Block`就可以拥有这个“容器”。

> 备注：一个需要写数据的线程，称为一个`writer`。 （这里写的数据内容是`Block`的内容）

一旦其中一个`writer`彻底的结束了IO操作，那么它可以提交这个“事务”，会将“`Block`的数据”和对应的“容器”，`fsync`到磁盘。注意在这个事务提交的过程中，是可能有其它`writer`正在并发写入数据的。

**说明：为了保证在“磁盘上的数据”是处于一致状态的，如果事务提交失败，那么整个“容器”就会被标记为`read-only`，后续不可以在向该“容器”进行写入**  

这里有一个“权衡”：虽然允许并发的写入，可以获得更好的“容器利用率”。  
但是如果任何一个`writer`在`sync`时发生错误，会导致其它的`writer`也会失败，并且潜在地可能会让底层的“容器”崩溃。

**说明：在创建一个`Block`的时候，会从`DataDirGroup`中为这个`Block`选择一个“容器”**  
注意：这里在`CreateBlockOptions`中可以提供`hint`。比如说：对于`DiskRowset`的`Block`，应该放在它所属的`tablet` 所对应的`DataDirGroup`中。

所有的`LogBlockManager`的“元数据”请求，都是只需要“内存”结构就可以服务。  
所以在`Open`一个已有的`LogBlockManager`，所有的在磁盘上的“容器”会被解析，从而在内存中构建一个`map`，去描述所有`Block`的“是否存在”和“具体的位置”。  

在这个`map`中的每一个条目，大于是`64`字节。所以如果有`1千万`个`Block`，大于占用的内存是`610MB`。

>> 备注：filesystem block: 文件系统的块。在文件系统中，每个块有固定的大小。  
>> round up: 向上舍入。 注意和“四舍五入”不同，"round up"是全部都“向上舍入”。

**说明：所有的`Block`都要去适配“文件系统的块大小”，并且一个`hole punch request`的大小，也要求“向上舍入”为“最接近的“‘文件系统块大小’的整数倍”**   

结合这两点，可以做到：进行一次`hole punching`，是可以真正的 清理掉磁盘空间的。 而不是将磁盘上的所有数据，用`\0`来填充。

### 设计的取舍

**概括的说，`LogBlockManager`是为了持续地“读取”和“写入”来进行优化的。**  

这里的逻辑是这样的：在一个“容器”中的所有的`Block`，都是包含数据的，并且是被立即读取的，这样可以减少后续扫描请求的`seek`操作。  

但是这会带来一个开销：需要经常对“容器”进行“垃圾回收”。  

这里的一个优化：在较新版本的“文件系统”，是可以利用“文件系统的`hole punching`”去 回收空间。

**说明：“元数据文件”在磁盘上的存储，有几个考虑：**  
1. 使用简单的实现；
2. 持续的访问“空间占用”；
3. 可以轻松扩展到 非常大的`Block`数量；

>> manageability: 容易处理；容易管理；  
>> sustained: 持续的；  

更详细的说：将“元数据”和“数据”分开来存储，可以做到允许在`open LogBlockManager`的时候，可以高效的读取“元数据”。

一个“容器”不是一个单一的文件，并且为了使用这个“容器”，需要打开多个“文件描述符”。

而且，因为“元数据”是基于“日志”的，在“打开”的时候（即进程启动，要加载一个已有的`LogBlockManager`），是比较简单和高效的。（因为只需要“回放日志”即可）。

同样地，在为“容器”选择位置时，也是更喜欢“保持简单”，超过“性能”。  

>> pertain: 适合；使用；  
>> colacated: 

将来，可以使用“位置的`hint`”来保证“相关的数据”会被放在一起，从而提高查询的性能。

**说明：为什么所有的“元数据请求”都来自“内存”？**  
是为了“简单”，而牺牲了“内存空间占用”。  

>> balloon: 气球膨胀   

将来，如果有非常大的`Block`数量，这个在内存中的`map`，所占用的内存空间，可能会膨胀的很大。这时，可以再添加一些“落盘”的处理逻辑。




# `struct LogBlockManagerMetrics`

用来将多个`metric`指标封装一个类中。

>> agnostic: 不可知论的；  
>> implementation-agnostic: 与实现无关的；  （这里的“与实现无关的”，是指所有的`BlockManager`都会有的指标，即`BlockManagerMetrics`）

说明：这里的`metric`包含两类：
1. 一些与具体实现无关的；
2. 针对`LogBlockManager`所特定的；

```
struct LogBlockManagerMetrics {
  explicit LogBlockManagerMetrics(const scoped_refptr<MetricEntity>& metric_entity);

  // Implementation-agnostic metrics.
  BlockManagerMetrics generic_metrics;

  scoped_refptr<AtomicGauge<uint64_t>> bytes_under_management;
  scoped_refptr<AtomicGauge<uint64_t>> blocks_under_management;

  scoped_refptr<AtomicGauge<uint64_t>> containers;
  scoped_refptr<AtomicGauge<uint64_t>> full_containers;

  scoped_refptr<Counter> holes_punched;
  scoped_refptr<Counter> dead_containers_deleted;
};

#define MINIT(x) x(METRIC_log_block_manager_##x.Instantiate(metric_entity))
#define GINIT(x) x(METRIC_log_block_manager_##x.Instantiate(metric_entity, 0))
LogBlockManagerMetrics::LogBlockManagerMetrics(const scoped_refptr<MetricEntity>& metric_entity)
  : generic_metrics(metric_entity),
    GINIT(bytes_under_management),
    GINIT(blocks_under_management),
    GINIT(containers),
    GINIT(full_containers),
    MINIT(holes_punched),
    MINIT(dead_containers_deleted) {
}
#undef GINIT
```

# `class LogBlock`

在`.cpp`文件中定义。

用来描述“逻辑的`Block`”的“元信息”。

在将`Block`的数据，刷入到磁盘以后，会生成一个`LogBlock`结构。 

因为在`Block`的数据输入磁盘以后，这个`Block`对应的“元信息”就彻底的不会再改变了，并且这时“这个`Block`的数据”已经是被持久化、并且可被读取了。  

`LogBlock`类是继承自`RefCountedThreadSafe<>`，即是可以被“引用计数”的。 这样，可以简化在读取时的删除逻辑。  

注意：在所有的“增加”和“减少”引用计数的操作，都需要先申请`block manager`锁。

但是，在`~LogReadableBlock()`中，在“减少”引用计数的时候，是没有加锁的。所以这里必须继承`RefCountedThreadSafe<>`，而不是`RefCounted<>`。


























