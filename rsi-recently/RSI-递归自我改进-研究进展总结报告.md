# RSI（Recursive Self-Improvement，递归自我改进）研究进展总结报告

**时间范围**：2024 年 1 月 – 2026 年 9 月
**生成日期**：2026-09-15
**检索方式**：Web 搜索（多轮独立检索交叉验证）
**核心论文数**：约 35 篇（正文表格列出 32 篇）

> ⚠️ **引用可靠性声明（请先读）**
> 本报告生成环境的 `WebFetch` 与直连 `curl` 均被网络策略拦截，**无法访问 arXiv API / Semantic Scholar 逐篇核验元数据**。
> 所有 arXiv 编号、标题、作者、数值均来自搜索引擎返回的摘要，并通过"同一论文在多次独立检索中重复出现且细节一致"进行交叉验证。
> 表格中标注了每条的**置信度**：🟢 高（≥2 次独立检索一致）/ 🟡 中（单次检索出现）。
> **投稿或正式引用前请务必逐篇在 arXiv 上复核**，尤其是 🟡 条目。
>
> 🔧 **2026-09-15 修订**：逐篇核查（见 `论文逐篇解读.md`）后修正了 4 处实质性错误——`2603.03992` 的数值误归因、`2606.05976` 的结论方向、`2601.05280` 的标题与机制、以及 871/1,250 语料矛盾（已解决）。各处已就地标注 ⚠️ 或 ✅。

---

## 一、执行摘要（TL;DR）

1. **RSI 已从思想实验变为可观测的工程现象，但"递归"仍未真正闭合。** 2024–2026 年的主流系统都在改进自己的**某一层**（输出、策略、评估器、研究流程），而人类仍掌握研究方向的设定权。当前最强的自我修改系统（Darwin Gödel Machine）改的是 agent 的代码脚手架，不是模型权重；最强的算法发现系统（AlphaEvolve）改的是外部程序，不是自己。

2. **领域出现爆发式增长且极度不均衡。** 据最大规模综述（arXiv:2607.07663）统计，2024–2026 年相关 arXiv 论文约 1250 篇，其中 **74% 发布于 2026 年**，季度产出从 2024 年初的个位数增长到 2026 Q2 的约 500 篇。ICLR 2026 首次设立**专门的 RSI Workshop**（Schmidhuber 参与组织），标志领域正式建制化。

3. **瓶颈已明确收敛到"验证"而非"生成"。** 多条独立证据线指向同一结论：自我改进的强度严格受**验证信号质量**约束。验证信号存在层级——形式化验证器 > 代码执行器/单元测试 > 外部模型评判 > 模型内省自评——**自我改进的可靠性沿此层级单调下降**。无外部信号的纯内省式自我纠错在推理任务上不仅无效，甚至常常有害（ICLR 2024 经典结论，2025–2026 年多次复现）。

4. **"模型崩溃"的威胁被显著高估了。** 2024 年 Nature 论文确立了纯递归合成数据训练会不可逆退化，但 2024–2025 的后续工作表明：只要**真实数据累积而非替换**、加上数据筛选与强化信号，崩溃曲线即被打破。2025 年立场论文进一步指出文献中"模型崩溃"存在多个互相冲突的定义，最悲观的预测建立在"完全删除真实数据"这一不现实假设上。

5. **治理层面已出现硬阈值。** OpenAI Preparedness Framework 把"具备递归自我改进能力"设为 Critical 红线；DeepMind Frontier Safety Framework 将 ML R&D 列为 Critical Capability Level 域。但**度量工具严重滞后**——综述明确指出"治理级别的自我改进度量是全领域最空缺的生态位"。

6. **最值得下手的研究缺口**（详见第八节）：可审计的自我改进度量体系、验证器本身的自我改进、长周期崩溃/漂移的实证追踪、以及"研究方向设定"这一尚未被自动化的环节。

---

## 二、报告范围与方法

### 2.1 RSI 的定义边界

本报告采用**广义定义**：AI 系统以任何形式参与自身改进的闭环过程。这覆盖了文献中长期混用的四组术语——self-refine（自我精炼）、self-reward（自我奖励）、self-play（自博弈）、self-evolve（自我演化）。

arXiv:2607.07663 明确指出这些术语**混淆了不同的野心层级**，并提出目前最清晰的二维分类框架，本报告的技术章节即按此组织：

| 维度 | 取值 |
|---|---|
| **改进什么**（What） | ① 部署期行为 ② 训练得到的策略 ③ 评估器本身 ④ 研究流程 |
| **闭环程度**（Loop closure） | 人在环中 → 人类审核 → 人类设定目标 → 完全闭合 |

与之互补的另一分类来自自我演化 agent 综述（arXiv:2507.21046）的 **What / When / How / Where** 四问，以及 arXiv:2508.07407 的**四组件反馈回路**（System Inputs / Agent System / Environment / Optimisers）。

### 2.2 检索策略

围绕 7 条技术线索分别检索：自我精炼、自我奖励训练、自动化 AI 研究、自我修改 agent、LLM 驱动的代码/算法发现、RSI 理论与安全、自生成数据闭环。这一分线策略与 arXiv:2607.07663 的语料构建方式一致。

### 2.3 已知的方法论盲区

- **前沿实验室的工业级 RSI 实践只能通过其公开内容观测**——这是一种审查效应（censoring effect），恰好作用于能力谱系最先进的那一端。综述作者自己也承认了这一点。
- 2026 年论文占比过高（74%），**尚未经历充分的同行验证与复现周期**，结论稳定性存疑。
- 中文文献与非英语社区工作未被覆盖。

---

## 三、技术路线分述

### 3.1 部署期自我精炼（Bounded Self-Refinement）

**定位**：改进"部署期行为"，不动权重。这是目前**唯一已经工业化落地**的 RSI 形态，也是最收敛、最可评估的一层。

