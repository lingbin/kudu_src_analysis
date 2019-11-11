[TOC]

文件：`src/kudu/common/key_encoder.h`

# 概述
如“文件名字”所述，当前文件中的类，都是为了对一个`key`列进行编码。（如果是对`value`列进行编码，和本文件中的类无关）

# `template struct KeyEncoderTraits<...>`

模板类，针对每种`DataType`，都会有自己的特化类。

注意1：在本类中，类的内容是空的（没有提供任何接口和方法）。 在它的各种特化类中，会提供相应的接口。

注意2：在它的各个特化类中，所提供的接口都是`static`方法。  
所以在使用时，不需要“实例化”相应的`KeyEncoder`类，直接用“类名”来调用即可。

注意3：这里的模板参数`Buffer`类型。具体的“类型”为如下两种之一：`std::string`或`faststring`。


```
// 定义一个空的`Traits`，其中每个`DataType`都会有一个特化。

template<DataType Type, typename Buffer, class Enable = void>
struct KeyEncoderTraits {
};

```

## 针对“整型”的特化类

这里利用了`模板的SFINAE特性`，只有整型才能成功特化这个类型。

该类都是`static`方法，没有成员。

```
template<DataType Type, typename Buffer>
struct KeyEncoderTraits<Type,
                        Buffer,
                        typename base::enable_if<
                          base::is_integral<
                            typename DataTypeTraits<Type>::cpp_type
                          >::value
                        >::type
                       > {
...                          
}
```

### 子类型

#### `cpp_type`
对应`DataType`对应的`cpp_type`。

#### `unsigned_cpp_type`
当前整型对应的“无符号整型”。

```
  typedef typename DataTypeTraits<Type>::cpp_type cpp_type;
  typedef typename MathLimits<cpp_type>::UnsignedType unsigned_cpp_type;
```

### 接口列表

#### `private static SwapEndian()`

将一个“整型” 从“小端序” 转换为 “大端序”。

```
  static unsigned_cpp_type SwapEndian(unsigned_cpp_type x) {
    switch (sizeof(x)) {
      case 1: return x;
      case 2: return BigEndian::FromHost16(x);
      case 4: return BigEndian::FromHost32(x);
      case 8: return BigEndian::FromHost64(x);
      case 16: return BigEndian::FromHost128(x);
      default: LOG(FATAL) << "bad type size of: " << sizeof(x);
    }
    return 0;
  }
```

#### `static Encode()`

有两个重载函数。

1. 一个是传入`cpp_type`，
2. 一个是传入`void*`（实际也是`cpp_type*`类型）。

注意：经常的使用该方法 是这样的：   
`encoder.Encode(row.cell_ptr(column_idx), i + 1 == column_ids.size(), buf);`  
即传入的参数，本质上就是在“`Row`的`row_data_`中”的类型；

而对于整形，在`Row`的`row_data_`中，就是按照该类型对应的`cpp_type`进行保存的。

注意2： 对于上面的使用方法，第2个参数`i + 1 == column_ids.size()`，用来标识当前字段是否为“最后一个字段”。  
实际上，对于整型，因为不需要进行填充，这个参数是没有用的。

**注意：对于一个整形，这里的编码方法能够保证：“编码是保序的”。**  
即如果`a < b`，那么在编码后`str_a < str_b`。  

无论`a`, `b`是整数还是负数。

**编码方法：**  
1. 使用“大端序”进行编码；
2. 如果是“有符号整形”：
    + 对于正数：将首个bit位置为`1`;
    + 对于负数：会将首个bit位置为`0`；

>> 附录： “大小端”的区别，可以参见： https://www.cnblogs.com/luxiaoxun/archive/2012/09/05/2671697.html

