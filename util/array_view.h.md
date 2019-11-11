[TOC]

文件：`src/kudu/util/array_view.h`

# `template class ArrayView`

封装一个“数组”(`c array`, `std::vector`, `std::array`等)。

**注意这个类是`final`的，不可以从它继承**

```
template <typename T>
class ArrayView final {
  ...
}
```

**使用这个类的原因：**  

如果需要读写一个数组，那么一般会在“函数的参数”中会传递两个参数：1) 首个元素的指针；2) 数组的长度。

比如：
```
//   bool Contains17(const int* arr, size_t size) {
//     for (size_t i = 0; i < size; ++i) {
//       if (arr[i] == 17)
//         return true;
//     }
//     return false;
//   }
```

这种方式的好处是：  
这种方式使用的场合比较多，因为它不需要关心“数组”的具体类型。（`c array`, `std::vector`, `std::array`等都可以）。

但是它有两个缺点：
1. 比较容易出错，因为“调用者”必须显式的指定长度。

```
//   Contains17(arr, arraysize(arr));  // C array
//   Contains17(&arr[0], arr.size());  // std::vector
//   Contains17(arr, size);            // pointer + size
//   ...
```

2. 比较混乱。因为本质上是在传递一个东西（“指针”和“长度”都是在描述“数组”），但是却使用了两个参数。

**本类的成员：**  

每个`kudu::ArrayView<T>`对象中，包含两个成员:
1. `T*`类型的指针；（注意：当前`ArrayView`并不负责维护 指向的对象）；
2. 元素的个数（即所表示的数组的长度）；

**本类的行为：**  

支持在数组上的常见操作模式，比如：“按下标访问”，“使用迭代器”。

可以用如下的方式写代码（通过`for-each`来迭代“数组”）：

```
//   bool Contains17(kudu::ArrayView<const int> arr) {
//     for (auto e : arr) {
//       if (e == 17)
//         return true;
//     }
//     return false;
//   }
```

而且，`ArrayView`的一个对使用很友好的地方是：很多类型都可以“隐式地”转化为`ArrayView`，例如：

```
//   Contains17(arr);                             // C array
//   Contains17(arr);                             // std::vector
//   Contains17(kudu::ArrayView<int>(arr, size)); // pointer + size
//   ...
```

> 说明：这里`Contains(17)`的“参数”是需要一个`ArrayView`对象作为参数，但是在上面的使用时，传递的确实其它对象。  
>（在`C++`中，如果在声明`ArrayView`的构造函数时，没有使用`explicit`，那么就可以进行“隐式转化”）。

**注意1：与`const`的结合使用**  

`ArrayView<T>`和`ArrayView<const T>`是不同的类型，两者的区别是：是否允许修改其中的元素值。

一般情况下，可以像我们认为的那样，自动的进行“隐式转化”，也就是说：   
`vector<int>`既可以被转化为`ArrayView<int>`，也可以被转化为`ArrayView<const int>`。

但是注意：`const vector<int>` 只能被转化为`ArrayView<const int>`。

`ArrayView<int>`本身也可以转化为另一个`ArrayView<const int>`。

**注意2：性能的考虑**  

因为`ArrayView<T>`对象非常小（成员只有两个，一个是指针，一个是`size_t`），并且是“可拷贝的”，所以，在使用的时候， “按值传递” 可能比 “按`const`引用传递” 更高效。

>> 附录：在`c++`中，“引用”也是通过“指针”来实现的，只不过编译器封装了这个过程。  
>>   参见： https://www.cnblogs.com/xkfz007/archive/2012/02/05/2338758.html  

## 成员

```
  T* data_;
  size_t size_;
```

## 接口列表

### 构造函数

为了支持“隐式转化”，提供了多个“构造函数”（没有使用`explicit`，所以可以隐式转化）。

