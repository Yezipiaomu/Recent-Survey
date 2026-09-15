# 稀疏注意力架构横向对比：DeepSeek / Kimi / 智谱 / MiniMax

> 调研日期：2026-09-15
> 一句话结论：**四家看似都在做「稀疏注意力」，实际分属三条互不兼容的技术路线；
> 其中智谱根本没有自研，直接用了 DeepSeek 的 DSA。**

---

## 0. 问题定义：稀疏化到底在解决什么

1M 上下文有两个独立的成本墙，容易被混为一谈：

| 成本墙 | 来源 | 影响阶段 |
|---|---|---|
| **注意力计算 O(L²)** | 每个 query 对所有历史 token 算分 | prefill 为主 |
| **KV cache 显存 O(L)** | 每层每 token 都要存 K/V | decode 为主，且决定并发上限 |

**关键**：稀疏注意力（少算）只解决第一堵墙，**不解决第二堵**——被跳过的 token 仍然要留在 cache 里备选。
真正压 KV cache 需要另一类手段（压缩、共享、线性化）。

理解四家路线差异的最快方式，就是看它们各自主攻哪堵墙：

- **DeepSeek**：两堵墙都打，且是唯一系统性打第二堵的（CSA/HCA/CED 层层压 KV）
- **MiniMax**：主打第一堵，KV 用块级选择，保持未压缩的真实 KV
- **Kimi**：用线性注意力把第二堵墙直接拆掉（固定状态，与长度无关），代价是精度
- **智谱**：不自研，复用 DeepSeek 的方案

---

## 1. 三条技术路线

```
                   ┌─ 路线 A：选择性稀疏（保留 softmax attention，只是少算）
                   │    DeepSeek DSA · MiniMax MSA · 智谱（复用 DSA）
稀疏化 ────────────┤
                   ├─ 路线 B：线性化（换掉注意力本身，改用固定状态 RNN）
                   │    Kimi KDA
                   │
                   └─ 路线 C：压缩（不改变谁跟谁算，改变存什么）
                        DeepSeek CSA / HCA / CED
```

注意 DeepSeek 同时在 A 和 C 上出手，这是它架构效率领先的根本原因。

---

## 2. 逐家拆解

### 2.1 DeepSeek — 唯一走完「选择 → 压缩 → 重构」三步的

演进路径清晰，每一代解决上一代剩下的瓶颈：

#### 第一步：DSA（DeepSeek Sparse Attention，V3.2 引入）