```
例1： int32_t val = Ox12345678

    地址         0x0000    0x0001    0x0002    0x0003
    小端序：       78       56         34        12   -> 00010010 ，符号位为0
    转变符号位：   78       56         34        92   -> 10010010 ，符号位变为1
    转化为大端：   92       34         56        78  

例2： uint32_t val = Ox12345678

    地址         0x0000    0x0001    0x0002    0x0003
    小端序：       78       56         34        12   -> 00010010 ，符号位为0
    无符号类型，不需要转换“符号位”
    转化为大端：   12       34         56        78

例3： int32_t val = -1

    地址         0x0000    0x0001    0x0002    0x0003
    小端序：       FF        FF        FF        FF   -> 11111111 ，符号位为1
    转变符号位：   FF        FF        FF        7F   -> 01111111 ，符号位变为0
    转化为大端：   7F        FF        FF        FF
    
例4： int32_t val = -2

    地址         0x0000    0x0001    0x0002    0x0003
    小端序：       FF        FF        FF        FE   -> 11111110 ，符号位为1
    转变符号位：   FF        FF        FF        7E   -> 01111110 ，符号位变为0
    转化为大端：   7E        FF        FF        FF
```

**说明：为什么是保序的**  
对于“无符号整型”，转化为“大端序”以后，一定是保序的；

这里只需要分析下“有符号整型”：
1. 两个负数：从上面的例子中，比如 `-1`和`-2`的编码结果，在负数上是保序的；
2. 两个正数：和“无符号整型”一样，大端序一定是保序的；
3. 一正一负：因为正数在编码后，首位（"符号位"）变为`1`，负数在编码后,首位变为`0`，所以在编码后，“正数的字节序”一定是大于“负数的字节序”。

```
  static void Encode(cpp_type key, Buffer* dst) {
    Encode(&key, dst);
  }

  static void Encode(const void* key_ptr, Buffer* dst) {
    unsigned_cpp_type key_unsigned;
    memcpy(&key_unsigned, key_ptr, sizeof(key_unsigned));

    // To encode signed integers, swap the MSB.
    if (MathLimits<cpp_type>::kIsSigned) {
      key_unsigned ^= static_cast<unsigned_cpp_type>(1) << (sizeof(key_unsigned) * CHAR_BIT - 1);
    }
    key_unsigned = SwapEndian(key_unsigned);
    dst->append(reinterpret_cast<const char*>(&key_unsigned), sizeof(key_unsigned));
  }

```

说明1：对于“有符号整型”，转换首个"bit"位的方法：  
利用”位操作”，将原来的数值与 二进制`1000...000` 进行“异或”操作。  

说明2： `CHAR_BIT`表示“1个字节中的bit位数”  
参见： https://zh.cppreference.com/w/cpp/types/climits  

>> 问题：这里将`key_ptr`的值，赋值给`key_unsigned`，是使用的`memcpy()`，如果直接使用`reinterpret_cast<>`来进行复制？

#### `static EncodeWithSeparators()`

注意：对于整型，因为是定长的，所以在编码的时候并**不填充“分隔符”**。所以，这里直接调用`Encode()`来实现。

```
  static void EncodeWithSeparators(const void* key, bool is_last, Buffer* dst) {
    Encode(key, dst);
  }
```

#### `static DecodeKeyPortion()`

用来对“编过码的key”进行解码，结果放入到`cell_ptr`参数中。

注意1：因为整型在`Row`的`row_data_`中直接存储值的，所以这里不会用到`arena`参数的（用来存储`STRING`和`BINARY`所“间接引用”的数据）。

注意2：因为“整型”在编码时是不填充“分隔符”的，所以这里也用不到`is_last`参数。

注意3：在解码完以后，会修改传入的参数`Slice* encoded_key`：会将已经解析过的部分去掉。（通过`Slice::remove_prefix()`）

