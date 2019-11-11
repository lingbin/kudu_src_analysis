[TOC]

文件：`src/kudu/util/bitmap.h`

## 概述

该文件主要声明了两方便的内容：
1. 定义了一些列的工具函数；
2. `BitmapIterator`类；

`bitmap`的分布方式：
1. 在“字节间”，使用正序排列。
2. 在“字节内”，使用的是“倒序”排列。即按照“76543210”的顺序。

```
    |-------|-------|-------|-------|-------|-------|-------|
    0       8       16      24      32      40      48      56
    ^      ^
    |      |
    76543210
    
给定一个idx，那么：
1. 它所对应的字节是： bitmap[idx >> 3]
2. 它在那个字节内，所对应的bit位是： 1 << (idx & 7)


其实，在对应的字节内部，该bit位的逻辑序号就是 (idx & 7)，那么获取对应的mask（将指定一段区间的值设置为`1`）的方法：

   |-------|
   ^      ^
   76543210
   
   上面是一个字节的分布，将这个“逻辑顺序”记为`x`(x的取值范围是`[0, 7]`)，并假如 x = 3
   
1. 将序号范围 [x, 8)的字节都设置为 `1`:
   即
       11111---|
       ^   ^  ^
       7654x210
   方法是： 0xff << (idx & 7)
   
2. 将序号范围 [0, x)的字节都设置为 `1`:(注意不包含`x`)
   即
       |----111|
       ^   ^  ^
       7654x210
   方法一： 0xff >> (8 - (idx & 7))
   方法二： (1 << x) - 1
```

## 工具函数列表

### `BitmapSize()`

给定一个数量，计算如果一个“位图”能够表示这么多的比特位，需要多少个字节。

```
inline size_t BitmapSize(size_t num_bits) {
  return (num_bits + 7) / 8;
}
```

### `BitmapSet()`

给定一个序号，将对应的位设置为`1`。

```
inline void BitmapSet(uint8_t *bitmap, size_t idx) {
  bitmap[idx >> 3] |= 1 << (idx & 7);
}
```

### `BitmapChange()`

给定一个“序号”和“目标值”，将对应的位设置为指定的 value。

```
// Switch the given bit to the specified value.
inline void BitmapChange(uint8_t *bitmap, size_t idx, bool value) {
  bitmap[idx >> 3] = (bitmap[idx >> 3] & ~(1 << (idx & 7))) | ((!!value) << (idx & 7));
}
```

说明1：   
`!!value` 的作用(注意有2个'`!`')： 因为`value`是一个`bool`类型，“对它取反”可以转化为一个整形，所以这里通过“两次取反”将它转化为一个整型。 
+ 原值为`true`，转化为`1`;
+ 原值为`false`，转化为`0`;

说明2:   
前半部分 `(bitmap[idx >> 3] & ~(1 << (idx & 7)))`是将对应的位“清零”。

注意：如何将一个“位”修改为指定的值：
1. 不能只简单地使用“按位相与”操作。  
    比如：原值为`0`，要设置为`1`，但是 “按位相与”的结果也是`0`，所以不符合要求。
2. 不能只简单地使用“按位相或”操作。
    比如：原值为`1`，要设置为`0`，但是 “按位相或”的结果也是`1`，所以不符合要求。
3. 不能只简单地使用“按位相 异或”操作。
    比如：原值为`1`，要设置为`1`，但是 “按位相异或”的结果也是`0`，所以不符合要求。

正确的方法有几种：(注意目标值，可能会`0`，也可能为`1`)

**方法一：**  
1. 先将 指定的位清零； 
2. 然后进行与“目标值”进行“按位相或”操作。

因为`0`与“任何目标值`val`” 进行“按位相或”，结果仍是`val`；

**方法二：**  
1. 先将 指定的位 置`1`； 
2. 然后进行与“目标值”进行“按位相与”操作。

因为`1`与“任何目标值`val`” 进行“按位相与”，结果仍是`val`；

**方法三：**  
1. 如果“目标值”为`1`，那么就进行“按位或”操作；
2. 如果“目标值”为`0`，那么就进行“按位与”操作

如果目标值为`1`，因为无论“原值”是多少，“按位或”的结果都是`1`;  
如果目标值为`0`，因为无论“原值”是多少，“按位与”的结果都是`0`;  

>> 说明1：在“方法三”中，会有一个“分支判断”。但是在前两种方法中，是没有分支判断的。

### `BitmapClear()`

