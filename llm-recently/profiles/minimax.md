# MiniMax 公司档案

> 调研日期：2026-09-15 · [返回索引](../README.md)
> 定位：**唯一同时在文本与视听两条线推进的**。视听是真正的护城河；
> 文本线的自报数据与第三方落差在四家中最大。

---

## 1. 模型谱系

### 文本线

| 时间 | 版本 | 规模 | 关键点 |
|---|---|---|---|
| 2025-10 | M2 | 230B / 10B | 开源 MoE，编码与 agent |
| 2025-12-23 | M2.1 | 230B / 10B | 扩展 Rust / Java / C++ 支持；开源 VIBE 基准 |
| **2026-02-12** | **M2.5** | 230B / 10B | **架构不变，纯靠 20 万+ 真实环境的大规模 RL**；另有 M2.5-Lightning |
| 2026-03-18 | M2.7 | — | 「首个深度参与自身演化的模型」；205K 上下文，$0.24/M 起 |
| **2026-06-01** | **M3** | **428B**（约 22B 激活） | MIT；自研 **MSA**；1M 上下文；原生多模态 |

**M2.5 值得单独看**：架构与 M2 完全相同，全部提升来自 RL——
SWE-Bench Verified 80.2%、Multi-SWE-Bench 51.3%（第一）、BrowseComp 76.3%；
比 M2.1 **快 37%**，成本约 Claude Opus 4.6 的 **1/10**。
这是「不动架构、只做 RL」路线的一个早期成功案例（智谱后来在 GLM-5.3 上走了同样的路）。

### 视听线

| 时间 | 产品 | 关键点 |
|---|---|---|
| 2026-01-23 | **Speech 2.8** | 7 种**逐句可控**情感；约 10 秒样本语音克隆 |
| **2026-07-31** | **H3 / Hailuo 3.0** | omni-modal 视频模型 |
| — | Music 3.0 | ⚠️ **2026-08-20 起不对新用户开放** |

---

## 2. MSA（MiniMax Sparse Attention）