```
  static Status DecodeKeyPortion(Slice* encoded_key,
                                 bool /*is_last*/,
                                 Arena* /*arena*/,
                                 uint8_t* cell_ptr) {
    if (PREDICT_FALSE(encoded_key->size() < sizeof(cpp_type))) {
      return Status::InvalidArgument("key too short", KUDU_REDACT(encoded_key->ToDebugString()));
    }

    unsigned_cpp_type val;
    memcpy(&val,  encoded_key->data(), sizeof(cpp_type));
    val = SwapEndian(val);
    if (MathLimits<cpp_type>::kIsSigned) {
      val ^= static_cast<unsigned_cpp_type>(1) << (sizeof(val) * CHAR_BIT - 1);
    }
    memcpy(cell_ptr, &val, sizeof(val));
    encoded_key->remove_prefix(sizeof(cpp_type));
    return Status::OK();
  }
```

## 针对“BINARY”的特化类

和“针对整型进行特化”一样，该类提供的接口都是`static`方法，没有成员。

```
template<typename Buffer>
struct KeyEncoderTraits<BINARY, Buffer> {
  ...
}
```

### 子类型

```
  static const DataType key_type = BINARY;
```

### 接口列表

#### `static Encode()`
有两个重载函数。

1. 一个是传入`Slice`，
2. 一个是传入`void*`（实际也是`Slice*`类型）。

注意1：经常的调用该方法是这么调用的：   
`encoder.Encode(row.cell_ptr(column_idx), i + 1 == column_ids.size(), buf);`  
即传入的参数，本质上就是在`Row`的`row_data_`中的类型；

对于`BINARY`和`STRING`，在`Row`的`row_data_`中，就是按照该类型对应的`cpp_type`进行保存的。

注意2： 对于上面的使用方法，第2个参数`i + 1 == column_ids.size()`，用来标识当前字段是否为“最后一个字段”。  
实际上，对于整型，因为不需要进行填充，这个参数是没有用的。


```
 static void Encode(const void* key, Buffer* dst) {
    Encode(*reinterpret_cast<const Slice*>(key), dst);
  }

  // simple slice encoding that just adds to the buffer
  inline static void Encode(const Slice& s, Buffer* dst) {
    dst->append(reinterpret_cast<const char*>(s.data()),s.size());
  }
```

注意：对于`BINARY`和`STRING`类型来说，使用`KeyEncoder::Encode()`方法，默认调用的不是这里的`Encode()`方法，而是下面的`EncodeWithSeparators()`方法。即 **对于这两种类型，是需要填充“分隔符”的**。

#### `static EncodeWithSeparators()`

对于`BINARY`和`STRING`类型，是变长的，**需要在编码的时候填充“分隔符”**。

**为了能够保存的进行编码，一定要填充“分隔符”**  
```
比如：有 col1 和col2 两列
假定有两行数据:
        row1 = ("aa", "bb") 
        row2 = ("aabb", "")
        
    这里如果逐列比较，结果很明显： row1 < row2

但是如果编码的时候，不填充“分隔符”，那么这两行编出来的数据是一样的，都是"aabb"，
那么就无法区分两行的先后顺序。 即 **这种编码方法，无法做到保序**。
```

**如何填充分隔符**  

首先：为了解决上面例子中的“保序”问题，所填充的“分隔符”要比所有的"字节值"都要小，即必须是`\x00`才可以。


说明：“填充的字符，要比其它所有字符小”，只要这样，填充以后，该字符才会不会影响排序效果。参加下面的例子。

```
比如：有 col1 和col2 两列
假定有两行数据:
        row1 = (\x1234, \x00) 
        row2 = (\x123400, "")
        
    这里如果逐列比较，结果很明显： row1 < row2

如果填充的“分割符”不是最小的，即不是`\x00`，假如分隔符为`\xmm` ( > \x00)，那么填充的结果为： 
       encoded_row1 = (\x1234mm00)
       encoded_row2 = (\x123400mm)
    
    可见编码后的结果中，encoded_row1 > encoded_row2，即编码后的顺序和之前是不同的。
```

其次：因为原始数据中可能有的字节值为`\x00`，所以使用一个`\x00`还不够：
1. 使用`\x00\x00`来作为“分隔符”。（共两个字节）
2. 如果原始数据中的`\x00`，转为`\x00\x01`。（是为了保证“转化后的字符”比“分割字符(`\x00\x01`)”要大）

