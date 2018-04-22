## 自定义层

* 如果自定义层为loss layer，则在模型定义文件中得显式指出loss_weight，
  用来告诉caffe这一层是loss层. 否则，caffe会认为这一层是普通层而不进行后向传播(loss层没有后继)

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

* 上一条定义的loss_weight放在top[0].diff[0]中，后向传播时可以用到.

## sigmoid cross entropy loss layer

* caffe的SigmoidCrossEntropyLossLayer将sigmoid层和CrossEntropyLoss层一起计算，
  原因是一起计算有助于computational stability.

* 这里的关键在于，log(sigmoid(x)) 和 log(1-sigmoid(x)) 在x的绝对值较大的时候不稳定，
  需要根据x的正负性来做调整.

* 这里需要注意的点是，在数值计算中，计算log(x)的值时x不能接近于0；
  在计算exp(x)的值时x不能为大的正数.
