## neon中的数据结构

neon的基本类型定义与`arm_neon.h`中, 有:

```c++
typedef __Uint8x16_t uint8x16_t;
typedef __Uint16x8_t uint16x8_t;
typedef __Uint32x4_t uint32x4_t;
typedef __Uint64x2_t uint64x2_t;
......
```

其中, `uint8x16_t`指的是16个uint8_t的类型; 其他的类似.

奇怪的是, 在gcc的include文件中找不到`__Uint8x16_t`的定义, 并且, 下面这段代码也可以编译通过:

```c++
//// 注意这里没有include任何头文件
//// 测试环境为: gcc 5.4.0, Linux firefly 4.4.77
int main(int argc, char *argv[]) {
  __Uint8x16_t num;  //__ 消除atom高亮
  return 0;
}
```

这说明`__Uint8x16_t`是内置类型, 通过编译器的提示, `__Uint8x16_t`的类型为:
`__vector(16) unsigned char`, 其他的可以据此推断出来.

除了基本类型之外, neon也定义了数组类型. 数组类型就是基本类型的数组.
比如`uint8x16_t`对应的数组类型有如下几个:

```c++
typedef struct uint8x16x2_t {
  uint8x16_t val[2];
} uint8x16x2_t;

typedef struct uint8x16x3_t {
  uint8x16_t val[3];
} uint8x16x3_t;

typedef struct uint8x16x4_t {
  uint8x16_t val[4];
} uint8x16x4_t;
```

对于基本类型及其数组, 使用上, 可以将它们看成是128字节或者256字节的数据块,
比如, 可以用下面的方法对其进行输出:

```c++
#include <string>

static inline std::string pad_left(const std::string& str,
                                   const size_t length,
                                   const char value = ' ') {
  if (str.length() >= length) { return str; }
  return std::string(length - str.length(), value) + str;
}

template <class T> std::string to_string(const T* value, const int num) {
  const int elem_length = 4;
  std::string vstr = std::to_string(value[0]);
  std::string str = pad_left(vstr, elem_length);
  for (int i = 1; i < num; ++i) {
    vstr = std::to_string(value[i]);
    str += "," + pad_left(vstr, elem_length);
  }
  return "(" + str + ")";
}

template <> inline std::string to_string(const int8_t* value, const int num) {
  const int elem_length = 4;
  std::string vstr = std::to_string(static_cast<int>(value[0]));
  std::string str = pad_left(vstr, elem_length);
  for (int i = 1; i < num; ++i) {
    vstr = std::to_string(static_cast<int>(value[i]));
    str += "," + pad_left(vstr, elem_length);
  }
  return "(" + str + ")";
}

template <> inline std::string to_string(const uint8_t* value, const int num) {
  const int elem_length = 4;
  std::string vstr = std::to_string(static_cast<int>(value[0]));
  std::string str = pad_left(vstr, elem_length);
  for (int i = 1; i < num; ++i) {
    vstr = std::to_string(static_cast<int>(value[i]));
    str += "," + pad_left(vstr, elem_length);
  }
  return "(" + str + ")";
}

template <class T> std::string to_string(const T* value,
                                         const int rows,
                                         const int cols) {
  std::string str = to_string(value, cols);
  for (int i = 1; i < rows; ++i) {
    value += cols;
    str += ",\n " + to_string(value, cols);
  }
  return "(" + str + ")";
}

#define TO_STRING(TYPE, VALUE, NUM) \
  to_string<TYPE>((const TYPE*) (&(VALUE)), (NUM)).c_str()

#define TO_STRING2(TYPE, VALUE, COLS, ROWS) \
  to_string<TYPE>((const TYPE*) (&(VALUE)), (ROWS), (COLS)).c_str()

////  使用方法:
uint8x16_t num1;
LOG(INFO) << TO_STRING(uint8_t, num1, 16);
uint8x16x3_t num2;
LOG(INFO) << TO_STRING(uint8_t, num1, 16, 3);
```

## neon的加速比

 一个用普通c++写的程序, 如果编译时用`-O3`选项, 则编译器会试图将操作矢量化.
 这将部分抵消neon intrinsics的优势, 故而加速比会显著低于理论值.

如果有两个程序, 实现同样的功能, 一个用普通的c++写的, 一个用neon实现的, 在编译选项`-O3`下,
gcc(版本5.4.0)的加速比一般不高, 大约1.12左右, clang(版本6.0.0)的加速比可以达到1.71,
说明对于neon intrinsics来说, clang的支持比gcc的更好. 对于Intel平台下的sse也有类似的结论.

根据一些帖子如: [ARM NEON Optimization](http://hilbert-space.de/?p=22),
[An Introduction to ARM NEON](http://peterdn.com/post/an-introduction-to-ARM-NEON.aspx)
的实验结果, 直接用neon汇编会比intrinsics要快很多, 但是汇编与平台相关, 较为复杂, 除非万不得已,
否则还是要慎重.

## 其他资源

1. [Neon Intrinsics各函数介绍](https://blog.csdn.net/fengbingchun/article/details/38085781)

2. [Ne10](https://github.com/projectNe10/Ne10)
