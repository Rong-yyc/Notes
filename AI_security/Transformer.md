# AI 概述

## 模型组成	

​		我们平时所说的大模型，主要构成有三个部分：模型架构、配置信息以及权重参数。为了方便理解，可以类比于汽车：

**模型架构：**模型架构就类似于汽车的种类，是小轿车、SUV还是跑车、越野，不同的车型都有其独特的设计和用途；同理，不同的神经网络架构也适用于不同类型的任务。例如：

- 多层感知机（MLP）：适合基本的分类任务
- 卷积神经网络（CNN）：擅长处理图像数据
- 递归神经网络（RNN）或长短时记忆网络（LSTM）：专为序列化数据设计
- Transformer：适用于各种复杂的自然语言处理任务

**配置文件：**配置文件就像确定了汽车的类型后，具体这款车的详细规格，如车轮大小、车身长度及发动机排量等；同理，即使选择了某种模型架构，不同型号之间依旧存在差异，这些配置会影响模型的复杂度、计算成本以及最终的表现。如：

- 对于 Transformer 模型而言，隐藏层的数量、注意力头的数量、前馈神经网络的维度等参数

**权重：**权重就像是各个厂家对同一款车型的不同调校，使得地盘更稳定、悬挂系统更舒适、动力响应更快等，从而让车子在对于的场景中表现出更好的性能；同理，即使是相同的模型和相同的配置，不同的权重导致模型有不同的表现。如：

- 两个基于相同 Transformer 架构的模型，使用不同的预训练权重，它们在文本生成、翻译会问答等任务上表现可能会有显著差异。

# Transformers 库

​		`HunggingFace` 的 `Transformers` 库包含了大量的模型架构，同时提供了丰富的 API 来简化和扩展模型的使用。这些 API 可以大致分为几类：模型加载、数据处理、推理预测、训练与微调以及一些辅助工具。

## from_pretrained() 方法

​		`Transformer` 里调用 `MODEL.from_pretrained` 方法时会执行以下过程：

- **加载预训练模型：**可以选择从本地加载已下载的预训练模型[^1]，或者提供模型名称从 `HuggingFace` 的模型仓库下载预训练模型；
- **加载配置：**加载预训练模型的配置文件[^2]，配置包含了模型的名称、架构、参数等信息。这些配置参数用来定义模型的结构；
- **初始化模型：**使用配置文件中的参数初始化模型，构建模型的各个层和结构；
- **载入权重：**将预训练模型权重载入到初始化的模型结构中；
- **创建实例：**返回加载了权重的模型实例，利用这个实例进一步完成特定的下游任务

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification

