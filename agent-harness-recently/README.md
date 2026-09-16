# Agent Harness 调研：Claude Code / Codex / OpenCode / DeepSeek Harness / OpenClaw 及国产各家

> 调研日期：2026-09-16
> 范围：截至 2026 年 9 月中旬的编程型 harness、通用助理框架、办公任务型产品与协议层生态
> 姊妹调研：[../llm-recently](../llm-recently/README.md)（中国四大模型厂商） · [../gpt-claude-recently](../gpt-claude-recently/README.md)（OpenAI vs Anthropic）

> ⚠️ **利益相关声明**
> 本调研由 Claude（Anthropic 的模型）完成，且对比对象包含 Claude Code（Anthropic 自家产品）。
> 为尽量抵消偏向，本调研遵守三条规则：
> 1. 能力结论**优先采用标准化 harness 口径**（Scale SEAL、Terminus 2），厂商自有 scaffold 分数一律标注
> 2. 对 Claude Code 的不利事实（**token 消耗为 Codex 的 3–4 倍**、闭源、沙箱在应用层而非内核层）
>    与对竞品的负面事实**同等力度呈现**
> 3. 国产厂商部分的来源以自媒体与竞品方测评为主，**逐条标注可信度**，不替读者下结论

---

## 文档索引

| 文档 | 内容 | 适合谁看 |
|---|---|---|
| [agent-harness-2026-09.md](agent-harness-2026-09.md) | **总览**：四层分类、六职责框架、全量横向对比、五个判断 | 先看这个 |
| [01-protocol-layer.md](01-protocol-layer.md) | **协议层三分天下**：MCP / ACP / A2A 的边界与落地 | 关心互操作与长期押注 |
| [02-benchmark-credibility-audit.md](02-benchmark-credibility-audit.md) | **评测可信度审计**：为什么跨 harness 的分数不可比 | 要用分数做决策 |
| [03-cost-and-selection.md](03-cost-and-selection.md) | **成本与选型**：70 倍 token 差、缓存、并行代价、场景推荐 | 要选一个来用 |
| [04-openclaw-security.md](04-openclaw-security.md) | **专题：OpenClaw 安全**：CNCERT 预警、零点击外泄、供应链 | 考虑部署常驻 agent |
| [profiles/claude-code.md](profiles/claude-code.md) | Claude Code 档案 | |
| [profiles/codex-cli.md](profiles/codex-cli.md) | OpenAI Codex 档案 | |
| [profiles/opencode.md](profiles/opencode.md) | OpenCode 档案 | |
| [profiles/deepseek-harness.md](profiles/deepseek-harness.md) | DeepSeek Harness (dsh) 档案 | |
| [profiles/openclaw-and-forks.md](profiles/openclaw-and-forks.md) | OpenClaw 及国产分支（ArkClaw/CoPaw/QClaw/AutoClaw/DuClaw） | |
| [profiles/china-vendors.md](profiles/china-vendors.md) | 国产厂商全景：WorkBuddy、豆包、国产 CLI 四强 | |

---

## 五个核心结论

### 1. 这些东西不在同一个抽象层上，直接横评就是比错

| 层级 | 代表 | 本质 |
|---|---|---|
| **① 编程型 Harness** | Claude Code、Codex CLI、OpenCode、dsh | 本地「模型 + 工具循环」，读写工作区、跑命令、管计划 |
| **② 通用助理框架** | OpenClaw 及国产分支 | 常驻 + 主动触发 + IM 渠道，目标不是写代码 |
| **③ 办公任务型产品** | WorkBuddy、豆包任务模式、Kimi Work、QoderWork | 面向非技术用户的成品，底层可能就是 ①/② 换话术 |
| **④ 模型** | 豆包 Seed、DeepSeek-V4、混元 Hy3、GLM、Kimi | **不是框架** |

**「豆包」属于 ④ 不是框架**。火山官方口径：豆包是「交互型」（对话、多模态生成），
**ArkClaw 才是「执行型」**（自主跑任务、闭环）；ArkClaw 可一键把推理模型切成豆包。

