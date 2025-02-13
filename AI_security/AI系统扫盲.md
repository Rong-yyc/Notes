# 底层基础概念

## 深度学习框架

【概念】：深度学习框架是为深度学习模型的开发、训练和部署提供支持的软件库或平台。它封装了底层的计算操作、优化算法等复杂细节，为开发者提供了高层次的编程接口，使他们能够更便捷地构建、训练和优化深度学习模型，而无需深入了解底层硬件和复杂的数学计算。

【例子】：通过 `TensorFlow` 的 `Keras API` 构建一个包含两个全连接层的神经网络

```python
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense

model = Sequential()
model.add(Dense(64, activation='relu', input_shape=(input_dim,)))
model.add(Dense(10, activation='softmax'))
```

## 算子库

【概念】：算子库是一系列经过优化的函数集合，这些函数实现了深度学习模型中各种基本操作（即算子）。深度学习模型由大量复杂的数学运算组成，如卷积、池化、激活函数计算、矩阵乘法等，算子库将这些运算封装成一个个可调用的函数。<font color="honydew">不同厂商和深度学习框架都有各自的算子库，</font>

【例子】：特定厂商算子库 —— NVIDIA 的 cuDNN

```
卷积算子：cudnnConvolutionForward 用于执行前向卷积操作
池化算子：cudnnPoolingForward 用于执行前向池化操作
```

【例子】：特定框架算子库 —— TensorFlow 算子库

```
矩阵乘法：tf.matmul
卷积层：tf.nn.conv2d 用于二维卷积操作
池化层：tf.nn.max_pool2d 用于执行二维最大池化操作
```

## 人工智能指令集

【概念】：人工智能指令集（AI Instruction Set）是专为加速人工智能计算任务（如矩阵运算、神经网络推理/训练）而设计的硬件级指令集合。它们通常集成在处理器（NPU、GPU、TPU等）或专用加速芯片中，通过硬件级别的并行化、低精度计算、张量操作优化等手段，显著提升AI任务的执行效率。