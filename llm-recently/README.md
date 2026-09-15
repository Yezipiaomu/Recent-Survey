# 中国四大模型厂商调研：DeepSeek / Kimi / 智谱 / MiniMax

> 调研日期：2026-09-15
> 范围：截至 2026 年 9 月中旬的研发进展、技术架构、评测可信度与选型建议

---

## 文档索引

| 文档 | 内容 | 适合谁看 |
|---|---|---|
| [china-llm-labs-2026-09.md](china-llm-labs-2026-09.md) | **总览**：四家的模型节奏、技术要点、商业数据、横向对比 | 先看这个 |
| [01-sparse-attention-comparison.md](01-sparse-attention-comparison.md) | **稀疏注意力架构横向对比**：DSA / KDA / MSA / IndexShare 逐个拆解 | 关心架构与长上下文 |
| [02-benchmark-credibility-audit.md](02-benchmark-credibility-audit.md) | **评测可信度审计**：哪些数字能信、哪些是营销 | 要用分数做决策 |
| [03-selection-guide.md](03-selection-guide.md) | **选型落地建议**：价格、成本陷阱、场景推荐、实测方案 | 要选一个来用 |
| [profiles/deepseek.md](profiles/deepseek.md) | DeepSeek 完整档案 | |
| [profiles/kimi-moonshot.md](profiles/kimi-moonshot.md) | 月之暗面 / Kimi 完整档案（含蒸馏争议客观梳理） | |
| [profiles/zhipu-zai.md](profiles/zhipu-zai.md) | 智谱 / Z.ai 完整档案 | |
| [profiles/minimax.md](profiles/minimax.md) | MiniMax 完整档案 | |

---

## 五个核心结论

### 1. 架构竞争已从「堆参数」转到「每 token 激活多少」

Kimi K3 用 2.8T 总参数只激活 **104B**；DeepSeek V4.1-Flash 用 552B 骨干在 prefill 时只激活 **8B**。
这两个数字比任何 benchmark 都更能说明方向。

### 2. 「稀疏注意力」四家路线完全不同——其中智谱根本没自研

| 厂商 | 路线 | 自研程度 |
|---|---|---|
| DeepSeek | token 级选择（DSA）+ KV 压缩（CSA/HCA）+ 拓扑重构（CED） | 全自研，**行业基线** |
| Kimi | 线性注意力（KDA）+ 24 层全注意力兜底 | 全自研，**唯一换掉注意力本身的** |
| MiniMax | 块级选择（MSA），保持未压缩真实 KV | 全自研 |
| **智谱** | **直接用 DeepSeek 的 DSA**（架构名就叫 `glm_moe_dsa`）+ 自研 IndexShare | **仅增量改进** |

→ 底层架构正在变成公共品。**看架构创新盯 DeepSeek / Kimi / MiniMax，看后训练与落地盯智谱。**

### 3. 厂商自报的分数普遍不能直接用

- **DeepSeek** 自报 Terminal Bench 2.1 = 87.9%，**比第三方测出的全球最高分（80.9%）还高 7 分**
- **MiniMax** 自称「编码与 agent 前沿」，但 BenchLM 的 **agentic 类目给它排第 107**
- **智谱**在 **Claude Code 内**跑自己的评测，CyberGym 是单次 pass@1 无方差
- **Kimi** 是四家里自报成分最少的——主要宣称都来自第三方榜单

可直接用于决策的只有 Artificial Analysis v4.3 和 Vals AI。详见 [02](02-benchmark-credibility-audit.md)。

### 4. agent 负载选型看 cache read 单价，不看 token 单价

| 模型 | cache read | 相对成本 |
|---|---|---|
| DeepSeek V4.1-Flash（离峰） | $0.003 | **1×** |
| GLM-5.3 | $0.26 | ~87× |
| Kimi K3 | $0.30 | **~100×** |

同一场景 DeepSeek 约 $0.15、Kimi K3 约 $15。**这个量级差距足以盖过模型能力差异。**

另一个坑：GLM-5.3 与 GLM-5.2 **token 单价完全相同**，但每任务成本从 $0.44 涨到 **$0.68（+55%）**，
纯粹因为更啰嗦。详见 [03](03-selection-guide.md)。

### 5. 收入撑不起估值，开源正在收紧

| 厂商 | 收入 | 估值/市值 |
|---|---|---|
| Kimi | ARR >$300M | $500 亿（IPO 递表） |
| 智谱 | 上半年 $136M | ~$160 亿 |
| MiniMax | 上半年 $117M | 已上市 |
| DeepSeek | — | 拟融资估值 $710–740 亿 |

智谱披露了最关键的一个数字：**API 业务毛利仅 0–10%，真正赚钱的是类 Palantir 的本地化部署（整体毛利 40%）。**

同时四家全部走向公开市场后，开源从战略变成战术：
GLM-5.3 权重延后两周、Kimi K3 加收入门槛（年收入 >$2000 万须签约）、MiniMax 设 $2000 万红线。

---

## ⚠️ 使用须知

1. **区分两个「差距」数字**。Stanford HAI 的「中美差距 2.7%」指头部模型（含闭源）；
   Artificial Analysis v4.3 显示**开源最强（44）落后前沿（53）9 分**，约 17%。二者衡量对象不同。
2. **「中国占 OpenRouter 45% 流量」是用量指标，不是能力指标**，价格是主要驱动力。
3. **基准名字相近但不可比**：SWE-bench ≠ SWE-bench Verified；
   Terminal-Bench 2.1 ≠ 3.0 ≠ v4.0。本次调研中这已造成至少三处口径冲突。
4. **第三方榜单是滚动的**。同一模型的同一分数，在不同时间点的排名可能相反
   （M3 的 58.94 排名低于 GLM-5.3 的 57.0）。引用时必须带索引版本号和日期。
5. **本调研基于公开报道与检索**，未做任何实机测试。所有选型建议都应经 [03](03-selection-guide.md) §5 的实测流程验证。

---

## 已知勘误

- **2026-09-15**：初版 `china-llm-labs-2026-09.md` 称智谱使用「native sparse attention」有误，
  实为直接采用 DeepSeek 的 DSA。已更正，并在 [01](01-sparse-attention-comparison.md) §2.3 详述。

## 最大的信息缺口

- **DeepSeek V4.1-Flash 无任何第三方评分**（AA 榜跟踪的仍是 V4-Pro 0813，36 分开源第五）。
  其「超过 V4-Pro」的宣称目前完全无法验证。
- **token 级 vs 块级稀疏选择谁更优**，DeepSeek 与 MiniMax 的宣称直接冲突，无中立实验。
- **Kimi 蒸馏指控**的证据始终未公开（白宫与 Anthropic 均未释出访问日志或取证材料）。
