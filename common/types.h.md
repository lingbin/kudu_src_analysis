[TOC]

文件： `src/kudu/common/types.h`

## 概述

该文件中，
对外的接口类是`TypeInfo`。 

但是在初始化它的时候，使用了“模板元编程”。通过定义各个不同的类，避免了普通方式下进行初始化时大量采用`switch... case...`的方式。

## 好处：

上层的用户代码中，只感知1个类(`TypeInfo`类)，不感知模板（因为在调用的时候，不需要使用模板，所以写代码很方便）。

在`TypeInfo`类中，有几个是函数指针。 使用模板元编程，可以避免在调用的时候，进行各种`switch case的判断`，或者各种虚函数调用。

## `class TypeInfo`

封装一个类型的所有信息。

即给定一个类型：应该能获取到的所有信息，都在该类中封装。

注意：  
对于`decimal`类型，它的“精度”和“有效数字”属性，属于该类型的“属性”(`TypeAttribute`)，**在该类中不包含**。

### 成员

#### `type_`
#### `physical_type_`

在底层存储时，在`C++`语言中，使用的类型。

#### `name_`
#### `size_`
#### `min_value_`
#### `max_value_`
针对当前类型的最大值。 如果有些类型没有最大值（比如: `BINARY`和`STRING`类型），那么`max_value_`的值为`nullptr`;

说明： `BINARY`和`STRING`类型的最小值，是空字符串（""）。

#### `is_virtual_`
 
表示当前`type`，只会在投影(`projection`)中使用。不会在`tablet`的`schema`中使用。

目前只有`IS_DELETE`类型(将来，“虚拟列”可能会包含多种 “列类型”，即可能会增加新类型)。

>> 说明:   
>> 参见(`commit_id:3e61d754c2f91b4293ca8b1178cae681272841a4`)的commit_msg,在该提交中引入的"virtual column"。

"virtual column"的含义是：是指仅仅在Kudu内部使用的列，不会在物理存储时使用。在“create/alter”一个table的时候，不会包含"virtual column"，但是在scan的时候可能会被添加到"projection columns"中。

在当前的kudu实现中，使用了一个逻辑的数据类型(当前只有`IS_DELETE`)来描述"virtual column"。

注意：`IS_DELETE`类型，只是"virtual column"的一种，虽然当前只有这一种，但是将来可能添加更多种类型。

在Kudu进行scan的路径上，如果有一个子系统需要用到"virtual column"，那么它需要：
1. 先确定当前"projectiong"是否包含这个"virtual"的列；
2. 在“投影”时，根据它在"projection"中的定义，这个列的值可能为“默认值”或者"null";
3. 后续由该子系统 负责对它进行填充“有意义的”值。

>> 问题：什么叫做只在“投影”中使用？ 在“投影”中用来干什么？

```
class TypeInfo {
 public:
  DataType type() const { return type_; }
  // Returns the type used to actually store the data.
  DataType physical_type() const { return physical_type_; }
  const std::string& name() const { return name_; }
  const size_t size() const { return size_; }
  void AppendDebugStringForValue(const void *ptr, std::string *str) const;
  int Compare(const void *lhs, const void *rhs) const;
  // Returns true if increment(a) is equal to b.
  bool AreConsecutive(const void* a, const void* b) const;
  void CopyMinValue(void* dst) const {
    memcpy(dst, min_value_, size_);
  }
  bool IsMinValue(const void* value) const {
    return Compare(value, min_value_) == 0;
  }
  bool IsMaxValue(const void* value) const {
    return max_value_ != nullptr && Compare(value, max_value_) == 0;
  }
  bool is_virtual() const { return is_virtual_; }
 private:
  friend class TypeInfoResolver;
  template<typename Type> explicit TypeInfo(Type t);

  const DataType type_;
  const DataType physical_type_;
  const std::string name_;
  const size_t size_;
  const void* const min_value_;
  // The maximum value of the type, or null if the type has no max value.
  const void* const max_value_;
  // Whether or not the type may only be used in projections, not tablet schemas.
  const bool is_virtual_;

  typedef void (*AppendDebugFunc)(const void *, std::string *);
  const AppendDebugFunc append_func_;

  typedef int (*CompareFunc)(const void *, const void *);
  const CompareFunc compare_func_;

  typedef bool (*AreConsecutiveFunc)(const void*, const void*);
  const AreConsecutiveFunc are_consecutive_func_;
};

```

