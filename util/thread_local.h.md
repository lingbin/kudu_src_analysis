[TOC]

文件：`src/kudu/util/thread_local.h`

# 一些宏

## “代码块级别”的静态`thread-local`变量
“代码块级别”是指，当前变量是定义在“函数内部”或“代码块内部”。

### `BLOCK_STATIC_THREAD_LOCAL` 

如宏的名字，这个宏定义了一个“块级别的静态的、`thread-local`变量”。

使用方法类似于`c++ 11`中的`thread_local`。

注意：其中的指针的初始化，是“lazy”的。只有第一个进入该代码块的线程，来进行初始化。

说明1：在该宏的执行时（即第一次运行进入该代码块，从而导致进行初始化的时候），会调用`type T`的构造函数。

说明2：因为是“thread-local”变量，所以在线程退出的时候，会进行析构。 但是要注意的是，理论上也必须要去手动释放。因为这里本质上是一个指针，并不会自动的去释放。  

不过，这里的“手动释放”操作，已经进行了封装，参见`AddDestructor()`的实现，利用`pthread_key_t`来封装了所有的`静态变量`的“清理函数”，从而做到了，在线程退出的时候，会自动去调用这里添加的“清理函数”，从而不需要我们在显式的调用这些清理函数。

>> 在代码中有注释，这里的实现，有参考如下资源：
>> 1. <http://pocoproject.org/docs/Poco.ThreadLocal.html>
>> 2. <http://stackoverflow.com/questions/12049684/>
>> 3. C++11 thread_local API

### 使用举例

```
// // Invokes a 3-arg constructor on SomeClass:
// BLOCK_STATIC_THREAD_LOCAL(SomeClass, instance, arg1, arg2, arg3);
// instance->DoSomething();
//
```

### 宏定义
```
#define BLOCK_STATIC_THREAD_LOCAL(T, t, ...)                                    \
static __thread T* t;                                                           \
do {                                                                            \
  if (PREDICT_FALSE(t == NULL)) {                                               \
    t = new T(__VA_ARGS__);                                                     \
    threadlocal::internal::AddDestructor(threadlocal::internal::Destroy<T>, t); \
  }                                                                             \
} while (false)
```

## “类级别”的静态`thread-local`变量

### 使用说明

和上面“块级别”的“静态`thread-local`变量”是很像的，但是因为是和“类”有关（“类成员的静态变量”和“方法中的静态变量”）是有区别的。

区别是： “类成员的静态变量”，需要在`.cpp`文件中，重新定义一次。

所以使用方法是：
1. 在类内部进行声明：`DECLARE_STATIC_THREAD_LOCAL(Type, instance_var_) `
2. 在类外部的实现中，使用`DEFINE_STATIC_THREAD_LOCAL(Type, Classname, instance_var_)`来定义这个变量。
3. 每个线程中，在使用该变量的时候，都调用一次`INIT_STATIC_THREAD_LOCAL(Type, instance_var_, ...)`来进行初始化。

说明`INIT_STATIC_THREAD_LOCAL()`是比较轻量的。可以在所有要访问“该 `static thread-local`成员属性”的方法中，进行调用。

注意：因为上面的这些要求，一般应该讲这个`static thread-local`成员属性，设置为`private`的。

```
// Example usage:
//
// // foo.h
// #include "kudu/utils/file.h"
// class Foo {
//  public:
//   void DoSomething(std::string s);
//  private:
//   DECLARE_STATIC_THREAD_LOCAL(utils::File, file_);
// };
//
// // foo.cc
// #include "kudu/foo.h"
// DEFINE_STATIC_THREAD_LOCAL(utils::File, Foo, file_);
// void Foo::WriteToFile(std::string s) {
//   // Call constructor if necessary.
//   INIT_STATIC_THREAD_LOCAL(utils::File, file_, "/tmp/file_location.txt");
//   file_->Write(s);
// }
```

### `DECLARE_STATIC_THREAD_LOCAL()宏` 

如宏的名字，这个宏定义了一个“类级别的、静态的、`thread-local`变量”。

>> vigilance: 警戒；警觉；  

`DECLARE_STATIC_THREAD_LOCAL(Type, instance_var_)`：必须放在“类声明”中（通常是`.h`文件），即该宏必须出现的 `class T{...};`的声明内部，就和定义普通的“类成员属性”一样。

### `DEFINE_STATIC_THREAD_LOCAL()宏`

### `INIT_STATIC_THREAD_LOCAL()宏`

注意：这个宏可以被调用多次。（如果已经被调用过，宏内部会什么都不做）

所以，对于所有要访问该“`static thread-local`成员属性”的对象，都可以调用该宏。

```
// Goes in the class declaration (usually in a header file).
// dtor must be destructed _after_ t, so it gets defined first.
// Uses a mangled variable name for dtor since it must also be a member of the
// class.
#define DECLARE_STATIC_THREAD_LOCAL(T, t)                                                     \
static __thread T* t

// You must also define the instance in the .cc file.
#define DEFINE_STATIC_THREAD_LOCAL(T, Class, t)                                               \
__thread T* Class::t

// Must be invoked at least once by each thread that will access t.
#define INIT_STATIC_THREAD_LOCAL(T, t, ...)                                       \
do {                                                                              \
  if (PREDICT_FALSE(t == NULL)) {                                                 \
    t = new T(__VA_ARGS__);                                                       \
    threadlocal::internal::AddDestructor(threadlocal::internal::Destroy<T>, t);   \
  }                                                                               \
} while (false)
```

>> 问题：上面的一段注释是什么意思？ 什么是`dtor must be destructed _after_ t`?

# 全局变量

## `destructors_key`
在整个进程中，用来绑定“析构函数”，从而在进程退出时执行。