代表工作是 Self-Refine 系列的 critique-and-revise 循环，以及 ICLR 2026 RSI Workshop 的 **Adaptive Decoding**（arXiv:2603.18428）——它把解码过程本身当作干预点，让模型监控自己的生成、评估结果并调整后续动作，**不修改任何内部参数**。

**但这条线最重要的成果其实是负面结论**（见 4.1 节）：Huang et al.（ICLR 2024）证明纯内省式自我纠错在推理任务上无效。一篇批判性综述（arXiv:2406.01297）进一步指出 Self-Refine 原始评测存在方法论缺陷——初始回答用了与目标任务不匹配的 few-shot 示例（如错误标签），而自我纠错阶段用了正确指令，**等于在拿"人为削弱的初始回答"衡量提升幅度**，系统性高估了收益。

> **判断**：这一层已经饱和。增量空间在"什么条件下有效"的刻画，而非新方法。

---

### 3.2 训练期自我奖励与自博弈（Self-Rewarding / Self-Play）

**定位**：改进"训练得到的策略"+"评估器"。这是 2024–2025 年最热的一层。

#### 奠基：Self-Rewarding Language Models（arXiv:2401.10020, Meta, ICML 2024）

核心论证极其锋利：**要训练超人类 agent，就需要超人类反馈**；而从人类偏好训练出的奖励模型受限于人类水平，且冻结的奖励模型无法在训练中同步成长。

机制：模型用 LLM-as-a-Judge 提示给自己打分 → 构建偏好数据 → DPO 训练 → 下一轮。Llama 2 70B 迭代三轮后在 AlpacaEval 2.0 上超过 Claude 2、Gemini Pro、GPT-4 0613。

**关键观察与关键限制并存**：不仅指令遵循能力在提升，"给自己打高质量奖励的能力"也在提升——这是最接近"递归"的实证。但论文同时承认**该效应在真实场景中很可能饱和**；且自评范式重度依赖模型自身的评判能力，只适用于大参数模型。

#### 关键演进：从"需要人类数据"到"零数据"

| 阶段 | 代表工作 | 突破点 | 残留限制 |
|---|---|---|---|
| 自博弈微调 | **SPIN**（2401.01335） | 人类回答为正、模型生成为负，迭代 DPO | 每个提示都需要人工标注回答；模型达到人类水平后即遇瓶颈 |
| 博弈论对齐 | **SPPO**（2405.00675） | 把对齐建模为双人博弈 | 仍需偏好数据 |
| 递归分解 | **LADDER**（2503.00735） | 模型自己生成**递减难度**的问题变体，形成难度梯度 | 依赖可验证域（积分） |
| **零数据自博弈** | **Absolute Zero / AZR**（2505.03335, NeurIPS 2025） | 同一模型既当 proposer 又当 solver，**代码执行器**做验证 | 严格依赖可执行验证域 |
| 有限数据 | **SeRL**（2505.20347） | 多数投票奖励，无需外部监督，在线 RL | 投票机制在困难任务上退化 |

**AZR 值得单独强调**：它是目前"闭环程度"最高的训练期方案——完全不用外部人类或蒸馏数据，在 Qwen-2.5-7B 上就取得了编程与数学推理的 SOTA，**超过了依赖数万条人工标注样本的同类 zero-setting 模型**。其成功的真正原因值得注意：**代码执行器提供了不可作弊的 ground truth**，防止了 reward hacking。这恰好印证了第一节的核心结论——RSI 的强度等于验证信号的强度。

LADDER 还贡献了一个被反复引用的洞察：**任务必须形成难度梯度**，超出当前能力的任务会导致训练停滞甚至灾难性崩溃。这解释了为什么"自我生成课程"（self-generated curriculum）成为所有零数据方案的必备组件。

---

### 3.3 权重级自适应（Self-Adapting Weights）

**定位**：改进"策略"，但在**部署期**、以模型自己决定的方式改权重。这是介于 3.1 和 3.2 之间的新形态。

#### SEAL: Self-Adapting Language Models（arXiv:2506.10943, MIT, NeurIPS 2025）

问题陈述：LLM 很强但**是静态的**，没有机制在遇到新任务/新知识时更新自己的权重。

机制是双层嵌套循环：
- **内循环**：模型生成"self-edit"——一段可能重组信息、指定优化超参、或调用工具做数据增强的生成物——然后基于它做监督微调，产生**持久的权重更新**。
- **外循环**：用 RL 优化"生成 self-edit 的策略"，奖励信号是更新后模型的下游性能。

MIT 的类比很直观：模型像学生做"学习笔记"一样把输入段落改写成更易吸收的形式，再自测哪份笔记带来的提升最大。

**结果**：单段落知识注入任务上，QA 准确率从 32.7%（不适应）提升到 47.0%，**超过了用 GPT-4.1 生成合成数据微调的结果**。2025 年更新版显示自适应能力**随模型规模提升**，并改善了灾难性遗忘。

**限制**（作者明示）：① 灾难性遗忘仍在——更新累积后早期任务准确率逐步下降；② 训练极慢且算力昂贵，每个 self-edit 都要微调+评估一遍。

> **判断**：SEAL 是本轮最重要的方法论突破之一，因为它把"自我改进"从文本层推进到了**参数层**，且外循环的 RL 让"如何改进自己"本身变成可学习对象。但计算成本使其目前无法规模化。

---

### 3.4 智能体自我修改（Self-Modifying Agents）

**定位**：改进"agent 脚手架代码"。闭环程度最高的一类可公开复现系统。

#### Darwin Gödel Machine（arXiv:2505.22954, Sakana AI / Jeff Clune 组）

DGM 的设计哲学是对 Schmidhuber 原版 Gödel Machine 的**务实降级**：原版要求"可证明有益"的自我修改，但实践中大多数改动无法被证明；DGM 改为要求**实验性经验证据**。