```
比如：有 col1 和col2 两列
假定有两行数据:
        row1 = (\x1234, \x0056) 
        row2 = (\x123400, "56")
        
这里如果逐列比较，结果很明显： row1 < row2
    
  这里 原始数据中有`\x00`，使用单个字节的 `\x00`作为分割符，也是有问题的。
    
    使用`\x00`（单个字节）作为分隔符：
       encoded_row1 = (\x1234000056)
       encoded_row2 = (\x1234000056)
    编码结果是 encoded_row1 > encoded_row2
    
    
  而如果使用`\x00\x00`（两个字节）作为分隔符，并且把原始数据中的`\x00`替换为`\x00\x01`：
  （注意：对于最后一个列，是不需要进行这个转化的。  
          即对于`row1`中的`\x0056`，不需要将其中的`\x00`转化为`\x00\x00`），
    说明：下面的结果中，用来填充的`\x0000`，使用了单引号 括起来。
       encoded_row1 = (\x1234 '0000' 0000 56)
       encoded_row2 = (\x1234 0001 '0000' 56)
       
    可见编码之后是保序的： encoded_row1 < encoded_row2
```

注意：如果当前要编码的列，是此次编码的最后一个列（即`is_last == true`)，那么**既不需要在后面追加“分隔符”，也不需要转化`\x00`**，直接添加到末尾即可。  

```
  static void EncodeWithSeparators(const void* key, bool is_last, Buffer* dst) {
    EncodeWithSeparators(*reinterpret_cast<const Slice*>(key), is_last, dst);
  }

  inline static void EncodeWithSeparators(const Slice& s, bool is_last, Buffer* dst) {
    if (is_last) {
      dst->append(reinterpret_cast<const char*>(s.data()), s.size());
    } else {
      int old_size = dst->size();
      dst->resize(old_size + s.size() * 2 + 2);

      const uint8_t* srcp = s.data();
      uint8_t* dstp = reinterpret_cast<uint8_t*>(&(*dst)[old_size]);
      int len = s.size();
      int rem = len;

      while (rem >= 16) {
        if (!SSEEncodeChunk<16>(&srcp, &dstp)) {
          goto slow_path;
        }
        rem -= 16;
      }
      while (rem >= 8) {
        if (!SSEEncodeChunk<8>(&srcp, &dstp)) {
          goto slow_path;
        }
        rem -= 8;
      }
      // Roll back to operate in 8 bytes at a time.
      if (len > 8 && rem > 0) {
        dstp -= 8 - rem;
        srcp -= 8 - rem;
        if (!SSEEncodeChunk<8>(&srcp, &dstp)) {
          // TODO: optimize for the case where the input slice has '\0'
          // bytes. (e.g. move the pointer to the first zero byte.)
          dstp += 8 - rem;
          srcp += 8 - rem;
          goto slow_path;
        }
        rem = 0;
        goto done;
      }

      slow_path:
      EncodeChunkLoop(&srcp, &dstp, rem);

      done:
      *dstp++ = 0;
      *dstp++ = 0;
      dst->resize(dstp - reinterpret_cast<uint8_t*>(&(*dst)[0]));
    }
  }
```

说明：这里的实现，对于“原始数据”中没有`\0`的情况作了很多优化，因为认为这是一个最常见的场景。

注意：这里的`Buffer`类型，是模板参数。具体的类型为如下两种之一：`std::string`或`faststring`。

#### `static DecodeKeyPortion()`

用来对“编过码的key”进行解码，结果放入到`cell_ptr`参数中。

注意1：因为`BINARY`和`STRING`类型在`Row`的`row_data_`中存储的是`Slice`对象，所以`cell_ptr`实际指向的对象是`Slice`类型。

注意2：`cell_ptr`(是一个`Slice`对象)，它所“间接引用”的数据是在`arena`中的。