model_name = "bert-base-uncased"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModel.from_pretrained(model_name)
```

​		当使用 `from_pretrained()` 方法加载模型时，配置文件和预训练权重通常是从 Hugging Face的模型仓库[Model Hub](https://huggingface.co/models) 下载的。通常配置文件是 JSON 格式，包含了模型的超参数[^3]和其他设置。如下（部分作了解释）：

```json
{
  "architectures": [
    "Qwen2ForProcessRewardModel"								// 模型使用的架构
  ],
  "attention_dropout": 0.0,										// 注意力机制的丢弃率，表示不丢弃
  "auto_map": {														
    "AutoConfig": "configuration_qwen2_rm.Qwen2RMConfig",		// 自动配置类的映射 
    "AutoModel": "modeling_qwen2_rm.Qwen2ForProcessRewardModel"	// 自动模型类的映射
  },
  "bos_token_id": 151643,										// 开始标记的 ID
  "eos_token_id": 151645,										// 结束标记的 ID
  "hidden_act": "silu",											// 隐藏层激活函数，用于给神经网络引入非线性
  "hidden_size": 3584,											// 隐藏层神经元的数量，决定了模型的复杂度和表示能力
  "initializer_range": 0.02,									// 初始化权重的范围
  "intermediate_size": 18944,									// 中间层的大小，会影响信息的传递和处理
  "max_position_embeddings": 4096,								// 最大位置嵌入数
  "max_window_layers": 28,										// 最大窗口层数
  "model_type": "qwen2",										// 模型类型，该模型属于 qwen2 类型
  "num_attention_heads": 28,									// 注意力头的数量，多个注意力头可以并行处理信息
  "num_hidden_layers": 28,										// 隐藏层的层数，影响模型深度和性能的关键因素
  "num_key_value_heads": 4,										// 键值对注意力头的数量
  "rms_norm_eps": 1e-05,										// RMS 归一化的小正数参数
  "rope_theta": 10000.0,										
  "sliding_window": 131072,										// 滑动窗口的大小
  "tie_word_embeddings": false,									// 是否绑定词嵌入
  "torch_dtype": "bfloat16",									// 指定在PyTorch中使用的数据类型，半精度浮点数
  "transformers_version": "4.37.0",								// Transformers 库的版本
  "use_cache": true,											// 是否使用缓存
  "use_mrope": false,									
  "use_sliding_window": false,									// 是否使用滑动窗口
  "vocab_size": 152064											// 词汇表的大小
}
```

​		预训练权重则取决于使用的框架，可以是 `PyTorch` 或 `TensorFlow` 格式，详见文档 [模型文件格式](https://github.com/Rong-yyc/Notes/tree/main/AI_security/模型文件格式.md) 

- `PyTorch`：一般为二进制文件，扩展名为 `.bin`、`.pt` 或 `.pth`
- `TensorFlow`：可能是 `.ckpt` 文件或是 SaveModel 格式

[^1]: 当用户安装 `Transformer` 库的时候，许多i模型架构就已经以 Python 包的形式保存在本地环境中了。模型架构是以 Python 源代码（.py）文件的形式保存的，每个模型类（如`BertModel`、`T5ForConditionalGeneration`等）都有对应的Python文件，这些文件包含了模型架构的定义。
[^2]: 虽然模型架构是随库一起安装的，但**配置文件 `config.json` 并不会自动下载**。它们只有在你首次使用`from_pretrained()` 方法加载特定模型时才会被下载，并且会被缓存到一个特定的目录中，通常是：`~/.cache/huggingface/transformers/`（Linux/macOS）或 `%APPDATA%\Hugging Face\transformers\`（Windows）。这些文件会在未来的请求中直接从本地加载，以避免重复下载。
[^3]: 在机器学习和深度学习中，**超参数（Hyperparameters）**是指那些在模型训练开始之前需要设置的参数，它们不是通过训练数据直接学习到的，而是由用户或自动化工具选择和调整的，如隐藏层的数量及其大小、学习率、最大迭代次数等。超参数对模型的性能有着重要影响，因为它们控制着模型的学习过程和结构。

# AutoModel.from_pretrained() 原理【[参考](https://blog.csdn.net/weixin_41878387/article/details/138681520)】

```python
model = AutoModel.from_pretrained(model_name)
```

​		实际上 `AutoModel.from_pretrained()` 其实是调用了父类 `_BaseAutoModelClass` 里面的 `from_pretrained` 方法。该父类位于 `transformers/models/auto/auto_factory.py` 文件中，根据代码逻辑，先加载配置文件。

## 加载配置文件

​		首先通过 `AutoConfig` 类加载对应的配置文件。

```python
config, kwargs = AutoConfig.from_pretrained(
                pretrained_model_name_or_path,
                return_unused_kwargs=True,
                trust_remote_code=trust_remote_code,
                code_revision=code_revision,
                _commit_hash=commit_hash,
                **hub_kwargs,
                **kwargs,
            )
```

​		该类位于 `transformers/models/auto/configuration_auto.py` 文件中，在该类的 `from_pretrained()` 方法中通过如下代码加载模型配置参数：

```python
config_dict, unused_kwargs = PretrainedConfig.get_config_dict(pretrained_model_name_or_path, **kwargs)
```

​		而 `PretrainedConfig` 类位于 `transformers/configuration_utils.py` 文件中，在 `get_config_dict()` 方法中，实际上调用的是 `_get_config_dict()` 方法，该方法部分关键源代码如下：

```python
if os.path.isfile(os.path.join(subfolder, pretrained_model_name_or_path)):
    # 比较特殊的情况下配置文件是一个本地文件
    resolved_config_file = pretrained_model_name_or_path
    is_local = True
elif is_remote_url(pretrained_model_name_or_path):
    # 如果是需要下载的配置文件，这里会是一个远程的 url
    configuration_file = pretrained_model_name_or_path if gguf_file is None else gguf_file
    # 通过 download_url() 方法下载配置文件
    resolved_config_file = download_url(pretrained_model_name_or_path)
...
try:
    if gguf_file:
        config_dict = load_gguf_checkpoint(resolved_config_file, return_tensors=False)["config"]
    else:
        # Load config dict
        config_dict = cls._dict_from_json_file(resolved_config_file)
...
return config_dict, kwargs
```

​		通过 `download_url()` 方法下载文件时，源代码如下：

```python
def download_url(url, proxies=None):
	tmp_fd, tmp_file = tempfile.mkstemp()
    with os.fdopen(tmp_fd, "wb") as f:
        http_get(url, f, proxies=proxies)
    return tmp_file
