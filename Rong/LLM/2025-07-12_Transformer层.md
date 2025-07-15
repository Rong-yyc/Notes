# Transformer层

​		Transformer 层在大语言模型中主要负责对输入文本进行多层次的特征提取和语义理解，可以类比为人类阅读时的“逐层深入思考”，在 `MiniMindModel` 中，主要是由 `MiniMindBlock` 类实现的，体现代码如下：

```python
self.layers = nn.ModuleList([MiniMindBlock(l, config) for l in range(self.num_hidden_layers)])
```

​		这里说明实际上我们设定的超参数 `num_hidden_layers` 指的是大语言模型中 `Transformer` 层的数量，并不包括归一化层和线性层等。在实例化 `MiniMindBlock` 时会传入一个整数 `id` 以及模型的配置。

​		`MiniMindBlock` 是模型中的一个核心组件，代表了 `Transformer` 架构中的一个基本块（或层），每个 `MiniMindBlock` 都包含自注意力机制和前馈网络，并应用了残差连接和层归一化，首先我们看类的定义和 `__init__` 函数		

```python
class MiniMindBlock(nn.Module):
    """继承了 nn.Module 类，好处是接入HuggingFace的生态，也使得代码更简洁"""
    def __init__(self, layer_id: int, config: MiniMindConfig):
        super().__init__()										# 调用父类的初始化方法
        # 从配置中获取注意力头的数量、隐藏层维度、以及每个注意力头的维度这些基本参数
        self.num_attention_heads = config.num_attention_heads	
        self.hidden_size = config.hidden_size
        self.head_dim = config.hidden_size // config.num_attention_heads
        
        # 记录当前层的 ID
        self.layer_id = layer_id
        
        # 实例化一个 Attention 对象
        self.self_attn = Attention(config)
        # 实例化一个 RMSNorm 对象用来处理输入的层归一化
        self.input_layernorm = RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
        # 实例化一个 RMSNorm 对象用来处理输出的层归一化
        self.post_attention_layernorm = RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
        # 根据 config.use_moe 的值来决定使用哪种前馈网络
        self.mlp = FeedForward(config) if not config.use_moe else MOEFeedForward(config)
```

## \_\_init\_\_方法

​		在该方法的第一行首先调用了父类 `nn.Module` 类的初始化函数，这是 `PyTorch` 模块的常见做法，具体这行代码做了些什么，可以查阅笔记 [Module类详解](../PyTorch/Module类详解.md)

​		然后紧接着从模型配置 `config` 中获取注意力头的数量、隐藏层维度、每个注意力头的维度等构建注意力层的基本参数。记录当前层的 `id`，后续可能需要用到

​		然后实例化了四个层，分别是 `Attention` 层以及两个归一化层和一个前馈网络层

## forward 方法

```python
def forward(self, hidden_states, position_embeddings, past_key_value=None, use_cache=False, attention_mask=None):
    # 保存输入的副本，用于后续的残差连接
    residual = hidden_states	
    # 自注意力计算，计算注意力权重和输出
    hidden_states, present_key_value = self.self_attn(
        # 对输入进行归一化，提升训练稳定性
        self.input_layernorm(hidden_states), position_embeddings,
        past_key_value, use_cache, attention_mask
    )
    # 残差连接
    hidden_states += residual
    # 对注意力输出进行归一化，带残差连接的前馈网络
    hidden_states = hidden_states + self.mlp(self.post_attention_layernorm(hidden_states))
    return hidden_states, present_key_value		
```

​		该层需要上一层的输出 `hidden_states, position_dmbeddings` 作为输入。`hidden_states` 自然就是上一层计算得到的隐藏状态，而 `position_embeddings` 则是位置编码，用于 `RoPE`

​		首先存储上一层的输出结果，将上一层输出进行归一化 `self.input_layernorm(hidden_states)`，然后直接调用 Attention 层 `self.self_attn` 计算得到结果，将上一层的输出结果加上得到残差结果 `hidden_states += residual`，交给 `self.post_attention_layernorm` 层进行输出归一化。最后交给前馈层计算并残差连接得到输出结果