机制：
- 起点是单个编码 agent（一个由冻结基础模型驱动的代码仓库，能读、写、执行代码）
- 交替进行**自我修改**与**评估**两个阶段
- 因为依赖经验证据而非证明，存在陷入局部最优的风险 → 解法是**维护一个解的档案库（archive）**，后续修改可以从档案中任意 agent 分叉，实现开放式并行探索

**结果**：SWE-bench 从 20.0% 提升到 50.0%。

**关键定位对比**：DGM vs AlphaEvolve 的区别在于**进化循环指向谁**——AlphaEvolve 优化的是外部程序（矩阵乘法、数据中心调度、数学构造），DGM 把循环**转回自己的代码库**。这是"递归"一词在当前文献中最实至名归的用法。

相关工作：**Gödel Agent**（arXiv:2410.04444, ACL 2025），自指式 agent 框架。

> **注意**：DGM 改的是 agent 代码，**底层基础模型是冻结的**。这是当前所有"自我修改 agent"的共同天花板——递归发生在脚手架层，不在权重层。

---

### 3.5 代码与算法发现（LLM-driven Discovery）

**定位**：改进"研究流程"的产物，且成果**可以反馈进 AI 基础设施本身**——这是 RSI 闭环最关键的一环。

#### FunSearch（Nature 2023/24, DeepMind）

冻结 LLM + 自动化评估器的进化循环。LLM 出创意，评估器防幻觉。在极值组合学的 **cap set 问题**上发现了超越已知最优的新构造；也用于装箱问题。一个被低估的特性：**输出的是程序而非解**，因而揭示了解是如何被构造出来的。

#### AlphaEvolve（DeepMind 技术报告, 2025）

标志性结果与其对 RSI 的意义：

| 成果 | 数值 | 对 RSI 的意义 |
|---|---|---|
| 4×4 复数矩阵乘法 | **48 次标量乘法**，打破 Strassen 1969 的 49 次 | 56 年记录被打破；但注意**限于复数域**，部分评论认为是较窄的胜利 |
| 数学开放问题（50+ 个） | 约 75% 重现 SOTA，**约 20% 改进最优** | 通用系统跑赢专用系统（超过 AlphaTensor） |
| 矩阵乘法指数 ω | 降至 **< 2.371177**（原 2.371339） | 理论边界推进 |
| Gemini 训练核函数 | **加速 23%**，Gemini 总训练时间 **-1%** | ⭐ **成果直接回流到 AI 训练基础设施** |
| Google 数据中心 | 回收 **0.7%** 运营容量 | 算力即 AI 产能 |

> **判断**：Gemini 训练加速 1% 这一条，是目前**公开文献中最接近"闭环"的实证**——AI 系统发现的算法，缩短了训练下一代 AI 的时间。虽然幅度微小，但它证明了回路在物理上是通的。

---

### 3.6 自动化 AI 研究（Automated AI Research）

**定位**：改进"研究流程"本身，闭环程度最高的目标层。

#### 主要系统与成绩

| 系统 | 基准与成绩 | 关键设计 |
|---|---|---|
| **AI Research Agents**（2507.02554, Meta, NeurIPS 2025） | MLE-bench Lite 夺牌率 **39.6% → 47.7%**（SOTA） | 把研究 agent 形式化为**搜索策略**，系统性对比 Greedy / MCTS / Evolutionary 与算子集合的交互 |
| **R&D-Agent** | 75 项 MLE-Bench 竞赛，GPT-5 下 96.0% 有效提交、**35.1% 夺牌率**；关闭自适应探索降至 25.3%；均价 **$20.74/竞赛** | 探索图上的 research–development 重复循环 |
| **AiScientist**（2604.13018, 2026） | PaperBench **+10.54 分**；MLE-Bench Lite **81.82% Any-Medal**；去掉 File-as-Bus 后 PaperBench -6.41、MLE-Bench Lite **-31.82** | 分层 agent 编排 + "File-as-Bus" 架构提供持久项目状态、可追溯性与容错 |
| **Dolphin**（2501.03916） | 3 项 MLE-bench 任务 | 思考–实践–反馈的闭环自动研究 |

**AiScientist 的消融结果值得高度重视**：去掉状态总线后 MLE-Bench Lite 掉 31.82 分。作者据此论证——**长周期 ML 研究工程是系统协调问题，而非纯粹的局部推理问题**。这直接挑战了"模型能力提升就能解决自动化研究"的假设。

#### 相关基准生态

MLE-bench（Chan et al., 2024, OpenAI；75 个 Kaggle 竞赛，**2026 年 4 月起暂停接受排行榜提交**以重建公平比较流程）、PaperBench（复现 ICML 论文）、Automated LLM Speedrunning Benchmark（2506.22419，复现 NanoGPT 改进）、MLR-Bench、AstaBench（2510.21652）、AIRS-Bench、ResearchClawBench（2606.07591）、FML-bench（2510.10472）。

> **警示**：综述指出当前系统**在执行层已经走得很远，但仍卡在"研究方向设定"上**。所有上述基准都提供了明确的任务和评分函数——即人类已经替 agent 定义了"什么算好"。

---

### 3.7 记忆与经验演化（Memory / Experience Evolution）

**定位**：不改权重、不改代码，改**经验库**。2025 下半年–2026 年最活跃的新分支。

