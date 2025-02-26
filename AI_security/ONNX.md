# 介绍

## ONNX

​		`ONNX` 是一种用于深度学习模型的开源标准，用来表示深度学习模型的开放格式。所谓开放就是 ONNX 定义了一组与环境、平台均无关的标准格式，来增强各种AI模型的可交互性。是由 Facebook 和 Microsoft 共同开发的，目的是让研究人员和工程师更容易在不同的深度学习框架和硬件平台之间迁移模型。

​		ONNX 的主要优点之一是它允许轻松地从一个框架（例如 `PyTorch`）导出模型，并导入到另一个框架（例如 `TensorFlow`）中。这对于想要尝试不同框架来训练和部署模型的研究人员，或者需要在不同硬件平台上部署模型的工程师特别有吸引力。

## ONNX Runtime

​		ONNX Runtime 是一个用于执行 ONNX（开放神经网络交换）模型的**开源推理引擎**[^1]。它被设计为高性能和轻量级，非常适合部署在各种硬件平台上，包括边缘设备、服务器和云服务。

​		ONNX Runtime时提供 `C++ API`、`C# API` 和 `Python API` 用于执行 ONNX 模型。还提供对多个后端的支持，包括 CUDA 和 OpenCL，这使得它可以在各种硬件平台上运行，例如 NVIDIA GPU 和 Intel CPU。

## ProtoBuf

​		**Protocol Buffer（简称 Protobuf）** 是 Google 开发的一种高效、跨语言的**数据序列化协议**，用于结构化数据的存储和传输。它的核心目标是提供一种比 XML 或 JSON **更小、更快、更简单**的二进制数据格式，同时支持多种编程语言（如 C++、Python、Java、Go 等）。

​		具有几个特性

- **二进制格式**：数据以紧凑的二进制形式存储，体积小、解析速度快（通常比 JSON/XML 快 3-10 倍）。
- **跨语言支持**：通过定义统一的 `.proto` 文件，可生成不同编程语言的代码，实现多语言数据交互。
- **强类型与结构化**：需预先定义数据结构（类似数据库表结构），确保数据格式的严格性。
- **向后/向前兼容**：支持字段的增删和修改，新旧版本协议可共存，避免破坏性升级。

​		核心组件是一个 `.proto` 文件：定义数据接口的接口描述文件；一个 `ProtoBuf` 编译器（`protoc`）将 `.proto` 文件编译为编程语言代码，生成序列化/反序列化接口，以及各语言依赖的 `ProtoBuf` 运行时库

[^1]: 开源推理引擎是一种**公开源代码**的软件工具或框架，主要用于在机器学习/深度学习模型中执行**推理任务**（即使用训练好的模型对新数据进行预测或决策）。它的核心目标是将训练好的模型高效部署到实际应用中

# ONNX格式解析

## onnx.proto

​		这是 ONNX 项目中的一个核心文件，定义了 ONNX 格式的数据结构和序列化规则。这个文件是基于 `Protocol Buffers` 的接口定义文件，用于生成不同编程语言（如 `C++、python`）的序列化/反序列化代码。该文件定义了一些数据结构，描述 ONNX 模型的所有组成部分，关键部分如下：

- 模型元信息（`ModelProto`）
  - 计算图（`GraphProto`）
    - 计算节点（`NodeProto[]`）
    - 权重参数（`TensorProto`）
    - 输入输出张量描述（`ValueInfoProto`）
    - 节点属性参数（`AttributeProto`）
  - 算子版本信息（`OperatorSetIdProto`）
  - 元数据/框架版本（`MetadataPropsProto`）