注意3：因为数据在编码的时候，原始数据中的`\x00`被转换为`\x00\x01`，所以在解码的时候，需要检测并还原。

注意4：如果当前列是最后一列（即`is_last == true`)，在编码的时候并“没有”进行转化`\x00`，所以在解码的时候，如果`is_last == true`，也是“不需要”进行检测`\x00`的。

注意5：在解码完以后，会修改传入的参数`Slice* encoded_key`：会将已经解析过的部分去掉。（通过`Slice::remove_prefix()`）

```
  static Status DecodeKeyPortion(Slice* encoded_key,
                                 bool is_last,
                                 Arena* arena,
                                 uint8_t* cell_ptr) {
    if (is_last) {
      Slice* dst_slice = reinterpret_cast<Slice *>(cell_ptr);
      if (PREDICT_FALSE(!arena->RelocateSlice(*encoded_key, dst_slice))) {
        return Status::RuntimeError("OOM");
      }
      encoded_key->remove_prefix(encoded_key->size());
      return Status::OK();
    }

    uint8_t* separator = static_cast<uint8_t*>(memmem(encoded_key->data(), encoded_key->size(),
                                                      "\0\0", 2));
    if (PREDICT_FALSE(separator == NULL)) {
      return Status::InvalidArgument("Missing separator after composite key string component",
                                     KUDU_REDACT(encoded_key->ToDebugString()));
    }

    uint8_t* src = encoded_key->mutable_data();
    int max_len = separator - src;
    uint8_t* dst_start = static_cast<uint8_t*>(arena->AllocateBytes(max_len));
    uint8_t* dst = dst_start;

    for (int i = 0; i < max_len; i++) {
      if (i >= 1 && src[i - 1] == '\0' && src[i] == '\1') {
        continue;
      }
      *dst++ = src[i];
    }

    int real_len = dst - dst_start;
    Slice slice(dst_start, real_len);
    memcpy(cell_ptr, &slice, sizeof(Slice));
    encoded_key->remove_prefix(max_len + 2);
    return Status::OK();
  }
```

说明1：`memmem()`函数的作用：类似于`strstr()`，用于在一块内存中寻找匹配另一块内存的内容的第一个位置。

说明2：如果要解码的列不是“最后一列”，那么必须能够找到分隔符（`\x00\x00`）,否则，说明当前的“编码结果”有问题，报错；

说明3：在解码的过程中，如果有遇到`\x00\x01`，那么需要转化为`\x00`。

说明4: 在解码完一列之后，如果该列不是“最后一列”，那么在去除头部的“已经解析过”的内容时，要在本次解析的长度上加上2。原因是要去掉“分隔符”(`\x00\x00`，是两个字节)。


#### `private SSEEncodeChunk()`

使用`SSE`函数（向量化）来拷贝字符串，在编码的时候使用。

模板参数`<LEN>`有两个可能的取值： `8` 和 `16`。

返回值：  
1. 如果当前字符串中是`\x00`，那么函数返回`false`（上层会转化为普通的方式`EncodeChunkLoop()`）。
2. 如果当前字符串中没有`\x00`，那么该函数返回`true`，表示成功进行了拷贝。


