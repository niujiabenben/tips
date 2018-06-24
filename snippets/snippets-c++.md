##  获取系统时间

只用c++11的头文件`<chrono>`并不能完成这个任务, 因为获取系统时间需要`std::tm`这个结构体,
而`std::tm`定义在`<ctime>`头文件中, 所以这里我只用`<ctime>`头文件中的函数:

```c++
#include <ctime>
#include <string>

const std::string getCurrentSystemTime() {
  std::time_t t = std::time(nullptr);
  std::tm tm = *std::localtime(&t);
  char stamp[64] = { 0 };
  size_t size = std::strftime(stamp, sizeof(stamp), "%Y-%m-%d %H:%M:%S", &tm);
  return std::string(stamp, size);
}
```

## DISABLE_COPY_MOVE_ASIGN

这个宏将`copy constructor`, `move constructor`, `copy assignment operator`,
`move assignment operator`这四个函数都设置成`delete`, 以避免误用.

```c++
#ifndef DISABLE_COPY_ASIGN
#define DISABLE_COPY_ASIGN(classname)              \
  classname(const classname&) = delete;            \
  classname& operator=(const classname&) = delete
#endif  // DISABLE_COPY_ASIGN

#ifndef DISABLE_MOVE_ASIGN
#define DISABLE_MOVE_ASIGN(classname)         \
  classname(classname&&) = delete;            \
  classname& operator=(classname&&) = delete
#endif  // DISABLE_MOVE_ASIGN

#ifndef DISABLE_COPY_MOVE_ASIGN
#define DISABLE_COPY_MOVE_ASIGN(classname)  \
  DISABLE_COPY_ASIGN(classname);            \
  DISABLE_MOVE_ASIGN(classname)
#endif  // DISABLE_COPY_MOVE_ASIGN
```

## glog输出到文件

```c++
int main(int argc, char* argv[]) {
  //// Windows下这里必须设置为true, 否则也无法输出到文件, 原因尚不清楚
  FLAGS_logtostderr = true;
  //// 这里设置log目录
  FLAGS_log_dir = "./log";
  google::InitGoogleLogging(argv[0]);
}
```
