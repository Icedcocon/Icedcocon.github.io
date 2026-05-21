# KV Cache 对推理效率的影响


## 0 计算量分析

> [!NOTE]
`FLOPs`：floating point operations 指的是浮点运算次数，一般特指乘加运算次数，**理解为计算量**，可以用来衡量算法/模型时间的复杂度。
>
对于矩阵 $A \in R^{1 \times n}$ 和 $B \in R^{n \times 1}$ 的矩阵乘法的 `FLOPs` 为 $2n$；**对于矩阵 $A \in R^{m \times n}$ 和 $B \in R^{n \times p}$ 的矩阵乘法的 `FLOPs` 为 $2mnp$**。

`Pytorch` 实现线性层的函数为 `nn.Linear(in_features, out_features, bias=True)`，其中线性层权重的维度大小是 `[out_features, in_features]`，对应的计算公式为:

$$ y = xW^T + bias $$

线性层（全连接层/映射层）的 `FLOPs` 计算：假设 $I$ 是输入层的维度，$O$ 是输出层的维度，对应全连接层（线性层）的权重参数矩阵维度为 $[I, O]$。

- 不考虑 bias，全连接层的 $$ \text{FLOPs} = (I + I - 1) \times O = (2I - 1)O \approx 2IO $$
- 考虑 bias，全连接层的 $$ \text{FLOPs} = (I + I - 1) \times O + O = 2IO $$

对于 transformer 模型来说，其计算量**主要**来自 `MHA` 层和 `MLP` 层中的矩阵乘法运算。先考虑 `batch_size = 1` 和 输入序列长度为 $s$ 的情况。

### 0.1 MHA(Attention) 层计算量

对于 Attention 层，输入输出矩阵 `QKVO` 大小一模一样，形状都是 $[s, h]$。

#### 0.1.1 prefill 阶段

先分析 `MHA` 块的计算量：

1. **计算 Q、K、V**：对输入矩阵做线性变换，输入 `tokens` 序列的 **`embedding` 向量**的形状为 $[s, h]$，做线性变换的权重矩阵 $W_Q, W_K, W_V \in R^{h \times h}$，矩阵乘法的输入输出形状为: $$ [s, h] \times [h, h] \rightarrow [s, h] $$ `FLOPs`: $$ 3 \times 2sh^2 = 6sh^2 $$

2. **Self-Attention 层**，`MHA` 包含 `heads` 数目的 `Self-Attention` 层，这里直接分析所有 `Self-Attention` 层的 `FLOPs`:

    - **$QK^T$ 打分计算**：每个头需要计算 Query 和 Key 的点积，所有头的 $QK^T$ 矩阵乘法的输入和输出形状为: $$ [s, h] \times [h, s] \rightarrow [s, s] $$ `FLOPs`: $$ 2s^2h $$
    - **softmax 函数**：softmax 函数不会改变输入矩阵的维度，即 $$ [s, s] \rightarrow [s, s] $$，native softmax 涉及 `FLOPs` $$ 4sh $$ (或忽略)。
    - **应用注意力权重**：计算在 $V$ 上的加权 $score \cdot V$，矩阵乘法的输入输出形状: $$ [s, s] \times [s, h] \rightarrow [s, h] $$ `FLOPs`: $$ 2s^2h $$

    `attention_scale` ($1/\sqrt{k}$) 是逐元素操作、`attn_softmax` (softmax) 的计算量较小，因此都忽略不计。故 `Scale Dot Product Attention` 层内部只估算两个矩阵乘法的计算量为 $$ 4s^2h $$。

3. **多头拼接和线性映射**：所有注意力头输出拼接后通过线性映射，`concat` 不涉及数学运算，只涉及内存操作。矩阵乘法的输入和输出形状为: $$ [s, h] \times [h, h] \rightarrow [s, h] $$，**attention 后的线性映射的 `FLOPs`: $$ 2sh^2 $$**。

**综上，prefill 阶段 `MHA` 块的 `FLOPs`: $$ 6sh^2 + 4s^2h + 2sh^2 = 8sh^2 + 4s^2h $$**

#### 0.1.2 decode 阶段

1. **计算 Q、K、V**：每个 token 的 embedding 向量 $t_e \in R^{1 \times h}$，对应的，3 个矩阵乘法的输入和输出形状为: $$ [1, h] \times [h, h] \rightarrow [1, h] $$ `FLOPs`: $$ 3 \times 2h^2 = 6h^2 $$