​		这里的输出中：

- `hidden_states`：经过整个 Block 处理后的特征信息
- `present_key_value`：当前的 **KV 缓存**，用于推理时的增量计算

# 注意力层（Attention）

​		在 Transformer 层中，最核心、神秘的就是其中的 Attention 层，其实 Attention 机制模仿了人类的注意力机制——当处理一个词时，我们会根据上下文动态地关注其他相关的词

## 注意力层组成

​		其实在注意力层中，最重要的是三个矩阵：Q（Query）、K（Key）、V（Value）

```python
class Attention(nn.Module):
    def __init__(self, args: MiniMindConfig):
        # 调用父类的初始化函数
        super().__init__()
        
        # 配置共享键值头的数量等超参数
        self.num_key_value_heads = args.num_attention_heads if args.num_key_value_heads is None else args.num_key_value_heads
        assert args.num_attention_heads % self.num_key_value_heads == 0
        self.n_local_heads = args.num_attention_heads
        self.n_local_kv_heads = self.num_key_value_heads
        self.n_rep = self.n_local_heads // self.n_local_kv_heads
        self.head_dim = args.hidden_size // args.num_attention_heads
        
        # Query投影层
        self.q_proj = nn.Linear(args.hidden_size, args.num_attention_heads * self.head_dim, bias=False)
        # Key 投影层
        self.k_proj = nn.Linear(args.hidden_size, self.num_key_value_heads * self.head_dim, bias=False)
        # Value 投影层
        self.v_proj = nn.Linear(args.hidden_size, self.num_key_value_heads * self.head_dim, bias=False)
        # 输出投影层
        self.o_proj = nn.Linear(args.num_attention_heads * self.head_dim, args.hidden_size, bias=False)
        # 用于正则化，防止过拟合
        self.attn_dropout = nn.Dropout(args.dropout)
        self.resid_dropout = nn.Dropout(args.dropout)
        self.dropout = args.dropout
        # 是否使用高效的 Flash Attention
        self.flash = hasattr(torch.nn.functional, 'scaled_dot_product_attention') and args.flash_attn
```

## 位置编码（Position Embedding）

​		 `Transformer` 的自注意力机制本身不区分 `token` 的顺序，为了让模型知道每个 `token` 在序列中的位置，需要引入位置编码。**位置编码通过修改输入表示**，使模型能够区分不同位置的词元。这部分我觉得篇幅占比有点大，可以单独作为一篇笔记 [位置编码详解](./2025-07-10_位置编码.md) 

## 数据流动

​		下面将 Attention 的 forward 方法分块讲解，首先是处理输入

```python
def forward(self, x: torch.Tensor, 				# 输入 
			position_embeddings: Tuple[torch.Tensor, torch.Tensor],
            past_key_value: Optional[Tuple[torch.Tensor, torch.Tensor]] = None,
            use_cache=False,
            attention_mask: Optional[torch.Tensor] = None):
        bsz, seq_len, _ = x.shape
```

- 输入：形状为 `[batch_size, seq_len, hidden_dim]` 的序列 `X`

- `position_embeddings`：位置编码（`RoPE` 使用的是 `cos/sin`）

- `past_key_value`：历史KV缓存（推理用）

- `attention_mask`：掩码（处理padding等）


​		获取输入的形状，拿到 `batch_size, seq_len` 等信息

```python
xq, xk, xv = self.q_proj(x), self.k_proj(x), self.v_proj(x)
xq = xq.view(bsz, seq_len, self.n_local_heads, self.head_dim)
xk = xk.view(bsz, seq_len, self.n_local_kv_heads, self.head_dim)
xv = xv.view(bsz, seq_len, self.n_local_kv_heads, self.head_dim)
```

​		将输入张量 x（通常是词嵌入和位置编码的组合）通过三个独立的线性层进行线性变换，分别生成查询（Query, xq）、键（Key, xk）和值（Value, xv）这三个向量。

​		接下来三行是对这三个向量进行 `reshape`，主要是为了实现多注意力，将原始的 `[bsz, seq_len, d_model]` 的向量拆分为 `n_local_heads` 个低维度的”头“，每个头的维度是 `head_dim`，得到一个 `[bsz, seq_len, n_local_heads, head_dim]` 形状的张量