```
  template<int LEN>
  static bool SSEEncodeChunk(const uint8_t** srcp, uint8_t** dstp) {
    COMPILE_ASSERT(LEN == 16 || LEN == 8, invalid_length);
    __m128i data;
    if (LEN == 16) {
      // Load 16 bytes (unaligned) into the XMM register.
      data = _mm_loadu_si128(reinterpret_cast<const __m128i*>(*srcp));
    } else if (LEN == 8) {
      // Load 8 bytes (unaligned) into the XMM register
      data = reinterpret_cast<__m128i>(_mm_load_sd(reinterpret_cast<const double*>(*srcp)));
    }
    // Compare each byte of the input with '\0'. This results in a vector
    // where each byte is either \x00 or \xFF, depending on whether the
    // input had a '\x00' in the corresponding position.
    __m128i zeros = reinterpret_cast<__m128i>(_mm_setzero_pd());
    __m128i zero_bytes = _mm_cmpeq_epi8(data, zeros);

    // Check whether the resulting vector is all-zero.
    bool all_zeros;
    if (LEN == 16) {
      all_zeros = _mm_testz_si128(zero_bytes, zero_bytes);
    } else { // LEN == 8
      all_zeros = _mm_cvtsi128_si64(zero_bytes) == 0;
    }

    // If it's all zero, we can just store the entire chunk.
    if (PREDICT_FALSE(!all_zeros)) {
      return false;
    }

    if (LEN == 16) {
      _mm_storeu_si128(reinterpret_cast<__m128i*>(*dstp), data);
    } else {
      _mm_storel_epi64(reinterpret_cast<__m128i*>(*dstp), data);  // movq m64, xmm
    }
    *dstp += LEN;
    *srcp += LEN;
    return true;
  }
```

#### `private EncodeChunkLoop()`

```
  // Non-SSE loop which encodes 'len' bytes from 'srcp' into 'dst'.
  static void EncodeChunkLoop(const uint8_t** srcp, uint8_t** dstp, int len) {
    while (len--) {
      if (PREDICT_FALSE(**srcp == '\0')) {
        *(*dstp)++ = 0;
        *(*dstp)++ = 1;
      } else {
        *(*dstp)++ = **srcp;
      }
      (*srcp)++;
    }
  }
```

# `template class EncoderResolver`

一个**单例类**，用来维护一个从`DataType`到`KeyEncoder`对象的映射。 

注意1：当前类仅在`.cpp`文件中定义。只供在本`.cpp`文件中定义的全局函数使用(`GetKeyEncoder()`)。

注意2：这个类是模板类，模板参数是`std::string`或者`faststring`。所以在进程中，会存在两个`EncoderResolver`对象：`EncoderResolver<std::string>`和`EncoderResolver<faststring>`。

## 成员

```
  vector<unique_ptr<KeyEncoder<Buffer>>> encoders_;
```

**说明：这里使用的数据结构是`std::vector`，而不是`std::unordered_map`。**  

原因是：
1. 从该结构中获取指定类型对应的`KeyEncoder`对象，会在一些“关键路径”上使用，所以对性能的要求比较高；
2. 数据类型是很少的，并且它们的值都比较小，所以使用`vector`的元素数量很少，是可控的。

**注意：在这个`std::vector`中，每种数据类型所对应的`KeyEncoder`类，在该列表中的 下标 就是 `DataType` 的值。**  

## 接口列表

### `template GetKeyEncoder()`

指定一个数据类型，获取对应的`KenEncoder`对象。

是一个模板函数，模板参数是`std::string`或者`faststring`。在调用时，根据不同的模板参数，分别去向不同类型的`EncoderResolver<T>`单例对象中去查找。

```
  const KeyEncoder<Buffer>& GetKeyEncoder(DataType t) {
    DCHECK(HasKeyEncoderForType(t));
    return *encoders_[t];
  }
```
### `HasKeyEncoderForType()`

是否预先注册了 某个`DataType`对应的`KeyEncoder`对象。

```
  const bool HasKeyEncoderForType(DataType t) {
    return t < encoders_.size() && encoders_[t];
  }
```

说明：`encoders`的成员是`std::unique_ptr<>`类型，这里的`encoders_[t]`实际上检查的是，这个智能指针是否为`nullptr`。

### `private AddMapping()`

在该类的“构造函数”中，对于每种“可以作为`key`的类型”，都会添加一个映射关系。

# 全局函数
## `IsTypeAllowableInKey()`  

检查某个类型是否可以作为`key`列；（在创建表等接收到来自用户的`schema`信息时，会进行检查）。

判断的方法：
如果某个类型存在对应的`KeyEncoder`对象，那么就是可以的。

