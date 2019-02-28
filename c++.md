## std::ref的作用

在函数式编程中, std::bind()是传值方式, 如果目标函数的参数是按照**传值**的方式传递,
则从函数绑定到函数调用完毕整个过程中需要经过两次的值拷贝, 一次是调用std::bind(),
一次是调用绑定后的函数. 如果用户的意图是以**引用**的方式传递参数,
则调用std::bind()时还是传值的方式, 而调用绑定后的函数用的是引用的方式,
对于用户来说, 综合的结果还是以传值的方式传递参数(如下面的例子所示).
这时, 需要在调用std::bind()时对参数用std::ref()进行封装.
类似的还有std::thread()等等.

 ```c++
 #include <iostream>
 #include <functional>

 void addone(int& num) { ++num; }

 int main(int argc, char* argv[]) {
   int num = 0;

   auto fun1 = std::bind(addone, num);  // 传值方式
   fun1();  
   std::cout << "num = " << num << std::endl;

   auto fun2 = std::bind(addone, std::ref(num));  // 引用封装
   fun2();  
   std::cout << "num = " << num << std::endl;
   return 0;
 }

 // 输出:
 num = 0
 num = 1
 ```

## std::thread中的join()和detach()

当用户通过定义一个std::thread对象来开启一个线程时, 这个对象本身属于主线程,
子线程从传入到std::thread对象的函数中开始. 当调用join()和detach()时,
只是对主线程中的std::thread对象进行操作. 具体来说, 当调用join()时,
主线程会阻塞等待子线程执行完成, 然后进行子线程资源的清理工作, 清理完毕之后返回,
这时, 子线程已经销毁, 再次调用join()会引发异常. 当调用detach()时,
std::thread对象和子线程解绑定, 这时, 主线程通过std::thread对象操作子线程的功能不复存在,
故而失去了对子线程的控制.
<br>由此可知, 调用join()或者detach()后, 子线程要么销毁, 要么与std::thread接绑定,
都会使得std::thread对象无对应的线程.

```c++
#include <thread>
#include <iostream>

// 子线程从hello函数体开始
void hello() { std::cout <<"hello world!" << std::endl; }

int main(int argc, char* argv[]) {
  // 这里my_thread属于主线程
  std::thread my_thread(hello);
  my_thread.join();
  return 0;
}
```

## c++11中的拷贝构造函数和移动构造函数

c++98中, 一个函数返回局部变量调用的是拷贝构造函数, 如下面这个例子,
`Hello foo() { return Hello(); }` 这个语句中先调用一次默认构造函数构造出一个临时变量A,
然后通过一次拷贝构造函数构造出临时变量B作为foo()的返回值, A的作用于为foo()的函数体,
所以A首先析构. 在语句`Hello hello = foo();`中, 用B作为参数再一次调用拷贝构造函数,
构建出main()函数中的临时变量hello, 这时, 临时变量B作为函数foo()的返回值的使命已经完成,
所以B进行析构. `return 0;`语句之前, hello也进行析构.

```c++
// 编译时要加上编译选项: -fno-elide-constructors
#include <iostream>

class Hello {
 public:
  Hello() {
    std::cout << "default constructor: " << ++count_ << std::endl;
  }
  Hello(const Hello& other) {
    std::cout << "copy constructor: " << ++count_ << std::endl;
  }
  ~Hello() {
    std::cout << "destructor: " << --count_ << std::endl;
  }

 public:
  static int count_;
};

int Hello::count_ = 0;

Hello foo() { return Hello(); }

int main(int argc, char *argv[]) {
  Hello hello = foo();
  return 0;
}

// 输出
default constructor: 1
copy constructor: 2
destructor: 1
copy constructor: 2
destructor: 1
destructor: 0
```

c++11中, 函数返回局部变量用的是移动构造函数, 如下面这个例子, 调用过程和上一个例子完全相同,
只是将复制构造函数改成了移动构造函数. 另外, c++11中函数返回右值时(返回局部变量即为右值),
只能调用移动构造函数. 如果移动构造函数被设置成了delete, 则编译会报错.
如果不显式设置移动构造函数, 则会调用复制构造函数, 输出与上一个例子完全相同.

```c++
  // 编译时要加上编译选项: -fno-elide-constructors
#include <iostream>

class Hello {
 public:
  Hello() {
    std::cout << "default constructor: " << ++count_ << std::endl;
  }
  Hello(const Hello& other) {
    std::cout << "copy constructor: " << ++count_ << std::endl;
  }
  Hello(Hello&& other) {
    std::cout << "move constructor: " << ++count_ << std::endl;
  }
  ~Hello() {
    std::cout << "destructor: " << --count_ << std::endl;
  }

 public:
  static int count_;
};

int Hello::count_ = 0;

Hello foo() { return Hello(); }

int main(int argc, char *argv[]) {
  Hello hello = foo();
  return 0;
}

// 输出
default constructor: 1
move constructor: 2
destructor: 1
move constructor: 2
destructor: 1
destructor: 0
```

结论:
**在c++11中, 当函数按值返回局部变量时, 优先调用移动构造函数,
如果没有显式定义移动构造函数, 则采用复制构造函数, 与c++98标准兼容.
如果显式定义移动构造函数为delete, 则编译会报错.**

## gcc中的__restrict

用于修饰指针, 用户确保指向该数据块的只有这一个指针, 编译器针对这个保证对代码进行优化.
实例参考: [如何理解C语言关键字restrict？](https://www.zhihu.com/question/41653775)
