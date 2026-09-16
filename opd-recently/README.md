# On-Policy Distillation（OPD，在线策略蒸馏）研究进展调研

> 调研日期：2026-09-16
> 范围：2023 年 GKD/MiniLLM 起源至 2026 年 9 月中旬的方法演进、机制理解、工业落地与工程实践
> 一句话结论：**OPD 已从"RL 的廉价替代"变成 2026 年后训练的标准环节，
> 但它的主导形态已经不是原版的单教师压缩，而是多教师能力融合（MOPD）；
> 与此同时学术界刚刚搞清楚它为什么有效——以及它为什么这么脆。**

---

## 文档索引

| 文档 | 内容 | 适合谁看 |
|---|---|---|
| [opd-2026-09.md](opd-2026-09.md) | **总览**：坐标定位、五条分歧轴、共识与争议全图 | 先看这个 |
| [01-mechanism-and-failure-modes.md](01-mechanism-and-failure-modes.md) | **机制与失败模式**：97–99% 概率质量、overlap 消融、两个成功前提、三条硬限制 | 想知道 OPD 到底在干什么 |
| [02-variant-taxonomy.md](02-variant-taxonomy.md) | **变体分类法**：七类教师 / 八类目标 / 五轴分歧 / 参数几何之争 | 要读文献、选方法 |
| [03-industrial-recipes.md](03-industrial-recipes.md) | **工业界配方对照**：28 个生产系统怎么用 OPD，MOPD 如何成为主导范式 | 关心前沿实验室在做什么 |
| [04-practical-guide.md](04-practical-guide.md) | **落地指南**：两条救援配方、开跑前诊断、框架选型、决策树 | 要动手训 |

---

## 引用可靠性声明（请先读）

本报告的材料分三个置信层级，正文已就地标注：

| 标记 | 含义 | 本次覆盖 |
|---|---|---|
| 🟢 **高** | 已用 `curl` 拉取原文并逐段核对 | 仅 **Rethinking OPD（arXiv:2604.13016）全文** + Thinking Machines 博客 |
| 🟡 **中** | 来自社区论文库 `awesome-on-policy-distillation` 的条目描述（该库结构化、维护活跃，但**未逐篇核验原文**） | 579 条目中引用的约 60 条、28 个工业系统配方表 |
| 🟠 **低** | 仅来自搜索引擎返回的摘要 | 少量补充性数值，已单独标注 |

两点必须说明：

