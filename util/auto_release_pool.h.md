[TOC]

## 概述

对象池。

在一个操作过程（通常是一个事务的处理过程）中，所有的对象都通过该“对象池”进行分配，那么在操作结束以后，就可以通过该对象，自动的释放所有的中间对象。

## `private struct AutoReleasePool::GenericElement`

封装一个对象。

有两个子类，分别对应 1) 普通类；2) 数组；

各个子类的实现中，区别主要是：在相应的析构函数中，在释放内存时，是释放一个普通对象(`delete xxx`)，还是释放一个数组(`delete[] xxx`)。

>> 问题：为什么需要定义一个这样的基类？
>> 答：因为两种类型的子类，有不同的实现（主要是析构函数）。但是 两种子类无法相互替换。 只有封装一个基类出来，然后在外层用户(`AutoReleasePool`类)中，就可以不需要区分类型，而只面向`GenericElement`类型即可。  
>> 而两种类型的子类，在析构时，根据自己的实际类型，分别调用相应的析构函数(面向对象的“多态”)。

```
  struct GenericElement {
    virtual ~GenericElement() {}
  };

```
### `private struct AutoReleasePool::SpecificElement`

GenericElement的子类。封装一个“**普通对象**”。

```
  template <class T>
  struct SpecificElement : GenericElement {
    explicit SpecificElement(T *t): t(t) {}
    ~SpecificElement() {
      delete t;
    }

    T *t;
  };
```

### `private struct AutoReleasePool::SpecificArrayElement`

GenericElement的子类。封装一个“**数组**”。

```
  template <class T>
  struct SpecificArrayElement : GenericElement {
    explicit SpecificArrayElement(T *t): t(t) {}
    ~SpecificArrayElement() {
      delete [] t;
    }

    T *t;
  };
```

## 成员

```
  typedef std::vector<GenericElement *> ElementVector;
  ElementVector objects_;
  base::SpinLock lock_;
```

`objects_`是对象的容器。

说明：
1. 该类是一个“对象池”，所以一定会有一个 容器成员，用来容纳多个对象。
2. 为了防止并发互斥，所以也一定会有一个`lock`来保证互斥。

**注意：多态的使用**  
上面已经说过，在容器中，只需要关注封装类型的基类（不需要区分是“普通对象”还是“数组”）。

但是在向对象容器中添加元素时，需要根据 “待添加对象”的类型，构造不同的“封装类型”（`SpecificElement`或`SpecificArrayElement`）。 

在本类中，这种区分是通过提供两个方法来实现的：`Add()`和`AddArray()`。

## 接口函数

### `Add()` -- 分配对象

```
  template <class T>
  T *Add(T *t) {
    base::SpinLockHolder l(&lock_);
    objects_.push_back(new SpecificElement<T>(t));
    return t;
  }
```

### `AddArray()`  -- 分配数组对象

```
  // Add an array-allocated object to the pool. This is identical to
  // Add() except that it will be freed with 'delete[]' instead of 'delete'.
  template<class T>
  T* AddArray(T *t) {
    base::SpinLockHolder l(&lock_);
    objects_.push_back(new SpecificArrayElement<T>(t));
    return t;
  }
```

### `DonateAllTo`  -- 转移对象到另一个“对象池”

```
  // Donate all objects in this pool to another pool.
  void DonateAllTo(AutoReleasePool* dst) {
    base::SpinLockHolder l(&lock_);
    base::SpinLockHolder l_them(&dst->lock_);

    dst->objects_.reserve(dst->objects_.size() + objects_.size());
    dst->objects_.insert(dst->objects_.end(), objects_.begin(), objects_.end());
    objects_.clear();
  }
```




