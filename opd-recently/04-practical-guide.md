# OPD 落地指南：诊断、配方、框架选型

> 调研日期：2026-09-16
> 置信度：配方部分 🟢（来自 `Rethinking OPD` 原文对照实验）；框架部分 🟡（社区论文库条目描述）
> 一句话结论：**开跑前先测师生 top-k overlap ratio——
> 它能在烧算力之前告诉你这次 OPD 会不会白跑。**

---

## 1. 开跑前：三项诊断

OPD 的失败往往在第一步就已注定。以下三项在投入训练前完成。

### 1.1 测 top-k overlap ratio（最重要）🟢

**依据**：成功的 OPD 的 signature 就是共享高概率 token 上的渐进对齐，
overlap ratio 会从 ~72% 稳步升到 91%+；失败时 overlap 停滞。

**工具**：`thunlp/OPD` 官方代码含 top-k 师生 overlap 诊断，**已并入 verl 上游**。

**怎么用**：用少量 prompt 让学生采样，对每个位置算学生 top-k 与教师 top-k 的交集比例。
起手 overlap 低 → 思维模式不兼容 → 先做冷启动（§2.1），别直接上 OPD。

### 1.2 判断教师是否真有"新知识"🟢

**反模式**：选同族更大的模型当教师。
`Rethinking OPD` §3.3 显示同族 1.5B / 7B 教师**从学生视角分布上不可区分**——
大一号可能什么都教不了。

**判断依据**：教师是否经过了学生没经历过的后训练？
- 同 pipeline 出来的教师 → 增益有限
- **post-trained 教师**（做过学生没做过的 RL / 专项训练）→ 增益显著

### 1.3 确认 tokenizer 兼容性 🟡

跨模型族蒸馏的第一道墙。不兼容则需要支持跨 tokenizer 的框架（NeMo-RL / KDFlow / EasyOPD），
或改用黑盒 OPD 路线（outcome 级信号，放弃 logits）。

---

## 2. 两条救援配方（🟢 有对照实验）

来自 `Rethinking OPD` §5。适用前提：教师确实有新知识（否则救不回来），
但师生思维模式差距大。

### 2.1 配方一：Off-policy 冷启动

```
教师生成 rollouts  →  学生 SFT（冷启）  →  去重  →  标准 OPD
```

**原文验证设置**

| 项 | 值 |
|---|---|
| 学生 | Qwen3-1.7B-Base |
| 教师 | Qwen3-4B（Non-thinking） |
| prompt 源 | OpenThoughts3-1.2M 数学子集 |
| 冷启规模 | 教师生成 **200K** responses → SFT |
| OPD 规模 | 对 SFT prompt 去重后剩余 **~30K** prompts |

**收益**：不只是起步快——**最终性能天花板也被抬高**，差距贯穿全程。

**机制侧可观测指标**（用来确认配方生效）：
- 起手 overlap ratio 显著更高，轨迹平滑
- entropy gap 显著更小（更贴近教师置信度剖面）
- 对照的 base 初始化学生会出现**明显的前期震荡**

> 💡 这正是 Qwen3（2025）的做法。工业直觉先于学术解释约一年。

### 2.2 配方二：教师对齐的 prompt

两个粒度独立有效：

**（a）模板对齐**——用教师后训练时的 prompt 格式，而不是你习惯的格式。

原文的对照例子（同样的 DAPO-Math-17K 题目，只换模板）：

| | 模板 |
|---|---|
| 原始 DAPO | `Solve the following math problem step by step. The last line of your response should be of the form Answer: $Answer ... Remember to put your answer on its own line after "Answer:".` |
| **教师对齐** | `{Question} Please reason step by step, and put your final answer within \boxed{}.` |

教师对齐模板全程准确率与 overlap 增长都更高。**改一行模板的成本，换实打实的收益。**

**（b）内容对齐**——prompt 集尽量与教师的后训练/RL 训练集同源。
同源 prompt 的共享 token 质量集中度更高、熵显著更低。

> ⚠️ **必须混入 OOD prompt 防熵坍缩。**
> 原文：*"such prompts should be mixed with out-of-distribution prompts to prevent entropy collapse."*

---

## 3. 超参与训练期监控

### 3.1 优先级排序（基于 §6 的限制）

| 优先级 | 项 | 依据 |
|---|---|---|
| **高** | **响应长度**——存在 sweet spot，不是越长越好 | §6.1 🟢 |
| **高** | off-policy 冷启比例 | §5.1 🟢 |
| **高** | prompt 来源与模板 | §5.2 🟢 |
| 中 | top-k 的 k（论文默认 **16**） | §4.2 🟢 |
| 中 | 稳定化手段（截断 / 归一化 / trust region） | 见 §4 |
| **低** | 复杂 token 加权方案 | §6.3：采样 token 已足够 🟢 |

最后一行值得强调：**先把最简版本跑通，再考虑 TA-OPD / FiRe 这类精细加权**。
论文明确认为简单估计器可能已经够用。

### 3.2 训练期监控指标

| 指标 | 健康信号 | 异常含义 |
|---|---|---|
| **overlap ratio** | 稳步单调上升（如 72% → 91%） | 停滞 = 训练无进展，尽早止损 |
| **entropy gap** | 平稳收窄 | 骤降 = 熵坍缩（检查 prompt 多样性） |
| 响应长度 | 稳定在 sweet spot | 单调增长 = 长度膨胀，需修正 |
| 后段 token 的 loss | 与前段同步 | 后段异常 = §6.1 的深度退化在起作用 |

