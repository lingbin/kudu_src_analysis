[TOC]

文件: `src/kudu/common/column_predicate.h`

参见`ColumnPredicatePB`（文件`src/kudu/common/common.proto`）

# `enum PredicateType`

## `None`
一个空的过滤条件，始终返回`false`。

## `Equality`
是否相等；

## `Range`
是否属于某个range： 即在`[start, end)`中。

## `IsNotNull`
是否非空；

## `IsNull`
是否为空；

## `InList`
是否在一个list中；

## `InBloomFilter`
是否在bloomfilter中；

>> 问题：这个的用途场景是？

```
enum class PredicateType {
  None,
  Equality,
  Range,
  IsNotNull,
  IsNull,
  InList,
  InBloomFilter,
};

```

# `class ColumnPredicate`

描述一个过滤条件。

注意：有了`ColumnPredicate`对象，然后可以让一个`column block`的数据经过这个`ColumnPredicate`对象，符合条件的会被保留下来，不符合的会被过滤掉。

如果两个过滤条件是作用在 同一列 上的，那么可以合并成一个（即将多个`ColumnPredicate`对象合并为1个，参见`ColumnPredicate::Merge()`方法）。

有很多种过滤方式。

不同种类的过滤条件，在“合并”和“判断”的时候（"merging and evaluating"），有不同的行为。

注意： 在一个`ColumnPredicate`对象中，它是不拥有它内部所引用的数据的，所以这些被引用数据的生命周期，一定要被管理起来，从而保证在`ColumnPredicate`的生命周期内，这些指针数据不会失效。

一般情况下，一个`ColumnPredicate`对象的声明周期，会有两种场景：
1. 在`client`端：会被绑定到一个`scan`上；
2. 在`server`端：会被绑定到一个`scan iterator`上；

## `class ColumnPredicate::BloomFilterInner`

描述在“过滤条件”中使用的`bloom filter`条件。

### 成员


### 接口

只是提供了对各个成员的`setter`和`getter`方法，没有其它方法。

## 成员

### `predicate_type_`
过滤条件的类型.

### `column_`
针对“哪一列”进行过滤。

### `lower_`
过滤条件的下限。

有两种含义：
1. `range`类型：表示 数据区间的下边界，是“包含”的。
2. `equality`类型：表示 需要相等的值。

### `upper_`
只用于`range`类型，表示上边界，是“不包含”的。

### `values_`
用于`inList`类型，表示目标值的集合，是一个`std::vector`类型。

### `bloom_filters_`
用于`InBloomFilter`类型，表示`bloom_filters_`的集合。

```
  PredicateType predicate_type_;
  ColumnSchema column_;
  const void* lower_;
  const void* upper_;
  std::vector<const void*> values_;
  std::vector<BloomFilterInner> bloom_filters_;
```

## 方法

### `private`的构造函数

这3个构造函数有不同的用途：
1. 创建`equality`/`range`/`is_null`/`is_not_null`类型；
2. 创建`inList`类型；
3. 创建`BloomFilter`类型；

```
ColumnPredicate::ColumnPredicate(PredicateType predicate_type,
                                 ColumnSchema column,
                                 const void* lower,
                                 const void* upper)
    : predicate_type_(predicate_type),
      column_(move(column)),
      lower_(lower),
      upper_(upper) {
}

ColumnPredicate::ColumnPredicate(PredicateType predicate_type,
                                 ColumnSchema column,
                                 vector<const void*>* values)
    : predicate_type_(predicate_type),
      column_(move(column)),
      lower_(nullptr),
      upper_(nullptr) {
  values_.swap(*values);
}

ColumnPredicate::ColumnPredicate(PredicateType predicate_type,
                                 ColumnSchema column,
                                 std::vector<BloomFilterInner>* bfs,
                                 const void* lower,
                                 const void* upper)
    : predicate_type_(predicate_type),
      column_(move(column)),
      lower_(lower),
      upper_(upper) {
  bloom_filters_.swap(*bfs);
}
```

说明：  
参数中的`ColumnSchema`对象，虽然使用的是`对象形式`，但是这里使用的是`move`语义，所以也是不会进行拷贝的。


### 静态的工厂方法

#### `static Equality()`