2. **Self-Attention 层**：

    - $QK^T$：矩阵乘法的输入输出形状为: $$ [1, h] \times [h, s+o] \rightarrow [1, s+o] $$ `FLOPs`: $$ 2h(s+o) $$
    - $score \cdot V$: 矩阵乘法的输入输出形状为: $$ [1, s+o] \times [s+o, h] \rightarrow [1, h] $$ `FLOPs`: $$ 2h(s+o) $$

    通过上述两个公式，可以看出随着输出 `token` 的增加，计算量也随之线性增加，这也是我们在 llm 推理时观察到的越到后生成 `token` 越慢的原因。

    > 在实际代码中，对于每一轮解码的 flops 上述公式有时等效于: $2sh$？

3. 输出线性映射层: 矩阵乘法 `matmul` 的输入输出形状为: $$ [1, h] \times [h, h] \rightarrow [1, h] $$ `FLOPs`: $$ 2h^2 $$

**综上，decode 阶段 `MHA` 层每一轮解码的 `FLOPs`: $$ 6h^2 + 4(s+o)h + 2h^2 = 8h^2 + 4(s+o)h $$**。

#### 0.1.3 kv cache 节省了多少计算量

这里，我简单分析，对于上下文长度 $s$，不使用 kv cache 的 self-attention 的总计算量复杂度为：总计算量：$$ O(s^3h) $$，使用后的总计算量近似为 $$ O(s^2h) $$。计算量节省比率：

$$ \text{节省比率} = \frac{O(s^3h) - O(s^2h)}{O(s^3h)} = 1 - \frac{1}{s} $$

当 $s$ 较大时，$\frac{1}{s}$ 接近于 0，节省比率接近于 100%！

换种说法，计算复杂度从 $O(s^3h)$ 降低到 $O(s^2h)$，**即使用 kv cache 可节省约 $s$ 倍的计算量，输出 tokens 数越多，计算量节省越可观**。

### 0.2 MLP 层计算量

#### prefill 阶段

先分析 `prefill` 阶段 `Feed-forward`（MLP/FFN）层的计算量分析。包含两个线性层，以及一个 `relu` 激活层（逐元素操作，flops 很小 $\approx 5 \cdot 4h$，可忽略）。`MLP` 两个线性层的权重参数矩阵: $W_1 \in R^{h \times 4h}$、 $W_2 \in R^{4h \times h}$，`MLP` 的输入矩阵: $\in R^{s \times h}$。

1. 第一个线性层，线性层对应矩阵乘法的输入和输出形状为 $$ [s, h] \times [h, 4h] \rightarrow [s, 4h] $$ `FLOPs` 为 $$ 8sh^2 $$
2. 第二个线性层，矩阵乘法的输入和输出形状为 $$ [s, 4h] \times [4h, h] \rightarrow [s, h] $$ `FLOPs` 为 $$ 8sh^2 $$

**因此，`prefill` 阶段 `MLP` 层的 `FLOPs`: $$ 2 \times 8sh^2 = 16sh^2 $$**。

#### decode 阶段

除了 `MHA` 层的 `FLOPs` 计算公式需要明显区分 `prefill` 和 `decode` 阶段，其他层只需要将 `prefill` 阶段的计算公式中的 $s$ 设置为 $1$。即对于 `decode` 阶段的 `MLP` 层的 `FLOPs` = $16h^2$

### 0.3 模型总计算量

除了 `MHA`、`MLP` 块的计算量之外：

