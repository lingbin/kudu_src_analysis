[TOC]

文件: `src/kudu/util/scoped_cleanup.h`

## 概述

用来自动的释放资源。

## `ScopedCleanup`类

封装一个函数对象，在代码执行超出作用域的时候，会自动执行该函数对象。

实现方法：在`ScopedCleanup`类的析构函数中，调用它内部封装的“函数对象”。

在使用时（不使用上面的``宏），应该使用`MakeScopedCleanup()`来显式构造出来一个`ScopedCleanup`对象。

显式使用该对象的好处：可以进行“取消”操作(`cancel()方法`)。

>> 扩展：可以类比`std::lock_guard<>`和`std::unique_lock<>`。

### `cancel()`接口

```
template<typename F>
class ScopedCleanup {
 public:
  explicit ScopedCleanup(F f)
      : cancelled_(false),
        f_(std::move(f)) {
  }
  ~ScopedCleanup() {
    if (!cancelled_) {
      f_();
    }
  }
  void cancel() { cancelled_ = true; }

 private:
  bool cancelled_;
  F f_;
};
```

## `SCOPED_CLEANUP`宏

运行一个指定的函数体（使用大括号 括起来的一个代码段）。

注意：如果希望能够取消的场景，不应该使用这个宏，而是应该显式使用下面的类。

### 使用举例

```
  Example:
    int fd = open(...);
    SCOPED_CLEANUP({ close(fd); });
```

### 实现方法

```
#define SCOPED_CLEANUP(func_body) \
  auto VARNAME_LINENUM(scoped_cleanup) = MakeScopedCleanup([&] { func_body })
```

## `MakeScopedCleanup()`模板方法

```
// Creates a new scoped cleanup instance with the provided function.
template<typename F>
ScopedCleanup<F> MakeScopedCleanup(F f) {
  return ScopedCleanup<F>(f);
}
```

**注意：**  
在c++11以后，这里的参数（一个函数对象），可以直接使用lambda表达式。

例如：
```
  auto cleanup = MakeScopedCleanup([&]() {
      // If we failed to create the thread, we need to undo all of our prep work.
      t->tid_ = INVALID_TID;
      t->Release();
    });
    
在需要取消时：

  cleanup.cancel();

```