## 实现方式 -- `template<T> class DataTypeTraits`
针对每种类型，都定义了相关的 `DataTypeTraits<T>`的模板特化类。

为了定义衍生类型，会先定义个`DerivedTypeTraits<T>`类型，然后这些“衍生类型”所对应的`DataTypeTraits<T>`，都会继承自`DerivedTypeTraits<K>`(这里`T`表示衍生类型，`K`表示父类型，即衍生自的类型。)

在`DerivedTypeTraits<K>`的实现中，每个方法的实现，都是通过调用它所封装的“非衍生类型”所对应的`DataTypeTraits<K>`类型的相应方法。

>> 问题： 对于“衍生类型”，为什么要采用“先定义出来一个`DerivedTypeTraits<T>`, 然后再继承它的方式”？ 而不是采用“直接继承”的方式，比如让`DataTypeTraits<STRING>`直接继承自`DataTypeTraits<BINARY>`?  语法上是否允许？

### 定义了`cpp_type`子类型

使用`typedef`定义“子类型” `cpp_type`，就是对应的`C++`类型。

每个特化类中，该类型的值不同。 比如`DataTypeTraits<UINT8>::cpp_type`是`uint8_t`; 而`DataTypeTraits<UINT32>::cpp_type`是`uint32_t`。

1. 对于整型：和`C++`的整型一一对应。 `INT128`对应的`__int128`（参见：`src/kudu/util/int128.h`）;
2. 对于浮点型（`FLOAT`和`DOUBLE`）：分别对应`C++`语言中的`float`和`double`；
3. 对于二进制类型（`STRING`和`BINARY`）：对应的是`Slice`。

注意：对于二进制类型，不是对应的`std::string`，而是`Slice`。因为`Slice`内部并不真正分配内存，所以使用的时候，一定要保证这两个类型的值，所对应的内存是有效的。

### 定义成员`physical_type`

类型也是`DataType`，但是 该成员的值 在 “不同的特化类”中取不同的值:
  + 对于“非衍生类型”，该属性的值就是自身类型值；
  + 对于“衍生类型”，该属性的值，是底层真正的类型；比如 `STRING`类型是特化的`BINARY`，所以在`DataTypeTraits<STRING>`的`physical_type`就是`BINARY`。

注意：这里经常提到“类型”、“类型值”等，要注意区分它们的含义。  
比如：`INT8`在kudu中逻辑上表示一个类型，但是实际上是一个值，因为它是`enum`中的一个成员。

### 接口函数

最终会在`TypeInfo`类中，有一系列函数指针成员，指向这些接口函数。

#### `Compare()`

对于任意两个该类型的值，进行比较。

注意：这里没有“跨类型”的比较，都是“相同类型”的。

#### `AreConsecutive()`

>> consecutive: 连贯的；连续不断的

判断两个值是否为连续的。即在该类型上是否是“紧挨着”的。

比如：

判断方法：
1. 对于整数类型: 直接通过数值是否差1来判断。  
    比如两个整形数值，`a`和`b`，如果`b = a + 1`，那么说两个数是紧挨着的。
2. 对于浮点类型（`FLOAT`和`DOUBLE`）：使用`std::nextafter()`来判断是否为“连续的”。  
    参考： http://www.cplusplus.com/reference/cmath/nextafter/ 
3. 对于 字符串类型(`STRING`和`BINARY`)：两者长度差1，并且多了一个`null`字节('\0'字符)；

注意：对于整形数值，需要判断`a+1`是否会溢出。

这里判断溢出的方法是：因为 `a`和`b`都是相同的类型，如果`a`<`b`，那么`a`和`b`之间至少要差1，因为`b`没有溢出，那么`a+1`一定不会溢出。

