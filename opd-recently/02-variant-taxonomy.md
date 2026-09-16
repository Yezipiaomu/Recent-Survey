# OPD 变体分类法：七类教师 / 八类目标 / 三种理论视角

> 调研日期：2026-09-16
> 置信度：🟡 分类框架与条目归属来自社区论文库 [awesome-on-policy-distillation](https://github.com/chrisliu298/awesome-on-policy-distillation)（579 条目），
> **未逐篇核验原文**；2606–2608 段论文超出本报告生成模型知识截止（2026-05）。
> 一句话结论：**"教师是谁"这条轴比"loss 怎么写"重要得多——
> 它把 OPD 切成了三条几乎不交叉的文献支流。**

---

## 0. 两套正交分类

社区论文库对核心论文提供了两个交叉视图。它们正交，一篇论文通常同时落在两张表里：

- **按教师类型**（谁提供监督）→ 决定你能做什么
- **按主要目标**（为什么做）→ 决定你该读哪一支

首篇 OPD 专门综述（arXiv:2604.00626）的三轴划分与之呼应：
**反馈信号**（logit / outcome / self-play）× **教师可及性**（白盒 / 黑盒 / 无教师）× **损失范围**（token / sequence / hybrid）。

---

## 1. 按教师类型（七类）

条目规模差异极大，这个分布本身就是领域重心的指示器：

| 教师类型 | 规模 | 含义 | 代表工作 |
|---|---|---|---|
| **External white-box** | **最大宗（~95+）** | 外部更强模型，可拿 logits | GKD, MiniLLM, DistiLLM-2, Rethinking OPD, TIP, TrOPD |
| **Self-teacher with privileged context** | **第二大宗（~90+）** | 同一模型，教师侧条件于特权信息（标准答案 / 执行反馈 / 提示） | OPSD, SDPO, CANON, BIRD, CriPO |
| Context-conditioned | ~16 | 教师条件于额外上下文，把上下文信息蒸进参数 | OPCD, OEL, Sleep, Agent Experience |
| **Multiple / lifecycle teachers** | ~14 | 多教师或生命周期不同阶段的 checkpoint | MAD-OPD, MOPD, CaMOPD, Open-MOPD |
| External black-box | ~9 | 拿不到 logits，只能用 outcome 级信号 | GAD, OVD, ROPD, OmniOPD |
| Self-teacher（无特权 / 无答案） | 3 | 自教师但不给特权信息 | CAST, TS-OPSD, SafeSteer |
| Internal self-teacher（跨深度） | 1 | 模型内部不同深度互为师生 | OISD |

### 1.1 最重要的结构性事实

**第二行已经膨胀到与第一行同量级。**

这意味着 "OPD" 一词现在覆盖两个差别很大的问题：

| | 跨模型知识转移 | 单模型自我改进（OPSD） |
|---|---|---|
| 教师 | 外部更强模型 | 自己 + 特权信息 |
| 前提 | 教师有新知识 | 特权信息能诱导出更好的策略 |
| 失败模式 | 思维模式不兼容 | **collapse**（长度坍缩、熵坍缩） |
| 天花板 | 教师水平（除非外推） | 特权信息的信息量 |
| 典型场景 | 模型压缩、能力融合 | 持续学习、崩溃后恢复 |

→ **讨论"OPD 有没有用"时不区分这两支，必然鸡同鸭讲。**

### 1.2 OPSD 支流已需要独立词汇表

该子领域大到出现了专门的批判性综述：
*One Symptom, Three Levers: A Critical Review of On-Policy Self-Distillation*（arXiv:2608.25936），
其定位是"围绕 collapse 这一单一症状统一各论文的术语，而非报告新实验"。

社区讨论中的两条相关观察（🟠 来自博客/推文，未同行评议）：
- OPSD 反转了 OPD 的 per-token 符号，并承受更大的 KL 冲击
- **正向的 teacher-agreement 压力携带有用信号，负向压力驱动 length collapse**——只保留正向即退化为 OPD
- naive self-distillation 会蒸出一个"幻觉反馈模板"，这解释了为什么它不work而 OPD work

---

## 2. 按主要目标（八类）

| 目标 | 规模 | 你在解决什么 | 代表 |
|---|---|---|---|
| **稳定性 / 目标函数设计** | 最大 | OPD 训不稳 | DistiLLM-2, Veto, Uni-OPD, AOPD, vOPD, OPD+ |
| **RL 替代 / 增强** | 很大 | RL 太贵或信号太稀 | SDPO, REOPOLD, RLSD, PGPO, Constitutional OPD |
| 压缩 / 强到弱 | 大 | 大模型太贵 | MiniLLM, GKD, Lightning OPD, TrOPD, SWITCH |
| 持续学习 | 中 | 灾难性遗忘 | SDFT, SPoT, OPCD, MixSD, Sleep |
| **Post-RL 固化 / 能力整合** | 中 | 多个专家要合成一个 | MOPD, ExOPD, W2S-OPD, RoCo-ACE |
| 推理压缩 | 小 | 思维链太长 | OPSDC, MPD, TRSD, BIRD |
| 黑盒蒸馏 | 小 | 拿不到 logits | GAD, OVD, ROPD, OmniOPD |
| 安全 / 对齐 | 小 | 对齐税、安全行为迁移 | SafeSteer, Constitutional OPD, DUET, SecOPD |

**"稳定性"是条目最多的类别之一——这本身就是结论**：OPD 难训是普遍体验，不是个别人的配置问题。

---

## 3. 五条设计轴的变体地图

总览文档（[opd-2026-09.md](opd-2026-09.md#2-五条分歧轴)）已列出五轴，此处补充各轴的具体变体：

### 轴 1：Token 粒度

| 做法 | 机制 | 代表 |
|---|---|---|
| 全 token | 学生完整 top-k 支撑集 | 标准 OPD |
| 熵筛选 | 优先高不确定性位置 | Entropy-Aware OPD |
| 散度筛选 | 优先师生强分歧位置 | TIP |
| **可学性筛选** | 区分可学 / 不可学分歧 | TA-OPD |
| **软重加权** | 教师置信度 × 学生困惑度 | FiRe-OPD |
| 序列级加权 | Beta 核按 pass rate 挑中等难度 | PACED, LION |
| 最简估计器 | 只算采样 token | 多数既有实现 |

> ⚠️ 与 [01 §5.3](01-mechanism-and-failure-modes.md#53-采样-token-的奖励已经足够) 交叉阅读：
> 原论文认为采样 token 已足够，本轴的复杂方案需自证价值。

### 轴 2：教师信任机制

| 做法 | 机制 |
|---|---|
| 无条件信任 | 标准 OPD |
| **Verifier 门控（SG-OPD）** | 不默认教师 token 级偏好可靠；教师与 verifier 符号一致时外推，不一致时插值 |
| **Best-of-N rollout 选择（BRTS）** | 优先选正确的；多条正确时选**离学生最近**的；全错时注入 ground truth |
| 教师熵/置信度加权 | 按教师确定性调节信号强度 |
| 漂移感知截断 | 生产侧做法（KAT-Coder-V2.5） |

### 轴 3：稳定化手段

见[总览 §2 轴 3](opd-2026-09.md#轴-3目标函数与稳定性)。核心矛盾：
**密集 token 信号是卖点，原始 KL 的梯度爆炸是病根。**

### 轴 4：on/off 混合

| 形态 | 说明 |
|---|---|
| GKD 的 α 混合 | α ∈ [0,1] 调蒸馏 vs RL |
| **两段式** | off-policy 冷启 → OPD（🟢 有对照实验支持，见 [01 §4.1](01-mechanism-and-failure-modes.md#41-配方一off-policy-冷启动)） |
| 对比式（DistiLLM-2） | 抬教师回复似然 + 压学生回复似然，离线在线并用 |
| MPD | 学生采轨迹保分布，教师压缩成简洁推理链 |
| **异步（veRL）** | 工程侧混合：牺牲严格 on-policy 保证换吞吐 |

### 轴 5：超越教师

| 立场 | 代表 |
|---|---|
| 天花板派 | 主流隐含假设 |
| reward 外推 | ExOPD |
| 多智能体辩论 | MAD-OPD |
| 工业折中 | MOPD 称"匹配或超过对应教师" |

---

## 4. 三种理论视角（互不兼容的框架之争）

2026 年出现了三条独立的理论化路线，它们对"OPD 到底是什么"给出不同答案：

### 4.1 散度视角（传统）

OPD = 在学生分布上最小化 reverse KL。
→ 关注散度选择、mode-seeking vs mass-covering。
→ 代表：GKD 谱系、f-divergence 统一框架。

### 4.2 状态分布视角

*Post-Training is About States, Not Tokens*（arXiv:2605.22731）主张：
**按训练状态的来源而非 loss 形式重划 SFT / RL / OPD。**
→ 解释了为什么"学生采样的状态"能胜过"退化的教师"。
→ 与 [01 §3.1](01-mechanism-and-failure-modes.md#31-现象渐进对齐与-9799) 的"学生访问状态"强呼应。

### 4.3 参数几何视角（最反直觉）

两篇独立分析得出同一结论：

| 论文 | 发现 |
|---|---|
| *Dense Supervision, Sparse Updates*（arXiv:2606.13657） | OPD 的 checkpoint 增量是**稀疏、off-principal 的**，形似 RLVR 而非 SFT 的密集改写 |
| *On the Geometry of OPD*（arXiv:2606.07082） | OPD 处于 "**relaxed off-principal regime + early subspace locking**"，是介于 SFT 与 RLVR 之间的**独立更新几何** |

→ **密集监督 ≠ 密集更新。**
OPD 在监督信号形态上像 SFT，在参数更新几何上像 RL。
这给经典四象限表打了个必要的补丁。

### 4.4 对比

| 视角 | OPD 是什么 | 优化建议指向 |
|---|---|---|
| 散度 | 一个 KL 目标 | 调散度、调加权 |
| 状态分布 | 一种状态采样策略 | 调 rollout 来源与 prompt 分布 |
| 参数几何 | 一种稀疏子空间更新 | 调学习率/子空间、理解为何早期锁定 |

三者不冲突但强调点不同。**配方层面，状态分布视角目前最能指导实践**——
`Rethinking OPD` 的两条救援配方（冷启动、prompt 对齐）本质都是在调整状态分布。

---

## 5. 领域扩展（已超出纯文本 LLM）

社区论文库的领域扩展章节规模可观，说明 OPD 已经外溢：

| 领域 | 说明 | 生产案例（🟡） |
|---|---|---|
| **Agents / Tool-Use** | **最大扩展章节**；与 §1 的 long-horizon 弱点直接冲突 | Agents-A1, Coach |
| 多模态 / VLM | 跨模态多教师 | Kwai Keye-VL-2.0, InternVideo3, Vision-OPD |
| 语音 / 音频 | 音文对齐 | Qwen3.5-Omni, Audex |
| 扩散 / 流 / 生成媒体 | 沿自身 rollout 轨迹蒸馏 | DiffusionOPD |
| 具身 / 机器人 / 控制 | 32B → 2B | HY-Embodied-0.5 |
| **投机解码（draft 模型训练）** | 独立的工程分支 | SpecForge, TorchSpec |

⚠️ Agent 方向的投入强度与 [01 §5.1](01-mechanism-and-failure-modes.md#51-奖励质量随轨迹深度退化)
"奖励质量随轨迹深度退化"的发现**正面冲突**。这是当前领域最值得关注的张力。

---

## 6. 跨 tokenizer 与模型族问题

一个容易被忽略但工程上卡脖子的子问题：**教师和学生 tokenizer 不同怎么办**。

社区论文库为此单列了 "Cross-Tokenizer and Model-Family Enablers" 章节，
框架侧 NeMo-RL、KDFlow、EasyOPD 都把跨 tokenizer 作为卖点，
HuggingFace 的 GOLD walkthrough 标题直接叫 *Unlocking OPD for Any Model Family*。

→ **如果你想用 A 家的教师蒸 B 家的学生，先确认框架支持跨 tokenizer**，否则这是第一道墙。
ByteDance Seed 的实践笔记也把"跨 tokenizer 陷阱"与"agent 合并中的 reward hacking"并列为两大坑（🟠）。

---

**相关文档**：[总览](opd-2026-09.md) · [01 机制与失败模式](01-mechanism-and-failure-modes.md) · [03 工业界配方](03-industrial-recipes.md)
