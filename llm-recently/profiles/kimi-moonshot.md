# 月之暗面 / Kimi 公司档案

> 调研日期：2026-09-15 · [返回索引](../README.md)
> 定位：**规模与架构双押的激进派**。四家里唯一换掉注意力机制本身的，
> 也是唯一正面临美国政府制裁威胁的。

---

## 1. 模型谱系（2026）

| 时间 | 版本 | 规模 | 关键点 |
|---|---|---|---|
| 2026-01 | K2.5 | 1T / 32B 激活 | MoE |
| 2026-04 | K2.6 | — | 推动估值从 33 亿升至 200 亿美元 |
| 2026-06 | K2.7 Code | — | 编码专用 |
| **2026-07-16** | **K3 发布** | **2.8T / 104B 激活** | 全球最大开源权重模型 |
| 2026-07-27 | K3 开权重 + 47 页技术报告 | | 自定义许可 |

---

## 2. Kimi K3 技术要点

完整的 KDA 机制拆解见 [01-sparse-attention-comparison.md §2.2](../01-sparse-attention-comparison.md)。

### 核心主张：不是更大，是每单位算力更聪明

- 2.8T 总参数，每 token 仅激活 **104B**
- **Stable LatentMoE**：896 专家选 16
- 声称**单位算力的智能提升约 2.5×**（相对 Kimi K2）
- 原生视觉 + 1M 上下文

### 两项架构创新

**KDA（Kimi Delta Attention）** — Gated DeltaNet 的细粒度门控扩展：

```
S_t = (I − β_t·k_t·k_tᵀ) · Diag(α_t) · S_{t−1} + β_t·k_t·v_tᵀ
```

- `(I − β·k·kᵀ)`：类 Householder 变换，**写入新值前先擦除该 key 上的旧内容**（避免朴素线性注意力的记忆污染）
- `Diag(α_t)`：**通道级遗忘门**——相对 Gated DeltaNet 的单标量 α，KDA 每个通道独立控制遗忘，可保留某些特征维度而激进遗忘另一些
- 已知局限：delta 编辑强度仍由**单标量 β** 控制，擦除量与写入量未解耦
- 硬件：特化的 DPLR 变体，比通用 DPLR **快近 2×**；kernel 以 **FlashKDA** 开源

**AttnRes（Attention Residuals）**：不再逐层均匀累积表示，改为**跨深度选择性检索**相关表示。

**层配比**：93 层中 **69 层 KDA : 24 层 Gated MLA**（≈2.9:1）。
必须保留全注意力层的原因：线性注意力的固定状态是有损压缩，**无法做精确全局检索**。

**一个反直觉发现**（来自 Kimi Linear 论文）：与 KDA 层集成时 **NoPE 优于 RoPE**——
位置感知可从门控 delta 递推本身涌现。

### 配套开源

高性能注意力 kernel（FlashKDA）、MoE 通信库、大规模 agent 环境基础设施。

---

## 3. 能力与第三方评价

**这是四家里自报成分最少的**——主要宣称都来自第三方榜单。

| 榜单 | 结果 |
|---|---|
| Artificial Analysis v4.3 | **44 分，开源并列第一**（与 GLM-5.3）；前沿 Fable 5.1 / GPT-6 Astra 为 53 |
| Vals AI | **开源第一** |
| Vals Terminal-Bench 2.1 | **80.9%，开源第一**（高于 Claude Fable 5 的 80.52%） |
| Vals SWE-bench Verified | 93.4%，总榜第三（GPT-5.6 Sol 96.2 > Fable 5 95.0 > K3） |
| Vals 编码类目 | 60.6，**低于 GLM-5.3 的 64.3** |
| WebDev Arena | **99 个模型中第 1，1678 Elo——首个登顶该榜的开源模型** |
| DeepSWE（自报） | 67.3，已注明 mini-SWE-agent harness |

**需打折的一项**：「单位算力智能 2.5×」基于 Kimi Linear 论文在 **48B/3B 规模**的实验，
**未在 2.8T 的 K3 规模上由第三方验证**。

### 标志性能力演示

- 长程编码：大仓库导航、终端工具编排、GPU kernel 优化、编译器开发、视觉在环游戏开发、CAD、芯片设计
- **K3 Swarm Max**：行业报告/文献综述类任务扇出到大量并行子 agent
- **48 小时自主芯片设计**：用开源 EDA 工具，走完架构设计 → 优化 → 验证全流程，
  设计一颗能运行自身纳米版的物理芯片

---

## 4. ⚠️ 风险

### 4.1 蒸馏指控（客观时间线梳理）

这件事有三个**不同**的事件被媒体混为一谈，需要分开看：

| 时间 | 事件 | 证据状态 |
|---|---|---|
| **2026-02** | Anthropic 指控 DeepSeek、Moonshot、MiniMax「蒸馏攻击」：约 **24000 个假账号**、**1600 万次** Claude 对话，用商业代理绕过中国访问限制；**其中 340 万次追溯到 Moonshot** | ✅ **有具体记录**，但**早于 K3**，是另一个时期的事 |
| **2026-07-22** | 白宫 OSTP 主任 Michael Kratsios 指控 Moonshot「开发了复杂的内部平台对美国模型进行大规模蒸馏」，可切换访问方式规避检测；称其使用 GB300 服务器（新购或经泰国转运）。**财政部威胁制裁** | ❌ **未公开任何证据**。财长 Bessent 提到在中国模型上发现美国模型的「水印」，但未说明水印是什么 |
| **2026-09-10** | Anthropic 发布威胁情报报告，指控 Moonshot **把部分 Kimi 用户请求静默转发给 Claude，再把 Claude 的回答当作 Kimi 的输出展示** | ❌ 遥测与归因**未经公开独立审计**。这是产品级路由指控，性质比训练蒸馏更严重 |