```
// 判断两个整数类型，是否连续。
template<DataType Type>
static int AreIntegersConsecutive(const void* a, const void* b) {
  typedef typename DataTypeTraits<Type>::cpp_type CppType;
  CppType a_int = UnalignedLoad<CppType>(a);
  CppType b_int = UnalignedLoad<CppType>(b);
  // Avoid overflow by checking relative position first.
  return a_int < b_int && a_int + 1 == b_int;
}
```

注意：这个方法也是可以直接应用于`bool`类型的。

即如果`a`和`b`都是`bool`类型，那么只有一种情况，可以认为`a`和`b`是连续的：即`a == false && b == true`。  
因为`false`会被认为是0，`true`会被认为是1。  
所以仍然可以使用条件`a + 1 == b`来判断。

#### `AppendDebugStringForValue()`

在调试时，打印当前的值。

将当前的值转化为 字符串，添加到参数`str`的末尾。

#### `min_value()`和`max_value()`

获取特定类型的最大最小值。

## 具体实现

### `template<> struct DataTypeTraits<BOOL>`  -- 针对`BOOL`类型的特化类

对于`BOOL`类型来说，`min_value_`为`false`，`max_value_`为`true`。

### `template<> struct DataTypeTraits<BINARY>`  -- 针对`BINARY`类型的特化类

和上面的基本规则 相比，针对`BINARY`类型，特殊地方在于：

1. 子类型`cpp_type`: 使用`Slice`来表示。注意：不是用的`C++`中的基本类型(`char*`)，也不是`std::string`。
2. `AppendDebugStringForValue()`函数：  
    该函数是在调试时进行打印。对于`BINARY`的值，使用“十六进制”进行打印，并且内容前后加上“双引号”。
3. `AreConsecutive()`函数：  
    判断两个“二进制”的值是否“连续”。  
    假定两个值是`a`和`b`，需要同时满足以下条件，才能认为是“连续的”：
    + 两者的长度差1；
    + `b`的最后一个字符为`0`;
    + 不考虑`b`的最后字符，`a`和`b`的内容完全相同。

```
  static bool AreConsecutive(const void* a, const void* b) {
    const Slice *a_slice = reinterpret_cast<const Slice *>(a);
    const Slice *b_slice = reinterpret_cast<const Slice *>(b);
    size_t a_size = a_slice->size();
    size_t b_size = b_slice->size();

    // Strings are consecutive if the larger is equal to the lesser with an
    // additional null byte.

    return a_size + 1 == b_size
        && (*b_slice)[a_size] == 0
        && a_slice->compare(Slice(b_slice->data(), a_size)) == 0;
  }
```
4. `min_value()`
    最小值为 “空字符串”，即什么都没有(`""`)；
5. `max_value()`
    没有最大值。所以该函数返回`nullptr`；

### `template<DataType physicalType> struct DerivedTypeTraits`

“衍生类型”的基类。

目前的“衍生类型”有：
1. `STRING`：衍生自`BINART`;
2. `UNIXTIME_MICROS`: 衍生自`INT64`;
3. `DECIMAL32`: 衍生自`INT32`;
4. `DECIMAL64`: 衍生自`INT64`;
5. `DECIMAL128`: 衍生自`INT128`;

在该“衍生类型的基类”中，所有的方法，都是通过直接调用“依赖的真实类型”的相关函数。

```
template<DataType PhysicalType>
struct DerivedTypeTraits {
  typedef typename DataTypeTraits<PhysicalType>::cpp_type cpp_type;
  static const DataType physical_type = PhysicalType;

  static void AppendDebugStringForValue(const void *val, std::string *str) {
    DataTypeTraits<PhysicalType>::AppendDebugStringForValue(val, str);
  }

  static int Compare(const void *lhs, const void *rhs) {
    return DataTypeTraits<PhysicalType>::Compare(lhs, rhs);
  }

  static bool AreConsecutive(const void* a, const void* b) {
    return DataTypeTraits<PhysicalType>::AreConsecutive(a, b);
  }

  static const cpp_type* min_value() {
    return DataTypeTraits<PhysicalType>::min_value();
  }

  static const cpp_type* max_value() {
    return DataTypeTraits<PhysicalType>::max_value();
  }
  static bool IsVirtual() {
    return DataTypeTraits<PhysicalType>::IsVirtual();
  }
};
```

