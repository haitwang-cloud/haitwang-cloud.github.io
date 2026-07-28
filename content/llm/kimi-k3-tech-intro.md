+++
title = 'Kimi K3 Tech Intro'
date = 2026-07-28T14:13:53+08:00
draft = false
+++

# vLLM x Kimi K3：生产级推理引擎技术实现全景

本文整理 Kimi K3 在 vLLM 中的核心工程实现，重点覆盖混合 prefix caching、KDA 内核优化、序列并行、投机解码，以及 Prefill/Decode 分离等关键能力。

## 一、Kimi K3 架构总览

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        Kimi K3 (2.8T 参数)                          │
│                                                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ KDA 层  │  │ KDA 层  │  │ KDA 层  │  │Full Attn│  │ KDA 层  │  │
│  │线性注意力│  │线性注意力│  │线性注意力│  │MLA,每4层│  │线性注意力│  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │             │             │             │             │       │
│       └─────────────┴─────────────┴──────┬──────┴─────────────┘       │
│                                          │                           │
│                              ┌───────────▼───────────┐               │
│                              │   Attention Residual   │               │
│                              │   跨深度残差混合       │               │
│                              └───────────┬───────────┘               │
│                                          │                           │
│                              ┌───────────▼───────────┐               │
│                              │   Stable LatentMoE    │               │
│                              │   896 专家 / 16 激活   │               │
│                              │   MXFP4 权重 + SiTU    │               │
│                              └───────────────────────┘               │
│                                                                     │
│  上下文窗口: 1M tokens │ 原生视觉 │ 93 层 │ 896 routed + shared experts │
└─────────────────────────────────────────────────────────────────────┘
```

### 架构组件与推理影响

| 架构组件 | 创新点 | 对推理引擎的影响 |
|---|---|---|
| KDA（Kimi Delta Attention） | 固定大小循环状态替代逐 token KV | 需要新的 cache 管理方式，并重设计 prefix caching |
| Attention Residual | 用深度方向注意力替代简单残差累加 | 增加跨层内存流量，需要融合内核 |
| Stable LatentMoE | `16/896` 稀疏激活 + 分位数均衡 | 路由、通信和 MoE 内核成为主要瓶颈 |
| MXFP4 + SiTU | 原生 4-bit 权重 + 新激活函数 | 需要专用 FP4 MoE 路径 |
| 原生视觉 | 视觉塔直接集成 | 需要多模态预处理和 ViT 并行策略 |

## 二、核心难题：混合 Prefix Caching

### 2.1 问题本质

```text
全注意力层 (MLA)                    KDA 层 (线性注意力)
┌────────────────────┐            ┌────────────────────┐
│ token 0: K₀, V₀   │            │                    │
│ token 1: K₁, V₁   │            │  一个固定大小的     │
│ token 2: K₂, V₂   │            │  循环状态矩阵 S     │
│ ...                │            │  + 卷积状态 C       │
│ token N: Kₙ, Vₙ   │            │                    │
│                    │            │  不随序列增长        │
│ 按 token 存储，     │            │  每 token 原地更新   │
│ 可精确回溯任意位置  │            │  不可回溯            │
└────────────────────┘            └────────────────────┘
     物理块 = 16 tokens               物理块 = 4096 tokens
```

旧设计的问题在于，prefix 匹配必须对齐物理块边界。假设两个请求共享前缀分别是 `4800` 和 `4900` tokens，而 KDA 的物理块大小是 `4096`，旧规则下只能复用前 `4096` tokens，剩余 `704` 到 `804` tokens 都会被白白重算。

### 2.2 解法：三概念解耦

```text
┌─────────────────────────────────────────────────────────────────┐
│                    新设计：三者独立                            │
│                                                                 │
│  ① 物理块大小 (4096 tok)                                        │
│     GPU 显存分配单元，用于存放 KDA 状态和 KV                    │
│                                                                 │
│  ② 调度器对齐                                                   │
│     执行停止点，保证 KV 组与 KDA 组的 num_computed_tokens 一致  │
│                                                                 │
│  ③ 前缀匹配单元 (256 tok)                                       │
│     哈希与匹配的细粒度间隔，使用链式哈希标识整个前缀            │
│                                                                 │
│  效果：匹配精度从 4096 tokens 提升到 256 tokens                 │
└─────────────────────────────────────────────────────────────────┘
```

这套设计把三个原本耦合的概念拆开：

1. 物理块大小只负责显存分配。
2. 调度器对齐只负责执行边界。
3. 前缀匹配单元只负责哈希与复用粒度。

### 2.3 写时复制（Copy-on-Write）

```text
请求 A (先到):
  prefill → 在 token 3000 处注册 KDA 状态快照 → 标记为 [共享/只读]
                                                        │