### 2. Harness 比模型更决定账单——同一个模型，差 70 倍

> 同一模型在 Aider / Claude Code / OpenClaw 里跑，**token 用量相差 70 倍**。

根因是 Context 这一职责：主流 coding agent 仍在用**最朴素的「全盘累积」**——
每条工具返回都追加进历史、下一轮原封不动重喂，直到逼近上限才压缩。

- Claude Code 每任务 token **约为 Codex 的 3–4 倍**
- 最耗：Claude Code + GLM-5.1 = **4.8M/任务**；最省：Cursor CLI + Opus 4.7 = 1.5M
- 而分数：61 vs 60 vs 60 —— **头部已到毫厘之间，竞争早就转移到成本与生态**

详见 [03](03-cost-and-selection.md)。

### 3. 跨 harness 的 benchmark 基本不可比，比模型榜单的问题更严重

SWE-bench Pro 同时存在**三套都"真实"的数字**：

| 数字 | 口径 | 可否对等比较 |
|---|---|---|
| **59.1%**（GPT-5.4 xHigh） | Scale 标准化 public set | ✅ |
| **80.0 / 81.2%**（Claude Fable 5 / 5.1） | 厂商自有 scaffold | ❌ |
| **47.1%**（Claude Opus 4.6） | Scale 私有 commercial set | ✅（防污染） |

且 OpenAI 2026-07 审计估计 public split **约 30% 任务存在缺陷**。
Terminal-Bench 反而更诚实——**直接按「模型-agent 组合」报告**，榜上 101 个 agent 来自 **23 种 scaffold**。

> **铁律：引用 coding agent 分数必须同时标注 harness + split + 试验次数。**

详见 [02](02-benchmark-credibility-audit.md)。

### 4. 真正的结构性差异在协议层，而它已经三分天下

| 协议 | 管什么 | 主导 | 状态 |
|---|---|---|---|
| **MCP** | Agent ↔ 工具（纵向） | Anthropic → Linux Foundation | 事实标准 |
| **ACP** | 编辑器 ↔ 编码 Agent | Zed + JetBrains | 2026-01 Registry 上线 |
| **A2A** | Agent ↔ Agent 委派 | Google → Linux Foundation | 2026-04 v1.0，150+ 组织 |

ACP 的类比是 LSP。国产里**只有 Kimi CLI 完整支持 ACP**。详见 [01](01-protocol-layer.md)。

### 5. 常驻型 agent 的安全问题已经到了监管层面

**CNCERT 已对 OpenClaw 发出安全风险警告**，并促使中国限制在政府系统中使用。
具体事件包括零点击数据外泄（链接预览渲染即外传）、消息对象注入、ClawHavoc 协同攻击、
Moltbook 泄露 41 家企业邮件档案、11 天 3 个 CVE。
学术评估中其**提示注入鲁棒性仅 57%**，且**沙箱是 opt-in 的**。

详见 [04](04-openclaw-security.md)。

---

## 一句话选型

| 场景 | 选 |
|---|---|
| 复杂多文件重构、要和 AI 一起推演 | Claude Code |
| 派活异步跑、要并行 + 强审批边界 | Codex |
| 不接受厂商锁定、要自己接模型 | OpenCode |
| 想自己拼一个 Agent 而不是用别人的 | DeepSeek Harness（接受 preview 的坑） |
| 个人 24h 助理 | OpenClaw（极客）/ CoPaw（钉钉飞书）/ QClaw（微信）/ ArkClaw（云端懒人） |
| 数据不能出内网 | AutoClaw 本地 / dsh + 本地模型 / WorkBuddy 本地优先 |
| 非技术岗办公 | WorkBuddy |
| 要在 Zed/JetBrains 里换 Agent | 认准 ACP 支持（Kimi CLI 是国产唯一） |

---

## 时效性提醒

本领域的保质期**以周计而非以年计**：ArkClaw 2026-03 上线、dsh 2026-08 才发 v0.1、
Codex 在我上一版认知中"没有 hooks"而现在已经有了。
**任何超过 3 个月的横评结论都应视为失效。**