```
ColumnPredicate ColumnPredicate::Equality(ColumnSchema column, const void* value) {
  CHECK(value != nullptr);
  return ColumnPredicate(PredicateType::Equality, move(column), value, nullptr);
}
```

注意： 传入的`value`是不会被拷贝的，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

#### `static Range()`

**是“左闭右开”区间**。

```
ColumnPredicate ColumnPredicate::Range(ColumnSchema column,
                                       const void* lower,
                                       const void* upper) {
  CHECK(lower != nullptr || upper != nullptr);
  ColumnPredicate pred(PredicateType::Range, move(column), lower, upper);
  pred.Simplify();
  return pred;
}
```

说明：“左右边界”不能同时为`null`。

如果一侧为`null`，那么说明这一侧是没有边界的。也就是说：
1. 只有`lower`为`null`: 区间是`(负无穷, upper)`
2. 只有`upper`为`null`: 区间是`[lower, 正无穷)`

注意： 传入的`value`是不会被拷贝的，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

#### `static InList()`

```
ColumnPredicate ColumnPredicate::InList(ColumnSchema column,
                                        vector<const void*>* values) {
  CHECK(values != nullptr);

  // Sort values and remove duplicates.
  std::sort(values->begin(), values->end(),
            [&] (const void* a, const void* b) {
              return column.type_info()->Compare(a, b) < 0;
            });
  values->erase(std::unique(values->begin(), values->end(),
                            [&] (const void* a, const void* b) {
                              return column.type_info()->Compare(a, b) == 0;
                            }),
                values->end());

  ColumnPredicate pred(PredicateType::InList, move(column), values);
  pred.Simplify();
  return pred;
}
```

说明1：  
必须提供一个有效的“值集合”。

说明2：  
会对传入的“值集合”进行“排序”和“去重”。

说明3：这个`values`数组，是可能会为一个空数组的（即`values.empty() == true`）。 因为用户的输入条件可能为`WHERE col1 in ()`，即用户可能提供一个“空列表”。

注意： 传入的`value`是不会被拷贝的，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

#### `static InBloomFilter()`

```
ColumnPredicate ColumnPredicate::InBloomFilter(ColumnSchema column,
                                               std::vector<BloomFilterInner>* bfs,
                                               const void* lower,
                                               const void* upper) {
  CHECK(bfs != nullptr);
  CHECK(!bfs->empty());
  ColumnPredicate pred(PredicateType::InBloomFilter, move(column), bfs, lower, upper);
  pred.Simplify();
  return pred;
}
```
注意： 传入的`value`是不会被拷贝的，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

#### `static InclusiveRange()`

传入一个“左闭右闭”的区间，生成一个“查询条件”。

**注意1：**  
传入参数`lower`和`upper`不能同时为`null`

**注意2：**  
传入的`lower`和`upper`都不会被拷贝，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

**注意3：**  
因为底层处理的时候，一定要是一个“左闭右开”的区间，即最终返回的`ColumnPredicate`对象中，一定是一个“左闭右开”的区间，所以这里需要转化一下：根据当前column的类型，将`upper`向上增加一个单位。

即，只要传递进来的`upper`不是空（注意：这时`lower`是可能为`null`的），那么就一定将它增大一个单位。

**注意4：**  
参数`upper`可能已经是该类型的“最大值”，这个时候也要进行处理。

如果`upper`已经是最大值（无法增加一个单位），那么：
1. 如果`lower`为`null`,说明传递进来的区间，是`(负无穷，最大值]`，它是等价于`(负无穷，正无穷)`的。也就是说，只要该列的值不为`null`（`null`与任何值比较大小，结果都是`false`）,那么它的判断结果一定为`true`，所以这里会再分为两种情况：
  + 1.1 如果该列“不可以为`null`”，也就相当于，没有这个条件(返回`boost::none`)；
  + 1.2 如果该列“可以为`null`”，那么要注意，那就转化为`ColumnPredicate::IsNotNull()`。
2. 如果`lower`不为空，那么说明传递进来的区间是`[lower，最大值]`，它等价于`[lower, 正无穷)`。

**注意5:**   
因为可能会忽略掉这个条件（传递进来的区间是`(负无穷，最大值]`）,所以当前函数返回类型是`boost::optional<ColumnPredicate>`。在需要忽略时，返回`boost::none`。这样调用者就可以检查并忽略掉这个条件。

