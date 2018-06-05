## 自定义层

如果自定义层为loss layer，则在模型定义文件中得显式指出loss_weight，用来告诉caffe这一层是loss层. 否则，caffe会认为这一层是普通层而不进行后向传播(loss层没有后继)

  ```c++
  layer {
    name: "loss/attr2"
    type: "Python"
    bottom: "check2"
    bottom: "label/attr"
    top: "loss/attr2"
    python_param {
      module: "layers.weighted_sigmoid_cross_entropy_loss_layer"
      layer: "SigmoidCrossEntropyLossLayer"
    }
    loss_weight: 10.1
  }
  ```

顺便说一下, 在layer中定义的loss_weight放在top[0].diff[0]中，后向传播时可以用到.


## sigmoid cross entropy loss layer

caffe的SigmoidCrossEntropyLossLayer将sigmoid层和CrossEntropyLoss层一起计算，原因是一起计算有助于computational stability. 这里的关键在于，`log(sigmoid(x))`和`log(1-sigmoid(x))`在x的绝对值较大的时候不稳定，需要根据x的正负性来做调整. 这里需要注意的点是，在数值计算中，计算log(x)的值时x不能接近于0；在计算exp(x)的值时x不能为大的正数.


## caffe中的caffe.io.load_image()

caffe.io.load_image()读取图像的时候, 将每一个值归一化到[0, 1]之间 (每一个值除以255), 并且是以RGB的形式排列. 下面的代码给出了caffe.io.load_image()方法读取的图像和cv2.imread()方法读取的图像之间的转换方法:

```Python

import cv2
import caffe
import numpy as np

### cv2.imread()方法得到的image转化caffe中的格式
def image_format_cv2caffe(image):
    return image[:, :, ::-1].astype(np.float32) / 255.0

### caffe.io.load_image()方法得到的image转化成cv2中的格式
def image_format_caffe2cv(image):
    return (image[:, :, ::-1] * 255.0).astype(np.uint8)
```

值得注意的是, 由于使用的解码器的不同, 两者读取jpg格式的图像的结果也有些许差异. 经测试可得, 以[0, 255]范围计算,
误差一般不超过1. 对于无损压缩格式(比如png)的图片两者读取的结果完全一样.

标准SSD算法中, 如果采用caffe自带的caffe.io.Transformer进行预处理, 则需要设置的参数如下:

```Python
### cv2.imread()方法得到的image
transpose = (2, 0, 1)
mean = np.array([104, 117, 123], dtype=np.float32)

### caffe.io.load_image()方法得到的image
transpose = (2, 0, 1)
channel_swap = (2, 1, 0)
raw_scale = 255
mean = np.array([104, 117, 123], dtype=np.float32))
```