1. **知识截止**：本报告生成模型的训练数据截止 2026 年 5 月。文中 arXiv 编号 **2606 / 2607 / 2608 段（2026 年 6–8 月）的论文与系统全部超出该截止**，信息来源是上述社区论文库，**未经原文核验**。正式引用前请逐篇复核。
2. **一处已修正的错误**：初轮调研曾转述"把 RL 后模型向其 pre-RL checkpoint 蒸馏会精确退回 pre-RL 性能"——该说法**在原文中不存在**。原文 §3.3 的实际结论是同族 1.5B/7B 教师"从学生视角分布上不可区分"。方向一致但强度弱得多。详见 [01](01-mechanism-and-failure-modes.md#22-条件二新知识而非规模)。

---

## 六个核心结论

### 1. OPD 在数学上就是 RL，只是奖励换成了教师的对数似然比

在对生成序列做 stop-gradient 的标准假设下，on-policy reverse-KL 的梯度可分解为 REINFORCE 形式，
per-token advantage 为 `A_t = 1 + log π_θ − log π_T`，等价于最大化逐 token 奖励 `r_t = log π_T − log π_θ`。

→ **"密集奖励的 RL"和"在自己分布上做的蒸馏"是同一件事的两种叫法。**
经典的四象限定位（SFT = off-policy+密集，RL = on-policy+稀疏，OPD = on-policy+密集）只说对了一半，见结论 5。

### 2. 有效的 OPD 只发生在一个很小的共享 token 集上——占 top-k 的一部分，却集中 97–99% 概率质量 🟢

`Rethinking OPD` 的因果消融（k=16）：只在师生 top-k **交集**上算 loss，
就能**几乎完全恢复**标准 OPD 的收益；只在**非交集**上算则始终明显更弱。

且这个过程自强化：overlap ratio 从 ~72% 稳步升到 **91% 以上**——
原文措辞是 *"grows not despite but because of the optimization"*。

→ **所有 token 筛选流派（熵 / KL 散度 / 可学性 / 软加权）本质是在用不同代理量逼近同一个 overlap 区。**
它们的差异不在"有没有用"，而在逼近方式。

### 3. 决定成败的是"思维模式兼容"和"教师有没有新知识"，都不是教师有多强 🟢

原文识别的两个支配条件：
1. 师生需共享**兼容的思维模式**（top-k 分布重叠率高）
2. 即便分数更高、模式一致，教师也必须提供**学生训练中未见过的新能力**

由此产生反直觉失败模式：**更强的教师可能完全无效，而更弱的教师反而奏效。**

→ 工业界独立撞到同一结论：NebulaExp-8B 的多教师消融结论是 **"teacher capability outweighs scale"**（🟡）。
这是目前 OPD 领域少有的强共识。

### 4. 2026 年工业界的主导形态是多教师 OPD（MOPD），不是原版单教师压缩 🟡

28 个生产系统的演化是三级跳：
**2024 纯 KD（Gemma 2）→ 2025 off-policy 冷启 + on-policy 两段式（Qwen3）→ 2026 多教师能力融合**。

采用 MOPD 的包括 DeepSeek-V4、Nemotron 3 Ultra（10+ 教师、迭代刷新）、
Mach-Mind-4-Flash（10+ 专家、终结 mixed-reward 跷跷板）、KAT-Coder-V2.5、Baichuan-M3、MiMo-V2-Flash 等。
KAT-Coder 给这个模式起了个准确的名字：**Specialize-then-Unify**。

→ **"某某用了 OPD"已经是一句没有信息量的话**——至少要区分是做压缩、做能力融合、做抗遗忘修复，还是替代 RL。

### 5. 反直觉：密集监督 ≠ 密集更新，OPD 的参数几何更像 RL 而非 SFT 🟡

两篇独立的参数空间分析得出同一结论：OPD 的 checkpoint 权重增量是**稀疏的、off-principal 的**，
几何上更接近 RLVR，而不像 SFT 的密集改写。

→ OPD 在**监督信号形态**上像 SFT，在**参数更新几何**上像 RL。结论 1 的四象限表需要打上这个补丁。

### 6. 三条硬限制决定了 OPD 目前的天花板在哪 🟢

1. **奖励质量随轨迹深度退化**——响应长度存在 sweet spot，不稳定性**起源于后段 token**，教师续写能力随 prefix 深度下降
2. **全局信息丰富的奖励不保证局部可利用**——全局奖励结构完好，问题出在局部优化几何
3. **采样 token 的奖励已经够了**——复杂 token 加权方案未必值回成本

原文的收尾判断是：OPD 密集奖励的"免费午餐"是有代价的，**能否 scale 到 long-horizon 蒸馏是开放问题**。
这直接指向 agent 场景这个当前最热的应用方向。

---

## 领域规模参考

社区论文库 `awesome-on-policy-distillation` 截至本次调研收录 **579 条目**，
分为综述 / 核心论文（基础、桥接、稳定性、自蒸馏、上下文内化、效率系统）/ 邻接工作 /
领域扩展（agent、多模态、语音、扩散、具身、投机解码）/ 工业报告 / 框架实现。

其中**带特权信息的自教师（OPSD 系）已膨胀到与经典外部教师几乎同量级**——
这意味着 "OPD" 一词现在指两个差别很大的东西：**跨模型知识转移** vs **单模型自我改进**。
讨论时不区分必然鸡同鸭讲。

---

## 主要来源

**已核验原文 🟢**
- [Rethinking On-Policy Distillation of LLMs: Phenomenology, Mechanism, and Recipe](https://arxiv.org/abs/2604.13016)（ShanghaiTech / 清华 / UIUC / 人大，2026-04-14）· [代码](https://github.com/thunlp/OPD)
- [On-Policy Distillation — Thinking Machines Lab](https://thinkingmachines.ai/blog/on-policy-distillation/)（2025-10，领域起点）

**结构化二手源 🟡**
- [awesome-on-policy-distillation](https://github.com/chrisliu298/awesome-on-policy-distillation)（579 条目，含工业配方表与双重分类法）
- [A Survey of On-Policy Distillation for LLMs](https://arxiv.org/abs/2604.00626)（首篇 OPD 专门综述）
- [Multi-Teacher On-Policy Distillation: A New Post-Training Primitive](https://yumoxu.notion.site/multi-teacher-on-policy-distillation)

**经典前置**
- [GKD: On-Policy Distillation of LMs — Learning from Self-Generated Mistakes](https://arxiv.org/abs/2306.13649)
- [MiniLLM](https://arxiv.org/abs/2306.08543) · [DistiLLM-2](https://arxiv.org/abs/2503.07067)
