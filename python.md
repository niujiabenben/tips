## 迭代器执行方式

* 带有yield的函数不再是一个普通函数, Python解释器会将其视为一个generator. 调用函数时,
  函数返回一个iterable对象. 在for循环执行时，每次循环都会执行函数内部的代码，
  执行到yield时, 函数就返回一个迭代值, 下次迭代时, 代码从yield的下一条语句继续执行,
  而函数的本地变量看起来和上次中断执行前是完全一样的, 于是函数继续执行, 直到再次遇到yield.
  <br>参考: [Python yield 使用浅析](https://www.ibm.com/developerworks/cn/opensource/os-cn-python-yield/)
  参考实例:

    ```python
    def lmdb_data_provider(lmdb_file):
        env = lmdb.open(lmdb_file, readonly=True)
        for key, value in env.begin().cursor():
            datum = caffe.proto.caffe_pb2.Datum()
            datum.ParseFromString(value)
            data = np.fromstring(datum.data, dtype=np.uint8)
            data = data.reshape((datum.channels, datum.height, datum.width))
            data = data.transpose((1,  2, 0))
            label = datum.label
            yield data, label
    ```

## classmethod and staticmethod

* 实例方法, 第一个参数为self, 实例方法中可以使用类成员和实例成员.

* 类方法, 由classmethod修饰, 第一个参数为cls, 类方法中可以使用类成员, 不能使用实例成员.

* 静态方法, 由staticmethod修饰, 调用时没有隐含参数, 不能使用类和实例中的任何成员.


## 在Anaconda下用pip安装python包出错

* 环境: Windows 7, Anaconda, python2.7
* 安装命令: pip install --upgrade google-cloud-vision
* 错误信息: Cannot remove entries from nonexistent file ..\anaconda2\lib\site-packages\easy-install.pth
* 解决方法: pip install --upgrade google-cloud-vision --ignore-installed


## python中的正则表达式

* re.match(pattern, string[, flags]) vs re.search(pattern, string[, flags])
  1. re.match是完全匹配(至少需要匹配字符串头)
  2. re.search是从字符串中查找, 匹配的是子字符串

* match.group() vs match.groups()
  1. match.group()返回的是分组结果
  2. match.groups()返回的是匹配结果

  ```python
    import re
    line = "solver.cpp:231] Iteration 22500, loss = 0.00166097, time = 1.541s"
    ### Iteration后面匹配正整数, loss后面匹配小数
    pattern = re.compile(r'Iteration (\d+), loss = (\d+\.\d+)')
    match = re.search(pattern, line)
    print "group:", match.group()
    print "groups:", match.groups()

    ### 输出
    group: Iteration 22500, loss = 0.00166097
    groups: ('22500', '0.00166097')
    ```

* 以上参考: [Python正则表达式指南](https://www.cnblogs.com/huxi/archive/2010/07/04/1771073.html)


## python中的变量访问控制(private, protected, & public)

* python中所有的常规变量都是public变量, 如: self.num
  <br>以一个下划线开头的变量为protected变量, 如：self.\_num
  <br>以两个下划线开头的都是private变量, 如: self.\_\_num

* 上述规则仅仅只是社区的约定, python解释器不负责实现上述的访问控制协议.

* 对于任何以两个下划线开头的变量, python解释器在解释该变量时会加上其类名.

  ```python
    class Test:
        def __init__(self):
            self._num = 10
            self.__num = 11

    if __name__ == "__main__":
        test = Test()
        print test._num
        # print test.__num     ## 错误：双下划线变量开头必须加上其类名
        print test._Test__num  ## 正确
  ```

* 以上参考: [Private, protected and public in Python ](http://radek.io/2011/07/21/private-protected-and-public-in-python/)


## python中的argparse

示例参考: [官方argparse文档](https://docs.python.org/3/library/argparse.html)

```python
import argparse

parser = argparse.ArgumentParser(description="An argparse example.")

parser.add_argument("str_type_position_argument",
                    help="A position argument with type 'str' (default).")

parser.add_argument("int_type_position_argument", type=int,
                    help="A position argument with type 'int'.");

parser.add_argument("--int_type_optional_argument", type=int, default=0,
                    help="An optional argument with type 'int' "
                    "and default value '0'.")

parser.add_argument("--bool_type_optional_argument", action="store_true",
                    help="A bool optional argument, if specified, "
                    "args.bool_type_optional_argument is True, "
                    "otherwise it's False.")

parser.add_argument("-c", "--choise", type=int, choices=[0, 1, 2],
                    help="An int type optional argument, with short "
                    "version '-c', and a range to chose from.")

parser.add_argument("-o", "--count", action="count", default=0,
                    help="An int type optional argument, whose value is "
                    "the number of occurrences of 'o'.")

group = parser.add_mutually_exclusive_group()

group.add_argument("-v", "--verbose", action="store_true",
                   help="Mutually exclusive group argument, "
                   "mutually exclusive with 'quiet'.")

group.add_argument("-q", "--quiet", action="store_true",
                   help="Mutually exclusive group argument, "
                   "mutually exclusive with 'verbose'.")


args = parser.parse_args()
```

## Other Tips

* python多线程响应ctrl-c: 将所有子线程都设为daemon线程: thread.daemon = True

* python读取c++库: ctypes.cdll.LoadLibrary('libgoodsdetect.so')
  <br>用ctypes.create_string_buffer(1024)创建可写的buffer, 对应c++的char*