**注意6：**  
因为因为最终返回的`ColumnPredicate`对象中，所引用的是`upper_1`，是“增加1个单位的`upper`”，而`upper_1`的真正内存地址，也必须指向一个有效的地址。 所以在参数中，会传入一个“内存池”对象(`arena`)，“增大的`upper`”的内存是在这个“内存池”中分配的。

```
boost::optional<ColumnPredicate> ColumnPredicate::InclusiveRange(ColumnSchema column,
                                                                 const void* lower,
                                                                 const void* upper,
                                                                 Arena* arena) {
  CHECK(lower != nullptr || upper != nullptr);

  if (upper != nullptr) {
    // Transform the upper bound to exclusive by incrementing it.
    // Make a copy of the value before incrementing in case it's aliased.
    size_t size = column.type_info()->size();
    void*  buf = CHECK_NOTNULL(arena->AllocateBytes(size));
    memcpy(buf, upper, size);
    if (!key_util::IncrementCell(column, buf, arena)) {
      if (lower == nullptr) {
        if (column.is_nullable()) {
          // If incrementing the upper bound fails and the column is nullable,
          // then return an IS NOT NULL predicate, so that null values will be
          // filtered.
          return ColumnPredicate::IsNotNull(move(column));
        } else {
          return boost::none;
        }
      } else {
        upper = nullptr;
      }
    } else {
      upper = buf;
    }
  }
  return ColumnPredicate::Range(move(column), lower, upper);
}
```

说明1：最后调用`ColumnPredicate::Range()`来进行构造，在该方法中，会调用`Simplify()`的。所以在本方法中，不需要再去调用`Simplify()`了。

说明2：这里会利用“返回值优化”，即实际上是没有对象拷贝的。

#### `static ExclusiveRange()`

传入一个“左开右开”的区间，生成一个“查询条件”。

**注意1：**  
传入参数`lower`和`upper`不能同时为`null`

**注意2：**  
传入的`lower`和`upper`都不会被拷贝，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。

**注意3：**  
和`InclusiveRange()`类似，因为要返回一个“左闭右开”的区间，所以需要对`lower`进行转化：根据当前column的类型，将`lower`向上增加一个单位。

即，只要传递进来的`lower`不是空（注意：这时`upper`是可能为`null`的），那么就一定将`lower`增大一个单位。

**注意4：**  
如果`lower`已经是最大值（无法增加一个单位），那么说明传进来的区间是`(最大值，upper)`，这个时候，无论`upper`值为什么，都是一个无效的区间，即用它来判断 任何值，结果都是`false`。这时，返回`ColumnPredicate::None(move(column))`，表示一个无效的判断（即所有判断结果都是`false`）.

**注意5: **  
和`InclusiveRange()`类似，因为最终返回的`ColumnPredicate`对象中，所引用的`lower_1`的是“增加1个单位的`lower`”，而`lower_1`的真正内存地址，也必须指向一个有效的地址。 所以在参数中，会传入一个“内存池”对象(`arena`)，“增大的`lower_1`”的内存是在这个“内存池”中分配的。

```
ColumnPredicate ColumnPredicate::ExclusiveRange(ColumnSchema column,
                                                const void* lower,
                                                const void* upper,
                                                Arena* arena) {
  CHECK(lower != nullptr || upper != nullptr);

  if (lower != nullptr) {
    // Transform the lower bound to inclusive by incrementing it.
    // Make a copy of the value before incrementing in case it's aliased.
    size_t size = column.type_info()->size();
    void* buf = CHECK_NOTNULL(arena->AllocateBytes(size));
    memcpy(buf, lower, size);
    if (!key_util::IncrementCell(column, buf, arena)) {
      // If incrementing the lower bound fails then the predicate can match no values.
      return ColumnPredicate::None(move(column));
    } else {
      lower = buf;
    }
  }
  return ColumnPredicate::Range(move(column), lower, upper);
}
```

说明1：最后调用`ColumnPredicate::Range()`来进行构造，在该方法中，会调用`Simplify()`的。所以在本方法中，不需要再去调用`Simplify()`了。

说明2：这里会利用“返回值优化”，即实际上是没有对象拷贝的。

#### `static IsNotNull()`