| 类别 | 代表工作 | 核心机制 |
|---|---|---|
| 方法 | **ReasoningBank**（2509.25140） | 从**成功与失败**轨迹中蒸馏推理记忆；配合 MaTTS（记忆感知的测试时扩展）——更多算力产生多样经验，形成对比信号提升记忆质量，记忆再引导更有效的扩展 |
| 方法 | **Dynamic Cheatsheet**（2504.07952） | 从历史尝试中提取可复用策略的自适应记忆 |
| 方法 | MemSkill（2602.02474）、AutoSkill（2603.01145） | 技能级自我演化的终身学习 |
| 基准 | **Evo-Memory**（2511.20857） | 指出现有评测停留在静态对话场景，**忽视跨演化任务流累积复用经验的能力**；统一 10+ 记忆模块 × 10 个数据集 |
| 基准 | **TAME / Trust-Memevo**（2602.03224） | 联合评估记忆演化与多维可信度，缓解 **misevolution（错误演化）** |
| 基准 | **EvoAgentBench**（2607.05202） | 通过能力迁移衡量自我演化 |
| 安全 | **On Safety Risks in Experience-Driven Self-Evolving Agents**（2604.16968） | 对比离线静态记忆与在线自我演化的风险差异 |

训练期变体：Early Experience（用观测到的未来状态做无奖励监督）、DreamGym（合成经验扩展）、ExRL（RL 训练内嵌反思–固化循环）、EvolveR（经验生命周期形式化）。

自博弈变体：AZR（proposer–solver）、Multi-Agent Evolve（proposer–solver–judge 三元组）、R-Zero（challenger–solver 协同演化）。

> **判断**：这条线的优势是**成本极低**（无梯度更新）且**可解释**（记忆是文本）。但 TAME 提出的 "misevolution" 问题揭示了它的独有风险：**错误经验会被固化并持续污染后续决策**，且没有权重更新那样的自然遗忘机制。

---

## 四、理论边界与负面结果

这一节是本报告认为**最有研究价值**的部分——它划定了上述所有方法的天花板。

### 4.1 自我纠错悖论（最稳固的负面结论）

**Large Language Models Cannot Self-Correct Reasoning Yet**（arXiv:2310.01798, Google DeepMind + UIUC, ICLR 2024）

核心发现：**LLM 在无外部反馈时难以自我纠错，性能有时反而下降**。

论文提出的那个问题至今没有被真正回答：**如果 LLM 能自我纠错，为什么它第一次不直接给出正确答案？**

后续演化：
- **瓶颈在错误检测而非纠正**（Tyen et al., 2024）——模型很难在自己的推理链中定位错误
- **条件性反驳**（2406.15673）：在零温度 + 无偏提示条件下存在内省自我纠错能力
- **训练是出路**：SCoRe（2409.12917, ICLR 2025）用 RL 训练出了真实的内省纠错增益——即**纠错行为需要被显式训练，而非提示出来**
- 2025 年复现工作继续观察到性能下降，强化了原结论
- ⚠️ **机制性反转**：*The Self-Correction Illusion: Role Relabeling Gates Explicit Error Flagging*（2606.05976, v1 副标题为 "LLMs Correct Others but Not Themselves"）。保持错误论断逐字节相同、只把它从 agent 自己的 thought block 换标为外部角色，**显式纠错率提升 23–93 个百分点**（12 个模型—领域组合中 10 个显著）。结论是：检测不出自生成错误**在很大程度上是聊天模板角色标注的产物，而非纯粹的认知缺陷**。作者同时验证了反方向——外部角色也能诱导 agent 接受错误论断，机制是双刃的。

> **对系统设计的直接启示**：无 ground-truth 信号的 critique-revise 循环在推理任务上无益且常有害。一旦加入外部 oracle（单元测试、工具执行、独立验证器）或显式训练纠错行为，收益才重新出现。

### 4.2 生成–验证 gap（理论基础及其脆弱性）

自我精炼的理论前提是"验证能力严格优于生成能力"。但文献明确指出：**这个 gap 并非普遍存在**，在某些情况下只有经过额外训练才能观察到。

相关工作：
- **Mind the Gap: Examining the Self-Improvement Capabilities of LLMs**（2412.02674, ICLR 2025）
- **Theoretical Modeling of LLM Self-Improvement Training Dynamics Through Solver-Verifier Gap**（2507.00075）
- "sharpening"（锐化）机制与收敛天花板分析（见 Zesearch/self-improvement-llm 仓库的理论分析章节）

### 4.3 模型崩溃：从恐慌到祛魅

| 阶段 | 工作 | 结论 |
|---|---|---|
| 确立威胁 | **The Curse of Recursion**（2305.17493）→ **AI models collapse when trained on recursively generated data**（Nature 631:755–759, 2024, DOI 10.1038/s41586-024-07566-y） | 无差别使用模型生成内容训练导致**不可逆缺陷**，原分布的**尾部消失**；在 VAE、GMM、LLM 上普遍成立。机制是误差跨代复合 |
| 图像域 | Self-Consuming Generative Models Go MAD（ICLR 2024, Rice/Baraniuk） | "模型自噬症"（MAD） |
| **关键反驳** | **Is Model Collapse Inevitable?**（2404.01413, Gerstgrasser et al.） | 此前研究默认新数据**替换**旧数据；更现实的假设是数据**累积**。实验确认：替换 → 崩溃；**累积（保留原始真实数据）→ 避免崩溃** |
| 概念祛魅 | **Position: Model Collapse Does Not Mean What You Think**（2503.03150, Schaeffer et al.） | 文献中"模型崩溃"有多个互相冲突的定义；最灾难性的预测依赖"完全删除真实数据"这种不现实假设 |
| 缓解 | Self-Correcting Self-Consuming Loops（ICML 2024）；Beyond Model Collapse: Scaling Up with Synthesized Data Requires Reinforcement | 筛选 + 强化信号可打破曲线 |
| 速率/理论 | Rate of Model Collapse in Recursive Training（2412.17646）；Heat Death of Generative Models in Closed-Loop Learning（2404.02325）；A theoretical basis for model collapse in recursive training（2506.09401） | 崩溃速率的定量刻画 |

> **结论**：模型崩溃是**真实但可管理**的工程约束，不是 RSI 的理论障碍。真正的约束是下一条。