将指定bit位的值“清零”。

```
inline void BitmapClear(uint8_t *bitmap, size_t idx) {
  bitmap[idx >> 3] &= ~(1 << (idx & 7));
}
```

### `BitmapTest()`

给定一个比特位，测试它的值是否为1。

如果为1，返回true；否则，返回false；

**注意：该方法等价于 `BitMapGet()`。** （如果该bit位的值为`1`，就返回`1`，而这里返回的`true`，转化为整型，值就是`1`） 

>> 说明： 正是因为因为它们是等价的，所以并没有再单独实现`BitMapGet()`方法。

```
inline bool BitmapTest(const uint8_t *bitmap, size_t idx) {
  return bitmap[idx >> 3] & (1 << (idx & 7));
}
```

### `BitmapMergeOr()`

将两个`bitmap`按照 “按位相或”的方式进行合并。

参数`n_bits`，表示参与计算的`bit`位数。

```
// Merge the two bitmaps using bitwise or. Both bitmaps should have at least
// n_bits valid bits.
inline void BitmapMergeOr(uint8_t *dst, const uint8_t *src, size_t n_bits) {
  size_t n_bytes = BitmapSize(n_bits);
  for (size_t i = 0; i < n_bytes; i++) {
    *dst++ |= *src++;
  }
}
```

**使用场景：将两个`bitmap`取“并集”**

### `BitmapChangeBits()`

将一段bit位，设置成对应的值。

设置的区间是：`[offset, offset + num_bits)`。注意是“左闭右开”区间。

实现方法：如下图，分为3个部分。
```
                       |<1>| <---- 2 ----> |<3>|     
   |-------|-------|---++++|+++++++|+++++++|+++----|-------|-------|
                       ^                       ^
                       |                       |
                    offset               offset + num_bits
                    
注意：因为在每个字节内，bit位的顺序是按照逆序(即`76543210`)的顺序排列的，所以在物理上的分布是：
                   |<1>|   |<---- 2 -----> |   |<3>|     
   |-------|-------|++++---|+++++++|+++++++|----+++|-------|-------|
                       ^                       ^
                       |                       |
                    offset               offset + num_bits
```

```
void BitmapChangeBits(uint8_t *bitmap, size_t offset, size_t num_bits, bool value) {
  DCHECK_GT(num_bits, 0);

  size_t start_byte = (offset >> 3);
  size_t end_byte = (offset + num_bits - 1) >> 3;
  int single_byte = (start_byte == end_byte);

  // Change the last bits of the first byte
  size_t left = offset & 0x7;
  size_t right = (single_byte) ? (left + num_bits) : 8;
  uint8_t mask = ((0xff << left) & (0xff >> (8 - right)));
  if (value) {
    bitmap[start_byte++] |= mask;
  } else {
    bitmap[start_byte++] &= ~mask;
  }

  // Nothing left... I'm done
  if (single_byte) {
    return;
  }

  // change the middle bits
  if (end_byte > start_byte) {
    const uint8_t pattern8[2] = { 0x00, 0xff };
    memset(bitmap + start_byte, pattern8[value], end_byte - start_byte);
  }

  // change the first bits of the last byte
  right = offset + num_bits - (end_byte << 3);
  mask = (0xff >> (8 - right));
  if (value) {
    bitmap[end_byte] |= mask;
  } else {
    bitmap[end_byte] &= ~mask;
  }
}
```

### `BitmapFindFirst()`

从指定的偏移量开始，查找是否存在指定的值。

如果存在，函数的返回值为`true`，所对应的下标 是从 `idx`参数返回。没找到时，返回`false`。

```
    00000000000000000000000000000000000001000000000000000000
    |-------|-------|-------|-------|-------|-------|-------|
               ^                         ^                  ^
               |                         |                  |
             offset                      |               bitmap_size
                                        idx
```

