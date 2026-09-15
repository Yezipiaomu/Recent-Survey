# 评测可信度审计：四家自报数字哪些能信

> 调研日期：2026-09-15
> 一句话结论：**四家的自报分数里，DeepSeek 的 Terminal-Bench 自报值高过第三方测出的全球最高分，
> MiniMax 的自报与第三方落差最大，智谱的数字方法学问题最集中。
> 唯一可直接用于决策的是 Artificial Analysis v4.3 和 Vals AI 的独立 harness 结果。**

---

## 0. 审计方法

给每个数字打三档：

| 档 | 含义 | 可用于 |
|---|---|---|
| **A** | 第三方独立 harness 测出，方法学公开 | ✅ 直接用于选型决策 |
| **B** | 厂商自报，但第三方在同方向有交叉印证（排序一致） | ⚠️ 可用于判断趋势，不可用于精确比较 |
| **C** | 纯自报：自选 harness / 自选对手分数 / 单次运行 / 无方差 | ❌ 仅作厂商叙事参考 |

---

## 1. A 档：第三方独立结果

### 1.1 Artificial Analysis Intelligence Index v4.3（最权威的综合口径）

> [方法论](https://artificialanalysis.ai/articles/artificial-analysis-intelligence-index-v4-3) ·
> [榜单](https://artificialanalysis.ai/evaluations/artificial-analysis-intelligence-index)

**开源权重排名：**

| 排名 | 模型 | 分数 |
|---|---|---|
| 1 | **GLM-5.3 (max)** | **44** |
| 1 | **Kimi K3 (max)** | **44** |
| 3 | GLM-5.3-Flash | 42 |
| 4 | Qwen3.8 2.4T A95B | 40 |
| 5 | DeepSeek V4-Pro 0813 (max) | 36 |

**前沿对照**：Claude Fable 5.1 和 GPT-6 Astra 均为 **53**。
→ **开源最强落后前沿 9 分**（不是 2.7%，见 §4.1 的口径辨析）。

**v4.3 方法论要点**（决定了这个分数意味着什么）：
- 10 项评测：AA-Briefcase、GDPval-AA v2、AutomationBench-AA、Terminal-Bench v4.0、SciCode、
  Humanity's Last Exam、GDP.pdf、CritPt、AA-Omniscience、AA-LCR v1.1
- **私有题目/答案占 45% 权重**（v4.2 为 40%）← 这是抗污染的关键设计
- v4.3 提高了 agent 编码任务难度，拓宽了 agent 工作流类型

**⚠️ 重要缺口**：**DeepSeek V4.1-Flash 尚未进榜**。当前跟踪的仍是 V4-Pro 0813。
这意味着 DeepSeek 最新一代**没有任何第三方综合评分**。

### 1.2 Vals AI（独立 harness，编码/agent 更细）

> [vals.ai](https://www.vals.ai/home)

| 榜单 | 结果 |
|---|---|
| **Vals Index（综合）** | Kimi K3 开源第一；**GLM-5.3 = 57.0**，开源第二、总榜第 13（GLM-5.2 为第 18） |
| **Vals Index 编码类目** | **GLM-5.3 = 64.3（开源第一）** > DeepSeek V4-Flash 61.4 > Kimi K3 60.6 |
| **Terminal-Bench 2.1** | **Kimi K3 = 80.9%** > GLM-5.3 = 71.5%（GLM-5.2 为 67.8%） |
| **SWE-bench Verified** | GPT-5.6 Sol 96.2% > Claude Fable 5 95.0% > Kimi K3 93.4% > Claude Opus 4.8 88.6% |

前沿对照：Claude Fable 5 拿下 Vals Index 第一（75.15%）和多模态第一（74.15%），
并领跑所有有评分的编码基准（Vibe Code Bench 90.35%、SWE-bench Verified 95.00%、
Terminal-Bench 2.1 80.52%、LiveCodeBench 89.78%、IOI v1 72.25%）。

**注意一处口径冲突**：Vals AI 官方 X 账号称 GLM-5.3「SWE-bench 95.4%、Vibe Code Bench 78.1%（比 5.2 涨 14 分）」，
若成立 GLM-5.3 应排 SWE-bench 第二。但这与上表「Kimi K3 93.4% 第三」的列表未同时出现。
**可能是 SWE-bench 与 SWE-bench Verified 两个不同数据集**，勿混用。

### 1.3 榜单是滚动的——一个必须注意的陷阱

同一个第三方对同一款模型的结论会随时间变化：

- **2026 年 6 月**（M3 刚发）：Vals AI 称 MiniMax M3 综合第 6（58.94%），**开源第一**
- **2026 年 8 月后**（GLM-5.3 / K3 已发）：Vals AI 称 **Kimi K3 开源第一**，GLM-5.3 第二（57.0）

M3 的 58.94 数值上高于 GLM-5.3 的 57.0，但排名反而落后——
**说明 Vals Index 的组成或计分在此期间变过，两个数字不可直接相减**。

同理，MiniMax M3 的 Artificial Analysis「Intelligence Index 30」
与 GLM-5.3/K3 的「44」**大概率不是同一版本索引**（v4.1 vs v4.3），
横向比较必须确认版本号。

---

## 2. C 档：问题最集中的自报数字

### 2.1 DeepSeek — 自报值超过第三方全球最高分 🚩

DeepSeek 自报 V4-Pro（0813）**Terminal Bench 2.1 = 87.9%**。

对照 Vals AI 独立 harness 的同一基准：

| 模型 | Terminal-Bench 2.1（Vals 独立测） |
|---|---|
| Kimi K3 | 80.9% |
| Claude Fable 5 | 80.52% |
| GLM-5.3 | 71.5% |
| **DeepSeek 自报 V4-Pro** | **87.9%** ← 比第三方测出的全球最高还高 7 分 |

**这不一定是造假**——Terminal-Bench 对 agent 脚手架极度敏感，
同一模型换 harness 分数可以差 10 分以上。但它明确说明：
**DeepSeek 的 87.9 与 Vals 的 80.9 不在同一个测量体系里，不能放进同一张表比较。**

同时注意 §1.1 的反差：DeepSeek V4-Pro 在 Artificial Analysis 综合榜只有 **36 分，开源第五**，
落后 GLM-5.3 / Kimi K3 **8 分**。自报的编码强势与第三方综合评分之间存在明显张力。

**待补**：V4.1-Flash 自报「编码与 agent 基准超过 V4-Pro」，
但 V4.1-Flash 无任何第三方评分，V4-Pro 的第三方评分又只有 36。**这条链路目前完全无法验证。**

### 2.2 智谱 — 方法学问题最密集

| 问题 | 细节 |
|---|---|
| **用竞品 harness 测自己** | GLM-5.3 的 CyberGym、ExploitGym、ExploitBench、Terminal Bench 等评测**在 Claude Code 2.1.207 内运行** |
| **单次运行无方差** | CyberGym 84.5% 是跨 1507 个任务的**单次 pass@1**，未给方差；其超越 Mythos 5（83.8）和 GPT-5.6 Sol（83.6）的差距**完全在运行间噪声内** |
| **自选对手分数** | 几乎每个数字都由智谱在自家 harness 上、用自己选定的竞品分数对比 |
| **发布时零外部复现** | 上线时无任何外部实验室复现过任何一项 |
| **基准版本的叙事选择** | 自报 **Terminal-Bench 3.0 从 4.6 → 28.3（6.2×）**；而 Vals 在 **Terminal-Bench 2.1** 上测出 71.5%。两者版本不同不可比，但 3.0 的低基数让「6.2 倍」这个叙事格外醒目 |

**一个反向加分项**：GDPval 一行由 **Artificial Analysis 而非智谱**评分，GLM-5.3 得 **1769**，高于两家闭源领先者。
这是智谱数据里少数 A 档的。

**一个容易被忽略的负面独立发现**：
Artificial Analysis 估算 GLM-5.3 在 Intelligence Index 上的**每任务成本约 $0.68，而 GLM-5.2 约 $0.44**——
**token 单价完全相同**，差异来自 GLM-5.3 明显更啰嗦。
→ **分数涨了，但单位任务成本涨了 55%。** 这个信息不会出现在厂商的发布材料里。

### 2.3 MiniMax — 自报与第三方落差最大

| 来源 | M3 的评价 |
|---|---|
| **MiniMax 自报** | SWE-Bench Pro 59.0%、MCP Atlas 74.2%、BrowseComp 83.5（超 Opus 4.7 的 79.3）、OmniDocBench 超 Gemini 3.1 Pro、SVG-Bench 超 Opus 4.7、Claw-Eval 登顶 |
| **Artificial Analysis** | Intelligence Index **30**（同体量开源中位数 18——**这个对比是正面的**） |
| **Vals AI** | 综合第 6（58.94%）开源第一；但 **SWE-bench Verified 75.00% 仅第 17**、**Terminal-Bench 2.1 53.56% 仅第 12** |
| **BenchLM** | 232 个模型中**第 52**（61.55/100）；多模态最强（第 34），**Agentic 最弱（第 107）** |

**核心矛盾**：MiniMax 把 M3 定位为「编码与 agent 前沿」，
但第三方在 **agent 维度给出的评价恰恰最低**（BenchLM agentic 第 107）。

**MiniMax 自己披露的口径问题**（这一点值得肯定——主动披露了）：
多项结果跑在**自家基础设施**上，常使用 **Claude Code / Mini-SWE-Agent / Terminus** 作为 agent 脚手架。

### 2.4 Kimi — 自报问题相对最少

K3 的主要宣称（WebDev Arena 第一 1678 Elo、Artificial Analysis 第 3、Vals 第 2）
**基本都来自第三方榜单而非自测**，这是四家里最干净的。

自报项主要是 DeepSWE 67.3（mini-SWE-agent harness，已注明 harness）。

第三方也基本支持其定位：AA v4.3 开源并列第一（44）、Vals AI 开源第一、Terminal-Bench 2.1 开源第一（80.9%）。

**唯一需要打折的**：「单位算力智能提升 2.5×」是架构层面的自报口径，
基于 Kimi Linear 论文在 **48B/3B 规模**上的实验，
**未在 2.8T 的 K3 规模上由第三方验证过**。

---

## 3. 逐项裁决表

| 数字 | 来源 | 档 | 可否用于决策 |
|---|---|---|---|
| GLM-5.3 = Kimi K3 = 44（AA v4.3 开源第一） | Artificial Analysis | **A** | ✅ |
| Kimi K3 Terminal-Bench 2.1 = 80.9% | Vals AI | **A** | ✅ |
| GLM-5.3 编码类目 64.3 开源第一 | Vals AI | **A** | ✅ |
| GLM-5.3 每任务 $0.68（vs 5.2 的 $0.44） | Artificial Analysis | **A** | ✅ 成本决策必看 |
| M3 BenchLM agentic 第 107 | BenchLM | **A** | ✅ |
| GLM-5.3 GDPval 1769 | Artificial Analysis 评分 | **A** | ✅ |
| M3 Intelligence Index 30 vs 同体量中位数 18 | Artificial Analysis | **A** | ⚠️ 确认索引版本 |
| Kimi K3 WebDev Arena 第一（1678 Elo） | WebDev Arena | **A** | ✅ |
| DeepSeek V4-Pro Terminal Bench 2.1 = 87.9% | 自报 | **C** | ❌ 超过第三方全球最高 |
| DeepSeek V4.1-Flash「超过 V4-Pro」 | 自报 | **C** | ❌ 无任何第三方评分 |
| 智谱 CyberGym 84.5% | 自报，单次 pass@1 | **C** | ❌ 差距在噪声内 |
| 智谱 Terminal-Bench 3.0 = 28.3（6.2×） | 自报，Claude Code harness | **C** | ❌ 版本不可比 |
| 智谱「编码提升 50%」 | 自报 | **C** | ❌ |
| MiniMax M3 BrowseComp 83.5 超 Opus 4.7 | 自报，自家基础设施 | **C** | ❌ |
| MiniMax M3 SWE-Bench Pro 59.0% | 自报 | **C** | ❌ 第三方 SWE-bench Verified 仅第 17 |
| Kimi「单位算力智能 2.5×」 | 自报，48B 规模外推 | **B** | ⚠️ 方向可信，倍数存疑 |
| 智谱「发现 2436 个漏洞 / 269 项目」 | 自报 | **B** | ⚠️ 可核查但未被核查 |

---

## 4. 两个流传很广但需要辨析的宏观数字

### 4.1「中美差距收窄到 2.7%」

来源是 **Stanford HAI AI Index 2026**，指的是**头部模型**（含闭源）之间的差距。

但 Artificial Analysis v4.3 显示：**开源最强（44）vs 前沿（53）差 9 分**，
按比例算约 **17%**。

两个数字不矛盾——**衡量对象不同**：
- 2.7% 是「中国最强模型 vs 美国最强模型」（可能含未公开/闭源系统）
- 9 分是「开源权重最强 vs 闭源前沿最强」

引用时务必说明是哪一种，否则会得出过于乐观的结论。

### 4.2「中国实验室占 OpenRouter 45% token 流量」

这是**用量**指标，不是**能力**指标。
驱动因素中价格占很大比重（见 [03-selection-guide.md](03-selection-guide.md)：
DeepSeek 离峰输入 $0.15/M vs Claude/GPT 的 $3–15/M）。
用它论证「能力追平」是偷换概念；用它论证「性价比路线成功」则完全成立。

---

## 5. 给自己做评测时的建议

基于以上审计，如果要自测：

1. **固定 harness**。Terminal-Bench 类基准换脚手架分数可差 10 分以上，
   DeepSeek 87.9 vs Vals 80.9 的差距很可能主要来自这里。
2. **跑多次报方差**。智谱 CyberGym 单次 pass@1 的教训——1–2 分的差距毫无意义。
3. **量每任务成本而非 token 单价**。GLM-5.3 的例子说明二者可以完全脱钩。
4. **区分 SWE-bench 与 SWE-bench Verified**、**Terminal-Bench 2.1 与 3.0 与 v4.0**。
   这几组名字相近的基准在本次调研中已造成至少三处口径冲突。
5. **优先看私有题目占比高的榜**。AA v4.3 的 45% 私有权重是目前抗污染做得最好的。

---

## 来源

- [Artificial Analysis Intelligence Index v4.3 方法论](https://artificialanalysis.ai/articles/artificial-analysis-intelligence-index-v4-3)
- [Artificial Analysis Intelligence Index 榜单](https://artificialanalysis.ai/evaluations/artificial-analysis-intelligence-index)
- [Artificial Analysis 开源模型对比](https://artificialanalysis.ai/models/open-source)
- [Artificial Analysis v4.1（早期版本，M3 的 30 分口径）](https://artificialanalysis.ai/articles/artificial-analysis-intelligence-index-v4-1)
- [Vals AI](https://www.vals.ai/home) · [Vals AI — MiniMax-M3](https://www.vals.ai/models/minimax_MiniMax-M3)
- [BenchLM — Artificial Analysis 镜像榜](https://benchlm.ai/benchmarks/artificialanalysis) · [BenchLM — MiniMax M3](https://benchlm.ai/models/minimax-m3)
- [AI News — GLM-5.3 基准方法学质疑](https://www.artificialintelligence-news.com/news/zhipu-glm-5-3-benchmarks-explained/)
- [The Decoder — GLM-5.3 发布](https://the-decoder.com/zhipu-ai-releases-glm-5-3-claims-its-the-strongest-open-weights-coding-model/)
- [24/7 Wall St — 开源落后前沿 9 分](https://247wallst.com/cards/xpost-01m1yj852p230gg5tksfn4q7bh)
- [Morph — 2026 最佳开源编码模型对比](https://www.morphllm.com/best-open-source-coding-model-2026)