### `template<> struct DataTypeTraits<STRING>`  -- 针对`STRING`类型的特化类

衍生自`BINARY`。

所以也继承了`DataTypeTraits<BINARY>`的一些属性：
1. `子类型cpp_type`为`Slice`类型；
2. `physical_type`属性的值为`BINARY`。

两者的区别，仅仅是`AppendDebugStringForValue()`方法：
1. 在`BINARY`中，是以“十六进制”的方式进行打印。
2. 在`STRING`中，是以“utf8字符”的方式进行打印。

```
template<>
struct DataTypeTraits<STRING> : public DerivedTypeTraits<BINARY>{
  static const char* name() {
    return "string";
  }
  static void AppendDebugStringForValue(const void *val, std::string *str) {
    const Slice *s = reinterpret_cast<const Slice *>(val);
    str->push_back('"');
    str->append(strings::Utf8SafeCEscape(s->ToString()));
    str->push_back('"');
  }
};
```

### `template<> struct DataTypeTraits<UNIXTIME_MICROS>`  -- 针对`UNIXTIME_MICROS`类型的特化类

衍生自`INT64`类型(有符号，不是`UINT64`)。  

两者的区别，仅仅是`AppendDebugStringForValue()`方法：
1. `INT64`类型，直接打印整形数值；
2. `UNIXTIME_MICROS`类型，打印格式为“YYYY-MM-DDTHH:MM:SS.%06dZ”。

比如时间是: “2019年10月1日 10点5分0秒 123毫秒 456微秒”
那么打印的格式为"2019-10-01 10:05:00.123456Z".

#### `Kudu`中的时间存储格式

分为两部分：
1. 后面的6个“十进制位”用来表示 “微秒”；
2. 前面的所有位用来表示：从`epoch`到现在的秒数（如果为负，表示早于`epoch`）；

因为最后6个“十进制位”表示“微秒”，所以两个该类型的数值（底层对应`int64_t`类型）“差1”，即表示两个时间点“相差1微秒”。

两部分合起来：表示一个“时间点”距离`epoch`的 微秒数。

>> 备注：这部分的实现，可以参照着单测来理解。

因为时间戳部分是从“当前时间点”距离`epoch`的时间差值（即两者的时间差）。

对于负数，应该进行一些转换。

**为什么要进行转换**  

>> 注意：在用类似`time()`函数去获取时间戳时，如果不足1秒的部分，会被忽略。比如当前时间戳是`5.4`秒，那么实际上返回的是`5`秒。

>> 注意：这里有两种事件，各自都有自己的参照点。
+ “计算时间戳”：参照点是一个固定的点，即`epoch`点；
+ “描述时间戳”：即给定一个时间戳，要转化为一个字符串表示形式。参照点不是一个固定的点，是 当前时间点的前一个“时间单位”对应的整数点。

“描述时间戳”时，根据描述的精度，“参照点”是该时间点 **前面的** “整年”、“整月”、“整日”、“整小时”、“整分钟”、“整秒”。 比如`7:12:34`中，“最小时间单位”是“秒”

也就是说，在“描述时间戳”时，说的是“相对于上一个‘最小时间单位’之后，又过了多长时间”。 

这里的关键是：在这两种计算方式下（“计算时间戳”和“描述时间戳”），所采用的“参照点”，是否都在该“时间点”的同一侧。


+ 对于`epoch`之后的“时间戳”，两种计算方式的参照点，都在“时间点”的左侧，所以无需转换； 
+ 但是对于`epoch`之前的“时间戳”（即时间戳数值为负数时），两种计算方式的参照点，不是在同一侧的。所以在描述时要进行转换。

**转换的方法：**  
1. 将“秒数”减一（因为这里的精度就是“秒”）；
2. “微秒数”取值为(1000000 - "微秒数")；

```
-------+---------.-----+-----------.-------------+----> 时间戳
       |         ^     |           |             ^
       |         |    epoch        |             |
       |        v1 点              |            v2点
     描述v1的参照点            描述v2的参照点
         
```
如上图：
1. 对于`v1`点：“计算时间戳的‘参照点’(`epoch`)”和“描述时间戳时使用的‘参照点’”，在 `v1` 的两侧，需要进行转换；
2. 对于`v2`点：“计算时间戳的‘参照点’(`epoch`)”和“描述时间戳时使用的‘参照点’”，在 `v2` 同一侧，不需要进行转换；

