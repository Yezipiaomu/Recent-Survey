# 评测可信度审计：为什么跨 harness 的分数不可比

> 调研日期：2026-09-16 · [返回索引](README.md)
> 姊妹文档：[../llm-recently/02-benchmark-credibility-audit.md](../llm-recently/02-benchmark-credibility-audit.md)（模型层的同类审计）
> 一句话结论：**模型榜单的问题是"厂商自报偏高"，harness 榜单的问题更严重——
> 同一个基准同时存在三套都"真实"的数字，且基准自己有约 30% 的任务是坏的。
> 引用任何 coding agent 分数，必须同时标注 harness + split + 试验次数。**

---

## 0. 铁律

```
❌ 「Claude Fable 5.1 在 SWE-bench Pro 上 81.2%」
✅ 「Claude Fable 5.1 在 SWE-bench Pro（厂商自有 scaffold，llm-stats 聚合口径）上 81.2%；
    同一基准 Scale 标准化 SEAL harness 的最高分是约 61.5%」
```

---

## 1. SWE-bench Pro：三套数字并存，全都"真实"

### 1.1 基准规模

来自 **41 个仓库的 1,865 个问题**，分 **public / held-out / commercial** 三个 split，
任务可跨多文件、执行周期长。定位是 Verified 饱和之后的新战场。

### 1.2 三个"最佳"

| 分数 | 模型 | 口径 | 可对等比较 |
|---|---|---|---|
| **59.1%** | GPT-5.4 xHigh | **Scale 标准化 public set** | ✅ |
| **80.0%** | Claude Fable 5 | llm-stats **厂商聚合**（各家自有 scaffold） | ❌ |
| **47.1%** | Claude Opus 4.6 | **Scale 私有 commercial set** | ✅（防污染） |

> 三者都真实，**差距来自 scaffolding 和数据划分**，而多数引用者从不说明用的是哪一套。

### 1.3 2026-09 榜首

截至 2026-09-10（共覆盖 72 个模型）：

| 排名 | 模型 | 分数 | 口径 |
|---|---|---|---|
| 1 | Claude Fable 5.1 | **81.2%** | 厂商 scaffold |
| 2 | Claude Mythos 5 | 80.3% | 厂商 scaffold |
| 3 | Claude Fable 5 | 80.0% | 厂商 scaffold |

另一家（CodingFleet，2026-09-08）列出：Fable 5.1 **81.2%**、Claude Opus 5 **79.2%**、
Qwen3.8 Max **67.7%**。

**而在 Scale 标准化 SEAL harness 上，最高约 61.5%（Muse Spark 1.1）——这才是可对等比较的口径。**

差距约 **20 个百分点**，全部来自 scaffold。

### 1.4 ⚠️ 基准本身有缺陷

> **OpenAI 2026-07 的审计估计，731 个 public split 任务中约 30% 存在缺陷。**

所以把 SWE-bench Pro 当作长周期仓库工作的证据之前，必须先对齐：
**split、scaffold、工具预算、重试策略、运行次数。**

---

## 2. Terminal-Bench：更诚实的做法

### 2.1 它直接承认 harness 影响

Terminal-Bench **按「模型-agent 组合」报告成绩**，而不是按模型。
常见组合：Claude Code、OpenAI Codex CLI、Terminus、Goose。

有研究在抓取时发现榜上有 **101 个 agent，来自 23 种不同 scaffold**
（OpenHands、Codex CLI、Aider 及各类自研实现）；其中 50 个是 2025-10 上线时就有的，
另外 51 个在随后约 3.5 个月内加入。

→ **这个比例本身就说明：大家都知道换 scaffold 能换分数。**

### 2.2 基准构成与提交规则

- **2.0**：89 个源自真实工作流的终端任务（4 Easy / 55 Medium / 30 Hard），16 个类别
  （软件工程、调试、安全、机器学习、科学计算等）；每任务独立容器 + 人工参考解 + 可执行验证测试
- **提交要求**：每任务 5 次试验（`-k 5`），使用任务自带基准环境与默认约束，
  **不得覆盖超时或 CPU/内存/存储限制**