```
// Find the first bit of the specified value, starting from the specified offset.
bool BitmapFindFirst(const uint8_t *bitmap, size_t offset, size_t bitmap_size,
                     bool value, size_t *idx);
                     

bool BitmapFindFirst(const uint8_t *bitmap, size_t offset, size_t bitmap_size,
                     bool value, size_t *idx) {
  const uint64_t pattern64[2] = { 0xffffffffffffffff, 0x0000000000000000 };
  const uint8_t pattern8[2] = { 0xff, 0x00 };
  size_t bit;

  DCHECK_LE(offset, bitmap_size);

  // Jump to the byte at specified offset
  const uint8_t *p = bitmap + (offset >> 3);
  size_t num_bits = bitmap_size - offset;

  // Find a 'value' bit at the end of the first byte
  if ((bit = offset & 0x7)) {
    for (; bit < 8 && num_bits > 0; ++bit) {
      if (BitmapTest(p, bit) == value) {
        *idx = ((p - bitmap) << 3) + bit;
        return true;
      }

      num_bits--;
    }

    p++;
  }

  // check 64bit at the time for a 'value' bit
  const uint64_t *u64 = (const uint64_t *)p;
  while (num_bits >= 64 && *u64 == pattern64[value]) {
    num_bits -= 64;
    u64++;
  }

  // check 8bit at the time for a 'value' bit
  p = (const uint8_t *)u64;
  while (num_bits >= 8 && *p == pattern8[value]) {
    num_bits -= 8;
    p++;
  }

  // Find a 'value' bit at the beginning of the last byte
  for (bit = 0; num_bits > 0; ++bit) {
    if (BitmapTest(p, bit) == value) {
      *idx = ((p - bitmap) << 3) + bit;
      return true;
    }
    num_bits--;
  }

  return false;
}
```

说明：为了提高效率，对于中间一段区间的检查，使用“数值比较”的方式。

1. 如果超过`64`个bit位，那么使用`uint64_t`来比较。
2. 小于`64`个bit位，但大于`8`个bit位时，使用`uint8_t`来比较。

### `BitmapFindFirstSet()`

从指定的偏移量开始，找到第一个值为`1`的bit位。

```
// Find the first set bit in the bitmap, at the specified offset.
inline bool BitmapFindFirstSet(const uint8_t *bitmap, size_t offset,
                               size_t bitmap_size, size_t *idx) {
  return BitmapFindFirst(bitmap, offset, bitmap_size, true, idx);
}
```

### `BitmapFindFirstZero()`
从指定的偏移量开始，找到第一个值为`0`的bit位。

```
// Find the first zero bit in the bitmap, at the specified offset.
inline bool BitmapFindFirstZero(const uint8_t *bitmap, size_t offset,
                                size_t bitmap_size, size_t *idx) {
  return BitmapFindFirst(bitmap, offset, bitmap_size, false, idx);
}
```

### `BitmapIsAllSet()`
从某个序号开始，检查是否所有`bit`位都已经被设置为`1`了。

```
// Returns true if the bitmap contains only ones.
inline bool BitmapIsAllSet(const uint8_t *bitmap, size_t offset, size_t bitmap_size) {
  DCHECK_LE(offset, bitmap_size);
  size_t idx;
  return !BitmapFindFirstZero(bitmap, offset, bitmap_size, &idx);
}
```

逻辑：如果没有找到`0`，那么就说明“全是`1`”。

### `BitmapIsAllZero()`

从某个序号开始，检查是否所有`bit`位都已经被设置为`0`了。

```
// Returns true if the bitmap contains only zeros.
inline bool BitmapIsAllZero(const uint8_t *bitmap, size_t offset, size_t bitmap_size) {
  DCHECK_LE(offset, bitmap_size);
  size_t idx;
  return !BitmapFindFirstSet(bitmap, offset, bitmap_size, &idx);
}
```
逻辑：如果没有找到`1`，那么就说明“全是`0`”。


### `BitmapEquals()`

检查两个`bitmap`，查看是否相等。

注意：这里有个假设的前提：两个`bitmap`的`size`是相同的（即含有相同数量的`bit`位）。

```
inline bool BitmapEquals(const uint8_t* bm1, const uint8_t* bm2, size_t bitmap_size) {
  // Use memeq() to check all of the full bytes.
  size_t num_full_bytes = bitmap_size >> 3;
  if (!strings::memeq(bm1, bm2, num_full_bytes)) {
    return false;
  }

  // Check any remaining bits in one extra operation.
  size_t num_remaining_bits = bitmap_size - (num_full_bytes << 3);
  if (num_remaining_bits == 0) {
    return true;
  }
  DCHECK_LT(num_remaining_bits, 8);
  uint8_t mask = (1 << num_remaining_bits) - 1;
  return (bm1[num_full_bytes] & mask) == (bm2[num_full_bytes] & mask);
}
```

说明：比较分为两部分：
1. “整字节”的部分： 直接使用`strings::memeq()`进行计较。
2. “尾部多余的少数bit位”：将这些`bit`位整体转化为一个整型，然后再进行比较。（这样的好处是，只需要比较一次）