```

​		这里返回的 `tmp_file` 实际上是 `tmp_fd` 的路径，会被 `resolved_config_file` 接收，根据 `gguf_file` 的标记，如果是 `gguf` 格式文件，则被 `load_gguf_checkpoint()` 方法处理，如果不是则由 `_dict_from_json_file()` 方法处理。

### load_gguf_checkpoint()方法

​		该方法位于文件 `transformers/modeling_gguf_pytorch_utils/py` 文件中，主要用于从 `gguf` 格式文件还原配置参数

- [ ] 阅读laod_gguf_checkpoint()源码

### _dict_from_json_file() 方法

​		该方法主要用于从 `json` 格式文件中还原配置参数

## 加载预训练模型

```python
if has_remote_code and trust_remote_code:
    class_ref = config.auto_map[cls.__name__]
    model_class = get_class_from_dynamic_module(
        class_ref, pretrained_model_name_or_path, code_revision=code_revision, **hub_kwargs, **kwargs
    )
    _ = hub_kwargs.pop("code_revision", None)
    cls.register(config.__class__, model_class, exist_ok=True)
    model_class = add_generation_mixin_to_remote_model(model_class)
    return model_class.from_pretrained(
        pretrained_model_name_or_path, *model_args, config=config, **hub_kwargs, **kwargs
    )
elif type(config) in cls._model_mapping.keys():
    model_class = _get_model_class(config, cls._model_mapping)
    return model_class.from_pretrained(
        pretrained_model_name_or_path, *model_args, config=config, **hub_kwargs, **kwargs
    )
raise ValueError(
    f"Unrecognized configuration class {config.__class__} for this kind of AutoModel: {cls.__name__}.\n"
    f"Model type should be one of {', '.join(c.__name__ for c in cls._model_mapping.keys())}."
)
```

### 加载远程模型

​		如果加载的是远程仓库的模型且被信任的话，就是第一个 `if` 分支。首先通过 `config.auto_map` 从预训练模型的映射中获取对应的模型类，然后通过 `get_class_from_dynamic_module()` 函数，加载该模型类。该函数位于 `transformers/dynamic_module_utils.py` 文件中，部分关键代码如下：

```python
if "--" in class_reference:
	repo_id, class_reference = class_reference.split("--")
else:
	repo_id = pretrained_model_name_or_path
...
final_module = get_cached_module_file(
        repo_id,
        module_file + ".py",
        cache_dir=cache_dir,
        force_download=force_download,
        resume_download=resume_download,
        proxies=proxies,
        token=token,
        revision=code_revision,
        local_files_only=local_files_only,
        repo_type=repo_type,
    )
return get_class_in_module(class_name, final_module, force_reload=force_download)
```

​		可见实际上调用的是`get_cached_module_file()` 方法。在该方法中面对远程模型时，会先检测缓存中有无该文件，没有的话就调用 `cached_file()` 方法，而在 `cached_file()` 方法中，会调用 `hf_hub_download()` 方法来下载。该方法主要用于本地缓存中没有的文件，最终返回的是下载到本地后文件的路径。而后被返回到 `get_cached_module_file()` 方法中，该方法后续会根据下载的模型创建动态模块，将解析后的模块文件及其依赖文件复制到缓存目录中，最后返回缓存模块文件的路径给 `get_class_from_module()` 方法。

​		而后在 `get_class_from_dynamic_module()` 方法中，最后会返回模块中的类名给到 `_BaseAutoModelClass.from_pretrained()` 方法中的 `model_class`

### 加载本地模型

​		如果加载的是本地下载的预训练模型，则是 `elif` 分支，会先利用配置文件 `config.json` 中的 `architecrtures` 这个属性来匹配要加载的模型（调用 `_get_model_class` 方法），然后调用模型类 `model_class` 的 `from_pretrained` 方法来加载模型。

## 加载权重参数

​		在 `_BaseAutoModelClass.from_pretrained()` 方法中， 无论是远程模型还是本地模型，最后都是通过 `model_clsss.from_pretrained()` 方法来将权重参数赋给大模型的。而实际上 `from_pretrained()` 是写在所有预训练模型类的基类 `PreTrainedModel` 类中的，位于 `transformers/modeling_utils.py` 文件中。子类重写该方法的必要性通常取决于模型的特定需求和功能。

# Torch

​		除了上述加载模型的方式以外，还可以保存整个模型文件为 `.pt` 格式。例如通过 `torch.save` 保存了一个完整的 `nn.Module` [^4]对象，就可以直接用如下代码加载：

```python
import torch
model = torch.load('model.pt')
```

[^4]: `nn.Module`是 PyTorch 中用于构建神经网络的基类。它是一个抽象类，提供了许多有用的功能和方法，是 PyTorch 中构建神经网络模型的基础。