```python
cos, sin = position_embeddings
xq, xk = apply_rotary_pos_emb(xq, xk, cos[:seq_len], sin[:seq_len])
```

​		从预计算好的 `position_embeddings` 中取出 `cos` 和 `sin` 两个大表

​		然后调用 `apply_rotary_pos_emb` 函数将位置信息”旋转”到查询张量和键向量中

```python
# kv_cache实现
if past_key_value is not None:
    xk = torch.cat([past_key_value[0], xk], dim=1)
    xv = torch.cat([past_key_value[1], xv], dim=1)
past_kv = (xk, xv) if use_cache else None
```

​		这部分代码主要用于实现 `kv_cache`，如果设定了 `use_cache` 为 True，则每次计算出 `xk` 和 `xv` 后，将其记录在 `past_kv` 中。这里的 `past_key_value` 是由框架负责传递的参数，在预测第一个 token 时，就是 None。第二轮及以后，框架就会自动帮你把前一轮的 xk, xv 拼接在 `past_key_value` 中，这样就实现了 `KV` 缓存

```python
xq, xk, xv = (
    xq.transpose(1, 2),
    repeat_kv(xk, self.n_rep).transpose(1, 2),
    repeat_kv(xv, self.n_rep).transpose(1, 2)
)
```

​		这段代码实现了 GQA（分组查询注意力）。在这种模式下，会有一些注意力头共享 KV 矩阵，也就是说 Query 的头数多于 KeyValue 的头数。这里使用 `repeat_kv` 函数将 KV 头重复多次，使其头数与 `xq` 的头数匹配。然后交换三个矩阵的第一个和第二个维度，目的是为了使用高效的库函数，及下面的 `scaled_dot_product_attention` 函数。这种库函数通常要求张量的形状是 `[bsz, num_heads, seq_len, head_dim]`

```python
if self.flash and seq_len != 1:
    # 设置 dropout 的概率，是一种只在模型训练阶段使用的正则化技术，用于防止过拟合，非训练阶段设为 0
    dropout_p = self.dropout if self.training else 0.0
    # 处理 attention_mask，用于告诉模型哪些地方是有效词元，哪些地方是 padding
    attn_mask = None
    if attention_mask is not None:
        # 将掩码形状调整为与多头注意力的内部计算维度相匹配的格式
        attn_mask = attention_mask.view(bsz, 1, 1, -1).expand(bsz, self.n_local_heads, seq_len, -1)
        # 将掩码转换为布尔类型 
        attn_mask = attn_mask.bool() if attention_mask is not None else None

    # 使用内置的函数，完成官方的 Flash Attention 计算
    output = F.scaled_dot_product_attention(xq, xk, xv, attn_mask=attn_mask, dropout_p=dropout_p, is_causal=True)
else:
    ...
```

​		这段部分代码实际上就是注意力计算的核心部分，其中 `if` 分支处理的是启用 Flash Attention[^1] 且序列长度不为 1（通常是第一次处理 prompt）的情况， `else` 分支处理的是未使用 `Flash Attention` 时的标准手动实现。使用了 `PyTorch` 2.0 版本及以上提供的 `Flash Attention` 来高效的完成注意力分数的计算和加权求和。

> **备注：**`Flash Attention` 的主要优势在于它通过 `tiling`（分块）和重新计算的技术来避免实例化巨大的 `(seq_len, seq_len)` 注意力矩阵，从而大大减少了显存占用和 IO 开销，在 `seq_len` 比较长的时候非常显著。但是，当模型处于自回归生成阶段，每次只处理一个新 token 时，`seq_len` 就等于1。在这种情况下，注意力矩阵只是一个 `(1, d_model)` 的向量，本身并不大，`Flash Attention` 的复杂调度机制反而可能没有传统的实现快。