```
ColumnPredicate ColumnPredicate::IsNotNull(ColumnSchema column) {
  return ColumnPredicate(PredicateType::IsNotNull, move(column), nullptr, nullptr);
}
```

#### `static IsNull()`

注意：如果当前column是 “不可为`null`”的(即`column_schema.is_nullable() == false`)，那么直接返回`ColumnPredicate::None`，表示所有判断结果都是`false`。

```
ColumnPredicate ColumnPredicate::IsNull(ColumnSchema column) {
  return column.is_nullable() ?
         ColumnPredicate(PredicateType::IsNull, move(column), nullptr, nullptr) :
         None(move(column));
}
```

#### `static None()`

```
ColumnPredicate ColumnPredicate::None(ColumnSchema column) {
  return ColumnPredicate(PredicateType::None, move(column), nullptr, nullptr);
}
```

### `private Simplify()`

如果可能的话，就简化下过滤条件。 

比如说：对于`range`类型，如果`lower`和`upper`是相等的，那么就可以转化为一个`equality`类型。

对于`None`/`Equality`/`IsNotNull`/`IsNull`类型，不需要进行转化。

即只有`Range`/`InList`/`InBloomFilter` 类型，可能会需要转化。

#### `Range`类型条件的转化

注意：这时，`lower`和`upper`是不可以同时为`nullptr`的。

**1. 如果`lower`和`upper`都不为`nullptr`**  
因为表达的区间是`[lower, upper)`。

1. 如果`lower == upper`，即表达的是一个空区间，这个时候，用“任何值”（包括`null`）来进行判断，结果都是`false`，所以转化为`ColumnPredicate::None()`。
2. 如果`lower`和`upper`只差1个单位（即`type_info->AreConsecutive(lower_, upper_)`），那么说明区间中只包含一个值，相当于`[lower, lower]`，这时转化为`Enqality`类型。

**2. 如果只有`lower`不为`nullptr`（即`upper`为`nullptr`）**  

这时表达的区间是`[lower, 正无穷)`

1. 如果`lower`是该类型的最小值，那么说明这个区间已经包含了“该类型的值域上所有的值”（除了`null`以外的所有值，判断结果都为`true`），这个时候就可以转化为`IsNotNull`类型。
2. 如果`lower`的值是该类型的最大值，那么区间是`[最大值，正无穷)`，即只包含“最大值”这一个值，那么转化为`Equality`类型。

>> 问题：目前没有一种类型是表示`ALL`的，即所有判断一定是返回`true`的。对于这种类型，那么是可以忽略掉这个条件的。（如果有的话，对于`IsNotNull`类型，如果从该列的元信息可以判断该列不能为`nullable`的话，那么就这可以转化为这个类型。）

>> 问题：实际上，对于为`ALL`的类型，在`sql语句`的判断条件是`OR`的时候，是不能忽略的。

>> 问题：在kudu中，对于`OR`类型的条件连接，是如何处理的？

**3. 如果只有`upper`不为`nullptr`（即`lower`为`nullptr`）**  
这时表达的区间是`(负无穷，upper)`

1. 如果`upper`的值是该类型的最小值，说明这是一个“空区间”，任何值（包括`null`）进行判断的结果都是`false`，所以转化为`None`类型。

说明：如果`upper`的值是该类型的最大值，那么判断区间是`(负无穷，最大值）`，这是一个有效的区间（只有“最大值”进行判断时返回值是`false`），所以这时无法进行简化。

#### `InList`类型条件的转化

说明：“值集合”中的值，是经过“排序”和“去重”的。

1. 如果提供的“值集合”为空（即没有提供任何值），那么转化为`None`；
2. 如果只提供了一个值，那么转化为`Equality`类型；
3. 如果是`BOOL`类型（值域只有两个值:`true`和`false`），那么可以检查下，如果“值集合”中的值正好是两个，那么就转化为`IsNotNull`类型；

说明：理论上，对于整型类型，也可以进行检查“值集合”中值的个数，如果已经“值集合”中值的个数，等于该类型的值的最大个数，那么就转化为`IsNotNull`类型；

比如：对于`UINT8`类型（取值范围是`[0, 511]`，一共`512`个整数），这时，如果“值集合”中的值的个数为`512`，所以已经覆盖了所有值，可以转化为`IsNotNull`类型；