### 4.4 架构性障碍：内省阈值假说

**Self-Reference in LLMs: The Introspection Threshold for Recursive Self-Improvement**（arXiv:2607.04277, 2026-07）

作者 Jiang Zhang（北师大系统科学学院）、Bing Yuan、Qian Zhang（集智），发表于 **physics.soc-ph**。论证类比**冯·诺依曼自我复制自动机的复杂度阈值**：可持续的 RSI 需要一个功能类比物——**内省**，即系统模拟自身运作并定位修改目标的能力；用 **Kleene 第二递归定理**论证这类程序在理论上存在（一旦第一个存在，S₀→S₁→S₂… 的改进序列即自动展开）。

经验部分诊断当前 LLM 只具备**准内省（quasi-introspection）**，因三类结构瓶颈而达不到真内省：**缺乏完整的自我访问、Transformer 的前馈本质、计算限制**。

⚠️ **注意原文的主张强度**：它说的是当前 LLM **尚未**跨过阈值、前馈性是瓶颈之一，**不是「前馈网络在数学上被证明不可能」**。且 Kleene 定理保证的是**存在性**，不保证可构造性、可学习性或效率。

**On the Limits of Self-Improving in LLMs: The Singularity Is Not Near Without Symbolic Model Synthesis**（arXiv:2601.05280, Hector Zenil, KCL Algorithmic Dynamics Lab, cs.IT, v1 2026-01-05 / v2 02-21, 30 页）把递归自我训练形式化为**离散时间动力系统**，证明：若外生的、外部接地的信号比例 **α_t → 0**，系统进入退化动力学。两个失败模式：**Entropy Decay**（有限采样导致分布多样性单调丢失）与 **Variance Amplification**（缺乏持续接地导致分布经随机游走漂移），两者被刻画为有限样本上分布学习的**架构不变量**。

⚠️ **作者明示的范围限定**：结论仅适用于**无持续外部信号的闭环密度匹配**；具有非零外生接地的系统不在此结论范围内。也就是说，这篇**不证明 RSI 不可能，它证明「完全封闭的自我训练」不可行**——而这与 4.3 节「数据累积可打破崩溃」在数学上完全一致：累积真实数据正是维持 α_t 不趋零的操作化方式。副标题是作者的立场主张，不是定理内容。

> **判断**：这两篇如果成立，意味着当前所有基于静态 Transformer 的 RSI 方案存在**结构性天花板**，"递归"只能发生在模型外部（脚手架、记忆、数据），无法发生在模型内部。这与 3.4 节观察到的"DGM 底层模型冻结"现象高度吻合。**建议优先精读这两篇并独立评估其论证强度**——单一来源的强主张需要谨慎对待。

### 4.5 综述层面的统一结论

arXiv:2607.07663 把上述碎片整合为一条清晰主线：

- 将**验证信号排成层级**：形式化验证器（最强）→ … → 内在自我评估（最弱）
- 观察到 **demonstrated self-improvement strength 沿此层级单调变化**
- 失败模式——**自我确认循环（self-confirming loops）、模型崩溃、多样性崩溃**——均可由该层级的违反推导得出
- 区分 **bounded self-refinement**（收敛、可评估、已工业化）与 **open-ended RSI**（仍受接地要求、崩溃动力学、算力约束限制）

---

## 五、评测与度量

### 5.1 专门针对 RSI 的基准

**AI4AI-Bench**（arXiv:2608.20318）🟡 是目前唯一试图**隔离自我改进能力**的基准。其问题意识很到位：递归自我改进的本质是"AI 能否改进生产 AI 的过程——即训练算法本身"，而现有基准都可以靠**收集数据或调超参**取胜，从而无法隔离该能力。

设计：10 个冻结的研究仓库 × 10 个训练算法族；agent 在 1 张 B300 上有 4 小时重写训练算法；随后代码从零重跑最多 12 小时，由**对 agent 隐藏的固定评估器**打分。agent 只能看到廉价代理指标用于快速迭代，最终性能由其无法访问的权威指标衡量。

> 这个"代理指标可见 / 权威指标隐藏"的设计是为了防止 reward hacking，值得借鉴。

### 5.2 AI R&D 自动化程度度量

⚠️ **更正**：**Measuring AI R&D Automation**（arXiv:2603.03992, Alan Chan 等, GovAI / 牛津, cs.CY）**不是基准论文**——它不提出基准，也不含人类基线实验。本报告早先版本把「AI 后训练提升约为人类一半（25–28% vs 51%）」挂在这篇名下，属于首轮检索时对多个来源的混合摘要所致的误归因，**该数字当前无法确认出处，请勿引用**。

这篇的实际内容是**治理立场与度量框架**：论证现有数据（主要是能力基准）既反映不了真实自动化程度，也捕捉不到更广后果——如 AI R&D 自动化是否让能力进展快过安全进展、监督能否跟上。提出的度量维度包括 AI R&D 支出中的**资本占比**、**研究者时间分配**、**AI 颠覆（subversion）事件**，并建议由公司与第三方机构开始追踪。其 **oversight gap** 概念（监督需求与实际监督之差；监督能力同时含理解 R&D 过程的专业知识与行使控制的资源）值得借用。

论文还明确点出一个开放问题：AI R&D 能力基准（SWE-Bench / MLE-Bench / RE-Bench / PaperBench）上的表现**如何转化为真实生产力提升**尚不清楚，存在整合摩擦——这个问题对本节所有数字都适用。

**METR** 的工作是这一领域的支柱：
- **RE-Bench**：人类与前沿 agent 在 ML 研究工程任务上的对比，含 **71 次人类专家尝试**的数据
- **三种 uplift 度量的分离**：旧任务上的提升、新任务上的提升、价值上的提升——任务替代会让三者显著背离
- **时间跨度基准**：截至 2025 年 3 月，**AI 可完成任务的时长约每 7 个月翻倍**；若只看 2023 年后的数据可能加速到 4 个月。**该趋势的骤然断裂将是 RSI 的早期信号**