- `Embedding` 层只是一个查找表，没有进行显式的乘法运算，因此严格来说，Embedding 层本身不会产生 `FLOPs`，但可以通过其输出维度来推导其他层的 `FLOPs`。
- `LayerNorm` 操作是**逐元素**进行的，因此不存在通用的公式来。`LayerNorm` 层的两个权重都是一个长度为 $h$ 的向量，`FLOPs` 可以预估为: $2h$，但**通常忽略不计**。
- 最后的输出层（线性层）的**将隐藏向量映射为词表大小，得到每个 token 对应的 logits 向量**。线性层的权重矩阵为：$W_{last} \in R^{h \times V}$，矩阵乘法的输入和输出形状为: $$ [s, h] \times [h, V] \rightarrow [s, V] $$。`FLOPs`: $$ 2shV $$。

	综上分析可知， $n$ 层 `decoder block/layer` 的总计算量`大约为: $$ n(8sh^2 + 4s^2h + 16sh^2) = 24nh^2s + 4nhs^2 $$。而在输入数据形状为 $[b, s]$ 的情况下，一次训练/推理：

1. `prefill` 阶段总的计算量：

$$ b \times (24nh^2s + 4nhs^2) + 2bshV = 24nh^2bs + 4nhbs^2 + 2bshV $$

2. `decode` 阶段每轮的计算量：

$$ b \times (8nh^2 + 4nh(s+o) + 16nh^2) + 2bhV = 24nh^2b + 4nhb(s+o) + 2bhV $$

> 关于 llm flops 的估算，其实还有一个很简单的方法，就是**直接估算每个 token 的 flops 且只分析 qkv和输出层的矩阵计算，以及 mlp 层的矩阵计算**，这种分析过程更简单，可以直接得到每个 token 的对应的计算量为 $$ 8nh^2 + 16nh^2 = 24nh^2 $$。

#### 0.3.1 计算量定性和定量结论

**当隐藏维度 $h$ 比较大，且远大于序列长度 $s$ 时，则可以忽略一次项**：

1. `prefill` 阶段的计算量 `FLOPs` 可以近似为 $24nh^2bs$。
2. `decode` 阶段每轮 `forward` 的计算量为 $24nh^2b$，模型参数量为 $12nh^2$；
    
    > 每个 token 对应的计算量为 $24nh^2$。
    

因为，输入的 `tokens` 总数为 $bs$（即上下文总长度），即对于一个 `token` 存在等式: $$ \frac{24nh^2}{12nh^2} = 2 $$。所以，我们可以近似认为：**在一次前向传播中，对于每个 `token` 和 每个模型参数，需要进行 2 次浮点数运算，即一次乘法法运算和一次加法运算**。

> 实际会有不到 `2%` 的误差，主要是因为我们忽略了一些小算子的计算量。

一次迭代训练包含了前向传递和后向传递，后向传递的计算量是前向传递的 `2` 倍。因此，前向传递 + 后向传递的系数 $= 1 + 2 = 3$ 。**即一次迭代训练中，对于每个 token 和 每个模型参数，需要进行 6 次浮点数运算**。

有了上述训练和推理过程中计算量与参数量关系的结论。接下来，我们就可以估计一次迭代训练 `GPT3-13B` 所需要的计算量。对于 GPT3，每个 token，每个参数进行了 6 次浮点数运算，再乘以参数量和总 `tokens` 数就得到了总的计算量。GPT3 的模型参数量为 12850M，训练数据量 300B tokens。

$$ 6 \times 12850 \times 10^6 \times 300 \times 10^9 = 2.313 \times 10^{22} $$

计算结果和下表所示结果相符合。

> 估算训练一个 transformer 模型所需的算力成本的公式可参考文章[Transformer 估算 101](https://mp.weixin.qq.com/s/MFgTUDAOODgMDb59eZC9Cw)。本章主要参考 [Transformer Inference Arithmetic](https://kipp.ly/blog/transformer-inference-arithmetic/) 以及 [分析transformer模型的参数量、计算量、中间激活、KV cache](https://zhuanlan.zhihu.com/p/624740065)。

这个表总结了常见大型语言模型（LLM）的**参数数量、序列长度、批次大小、隐藏层大小、层数和每次前向推理的浮点操作数总量（FLOPs）**，`FLOPs` 以 T（万亿）为单位。

| Model           | Parameters | Sequence Length | Batch Size | Hidden Size | Number of Layers | FLOPs (prefill)    |
| --------------- | ---------- | --------------- | ---------- | ----------- | ---------------- | ------------------ |
| GPT-3 (175B)    | 175B       | 2048            | 8          | 12288       | 96               | ~7.0 × 10³ T FLOPs |
| GPT-3 (13B)     | 13B        | 2048            | 8          | 4096        | 40               | ~4.4 × 10² T FLOPs |
| BERT-Large      | 345M       | 512             | 8          | 1024        | 24               | ~2.4 × 10¹ T FLOPs |
| T5-11B          | 11B        | 512             | 8          | 1024        | 24               | ~1.4 × 10² T FLOPs |
| LLaMA-13B       | 13B        | 2048            | 8          | 5120        | 40               | ~4.4 × 10² T FLOPs |
| PaLM-540B       | 540B       | 2048            | 8          | 16384       | 96               | ~6.7 × 10⁴ T FLOPs |
| ChatGPT (GPT-4) | 175B       | 2048            | 8          | 12288       | 96               | ~7.0 × 10³ T FLOPs |


## 1. Qwen3-8B

基于提供的 `config.json` 配置，我们逐一计算 Qwen3-8B 的推理计算量。

**关键参数提取：**
- **$h$ (hidden_size)**: 4096
- **$n$ (num_hidden_layers)**: 36
- **$V$ (vocab_size)**: 151936
- **$n_{heads}$**: 32
- **$n_{kv\_heads}$**: 8 (GQA, Grouped Query Attention)
- **$h_{inter}$ (intermediate_size)**: 12288 (SwiGLU)

### 1.1 单个 Token 的计算量分析

我们采用文中 **0.3.1** 提到的方法，将计算量分为 **Dense 部分**（与上下文长度无关，如线性层、MLP）和 **Sparse 部分**（与上下文长度 $s$ 成正比，如 Attention Score）。

#### 1.1.1 Dense 部分 (矩阵投影与 MLP)

1.  **Attention 投影层 (GQA)**:
    - Query ($W_Q$): $[h, h]$ $\rightarrow$ $2h^2$ FLOPs
    - Key ($W_K$): $[h, h/4]$ $\rightarrow$ $2h(h/4) = 0.5h^2$ FLOPs (KV heads 是 Q heads 的 1/4)
    - Value ($W_V$): $[h, h/4]$ $\rightarrow$ $2h(h/4) = 0.5h^2$ FLOPs
    - Output ($W_O$): $[h, h]$ $\rightarrow$ $2h^2$ FLOPs
    - **单层 Attn 投影总和**: $5h^2$

2.  **MLP 层 (SwiGLU)**:
    - 包含 3 个线性层 ($W_{gate}, W_{up}, W_{down}$)，维度均为 $[h, h_{inter}]$ 或 $[h_{inter}, h]$。
    - $h_{inter} = 12288 = 3h$。
    - **单层 MLP 总和**: $3 \times 2h(3h) = 18h^2$

3.  **Logits 输出层**:
    - $2hV$

**全模型 Dense FLOPs (per token):**
$$ FLOPs_{dense} = n \times (5h^2 + 18h^2) + 2hV = 23nh^2 + 2hV $$

代入数值：
- $23nh^2 = 23 \times 36 \times 4096^2 \approx 13.89 \times 10^9$
- $2hV = 2 \times 4096 \times 151936 \approx 1.24 \times 10^9$
- **Total Dense $\approx 15.13$ GFLOPs**

#### 1.1.2 Sparse 部分 (Attention 运算)

Attention 机制中的 $QK^T$ 和 $Score \cdot V$ 运算。
- **Decode 阶段 (第 $s$ 个 token)**: $QK^T$ 计算量 $2sh$，Attention $2sh$。总计 $4sh$。
- **Encode 阶段 (长度 $s$)**: 平均每个 token 的计算量也是 $4sh$。

**全模型 Sparse FLOPs (per token at length $s$):**
$$ FLOPs_{sparse}(s) = n \times 4sh = 144 \times 4096 \times s \approx 0.59s \text{ MFLOPs} $$

### 1.2 估算方法验证

文中给出了两种快速估算方法，我们来验证其在 Qwen3-8B 上的准确性。

| 估算方法      | 公式           | 计算结果                                                                             | 与精确值 (15.13G) 误差 | 备注                         |
| :-------- | :----------- | :------------------------------------------------------------------------------- | :--------------- | :------------------------- |
| **通用公式法** | $24nh^2$     | $24 \times 36 \times 4096^2 \approx \mathbf{14.50 \text{ G}}$                    | $-4.2\%$         | 忽略了 Logits 和 SwiGLU/GQA 差异 |
| **参数量法**  | $2 \times P$ | 参数量 $P \approx 7.6\text{B}$ <br> $2 \times 7.6 \approx \mathbf{15.20 \text{ G}}$ | $+0.5\%$         | **最准确**                    |

> 注：参数量 $P$ 估算：$P \approx n(11.5h^2) + hV \approx 6.95\text{B} + 0.62\text{B} = 7.57\text{B}$。

### 1.3 结论

对于 Qwen3-8B：
1.  **Dense 计算量**占主导：约 **15.1 GFLOPs / token**。
2.  **KV Cache 的影响**：随着上下文长度 $s$ 增加，Attention 计算量线性增加。
    - 当 $s = 4096$ (4k) 时，Attn FLOPs $\approx 2.4 \text{ G}$，总 FLOPs $\approx 17.5 \text{ G}$。
    - 当 $s = 32768$ (32k) 时，Attn FLOPs $\approx 19.3 \text{ G}$，此时 Attention 计算量超过 Dense 部分，推理速度将显著下降。



##   附录一

在推理的 Prefill 阶段，模型需要处理输入的 Prompt。假设输入 Prompt 的总长度为 $L_{total}$，其中 $L_{cached}$ 为已命中的前缀缓存长度（无需重新计算 KV），$L_{new}$ 为需要新计算的 Token 长度。即：
$$ L_{total} = L_{cached} + L_{new} $$

### 1.1 MLA (Multi-Head Latent Attention) 模块

MLA 通过低秩压缩大大减少了 KV Cache 的显存占用，但在计算量上主要包含投影（Projection）、注意力计算（Attention）和输出映射（Output）三个部分。

**参数定义：**
- $B$: Batch Size
- $D_{model}$: 模型隐藏层维度 (h_dim)
- $D_{Q\_rank}, D_{KV\_rank}$: Q 和 KV 的 LoRA 压缩秩
- $N_{heads}$: 注意力头数
- $D_{head}$: 每个头的维度 (包含 RoPE 部分)
- $D_{v\_head}$: V 的头维度

#### A. 线性投影 (Linear Projections)
仅需对 **新输入的 Token ($L_{new}$)** 进行 Q、K、V 的投影计算。缓存的 Token 对应的 KV 向量已经存储在 Cache 中。

$$
\begin{aligned}
\text{FLOPs}_{Q\_proj} &= 2 \cdot B \cdot L_{new} \cdot D_{model} \cdot D_{Q\_rank} + 2 \cdot B \cdot L_{new} \cdot D_{Q\_rank} \cdot N_{heads} \cdot D_{head} \\
\text{FLOPs}_{KV\_proj} &= 2 \cdot B \cdot L_{new} \cdot D_{model} \cdot D_{KV\_rank} + 2 \cdot B \cdot L_{new} \cdot D_{KV\_rank} \cdot N_{heads} \cdot D_{head}
\end{aligned}
$$

> **注意**：KV 投影仅与 $L_{new}$ 有关，这是 KV Cache 节省计算量的第一个来源。

#### B. 注意力计算 (Attention Computation)
Q 向量（来自 $L_{new}$）需要与所有的 K 向量（来自 $L_{cached} + L_{new}$）进行交互。由于采用 Causal Masking，新 Token 仅关注其之前的位置。

1.  **Score 计算 ($Q \times K^T$)**:
    *   **New to Cached (矩形区域)**: 新 Token 关注所有已缓存 Token。
        $$ \text{FLOPs}_{part1} = 2 \cdot B \cdot N_{heads} \cdot L_{new} \cdot L_{cached} \cdot D_{head} $$
    *   **New to New (梯形/三角形区域)**: 新 Token 内部的自注意力（平均关注长度为 $L_{new}/2$）。
        $$ \text{FLOPs}_{part2} \approx 2 \cdot B \cdot N_{heads} \cdot L_{new} \cdot \frac{L_{new}}{2} \cdot D_{head} = B \cdot N_{heads} \cdot L_{new}^2 \cdot D_{head} $$
    *   **总 Score 计算**:
        $$ \text{FLOPs}_{score} = 2 \cdot B \cdot N_{heads} \cdot L_{new} \cdot (L_{cached} + \frac{1}{2}L_{new}) \cdot D_{head} $$

2.  **Aggregation 计算 ($Score \times V$)**:
    同理，聚合计算也遵循相同的稀疏性。
    $$ \text{FLOPs}_{agg} = 2 \cdot B \cdot N_{heads} \cdot L_{new} \cdot (L_{cached} + \frac{1}{2}L_{new}) \cdot D_{v\_head} $$

> **分析**：KV Cache 带来的计算量节省主要体现在：
> 1.  **线性层 (Projection)**: 完全省去了 $L_{cached}$ 部分的 $Q, K, V$ 投影。
> 2.  **Attention**: 省去了 $L_{cached}$ 内部的自注意力计算（即 $L_{cached} \times L_{cached}$ 区域）。剩余的计算量主要来自新 Token 对历史 Token 的关注（$L_{new} \times L_{cached}$）。

#### C. 输出映射 (Output Projection)
仅针对新 Token 的输出进行映射。
$$ \text{FLOPs}_{out} = 2 \cdot B \cdot L_{new} \cdot N_{heads} \cdot D_{v\_head} \cdot D_{model} $$

---

### 1.2 MoE (Mixture of Experts) FFN 模块

MoE 层通过稀疏激活减少了单次前向传播的参数量。

**参数定义：**
- $N_{shared}$: 共享专家数量
- $N_{routed}$: 路由专家总数
- $N_{active}$: 激活的路由专家数量
- $D_{inter}$: FFN 中间层维度

#### A. 路由计算 (Gating)
对新 Token 计算路由权重。
$$ \text{FLOPs}_{gate} = 2 \cdot B \cdot L_{new} \cdot D_{model} \cdot N_{routed} $$

#### B. 专家计算 (Expert Computation)
每个 FFN 包含 3 个线性层（Gate, Up, Down），计算量约为 $3 \times 2 \times D_{model} \times D_{inter}$。仅计算激活的专家。

$$ \text{FLOPs}_{FFN} = 2 \cdot B \cdot L_{new} \cdot (N_{shared} + N_{active}) \cdot 3 \cdot D_{model} \cdot D_{inter} $$

---

###  2. KV Cache 带来的效率提升分析

我们将总计算量简化对比：

1.  **无 KV Cache (0% Match)**:
    - 需计算全量 Prompt ($L_{total}$) 的所有投影和 FFN。
    - Attention 复杂度为 $O(L_{total}^2)$。
    
2.  **使用 KV Cache (例如 50% Match)**:
    - 仅计算后半部分 ($L_{new}$) 的投影和 FFN。**节省了 $L_{cached}$ 部分的线性层计算**。
    - Attention 复杂度降为 $O(L_{new} \cdot L_{total})$。虽然仍与总长度相关，但系数减小。

### 总结公式
$$ \text{GFLOPs}_{saving} \approx \underbrace{C_{linear} \cdot L_{cached}}_{\text{投影与FFN节省}} + \underbrace{C_{attn} \cdot L_{cached}^2}_{\text{Attention部分节省}} $$
*(注：Attention 部分的节省来自于不再计算 $L_{cached}$ 之间的相互注意力)*

---

## 附录二：计算量折线图绘制代码

以下 Python 代码用于绘制以 **Prefix Cache Length** 作为横坐标，以计算量 Gflops 作为纵坐标的，在不同固定序列长度下的计算量对比图（如 Prefill 1k, 4k, 8k, 16k Sequence）。

```python
import matplotlib.pyplot as plt
import numpy as np

