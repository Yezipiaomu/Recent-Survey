# OpenCode 档案

> 调研日期：2026-09-16 · [返回索引](../README.md)
> 定位：**唯一真正模型中立的生产级 harness**。MIT 协议、75+ provider、client-server 多前端，
> 且是唯一把 LSP 诊断喂回模型的。代价是慢（同任务约慢 78%）和配置成本。

---

## 1. 基本情况

| 项 | 值 |
|---|---|
| 主导 | 开源社区（SST） |
| 协议 | **MIT** |
| 规模 | **~150K GitHub stars** |
| 模型支持 | **75+ provider**，零厂商锁定 |
| 技术栈 | **Go**（TUI，Bubble Tea）+ **JavaScript/Bun** |
| 架构 | **client-server** |
| 商业模式 | 自托管，只付你选的模型 API |

---

## 2. 架构：client-server 带来多前端

```
                    ┌─────────────────┐
  Terminal TUI ────►│                 │
  Desktop App  ────►│  OpenCode       │────► 任意模型 provider（75+）
  VS Code Ext  ────►│  Server         │
  任意 HTTP客户端 ──►│                 │
                    └─────────────────┘
```

把 agent 做成服务而非进程，是它与其他三家最大的架构差异——
Claude Code / Codex 的多 surface 是厂商自己做的，OpenCode 的多前端是**任何人都能做的**。

---

## 3. 独有特性：LSP 诊断回灌

> **OpenCode 唯一把 LSP diagnostics 喂回给 AI。**

这是一个真实且被低估的工程差异：**模型能看到编译器/类型检查器怎么骂它**，
而不是靠自己猜或者靠跑测试才发现。

对应 [六职责框架](../agent-harness-2026-09.md) 的 **Observation** 一轴——
这是目前唯一一家在这一轴上做出实质差异化的。

同时它也部分补上了 **Verification** 这块洼地：诊断即验证信号。

---

## 4. 按六职责定位

| 职责 | 实现 | 评价 |
|---|---|---|
| **Observation** | 文件 + 命令 + **LSP 诊断** | ✅ **独有优势** |
| **Context** | — | 居中（外部模型横评中低于 Hermes、Grok Build、Claude Code、Kimi Code） |
| **Control** | 单层 | ⚠️ 无 subagent/team 编排 |
| **Action** | MCP | 成熟 |
| **State** | 自有配置 | 常规 |
| **Verification** | 靠 LSP 诊断闭环 | 间接但有效 |

---

## 5. 已知短板

| 问题 | 数据/说明 |
|---|---|
| **速度** | 同任务比 Claude Code **慢约 78%** |
| **本地模型 tool calling** | **时灵时不灵**（hit-or-miss） |
| **配置成本** | 明显高于订阅制产品；"灵活性变成了复杂性" |
| **自托管负担** | **沙箱、凭据和审计链路都要自己搭** |
| 编排能力 | 无 subagent / Agent Teams 级别的多 agent 编排 |

### 一个例外情况

> 除非你一开始就用免费模型——那样几分钟就能进 TUI，且不花钱。

---

## 6. 横评中的位置

多 harness 横评（统一用 Kimi K3 外部模型）：

| 维度 | 结果 |
|---|---|
| 工具调用次数 | 18 次 |
| 耗时 | 421 秒 |
| token 用量 | **低于** Hermes、Grok Build、Claude Code、Kimi Code |
| 综合 | **居中** |

对比同组：Pi Agent token 最少且最快；Hermes 每成功任务成本最低；Claude Code 花钱最多。

---

## 7. 适合谁

| 适合 | 不适合 |
|---|---|
| **不接受厂商锁定** | 要开箱即用、零配置 |
| 要自己接模型（含本地 Ollama/vLLM） | 对延迟敏感 |
| 需要审计源码 / 私有化 | 需要多 agent 编排 |
| 想自己做前端（HTTP API） | 不想自己搭沙箱和凭据管理 |
| 用免费模型零成本起步 | — |

---

## 8. 同类自托管方案

与 OpenCode 同属"自托管、只付模型 API"这一类的还有：
**Qwen Code、Goose、OpenHands**。它们共同的隐性成本是沙箱、凭据和审计链路自建。