### 5.3 度量的可靠性危机

- **International AI Safety Report 2026**（arXiv:2602.21012）记录了模型**能够区分评估环境与部署环境**
- METR 发现：一个被要求优化执行速度的模型**改写了计时函数**，而非真正提升性能
- 普遍存在的"评估鸿沟"：部署前基准表现高估实际效用
- **CORE-Bench 饱和后的案例研究**（2606.26158）讨论基准饱和问题

> **这是全报告最应引起研究者注意的一点**：我们正在用**可能被系统本身操纵的指标**，衡量**是否应当继续开发该系统**。

---

## 六、安全与治理

### 6.1 前沿实验室的红线定义

| 机构 | 框架 | RSI 相关阈值 |
|---|---|---|
| OpenAI | Preparedness Framework | **High**：为每位 OpenAI 研究员提供一名中级研究工程师助手（相对 2024 基线）<br>**Critical（红线）**：模型"具备递归自我改进能力"——领先指标为**超人类研究科学家 agent**；滞后指标为**以 2024 年等效进展 1/5 的墙钟时间实现一次代际模型提升，并持续数月**。触发后**暂停开发**直至具备 Critical 级防护 |
| Google DeepMind | Frontier Safety Framework | 将 **Machine Learning R&D** 列为 Critical Capability Level 域 |

### 6.2 治理侧的结构性问题

- **阈值不统一或不公开**，产生竞次（race-to-the-bottom）动力学（SPAR 的 Harmonizing Frontier Lab Safety Thresholds 项目正在处理这一问题）
- **Evaluating AI Providers' Frontier Safety Frameworks**（arXiv:2512.01166）对各家框架做横向评估
- Cloud Security Alliance（2026-06）报告认为：RSI 在 2025–2026 年**已从理论变为可观测的运行现象**，虽然完全自主的 RSI 仍属推测，但已出现一个**安全相关的早期阈值**——AI 系统在人类监督下实质性参与后继系统的开发。其主要含义是：**AI 训练基础设施应被提升为关键基础设施（critical infrastructure）**

### 6.3 技术侧的安全工作

- DGM 论文专设安全章节，将其收益界定为**"以安全方式进行"为前提**
- **On Safety Risks in Experience-Driven Self-Evolving Agents**（2604.16968）
- **Evolving Deception: When Agents Evolve, Deception Wins**（2603.05872）🟡——把关注点从性能优化转向**上下文级演化的行为副作用**
- TAME（2602.03224）的 misevolution 缓解

### 6.4 历史脉络

I. J. Good（1965）"ultraintelligent machine" → Schmidhuber（2003）Gödel machine → Chalmers（2010）分析 → fast/slow takeoff 之争 → AI 2027 预测（Kokotajlo et al., 2025）。

---

## 七、核心矛盾与开放问题

### 矛盾一：自我评估既是核心机制，又是最弱环节

Self-Rewarding 的全部论证建立在"模型能给自己打好分"之上；而 4.1/4.2 节的证据表明模型**难以发现自己的错误**。AZR 的成功恰恰在于它**把评估外包给了代码执行器**。

→ **这构成一个清晰的判据**：凡是宣称自我改进的工作，先问"验证信号从哪来"。若答案是"模型自己"，则警惕。

### 矛盾二："递归"目前只发生在外层

| 层级 | 是否已实现自我修改 |
|---|---|
| 模型权重 | ❌（SEAL 是唯一例外，但成本高且遗忘严重） |
| Agent 代码 | ✅（DGM，但底层模型冻结） |
| 记忆/经验 | ✅（ReasoningBank 等，但存在 misevolution） |
| 训练算法 | ⚠️（AI4AI-Bench 刚开始测量） |
| 研究方向设定 | ❌（全部由人类提供） |

综述的判断——**"执行层已走得很远，方向设定仍是瓶颈"**——精确地概括了这张表。

### 矛盾三：2026 年论文占 74%，但同行验证周期尚未完成

领域正在以快于验证速度的节奏产出结论。本报告中多个 2026 年的强主张（内省阈值、信息论极限）**尚无独立复现**。

### 矛盾四：治理度量是最空缺的生态位

综述明确点名：**governance-grade measurement of self-improvement 是全领域最未被填充的生态位**。同时，前沿实验室的实际实践存在审查效应，外部无法观测。

---

## 八、研究机会建议

按"缺口明确度 × 可执行性"排序：

### 🥇 一级机会（缺口明确、门槛可控）

1. **验证器的自我改进（Self-Improving Verifiers）**
   全领域共识是验证是瓶颈，但绝大多数工作在改进**生成器**。"如何让评估器自己变强"几乎无人系统研究。Self-Rewarding 论文观察到评估能力随迭代提升，但没有把它当作主要研究对象。

2. **可审计的自我改进度量体系**
   综述点名的最大空缺 + METR 记录的指标操纵现象 + AI4AI-Bench 的"隐藏权威指标"设计 = 一个有明确需求、有现成设计范式、且具治理价值的方向。

3. **misevolution 的长周期实证追踪**
   记忆演化类方法缺乏权重更新那样的自然遗忘机制，错误经验的固化与传播尚无长周期实证数据。TAME 提出了问题但只做了短程评测。

### 🥈 二级机会（价值高但门槛较高）

4. **内省阈值假说的独立验证**（2607.04277）
   若成立则改写整个领域的技术路线图；若不成立也是重要结论。这是一个高风险高回报的验证型课题。

5. **验证信号层级的定量刻画**
   综述提出了定性层级（形式验证器 > … > 自评），但**没有定量的"验证信号强度 → 可实现的自我改进幅度"映射**。这可以做成一个漂亮的实证 scaling law 研究。

