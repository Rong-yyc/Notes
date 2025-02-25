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
    - 算子节点（`NodeProto`）
    - 张量（`TensorProto`）
    - 输入节点（`ValueInfoProto`）
    - 输出节点（`ValueInfoProto`）

​		通过 `Protobuf` 编译器[^2]（``protoc`），`onnx.proto` 可以生成对应编程语言的类库，用于直接操作 ONNX 模型（如加载、修改、保存模型）。生成的类库中会包含：`proto` 文件中定义的数据结构、自动生成的序列化/反序列化方法

## ModelProto

​		加载了一个 ONNX 后，我们获得的就是一个 `ModelProto`，它包含了一些版本信息，生产者信息和一个 `GraphProto`。

## GraphProto

​		在 `GraphProto` 中又包含了四个 repeated 数组，分别是 node (`NodeProto`类型)，input (`ValueInfoProto`类型)，output (`ValueInfoProto`类型) 和 initialize (`TensorProto`类型)

- node 中存放了模型中的所有计算节点
- input 中存放了模型的输入节点
- output 中存放了模型的所有输出节点
- initialize 中存放了模型的所有权重参数

[^2]: **Protocol Buffer 编译器（protoc）** 是 Google 为 Protocol Buffers 设计的专用工具，用于将 `.proto` 文件（定义数据结构的接口描述文件）编译成目标编程语言的代码（如 Python、C++、Java 等）。生成的代码提供了序列化（对象转二进制）和反序列化（二进制转对象）的接口，使开发者无需手动处理二进制数据的底层细节。

# 拓扑关系

​		拓扑关系在ONNX中是如何表示的呢？ONNX的每个计算节点都会有 `input` 和 `output` 两个数组，这两个数组是string类型，通过 `input` 和 `output `的指向关系，就可以利用上述信息快速构建出一个深度学习模型的拓扑图。

​		这里要注意一下，`GraphProto` 中的 `input` 数组不仅包含一般理解中的计算图输入的那个节点，还包含了模型中所有的权重。例如，`Conv` 层里面的 `W `权重实体是保存在 `initializer` 中的，那么相应的会有一个同名的输入在 `input` 中，其背后的逻辑应该是把权重也看成模型的输入，并通过 `initializer` 中的权重实体来对这个输入做初始化，即一个赋值的过程。

​		最后，每个计算节点中还包含了一个 `AttributeProto` 数组，用来描述该节点的属性，比如 `Conv` 节点或者说卷积层的属性包含 `group`，`pad`，`strides` 等等，每一个计算节点的属性，输入输出信息都详细记录在[Operators.md](https://github.com/onnx/onnx/blob/master/docs/Operators.md)。