```
  // Construct an empty ArrayView.
  ArrayView() : ArrayView(static_cast<T*>(nullptr), 0) {}

  // Construct an ArrayView for a (pointer,size) pair.
  template <typename U>
  ArrayView(U* data, size_t size)
      : data_(size == 0 ? nullptr : data), size_(size) {
    CheckInvariant();
  }

  // Construct an ArrayView for an array.
  template <typename U, size_t N>
  ArrayView(U (&array)[N]) : ArrayView(&array[0], N) {} // NOLINT(runtime/explicit)

  // Construct an ArrayView for any type U that has a size() method whose
  // return value converts implicitly to size_t, and a data() method whose
  // return value converts implicitly to T*. In particular, this means we allow
  // conversion from ArrayView<T> to ArrayView<const T>, but not the other way
  // around. Other allowed conversions include std::vector<T> to ArrayView<T>
  // or ArrayView<const T>, const std::vector<T> to ArrayView<const T>, and
  // kudu::faststring to ArrayView<uint8_t> (with the same const behavior as
  // std::vector).
  template <typename U>
  ArrayView(U& u) : ArrayView(u.data(), u.size()) {} // NOLINT(runtime/explicit)
```

**注意1：如果`size == 0`，那么构造出来的`ArrayView`对象中，`data_`的指针恒为`nullptr`。**  

也就是说，即使用户传了一个 有“效的指针”，但是 指定的`size`值为`0`，这个时候生成的`ArrayView`对象中，指针变量`data_`的值也是`nullptr`。

这样做的目的是：因为`size == 0`，所以进行任何访问都是无效的。这样将`data_`显式的指定为`nullptr`,可以避免出现了越界访问，而却没有被用户感知到。  

因为当传入的`data`指针是有效的，但`size==0`，这时如果构造出来的`ArrayView obj`对象中，没有显式把`data_`赋值为`nullptr`，那么通过`RELEASE`模式下编译后（`DCHECK`不会生效，见`operator[]`函数，在`NDEBUG`模式下会通过`DCHECK()`来检查下标是否越界），通过`obj[i]`去访问，就可能不会立即感知到错误。

**注意2：所有的构造函数，最终都是调用“第`2`个构造函数”来初始化的。**  

**注意3：第`2`、`3`、`4`个构造方法都是“模板方法”，并且“模板参数的类型”（`typename U`）与当前类的“模板参数的类型”(`typename T`)是不同的。**  

因为当前类是“模板类”（模板参数的类型为`T`），而对于“模板方法”的构造函数，因为要支持使用“别的类型”来构建当前类型，所以在“这些构造方法”中，要使用“不同的模板参数类型”。

#### 第3个构造函数

是通过`c array`对象来创建`ArrayView`对象。

模板参数有两个：
1. 数组元素的类型；
2. 数组的长度；

注意1：这是一个“模板参数”，所以在代码中，根据该函数的参数不同，会特化出来很多版本。（相当于，`ArrayView`类有很多类似的“构造函数”）

注意2：这里通过模板参数来获取“数组类型”的方式。  
在`U (&array[N])`，`array`是“变量名”，整体是声明一个“长度为`N`的数组的引用”。  
在“初始化列表”中，会去调用第`2`个构造函数。传递的参数为`&array[0]`（即第一个元素的指针，所以传递的是一个`U*`类型。）

#### 第4个构造函数

要求模板参数有两个方法：`data()`和`size()`。

根据任何有`data()`和`size()`方法的类型，都可以用来构造`ArrayView`对象。

1. `data()`的返回值，就是当前`ArrayView`中元素的类型；
2. `size()`的返回值，会被“隐式转换”为`size_t`类型；

注意：正是因为有这个“构造方法”，既做到了能够支持从`ArrayView<T>`到`ArrayView<const T>`的隐式转化，但是不支持 “反向”的转化（即，不支持`ArrayView<const T>`到`ArrayView<T>`的隐式转化）。

因为`ArrayView<const T>::data()`返回的是 `const T*`类型，不能转化为`T*`。

**注意1：这里的参数是一个类型为`U`的“普通引用”，不是`const`引用**
如果参数写为`const U& u`，那么在某些特化版本中，就是**显式地写了“拷贝构造函数”**。  

**注意2：因为没有显式的定义“拷贝构造函数”和“赋值构造函数”，所以该类会支持默认的实现**  
因为本类只有两个成员，所以这两个函数的默认实现就是：直接将这两个成员直接赋值。