说明1：因为当前类就是`KeyEncoder`，从名字就可以看出来，是用来对`key`列进行编码的。  
也就是说，如果某个类型可以作为`key`列，那么一定存在对应的`KeyEncoder`对象。  

说明2：因为对于`EncoderResolver<>`的两种模板参数，返回的结果是一样的，所以当前函数就没有必要定义为模板函数，内部直接使用`EncoderResolver<faststring>`来进行判断。

说明3：目前`kudu`支持的类型中，只有`float`和`double`、`DECIMAL`类型是“不支持”用作`key`的。

```
const bool IsTypeAllowableInKey(const TypeInfo* typeinfo) {
  return Singleton<EncoderResolver<faststring>>::get()->HasKeyEncoderForType(
      typeinfo->physical_type());
}
```

## `GetKeyEncoder()` 

获取指定的`DataType`所对应的`KeyEncoder`对象。

注意：是模板函数。模板参数是`std::string`或者`faststring`。  
在调用时，根据不同的模板参数，分别去向不同类型的`EncoderResolver<T>`单例对象中去查找。

```
template <typename Buffer>
const KeyEncoder<Buffer>& GetKeyEncoder(const TypeInfo* typeinfo) {
  return Singleton<EncoderResolver<Buffer>>::get()->GetKeyEncoder(typeinfo->physical_type());
}
```


# `template class KeyEncoder`

对`key`列的“编码器”。

## 成员

共有`3`个函数成员

### `encode_func_`
编码的函数。

### `encode_with_separators_func_`
添加“分隔符”的“编码函数”。

### `decode_key_portion_func_`
解码函数

```
  typedef void (*EncodeFunc)(const void* key, Buffer* dst);
  const EncodeFunc encode_func_;
  
  typedef void (*EncodeWithSeparatorsFunc)(const void* key, bool is_last, Buffer* dst);
  const EncodeWithSeparatorsFunc encode_with_separators_func_;

  typedef Status (*DecodeKeyPortionFunc)(Slice* enc_key, bool is_last,
                                       Arena* arena, uint8_t* cell_ptr);
  const DecodeKeyPortionFunc decode_key_portion_func_;
```

## 接口列表

### 构造函数

```
  template<typename EncoderTraitsClass>
  explicit KeyEncoder(EncoderTraitsClass t)
    : encode_func_(EncoderTraitsClass::Encode),
      encode_with_separators_func_(EncoderTraitsClass::EncodeWithSeparators),
      decode_key_portion_func_(EncoderTraitsClass::DecodeKeyPortion) {
  }
```

### `Encode()`

根据不同的类型，会调用不同的`Encode()`方法。

1. 对于`整型`，不需要填充分隔符，都是调用“第`1`个”。
2. 对于`STRING`和`BINARY`，需要填充分隔符，调用“第`2`个”

```
  // Encodes the provided key to the provided buffer
  void Encode(const void* key, Buffer* dst) const {
    encode_func_(key, dst);
  }

  // Special encoding for composite keys.
  void Encode(const void* key, bool is_last, Buffer* dst) const {
    encode_with_separators_func_(key, is_last, dst);
  }
```

### `ResetAndEncode()`
```
  void ResetAndEncode(const void* key, Buffer* dst) const {
    dst->clear();
    Encode(key, dst);
  }
```

### `Decode()`

将`encoded_key`中的数据，进行解码，将结果保存在`cell_ptr`中。

注意1：在解码完以后，会修改传入的参数`Slice* encoded_key`：会将已经解析过的部分去掉。（通过`Slice::remove_prefix()`）

注意2：如果是“复合主键”的最后一列，那么`is_last`的值应该为`true`;

注意3：对于有间接引用数据的类型（`Binary`），那么它所间接引用的数据，是在`arena`参数中分配的。

```
  Status Decode(Slice* encoded_key,
                bool is_last,
                Arena* arena,
                uint8_t* cell_ptr) const {
    return decode_key_portion_func_(encoded_key, is_last, arena, cell_ptr);
  }
```
















