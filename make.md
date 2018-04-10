## Makefile总规则

```Makefile
target: prerequisites
    command
```

target可以是任意的标签, 如果target为一个实体, 比如文件和目录,
则当该target存在且比其依赖的所有文件要新时, 下面的command并不执行.
如果target不存在, 或者某个依赖文件比其要新, 则执行command, 生成target.

如果target仅仅为一个标签, 并不对应系统中的任何实体, 则在所有情况下都会执行的command,
因为始终无法找到target对应的文件或者目录.

prerequisites中要么为实体(文件或者目录), 要么为其他的target. 在检查前提条件时,
对每一个前提条件, 先检查是否有对应的文件或者目录, 如果没有, 则看有没有对应的target,
如果没有对应的target, 则报错; 如果有对应的target, 则执行该target, 生成对应的文件.

command可以是任意的shell命令, 除了生成target对应的文件, 还可以干其他的任何事情.

target可以有多个, 用空格分开, 或者使用通配符, 如:
```Makefile
### 编译src目录下所有.cpp文件
build/%.o: src/%.cpp
    g++ -c -o $@ $^

### 用protoc编译$(PROTODIR)目录下所有.proto文件
$(PROTODIR)/%.pb.cc $(PROTODIR)/%.pb.h : $(PROTODIR)/%.proto
    cd $(PROTODIR); protoc --cpp_out=. $(notdir $<)
```


## Makefile中的变量

Makefile中的变量类似C/C++中的宏，在Makefile执行的时候会原样展开.
所以, 对于语句`objects = *.o`, 在执行的时候所有`objects`引用会展开为`*.o`,
如果需要与shell类似的展开方式, 需要用wildcard命令, 即`objects := $(wildcard *.o)`.

Makefile变量赋值方式:
* `foo = $(bar)`: bar的值可以定义在文件的任何一处，可能引起递归定义;
* `foo := $(bar)`: bar的值只能定义在这条语句之前，否则bar的值为空;
* `foo ?= bar`: 表示如果foo的值没定义过，就让foo的值为bar，否则什么也不做;
* `foo += bar`: 表示在foo变量中追加值bar.


## Makefile中的命令

@的作用: 通常，make会把其要执行的命令行在命令执行前输出到屏幕上.
当我们用“@”字符在命令行前，那么，这个命令将不被make显示出来.

Makefile的每一条命令都是在工作目录中执行, 如果要让上一条命令的结果应用在下一条命令时，
应该使用分号分隔这两条命令.如:
```Makefile
cd /home/hchen
pwd  # 这条语句会打印出Makefile的工作目录, 上一条语句结束, 系统就回到工作目录中.
cd /home/hchen; pwd  # 这条语句会打印出`/home/hchen`
```

## Makefile中使用CUDA

CUDA代码用nvcc编译, 其Makefile结构和c++代码类似, 只需要注意NVCCFLAGS和CUDA_ARCH的设置即可.

```Makefile
### g++编译设置
CC := g++
CFLAGS := -g -Wall -fpic -O2 -Wno-unused-function

### nvcc编译设置
NVCC := nvcc
NVCCFLAGS := -ccbin=$(CC) $(foreach FLAG, $(CFLAGS), -Xcompiler $(FLAG))
CUDA_ARCH := -gencode arch=compute_35,code=sm_35 \
             -gencode arch=compute_50,code=sm_50 \
             -gencode arch=compute_52,code=sm_52 \
             -gencode arch=compute_61,code=sm_61

### 项目结构
INCDIR := include
SRCDIR := src
BUILDDIR := build

### 头文件和库文件设置
INCLUDE := $(INCDIR) /usr/local/cuda/include /usr/local/cuda/samples/common/inc
LIBRARY := /usr/local/cuda/lib64
LIBS := dl m z rt glog cudart cublas curand

INCLUDE := $(foreach INC, $(INCLUDE), -I $(INC))
LIBRARY := $(foreach LIB, $(LIBRARY), -L $(LIB))
LIBS := $(foreach LIB, $(LIBS), -l$(LIB))

### $(SRCDIR)包含所有的库cu
SRC_SRC_CU := $(shell find $(SRCDIR) -type f -name *.cu)
OBJ_SRC_CU := $(addprefix $(BUILDDIR)/, ${SRC_SRC_CU:.cu=.cuo})

### cu文件的编译
$(OBJ_SRC_CU): $(BUILDDIR)/%.cuo : %.cu $(HEADERS) | $(ALL_BUILD_DIRS)
    $(NVCC) $(NVCCFLAGS) $(CUDA_ARCH) -c -o $@ $< $(INCLUDE)
```

## Makefile中使用protoc

如果有多个proto文件并且相互间有依赖关系, 则用protoc编译proto文件时,
生成的.h和.cc文件会根据编译时指定的--proto_path包含依赖的.h,
有些时候这并不是我们想要的. 这里采取一个简单的方法, 编译proto时, 先cd到对应的目录.

```Makefile
### protobuf对应的文件
DEF_PROTO := $(shell find $(PROTODIR) -type f -name *.proto)
OBJ_PROTO := $(addprefix $(BUILDDIR)/, ${DEF_PROTO:.proto=.pb.o})
HEADERS += $(patsubst %.proto, %.pb.h, $(DEF_PROTO))

### 编译protoc生成的.pb.cc文件
$(OBJ_PROTO): $(BUILDDIR)/%.pb.o : %.pb.cc $(HEADERS) | $(ALL_BUILD_DIRS)
    $(CC) $(CFLAGS) -c -o $@ $< $(INCLUDE)

### 用protoc编译proto文件
$(PROTODIR)/%.pb.cc $(PROTODIR)/%.pb.h : $(PROTODIR)/%.proto
    cd $(PROTODIR); protoc --cpp_out=. $(notdir $<)
```

## 其他

有关Makefile的其他的一些知识, 比如Makefile中的函数, Makefile的命令行参数,
级联的Makefile的引用, 等等, 都可以在<跟我一起写makefile>中找到,
后面的实际使用中有什么不明白的, 可以看<跟我一起写makefile>中的对应章节.