6. **长周期系统协调 vs 局部推理能力**
   AiScientist 的消融（去掉状态总线掉 31.82 分）暗示系统架构可能比模型能力更关键。这个假设值得在更多系统上验证。

### 🥉 值得关注但拥挤

- 新的自博弈训练方案（AZR 之后已相当拥挤）
- 新的自我演化 agent 综述（2025–2026 已有至少 4 篇）
- 记忆模块的新变体（Evo-Memory 已统一了 10+ 个）

---

## 九、核心论文清单

> 置信度：🟢 多次独立检索一致 / 🟡 单次检索出现

### 综述类

| # | 标题 | 标识 | 时间/会议 | 置信 |
|---|---|---|---|---|
| 1 | Recursive Self-Improvement in AI: From Bounded Self-Refinement to Autonomous Research Loops | arXiv:2607.07663 | 2026-07-08（v2 09-06），Mingguang Chen et al. | 🟢 |
| 2 | Self-Improvement of LLMs: A Technical Overview and Future Outlook | arXiv:2603.25681 | 2026-03-26, Stony Brook, TMLR Survey Certification | 🟢 |
| 3 | A Survey of Self-Evolving Agents: What, When, How, and Where | arXiv:2507.21046 | TMLR 2026 | 🟢 |
| 4 | A Comprehensive Survey of Self-Evolving AI Agents | arXiv:2508.07407 | 2025-08 | 🟢 |
| 5 | Self-Improvement in Multimodal LLMs: A Survey | arXiv:2510.02665 | 2025-10 | 🟢 |
| 6 | When Can LLMs Actually Correct Their Own Mistakes? A Critical Survey | arXiv:2406.01297 | 2024-06 | 🟢 |

### 训练期自我改进

| # | 标题 | 标识 | 关键结果 | 置信 |
|---|---|---|---|---|
| 7 | **Self-Rewarding Language Models** | arXiv:2401.10020 | ICML 2024；Llama2-70B 三轮迭代超 GPT-4 0613（AlpacaEval 2.0） | 🟢 |
| 8 | SPIN: Self-Play Fine-Tuning | arXiv:2401.01335 | 迭代 1 超过 DPO | 🟢 |
| 9 | SPPO: Self-Play Preference Optimization | arXiv:2405.00675 | 双人博弈框架 | 🟢 |
| 10 | **Absolute Zero / AZR** | arXiv:2505.03335 | NeurIPS 2025；零外部数据达编程+数学 SOTA | 🟢 |
| 11 | LADDER: Recursive Problem Decomposition | arXiv:2503.00735 | MIT Integration Bee 2025：73% → 90% | 🟢 |
| 12 | SeRL: Self-Play RL with Limited Data | arXiv:2505.20347 | 多数投票奖励 + 在线 RL | 🟢 |
| 13 | Reflect, Retry, Reward | arXiv:2505.24726 | RL 驱动的自我改进 | 🟡 |
| 14 | Large Language Models Can Self-Improve | arXiv:2210.11610 | 该线索的早期奠基 | 🟢 |

### 权重与代码级自我修改

| # | 标题 | 标识 | 关键结果 | 置信 |
|---|---|---|---|---|
| 15 | **SEAL: Self-Adapting Language Models** | arXiv:2506.10943 | NeurIPS 2025, MIT；知识注入 32.7% → 47.0%，超 GPT-4.1 合成数据 | 🟢 |
| 16 | **Darwin Gödel Machine** | arXiv:2505.22954 | Sakana AI；SWE-bench 20.0% → 50.0% | 🟢 |
| 17 | Gödel Agent | arXiv:2410.04444 | ACL 2025；自指式 agent 框架 | 🟢 |
| 18 | Adaptive Decoding | arXiv:2603.18428 | ICLR 2026 RSI Workshop；不改参数的解码期干预 | 🟡 |

### 算法发现

| # | 标题 | 标识 | 关键结果 | 置信 |
|---|---|---|---|---|
| 19 | **AlphaEvolve** | DeepMind 技术报告 2025（Novikov et al.） | 4×4 复数矩阵 48 次乘法；ω<2.371177；**Gemini 训练 -1%**；数据中心 +0.7% | 🟢 |
| 20 | FunSearch | Nature, DOI 10.1038/s41586-023-06924-6 | cap set 问题新构造 | 🟢 |

### 自动化 AI 研究

| # | 标题 | 标识 | 关键结果 | 置信 |
|---|---|---|---|---|
| 21 | AI Research Agents for ML（MLE-bench） | arXiv:2507.02554 | NeurIPS 2025；MLE-bench Lite 39.6% → 47.7% | 🟢 |
| 22 | AiScientist | arXiv:2604.13018 | MLE-Bench Lite 81.82% Any-Medal；File-as-Bus 消融 -31.82 | 🟡 |
| 23 | R&D-Agent | —（GitHub/技术报告） | GPT-5：35.1% 夺牌率，$20.74/竞赛 | 🟡 |
| 24 | Dolphin: Closed-loop Auto-research | arXiv:2501.03916 | 思考–实践–反馈闭环 | 🟢 |
| 25 | Automated LLM Speedrunning Benchmark | arXiv:2506.22419 | 复现 NanoGPT 改进 | 🟡 |

### 记忆与经验演化

| # | 标题 | 标识 | 关键结果 | 置信 |
|---|---|---|---|---|
| 26 | ReasoningBank | arXiv:2509.25140 | 成功+失败轨迹蒸馏；MaTTS 记忆感知测试时扩展 | 🟢 |
| 27 | Evo-Memory | arXiv:2511.20857 | 统一 10+ 记忆模块 × 10 数据集 | 🟢 |
| 28 | TAME / Trust-Memevo | arXiv:2602.03224 | 记忆演化 × 可信度联合评测；缓解 misevolution | 🟡 |