请求 B (后到):                                          │
  前缀匹配命中 token 3000 ──────────────────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────────┐
  │ Step 1: COPY  缓存状态到 B 的私有 GPU 内存        │
  │ Step 2: WRITE B 在私有副本上继续 decode          │
  │ Step 3: 原始缓存保持不变，后续请求仍可复用       │
  └─────────────────────────────────────────────────┘
```

写时复制保证共享前缀既能复用，又不会被后续 decode 污染。

### 2.4 五个关键实现细节

| 细节 | 问题 | 解法 |
|---|---|---|
| 调度器边界停止 | 状态必须精确对应宣告的 token 数 | 在哈希边界而不是物理块边界停止 |
| `num_computed_tokens` 统一 | KV 块是 `16`，KDA 块是 `4096`，但逻辑位置必须一致 | 调度器强制两组对齐到同一逻辑位置 |
| 链式哈希 | 只哈希尾部会导致不同前缀碰撞 | 使用 `hash_n = H(hash_{n-1}, tokens[n-1:n])` |
| 同步骤复用延迟 | 注册与扩展并发会引入竞态条件 | 推迟到下一步复用，保证复制原子完成 |
| 跨 Worker 传输 | PD 分离时逻辑前缀需要跨节点传递 | 传输逻辑标识，而不是物理地址 |

## 三、Cache 保留策略

### 3.1 按间隔保留（Interval-based）

```text
序列: [0 ──────── 32K ──────── 64K ──────── 96K ──── prompt_end]

MLA KV:  ████████████████████████████████████████████████  (每 block 都缓存)

KDA 状态: ◆                 ◆                 ◆              ◆
          │                 │                 │              │
       间隔 checkpoint   间隔 checkpoint   间隔 checkpoint  prompt 边界
```

- `VLLM_PREFIX_CACHE_RETENTION_INTERVAL=32768`：每 `32K` tokens 保留一个 checkpoint。
- `VLLM_PREFIX_CACHE_RETENTION_INTERVAL=0`：只保留 prompt 结束处的 checkpoint。

### 3.2 Marconi 式选择性保留

```text
请求 1: [系统提示][工具定义][用户问题 A]
  → 首次出现，不缓存 KDA 状态（只保留 prompt 结束处）

请求 2: [系统提示][工具定义][用户问题 B]
  → 第二次命中，说明该前缀确实会被共享
  → 在共享前缀边界缓存 KDA 状态

请求 3: [系统提示][工具定义][用户问题 C]
  → 直接复用已缓存的 KDA 状态
```

设计哲学是需求驱动：一次性前缀不占用 cache，重复出现的前缀才自动提升为缓存热点。

## 四、内核优化全景

### 4.1 KDA Decode 融合内核

旧实现中，每层需要 6 次以上内核启动，包含因果卷积、QK Norm、Gate 计算、KDA 循环更新、Gated RMSNorm 和输出写回。在 `93` 层、逐 token decode 的场景下，启动开销和中间张量都会非常大。

新实现把这些步骤融合到单个 CUDA 内核中：

```text
输入投影
  → 因果卷积
  → KDA 循环更新
  → Gate
  → RMSNorm
  → 直接写出输出
