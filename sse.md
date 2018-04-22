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

## 利用cpuid检测CPU对指令集的支持情况

Intel提供了一个平台来查询所有Intrinsics函数: [Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)
包括函数的头文件, 对应的CPUID, 功能描述, 伪代码形式的操作说明等等. 需要注意的是,
对于一个给定的CPU, 不是每一个Intrinsics函数都能支持.
如果程序中使用了CPU不支持的Intrinsics函数, 可以通过编译, 但运行时会报错:
`Illegal instruction`. 这个帖子 [How to check if a CPU supports the SSE3 instruction set?](https://stackoverflow.com/questions/6121792/how-to-check-if-a-cpu-supports-the-sse3-instruction-set)
给出了由CPUID判断CPU是否支持某一个Intrinsics版本的代码,
下面这段程序是根据帖子中的代码修改而来:

```c++
#include <cpuid.h>
#include <string>
#include <iostream>

//// 这里的bit_##intrinsics是定义在cpuid.h中的宏
#define CHECK_SUPPORT(cpuinfo, intrinsics)                  \
  do {                                                      \
    bool support = (((cpuinfo) & bit_##intrinsics) != 0);   \
    std::cout << "support "#intrinsics": "                  \
              << std::boolalpha << support << std::endl;    \
  } while (0)

void get_cpuid(int info[4], int type) {
  __cpuid_count(type, 0, info[0], info[1], info[2], info[3]);
}


int main(int argc, char *argv[]) {
  int info[4];
  get_cpuid(info, 0);
  int nIds = info[0];

  get_cpuid(info, 0x80000000);
  unsigned nExIds = info[0];

  //// 下面的这些对应关系都较为复杂, 不想自己从头去捋, 直接从原贴上搬下来.
  if (nIds >= 0x00000001){
    get_cpuid(info, 0x00000001);

    CHECK_SUPPORT(info[3], MMX);
    CHECK_SUPPORT(info[3], SSE);
    CHECK_SUPPORT(info[3], SSE2);

    CHECK_SUPPORT(info[2], SSE3);
    CHECK_SUPPORT(info[2], SSSE3);
    CHECK_SUPPORT(info[2], SSE4_1);
    CHECK_SUPPORT(info[2], SSE4_2);
    CHECK_SUPPORT(info[2], AES);
    CHECK_SUPPORT(info[2], AVX);
    CHECK_SUPPORT(info[2], FMA);
    CHECK_SUPPORT(info[2], RDRND);
  }

  if (nIds >= 0x00000007){
    get_cpuid(info, 0x00000007);

    CHECK_SUPPORT(info[1], BMI);
    CHECK_SUPPORT(info[1], BMI2);
    CHECK_SUPPORT(info[1], ADX);
    CHECK_SUPPORT(info[1], SHA);
    CHECK_SUPPORT(info[2], PREFETCHWT1);

    CHECK_SUPPORT(info[1], AVX512F);
    CHECK_SUPPORT(info[1], AVX512CD);
    CHECK_SUPPORT(info[1], AVX512PF);
    CHECK_SUPPORT(info[1], AVX512ER);
    CHECK_SUPPORT(info[1], AVX512VL);
    CHECK_SUPPORT(info[1], AVX512BW);
    CHECK_SUPPORT(info[1], AVX512DQ);
    CHECK_SUPPORT(info[1], AVX512IFMA);
    CHECK_SUPPORT(info[2], AVX512VBMI);
  }

  if (nExIds >= 0x80000001){
    get_cpuid(info, 0x80000001);

    CHECK_SUPPORT(info[2], ABM);
    CHECK_SUPPORT(info[2], SSE4a);
    CHECK_SUPPORT(info[2], FMA4);
    CHECK_SUPPORT(info[2], XOP);
  }
  return 0;
}
```

这段代码实际上就是检查记录CPUID的信息的对应位上是否为1. 实际使用时,
只需要留下我们关心的几项即可.

## 用SSE进行图像Pooling操作

最近在项目中需要用到对一幅图像进行kernel为4x4, stride为4的average pooling操作.
为了得到最大的运行速度, 也对这个操作用SSE进行了改写, 代码如下:

```c++
#include <stdint.h>
#include <immintrin.h>
#include <emmintrin.h>
#include <glog/logging.h>
#include <opencv2/core/core.hpp>

#define MM_SHUFFLE_EPI16(i) (((i) << 9) + (1 << 8) + ((i) << 1))

static void average_sse_48(const uint8_t* r0,
                           const uint8_t* r1,
                           const uint8_t* r2,
                           const uint8_t* r3,
                           uint8_t* dst) {
  const __m128i zeros = _mm_set1_epi8(0);

  __m128i rsum[6];
  for (int i = 0; i < 3; ++i, r0 += 16, r1 += 16, r2 += 16, r3 += 16) {
    //// row 0
    const int idlo = 2 * i;
    const int idhi = 2 * i + 1;
    __m128i mr = _mm_loadu_si128((__m128i*) r0);
    rsum[idlo] = _mm_unpacklo_epi8(mr, zeros);
    rsum[idhi] = _mm_unpackhi_epi8(mr, zeros);
    //// row 1
    mr = _mm_loadu_si128((__m128i*) r1);
    rsum[idlo] = _mm_add_epi16(rsum[idlo], _mm_unpacklo_epi8(mr, zeros));
    rsum[idhi] = _mm_add_epi16(rsum[idhi], _mm_unpackhi_epi8(mr, zeros));
    //// row 2
    mr = _mm_loadu_si128((__m128i*) r2);
    rsum[idlo] = _mm_add_epi16(rsum[idlo], _mm_unpacklo_epi8(mr, zeros));
    rsum[idhi] = _mm_add_epi16(rsum[idhi], _mm_unpackhi_epi8(mr, zeros));
    //// row 3
    mr = _mm_loadu_si128((__m128i*) r3);
    rsum[idlo] = _mm_add_epi16(rsum[idlo], _mm_unpacklo_epi8(mr, zeros));
    rsum[idhi] = _mm_add_epi16(rsum[idhi], _mm_unpackhi_epi8(mr, zeros));
  }

  const short ID[] = {
    MM_SHUFFLE_EPI16(0), MM_SHUFFLE_EPI16(1), MM_SHUFFLE_EPI16(2),
    MM_SHUFFLE_EPI16(3), MM_SHUFFLE_EPI16(4), MM_SHUFFLE_EPI16(5),
    MM_SHUFFLE_EPI16(6), MM_SHUFFLE_EPI16(7)
  };
  //// 分解第一组8个数据: BGR BGR BG
  const __m128i v1_group1 = _mm_setr_epi16(ID[0], ID[1], ID[2], -1, -1, -1, -1, -1);
  const __m128i v1_group2 = _mm_setr_epi16(ID[3], ID[4], ID[5], -1, -1, -1, -1, -1);
  const __m128i v1_group3 = _mm_setr_epi16(ID[6], ID[7], -1, -1, -1, -1, -1, -1);
  //// 分解第二组8个数据: R BGR BGR B
  const __m128i v2_group3 = _mm_setr_epi16(-1, -1, ID[0], -1, -1, -1, -1, -1);
  const __m128i v2_group4 = _mm_setr_epi16(ID[1], ID[2], ID[3], -1, -1, -1, -1, -1);
  const __m128i v2_group1 = _mm_setr_epi16(-1, -1, -1, ID[4], ID[5], ID[6], -1, -1);
  const __m128i v2_group2 = _mm_setr_epi16(-1, -1, -1, ID[7], -1, -1, -1, -1);
  //// 分解第三组8个数据: GR BGR BGR
  const __m128i v3_group2 = _mm_setr_epi16(-1, -1, -1, -1, ID[0], ID[1], -1, -1);
  const __m128i v3_group3 = _mm_setr_epi16(-1, -1, -1, ID[2], ID[3], ID[4], -1, -1);
  const __m128i v3_group4 = _mm_setr_epi16(-1, -1, -1, ID[5], ID[6], ID[7], -1, -1);

  __m128i average[2];
  for (int i = 0; i < 2; ++i) {
    const __m128i& v1 = rsum[3 * i];
    const __m128i& v2 = rsum[3 * i + 1];
    const __m128i& v3 = rsum[3 * i + 2];
    average[i] = _mm_shuffle_epi8(v1, v1_group1);
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v1, v1_group2));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v1, v1_group3));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v2, v2_group3));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v2, v2_group4));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v2, v2_group1));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v2, v2_group2));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v3, v3_group2));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v3, v3_group3));
    average[i] = _mm_add_epi16(average[i], _mm_shuffle_epi8(v3, v3_group4));
    average[i] = _mm_srli_epi16(average[i], 4);
  }
  average[0] = _mm_shuffle_epi8(average[0], _mm_setr_epi8(
      0, 2, 4, 6, 8, 10, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1));
  average[1] = _mm_shuffle_epi8(average[1], _mm_setr_epi8(
      -1, -1, -1, -1, -1, -1, 0, 2, 4, 6, 8, 10, -1, -1, -1, -1));
  __m128i result = _mm_add_epi8(average[0], average[1]);
  //// 保存前64位, 8个数字
  *((int64_t*) dst) = _mm_cvtsi128_si64(result);
  //// 保存后32位, 4个数字
  *((int32_t*) (dst + 8)) = _mm_cvtsi128_si32(_mm_srli_si128(result, 8));
}

cv::Mat AveragePoolingSSE(const cv::Mat& image) {
  const int dst_rows = image.rows / 4;
  const int dst_cols = image.cols / 4;
  cv::Mat dst(dst_rows, dst_cols, CV_8UC3);

  for (int i = 0; i < dst_rows; ++i) {
    const uchar* line0 = image.data + 4 * i * image.step;
    const uchar* line1 = line0 + image.step;
    const uchar* line2 = line1 + image.step;
    const uchar* line3 = line2 + image.step;
    uchar* dstdata = dst.data + i * dst.step;
    //// 结果图像中4个pixel, 对应12个value, 对应原图像中16个pixel, 48个value
    for (int j = 0; j < dst_cols; j += 4) {
      average_sse_48(line0, line1, line2, line3, dstdata);
      line0 += 48;
      line1 += 48;
      line2 += 48;
      line3 += 48;
      dstdata += 12;
    }
  }
  return dst;
}
```

源图像中4x4的区域对应结果图像中的一个像素, 由于一个像素包含rgb三个值,
则对于源图像中每一行需要处理12个值, 而对于`__m128i`而言, 一次性可以读取16个值,
所以我们在每一轮的计算中, 对每一行一次性读取3个`__m128i`, 共48个值, 对应结果图像中的4个像素.

这里我们主要看函数: `average_sse_48`, 这个函数首先将每一行对应的像素相加,
得到48个按BGR排列的值, 然后设法对这48个值进行重新排列, 并求和, 最终得到pooling结果.

注意到8位的数据在求和的过程中可能有溢出, 所以得先将数据转化成16位的,
这使得第一步的求和结果为一个6维的`__m128i`数组: `__m128i rsum[6];`,
每一个`__m128i`都包含8个16位的数据.

在`rsum[6]`中, 48个数据按照BGR的顺序交错排列, 注意到`rsum[6]`中, 头3维的排列顺序和后3维一样,
所以这里我们可以将其分为两组, 每一组包含3个`__m128i`, 每一个包含8个值, 总共24个值,
其排列顺序为:

|  数据ID  |      1     |       2     |      3     |
|:--------:|:----------:|:-----------:|:----------:|
| 排列顺序 | BGR BGR BG | R BGR BGR B | GR BGR BGR |

计算均值时, 需要对相邻的4个B, 4个G, 4个R分别求和, 得到均值并写入到结果图像中.
一种方法是将其中的每一个通道都分离出来, 每一个通道都包含8个值, 然后4个为一组求和取平均.

这里我们采用另一种方法: 上述24个数总共8对BGR: BGR_1, BGR_2, BGR_3, ..., BGR_8, 将其分成4组,
每一组的排列如下:

| 分组ID |     内容     |
|:------:|:------------:|
|   1    | BGR_1, BGR_5 |
|   2    | BGR_2, BGR_6 |
|   3    | BGR_3, BGR_7 |
|   4    | BGR_4, BGR_8 |

求和时, 只需要将4组数对应的位置相加即可.

最后需要注意的是, 这里我们没有做边界处理, 如果原图像的width不能被16整除,
则在结果图像中每一行的右边会有一些像素没有被处理, 如果需要实现完整的功能,
还需要对这些像素进行单独的处理, 不过通常情况下这些像素不是很多(这里每一行最多有3个),
处理起来并不怎么耗时.

在Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz的机器上, 处理一幅10240x10240的图像,
SSE版本耗时: 28.87ms, 而用指针操作实现的版本耗时: 65.3335ms, 加速比为2.26.

## 其他资源

1. [SSE instructions to add all elements of an array](https://stackoverflow.com/questions/10930595/sse-instructions-to-add-all-elements-of-an-array)

2. [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)

3. [SSE2 Intrinsics各函数介绍](https://blog.csdn.net/fengbingchun/article/details/18460199)

4. [Software optimization resources](http://www.agner.org/optimize/#vectorclass)
