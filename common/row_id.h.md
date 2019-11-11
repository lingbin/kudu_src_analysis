[TOC]

文件： `src/kudu/common/row_id.h`

在一个`RowSet`中，描述一行的“顺序号”。

## `rowid_t`

```
typedef uint32_t rowid_t;
```

之所以重新定义了这个类型`rowid_t`，而不是在代码中直接使用`uint32_t`，是为了代码更清晰。

注意：因为是`uint32_t`类型(取值范围是: `[0, 4294967295]`)，所以目前在一个`RowSet`中，最多只能有`4B`行。 （即40亿条）


## `EncodeRowId()`

将一个`rowid_t`类型，放入到一个`faststring`类型的buff中。

**编码格式：**  
直接按照 内存格式 进行编码。即x86的“小端”格式。

**编码以后，仍然可以使用`memcmp()`进行比较**  
因为`rowid_t`是无符号的整数（一定不是负数），它的内存方式是可以直接使用`memcmp()`进行比较的。

```
inline void EncodeRowId(faststring *dst, rowid_t rowid) {
  PutMemcmpableVarint64(dst, rowid);
}
```

>> 问题：`rowid_t`实际上是`uint32_t`类型，那为什么编码的时候，却使用`64`位整型的方式进行编码？

## `DecodeRowId()`

从一个`Slice`中，解析出`rowid_t`类型。

注意：在调用这个方法之后，`Slice`对象会被修改：其中的指针对前进，到`rowid_t`的下一个字符。

如果解码失败，返回`false`。

```
inline bool DecodeRowId(Slice *s, rowid_t *rowid) {
  uint64_t tmp;
  bool ret = GetMemcmpableVarint64(s, &tmp);
  DCHECK_LT(tmp, 1ULL << 32);
  *rowid = tmp;
  return ret;
}
```

说明：因为`rowid_t`实际上是`uint32_t`类型，而编码的时候，使用的是`uint64_t`的方式进行编码的。 所以这里在非`NDEBUG`模式下（会进行`DCHECK`），会检查这个“解码”出来的值，一定要 小于等于 `UINT_MAX`。

检查的方法是：将`1ULL`(`uint64_t`类型)左移`32`位，即它的值是`UINT32_MAX + 1`。

注意：这里进行移位的是`1ULL`，其中`"ULL"`的后缀，表示它是一个`uint64_t`类型。不能直接写为`1 << 32`。 因为字面量`1`（没有后缀），会被解析成`int32_t`类型，而左移`32`位，得到的是`0`。（因为`int32_t`有32个bit位，每次左移会右端补`0`，左移`32`位，即得到一个`32`个bit位都是`0`的数，即值为`0`）。