但是，实际上，对于整型，这种转化并不常见（用户很少会提供这样一个“完整值域”的方式，去使用`in`条件），所以这里并没有判断。

#### `InBloomFilter`类型条件的转化

>> 问题：待补充

```
// TODO(granthenke): For decimal columns, use column_.type_attributes().precision
// to calculate the "true" max/min values for improved simplification.
void ColumnPredicate::Simplify() {
  auto type_info = column_.type_info();
  switch (predicate_type_) {
    case PredicateType::None:
    case PredicateType::Equality:
    case PredicateType::IsNotNull: return;
    case PredicateType::IsNull: return;
    case PredicateType::Range: {
      DCHECK(lower_ != nullptr || upper_ != nullptr);
      if (lower_ != nullptr && upper_ != nullptr) {
        // _ <= VALUE < _
        if (type_info->Compare(lower_, upper_) >= 0) {
          // If the range bounds are empty then no results can be returned.
          SetToNone();
        } else if (type_info->AreConsecutive(lower_, upper_)) {
          // If the values are consecutive, then it is an equality bound.
          predicate_type_ = PredicateType::Equality;
          upper_ = nullptr;
        }
      } else if (lower_ != nullptr) {
        // VALUE >= _
        if (type_info->IsMinValue(lower_)) {
          predicate_type_ = PredicateType::IsNotNull;
          lower_ = nullptr;
        } else if (type_info->IsMaxValue(lower_)) {
          predicate_type_ = PredicateType::Equality;
        }
      } else if (upper_ != nullptr) {
        // VALUE < _
        if (type_info->IsMinValue(upper_)) {
          SetToNone();
        }
      }
      return;
    };
    case PredicateType::InList: {
      if (values_.empty()) {
        // If the list is empty, no results can be returned.
        SetToNone();
      } else if (values_.size() == 1) {
        // List has only one value, so convert to Equality
        predicate_type_ = PredicateType::Equality;
        lower_ = values_[0];
        values_.clear();
      } else if (type_info->type() == BOOL) {
        // If this is a boolean IN list with both true and false in the list,
        // then we can just convert it to IS NOT NULL. This same simplification
        // could be done for other integer types, but it's probably not as
        // common (and hard to test).
        predicate_type_ = PredicateType::IsNotNull;
        lower_ = nullptr;
        upper_ = nullptr;
        values_.clear();
      }
      return;
    };
    case PredicateType::InBloomFilter: {
      if (lower_ == nullptr && upper_ == nullptr) {
        return;
      }
      // Merge the optional lower and upper bound.
      if (lower_ != nullptr && upper_ != nullptr) {
        if (type_info->Compare(lower_, upper_) >= 0) {
          // If the range bounds are empty then no results can be returned.
          SetToNone();
        } else if (type_info->AreConsecutive(lower_, upper_)) {
          if (CheckValueInBloomFilter(lower_)) {
            predicate_type_ = PredicateType::Equality;
            upper_ = nullptr;
            bloom_filters_.clear();
          } else {
            SetToNone();
          }
        }
      } else if (lower_ != nullptr) {
        if (type_info->IsMinValue(lower_)) {
          lower_ = nullptr;
        } else if (type_info->IsMaxValue(lower_)) {
          if (CheckValueInBloomFilter(lower_)) {
            predicate_type_ = PredicateType::Equality;
            bloom_filters_.clear();
          } else {
            SetToNone();
          }
        }
      } else if (upper_ != nullptr) {
        if (type_info->IsMinValue(upper_)) {
          SetToNone();
        }
      }
      return;
    };
  }
  LOG(FATAL) << "unknown predicate type";
}
```

### `Merge()`  -- 合并多个`ColumnPredicate`对象

对于“针对同一列”的多个`ColumnPredicate`，如果是“并且”(`AND`)的关系，那么是可以合并的。

**注意：**  
传入`ColumnPredicate`对象中的数据都不会被拷贝，所以调用者必须保证“传入值”的生命周期 比 “返回的`ColumnPredicate`对象”的生命周期 要长。


### `Evaluate()`  -- 对数据进行校验


### `private template<T> EvaluateForPhysicalType()`

### `private template<T> EvaluateCellForBloomFilter()`


### `private CheckValueInRange()`


### `private CheckValueInList()`


### `private CheckValueInBloomFilter()`