```

收益是减少 kernel launch，避免中间张量，并原地更新卷积状态和循环状态。

### 4.2 Attention Residual 融合

标准残差通常是：

```text
x_out = x_in + sublayer(x_in)
```

而 Attention Residual 改成了深度方向加权聚合：

```text
x_out = Σᵢ αᵢ · RMSNorm(block_i)
α = softmax(pseudo_query · keys)
```

融合内核会在单次 Triton 或 CUDA kernel 中完成以下操作：

1. 读取最多 8 个缓存 block 和当前残差。
2. 计算 logits。
3. 执行 online softmax。
4. 做加权聚合。
5. 完成残差更新，并按需输出 RMSNorm。

### 4.3 MXFP4 MoE + SiTU

Kimi K3 的 MoE 路径可以概括为：

```text
输入 token
  → Router (Top-16, Quantile Balancing)
  → Routed Experts + Shared Expert
  → Latent 上投影 + RMSNorm + 逐元素相加
```

其中：

- NVIDIA 路径使用 `TRTLLM-Gen / DeepGEMM`，并支持 SiTU 参数映射。
- AMD 路径使用 `FlyDSL MLIR` 内核栈，覆盖 `A16W4 / A8W4` 融合算子。

### 4.4 LatentMoE 尾部融合

| 实现方式 | 数据路径 | 结果 |
|---|---|---|
| 旧实现 | 路由专家输出和共享专家输出分别 all-reduce，再做 RMSNorm 和上投影 | 两次 all-reduce，且存在冗余上投影 |
| 新实现 | 路由专家输出走 `all-reduce + RMSNorm + 列并行上投影`，共享专家输出走 `reduce-scatter` 后直接合并 | 少一次 all-reduce，减少冗余投影，延迟下降约 `20%`，端到端提升 `7%` 到 `8%` |

### 4.5 KDA 元数据构建器优化

| 路径 | batch=1 延迟 | 说明 |
|---|---|---|
| 旧路径 | `870 us` | 沿用通用 GDN 构建器，包含大量 K3 不需要的 FLA 元数据准备和细碎 eager PyTorch 算子 |
| 新路径 | `34 us` | 改为专用 KDA 构建器，剪掉无用路径，并融合为每序列一次 Triton 内核启动 |

这一项单独就把元数据构建延迟降低了约 `96%`，并带来约 `6%` 的端到端 DSpark 延迟改善。

### 4.6 低延迟 BF16 skinnyGEMM

在小 batch decode 场景里：

- `cuBLAS` 更偏向通用大形状优化，启动 TMA 和 Tensor Core 的固定开销较高。
- `skinnyGEMM` 直接把数据装入寄存器，走 CUDA Core FMA，更适合窄矩阵。

结果是内核级加速约 `8%` 到 `100%`，端到端延迟下降约 `10%`。

## 五、序列并行（TEP Prefill）

朴素 TEP 的问题是每层需要两次 `all-reduce`，而且 Attention Residual 会对整批 token 做冗余计算。

序列并行 TEP 的流程如下：

```text
QKV Proj
  → Attention
  → O_Proj
  → reduce-scatter
  → 每个 rank 仅持有部分 token
  → AttnRes 分片计算
  → MoE all-to-all dispatch / combine
  → all-gather
  → 下一层 QKV Proj
```

此外，针对中小消息尺寸还实现了自定义 `reduce-scatter / all-gather` 内核，在这类场景下比 NCCL 快 `1.7x` 到 `4.5x`。

## 六、DSpark 投机解码

```text
DSpark 草稿模型
  → 一次生成 7 个候选 token
  → Kimi K3 目标模型并行验证
  → 接受则直接输出，拒绝则回退