**注意3：正是因为显式定义了这个构造函数，可以保证“`ArrayView<const T>`无法隐式转化为`ArrayView<T>`”：**   
因为`ArrayView<const T>::data()`返回的是 `const T*`类型，即如果用一个`ArrayView<const T>`对象作为这个构造函数的参数，那么在这里返回的是`const T*`类型；  
而构造一个`ArrayView<T>`类，见第`2`个构造函数，需要的参数是`T*`类型（不是`const T*`）；  
又因为在`C++`中，`const T*`是无法转化为`T*`的。  
所以无法“`ArrayView<const T>`无法隐式转化为`ArrayView<T>`。（也就是说，因为`ArrayView<const T>::data()`是`const T*`类型，而`ArrayView<T>`的构造函数中，需要的是`T*`，所以无法隐式转化。）

而如果没有这个“构造函数”：
1. `ArrayView<T>` --> `ArrayView<T>`: 支持，默认的“拷贝构造函数”；
2. `ArrayView<T>` --> `ArrayView<const T>`：不支持。没有匹配的函数。    
    是不匹配默认的“拷贝构造函数”的，因为它的参数类型是`ArrayView<const T>`，不是`ArrayView<T>`。
3. `ArrayView<const T>` --> `ArrayView<T>`：不支持。没有匹配的函数。  
    与第`2`种类似，是不匹配默认的“拷贝构造函数”的，因为它的参数类型是`ArrayView<T>`，不是`ArrayView<const T>`。
4. `ArrayView<const T>` --> `ArrayView<const T>`：支持，使用默认的“拷贝构造函数”。

而如果有了这个“构造函数”：
1. `ArrayView<T>` --> `ArrayView<T>`: 支持，使用该“构造函数”（注意不再是默认的“拷贝构造函数”）
2. `ArrayView<T>` --> `ArrayView<const T>`：**可以支持了**。
3. `ArrayView<const T>` --> `ArrayView<T>`：仍然不支持。
4. `ArrayView<const T>` --> `ArrayView<const T>`：支持，使用该“构造函数”（注意不再是默认的“拷贝构造函数”）

说明1： 对于第`1`和`4`两种场景，因为“拷贝构造函数”中参数的类型是`const ArrayView&`类型，所以直接用一个“普通的（即非`const`的）`ArrayView`”去构造另外一个`ArrayView`(参数可以都是`T`，或者都是`const T`)，对应的“最佳参数匹配函数”都是这个“构造函数”，所以会使用这个函数，而不是“拷贝构造函数”。

说明2：对于对于第`1`和`4`两种场景，如果在该“构造函数”中要求的`data()`和`size()`方法，在该类中并没有提供，那么编译会报错，因为函数根据参数进行匹配的结果是“该构造函数”，但是却找不到对应的方法（`data()`和`size()`），所以报错。报错内容是`... has no member named ‘data’`.

**说明3：因为有了本“构造函数”，常用的“隐式转化”有：**  
1. `std::vector<T>` 到 `ArrayView<T>`或`ArrayView<const T>`;
2. `const std::vector<T>` 到 `ArrayView<const T>`;

**说明4：一个`faststring`对象，也提供了`data()`和`size()`方法，所以也可以用来构造`ArrayView<T>`类型**    
因为`faststring`中，封装的是`uint8_t`类型，所以对应的是`ArrayView<uint8_t>`类型。

1. 从`kudu::faststring` 到 `ArrayView<uint8_t>` 或 `ArrayView<const uint8_t>`；
2. 从`const kudu::faststring` 到 `ArrayView<const uint8_t>`;

### 作为“类array”，需要支持的接口

注意：因为当前`ArrayView`对象，是不负责维护“数组的具体内容”（即`data_`所指向的内存），所以即使当前`ArrayView<T>`对象是`const`的，对于`data()`和`operator[]`等函数返回的元素，也是可以被修改的。

如果不允许修改，应该使用`ArrayView<const T>`，这个时候`data()`返回的类型是`const T*`，所以是不可以修改的。

#### `data()` 和 `size()`

注意：见上面构造函数的讲解，这两个函数 也是为了“让 `ArrayView<>`对象之间能够相互转化”所需要的接口。

```
  size_t size() const { return size_; }
  T* data() const { return data_; }
```

注意1：这里的`data()`并没有针对“是否返回`const T*`来进行重载。”  