# DeepSeek-V2 Lite/Standard 近似配置
CONFIG = {
    "h_dim": 5120,                # d_model
    "heads": 128,                 # n_heads
    "qk_head_dim": 128,           # head_dim (including RoPE)
    "v_head_dim": 128,            # v_head_dim
    "q_lora_rank": 1536,
    "kv_lora_rank": 512,
    "moe_inter_dim": 2048,
    "n_shared_experts": 2,
    "n_active_experts": 6,
    "n_routed_experts": 160,
    "num_layers": 60
}

def calculate_flops(total_len, match_ratio, config):
    """
    计算单次推理请求的 GFLOPs
    total_len: 总 Context Length (Prompt 长度)
    match_ratio: 前缀匹配比例 (0.0 - 1.0)
    """
    bs = 1
    # 划分 Cache 和 New
    cache_len = int(total_len * match_ratio)
    new_len = total_len - cache_len
    
    if new_len == 0: return 0 # 极端情况
    
    # 提取配置
    h = config["h_dim"]
    n_h = config["heads"]
    d_h = config["qk_head_dim"]
    d_v = config["v_head_dim"]
    r_q = config["q_lora_rank"]
    r_kv = config["kv_lora_rank"]
    
    # 考虑 Causal Masking:
    # Part A: New queries attend to Cached keys (Rectangular: new_len * cache_len)
    # Part B: New queries attend to New keys (Triangular: new_len * new_len / 2)
    effective_attn_len = cache_len + new_len / 2
    
    # --- MLA 模块 FLOPs ---
    # 1. Projections (仅针对 new_len)
    # Q: Down + Up
    q_proj = 2 * bs * new_len * h * r_q + 2 * bs * new_len * r_q * n_h * d_h
    # KV: Down + Up (Up absorbed but conceptually calculated for FLOPs)
    # 注意：KV Projection 仅需计算新 Token
    kv_proj = 2 * bs * new_len * h * r_kv + 2 * bs * new_len * r_kv * n_h * d_h
    
    # 2. Attention (Q=new_len, K/V=total_len)
    # Score: Q * K^T
    attn_score = 2 * bs * n_h * new_len * effective_attn_len * d_h
    # Agg: Score * V
    attn_agg = 2 * bs * n_h * new_len * effective_attn_len * d_v
    
    # 3. Output Projection
    out_proj = 2 * bs * new_len * n_h * d_v * h
    
    mla_flops = q_proj + kv_proj + attn_score + attn_agg + out_proj
    
    # --- MoE 模块 FLOPs ---
    # 1. Gating
    n_routed = config["n_routed_experts"]
    gate_flops = 2 * bs * new_len * h * n_routed
    
    # 2. Experts (Shared + Active)
    n_shared = config["n_shared_experts"]
    n_active = config["n_active_experts"]
    d_inter = config["moe_inter_dim"]
    # FFN: Gate(h->inter) + Up(h->inter) + Down(inter->h) = 3 matrix muls
    expert_flops = 2 * bs * new_len * (n_shared + n_active) * 3 * h * d_inter
    
    moe_flops = gate_flops + expert_flops
    
    # 总 FLOPs (所有层)
    total_flops = config["num_layers"] * (mla_flops + moe_flops)
    
    return total_flops / 1e9  # 转换为 GFLOPs