```

DSpark 草稿模型包含：

- 低秩 Markov head
- Confidence head
- 原生 MLA 注意力
- 与目标模型共享的 KV 布局

性能表现：

| 配置 | 无投机解码 | 有 DSpark | 加速比 |
|---|---|---|---|
| TP16 | `118 tok/s` | `370 tok/s` | `3.14x` |

不同任务的接受率也有明显差异：

- 低熵任务（如代码）每步平均接受约 `4.73` tokens。
- 高熵任务（如创意写作）每步平均接受约 `2.61` tokens。

## 七、PD 分离（Prefill/Decode Disaggregation）

```text
┌──────────────────────┐          ┌──────────────────────┐
│ Prefill 节点         │          │ Decode 节点          │
│ TEP8                 │  NIXL    │ DEP16                │
│                      │  传输    │                      │
│ • 处理长 prompt      │ ═══════→ │ • 逐 token 生成      │
│ • 计算 KV + KDA      │          │ • 专家并行           │
│ • 序列并行           │          │ • 投机解码           │
└──────────────────────┘          └──────────────────────┘
```

NIXL Connector 需要传输两类逻辑视图：

1. token 级 MLA cache，也就是 paged KV。
2. request 级 KDA 状态，也就是循环状态和卷积状态。

在传输过程中，还需要：

- 交换 MLA 和 KDA 元数据。
- 构造对应的 transfer descriptor。
- 跟踪逻辑到物理的映射。
- 对尾部做清零，避免数据泄漏。

### 部分命中与 Offloading 协调

当本地 GPU cache 和远端 Mooncake 都存在前缀命中时，调度器不会简单优先本地，而是选择可复用 token 更多的一侧。例如：

- 本地可复用：`3200` tokens
- 远端可复用：`4500` tokens

这时系统会释放本地短尾部预留的 block，把所有 cache group 对齐到 `4500`，然后从远端拉取。

## 八、MLA 模块优化（PD 分离专用）

旧实现依赖 `torch.compile` 自动融合，问题是启动慢，而且很多内核并没有真正融合成功。

新实现改成手动融合，并区分 prefill 与 decode 两条路径：

### Prefill 路径

```text
QKV Proj → FlashInfer MLA → Gate Proj
                       → elementwise mul + sigmoid 融合进 gate epilogue
```

### Decode 路径

```text
QKV Proj → FlashInfer MLA ─┐
                           ├→ 合并输出
Gate Proj  (多流并行) ──────┘
```

这种拆分让 decode 能更充分利用多流并行，而 prefill 则更强调 epilogue 融合。

## 九、性能数据

### 单用户解码吞吐（GB300 NVL72, batch=1）

| 配置 | 无投机解码 | 有 DSpark | 加速比 |
|---|---|---|---|
| TP8 | `111 tok/s` | `331 tok/s` | `2.98x` |
| TP16 | `118 tok/s` | `370 tok/s` | `3.14x` |

### 精度验证

| Benchmark | 得分 |
|---|---|
| GSM8K | `0.976` |
| GPQA-Diamond | `0.939` |
| OCRBench | `0.889` |
| MMMU Pro Vision | `0.818` |

### Pareto 前沿

```text
吞吐 (TPGS)
  2000+ │ ●
        │   ●
        │     ●
        │       ●
        │         ●
  100+  │           ●  ← 100+ TPS/user
        └──────────────────────
        低延迟 ←──────→ 高吞吐
```

## 十、硬件与部署支持

| 平台 | 状态 | 关键组件 |
|---|---|---|
| NVIDIA Blackwell (`B300/GB300`) | Day-0 支持 | FlashKDA、融合 CUDA 内核、TRTLLM-Gen MXFP4 |
| NVIDIA Hopper (`H100/H200`) | Day-0 支持 | Flash-Flash-KDA、DeepGEMM |
| AMD `MI355X` | Day-0 初始支持 | FlyDSL MLIR 内核栈、`A16W4/A8W4` 融合算子 |

### 最低硬件要求

- 最低配置：`8 x NVIDIA B300`，或 `GB300 NVL72` 单节点。
- 替代方案：`16 x NVIDIA B200`。
- 生产部署：建议跨节点 EP + DP，并结合 RDMA 或 NVLink。

## 参考文章
- [Kimi K3 来了：vLLM 上的高效 Day-0 支持](https://mp.weixin.qq.com/s/RdDfuzjAtaiyyautADDOzg)
- [A Preview of Production-Scale Kimi K3 Support on vLLM](https://vllm.ai/blog/2026-07-22-kimi-k3-preview)