# OPD 的机制与失败模式：`Rethinking On-Policy Distillation` 深读

> 调研日期：2026-09-16
> 来源：arXiv:2604.13016v1（2026-04-14，ShanghaiTech / 清华 / UIUC / 人大）· [代码](https://github.com/thunlp/OPD)
> 置信度：🟢 **本文档全部内容已用 `curl` 拉取论文全文（316KB HTML）逐段核对**，
> 除明确标注外不含二手转述。
> 一句话结论：**有效的 OPD 只发生在一个集中了 97–99% 概率质量的共享 token 集上，
> 而且这个区域会自我强化——理解了这一点，几乎所有 OPD 现象都能解释。**

---

## 0. 论文结构

作者把问题拆成三段递进，结构本身就是很好的阅读地图：

| 章节 | 问题 | 回答 |
|---|---|---|
| §3 现象学 | OPD 什么时候成、什么时候败？ | 两个支配条件 |
| §4 机制 | 为什么在 token 层面奏效？ | 高概率 token 的渐进对齐 |
| §5 配方 | 失败了怎么救？ | 两条策略 |
| §6 讨论 | 代价是什么？ | 三条硬限制 |

实验主体模型：
- **JustRL-1.5B** = 在 DeepSeek-Distill-1.5B（DS-1.5B）上做 RL
- **Skywork-OR1-Math-7B**（SW-7B）= 在 DeepSeek-Distill-7B（DS-7B）上做 RL
- 另用 Qwen3-1.7B-Base / Qwen3-4B 系列做配方验证

---

## 1. 核心失败模式

论文开篇给出的观察，也是整篇的动机：

> We observe a striking failure mode: **a stronger teacher can completely fail to improve a student,
> even when a weaker teacher succeeds from lower initial alignment.**

更强的教师完全无法提升学生，而更弱的教师却能成功——**这直接否定了"选最强教师"这个默认直觉**。

---

## 2. 两个支配条件（§3 现象学）

### 2.1 条件一：思维模式一致性

师生需共享兼容的思维模式，可操作的度量是**两者 top-k 分布的重叠率**。

这个条件把"教师好不好"从一个绝对属性变成了**相对于特定学生的属性**——
同一个教师对 A 学生是好教师，对 B 学生可能完全无效。

### 2.2 条件二：新知识，而非规模

> even with consistent thinking patterns and higher scores,
> **the teacher must offer genuinely new capabilities beyond what the student has seen during training.**

分数更高 **不等于** 有新东西可教。

### 2.3 反向蒸馏验证（§3.3）

作者用 weak-to-strong reverse distillation 验证上述结论，原文表述：

> showing that **same-family 1.5B and 7B teachers are distributionally indistinguishable
> from the student's perspective.**

同族不同尺寸的教师，**从学生视角看分布上不可区分**。

> ⚠️ **勘误说明**
> 此前流传的一个说法是"把 RL 后模型向其 pre-RL checkpoint 蒸馏会*精确退回* pre-RL 性能"。
> **该表述在论文原文中不存在**，系二手摘要的强化转述。
> 原文实际主张是上述"分布不可区分性"，支持的是"**规模不等于新知识**"，
> 而非"OPD 会精确回滚性能"。方向一致，强度弱得多。引用时请以原文为准。

---

## 3. 机制：共享高概率区（§4）

这是全文最硬的部分。

### 3.1 现象：渐进对齐与 97–99%

论文摘要的原始表述：

> successful OPD is characterized by **progressive alignment on high-probability tokens
> at student-visited states**, a small shared token set that concentrates
> most of the probability mass (**97%–99%**).

两个要点常被混着读，但必须分开：
1. **位置**：对齐发生在*学生实际访问的状态*上（这是 on-policy 的意义）
2. **范围**：只在*师生共享的高概率 token* 上（这是"小集合、大质量"）

### 3.2 因果消融：overlap 区就是优化发生地（k=16）

作者把 top-k 支撑集拆成交集与对称差，分别单独训练：

| 变体 | 优化支撑集 | 结果 |
|---|---|---|
| **Student Top-k** | 学生完整 top-k 支撑集 `S_t(p)` | 基线 |
| **Overlap Top-k** | 师生 top-k **交集** `S_t(p) ∩ S_t(q)` | **三个 benchmark 上几乎完全恢复基线收益** |
| **Non-Overlap Top-k** | **对称差** `S_t(p) △ S_t(q)` | 始终明显更弱 |

原文结论：

> optimizing only the overlap region is **sufficient to recover nearly the full benefit**
> of standard Student Top-k OPD.

为什么 Student Top-k 和 Overlap Top-k 几乎一样？因为**学生独有支撑集里的那些额外 token 携带的概率质量极少**。
两者的 overlap-token advantage 曲线几乎不可区分；Non-Overlap 的幅值则小得多，
意味着它在 overlap token 上的有效梯度弱得多。

### 3.3 自强化动态（最优雅的一条发现）

| 变体 | overlap ratio 轨迹 |
|---|---|
| Student Top-k | ~72% → **91%+**，稳步 |
| Overlap Top-k | ~72% → **91%+**，稳步 |
| Non-Overlap Top-k | 先下降，后仅部分恢复 |

机制原文表述：

> once a token enters the shared high-probability region and is favored by the teacher,
> reverse-KL updates concentrate more mass on it, gradually pushing competing non-overlap tokens
> out of the student's top-k set. The overlap region thus grows
> **not despite but because of the optimization**, creating a **virtuous cycle**.

→ **OPD 成功时是一个正反馈循环；失败时 overlap 停滞，训练原地踏步。**
这为"OPD 要么很顺要么完全不动"的普遍体感提供了机制解释。

### 3.4 对方法论文的推论

既然增益来自共享高概率区，那么整个 token 筛选流派（熵 / 散度 / 可学性 / 软加权）
本质上都在**用不同代理量逼近同一个 overlap 区**。

这重新表述了该流派的价值主张：不是"发现了新的有效信号"，而是"更便宜地找到那个已知有效的区域"。
而 §6.3 的结论对此并不乐观——见第 5 节。

---

## 4. 两条救援配方（§5）

论文的可操作产出。前提逻辑很清楚：

> **拥有新知识是教师的内在属性**（改不了），
> **但师生思维模式的差距可以通过训练设计缩小**（这是可干预的）。

### 4.1 配方一：Off-policy 冷启动

**设置**
| 项 | 值 |
|---|---|
| 学生 | Qwen3-1.7B-Base |
| 教师 | Qwen3-4B（Non-thinking） |
| SFT prompt 源 | OpenThoughts3-1.2M 数学子集 |
| 冷启数据 | 教师生成 **200K** responses → 对学生做 SFT，得 Qwen3-1.7B-SFT |
| 后续 OPD | 去重后剩余 **~30K** prompts |
| 对照组 | 直接从 Qwen3-1.7B-Base 起跑纯 OPD，同教师同 prompt 集 |

**结果**

两段式显著优于纯 OPD，且关键在于：

> the performance gap **persists throughout training**, indicating that the off-policy cold start
> improves not only early optimization, but also **the final performance ceiling** of subsequent OPD.

不只是"起步快"，**天花板本身被抬高了**。

**机制侧证据**（三条一致）：
- SFT 初始化的学生**起手 overlap ratio 就高得多**，轨迹平滑稳定
- base 初始化的学生起点低，**前期剧烈震荡**后才缓慢恢复
- SFT 初始化的 **entropy gap 显著更小**，说明一开始就更贴近教师的置信度剖面

### 4.2 配方二：教师对齐的 prompt

从数据侧缩小差距。教师的策略是被它后训练时见过的 prompt 塑造的，所以用同源 prompt 监督更有效。

论文在**两个粒度**上分别做了受控实验：

**（a）prompt 模板对齐**——教师 JustRL-1.5B，学生 R1-Distill-1.5B，同一 DAPO-Math-17K 题目，只改模板：

| 模板 | 内容 |
|---|---|
| 原始 DAPO | `Solve the following math problem step by step. The last line of your response should be of the form Answer: $Answer ... Remember to put your answer on its own line after "Answer:".` |
| **教师对齐** | `{Question} Please reason step by step, and put your final answer within \boxed{}.` |

同样的题目，**只换提问格式**，教师对齐模板的准确率与 overlap 增长全程更高。

**（b）prompt 内容对齐**——教师 Qwen3-4B-Base-GRPO，学生 Qwen3-1.7B-Base，等量对比：
- DAPO-Math-17K（与教师 RL 训练集同源）
- DeepMath 子集（已对 DAPO 去重）

同源 prompt 集性能更强，**共享 token 上的质量集中度更高、熵显著更低**。

> ⚠️ **必须混入 OOD prompt**，否则熵坍缩。原文措辞：
> *"such prompts should be mixed with out-of-distribution prompts to prevent entropy collapse."*

---

## 5. 三条硬限制（§6）

论文最后一章自陈代价，是判断 OPD 适用边界的关键。

### 5.1 奖励质量随轨迹深度退化

三条子发现：

| 子标题 | 内容 |
|---|---|
| Response length exhibits a sweet spot | **响应长度存在最优区间**，不是越长越好 |
| Instability originates at later tokens | **不稳定性起源于后段 token** |
| Teacher continuation degrades with prefix depth | **教师的续写能力随 prefix 加深而退化** |

→ 这三条合起来，直指 **long-horizon / agent 场景是 OPD 当前最薄弱的地方**。
而这恰恰是 2026 年投入最大的应用方向（社区论文库的 "Agents and Tool-Use" 是最大的领域扩展章节）。

### 5.2 全局信息丰富的奖励不保证局部可利用

原文观察：**全局奖励结构在两种设置下都得到保持**，问题出在局部优化几何。
作者为此提出了一个关于局部优化几何的假设（未定论）。

→ 实践含义：**"教师的 reward 信号看起来是对的"不等于"学生能用上"**。
诊断时不能只看 reward 是否合理，要看 overlap 动态。

### 5.3 采样 token 的奖励已经足够

§6.3 标题即结论：**Sampled-Token Reward Is Already Sufficient**。

→ 对整个 token 加权流派是一盆冷水：最轻量的"只算学生实际采样那个 token"的估计器可能已经够用，
复杂方案需要自证值回成本。

### 5.4 论文自己的收尾判断

> OPD's apparent **free lunch of dense token-level reward comes at a cost**,
> raising the question of **whether OPD can scale to long-horizon distillation**.

---

## 6. 实操清单

从本文档可直接落地的六条：

1. **开跑前先测 overlap ratio**——它是成功与否的 signature，官方代码含 top-k 师生 overlap 诊断工具（已并入 verl 上游）。这可能是全文最实用的一条：**能预判 OPD 会不会白跑**
2. **别默认选最强教师**——先测与你的学生的 top-k 重叠率，以及教师是否真有新知识
3. **默认加 off-policy 冷启**——不只提速，抬天花板
4. **对齐 prompt 模板和内容**，但务必混 OOD 防熵坍缩
5. **先用最简 token 估计器**，复杂加权方案作为后续优化再考虑
6. **响应长度当超参调**——存在 sweet spot；若训练不稳，优先怀疑后段 token

---

**相关文档**：[总览](opd-2026-09.md) · [02 变体分类法](02-variant-taxonomy.md) · [04 落地指南](04-practical-guide.md)
