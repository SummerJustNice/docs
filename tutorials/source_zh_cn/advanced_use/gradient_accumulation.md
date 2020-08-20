# 梯度累积

<!-- TOC -->

- [梯度累积](#梯度累积)
    - [概述](#概述)
    - [建立梯度累积模型](#建立梯度累积模型)
        - [导入需要的库文件](#导入需要的库文件)
        - [加载数据集](#加载数据集)
        - [定义网络](#定义网络)
        - [定义训练模型](#定义训练模型)
        - [定义训练过程](#定义训练过程)
        - [训练并保存模型](#训练并保存模型)
    - [实验结果](#实验结果)
    

<!-- /TOC -->

<a href="https://gitee.com/mindspore/docs/blob/master/tutorials/source_zh_cn/advanced_use/gradient_accumulation.md" target="_blank"><img src="../_static/logo_source.png"></a>

## 概述

本教程介绍梯度累积的训练方式，目的是为了解决由于内存不足导致某些大型网络无法训练大Batch_size的问题。
  
传统的训练方式是每次计算得到loss和梯度后，直接用所得梯度对参数进行更新。
  
与传统的训练方式不同，梯度累积引入Mini-batch的概念，首先对每个Mini-batch的数据计算loss和梯度，但不立即更新模型参数，而是先对所得梯度进行累加，然后在指定数量（N）个Mini-batch之后，用累积后的梯度更新网络参数。下次训练前清空过往累积梯度后重新累加，如此往复。
  
最终目的是为了达到跟直接用N*Mini-batch数据训练几乎同样的效果。

> 本教程用于GPU、Ascend 910 AI处理器, 你可以在这里下载完整的样例代码：<https://gitee.com/mindspore/docs/tree/master/tutorials/tutorial_code/gradient_accumulation>

## 创建梯度累积模型

以MNIST作为示范数据集，自定义简单模型实现梯度累积。

### 导入需要的库文件
下列是我们所需要的公共模块及MindSpore的模块及库文件。

```python
import argparse
import os
from collections.abc import Iterable

import mindspore.nn as nn
from mindspore import ParameterTuple
from mindspore import context
from mindspore.nn import Cell
from mindspore.ops import composite as C
from mindspore.ops import functional as F
from mindspore.ops import operations as P
from mindspore.train.dataset_helper import DatasetHelper
from mindspore.train.serialization import _exec_save_checkpoint
from model_zoo.official.cv.lenet.src.dataset import create_dataset
from model_zoo.official.cv.lenet.src.lenet import LeNet5
```

### 加载数据集

利用MindSpore的dataset提供的`MnistDataset`接口加载MNIST数据集。

```python
def create_dataset(data_path, batch_size=32, repeat_size=1,
                   num_parallel_workers=1):
    """
    create dataset for train or test
    """
    # define dataset
    mnist_ds = ds.MnistDataset(data_path)

    resize_height, resize_width = 32, 32
    rescale = 1.0 / 255.0
    shift = 0.0
    rescale_nml = 1 / 0.3081
    shift_nml = -1 * 0.1307 / 0.3081

    # define map operations
    resize_op = CV.Resize((resize_height, resize_width), interpolation=Inter.LINEAR)  # Bilinear mode
    rescale_nml_op = CV.Rescale(rescale_nml, shift_nml)
    rescale_op = CV.Rescale(rescale, shift)
    hwc2chw_op = CV.HWC2CHW()
    type_cast_op = C.TypeCast(mstype.int32)

    # apply map operations on images
    mnist_ds = mnist_ds.map(input_columns="label", operations=type_cast_op, num_parallel_workers=num_parallel_workers)
    mnist_ds = mnist_ds.map(input_columns="image", operations=resize_op, num_parallel_workers=num_parallel_workers)
    mnist_ds = mnist_ds.map(input_columns="image", operations=rescale_op, num_parallel_workers=num_parallel_workers)
    mnist_ds = mnist_ds.map(input_columns="image", operations=rescale_nml_op, num_parallel_workers=num_parallel_workers)
    mnist_ds = mnist_ds.map(input_columns="image", operations=hwc2chw_op, num_parallel_workers=num_parallel_workers)

    # apply DatasetOps
    buffer_size = 10000
    mnist_ds = mnist_ds.shuffle(buffer_size=buffer_size)  # 10000 as in LeNet train script
    mnist_ds = mnist_ds.batch(batch_size, drop_remainder=True)
    mnist_ds = mnist_ds.repeat(repeat_size)

    return mnist_ds
```

### 定义网络

这里以LeNet网络为例进行介绍，当然也可以使用其它的网络，如ResNet-50、BERT等。

```python
def conv(in_channels, out_channels, kernel_size, stride=1, padding=0):
    """weight initial for conv layer"""
    weight = weight_variable()
    return nn.Conv2d(in_channels, out_channels,
                     kernel_size=kernel_size, stride=stride, padding=padding,
                     weight_init=weight, has_bias=False, pad_mode="valid")


def fc_with_initialize(input_channels, out_channels):
    """weight initial for fc layer"""
    weight = weight_variable()
    bias = weight_variable()
    return nn.Dense(input_channels, out_channels, weight, bias)


def weight_variable():
    """weight initial"""
    return TruncatedNormal(0.02)


class LeNet5(nn.Cell):
    """
    Lenet network

    Args:
        num_class (int): Number classes. Default: 10.

    Returns:
        Tensor, output tensor
    Examples:
        >>> LeNet(num_class=10)

    """
    def __init__(self, num_class=10, channel=1):
        super(LeNet5, self).__init__()
        self.num_class = num_class
        self.conv1 = conv(channel, 6, 5)
        self.conv2 = conv(6, 16, 5)
        self.fc1 = fc_with_initialize(16 * 5 * 5, 120)
        self.fc2 = fc_with_initialize(120, 84)
        self.fc3 = fc_with_initialize(84, self.num_class)
        self.relu = nn.ReLU()
        self.max_pool2d = nn.MaxPool2d(kernel_size=2, stride=2)
        self.flatten = nn.Flatten()

    def construct(self, x):
        x = self.conv1(x)
        x = self.relu(x)
        x = self.max_pool2d(x)
        x = self.conv2(x)
        x = self.relu(x)
        x = self.max_pool2d(x)
        x = self.flatten(x)
        x = self.fc1(x)
        x = self.relu(x)
        x = self.fc2(x)
        x = self.relu(x)
        x = self.fc3(x)
        return x
```

### 定义训练模型
将训练流程拆分为正向反向训练、参数更新和累积梯度清理三个部分：
- `TrainForwardBackward`计算loss和梯度，利用grad_sum实现梯度累加。
- `TrainOptim`实现参数更新。
- `TrainClear`实现对梯度累加变量grad_sum清零。

```python
_sum_op = C.MultitypeFuncGraph("grad_sum_op")
_clear_op = C.MultitypeFuncGraph("clear_op")


@_sum_op.register("Tensor", "Tensor")
def _cumulative_gard(grad_sum, grad):
    """Apply gard sum to cumulative gradient."""
    add = P.AssignAdd()
    return add(grad_sum, grad)


@_clear_op.register("Tensor", "Tensor")
def _clear_grad_sum(grad_sum, zero):
    """Apply zero to clear grad_sum."""
    success = True
    success = F.depend(success, F.assign(grad_sum, zero))
    return success


class TrainForwardBackward(Cell):
    def __init__(self, network, optimizer, grad_sum, sens=1.0):
        super(TrainForwardBackward, self).__init__(auto_prefix=False)
        self.network = network
        self.network.add_flags(defer_inline=True)
        self.weights = ParameterTuple(network.trainable_params())
        self.optimizer = optimizer
        self.grad_sum = grad_sum
        self.grad = C.GradOperation('grad', get_by_list=True, sens_param=True)
        self.sens = sens
        self.hyper_map = C.HyperMap()

    def construct(self, *inputs):
        weights = self.weights
        loss = self.network(*inputs)
        sens = P.Fill()(P.DType()(loss), P.Shape()(loss), self.sens)
        grads = self.grad(self.network, weights)(*inputs, sens)
        return F.depend(loss, self.hyper_map(F.partial(_sum_op), self.grad_sum, grads))


class TrainOptim(Cell):
    def __init__(self, optimizer, grad_sum):
        super(TrainOptim, self).__init__(auto_prefix=False)
        self.optimizer = optimizer
        self.grad_sum = grad_sum

    def construct(self):
        return self.optimizer(self.grad_sum)


class TrainClear(Cell):
    def __init__(self, grad_sum, zeros):
        super(TrainClear, self).__init__(auto_prefix=False)
        self.grad_sum = grad_sum
        self.zeros = zeros
        self.hyper_map = C.HyperMap()

    def construct(self):
        seccess = self.hyper_map(F.partial(_clear_op), self.grad_sum, self.zeros)
        return seccess
```

### 定义训练过程
每个Mini-batch通过正反向训练计算loss和梯度，通过mini_steps控制每次更新参数前的累加次数。达到累加次数后进行参数更新和
累加梯度变量清零。

```python
class GradientAccumulation:
    def __init__(self, network, loss_fn, optimizer):
        self._network = network
        self._loss_fn = loss_fn
        self._optimizer = optimizer

        params = self._optimizer.parameters
        self._grad_sum = params.clone(prefix="grad_sum", init='zeros')
        self._zeros = params.clone(prefix="zeros", init='zeros')
        self._train_forward_backward = self._build_train_forward_backward_network()
        self._train_optim = self._build_train_optim()
        self._train_clear = self._build_train_clear()

    @staticmethod
    def _transform_callbacks(callbacks):
        """Transform callback to a list."""
        if callbacks is None:
            return []

        if isinstance(callbacks, Iterable):
            return list(callbacks)

        return [callbacks]

    def _build_train_forward_backward_network(self):
        """Build forward and backward network"""
        network = self._network
        network = nn.WithLossCell(network, self._loss_fn)
        loss_scale = 1.0
        network = TrainForwardBackward(network, self._optimizer, self._grad_sum, loss_scale).set_train()
        return network

    def _build_train_optim(self):
        """Build optimizer network"""
        network = TrainOptim(self._optimizer, self._grad_sum).set_train()
        return network

    def _build_train_clear(self):
        """Build clear network"""
        network = TrainClear(self._grad_sum, self._zeros).set_train()
        return network

    def train_process(self, epoch, train_dataset, mini_steps=None):
        """
        Training process. The data would be passed to network directly.
        """
        dataset_helper = DatasetHelper(train_dataset, dataset_sink_mode=False, epoch_num=epoch)

        for i in range(epoch):
            step = 0
            for k, next_element in enumerate(dataset_helper):
                loss = self._train_forward_backward(*next_element)
                if (k + 1) % mini_steps == 0:
                    step += 1
                    print("epoch:", i + 1, "step:", step, "loss is ", loss)
                    self._train_optim()
                    self._train_clear()

            train_dataset.reset()

        _exec_save_checkpoint(self._train_forward_backward, "gradient_accumulation.ckpt", )
```

### 训练并保存模型
调用网络、优化器及损失函数，然后自定义`GradientAccumulation`的`train_process`接口，进行模型训练。

```python
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='MindSpore Gard Cumulative Example')
    parser.add_argument('--device_target', type=str, default="Ascend", choices=['Ascend', 'GPU'],
                        help='device where the code will be implemented (default: Ascend)')
    parser.add_argument('--data_path', type=str, default="./Data",
                        help='path where the dataset is saved')
    args = parser.parse_args()

    context.set_context(mode=context.GRAPH_MODE, device_target=args.device_target)
    ds_train = create_dataset(os.path.join(args.data_path, "train"), 32)

    network = LeNet5(10)
    net_loss = nn.SoftmaxCrossEntropyWithLogits(is_grad=False, sparse=True, reduction="mean")
    net_opt = nn.Momentum(network.trainable_params(), 0.01, 0.9)
    model = GradientAccumulation(network, net_loss, net_opt)

    print("============== Starting Training ==============")
    model.train_process(10, ds_train, mini_steps=4)
```

## 实验结果
在经历了10轮epoch之后，在测试集上的精度约为96.31%。

**执行训练**
1. 运行训练代码，查看运行结果。
    ```shell
    $ python train.py --data_path=./MNIST_Data
    ```
    输出如下，可以看到loss值随着训练逐步降低：

    ```shell
    epoch: 1 step: 27 loss is  0.3660637
    epoch: 1 step: 28 loss is  0.25238192
    ...
    epoch: 3 step: 2 loss is  0.12296932
    epoch: 3 step: 3 loss is  0.15799297
    ...
    epoch: 10 step: 448 loss is  0.06443884
    epoch: 10 step: 449 loss is  0.0067842817
    ```
    
2. 查看保存的CheckPoint文件。

    训练过程中保存了CheckPoint文件`gradient_accumulation.ckpt`，即模型文件。

**验证模型**

通过model_zoo下lenet网络的[eval.py](<https://gitee.com/mindspore/mindspore/tree/master/model_zoo/official/cv/lenet/train.py>)，使用保存的CheckPoint文件，加载验证数据集，进行验证。

```shell
$ python eval.py --data_path=./MNIST_Data --ckpt_path=./gradient_accumulation.ckpt
```

输出如下，可以看到使用验证的数据集，正确率在96.31%左右，与batch_size为32的验证结果一致。

```shell
============== Starting Testing ==============
============== {'Accuracy': 0.9631730769230769} ==============
```