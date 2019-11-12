[TOC]

文件：`src/kudu/tablet/rowset_tree.h`

# `class RowSetTree`

封装了一个`tablet`的所有`RowSet`的集合。

**目的：**  
给定一个`row key`，可以快速的找到包含这个`row key`的`RowSet`子集（即这个`row key`）可能包含在这些`RowSet`中。

另外，当前`RowSetTree`对象中，维护了所有`RowSet`形成的“线段”信息。  

比如：一个`tablet`有两个`RowSet`，区间分别是`[0, 2]`和`[1, 3]`。那么它实际上就有3个连续的线段：`[0, 1]`, `[1, 2]`和 `[2, 3]`。

## `enum RowSetTree::EndpointType`

会作为`RowSetTree::RSEndpoint`的属性。表示一个“点”是线段的“起点”还是“终点”。

```
  enum EndpointType {
    START,
    STOP
  };
```

## `struct RowSetTree::RSEndpoint`

这时一个`POD`结构。

包含3个属性：
1. 对应的`RowSet`对象指针；
2. 一个“点”，就是一个`row key`的值；
3. 该“点”是作为“起点”，还是作为“终点”；

```
  struct RSEndpoint {
    RSEndpoint(RowSet *rowset, EndpointType endpoint, Slice slice)
        : rowset_(rowset), endpoint_(endpoint), slice_(slice) {}

    RowSet* rowset_;
    enum EndpointType endpoint_;
    Slice slice_;
  };
```

## 私有的`struct QueryStuct`

因为是在`.cpp`文件中的“匿名的命名空间”中定义的，所以只有当前`cpp`文件可用。

在使用一组查询“批量地”访问当前“线段树”时，会使用该结构。

两个成员：
1. `slice`: 用户所给的`key`；
2. `idx`: 在一个批量请求中，该`key`所位于的“下标号”；

```
namespace {
    
struct QueryStruct {
  Slice slice;
  int idx;
};

} // anonymous namespace

```

## 私有的`struct RowSetWithBounds`

“线段树”中的一个条目。

在`cpp`文件中进行定义，所以只有当前`cpp`文件可以使用。

### `min_key`
当前线段的“起点”；

### `max_key`
当前线段的“终点”；

### `rowset`
当前“线段”所对应的`RowSet`列表。

>> purposeful： 有目的的；有决心的；

**注意： 这个`struct`结构中，各个成员的顺序不要随便变化。**  

因为经常访问`min_key`和`max_key`，所以将他们放在前面，是为了保证他们能够在一条`cpu cache line`（因为每个`cpu cache line`是64个字节，而这`min_key`和`max_key`的大小都是32字节）中。

访问该结构体的`rowset`属性的场景是非常少的，所以放在最后。

```
struct RowSetWithBounds {
  string min_key;
  string max_key;

  // NOTE: the ordering of struct fields here is purposeful: we access
  // min_key and max_key frequently, so putting them first in the struct
  // ensures they fill a single 64-byte cache line (each is 32 bytes).
  // The 'rowset' pointer is accessed comparitively rarely.
  RowSet *rowset;
};

```

## 私有的`class RowSetTree::RowSetIntervalTraits`

表示一个“线段”。是模板工具类`IntervalTree`的模板参数。

注意：在`cpp`文件中进行定义，所以只有当前`cpp`文件可以使用。

说明：其中模板工具类`IntervalTree`对其 模板参数的要求有：

1. 要有两个子类型
   + `point_type`：“点”的类型；
   + `interval_type`：“线段”的类型；
2. 要有几个方法
   + `static get_left()`: 获取该“线段”的“起点”；
   + `static get_right()`: 获取该“线段”的“起点”；
   + `static compare()`: 比较两个点；

```
struct RowSetIntervalTraits {
  typedef Slice point_type;
  typedef RowSetWithBounds *interval_type;

  static Slice get_left(const RowSetWithBounds *rs) {
    return Slice(rs->min_key);
  }

  static Slice get_right(const RowSetWithBounds *rs) {
    return Slice(rs->max_key);
  }

  static int compare(const Slice &a, const Slice &b) {
    return a.compare(b);
  }

  static int compare(const Slice &a, const QueryStruct &b) {
    return a.compare(b.slice);
  }

  static int compare(const QueryStruct &a, const Slice &b) {
    return -compare(b, a);
  }

  // When 'a' is boost::none:
  //  (1)'a' is +OO when 'positive_direction' is true;
  //  (2)'a' is -OO when 'positive_direction' is false.
  static int compare(const boost::optional<Slice>& a,
                     const Slice& b,
                     const EndpointIfNone& type) {
    if (a == boost::none) {
      return ((POSITIVE_INFINITY == type) ? 1 : -1);
    }

    return compare(*a, b);
  }

  static int compare(const Slice& a,
                     const boost::optional<Slice>& b,
                     const EndpointIfNone& type) {
    return -compare(b, a, type);
  }
};
```

## 成员