注意：如果该值是一个负数，表示它是`epoch`之前的某个时间点。

**强调：**  
也正是因为这种表示方式，可以看出在`Kudu`中，最小的时间精度为“微秒”。

一些接口方法的说明：
1. `AreConsecutive()`： 两个时间戳是连续的，是指“相差1微秒”；
2. `min_value()`: 因为`int64`的最小值为`int64_t::MIN`；
3. `max_value()`: 因为`int64`的最小值为`int64_t::MAX`，


```
static const char* kDateFormat = "%Y-%m-%dT%H:%M:%S";
static const char* kDateMicrosAndTzFormat = "%s.%06dZ";

template<>
struct DataTypeTraits<UNIXTIME_MICROS> : public DerivedTypeTraits<INT64>{
  static const int US_TO_S = 1000L * 1000L;

  static const char* name() {
    return "unixtime_micros";
  }

  static void AppendDebugStringForValue(const void* val, std::string* str) {
    int64_t timestamp_micros = *reinterpret_cast<const int64_t *>(val);
    time_t secs_since_epoch = timestamp_micros / US_TO_S;
    // If the time is negative we need to take into account that any microseconds
    // will actually decrease the time in seconds by one.
    int remaining_micros = timestamp_micros % US_TO_S;
    if (remaining_micros < 0) {
      secs_since_epoch--;
      remaining_micros = US_TO_S - std::abs(remaining_micros);
    }
    struct tm tm_info;
    gmtime_r(&secs_since_epoch, &tm_info);
    char time_up_to_secs[24];
    strftime(time_up_to_secs, sizeof(time_up_to_secs), kDateFormat, &tm_info);
    char time[34];
    snprintf(time, sizeof(time), kDateMicrosAndTzFormat, time_up_to_secs, remaining_micros);
    str->append(time);
  }
};

```

### `template<> struct DataTypeTraits<DECIMAL32>`  -- 针对`DECIMAL32`类型的特化类 

衍生自`INT32`类型。 两者的区别，仅仅是`AppendDebugStringForValue()`方法：

因为要完整的输出一个`DECIMAL`类型，要知道它的“有效数字”和“精度”信息。而这里没有这个信息。

所以在打印的时候，这里直接打印了底层对应的`int32_t`类型，然后打印了`_D32`作为后缀，来标识它是一个`DECIMAL32`类型。

```

template<>
struct DataTypeTraits<DECIMAL32> : public DerivedTypeTraits<INT32>{
  static const char* name() {
    return "decimal";
  }
  // AppendDebugStringForValue appends the (string representation of) the
  // underlying integer value with the "_D32" suffix as there's no "full"
  // type information available to format it.
  static void AppendDebugStringForValue(const void *val, std::string *str) {
    DataTypeTraits<physical_type>::AppendDebugStringForValue(val, str);
    str->append("_D32");
  }
};
```

### `template<> struct DataTypeTraits<DECIMAL64>`  -- 针对`DECIMAL64`类型的特化类 

衍生自`INT64`类型。 两者的区别，仅仅是`AppendDebugStringForValue()`方法：

在打印调试信息时，和`DataTypeTraits<DECIMAL32>`类似，无法打印完整信息。

这里的打印的是，底层对应的整型(`int64`)值和`_D64`后缀。

### `template<> struct DataTypeTraits<DECIMAL128>`  -- 针对`DECIMAL128`类型的特化类 

衍生自`INT128`类型。 两者的区别，仅仅是`AppendDebugStringForValue()`方法：

在打印调试信息时，和`DataTypeTraits<DECIMAL32>`类似，无法打印完整信息。

这里的打印的是，底层对应的整型(`int128`)值和`_D128`后缀。

### `template<> struct DataTypeTraits<IS_DELETED>`  -- 针对`IS_DELETED`类型的特化类 

