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

​		 `Transformer` 的自注意力机制本身不区分 `token` 的顺序，为了让模型知道每个 `token` 在序列中的位置，需要引入位置编码。**位置编码通过修改输入表示**，使模型能够区分不同位置的词元。位置编码有两种主流方式

### 1、绝对位置编码

​		在绝对位置编码中，其本质上是一个和 Attention 输入相同形状的向量，是通过计算得到的每个词元的位置信息，然后在 Attention 开始计算 Q、K、V 之前，将位置编码加到输入上，使得输入中包含每个词元的位置信息。

- 经典方法**（原始 Transformer）**：使用固定公式生成正弦/余弦编码，可外推到比训练更长的序列
- 可学习的位置编码：直接训练一个位置嵌入矩阵 `nn.Embedding(max_len, d_model)`，更灵活但可能受限于训练时的最大长度

### 2、相对位置编码

​		在相对位置编码中，位置编码是一个与**注意力分数矩阵**形状相关的张量，尺寸为 `[seq_len, seq_len, d_dim]`

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

```python
def forward(self,
                x: torch.Tensor,
                position_embeddings: Tuple[torch.Tensor, torch.Tensor],  # 修改为接收cos和sin
                past_key_value: Optional[Tuple[torch.Tensor, torch.Tensor]] = None,
                use_cache=False,
                attention_mask: Optional[torch.Tensor] = None):
        bsz, seq_len, _ = x.shape
        xq, xk, xv = self.q_proj(x), self.k_proj(x), self.v_proj(x)
        xq = xq.view(bsz, seq_len, self.n_local_heads, self.head_dim)
        xk = xk.view(bsz, seq_len, self.n_local_kv_heads, self.head_dim)
        xv = xv.view(bsz, seq_len, self.n_local_kv_heads, self.head_dim)

        cos, sin = position_embeddings
        xq, xk = apply_rotary_pos_emb(xq, xk, cos[:seq_len], sin[:seq_len])

        # kv_cache实现
        if past_key_value is not None:
            xk = torch.cat([past_key_value[0], xk], dim=1)
            xv = torch.cat([past_key_value[1], xv], dim=1)
        past_kv = (xk, xv) if use_cache else None

        xq, xk, xv = (
            xq.transpose(1, 2),
            repeat_kv(xk, self.n_rep).transpose(1, 2),
            repeat_kv(xv, self.n_rep).transpose(1, 2)
        )

        if self.flash and seq_len != 1:
            dropout_p = self.dropout if self.training else 0.0
            attn_mask = None
            if attention_mask is not None:
                attn_mask = attention_mask.view(bsz, 1, 1, -1).expand(bsz, self.n_local_heads, seq_len, -1)
                attn_mask = attn_mask.bool() if attention_mask is not None else None

            output = F.scaled_dot_product_attention(xq, xk, xv, attn_mask=attn_mask, dropout_p=dropout_p, is_causal=True)
        else:
            scores = (xq @ xk.transpose(-2, -1)) / math.sqrt(self.head_dim)
            scores = scores + torch.triu(
                torch.full((seq_len, seq_len), float("-inf"), device=scores.device),
                diagonal=1
            ).unsqueeze(0).unsqueeze(0)  # scores+mask

            if attention_mask is not None:
                extended_attention_mask = attention_mask.unsqueeze(1).unsqueeze(2)
                extended_attention_mask = (1.0 - extended_attention_mask) * -1e9
                scores = scores + extended_attention_mask

            scores = F.softmax(scores.float(), dim=-1).type_as(xq)
            scores = self.attn_dropout(scores)
            output = scores @ xv

        output = output.transpose(1, 2).reshape(bsz, seq_len, -1)
        output = self.resid_dropout(self.o_proj(output))
        return output, past_kv
```

## Q&A

1、`num_key_value_heads` 和 `num_attention_heads` 分别是什么？他们有什么关系？		

答：`attention_head` 是 Transformer 中并行计算的独立注意力单元，每个头都有自己的 Q、K、V 权重矩阵，每个头从不同角度学习序列中词与词的关系；`key_value_head` 是键值头，在某些高效变体模型中，多个注意力头共享一组键值对，能够减少计算量和内存占用，同时保留多查询头的灵活性，此时 `num_key_alue_heads` 表示共享的键值头数量。

2、位置编码的原理是什么？

答：