> 原始论文：[arXiv:2512.02556 — DeepSeek-V3.2](https://arxiv.org/abs/2512.02556)

两个组件：

1. **Lightning indexer**：用多头 ReLU 门控点积，把当前 query 对所有历史 token 打分。
   为了让打分本身不成为新瓶颈，它用了**少头数 + 低秩投影 + FP8**，
   每 FLOP 成本比主 MLA 低约**一个数量级**。
2. **细粒度 token 选择**：取 top-k（**k = 2048**），主注意力只在这个子集上跑 softmax。

复杂度：每层核心注意力 **O(L²) → O(Lk)**，k=2048 ≪ L。长序列下每 token GPU 成本最多降 **2×**。

**架构定位**：V3.2 相对 V3.1-Terminus 的**唯一**改动就是通过继续训练引入 DSA，
其余完全一致——这是一次干净的消融实验，也是 DSA 效果可信度较高的原因。

#### 第二步：CSA + HCA（V4 引入的混合注意力）

> 原始论文：[arXiv:2606.19348 — DeepSeek-V4](https://arxiv.org/abs/2606.19348)

DSA 少算了，但 KV cache 还在。V4 加了两种压缩，**分层交错使用**：

- **CSA（Compressed Sparse Attention）**：先沿**序列维度**压缩 KV cache，再在压缩后的表示上跑 DSA。
  即「压缩 + 选择」叠加。
- **HCA（Heavily Compressed Attention）**：对 KV cache 施加更激进的压缩，精度换容量。

配套还有两项：
- **mHC（Manifold-Constrained Hyper-Connections）**：增强常规残差连接
- **Muon 优化器**：加快收敛、提升训练稳定性

效果：1M token 处相比上一代 **KV cache 减少 90%、推理 FLOPs 减少 73%**。
MoE 部分仍是 DeepSeekMoE，MTP 配置与 V3 相同——**改动集中在注意力，其余不动**，这是 DeepSeek 一贯风格。

#### 第三步：CED（V4.1-Flash 的架构重构）

> 分析：[alphaXiv — DeepSeek-V4.1-Flash](https://www.alphaxiv.org/abs/2609.deepseek-v4-1-flash)

**Causal Encoder-Decoder**：40 层拆成 20 层因果编码器 + 20 层解码器。
关键在于——**解码器的全局 KV cache 不是各层自己算的，而是从编码器末层隐状态投影出来的**。

这一改动的直接后果：
- prefill 每 token 只激活 **8B** 参数，decode 激活 16B（总骨干 552B）
- 配合 **SWA Bounded Replay** 部署优化，常驻 KV cache（在 SSD 或主机内存）降到 V4-Flash 的约 **1/8**
- 配合层次化索引，**decode 计算量在上下文扩到 1M 时几乎保持恒定**

这是四家里唯一一个为了 agent 工作负载（输入重、prefill 重）**重构整个 backbone 拓扑**的。

#### 生态位

DSA 已成行业基础设施：
- [NVIDIA cuDNN 内置 DSA kernel](https://docs.nvidia.com/deeplearning/cudnn/latest/fe-oss-apis/dsa.html)（CuTe-DSL，Hopper SM90 / Blackwell SM100+；压缩 logits + Top-K 合并路径仅 SM100）
- 学术界 HISA、MISA、IndexCache、SAC 均以 DSA 为基线
- **智谱直接拿去用了**（见 2.3）

---

### 2.2 Kimi — 唯一真正换掉注意力的

> 原始论文：[arXiv:2510.26692 — Kimi Linear](https://arxiv.org/abs/2510.26692)
> 代码：[MoonshotAI/Kimi-Linear](https://github.com/MoonshotAI/Kimi-Linear)

#### KDA（Kimi Delta Attention）机制

KDA 是 **Gated DeltaNet 的细粒度门控扩展**。状态更新：

```
S_t = (I − β_t·k_t·k_tᵀ) · Diag(α_t) · S_{t−1} + β_t·k_t·v_tᵀ
o_t = S_tᵀ · q_t
```

拆成两部分看：

1. **Delta rule 项** `(I − β_t·k_t·k_tᵀ)`：这是一个类 Householder 变换，
   作用是**在写入新值之前，先擦除当前 key 上关联的旧内容**。
   这正是 DeltaNet 系列相对朴素线性注意力的核心优势——朴素线性注意力只会累加，导致记忆污染。

2. **通道级遗忘门** `Diag(α_t)`：这是 KDA 相对 Gated DeltaNet 的**唯一但关键**的改动。
   传统 Gated DeltaNet 整个 head 共用一个标量 α；
   KDA 给**每个通道单独一个 α**，于是模型可以保留某些特征维度、同时更激进地遗忘另一些。
   *（若向量门退化为标量，KDA 即还原为 Gated DeltaNet。）*

**已知局限**：真正的 delta-rule 编辑强度仍由**单个标量 β** 控制，
它同时决定「擦除多少旧内容」和「写入多少新值」——这两件事理论上应该可以分开控。

#### 硬件效率

用了 **DPLR（Diagonal-Plus-Low-Rank）转移矩阵的特化变体**：
把一串 rank-1 更新打包成紧凑形式、消除冗余计算，
比通用 DPLR 公式**快近 2×**，同时比通用形式更贴近经典 delta rule。
kernel 以 **FlashKDA** 开源。

#### 混合比例：为什么必须留全注意力

线性注意力的固定状态大小意味着**它无法做精确的全局检索**——
状态是有损压缩，找不回具体某个 token 的精确内容。

所以 Kimi Linear 用 **3:1 的 KDA : 全局 MLA 比例**；
K3 是 **69 层 KDA : 24 层 Gated MLA**（≈ 2.9:1，与论文比例一致），交错排布。
这不是妥协，是明确的设计：**用便宜的线性层承担绝大部分序列建模，用少量昂贵的全注意力层兜住精确检索。**

#### 一个反直觉的发现

论文报告：**与 KDA 层集成时，NoPE 表现优于 RoPE**——
说明位置感知可以从门控 delta 递推本身涌现出来，不必外挂位置编码。

#### 报告效果（Kimi Linear 48B/3B 规模）

| 指标 | 结果 |
|---|---|
| MMLU-Pro (4k) | 51.0，速度与全注意力相当 |
| RULER (128k) | 84.3，Pareto 最优，**3.98× 加速** |
| TPOT @ 1M | 比 MLA 快最多 **6.3×** |
| KV cache | 降低最多 **75%** |

---

### 2.3 智谱 — 不自研，复用 DeepSeek

**这是第一版调研里我搞错的地方，需要明确更正。**

事实：
- GLM-5 / GLM-5.1 采用 **DeepSeek 式 MLA + DSA** 架构，
  HuggingFace / NVIDIA NeMo AutoModel 里的架构标识符就叫 **`glm_moe_dsa`**
  （[NVIDIA NeMo 文档](https://docs.nvidia.com/nemo/automodel/latest/model-coverage/large-language-models/glm-5-moe-dsa)）
- GLM-5 技术报告（[arXiv:2602.15763](https://arxiv.org/html/2602.15763v1)）把「采用 DSA」
  列为降低训练/推理成本的架构创新——**采用，不是发明**
- GLM-5.1：744B / ~40B 激活，202K 上下文由 DSA 支撑

#### 唯一的自研改进：IndexShare（GLM-5.2）

GLM-5.2 上到 1M 上下文时，加了 **IndexShare**：

> **跨注意力头共享稀疏索引**。indexer 开销按组大小因子下降，
> 且 KV cache 条目可以加载一次、在整个 head group 内复用，直接缓解显存带宽压力。
> 声称效果：1M token 下有效计算量比「每头独立索引」**少约 2.9×**。

这个改进是合理且有价值的——DSA 原版每个头独立跑 indexer，在多头下确实浪费。
但它是**在别人的地基上加一层**，不是另起炉灶。

#### 战略解读

智谱把架构层面的投入压到最低，资源全部押在 **post-training**：
GLM-5.3 明确表示**沿用 GLM-5.2 的同一个 743B 底座，零预训练改动**，
全部提升来自「更多 RL 环境 + 更广任务覆盖 + 更多 RL 算力」。

这是一个清晰的分工判断：**底层架构是公共品（DeepSeek 已开源），差异化在后训练和垂直场景。**
考虑到智谱是四家里最早上市、财务压力最直接的，这个选择有其合理性——
但也意味着它在架构层面**没有护城河**。

---

### 2.4 MiniMax — 块级选择，押注硬件友好

> 原始论文：[arXiv:2606.13392 — MiniMax Sparse Attention](https://arxiv.org/abs/2606.13392)

#### MSA 机制

建立在 **GQA（Grouped Query Attention）**骨干上，而非 MLA。两个分支：

1. **Index Branch（选择器）**：轻量分支，对 key-value **块**打分。
   具体流程（来自 [llama.cpp 实现 PR](https://github.com/ggml-org/llama.cpp/pull/24908)）：
   - 每个 GQA group **独立**对可见因果上下文打分
   - 把分数 **max-pool 到 128-token 的 key block**
   - 选 **top-16 个 block**（query 所在的本地 block 强制包含）
2. **Main Branch**：只在选中的块上跑 softmax attention。

**KV 预算：16 × 128 = 2048 个 KV，与上下文长度无关**
→ decode 阶段的注意力成本**不再随上下文增长**。

#### Index Branch 怎么训

纯选择器，用**对 Main Branch 的 KL 对齐损失**训练，两阶段 warmup 调度，
并在 index 输入上做 **stop-gradient**，把辅助损失限制在 index 投影内（不污染主干）。

论文给了两条路径：
- **MSA-PT**：从零开始原生训练 MSA
- **MSA-CPT**：从全注意力 checkpoint 出发，替换掉 dense attention 后继续预训练

#### 与 MLA / DSA / MoBA 的区别（MiniMax 自己的论证）

- **vs MLA（DeepSeek）**：MLA 把 K/V 压到低维隐空间；
  **MSA 在标准 GQA 上操作真实、未压缩的 KV**，只是块级选择。
  MiniMax 称这解决了 M2 论文里指出的**精度损失和 prefix-caching 障碍**。
  ← 这一点对做 agent 的人很重要：**prefix cache 友好**。
- **vs DSA / MoBA**：声称 MSA 对 KV 的分块更精确，有效上下文覆盖更高。

#### 设计哲学

论文原话是遵循**奥卡姆剃刀**——大量消融后只保留必要组件，
并坚持稀疏 softmax attention 范式以**最大化复用现有软硬件基础设施**。
这与 Kimi「换掉注意力」的激进路线形成鲜明对比。

#### 报告效果

| 指标 | 结果 |
|---|---|
| 109B-MoE 规模下 | 在多数预训练与 agent 基准上**保持** GQA 全注意力基线能力 |
| 1M 上下文每 token 注意力计算 | **降 28.4×** |
| 1M prefill 延迟 | **约 9.7× 加速**（vs 全注意力 M2 架构） |
| 1M decode | **约 15.6× 加速** |

#### 工程配套

- 自研 kernel：**免 exp 的 top-k 选择**、**KV-outer 稀疏注意力**（把需要同一块的 query 批在一起）、连续内存访问保证每块只读一次
- [Together AI 报告](https://www.together.ai/blog/serving-minimax-m3-for-efficient-inference-unlocking-1m-token-context-and-multimodality-without-regrets)的服务端优化：KV-Block-Major 稀疏 kernel、MSA 的 paged attention 集成、优化的 index scoring kernel、Rust 多模态预处理网关 → 各并发档位**吞吐提升 81–125%**
- M3 结构细节：60 层（3 dense + 57 MoE）、128 专家

---

## 3. 横向对比表

| 维度 | DeepSeek | Kimi | 智谱 | MiniMax |
|---|---|---|---|---|
| **机制名** | DSA → CSA/HCA → CED | KDA | DSA（复用）+ IndexShare | MSA |
| **路线类别** | 选择 + 压缩 + 拓扑重构 | 线性化 | 选择（他人的） | 选择 |
| **注意力骨干** | MLA | MLA（24/93 层）+ KDA | MLA | **GQA** |
| **选择粒度** | **token 级** | 不适用（无选择） | token 级 | **block 级（128 token）** |
| **KV 预算** | top-k = **2048** | 固定状态（长度无关） | 同 DSA | top-16 × 128 = **2048** |
| **KV 是否压缩** | ✅ CSA/HCA/CED 层层压 | 不适用 | ❌ | ❌ 保持真实未压缩 KV |
| **prefix cache 友好** | 受压缩影响 | — | 受影响 | ✅ **明确设计目标** |
| **自研程度** | 全自研，行业基线 | 全自研 | **仅 IndexShare** | 全自研 |
| **是否训练时稀疏** | ✅ | ✅ | ✅ | ✅（**论文明确：不是推理优化，是模型语义**） |
| **kernel 生态** | NVIDIA cuDNN 官方支持 | FlashKDA 开源 | 蹭 DSA 生态 | llama.cpp PR + Together AI |
| **1M 加速（自报）** | FLOPs −73%，KV −90%（V4） | TPOT 6.3×，KV −75% | 有效计算 −2.9×（仅 IndexShare 增量） | prefill 9.7×，decode 15.6× |

---

## 4. 五个值得记住的观察

### ① 2048 这个数字不是巧合

DeepSeek DSA 的 `k = 2048`，MiniMax MSA 的 `16 blocks × 128 tokens = 2048`。
两家独立设计、不同粒度，**KV 预算收敛到同一个数量级**。

这暗示 2K 左右的有效注意力预算可能是当前模型规模下的某种经验最优——
再少精度掉，再多收益递减。值得关注后续是否有人系统性研究这个数字。

### ② 选择粒度是精度与硬件效率的直接权衡

- **token 级（DeepSeek）**：选得准，但 gather 操作内存访问离散，kernel 难写
- **block 级（MiniMax）**：连续内存访问，每块只读一次，硬件友好；
  但 128 个 token 捆绑选中/落选，粒度粗

MiniMax 声称块级划分「有效上下文覆盖更高」，DeepSeek 主张 token 级「细粒度」——
**双方的宣称直接冲突，目前没有中立的第三方对比实验**。这是最值得做的一个开放问题。

### ③ 稀疏是模型语义，不是推理开关

MSA 论文明确指出：**模型是带着稀疏注意力训练的，块选择是模型语义的一部分。**
这有两个实际后果：

- 不能事后给 dense 模型「加上」稀疏（除非走 MSA-CPT 那样的继续预训练）
- **不能在推理时关掉稀疏换精度**——关了模型就不认识自己了

部署时如果发现长上下文质量下降，不要指望「关掉稀疏」这个选项存在。

### ④ Kimi 是唯一承担范式风险的

其余三家都在「稀疏 softmax attention」范式内做优化，MSA 论文甚至明说这是为了
**最大化复用现有软硬件基础设施**。Kimi 的 KDA 换掉了注意力本身。

风险与回报都最大：
- **回报**：固定状态大小，KV cache 与长度彻底解耦，这是唯一能在 1M 以上继续线性外推的路线
- **风险**：必须靠 24 层全注意力兜底精确检索，说明线性化本身还不够；
  且 FlashKDA 之外没有厂商级 kernel 支持（对比 DSA 已进 cuDNN）

### ⑤ 智谱的选择暴露了行业分层

底层架构正在变成**公共品**：DeepSeek 开源 DSA → NVIDIA 做进 cuDNN → 智谱直接拿来用。
这条链路说明，未来可能只有少数几家真正做架构创新，其余在其上做后训练和场景。

对追踪者的意义：**看架构创新盯 DeepSeek / Kimi / MiniMax，看后训练与落地盯智谱。**

---

## 5. 开放问题（值得继续跟的）

1. **token 级 vs block 级选择，谁的有效上下文覆盖真的更高？** 目前只有两家互相宣称，无中立实验。
2. **KDA 的单标量 β 局限能否解掉？** 擦除强度和写入强度解耦，理论上应有收益。
3. **CED 架构能否推广到更大规模？** DeepSeek 称 V4.1-Flash 是「新架构族里最小的模型」，
   明确说要扩到更大——V4.1-Pro 会是这套架构的真正考验。
4. **2048 KV 预算的理论依据是什么？** 是否随模型规模/任务类型变化。
5. **智谱会不会被迫自研？** 如果 DeepSeek 停止开源（IPO 后有此压力），智谱的架构来源就断了。

---

## 6. 一手资料清单

| 论文/文档 | 内容 |
|---|---|
| [arXiv:2512.02556](https://arxiv.org/abs/2512.02556) | DeepSeek-V3.2，DSA 原始论文 |
| [arXiv:2606.19348](https://arxiv.org/abs/2606.19348) | DeepSeek-V4，CSA + HCA + mHC + Muon |
| [alphaXiv: V4.1-Flash](https://www.alphaxiv.org/abs/2609.deepseek-v4-1-flash) | CED 架构、SWA Bounded Replay |
| [arXiv:2510.26692](https://arxiv.org/abs/2510.26692) | Kimi Linear，KDA 原始论文 |
| [arXiv:2606.13392](https://arxiv.org/abs/2606.13392) | MiniMax Sparse Attention，MSA 原始论文 |
| [arXiv:2602.15763](https://arxiv.org/html/2602.15763v1) | GLM-5 技术报告 |
| [NVIDIA cuDNN DSA](https://docs.nvidia.com/deeplearning/cudnn/latest/fe-oss-apis/dsa.html) | DSA 官方 kernel |
| [NVIDIA NeMo glm_moe_dsa](https://docs.nvidia.com/nemo/automodel/latest/model-coverage/large-language-models/glm-5-moe-dsa) | 证明 GLM 复用 DSA |
| [MoonshotAI/Kimi-Linear](https://github.com/MoonshotAI/Kimi-Linear) | FlashKDA kernel |
| [llama.cpp PR #24908](https://github.com/ggml-org/llama.cpp/pull/24908) | MSA 实现细节（最好的工程视角） |
| [Doubleword: You Could Have Come Up With KDA](https://blog.doubleword.ai/you-could-have-come-up-with-kimi-delta-attention) | 从 softmax → 线性 → DeltaNet → KDA 的推导，含 Triton kernel |
| [MindStudio: GLM-5.2 架构](https://www.mindstudio.ai/blog/glm-5-2-architecture-index-share-sparse-attention) | IndexShare 细节 |
| [Together AI: 服务 M3](https://www.together.ai/blog/serving-minimax-m3-for-efficient-inference-unlocking-1m-token-context-and-multimodality-without-regrets) | MSA 服务端工程 |