完整拆解见 [01-sparse-attention-comparison.md §2.4](../01-sparse-attention-comparison.md)。
原始论文：[arXiv:2606.13392](https://arxiv.org/abs/2606.13392)

### 机制

建立在 **GQA 骨干**上（而非 MLA），两个分支：

1. **Index Branch（纯选择器）**：每个 GQA group **独立**对可见因果上下文打分 →
   把分数 **max-pool 到 128-token 的 key block** → 选 **top-16 个 block**
   （query 所在的本地块强制包含）
2. **Main Branch**：只在选中的块上跑 softmax attention

**KV 预算 = 16 × 128 = 2048，与上下文长度无关** → decode 注意力成本不再随上下文增长。

### Index Branch 的训练

用**对 Main Branch 的 KL 对齐损失**训练，两阶段 warmup，
并在 index 输入上做 **stop-gradient**，把辅助损失限制在 index 投影内。

两条路径：**MSA-PT**（从零原生训练）/ **MSA-CPT**（从全注意力 checkpoint 继续预训练）。

### 关键差异：保持真实未压缩的 KV

- **vs MLA**：MLA 把 K/V 压到低维隐空间；MSA 在标准 GQA 上操作**真实、未压缩的 KV**
- MiniMax 称这解决了 M2 论文指出的 **精度损失和 prefix-caching 障碍**
  → **对做 agent 的人很重要：prefix cache 友好**
- 声称对 KV 的分块比 DSA / MoBA 更精确，有效上下文覆盖更高
  （⚠️ 与 DeepSeek 的 token 级「细粒度」主张**直接冲突，无中立第三方实验**）

### 设计哲学

论文明确遵循**奥卡姆剃刀**——大量消融后只保留必要组件，
坚持稀疏 softmax 范式以**最大化复用现有软硬件基础设施**。
与 Kimi「换掉注意力」的激进路线形成对照。

### 报告效果

| 指标 | 结果 |
|---|---|
| 109B-MoE 规模 | 多数预训练与 agent 基准上**保持** GQA 全注意力基线能力 |
| 1M 每 token 注意力计算 | **−28.4×** |
| 1M prefill 延迟 | **约 9.7× 加速** |
| 1M decode | **约 15.6× 加速** |

### 工程配套

- 自研 kernel：免 exp 的 top-k 选择、**KV-outer 稀疏注意力**（把需要同一块的 query 批在一起）、
  连续内存访问保证每块只读一次
- M3 结构：60 层（3 dense + 57 MoE）、128 专家
- [llama.cpp PR #24908](https://github.com/ggml-org/llama.cpp/pull/24908) 是理解 MSA 最好的工程视角
- [Together AI](https://www.together.ai/blog/serving-minimax-m3-for-efficient-inference-unlocking-1m-token-context-and-multimodality-without-regrets) 服务端优化后各并发档位**吞吐提升 81–125%**

> ⚠️ **MSA 不是可选的推理优化**——模型是带着稀疏注意力训练的，
> 块选择是模型语义的一部分，**不能在推理时关掉换精度**。

---

## 3. H3 / Hailuo 3.0（真正的护城河）

另外三家在这条线上**没有对应产品**。

### 规格

- 文本/图像/视频/音频**统一上下文**
- **4–15 秒多镜头**视频，24 fps，六种画幅（21:9 到 9:16）
- 768p 模式可升 1440p；官方推荐 1440p 模式，21:9 下达 **2976×1248**
- **原生立体声，单次生成**，无独立音频阶段
- 支持**自然语言指令改片**，而非重新摇号

### 产品线关系（容易混淆）

**MiniMax** 是公司 · **Hailuo** 是消费级视频品牌 · **H3** 是第三代模型 · API ID 为 `MiniMax-H3`
Hailuo 2.3 / 2.3 Fast / Hailuo 02 已移入 **Legacy**。

### 立场

MiniMax 明确批评行业的**「任务孤岛」**——
把图像编辑、参考类型、语音、音效、音乐、各类视频变体拆成一堆专家模型。
其主张是：丢进一个产品图 + 一段动作片 + 一段语音，
在**同一次生成**中拿回带对白、音效和房间混响的 2K 视频。

### 开源计划

计划以 **MiniMax Community License** 放权重：年收入 <2000 万美元的组织可商用 + 署名。
（发布时称「未来几天」，当前状态需核实。）

---

## 4. ⚠️ 自报与第三方的落差（四家中最大）

| 来源 | M3 的评价 |
|---|---|
| **MiniMax 自报** | SWE-Bench Pro 59.0%、MCP Atlas 74.2%、**BrowseComp 83.5（超 Opus 4.7 的 79.3）**、OmniDocBench 超 Gemini 3.1 Pro、SVG-Bench 超 Opus 4.7、Claw-Eval 登顶 |
| **Artificial Analysis** | Intelligence Index **30**，同体量开源中位数 18 ← **这个对比是正面的**；109.3 tok/s，TTFT 1.33s |
| **Vals AI** | 综合第 6（58.94%）；**SWE-bench Verified 75.00% 仅第 17**、**Terminal-Bench 2.1 53.56% 仅第 12** |
| **BenchLM** | 232 个模型中**第 52**（61.55/100）；多模态最强（第 34）、**Agentic 最弱（第 107）** |

**核心矛盾**：MiniMax 把 M3 定位为「编码与 agent 前沿」，
但第三方在 **agent 维度给出的评价恰恰最低**。

**一个应当肯定的地方**：MiniMax **主动披露**了口径问题——
多项结果跑在自家基础设施上，常使用 Claude Code / Mini-SWE-Agent / Terminus 作为脚手架。
这比完全不说要负责任。

**注意榜单滚动**：2026 年 6 月 Vals AI 曾称 M3「开源第一」（58.94%）；
8 月后 GLM-5.3（57.0）被称为「开源第二」——**数值更低排名更高，说明索引组成变过，两数不可相减**。
同理 M3 的 AA「30 分」与 GLM-5.3/K3 的「44 分」**大概率不是同一版本索引**（v4.1 vs v4.3）。

---

## 5. 价格

| 项 | 数据 |
|---|---|
| M3 标价 | $0.60 / $2.40（每百万 token） |
| M3 实际 | **$0.30 / $1.20**（永久五折） |
| ⚠️ **断崖** | **五折仅在输入 ≤512K 时有效；>512K 恢复 $0.60/$2.40，翻倍** |
| OpenRouter | $0.23 输入 / $0.96 输出，cache read $0.05 |
| M2.7 | 205K 上下文，$0.24/M 起 |

→ **如果你就是冲着 1M 上下文去的，M3 的实际价格是 $0.60/$2.40，
此时相对 DeepSeek 的价格优势消失。**

---

## 6. 商业

| 项 | 数据 |
|---|---|
| 上市 | **2026-01 港股**，**首日翻倍** |
| 2026 上半年收入 | **1.17 亿美元**（四家中最低） |

### 蒸馏指控

Anthropic 于 2026-02 指控 DeepSeek、Moonshot、MiniMax 进行「蒸馏攻击」
（约 24000 假账号、1600 万次 Claude 对话）。
其中 **340 万次追溯到 Moonshot**；**针对 MiniMax 的具体数量未见披露**。
MiniMax 未成为白宫 7 月那轮点名的对象。

---

## 7. 值得继续跟的

1. **H3 权重是否真的按 Community License 放出**——发布时称「未来几天」，需核实
2. **Music 3.0 对新用户关闭的原因**——是版权压力、成本还是战略收缩？这是一个负面信号
3. **agent 能力的真实水平**：自报与 BenchLM 第 107 的落差需要自测来判断
4. **token 级 vs 块级选择之争**——MiniMax 与 DeepSeek 的宣称直接冲突，谁先拿出中立实验谁占优
5. **1.17 亿的收入**在四家中最低，而视听线的算力成本最高——单位经济模型是否成立

---

## 一手资料

- [arXiv:2606.13392 — MiniMax Sparse Attention（MSA 原始论文）](https://arxiv.org/abs/2606.13392)
- [MiniMax 官方 — M3](https://www.minimax.io/models/text/m3) · [M3 研究博客](https://www.minimax.io/blog/minimax-m3)
- [MiniMax 官方 — H3 / Hailuo 3.0](https://www.minimax.io/blog/minimax-h3)
- [MiniMax 官方 — M2.7](https://www.minimax.io/news/minimax-m27-en)
- [llama.cpp PR #24908 — MSA 实现细节](https://github.com/ggml-org/llama.cpp/pull/24908)
- [Together AI — 服务 M3 的推理优化](https://www.together.ai/blog/serving-minimax-m3-for-efficient-inference-unlocking-1m-token-context-and-multimodality-without-regrets)
- [VentureBeat — MSA 预告与 15.6× 长上下文提速](https://venturebeat.com/technology/minimax-teases-upcoming-m3-model-with-new-sparse-attention-mechanism-and-15-6x-response-speed-boost)
- [Artificial Analysis — MiniMax-M3](https://artificialanalysis.ai/models/minimax-m3)
- [Vals AI — MiniMax-M3](https://www.vals.ai/models/minimax_MiniMax-M3)
- [BenchLM — MiniMax M3](https://benchlm.ai/models/minimax-m3)
- [Maxime Labonne — M2.5 分析](https://medium.com/@mlabonne/minimax-m2-5-the-1-hour-frontier-model-92168de195b8)
- [HuggingFace — H3 解读](https://huggingface.co/blog/ResterChed/minimax-h3-hailuo-3-0)