​		通过 `Protobuf` 编译器[^2]（``protoc`），`onnx.proto` 可以生成对应编程语言的类库，用于直接操作 ONNX 模型（如加载、修改、保存模型）。生成的类库中会包含：`proto` 文件中定义的数据结构、自动生成的序列化/反序列化方法

## ModelProto

​		加载了一个 ONNX 后，我们获得的就是一个 `ModelProto`，这是模型层，它包含了：

- ir_version：模型格式版本号（如 ONNX 1.7）
- producer_name：生成框架信息（如 PyTorch 1.8）
- graph：核心计算图结构 `GraphProto`。

## GraphProto

​		在 `GraphProto` 中又包含了四个 repeated 数组，分别是 node (`NodeProto`类型)，input (`ValueInfoProto`类型)，output (`ValueInfoProto`类型) 和 initializer (`TensorProto`类型)

```python
# 典型结构示例
graph {
  name: "resnet50"
  input { name: "input.1" type { tensor_type {...} } }
  output { name: "output.1" type { tensor_type {...} } }
  node {
    op_type: "Conv" 
    input: "input.1"
    input: "conv1.weight"
    output: "conv1_output"
    attribute { name: "strides" ints: 2 }
  }
  initializer {  // 存储权重参数
    name: "conv1.weight" 
    data_type: FLOAT
    dims: [64,3,7,7]
    raw_data: <binary>
  }
}
```

- 节点（`NodeProto`）：每个算子对应一个节点，记录算子类型（如Conv）、输入输出张量名称、属性参数（如 stride = 2）
- 输入输出张量描述（`ValueInfoProto`）：定义输入/输出张量的形状和类型（如 float32 [1, 3, 224 ,224]）
- 初始化器（`TensorProto`）：存储权重参数的二进制数据，支持FP16/INT8等量化格式

[^2]: **Protocol Buffer 编译器（protoc）** 是 Google 为 Protocol Buffers 设计的专用工具，用于将 `.proto` 文件（定义数据结构的接口描述文件）编译成目标编程语言的代码（如 Python、C++、Java 等）。生成的代码提供了序列化（对象转二进制）和反序列化（二进制转对象）的接口，使开发者无需手动处理二进制数据的底层细节。

## OperatorSetIdProto

​		声明使用的算子集版本（如 `opset_version=13`），确保不同框架转换时的版本兼容性

# 拓扑关系

​		每个计算节点（`NodeProto`）的`input`和`output`数组本质上是字符串类型的张量名称标识符，通过 `input` 和 `output `的指向关系，就可以利用上述信息快速构建出一个深度学习模型的拓扑图。

```protobuf
input { name: "conv1.weight" type { tensor_type {...} } }
node {
  op_type: "Conv"
  input: "input_data", "conv1.weight"
  output: "conv1_output"
}
```

​		这里要注意一下，`GraphProto`的`input`字段包含的不仅是传统意义上的数据输入节点，还包括**所有权重参数的逻辑入口**。这种设计源于ONNX将权重视为"隐式输入"的特殊处理：

- **权重存储分离**：实际权重数据存放于`initializer`数组中（如`TensorProto`类型的`conv1.weight`）
- 输入别名映射：`input` 数组会为每个`initializer` 创建同名条目，例如：

```protobuf
input { name: "conv1.weight" type { tensor_type {...} } }
initializer { name: "conv1.weight" data_type: FLOAT ... }
```

​		这种设计实现了权重参数的"声明与赋值分离"，既保持了计算图的逻辑完整性，又支持运行时的高效参数加载

​		最后，每个计算节点中还包含了一个 `AttributeProto` 数组，用来描述该节点的属性，采用键值对形式描述算子的超参数，以卷积层为例

```protobuf
attribute { 
  name: "strides" 
  ints: [2, 2] 
  type: INTS 
}
attribute {
  name: "pads"
  ints: [1, 1, 1, 1]
  type: INTS
}
```

​		比如 `Conv` 节点或者说卷积层的属性包含 `group`，`pad`，`strides` 等等。

​		每一个计算节点的属性通过强类型定义（INTS/FLOATS/STRING等）确保跨平台一致性。ONNX官方维护的[Operators.md](https://github.com/onnx/onnx/blob/master/docs/Operators.md)文档详细规定了每个算子的：

- 必需/可选属性列表
- 输入输出张量数量约束
- 数据类型兼容性规则
- 不同算子集版本的行为差异