**第一行是核心。** overlap ratio 既是成功的 signature，也是自强化循环的观测量——
它不涨，训练就没在发生。

---

## 4. 框架选型 🟡

| 框架 | 差异化定位 | 适合 |
|---|---|---|
| **TRL** | GKD / GOLD / MiniLLM trainer，最易上手；`DistillationTrainer` 有 ~40× 加速案例（Qwen3-235B → 4B 数学） | **起步首选** |
| **NeMo-RL** | 大规模多教师 + 跨 tokenizer | MOPD、跨模型族 |
| **veRL** | **异步 on-policy KD——牺牲严格 on-policy 保证换吞吐** | 吞吐瓶颈场景（注意这是真实的取舍） |
| **thunlp/OPD** | `Rethinking OPD` 官方代码，**带 top-k overlap 诊断**（已并入 verl 上游） | **做诊断必备** |
| **EasyOPD** | verl 基座，方法级钩子覆盖跨 tokenizer / 自蒸馏 / step-wise | 要改方法 |
| **KDFlow** | 解耦后端，off-policy / on-policy / 跨 tokenizer 三合一 | 要对比多种蒸馏 |
| **Tinker Cookbook** | Thinking Machines SDK，单/多教师 OPD + 多轮工具使用配方 | 跟原版配方 |
| slime / ROLL / AReaL | 智谱 / 阿里 / 蚂蚁的统一 RL 栈，含 OPD | 已在用对应生态 |
| rLLM | UC Berkeley Sky，一等公民 OPD，verl 或 tinker 后端 | agent RL 场景 |
| MS-Swift | ModelScope 生态，含 GKD 与 OPSD | 国内生态 |
| SpecForge / TorchSpec | 投机解码 draft 模型训练 | 推理加速（不同问题） |

**推荐组合**：`TRL` 起步 → `thunlp/OPD` 做 overlap 诊断 → 规模化时按生态选 `NeMo-RL` / `veRL` / `slime`。

---

## 5. 决策树

```
你要解决什么？
│
├─ 模型太大、要出小模型
│   └─ 强到弱 OPD + top-k 截断
│      前置：测 overlap；教师须为 post-trained 而非同族放大
│      参考：OvisOCR2、MobileLLM-R1.5、Gemma 2
│
├─ 多个域专家要合成一个模型
│   └─ MOPD（多教师 OPD）
│      2026 工业主导方案；参考 DeepSeek-V4、KAT-Coder "Specialize-then-Unify"
│
├─ 多阶段 RL 后旧能力退化了
│   └─ 跨阶段蒸馏，用前序 checkpoint 当教师
│      天然满足思维模式兼容；参考 GLM-5
│
├─ RL 训练崩了 / 基座刷新后要恢复
│   └─ 在自己的 RL rollout 上自蒸馏
│      参考 MAI-Thinking-1
│
├─ 要修某个具体行为（工具调用、风格）
│   └─ hint-conditioned 自教师 OPD KL，叠加在 RL 之上
│      参考 Cursor Composer 2.5
│
├─ 没有外部教师，但有标准答案/执行反馈
│   └─ OPSD（特权上下文自教师）
│      ⚠️ 主要失败模式是 collapse，不是模式不兼容
│
└─ long-horizon / agent 场景
    └─ ⚠️ 谨慎：§6.1 显示奖励质量随轨迹深度退化，
       教师续写能力随 prefix 加深下降。这是当前最薄弱的方向。
```

---

## 6. 常见失败对照表

| 症状 | 最可能原因 | 处方 |
|---|---|---|
| **训练完全不动** | 思维模式不兼容，overlap 停滞 | off-policy 冷启（§2.1） |
| **换了更强的教师反而更差** | 同族放大，无新知识 | 换 post-trained 教师，先测分布可区分性 |
| **前期剧烈震荡** | base 直接起跑，初始 overlap 低 | 加 SFT 冷启 |
| **熵坍缩 / 多样性塌了** | prompt 过度对齐教师 | 混入 OOD prompt |
| **梯度爆炸** | 原始 KL 无截断 | top-k 截断 / 全局归一化 / trust region |
| **响应长度单调膨胀** | 长度未控，后段失稳 | 长度修正；检查 §6.1 的深度退化 |
| **pass@1 升了但 pass@k 掉了** | reverse KL 的 mode-seeking 收窄了能力边界 | 🟡 已知现象（arXiv:2608.11829），考虑混合 RL 目标 |
| **OPSD 长度坍缩** | 负向 teacher-agreement 压力 | 🟠 社区经验：只保留正向匹配 |

---

## 7. 三句话总结

1. **先诊断再训练**——overlap ratio 能在烧算力前预判成败，这是本领域性价比最高的一条实践
2. **教师选择比 loss 设计重要得多**——能力 > 规模，兼容 > 强大，post-trained > 同族放大
3. **从最简版本开始**——采样 token 估计器 + top-k 截断 + off-policy 冷启，这套跑通了再谈精细加权

---

**相关文档**：[总览](opd-2026-09.md) · [01 机制与失败模式](01-mechanism-and-failure-modes.md) · [03 工业界配方](03-industrial-recipes.md)
