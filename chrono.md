## c++11中的计时函数

这篇CSDN文章 [<C++11时间详解>](https://blog.csdn.net/luotuo44/article/details/46854229)
讲解非常详细, 这里我补充几点:

头文件`<chrono>`中, 定义了`std::chrono::duration`, 是一个模板类, system_clock中,
也有一个duration: `std::chrono::system_clock::duration`, 是前面模板类的一个实例,
这里的duration的精度由系统来决定, 比如, 在Ubuntu 16.04, gcc版本为5.4.0的环境下,
system_clock中的duration定义为:
    `std::chrono::duration<long int, std::ratio<1l, 1000000000l> >`
实际上就是 `std::chrono::nanoseconds`.

头文件`<chrono>`中, `time_point`和`system_clock`类的定义如下:

```c++
template<typename _Clock, typename _Dur> struct time_point {
  typedef _Clock                      clock;       
  typedef _Dur                        duration;    
  typedef typename duration::rep      rep;
  typedef typename duration::period   period;

  constexpr time_point() : __d(duration::zero()) { }  
  ......
 private:
   //// 这里是实际存储time_point的变量, 比如: system_clock::now()的返回值
  duration __d;  //__ 消除atom高亮
};

struct system_clock {  //} 消除atom高亮
  typedef chrono::nanoseconds                           duration;
  typedef duration::rep                                 rep;
  typedef duration::period                              period;
  typedef chrono::time_point<system_clock, duration>    time_point;
  ......
};
```

从代码中可以看出, 实际存储`time_point`的变量, 也是`duration`类型. 比如,
`system_clock::now()`返回一个`time_point`, 而实际的值存储在`duration __d`中.
由于这个`duration`实际上是`chrono::nanoseconds`, 精度够高, 不用担心精度损失.

这里给出用chrono模块写的一个计时类和一个计算频率的类, 简单易用.

```c++
#ifndef TIMER_H_
#define TIMER_H_

#include <chrono>
#include <glog/logging.h>

class Timer {
 public:
  //// 这里的system_clock只能用typedef, 不能用using
  typedef std::chrono::system_clock system_clock;
  typedef std::chrono::duration<float> second_type;
  typedef std::chrono::duration<float, std::milli> millisecond_type;
  typedef std::chrono::duration<float, std::micro> microsecond_type;

  Timer() : is_running_(false),
            has_run_once_(false),
            has_accumulated_(false),
            count_(0) {
    total_ = system_clock::duration::zero();
  }
  ~Timer() {}

  void Start() {
    CHECK(!is_running_) << "Timer is already started.";
    start_ = system_clock::now();
    is_running_ = true;
  }
  void Stop() {
    CHECK(is_running_) << "Timer is not started yet.";
    stop_ = system_clock::now();
    is_running_ = false;
    has_run_once_ = true;
    has_accumulated_ = false;
  }
  float MilliSeconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<millisecond_type>(stop_ - start_).count();
  }
  float MicroSeconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<microsecond_type>(stop_ - start_).count();
  }
  float Seconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<second_type>(stop_ - start_).count();
  }

  void Accumulate() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    CHECK(!has_accumulated_) << "This interval has allready been accumulated.";
    total_ += stop_ - start_;
    count_ += 1;
  }
  float TotalMilliSeconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<millisecond_type>(total_).count();
  }
  float TotalMicroSeconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<microsecond_type>(total_).count();
  }
  float TotalSeconds() {
    if (is_running_) { this->Stop(); }
    CHECK(has_run_once_) << "Timer has never been run at all.";
    return std::chrono::duration_cast<second_type>(total_).count();
  }
  float AverageMilliSeconds() { return this->TotalMilliSeconds() / count_; }
  float AverageMicroSeconds() { return this->TotalMicroSeconds() / count_; }
  float AverageSeconds() { return this->TotalSeconds() / count_; }
  void ResetAccumulator() {
    total_ = system_clock::duration::zero();
    count_ = 0;
    has_accumulated_ = false;
  }
  int count() { return count_; }

 private:
  system_clock::time_point start_;
  system_clock::time_point stop_;
  system_clock::duration total_;
  bool is_running_;
  bool has_run_once_;
  bool has_accumulated_;
  int count_;
};

class FrequencyCounter {
 public:
  typedef std::chrono::system_clock system_clock;
  typedef std::chrono::duration<float> second_type;

  //// 这里interval以秒为单位, interval=0.001为毫秒
  explicit FrequencyCounter(const float interval = 1.0f)
      : interval_(interval), count_(0), is_started_(false) {}
  ~FrequencyCounter() {}

  float add(int times = 1) {
    if (!is_started_) {
      stamp_ = system_clock::now();
      is_started_ = true;
    }
    count_ += times;
    auto current = system_clock::now();
    auto interval = std::chrono::duration_cast<second_type>(current - stamp_);
    float result = -1.0f;
    if (interval >= interval_) {
      result = float(count_) * interval_.count() / interval.count();
      stamp_ = current;
      count_ = 0;
    }
    return result;
  }

 private:
  const second_type interval_;
  system_clock::time_point stamp_;
  int count_;
  bool is_started_;
};

#endif  // TIMER_H_
```