def plot_flops_analysis():
    # 参考图片配置：固定 Total Length，观察 Cache Length 变化对 FLOPs 的影响
    target_lengths = [1024, 4096, 8192, 16384]
    
    fig, axes = plt.subplots(2, 2, figsize=(15, 10))
    axes = axes.flatten()
    
    for idx, total_len in enumerate(target_lengths):
        ax = axes[idx]
        
        # 生成 Prefix Cache Length 点 (从 0 到 total_len，步长为 total_len/20)
        cache_lens = np.linspace(0, total_len, 20).astype(int)
        
        flops_with_cache = []
        flops_without_cache = []
        
        # 计算基准值 (Without Cache) - 恒定值
        # 此时 cache_len = 0, new_len = total_len
        base_flops = calculate_flops(total_len, 0.0, CONFIG)
        
        for c_len in cache_lens:
            # 1. Without Cache: 始终重新计算所有 token
            flops_without_cache.append(base_flops)
            
            # 2. With Cache: 使用当前 c_len 作为 cache
            if total_len == 0:
                ratio = 0
            else:
                ratio = c_len / total_len
            
            # 边界保护
            if ratio > 1.0: ratio = 1.0
            
            flops = calculate_flops(total_len, ratio, CONFIG)
            flops_with_cache.append(flops)
            
        # 绘制曲线
        ax.plot(cache_lens, flops_with_cache, 'o-', label='with_cache', markersize=4)
        ax.plot(cache_lens, flops_without_cache, 's-', label='without_cache', markersize=4)
        
        ax.set_title(f"Prefill {total_len//1024}k Sequence")
        ax.set_xlabel("prefix cache length")
        ax.set_ylabel("Gflops")
        ax.grid(True, linestyle='--', alpha=0.7)
        ax.legend()

    plt.tight_layout()
    plt.savefig("kv_cache_efficiency.png")
    print("图表已生成: kv_cache_efficiency.png")

if __name__ == "__main__":
    plot_flops_analysis()
```

## 参考资料

1.  [llm 参数量-计算量-显存占用分析](https://www.armcvai.cn/2024-09-20/llm-params-flops.html) - 详细分析了 Transformer 模型的参数量与 FLOPs 计算，验证了 KV Cache 在 Prefill 和 Decode 阶段的计算特性。
2.  DeepSeek-V2/V3 Technical Reports - 关于 MLA 和 MoE 架构的参数细节。
https://mp.weixin.qq.com/s/en-68ltblTU_et1BWHDTMA
https://mp.weixin.qq.com/s/4XBFn_ChVjn_myTXVJClfw