```
   bitmap_1: xxxxyyyy xxxxyyyy xxxxyyyy xxxxyyyy abcd
   bitmap_2: xxxxyyyy xxxxyyyy xxxxyyyy xxxxyyyy abcd
            |  使用`strings::memeq()`进行比较  |   ^  
                                                   |
                                尾部的部分，小于1个字节，转化为整型，再进行比较 
```

### `BitmapCopy()`

将一个`bitmap`从`src`拷贝到`dst`中。   

说明1：在`src`中，从`src_offset`位置开始拷贝，在`dst`中从`dst_offset`位置开始接受“拷贝”过来的数据。

说明2：拷贝的`bit`总位数为`num_bits`。

注意：`src`和`dst`之间一定不能重叠。

```
void BitmapCopy(uint8_t* dst, size_t dst_offset,
                const uint8_t* src, size_t src_offset,
                size_t num_bits) {
  DCHECK_GT(num_bits, 0);

  // Prohibit overlap in debug builds.
#ifndef NDEBUG
  const uint8_t* src_start = src + (src_offset >> 3);
  const uint8_t* src_end = src + ((src_offset + num_bits) >> 3);
  uint8_t* dst_start = dst + (dst_offset >> 3);
  uint8_t* dst_end = dst + ((dst_offset + num_bits) >> 3);
  DCHECK(src_start >= dst_end || dst_start >= src_end)
      << "Source and destination overlap";
#endif

  // If the copy is entirely byte-aligned, we can use memcpy.
  if ((src_offset & 7) == 0 && (dst_offset & 7) == 0 && (num_bits & 7) == 0) {
    memcpy(dst + (dst_offset >> 3),
           src + (src_offset >> 3),
           BitmapSize(num_bits));
    return;
  }

  // Otherwise, we fallback to a slower bit-by-bit copy.
  //
  // TODO(adar): this can be optimized using word-by-word operations.
  for (size_t bit_num = 0; bit_num < num_bits; bit_num++) {
    BitmapChange(dst, dst_offset + bit_num, BitmapTest(src, src_offset + bit_num));
  }
}
```

说明1：在非`NDEBUG`模式下，会检查`src`和`dst`之间是否重叠。

如下图，如果两个“区间”不重合，那么必然是如下两种情况。  
``` 
场景1：
                             range1             range2
    ---------------------|<--------->|-------|<--------->|------->
    
场景2：
             range2          range1         
    ------|<------->|----|<--------->|--------------------------->
    
```

说明：因为这里的区间都是“左闭右开”区间，所以在判断的时候，判断时是有包含等于的。比如`src_start >= dst_end`。

说明2：如果在`src`和`dst`中，都是按照“字节对齐”的，那么可以直接使用`memcpy()`来进行拷贝。

可以进行`memcpy()`的条件：
1. 在`src`和`dst`中的“起始位置”都是`8`的倍数。
2. 拷贝的长度，也是`8`的倍数

判断是否为`8`的倍数（即判断是否“字节对齐”）：直接与`7`进行“按位相与”计算，如果结果为`0`，那么就是`8`的倍数。

说明3: 如果不是“直接对齐”的，那么就降级为比较慢的方式：逐个`bit`位进行拷贝。
实际上就是先用`BitmapTest()`获取`src`中相应bit位的值，然后调用`BitmapChange()`方法对`dst`中的相应bit位进行赋值。

### `BitmapToString()`
将一个bitmap转化为字符串。

```
string BitmapToString(const uint8_t *bitmap, size_t num_bits) {
  string s;
  size_t index = 0;
  while (index < num_bits) {
    StringAppendF(&s, "%4zu: ", index);
    for (int i = 0; i < 8 && index < num_bits; ++i) {
      for (int j = 0; j < 8 && index < num_bits; ++j) {
        StringAppendF(&s, "%d", BitmapTest(bitmap, index));
        index++;
      }
      StringAppendF(&s, " ");
    }
    StringAppendF(&s, "\n");
  }
  return s;
}
```

说明：打印的格式为：
1. 在每行的首部打印偏移量（宽度为`4`）；
2. 每行打印`8`个字节；
3. 每个字节包含`8`个bit位；
4. 每个bit位的值为`0`或`1`;