注意是`BOOL`类型的衍生类，即继承自`DerivedTypeTraits<BOOL>`（该类是封装的`DataTypeTraits<BOOL>`）。

标识一个“无效类型”。

另外，只有该类的`IsVirtual()`返回`true`。标识该类型只能在“投影”中使用。

## `template<T> struct TypeTraits`

继承自`DataTypeTraits<T>`，自然也继承了`DataTypeTraits`的多个方法。

增加了1个属性和1个方法。

1. `type`: 当前的`DataType`值。
2. `size()`: 所使用的底层C++数据类型的大小，这里就是返回`sizeof(cpp_type)`。 

注意：对于`STRING`和`BINARY`类型，它们的`cpp_type`是`Slice`类型。那么它们的`size()`返回值为`sizeof(Slice)`，这是一个固定值。
即无论`Slice`的数据内容是什么，`size()`的返回值都是一样的。

也就是说，对于这两种类型，`TypeInfo::size()`并不能得到它们的“值大小”（对于其它类型，是可以的）。

>> 问题： 因为只是增加了一个`type`属性和`size()`方法，为什么不直接在`DataTypeTraits<T>`中定义？然后这个类就可以不要了。

## `class Variant`

封装一个 特定数据类型和对应的值。

### `union NumbericValue`

注意是`union`类型，其中的值只能有1个，不是枚举。

```
  union NumericValue {
    bool     b1;
    int8_t   i8;
    uint8_t  u8;
    int16_t  i16;
    uint16_t u16;
    int32_t  i32;
    uint32_t u32;
    int64_t  i64;
    uint64_t u64;
    int128_t i128;
    float    float_val;
    double   double_val;
  };
```

### 成员

#### `type_`

对应的数据类型（`DataType`类型）；

#### `numeric_`

如果是数值类型(整型、浮点型、布尔型)时，用该变量来保存对应的值。

#### `vstr_`

`Slice`类型。 如果是`BINARY`或`STRING`类型，使用该属性保存值。

注意：这是一个`Slice`对象。 在对当前`Variant`丢向重新赋值时:
1. 如果之前是一个`STRING`或`BINARY`对象，那么需要手动释放（见`Clear()`方法）之前在`Slice`中的内存；
2. 如果新值是一个`STRING`或`BINARY`对象，那么需要进行字符串拷贝。（具体指向的数据地址，就是在当前类中`new`出来的，参见`Reset()`方法）

注意：一个`Variant`对象，所储存的值的类型是可变的。比如：一个指向`INT64`类型的`Variant`对象，可以通过`Reset()`方法修改为指向`STRING`类型。

每次`Reset()`时，旧值会被清空（如果之前是`Slice`对象，那么对应的内存空间会被`delete`掉）。

### 方法
#### `private Clear()`

用来清空当前对象。   

目前清理的内容只有一个：当类型为`STRING`或`BINARY`时，所引用的字符串内存，需要被释放掉。

而且的两个数据成员，因为在重新赋值时，都会被覆盖掉，所以不需要被清理。

在重新对当前`Variant`对象进行赋值时，会调用该方法。

```
  void Clear() {
    // No need to delete[] zero-length vstr_, because we always ensure that
    // such a string would point to a constant "" rather than an allocated piece
    // of memory.
    if (vstr_.size() > 0) {
      delete[] vstr_.mutable_data();
      vstr_.clear();
    }
  }
```

注意：如果当前字符串的长度为0，那么是不需要调用`delete`进行清理的。

因为如果 字符串为空，那么实际上 `vstr_`是引用了一个常量的空字符串（因为这个字符串，并不是被`new`出来的，所以不需要被`delete`）。

这个是通过调用`Slice->clear()`来保证的。在其中，会将`Slice`的指针指向一个“空字符串”。

## `private class TypeInfoResolver`

在`cpp`文件中实现，只是一个内部类。

是一个**单例类**，用来维护一个`DateType`=> `TypeInfo<DataType>`的映射。

## `GetTypeInfo()` -- 全局函数

给定一个`DataType`，用来获取它对应的`TypeInfo`对象。

调用单例类`TypeInfoResolver`的`GetTypeInfo()`接口，获得每个`DataType`对应的`TypeInfo`对象指针。