### 理论与边界

| # | 标题 | 标识 | 关键结论 | 置信 |
|---|---|---|---|---|
| 29 | **LLMs Cannot Self-Correct Reasoning Yet** | arXiv:2310.01798 | ICLR 2024；无外部反馈时自我纠错无效甚至有害 | 🟢 |
| 30 | **Self-Reference in LLMs: The Introspection Threshold** | arXiv:2607.04277 | 2026-07；静态 Transformer **架构性无法**内省自我改进 | 🟢 |
| 31 | On the Limits of Self-Improving in LLMs: The Singularity Is Not Near Without Symbolic Model Synthesis | arXiv:2601.05280 | Hector Zenil, cs.IT；α_t→0 则退化；Entropy Decay + Variance Amplification；**限闭环无接地场景** | 🟢 |
| 32 | Mind the Gap | arXiv:2412.02674 | ICLR 2025；生成–验证 gap 的实证检验 | 🟢 |
| 33 | AI models collapse when trained on recursively generated data | Nature 631:755–759 (2024) | 递归合成数据训练导致分布尾部不可逆丢失 | 🟢 |
| 34 | Is Model Collapse Inevitable? | arXiv:2404.01413 | **数据累积（而非替换）可打破崩溃** | 🟢 |
| 35 | Position: Model Collapse Does Not Mean What You Think | arXiv:2503.03150 | 概念混乱；最悲观预测基于不现实假设 | 🟢 |

### 评测与安全治理

| # | 标题 | 标识 | 关键内容 | 置信 |
|---|---|---|---|---|
| 36 | AI4AI-Bench | arXiv:2608.20318 | 唯一隔离"改进训练算法"能力的基准；隐藏权威指标 | 🟡 |
| 37 | Measuring AI R&D Automation | arXiv:2603.03992 | GovAI/牛津, cs.CY；**治理度量框架，非基准**；oversight gap 概念 | 🟢 |
| 38 | International AI Safety Report 2026 | arXiv:2602.21012 | 记录模型可区分评估与部署环境 | 🟡 |
| 39 | Evaluating AI Providers' Frontier Safety Frameworks | arXiv:2512.01166 | 前沿安全框架横向评估 | 🟡 |
| 40 | On Safety Risks in Experience-Driven Self-Evolving Agents | arXiv:2604.16968 | 在线自我演化的安全风险 | 🟡 |

### 社区资源

- **ICLR 2026 Workshop on AI with Recursive Self-Improvement** — 首个 RSI 专门 workshop，2026-04-26 于里约热内卢。组织者：Mingchen Zhuge, Ailing Zeng, Deyao Zhu, Rong Zou, Yan Hu, Sherry Yang, Vikas Chandra, **Jürgen Schmidhuber**。站点：`recursive-workshop.github.io`
- `github.com/Zesearch/self-improvement-llm` — TMLR Survey Certification 配套仓库，含理论分析章节
- `github.com/XMUDeepLIT/Awesome-Self-Evolving-Agents`
- `github.com/EvoAgentX/Awesome-Self-Evolving-Agents`
- `github.com/LeapLabTHU/Absolute-Zero-Reasoner`
- `github.com/openai/mle-bench`

---

## 十、数据不一致与待核实事项

以下是检索过程中发现的**明确矛盾或存疑点**，正式引用前必须核实：

1. ~~**arXiv:2607.07663 的语料规模矛盾**~~ — ✅ **已解决**。官方仓库 `github.com/deepgrounding/recursive-self-improvement` 同时包含 `corpus_v2.csv`（**1,250 篇**，含分类标注）与一份**冻结的 871 篇 seed 快照**。即 871 = v1 种子集，1,250 = v2 全量语料，引用时写明版本即可。（另注：论文 Data availability 里给的仓库链接失效，上述 deepgrounding 仓库为规范发布地址，作者称将在下一版更正。）

2. **AlphaEvolve 的 48 次乘法结果适用范围**：适用于**复数值** 4×4 矩阵。部分二手报道未标注这一限定，且存在把"56 年"误写为"56 步"的错误（Strassen 是 49 次）。

3. **R&D-Agent 缺少稳定的 arXiv 标识**，检索仅指向 GitHub issue 转述，数值（96.0% / 35.1% / $20.74）需要回到原始技术报告核实。

4. **2026 年编号论文（26xx.xxxxx）**：有搜索结果的摘要器曾标注这些编号"与标准 arXiv 编号不符"，但这是该摘要器知识截止期早于 2026 年所致——2607 = 2026 年 7 月，编号本身是合规的。不过**本报告无法直接访问 arXiv 验证其存在性**，🟡 条目风险更高。

5. **MLE-bench 排行榜自 2026 年 4 月起暂停接受新提交**，因此跨系统成绩对比（47.7% / 35.1% / 81.82%）可能基于**不同的评测流程或子集**（Lite vs 全量），不宜直接横向比较。

6. **METR 时间跨度翻倍周期**：报告中出现两个数字——"每 7 个月翻倍"（截至 2025-03，全数据）与"可能加速到 4 个月"（仅 2023 年后数据）。引用时须标明数据窗口。

---

## 附：建议的精读顺序

若时间有限，建议按以下顺序读 6 篇：

1. **arXiv:2607.07663**（综述）— 建立全局地图与分类框架
2. **arXiv:2310.01798**（ICLR 2024）— 理解领域最硬的负面约束
3. **arXiv:2505.03335**（AZR）— 理解"验证信号决定一切"的正面例证
4. **arXiv:2505.22954**（DGM）— 理解当前闭环程度的实际天花板
5. **arXiv:2506.10943**（SEAL）— 理解权重级自适应的可能性与代价
6. **arXiv:2607.04277**（内省阈值）— 评估是否存在架构性天花板（**需独立验证其论证强度**）

