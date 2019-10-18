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

## python中的logging

示例参考: [官方logging文档](https://docs.python.org/3/howto/logging.html)

```python
logging_root = "./data"

def config_global_logger():
    """设置全局的logger.

    全局logger在主线程中使用, 同时log到文件和console.
    """

    assert len(logging_root) > 0
    date = str(datetime.date.today())
    logging_file = os.path.join(logging_root, date, "log_global.txt")
    dirname = os.path.dirname(logging_file)
    if not os.path.exists(dirname):
        os.makedirs(dirname)

    ### basic setting, log to file
    format = "%(asctime)s %(filename)s:%(lineno)d] %(levelname)s: %(message)s"
    logging.basicConfig(
        filename=logging_file,
        level=logging.INFO,
        format=format,
        datefmt="%Y-%m-%d %H:%M:%S")

    ### also log to console
    console = logging.StreamHandler()
    console.setLevel(logging.INFO)
    console.setFormatter(logging.Formatter(format))
    logging.getLogger().addHandler(console)


def get_local_logger(name):
    """获取局部的logger.

    局部logger在子线程中使用, 只log到文件.
    """

    assert name != "global"
    assert len(logging_root) > 0

    date = str(datetime.date.today())
    name = "log_{}.txt".format(name)
    logging_file = os.path.join(logging_root, date, name)
    dirname = os.path.dirname(logging_file)
    if not os.path.exists(dirname):
        os.makedirs(dirname)

    logger = logging.getLogger(name)
    format = "%(asctime)s %(filename)s:%(lineno)d] %(levelname)s: %(message)s"
    file_handler = logging.FileHandler(logging_file)
    file_handler.setLevel(logging.INFO)
    file_handler.setFormatter(logging.Formatter(format))
    logger.addHandler(file_handler)
    ### 局部的logger是全局logger的child, 这里防止局部的log扩散到全局log中
    logger.propagate = False
    return logger
```

## Python Tutorial

**python tips**

```python
8 / 5    # result: 1.6 (普通除法)
8 // 5   # result: 1   (整数除法)
5 ** 2   # result: 25  (乘方)

### Function annotations are completely optional metadata information about the
### types used by user-defined functions
def foo(ham: str, eggs: str = 'eggs') -> str: pass

### nested list comprehension
[(x, y) for x in [1, 2, 3] for y in [3, 1, 4] if x != y]

### A tuple consists of a number of values separated by commas
a_tuple = 1, 2, 3, 4, 5
empty_tuple = ()
one_element_tuple = (1,)  # 逗号不能省, 但括号可以省略
### multiple assignment is realy just a combination of tuple packing and
### sequence unpacking
a, b, c = 1, 2, 3  # 左边是tuple, '='为sequence unpacking

### A set is an unordered collection with no duplicate elements
empty_set = set()
a_element_set = {1, 2, 3, 4}
set_comprehension = {x for x in 'abracadabra' if x not in 'abc'}

### dictionaries keys can be any immutable type, 如果是tuple其元素不能含有任何
### mutable的类型, 因为mutable类型可以原地改变其内容.
empty_dict = {}
dict_comprehension = {x: x**2 for x in (2, 4, 6)}
### 这里只需要是k-v对列表就可以
a_dict = dict([('sape', 4139), ('guido', 4127), ('jack', 4098)])
### 这里key为string类型, 结果为: {'sape': 4139, 'guido': 4127, 'jack': 4098}
another_dict = dict(sape=4139, guido=4127, jack=4098)

### 'in' and 'not in' check whether a value occurs (does not occur) in a sequence
"two" in "one_two_three"  # True
### 'is' and 'is not' compare whether two objects are really the same object
a = b = (1, 2)           # a is b 为 True
c, d = (1, 2), (1, 2)    # c is d 为 False
### Comparisons can be chained.
a < b == c   # 等价于 a < b and b == c

### 运算符优先级: numerical operators > comparison operators > not > and > or
a + b > c and not d or f  # 顺序: ((a + b) > c) and ((not d) or f)

### The Boolean operators 'and' and 'or' are short-circuit operators: their
### arguments are evaluated from left to right, and evaluation stops as soon as
### the outcome is determined. The return value is the last evaluated argument.
s1, s2, s3 = '', 'Trondheim', 'Hammer Dance'
non_null = s1 or s2 or s3  # non_null: 'Trondheim'

### Sequence objects may be compared to other objects with the same sequence type.
### 这里Sequence objects的比较实际上就是比较每一个元素
(1, 2, 3) == (1, 2, 3)     # True
[1, 2, 3] <= [1, 2, 4]     # true
(1, 2, 3) <  (1, 2, 3, 4)  # True

```

**python中, else可以与for一起连用**

Loop statements may have an else clause; it is executed when the loop terminates
through exhaustion of the list (with for) or when the condition becomes false
(with while), but not when the loop is terminated by a break statement.

```python
for n in range(2, 10):
    for x in range(2, n):
        if n % x == 0:
            print(n, 'equals', x, '*', n//x)
            break
    else:  # 这里else是和内层的for一起的
        # loop fell through without finding a factor
        print(n, 'is a prime number')
```

**Python中, 函数的默认参数只计算一次**

The default value is evaluated only once.

```python
def f(a, L=[]):
    L.append(a)
    return L

print(f(1))   # 输出: [1]
print(f(2))   # 输出: [1, 2]
print(f(3))   # 输出: [1, 2, 3]
```

**python中的module**

* 一个py文件就是一个module:
  <br>A module is a file containing Python definitions and statements.

* 一个带有__init__.py的目录就是一个package

* `import module_or_package`:
  <br>导入module/package, 不能导入module中的名字(比如函数名字, 变量名字, 等等)

* `from module_or_package import name1, name2`:
  <br>从module/package中导入名字name1和name2. 这里的名字可以是任何名字.

* python模块搜索顺序:
  <br>内置模块 -> 当前文件夹 -> PYTHONPATH指定路径 -> 与安装有关的默认路径

#### python的namespace和scope

c++中的namespace指的是一个container, 这个container中间有一些names.

python中的namespace是一个mapping, 常常用dict表示, 表示name及其指向的object的映射.
比如: 一个模块的全局变量和函数, 一个函数的局部变量和函数, 内置变量和函数, 等等.
这些mapping指的是同一级别的name的mapping, 在namespace内引用这些name的时候不需要带模块名.

python中的scope指的是空间作用域, 在某一个scope之内, 名字搜索顺序如下:
<br>当前作用域局部变量 -> 外层作用域变量 -> 当前模块中的全局变量 -> python内置变量

```python
def greeting():
    ### 这里name为全局变量, 在函数定义之后定义
    ### 只需要保证在函数调用的前面定义这个变量就可以了
    print("hello", name)

name = "world"  # name必须定义在greeting()函数调用之前
greeting()      # 这里输出: hello world
```

```python
def scope_test():
    def do_local():
        spam = "local spam"

    def do_nonlocal():
        nonlocal spam
        spam = "nonlocal spam"

    def do_global():
        ### 这里实际上定义了全局变量spam, 如果只是使用, 可以不用global关键字
        global spam
        spam = "global spam"

    spam = "test spam" # (相对于最里面的三个函数来说)这里是nonlocal
    do_local()
    do_nonlocal()
    do_global()

scope_test()
print("In global scope:", spam)
```

global关键字: 局部作用域中如果需要rebind全局变量, 需要用global申明
<br>nonlocal关键字: 局部作用域中如果需要rebind外层变量, 需要用nonlocal申明
<br>需要global和nonlocal两个关键字的原因是Python允许局部变量覆盖掉同名的全局变量.

all operations that introduce new names use the local scope: in particular,
import statements and function definitions bind the module or function name
in the local scope. 这里和c++中的#include语句有明显的差异.

可以通过del删除某个变量, 实际上是删除namespace中对应的名字.
<br>可以在任何地方对某一个namespace添加和删除元素, 局部变量除外: In fact,
local variables are already determined statically.

```python
class Employee:

    def print_member(self):
        ### 外部定义的变量, Employee.name为类变量, self.name为实例变量
        print(Employee.name, self.name)

john = Employee()
Employee.name = "hello"  # 这里添加类变量
john.name = 'world'      # 这里添加实例变量
john.print_member()      # 这里输出: hello world

### 在name dict中删掉变量
del Employee.name
del john.name
```

#### Python中的class

python中的class是一个namespace, 可以在任意地方对这个namespace添加名字

```python

class Hello:
    ### 这里的参数self只是Hello的一个实例 (甚至只需要是一个namespace就可以)
    def foo(self):
      ### 这里只是对self这个namespace添加一个变量var
      self.var = 0

### 这里实例化一个对象, 创造了一个新的namespace
h = Hello()
### 调用Hello.foo()函数
h.foo()
### 也可以这样调用Hello.foo()函数.
Hello.foo(h)
```

类中数据成员变量会覆盖同名的成员函数:
Data attributes override method attributes with the same name.

类中的名字搜索顺序: 派生类 -> 基类A -> 基类A的基类 -> ... -> 基类B -> 基类B的基类 -> ...
也就是说, 多重继承的情况下, Python以深度优先的方式搜索名字.

Python的每一个变量都带有其实际的类型(不像C++子类有时会隐式转换到基类),
所以每一个自定义类的变量都遵循上面的搜索顺序.
For C++ programmers: all methods in Python are effectively virtual.

一般情况下, 已一个下划线开头的类成员变量会被当做私有成员, 这只是一个约定.

**name mangling**

Any identifier of the form __spam (at least two leading underscores,
at most one trailing underscore) is textually replaced with _classname__spam,
where classname is the current class name with leading underscore(s) stripped.

```python
class Mapping:
    def __init__(self, iterable):
        self.items_list = []
        self.__update(iterable)

    def update(self, iterable):
        for item in iterable:
          self.items_list.append(item)

    __update = update  # private copy of original update() method

class MappingSubclass(Mapping):

  def update(self, keys, values):
      # provides new signature for update()
      # but does not break __init__()
      for item in zip(keys, values):
          self.items_list.append(item)

### Python解释器按如下顺序执行:
### 1. 在子类MappingSubclass中寻找__init__(), 没有找到;
### 2. 在基类Mapping中寻找__init__(), 找到了;
### 3. 执行Mapping.__init__()
### 4. 执行到 self.__update(iterable), 实际上是Mapping.update()
### 如果Mapping类中没有私有拷贝: '__update = update', 在__init__()中直接调用:
### self.update(iterable), 则因为传到__init__()方法的第一个参数self实际上是
### MappingSubclass类型的, 所以调用self.update(iterable)时,
### 调用的是MappingSubclass.update(), 这时会报参数不匹配错误.
mapping = MappingSubclass([1, 2, 3, 4])
```

## Other Tips

* python多线程响应ctrl-c: 将所有子线程都设为daemon线程: thread.daemon = True

* python读取c++库: ctypes.cdll.LoadLibrary('libgoodsdetect.so')
  <br>用ctypes.create_string_buffer(1024)创建可写的buffer, 对应c++的char*
