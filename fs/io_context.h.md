[TOC]

## 概述

使用`IOContext`对象来 **作为单一的接口**，在IO操作过程中各部分进行传递。

一个`IOContext`对象，用来表示一个高层级的IO操作。比如：一个`Scan`操作，一个`tablet`的启动等。

对于任意一个操作，在高层的模块，都需要有一个`IOContext`对象。然后，**在较低层级的模块，都是传递该对象的指针**。

在这些较低层级的模块中，都必须使用（该`IOContext`对象的）指针进行访问，即必须在该`IOContext`的生命周期内才能访问。

>> outlive: 比…活得长；比…经久  

例如：
1. 一个`Tablet::Iterator`会进行IO操作，所以应该拥有一个`IOContext`对象。 所有的“子迭代器”都是通过传递该对象的指针，并且会保存该指针。 这里的潜在假设是： 在这些“子迭代器”中，使用的`IOContext`指针 的生命周期，是不会超过父类（`Talbet::Iterator`）的。
2. `Tablet bootstrap`会进行IO操作，所以在启动时会创建一个`IOContext`，并传递给它的子模块（比如: `CFiles`）。这里的预期是：因为这些低层级的模块，生命周期可能长于 bootstrap过程，所以在这些子模块中，不保存`IOContext`的指针，而是在需要的时候将它们作为参数。

>> 说明：  
>> 上面的两个例子中，共同点是：都要使用`IOContext`指针。 
>> 不同点是： 是否会在“子模块”中保存这个指针。

也就是说：
1. 如果“子模块”的生命周期 小于 “顶层模块”（即一定是在所有“子模块”都结束之后，“顶层模块”才会结束）：那么这时在“子模块”中保存`IOContext`指针。
2. 相反，就不再“子模块”中保存`IOContext`指针。而只是在需要用它的时候，作为函数的参数传入进去。

```
// An IOContext provides a single interface to pass state around during IO. A
// single IOContext should correspond to a single high-level operation that
// does IO, e.g. a scan, a tablet bootstrap, etc.
//
// For each operation, there should exist one IOContext owned by the top-level
// module. A pointer to the context should then be passed around by lower-level
// modules. These lower-level modules should enforce that their access via
// pointer to the IOContext is bounded by the lifetime of the IOContext itself.
//
// Examples:
// - A Tablet::Iterator will do IO and own an IOContext. All sub-iterators may
//   pass around and store pointers to this IOContext, under the assumption
//   that they will not outlive the parent Tablet::Iterator.
// - Tablet bootstrap will do IO and an IOContext will be created during
//   bootstrap and passed around to lower-level modules (e.g. to the CFiles).
//   The expectation is that, because the lower-level modules may outlive the
//   bootstrap and its IOContext, they will not store the pointers to the
//   context, but may use them as method arguments as needed.
struct IOContext {
  // The tablet id associated with this IO.
  std::string tablet_id;
};
```

>> 问题：这里的tablet_id为什么是字符串类型？ 而不是`int64_t`？