- **指标**：avg@5 Accuracy（每任务 5 次取平均，再跨任务平均）
- **完整性规则会惩罚 reward hacking 轨迹**（例如从互联网检索任务答案）
- Artificial Analysis 的独立复现用 **Terminus 2 harness + e2b sandbox**，报告 3 次重复的 pass@1 平均

### 2.3 2.1 修订版揭示的关键事实

Terminal-Bench 2.1 修复了 2.0 中 **89 个任务里的 28 个**，问题分三类：

1. 构建后发生变化的**外部依赖**
2. **过紧的资源预算**
3. **指令与测试不匹配**

修订后不再有无解任务。而修订带来的分数变化是本文最重要的数据点：

> **增益最大的是 Claude Code + Opus 4.6，提升 12.1 个百分点。**

**换言之：基准自己的 bug 也是 harness 分数的来源之一。**
一个 12 个百分点的波动，足以颠覆任何"A 比 B 强 5 分"的结论。

---

## 3. Harness 效应的量级

| 观察 | 数据 |
|---|---|
| 同模型换 harness | Claude Opus 在 Claude Code 得 77%，在 Cursor 得 93%，**差 16 个百分点** |
| 跨研究的 harness 效应范围 | **5 – 40 个百分点**，取决于模型与任务类型 |
| 基准修订导致的单项变化 | **12.1 个百分点**（Terminal-Bench 2.0 → 2.1，Claude Code + Opus 4.6） |
| SWE-bench Pro 口径差 | **约 20 个百分点**（厂商 scaffold vs Scale SEAL） |

> **结论：harness 效应的量级 ≥ 模型代际差。**
> 「哪个模型强」这个问法在 agent 场景下本身就有问题。

---

## 4. 可直接用于决策的口径

| 用途 | 推荐口径 | 理由 |
|---|---|---|
| 比较**模型**的 agent 能力 | **Scale 标准化 SEAL harness**（SWE-bench Pro） | 唯一控制了 scaffold 变量 |
| 比较**模型 + harness 组合** | **Terminal-Bench 2.1**（avg@5，公开轨迹） | 明确按组合报告，且要求公开轨迹核验 |
| 独立复现 | **Artificial Analysis**（Terminus 2 + e2b） | 固定 harness + 固定 sandbox + 3 次重复 |
| ❌ 不要用于决策 | 厂商自有 scaffold 的 SWE-bench Pro 数字 | 无法对等 |
| ❌ 不要用于决策 | 发布周的任何分数 | 见姊妹文档：AA 曾两周内改三次索引 |

---

## 5. 审计清单

引用任何 coding agent 分数之前，逐项确认：

- [ ] 用的是哪个 **harness/scaffold**？（Claude Code / Codex CLI / Terminus 2 / SEAL 标准化 / 厂商自研）
- [ ] 用的是哪个 **split**？（public / held-out / commercial）
- [ ] 跑了几次？取平均还是取最好？（avg@5 vs pass@1 vs best-of-n）
- [ ] 基准版本？（Terminal-Bench 2.0 还是 2.1——差 12.1 个百分点）
- [ ] 资源约束有没有被覆盖？（超时、CPU、内存）
- [ ] 轨迹是否公开可核验？
- [ ] 该基准本身有没有已知缺陷率？（SWE-bench Pro public split ≈ 30%）

**任何一项缺失，该分数只能作方向性参考，不能进决策。**

---

## 6. 与模型层审计的对照

| | 模型榜单（[../llm-recently/02](../llm-recently/02-benchmark-credibility-audit.md)） | Harness 榜单（本文） |
|---|---|---|
| 主要问题 | 厂商自报偏高 | **同一基准三套数字并存** |
| 次要问题 | 评测索引频繁改版 | **基准自身约 30% 任务有缺陷** |
| 变量控制 | 模型固定，看分数 | **必须同时固定模型和 scaffold** |
| 可信来源 | Artificial Analysis v4.3、Vals AI | Scale SEAL、Terminal-Bench 2.1、AA(Terminus 2) |
| 效应量级 | 代际差数分 | **harness 效应 5–40 分** |