```python
else:
    # 计算注意力分数
    scores = (xq @ xk.transpose(-2, -1)) / math.sqrt(self.head_dim)
    scores = scores + torch.triu(
        torch.full((seq_len, seq_len), float("-inf"), device=scores.device),
        diagonal=1
    ).unsqueeze(0).unsqueeze(0)  # scores+mask

    # 手动应用注意力掩码
    if attention_mask is not None:
        extended_attention_mask = attention_mask.unsqueeze(1).unsqueeze(2)
        extended_attention_mask = (1.0 - extended_attention_mask) * -1e9
        scores = scores + extended_attention_mask

    # 进行归一化
    scores = F.softmax(scores.float(), dim=-1).type_as(xq)
    # 正则化防止过拟合
    scores = self.attn_dropout(scores)
    # 最后与 xv 相乘
    output = scores @ xv
```

​		这部分代码实现的是标准的、手动的缩放点积注意力算法：通过矩阵乘法计算得分，缩放，手动应用掩码，进行 `softmax` 归一化后，再与 `xv` 相乘得到输出

```python
output = output.transpose(1, 2).reshape(bsz, seq_len, -1)
output = self.resid_dropout(self.o_proj(output))
return output, past_kv
```

[^1]: `PyTorch` 2.0 及以上版本提供 `Flash Attention` 来高效的完成注意力分数的计算和加权求和。（通过 `scaled_dot_product_attention` 函数实现） ↩

## Q&A

1、`num_key_value_heads` 和 `num_attention_heads` 分别是什么？他们有什么关系？		

答：`attention_head` 是 Transformer 中并行计算的独立注意力单元，每个头都有自己的 Q、K、V 权重矩阵，每个头从不同角度学习序列中词与词的关系；`key_value_head` 是键值头，在某些高效变体模型中，多个注意力头共享一组键值对，能够减少计算量和内存占用，同时保留多查询头的灵活性，此时 `num_key_alue_heads` 表示共享的键值头数量。

2、位置编码的原理是什么？

答：通过将词元的绝对位置或相对位置加入计算，并将得到的结果一起参与计算，就能隐式的将词元的位置信息给到神经网络

# Q、K、V矩阵

## Q、K、V的生成

​		首先我们要明确，在注意力层中，实际上Q、K、V是通过将输入与三个矩阵 $W_Q,W_K,W_V$ 相乘得到的，这三个矩阵的参数会在训练中不断更新，实现Q、K、V三个矩阵的更新，即

​		$Q=XW_Q, K=XW_K, V=XW_V$

​		这三个矩阵的作用是将输入投影到不同的语义空间，便于后续计算

## Q、K的作用

​		首先，Q 是用来标记当前关注的词元，可以先考虑为单个 token，形状为 `(1, d_model)`，然后需要计算其与其他所有 token 的相关性，而 K 就相当于所有的 token 信息，形状为 `(seq_len, d_model)`，用 $QK^T$ 就能得到一个基础的，当前 token 与所有 token 的相关性。然后要计算每一个 token 与其他所有 token 的关系，所有就是`seq_len` 个 token 与其他所有 token 的关系。所有 Q 矩阵的形状也是 `(seq_len, d_model)`

​		所以总结下来，Q 和 K 是用于得到 token 之间的相关性分数，然后经过缩放（除以 $\sqrt{d_{model}}$）、归一化（`softmax` 归一化）

​		最后将被提取的 “信息” 即为注意力权重

## V的作用

​		其实按道理来说，根据直觉，将注意力权重直接乘以 X 就可以了，为什么还需要 V 矩阵呢？其实是因为在 X 的原始嵌入中可能包括多种混杂信息（如语义、位置、语法等），而 V 通过 $W_V$ 学习到哪些信息时候被注意力机制提取。

*示例：*以 “猫 吃 鱼” 为例，猫的原始嵌入可能编码了 “动物”、“名词” 等信息，而 V 中的 “猫” 可能强化了 “动作发起者” 这一角色

> 所以在注意力层中，Q、K、V都有着自己的作用，Q/K 矩阵用来学习相关性，V 矩阵用来提取更适合被注意力机制使用的信息

## 训练与推理