**专家质疑**：Snorkel AI 的 Braden Hancock、分析师 Nathan Lambert 等认为时间线不成立——
**Anthropic 的 Fable 公开仅 15 天 K3 就发布了**。有专家向 TechCrunch 表示，
这么强的模型在 Fable 发布这么短时间后出现，不可能主要来自蒸馏；
另有观点指出该规模的 API 蒸馏成本极高且受模型速度瓶颈限制。

**各方回应**：Moonshot 否认两项指控（但**未直接回应白宫关于蒸馏 Fable 的说法**）；
中国商务部回应称华盛顿立场是「AI 霸权主义」，警告将采取「一切必要措施」。

**结论**：公开证据不足以支持 K3 蒸馏自 Fable 的指控。
但**制裁风险本身是真实的**——采购时应把供应连续性风险单独计入，
不要因为指控未证实就忽略尾部风险。

### 4.2 许可不是真正的开源 🚩

- 年收入 **>2000 万美元**的公司若要把 K3 作为服务提供给外部客户，**须先与 Moonshot 谈合同**
- 月收入 >2000 万美元或 **MAU >1 亿**的产品须显著署名

→ 大中型企业对外提供 K3 服务**需要走法务流程**。

### 4.3 价格最贵

$3.00 输入 / $15.00 输出（每百万 token），cache read $0.30。
在重缓存的 agent 负载下，**cache read 比 DeepSeek V4.1-Flash 贵约 100 倍**。

### 4.4 工程注意事项

K3 被训练为**跨会话保留推理历史**。如果 agent 脚手架未正确回传历史，
或会话中途从其他模型切换到 K3，**输出质量会变得不稳定**。

---

## 5. 商业

| 项 | 数据 |
|---|---|
| 成立 | 2023-03，北京；创始人杨植麟、周昕宇、吴育昕 |
| 主要投资方 | 阿里巴巴、腾讯 |
| 估值轨迹 | 年初约 33 亿 → 5 月 200 亿 → **7 月 350 亿美元** |
| 融资 | 1 月 C 轮 5 亿；7 月再融 35 亿；年内累计至少 **80 亿美元** |
| **IPO** | **2026-09-03 报道秘密递交港股 IPO**，拟融 **30 亿美元**、估值 **500 亿**；承销商高盛 / 中金 / 德意志银行 |
| 收入 | **ARR 超 3 亿美元**（四家中最高） |

估值跃升的直接催化剂是 K3——一个 2.8T 模型，在多项基准登顶并缩小了与美国前沿模型的差距。

---

## 6. 值得继续跟的

1. **蒸馏指控的证据是否会公开**——若财政部实施制裁，将直接影响全球可用性
2. **港股 IPO 进程**，以及 500 亿估值 vs 3 亿 ARR 的落差如何向投资者解释
3. **KDA 的单标量 β 局限能否解掉**（擦除与写入强度解耦）
4. **K3 之后是否继续开源**，以及许可门槛会否进一步收紧

---

## 一手资料

- [arXiv:2510.26692 — Kimi Linear（KDA 原始论文）](https://arxiv.org/abs/2510.26692)
- [GitHub — MoonshotAI/Kimi-K3](https://github.com/MoonshotAI/Kimi-K3)
- [GitHub — MoonshotAI/Kimi-Linear（FlashKDA）](https://github.com/MoonshotAI/Kimi-Linear)
- [Doubleword — 从 softmax 推导到 KDA（含 Triton kernel）](https://blog.doubleword.ai/you-could-have-come-up-with-kimi-delta-attention)
- [VentureBeat — 最大开源模型发布](https://venturebeat.com/technology/chinas-moonshot-ai-releases-kimi-k3-the-largest-open-source-model-ever-rivaling-top-u-s-systems)
- [CNBC — K3 对标 OpenAI / Anthropic](https://www.cnbc.com/2026/07/17/moonshot-ai-kimi-k3-model-openai-anthropic-china.html)
- [Fortune — K3 推进中国 AI 至 Fable 级](https://fortune.com/2026/07/16/moonshots-kimi-k3-pushes-chinese-ai-into-fable-level-territory/)
- [TechCrunch — 专家认为蒸馏说不成立](https://techcrunch.com/2026/07/23/experts-say-exploiting-anthropics-fable-isnt-how-kimi-k3-got-so-good/)
- [CyberScoop — 白宫指控](https://cyberscoop.com/white-house-accuses-moonshot-ai-anthropic-model-distillation/)
- [Dealroom — Anthropic 指控请求路由](https://dealroom.co/news/150366-anthropic-alleges-moonshot-routed-some-kimi-user-requests-to-claude-then/)
- [Tech Startups — 港股 IPO 递表](https://techstartups.com/2026/09/03/chinese-ai-startup-moonshot-files-for-3-billion-hong-kong-ipo-at-50-billion-valuation/)