比如在`faststring`中，`data()`方法 有针对`const`来进行重载，是这样声明的：
```
const uint8_t* data() const { return xxx }
uint8_t* data() { return xxx }
```
这种重载方式：
1. 如果对象是`const faststring`，那么调用上面的，返回`const uint8_t*`类型，即返回一个“指向常量的指针”；
2. 如果对象是普通的`faststring`，那么调用下面的，即返回`uint8_t*`类型，返回一个“普通指针”。

但是在这里的`ArrayView`中，没有进行这种重载，所以如果是`const ArrayView<T>`类型，那么是无法调用`data()`方法的。（因为`const`对象只能调用“`const`成员方法”）。

**如果不希望通过“`data()`返回的指针”，来修改元素，那么应该使用`ArrayView<const T>`（而不是`const ArrayView<T>`）。**  

举例
1. `ArrayView<Slice>`类型：它的`data()`方法，返回的是`Slice*`类型，所以可以修改其中的内容。比如调用`Slice::removePrefix()`等方法。
2. `ArrayView<const Slice>`类型：它的`data()`方法，返回的是`const Slice*`类型. 因为是`const`的，所以不能调用`Slice::removePrefix()`等"非`const`函数”。（注意：这里在`ArrayView`的“模板参数”中的`T = const Slice`）。

注意：如果这里“针对为“`const`成员函数”进行了重载，那么对于`const ArrayView<T>类型`，就是可以调用`data()`方法了。但是返回值是`const T*`类型

即函数的定义修改为：
```
T* data() const { return data_; }
const T* data() const { return data_; }
```

因为`const ArrayView<Slice>::data()`返回的`const Slice*`，所以也是不调用`Slice::remove_prefix()`等“非`const`的成员函数”的。

#### `empty()`

```
  bool empty() const { return size_ == 0; }
```

#### 按照下标访问

```
  T& operator[](size_t idx) const {
    DCHECK_LT(idx, size_);
    DCHECK(data_);  // Follows from size_ > idx and the class invariant.
    return data_[idx];
  }
```

说明1：这里的实现可以看出来，如果`size_`为0，那么是不允许调用该方法的。  
因为传入参数`idx`为`size_t`类型，所以最小值为`0`，而如果`size==0`，那么检查`DCHECK_LT(idx, size_);`是一定不会通过的。

注意：这里在注释中强调了 将`DCHECK(data_)`放在`DCHECK_LT(idx, size_)`的后面。其实，放在“前面”和“后面”，只是在表达上略有一点小的区别。

>> 备注：构造函数保证了，在`size == 0`的时候，`data_`的值一定为`nullptr`。

首先，因为在使用下标进行数据访问的时候，`idx < size_`是一定要成立的，所以这个是一定要检查的。

其次，但是对于一个`ArrayView<T>`对象，`data_ == nullptr`是可能的（当`size == 0`的时候）。

如果先检查`DCHECK(data_)`，会给人的感觉是 在任何时候`data_`都不可以为`nullptr`（即容易让人产生误解）。

当然，因为这里两个检查都是在`operator[]`函数中，二者的先后顺序，对于检查结果没有影响。

### 支持`for...each`要提供的接口

参见：
1. http://ftxtool.org/2016/01/23/23/  C++ 如何让自己的类支持foreach 

要求是：
1. 提供两个函数：`begin()`和`end()`。
2. 返回的类型，就是“迭代器”。 要求是支持`operator*`, `operator++`和 `operator!=`。（这里因为返回的是指针类型，所以这三种操作是支持的）

```
  T* begin() const { return data_; }
  T* end() const { return data_ + size_; }
  
  const T* cbegin() const { return data_; }
  const T* cend() const { return data_ + size_; }
```

### `operator==()`

**注意：这里并不比较“具体的数据内容”，而只是比较数组指针**。

```
  friend bool operator==(const ArrayView& a, const ArrayView& b) {
    return a.data_ == b.data_ && a.size_ == b.size_;
  }
  
  friend bool operator!=(const ArrayView& a, const ArrayView& b) {
    return !(a == b);
  }
```

### `private CheckInvariant()`

在`ArrayView`中，一定成立的公式：  
  **当且仅当 `size==0`， `data_`才会为`nullptr`。**
  
也就是说，只要`size != 0`，那么`data_`一定不是`nullptr`。

```
  // Invariant: !data_ iff size_ == 0.
  void CheckInvariant() const { DCHECK_EQ(!data_, size_ == 0); }
```