```
// example:
0000: 01010101 01010101 01010101 01010101 01010101 01010101 01010101 01010101
0064: 01010101 01010101 01010101 01010101 01010101 01010101 01010101 01010101
0128: 01010101 01010101 01010101 01010101 01010
```

## `class BitmapIterator`
位图的“迭代器”。

它每次调用`Next()`，是跳过一段连续的相同值。

比如：
当前指向的bit位值为`1`,那么调用`Next()`，执行的操作是，从当前bit位开始向后查找，直到找到第一个为`0`的bit位。  
再次调用`Next()`，那么就会返回下一个为`1`的bit位。

也就是说：
1. 如果当前bit为的值为`1`，那么调用`Next()`后指向`0`;
2. 如果当前bit为的值为`1`，那么调用`Next()`后指向`1`;

### 使用方法：
```
// Example usage:
//   bool value;
//   size_t size;
//   BitmapIterator iter(bitmap, n_bits);
//   while ((size = iter.Next(&value))) {
//      printf("bitmap block len=%lu value=%d\n", size, value);
//   }
```
### 成员

#### `offset_`
当前指向位置，在整个bitmap中的“偏移量”。

#### `num_bits_`
当前`bitmap`的大小。

#### `map_`
用来存储`bitmap`的数据。

注意：当前`BitmapIterator`并不`own`这段内存。即在当前`BitmapIterator`对象析构的时候，并不会将这段内存释放掉。

```
  size_t offset_;
  size_t num_bits_;
  const uint8_t *map_;
```

### 接口列表
#### `done()`
是否已经达到了终点。

#### `SeekTo()`
将“当前位置”的指针，定位到指定的“偏移量”的位置。

```
  bool done() const {
    return (num_bits_ - offset_) == 0;
  }

  void SeekTo(size_t bit) {
    DCHECK_LE(bit, num_bits_);
    offset_ = bit;
  }
```

#### `Next()`
获取“下一个位置”。

说明：
1. 参数`value`：在调用该方法后，该值会被赋值为 当前`bit`位的值；
2. 返回值：在当前`Next()`指向期间，跳过的`bit`位数。

注意1：会跳过所有的 值等于‘当前bit位的值’的所有bit位。

注意2：该函数的返回值，表示当前指针 前进了 多少个`bit`位。

注意3：如果已经到达了终点，那么返回值为“所有剩余的bit位数”。  
      本地调用的含义是：经过了`剩余bit位`的`value`值。

注意：参见“使用方法”，在本函数返回的值为`0`时，结束循环。

```
// 1. 当前的状态
    01010101 01010101 00000000 00000000 01111111 11111111 
                      ^
                      |
                     curr
                     
// 2. 调用`Next()`
    01010101 01010101 00000000 00000000 01111111 11111111 
                                         ^
                                         |
                                        curr

    这时，`Next()`函数的返回值为`16`，表示`bit`指针，前进了`16`个bit位。
    
    本地调用的含义是：遇到了`16`个连续的`0`;

// 3. 再次调用`Next()`，因为找不到`0`，所以会走到终点
    01010101 01010101 00000000 00000000 01111111 11111111 
                                                         ^
                                                         |
                                                        curr
    这时，`Next()`函数的返回值为`15`，表示`bit`指针，前进了`15`个bit位。
    
    本地调用的含义是：遇到了`15`个连续的`1`;

// 4. 再次调用`Next()`，因为已经达到位图的末尾，所以直接返回`0`;(参见“使用举例”部分，循环体就结束了)
    
    
注意：上面只是逻辑表示，在实际的“位图”表示中，在每个字节内，是“从右向左”进行遍历的。
```

```
  size_t Next(bool *value) {
    size_t len = num_bits_ - offset_;
    if (PREDICT_FALSE(len == 0))
      return(0);

    *value = BitmapTest(map_, offset_);

    size_t index;
    if (BitmapFindFirst(map_, offset_, num_bits_, !(*value), &index)) {
      len = index - offset_;
    } else {
      index = num_bits_;
    }

    offset_ = index;
    return len;
```

## `class TrueBitIterator`
迭代所有的`1`。

和`BitmapIterator`类似，每次调用`Next()`，也可能会跳过一段`bit位`。

注意：和`BitmapIterator`不同的是：当前类是跳过所有的`0`。而`BitmapIterator`在后续的bit位中，跳过所有 “值等于当前bit位的值”的 bit位。

### 使用举例
```

// Iterator which yields the set bits in a bitmap.
// Example usage:
//   for (TrueBitIterator iter(bitmap, n_bits);
//        !iter.done();
//        ++iter) {
//     int next_onebit_position = *iter;
//   }
```