## `once`
类型为`GoogleOnceType`，用来保证`CreateKey()`函数只被执行一次。

其实就是保证`destructors_key`只被初始化一次。

```
static pthread_key_t destructors_key;
static GoogleOnceType once = GOOGLE_ONCE_INIT;
```

# `struct PerThreadDestructorList`
是链表的一个节点。

在`.cpp`文件中定义，所以只能在本`.cpp`文件中使用。

该链表中的每个节点，都是该线程对应的所有“析构函数”。

```
struct PerThreadDestructorList {
  void (*destructor)(void*);
  void* arg;
  PerThreadDestructorList* next;
};
```

# 全局函数

## 工具函数
在`.cpp`文件中定义。

### `static InvokeDestructors()`

清理函数。当一个线程结束的时候，会调用该函数，去清理“当前线程”在“线程私有数据”(`destructors_key`)中对应的部分。

该函数，在初始化`destructors_key`的时候，就会被注册进去。（参见`CreateKey()`）.

```
// Call all the destructors associated with all THREAD_LOCAL instances in this
// thread.
static void InvokeDestructors(void* t) {
  PerThreadDestructorList* d = reinterpret_cast<PerThreadDestructorList*>(t);
  while (d != nullptr) {
    d->destructor(d->arg);
    PerThreadDestructorList* next = d->next;
    delete d;
    d = next;
  }
}
```


### `static CreateKey()`

注意这个函数中，会初始化`destructors_key`，所以在一个进程中，该函数只需要被调用一次。

所以这里使用了`once`来进行保护。

```
// This key must be initialized only once.
static void CreateKey() {
  int ret = pthread_key_create(&destructors_key, &InvokeDestructors);
  // Linux supports up to 1024 keys, we will use only one for all thread locals.
  CHECK_EQ(0, ret) << "pthread_key_create() failed, cannot add destructor to thread: "
      << "error " << ret << ": " << ErrnoToString(ret);
}
```

说明：如注释：在`Linux`中，最多支持`1024`个`pthread_key_t`。

这里的实现中，对于所有的`thread local`变量，只使用一个（`destructors_key`）。 如果有多个`static thread-local`变量，那么会组成层链表。

说明2：这里注册了`InvokeDestructors()`作为清理函数，所以在一个线程结束的时候，就会调用`InvokeDestructors()`来析构，当前线程的所有的“`static thread-local`变量”。

## 接口函数

### `AddDestructor()`

添加一个“析构函数”。

```
void AddDestructor(void (*destructor)(void*), void* arg) {
  GoogleOnceInit(&once, &CreateKey);

  // Returns NULL if nothing is set yet.
  std::unique_ptr<PerThreadDestructorList> p(new PerThreadDestructorList());
  p->destructor = destructor;
  p->arg = arg;
  p->next = reinterpret_cast<PerThreadDestructorList*>(pthread_getspecific(destructors_key));
  int ret = pthread_setspecific(destructors_key, p.release());
  // The only time this check should fail is if we are out of memory, or if
  // somehow key creation failed, which should be caught by the above CHECK.
  CHECK_EQ(0, ret) << "pthread_setspecific() failed, cannot update destructor list: "
      << "error " << ret << ": " << ErrnoToString(ret);
}
```

注意1：因为`destructors_key`是“线程私有数据”，所以对它的修改，不需要加锁。

注意2：虽然全局只使用了一个`pthread_key_t`类型的变量（`destructors_key`），但是每个线程，都会对应一个“`static thread-local`变量”的链表。当线程结束时，该线程所对应的链表上的所有“清理函数”，都会被调用，所以，在这些“`static thread-local`变量”，所包含的“该线程对应的部分”，就会被析构掉。

```
                                   thread_1     thread_2     thread_3    thread_4
     static thread-local var1        v1_t1       v1_t2         v1_t3       ---
     static thread-local var2        v2_t1       v2_t2         -----       v2_t4
     static thread-local var3        v3_t1       -----         v3_t3       v3_t4
     static thread-local var4        -----       v4_t2         v4_t3       v4_t4
     
     上图中，如果是`-----`，表示该变量，在对应的`thread`中，没有相应的值。
```

如上图，如果`thread_1`结束运行，那么所有的`xx_t1`就会被“清理”。

### `Destroy()`

```
template<class T>
static void Destroy(void* t) {
  // With tcmalloc, this should be pretty cheap (same thread as new).
  delete reinterpret_cast<T*>(t);
}
```

注意：这虽然一个“模板函数”，但是所有的“特化版本”，都可以用“函数指针”（`void (*destructor)(void*);`）来表示。

所以在`PerThreadDestructorList`中，就用这个“函数指针”来指向一个“清理函数”。

说明：如果使用了`tcmalloc`库，这个函数理论上应该是否“非常轻量”的。

# 附录：“线程私有数据”，即`pthread_key_t`和`pthread_key_create()`的使用方法

参见：
1. https://www.cnblogs.com/lit10050528/p/4325882.html    pthread_key_t和pthread_key_create()详解
2. https://www.ibm.com/developerworks/cn/linux/thread/posix_threadapi/part2/index.html Posix线程编程指南(2)

`pthread_key_t`，表示一个“线程私有数据”（`Thread-specific Data`，或`TSD`）.

基本流程是这样的：
```
1. 创建`pthread_key_t`类型的变量

2. 调用`pthread_key_create()`来初始化。
   该函数的第2个参数，是一个“析构函数”（清理函数）。在线程退出时，自动调用这个“清理函数”。

3. 在每个线程中，使用`pthread_setspecific()`来设置值。该值只有在该线程中才能访问到。

4. 在每个线程中，使用`pthread_getspecific()`，来获取刚才设置进去的值。

5. 使用`pthread_key_delete()`，来删除一个“线程私有数据”。
```
