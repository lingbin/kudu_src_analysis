[TOC]

## 文件 - 见`util/memory/`目录

所有的内存 都是使用`BufferAllocator`来负责分配的。所以**是可以在应用程序级别，来控制内存分配的**。 

`BufferAllocator`类会实现一些管理逻辑（比如设置分配的内存上限）。

1个`BufferAllocator`对象可以被多个 线程使用；例如：对于一个用户的多个请求时，可以使用同一个 allocator对象，这样可以按照“用户级别”来对内存进行限制。



## `Buffer`类

封装一段内存，包含一个指针和一个长度。

```
class Buffer {
  void* data_;
  size_t size_;
  BufferAllocator* const allocator_;
}
```

>> 说明：这里有一个`allocator_`成员，是为了在析构当前`Buffer`对象时（在“析构函数”中），通知对应的`allocator_`。 

## `BufferAllocator`类

是一个基类，各个子类实现不同的分配方法。

基类中只提供了基本接口，包括3个方面

功能类型         | 接口  
------------     |---
分配内存         | `BestEffortAllocate()`
    -            | `Allocate()`
重新分配内存     | `BestEffortReAllocate()`
    -            | `ReAllocate()`
查看可用容量大小 | `Available()`


说明：
1. “申请内存”时，返回的是`Buffer`对象，而不是裸的`char*`或`uint8_t*`。
2. “释放内存”对应的函数是`free`，传递的参数也是`Buffer`对象。

注意：对于“释放内存(`FreeInternal()`)”是在`private`方法，所以

注意：在`BufferAllocator`中，将一些虚函数（例如：`AllocateInternal()`, `ReAllocateInternal()`, `FreeInternal()`）都设置为`private`的。 

这里涉及到一个`C++`的“最佳实践”：推荐一直将`virtual`函数设置为`private`。

1. 只要一个函数是“虚函数”，那么它就可以被子类重载，即使这个“虚函数”是`private`的。
2. 将一个 虚函数 设置为`private`，与`protected`相比，子类是不能直接调用这个`private`方法（如果声明为`protected`，那么子类可以直接调用）。
3. 但是一个函数之所以要被声明为`virtual`，那么就隐含着的意思是，子类要对这个函数进行定制。

一般情况，**只要确定“子类中 不需要 直接调用 父类的这个“虚函数”的实现”**，那么就应该在父类中把“虚函数”声明为`private`的。

在这个类中，另一个虚函数(`CreateBuffer()`)，因为子类需要直接调用，所以声明为`protected`。

参见：
1. https://stackoverflow.com/questions/2170688/private-virtual-method-in-c/35632273
2. http://www.gotw.ca/publications/mill18.htm  Virtuality  
3. https://isocpp.org/wiki/faq/strange-inheritance#private-virtuals  When should someone use private virtuals?  

### 关于指定内存大小的 注意事项

无论`requested`和`minimal`的大小是多少，都会返回一个有效的`Buffer`对象。其中通过`size`来表示是否有真正的内存空间。

1. 如果`reqeusted`为0，返回的Buffer对象中，内容是一个大小为0的“非空的”指针。
2. 如果`minimal`为0，返回的Buffer对象中，指针也是“非空的”，但是大小是否为0 取决于`requested`的大小，以及具体申请到的内存大小。

### `BestEffortReallocate()`方法

语义 类似与`realloc()`

```
void* realloc (void* ptr, size_t size);

```
1. `ptr` 指向原来的内存；size 是新的大小。
2. `size` 的值，与`ptr`的原大小没有关系，可以大于、等于，小于；
3. 函数的效果：重新申请`size`大小的内存，并且将`ptr`的数据拷贝到新申请的内存中，同时`ptr`指向的内存空间会被回收。
  - 3.1 如果`ptr`为`nullptr`，那么相当于`malloc()`，即申请`size`大小的内存。
  - 3.2 如果`size`大小为0，那么只会释放`ptr`所指向的内存，并不申请新内存，这时返回`nullptr`

注意：`ptr`必须指向动态申请的内存(即通过`malloc`申请的内存)

```
  Buffer* BestEffortReallocate(size_t requested,
                               size_t minimal,
                               Buffer* buffer) {
    DCHECK_LE(minimal, requested);
    Buffer* result;
    if (buffer == NULL) {
      result = AllocateInternal(requested, minimal, this);
      LogAllocation(requested, minimal, result);
      return result;
    } else {
      result = ReallocateInternal(requested, minimal, buffer, this) ?
          buffer : NULL;
      LogAllocation(requested, minimal, buffer);
      return result;
    }
  }
```

1. 如果参数中的`buffer`为`nullptr`，那么相当于申请一个`Buffer`，返回这个新申请的`Buffer`对象。  
2. 如果参数中的`buffer`不是`nullptr`，那么返回的就是参数中的`Buffer`对象。只不过该`Buffer`指向的‘内存地址’和‘大小’都变了。

在某些子类中，也可能会实现为`原地修改`。即在返回的`Buffer`对象中，仍然指向原来的内存地址。


## `HeapBufferAllocator`类

继承自`BufferAllocator`类。 

负责在“堆”上分配内存，没有总大小限制。 在内部使用`C`语言的相关方法(`malloc`, `realloc`, `free`)。

### `AllocateInternal()`方法

有两个参数来说明要申请的内存大小：`requested`和`minimal`。

当无法申请`requested`大小的内存时，会依次递减“一半”来重试。

```

-------------------- minimal ---------------------- requested
第1次：                -                               ^
第2次：                -               ^ (两者的一半处)
第3次：                -       ^ (再缩小一半)
...
```

这里的尝试步骤是：
1. 先尝试申请`requested`大小；
2. 如果申请失败，那么 尝试申请``

```
Buffer* HeapBufferAllocator::AllocateInternal(
    const size_t requested,
    const size_t minimal,
    BufferAllocator* const originator) {
  DCHECK_LE(minimal, requested);
  void* data;
  size_t attempted = requested;
  while (true) {
    data = (attempted == 0) ? &dummy_buffer[0] : Malloc(attempted);
    if (data != nullptr) {
      return CreateBuffer(data, attempted, originator);
    }
    if (attempted == minimal) return nullptr;
    attempted = minimal + (attempted - minimal - 1) / 2;
  }
}
```

### `ReAllocatorInternal()`方法

```
bool HeapBufferAllocator::ReallocateInternal(
    const size_t requested,
    const size_t minimal,
    Buffer* const buffer,
    BufferAllocator* const originator);
```

与`realloc()`不同，即使`requested`大小为0，也会返回一个有效的`Buffer`对象，只不过该`Buffer::size`的值为0。


## `ClearingBufferAllocator`类

包括另一个`BufferAllocator`对象，然后在每次“申请内存”和“重新申请内存”时，都进行“内存清零”操作。


## 中介者-设计模式

参见： https://www.cnblogs.com/gaochundong/p/design_pattern_mediator.html  