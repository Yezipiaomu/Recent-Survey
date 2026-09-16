# 工业界 OPD 配方对照：28 个生产系统怎么用

> 调研日期：2026-09-16
> 置信度：🟡 全部来自社区论文库 [awesome-on-policy-distillation](https://github.com/chrisliu298/awesome-on-policy-distillation)
> 的 "Technical Reports and Industrial Recipes" 表，**未逐个核验技术报告原文**。
> ⚠️ 2606–2608 段系统（2026 年 6–8 月）超出本报告生成模型知识截止（2026-05），风险更高。
> 一句话结论：**2026 年工业界的 OPD 主导形态已经不是原版单教师压缩，
> 而是多教师能力融合（MOPD）——"某某用了 OPD"已是一句没有信息量的话。**

---

## 1. 三级跳：2024 → 2025 → 2026

| 阶段 | 形态 | 解决什么 | 代表 |
|---|---|---|---|
| **2024** | 纯 KD 替代 next-token prediction | 小模型预训练质量 | Gemma 2（2B / 9B 学生） |
| **2025** | **off-policy 冷启 → on-policy 两段式** | 训练-推理分布不匹配 | Qwen3、Qwen3-Omni（OPD 后再接 GSPO）、HY-MT1.5 |
| **2026** | **多教师 OPD（MOPD）做能力融合** | 多域 RL 的能力打架与跷跷板 | 见 §2 |

注意 2025 年 Qwen3 的"先 off-policy 再 on-policy"**恰好就是** `Rethinking OPD` 一年后
用对照实验证明有效的救援配方（🟢，见 [01 §4.1](01-mechanism-and-failure-modes.md#41-配方一off-policy-冷启动)）。
**工业直觉先于学术解释约一年。**

---

## 2. MOPD：2026 年的主导范式

### 2.1 采用者清单

| 系统 | 厂商 | MOPD 做法 |
|---|---|---|
| **DeepSeek-V4** | DeepSeek | 域专家 SFT+GRPO → **用 OPD 做统一模型固化**（替换原混合 RL 阶段） |
| **Nemotron 3 Ultra** | NVIDIA | 迭代式 MOPD，**10+ 域专家教师**，每轮从学生刷新教师 |
| **Mach-Mind-4-Flash** | — | 路由式 reverse-KL 融合 10+ 专家，**终结 mixed-reward 的跷跷板效应** |
| **KAT-Coder-V2** | Kwaipilot | **"Specialize-then-Unify"**：5 个域专家 agent → 在学生轨迹上 OPD 统一 |
| **KAT-Coder-V2.5** | Kwaipilot | 同上 + **漂移感知截断**稳定 |
| **Baichuan-M3** | 百川 | 任务 RL → 离线策略蒸馏 → 多教师 OPD（**三段**） |
| **MiMo-V2-Flash** | 小米 | MOPD 一词的出处；域教师覆盖 Math / Code / IF / SWE / Tool Use |
| **Kwai Keye-VL-2.0** | 快手 | **跨模态**多教师 OPD，在学生多模态 rollout 上给密集 token 反馈 |
| **Agents-A1** | — | 终局多教师 OPD，路由教师通过**显著词表对齐**监督学生 rollout |
| **NebulaExp-8B** | — | 单/多教师对比消融 |
| **Nemotron-Cascade 2** | NVIDIA | Cascade RL + 多域 OPD |
| **Audex** | — | Cascade RL + 多域 OPD（Nemotron-Cascade 谱系），保住统一音文模型的文本智能 |

### 2.2 MOPD 解决的是什么问题

三个独立系统用了几乎相同的措辞描述动机，指向同一个痛点：

> **多域 RL 会打架。** 顺序优化不同目标会累积性地破坏先前获得的能力（GLM-5 的表述）；
> 混合奖励会产生"跷跷板效应"（Mach-Mind 的表述）。

MOPD 的解法是把"同时优化多个目标"换成"**先各自练到最好，再用密集 token 信号融合**"。
这比多任务 RL 的奖励配比调参稳定得多。

### 2.3 一条与学术结论的独立互证

**NebulaExp-8B 的消融结论：teacher capability outweighs scale（教师能力 > 教师规模）。**

这与 `Rethinking OPD` §3.2 "New Knowledge, Not Just Scale"（🟢）**完全吻合**，
且两者方法论完全独立（一个是学术受控实验，一个是工业系统消融）。

→ **这是目前 OPD 领域最强的共识之一**，可以放心当作选型原则。

---

## 3. 非 MOPD 的三种用法

同样叫"用了 OPD"，目标可以完全不同：

### 3.1 抗遗忘修复 — GLM-5（智谱）

> 在多阶段 RL 流水线中，顺序优化不同目标会累积性地退化先前能力，
> 因此把**在线跨阶段蒸馏（on-policy cross-stage distillation）作为最后一个阶段**来恢复早期 SFT 与 RL 阶段的技能。
> **教师是前序阶段的最终 checkpoint**，训练 prompt 从这些教师各自的 RL 训练集中按适当比例混采。

→ 教师是自家血统的早期 checkpoint，**离"自己教自己"只差一步**。
注意它同时用上了 `Rethinking OPD` 的两条配方精神：教师同族（思维模式天然兼容）+ prompt 与教师后训练数据同源。

### 3.2 定向行为修补 — Cursor Composer 2.5

> Hint-conditioned 自教师 OPD KL **加在 RL 之上**，用于定向修特定行为（工具调用、风格）；
> 建于 Kimi K2.5 之上。

→ 不是主训练手段，是**外科手术式的补丁**。属于 OPSD（特权上下文自教师）而非跨模型蒸馏。

### 3.3 崩溃后恢复 — MAI-Thinking-1（Microsoft）

> 在自己的 RL rollout 上自蒸馏，用于**崩溃后或基座策略刷新后恢复爬坡**。

→ 把 OPD 当作 RL 训练的**稳定器/急救包**，而非独立阶段。

---

## 4. 完整系统清单（按年份）

| 年份 | 系统 | OPD 用法 |
|---|---|---|
| 2024 | Gemma 2 | KD 替代 next-token prediction（2B / 9B 学生） |
| 2025 | Qwen3 | 强到弱；off-policy 后接 on-policy |
| 2025 | Qwen3-Omni | off-policy → on-policy，之后接 GSPO |
| 2025 | GLM-4.5 / 4.6 | 多阶段后训练，专家模型迭代 + RL |
| 2025 | HY-MT1.5 | 多阶段翻译：SFT + OPD + RL |
| 2026 | MiMo-V2-Flash | MOPD 作为后训练阶段 |
| 2026 | GLM-5 | 在线跨阶段蒸馏恢复早期能力 |
| 2026 | Typhoon-S | 极简主权配方：SFT + OPD + 小规模 RFT |
| 2026 | Nemotron-Cascade 2 | Cascade RL + 多域 OPD |
| 2026 | Baichuan-M3 | 任务 RL → 离线策略蒸馏 → 多教师 OPD |
| 2026 | MobileLLM-R1.5 | 终局 on-policy KD 作为相对 R1 的主要改进 |
| 2026 | Nanbeige4-3B-Thinking | 数学推理上 OPD 优于 off-policy |
| 2026 | **DeepSeek-V4** | 域专家 SFT+GRPO → OPD 统一固化 |
| 2026 | Qwen3.5-Omni | 专家蒸馏 → 特权输入自蒸馏对齐音频到文本 |
| 2026 | HY-Embodied-0.5 | 32B → 2B；学生 rollout + 教师 token 级监督 |
| 2026 | KAT-Coder-V2 | Specialize-then-Unify：5 专家 → OPD 统一 |
| 2026 | KAT-Coder-V2.5 | MOPD + 漂移感知截断 |
| 2026 | Cursor Composer 2.5 | Hint-conditioned 自教师 OPD KL 加在 RL 上 |
| 2026 | MAI-Thinking-1 | 自 RL rollout 自蒸馏，崩溃后恢复 |
| 2026 | Nemotron 3 Ultra | 迭代 MOPD，10+ 教师，每轮从学生刷新 |
| 2026 | InternVideo3 | 终局 reverse-KL，学生采样视频 rollout |
| 2026 | Kwai Keye-VL-2.0 | 跨模态多教师 OPD |
| 2026 | NebulaExp-8B | 单/多教师 OPD，教师能力 > 规模 |
| 2026 | Mach-Mind-4-Flash | 路由 reverse-KL 融合 10+ 专家 |
| 2026 | Agents-A1 | 终局多教师 OPD，显著词表对齐 |
| 2026 | Audex | Cascade RL + 多域 OPD，保文本智能 |
| 2026 | OvisOCR2 | 0.8B OCR 学生从 4B RL 教师蒸；**top-k reverse-KL** on 自身页面输出 rollout |
| 2026 | Gryphon-v2 | 生成-排序推荐，教师排序学生候选 rollout 替代级联 |

---

## 5. 可提取的五条工业模式

### 5.1 OPD 的位置几乎总是"最后一个阶段"

GLM-5、Nemotron 3 Ultra、InternVideo3、Agents-A1、MobileLLM-R1.5 都明确是 **final stage**。

→ **OPD 是收尾工具，不是主训练手段。** 前面必须有 SFT / RL 把能力先练出来。

### 5.2 教师越来越多来自"自家血统"

GLM-5 用前序 checkpoint、MAI-Thinking-1 用自己的 rollout、
Nemotron 3 Ultra **每轮从学生刷新教师**、Cursor 用 hint-conditioned 自教师。

→ 这**天然满足了 `Rethinking OPD` 的"思维模式兼容"条件**（🟢），
规避了最主要的失败模式。同族教师是工业界用脚投票的选择。

### 5.3 top-k 截断是生产标配

OvisOCR2 明确用 top-k reverse-KL，KAT-Coder-V2.5 用漂移感知截断，
`Rethinking OPD` 的消融也基于 top-k（k=16）。

→ 与 [01 §3.2](01-mechanism-and-failure-modes.md#32-因果消融overlap-区就是优化发生地k16) 一致：
**截断不是近似妥协，overlap 区本来就是全部收益所在。**

### 5.4 小模型是最大受益者

Gemma 2（2B/9B）、MobileLLM-R1.5（950M）、Nanbeige4-3B、
HY-Embodied-0.5（32B→2B）、OvisOCR2（4B→0.8B）、Typhoon-S。

→ 端侧/边缘模型是 OPD 最成熟的应用面，风险最低。

### 5.5 已外溢出文本 LLM

音频（Qwen3.5-Omni、Audex）、视频（InternVideo3）、多模态（Keye-VL-2.0）、
具身（HY-Embodied-0.5）、OCR（OvisOCR2）、推荐（Gryphon-v2）。

→ **凡是"有强教师、学生要自己跑推理"的场景，OPD 都在渗透。**

---

## 6. 选型启示

| 你的处境 | 工业界的对应做法 |
|---|---|
| 多个域专家要合并成一个模型 | **MOPD**（DeepSeek-V4、KAT-Coder 的 Specialize-then-Unify） |
| 多阶段 RL 后发现旧能力退化 | **跨阶段蒸馏**，用前序 checkpoint 当教师（GLM-5） |
| RL 训练崩了要恢复 | **自蒸馏急救**（MAI-Thinking-1） |
| 要修某个具体行为（工具调用/风格） | **hint-conditioned 自教师 KL 加在 RL 上**（Cursor Composer 2.5） |
| 要出端侧小模型 | **强到弱 OPD + top-k 截断**（OvisOCR2、MobileLLM-R1.5） |
| 多模态要保住文本智能 | **多域 OPD**（Audex、Qwen3.5-Omni） |

---

**相关文档**：[总览](opd-2026-09.md) · [01 机制与失败模式](01-mechanism-and-failure-modes.md) · [04 落地指南](04-practical-guide.md)
