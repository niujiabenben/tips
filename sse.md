## sse中的数据结构

gcc中, `__m128i`被定义为长度为2的long long型数组, `__m128d`被定义为长度为2的double型数组,
`__m128`被定义为长度为4的float型数组:

```c++
//// 头文件: emmintrin.h
typedef long long __m128i __attribute__ ((__vector_size__ (16), __may_alias__));
typedef double __m128d __attribute__ ((__vector_size__ (16), __may_alias__));
//// 头文件: xmmintrin.h
typedef float __m128 __attribute__ ((__vector_size__ (16), __may_alias__));
```

使用上, 可以将它们当成16个byte的数据块, 不过其首地址必须是align到16字节.

可以用下面的函数将其转成字符串输出:

```c++
#include <emmintrin.h>
#include <tmmintrin.h>
#include <string>

template <class T> std::string to_string(const __m128i& value) {
  const int N = 16 / sizeof(T);
  const T* tp = (T*) (&value);
  //// first value
  //// 一般我们不会用到long long, 这里cast到long就可以了.
  long val = static_cast<long>(tp[0]);
  std::string str = std::to_string(val);
  //// the rest
  for (int i = 1; i < N; ++i) {
    long val = static_cast<long>(tp[i]);
    str += ", " + std::to_string(val);
  }
  return "(" + str + ")";
}

//// 编译选项: g++ -g -Wall -fpic -O2 -std=c++11 -msse4
```

## SSE中的shuffle函数

sse中, 针对`__m128i`数据结构, 有如下的shuffle的函数:

```c++
//// 头文件: emmintrin.h
extern __m128i _mm_shufflehi_epi16(__m128i __A, const int __mask);
extern __m128i _mm_shufflelo_epi16(__m128i __A, const int __mask);
extern __m128i _mm_shuffle_epi32(__m128i __A, const int __mask);
//// 头文件: tmmintrin.h
extern __m128i _mm_shuffle_epi8(__m128i __X, __m128i __Y);
```

拿`_mm_shuffle_epi8`来举例, 可以用它来分离cv::Mat数据中的rgb值:

```c++
int main(int argc, char *argv[]) {
  //// 1, 2, 3分别表示B, G, R
  __m128i origin = _mm_setr_epi8(1, 2, 3, 1, 2, 3, 1, 2, 3, 1, 2, 3, 1, 2, 3, 1);
  //// order_*中每一个值对应origin中的位置, -1表示shuffle之后的结果中该位置为0
  __m128i order_b = _mm_setr_epi8(0, 3, 6, 9,  12, 15, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1);
  __m128i order_g = _mm_setr_epi8(1, 4, 7, 10, 13, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1);
  __m128i order_r = _mm_setr_epi8(2, 5, 8, 11, 14, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1);
  __m128i all_b = _mm_shuffle_epi8(origin, order_b);
  __m128i all_g = _mm_shuffle_epi8(origin, order_g);
  __m128i all_r = _mm_shuffle_epi8(origin, order_r);
  std::cout << "b: " << to_string<char>(all_b) << std::endl;
  std::cout << "g: " << to_string<char>(all_g) << std::endl;
  std::cout << "r: " << to_string<char>(all_r) << std::endl;
  return 0;
}

//// 输出
b: (1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
g: (2, 2, 2, 2, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
r: (3, 3, 3, 3, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
```

但是对于16位数据来说, 只有`_mm_shufflehi_epi16`和`_mm_shufflehi_epi16`两个函数可用,
分别shuffle高64位和低64位数据, 而高位和低位之间的数据却无法交换. 这篇文章
[SSE系列内置函数中的shuffle函数](https://www.cnblogs.com/quarryman/p/sse_shuffle.html)
详细讨论了shuffle系列函数的原理, 从而给出了由`_mm_shuffle_epi8`去实现16位数据的shuffle的功能.

实际上, 我们可以把16位数据看成是2个8位的, 其位置对应关系为:

| 16位id |  0   |  1   |  2   |  3   |   4  |   5    |   6    |   7   |
|:------:|:----:|:----:|:----:|:----:|:----:|:------:|:------:|:-----:|
| 8位id  | 0, 1 | 2, 3 | 4, 5 | 6, 7 | 8, 9 | 10, 11 | 12, 13 | 14, 15|

这样我们就可以通过6位的shuffle函数去shuffle16位的数据了. 但是在实际操作中, 16位的id更为直观,
拆成8位的id之后, 数目变大了一倍, 顺序也不好记忆, 很容易就搞乱了. 这里我们给出一个方法:
设计一个宏, 先找出16位id对应的两个8位id(也就是`2*id`和`2*id+1`), 然后将两个8位的id进行组合,
成为一个16位的数, 最后用`_mm_setr_epi16`来设置`order`数组:

```c++
//// 这里实际上是由 (i * 2 + 1) * 256 + (i * 2) 简化而来,
//// 注意Intel平台一般采用的是small ending.
#define MM_SHUFFLE_EPI16(i) (((i) << 9) + (1 << 8) + ((i) << 1))

int main(int argc, char *argv[]) {
  //// 1, 2, 3分别表示B, G, R
  __m128i origin = _mm_setr_epi16(1, 2, 3, 1, 2, 3, 1, 2);
  __m128i order_b = _mm_setr_epi16(
      MM_SHUFFLE_EPI16(0), MM_SHUFFLE_EPI16(3), MM_SHUFFLE_EPI16(6), -1, -1, -1, -1, -1);
  __m128i order_g = _mm_setr_epi16(
      MM_SHUFFLE_EPI16(1), MM_SHUFFLE_EPI16(4), MM_SHUFFLE_EPI16(7), -1, -1, -1, -1, -1);
  __m128i order_r = _mm_setr_epi16(
      MM_SHUFFLE_EPI16(2), MM_SHUFFLE_EPI16(5), -1, -1, -1, -1, -1, -1);
  __m128i all_b = _mm_shuffle_epi8(origin, order_b);
  __m128i all_g = _mm_shuffle_epi8(origin, order_g);
  __m128i all_r = _mm_shuffle_epi8(origin, order_r);
  std::cout << "b: " << to_string<short>(all_b) << std::endl;
  std::cout << "g: " << to_string<short>(all_g) << std::endl;
  std::cout << "r: " << to_string<short>(all_r) << std::endl;
  return 0;
}

//// 输出
b: (1, 1, 1, 0, 0, 0, 0, 0)
g: (2, 2, 2, 0, 0, 0, 0, 0)
r: (3, 3, 0, 0, 0, 0, 0, 0)
```

注意`order*`中我们不想要值的地方都用-1代替, 这是因为8位的-1的二进制表示为: 11111111,
16位的-1的表示刚好是两个8位的拼接, 所以不需要转换.