说明：这里的“使用方式”和`BitmapIterator`是不同的。

说明：该类虽然叫做“迭代器”，但是该类并没有提供`Next()`方法，而是使用`operator++()`来向前移动。

### 成员

#### `bitmap_`
用来存储`bitmap`的数据。

注意：当前`TrueBitIterator`对象并不`own`这段内存。  
即在当前`TrueBitIterator`对象析构的时候，并不会将这段内存释放掉。

#### `cur_byte_`
在位图中，当前位置所处的“字节”的值。

注意：这里是“字节”的值，并不是`bit位`的值。

注意2：随着`位置`的移动，这个值是在不断变化的。即使只是在“字节内”移动，该值也是变化的。

注意3：在每一个有效位置，该值的最后一位，一定是为`1`。 即`cur_byte & 1`的值一定为`true`。

在迭代的过程中，该变量的初始值为当前字节的值。如果在当前字节内前进的时候，该值会将逐渐的“右移”。

说明：在代码实现中，就是通过检测`if(cur_byte_ == 0)`来检查，是否需要跳过当前“字节”。

```
// 1. 当前的状态
                      |<-----|
    01010101 01010101 00001001 00000000 01010101 01010101 
                             ^
                             |
                            curr
                           
    这时`cur_byte_` == 00001001
    
// 2. 将它前进一位

                      |<-----|
    01010101 01010101 00001001 00000000 01010101 01010101 
                          ^
                          |
                         curr
    
    这时`cur_byte_` == 00000001
```

#### `cur_byte_idx_`
当前位置，处于“第几个”字节。

#### `n_bits_`
“位图”的总长度，即有多少个`bit`位。

#### `n_bytes_`
“位图”中，一共有多少个“字节”。

#### `bit_idx_`
当前位置，处于“位图”中的第几个`bit位`。

```
  const uint8_t *bitmap_;
  uint8_t cur_byte_;
  uint8_t cur_byte_idx_;

  const size_t n_bits_;
  const size_t n_bytes_;
  size_t bit_idx_;
```

### 接口列表

#### 构造函数

注意：如果“位图”的大小为`0`，那么直接将`cur_byte_idx_`的值置为`1`，表示已经迭代结束（参见`done()`的实现，如果`cur_byte_idx_ >= n_bytes_`，那么就会被认为是已经遍历完毕）。

注意2：在该对象构造出来以后，立即就会指向第一个值为`1`的bit位。（即在构造函数中，就会调用`AdvanceToNextOneBit()`）方法。

```
TrueBitIterator(const uint8_t *bitmap, size_t n_bits)
    : bitmap_(bitmap),
      cur_byte_(0),
      cur_byte_idx_(0),
      n_bits_(n_bits),
      n_bytes_(BitmapSize(n_bits_)),
      bit_idx_(0) {
    if (n_bits_ == 0) {
      cur_byte_idx_ = 1; // sets done
    } else {
      cur_byte_ = bitmap[0];
      AdvanceToNextOneBit();
    }
  }
```

#### `operator++()`
将当前迭代器前进，指向下一个`1`。

#### `done()`

```
  TrueBitIterator &operator ++() {
    DCHECK(!done());
    DCHECK(cur_byte_ & 1);
    cur_byte_ &= (~1);
    AdvanceToNextOneBit();
    return *this;
  }

  bool done() const {
    return cur_byte_idx_ >= n_bytes_;
  }

  size_t operator *() const {
    DCHECK(!done());
    return bit_idx_;
  }
```

#### `operator*()`
参见“使用举例”，该函数的返回值，表示“当前位置”在“位图”中的偏移量。

#### `private AdvanceToNextOneBit()`

```
 void AdvanceToNextOneBit() {
    while (cur_byte_ == 0) {
      cur_byte_idx_++;
      if (cur_byte_idx_ >= n_bytes_) return;
      cur_byte_ = bitmap_[cur_byte_idx_];
      bit_idx_ = cur_byte_idx_ * 8;
    }
    DVLOG(2) << "Found next nonzero byte at " << cur_byte_idx_
             << " val=" << cur_byte_;

    DCHECK_NE(cur_byte_, 0);
    int set_bit = Bits::FindLSBSetNonZero(cur_byte_);
    bit_idx_ += set_bit;
    cur_byte_ >>= set_bit;
  }
```