​		在训练时，更新的是矩阵 $W_Q, W_K, W_V$，在进行推理时不同的输入乘以这三个矩阵会得到不同的Q、K、V矩阵。且由于在生成式任务中，已生成 token 的 K、V 计算并不依赖于未来的 token，所以可以通过缓存的方式，减少每次推理时的计算开销

| 矩阵  | 存储信息类型           | 在“猫吃鱼”中的角色                                           |
| ----- | ---------------------- | ------------------------------------------------------------ |
| **Q** | **当前词的“问题”**     | “吃”的 Query 会问：“谁在吃？（主语）”“吃什么？（宾语）”      |
| **K** | **其他词的“回答标签”** | “猫”的 Key 回答：“我是主语”，“鱼”的 Key 回答：“我是宾语”     |
| **V** | **其他词的“具体答案”** | “猫”的 Value 提供：“我是猫，动物，施事者”，“鱼”的 Value 提供：“我是鱼，食物，受事者” |

## Q&A

1、为什么需要这三个矩阵？

答：**灵活性**：Q/K 负责动态匹配关系，V 负责编码具体信息，解耦后模型能更高效地学习不同功能。  

​      **可解释性**：例如，某些注意力头可能专门让动词关注主语（通过 Q-K 匹配），而 Value 提供主语的语义细节。

# 前馈层（Feed-Forward Network，FFN）

​		自注意力机制本身虽然非常强大，但其核心计算在很大程度上是线性的。如果模型中只有线性的计算，那么无论堆叠多少层，其最终效果也等同于一个单层的线性模型，这将极大地限制模型的表达能力。前馈层就是为了解决这个问题，引入了一个非线性激活函数，打破了这种线性。

## \_\_init\_\_方法

```python
class FeedForward(nn.Module):
    def __init__(self, config: MiniMindConfig):
        super().__init__()
        if config.intermediate_size is None:
            intermediate_size = int(config.hidden_size * 8 / 3)
            config.intermediate_size = 64 * ((intermediate_size + 64 - 1) // 64)
        self.gate_proj = nn.Linear(config.hidden_size, config.intermediate_size, bias=False)
        self.down_proj = nn.Linear(config.intermediate_size, config.hidden_size, bias=False)
        self.up_proj = nn.Linear(config.hidden_size, config.intermediate_size, bias=False)
        self.dropout = nn.Dropout(config.dropout)
        self.act_fn = ACT2FN[config.hidden_act]
```

​		这段代码首先判断是否已经指定了前馈层大小，如果没有则将 `hidden_size` 乘以 8/3，这是一个在现代大语言模型中被证明行之有效的经验法则，然后将计算出的 `intermediate_size` 向上取整到最接近的 64 的倍数。

​		紧接着是初始化网络层，定义了三个线性层和一个激活函数层，它们共同实现了一种名为 `SwiGLU (Swish Gated Linear Unit)` 的先进激活机制。

- `self.gate_proj`：门控投影层，将输入从 `hidden_size` 扩展到更大的 `intermediate_size`
- `self.up_proj`：上投影层，与 `gate_proj` 并行，也执行相同的维度扩展
- `self.down_proj`：下投影层，在经过门控和激活函数处理后，它负责将维度从 `intermediate_size` 压缩回原始的 `hidden_size`
- `self.dropout`：定义了一个标准的 Dropout 层，用于正则化，防止过拟合
- `self.act_fun`：从 transformers 库中获取激活函数，根据配置文件使用的是 `silu`

##  forward方法

```python
def forward(self, x):
    return self.dropout(self.down_proj(self.act_fn(self.gate_proj(x)) * self.up_proj(x)))
```

​		这里就完美的实现了 “门控机制”[^2]。`self.up_proj` 是数据通路，承载着主要信息。`self.act_fn(self.gate_proj(x))` 是门控通路，将其相乘就是通过门控向量，动态的筛选调节，逐个元素地去控制数据通路中信息的去留和强弱

[^2]: 门控机制是神经网络学会的一种“注意力”或“控制”能力，使其能够智能地、动态地筛选和调节其内部处理的信息。就像在出水处装了一个水龙头，可以控制水流开关以及大小，如果没有则只有开关
