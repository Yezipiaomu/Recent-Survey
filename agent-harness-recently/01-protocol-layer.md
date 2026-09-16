# 协议层三分天下：MCP / ACP / A2A

> 调研日期：2026-09-16 · [返回索引](README.md)
> 一句话结论：**选 harness 可能三个月就换，选协议是三年的押注。
> MCP 管工具、ACP 管编辑器、A2A 管 agent 间委派，三者互补不竞争；
> ACP 是其中最被低估的一个——它对编码 agent 的意义等同于 LSP 之于 IDE。**

---

## 0. 三条边界

```
                    ┌──────────────┐
      ACP           │              │        A2A
  编辑器 ↔ Agent    │    Agent     │   Agent ↔ Agent
  ───────────────►  │  (Harness)   │  ◄───────────────
   JSON-RPC/stdio   │              │   远程委派/artifact
                    └──────┬───────┘
                           │  MCP
                           ▼  Agent ↔ 工具（纵向，hub-and-spoke）
                    ┌──────────────┐
                    │  外部数据源   │
                    └──────────────┘
```

| 协议 | 全称 | 管什么 | 主导方 | 关键时间 |
|---|---|---|---|---|
| **MCP** | Model Context Protocol | Agent ↔ 工具 | Anthropic（2024-11）→ **2025-12 捐给 Linux Foundation 的 Agentic AI Foundation** | 事实标准 |
| **ACP** | **Agent Client Protocol** | 编辑器/IDE ↔ 编码 Agent | Zed Industries（+ JetBrains） | 2025-09 发布；**2026-01 Registry 上线** |
| **A2A** | Agent2Agent | Agent ↔ Agent 委派 | Google → Linux Foundation | **2026-04 v1.0**，150+ 组织 |

---

## 1. ACP：把 Agent 从编辑器里解耦出来

### 1.1 问题背景

在 ACP 之前，AI 编码 agent 与编辑器虽耦合紧密，但**互操作性并非默认**：

- 每个编辑器要为它想支持的**每个 agent** 构建定制集成
- 每个 agent 要实现**编辑器专有 API** 才能触达用户

→ M×N 的集成矩阵。

### 1.2 最好的类比是 LSP

> **LSP 在 2016 年把语言智能从 IDE 解耦；ACP 的目标是让开发者不换编辑器就能自由切换 Agent。**

这是理解 ACP 战略价值的唯一正确角度。

### 1.3 技术形态

- **开放的 JSON-RPC 契约**，把 Client（编辑器）与 Agent（AI 助手）解耦
  - **Client** 专注 UI 渲染与本地资源管控
  - **Agent** 专注 LLM 推理与任务编排
- **传输：JSON-RPC over stdio**
  - client spawn adapter 进程，通过 stdin/stdout 收发
  - 每条消息一行、`\n` 分隔
  - **不需要开端口、不走 HTTP，进程退出即断开**
- 建模了 initialize、会话建立、prompt turn、更新与取消等环节
- 复用 MCP 的 JSON 表示，同时新增面向编码 UX 的自定义类型（如**展示 diff**）
- 远程 agent 可部署云端，通过 HTTP/WebSocket 通信，但**完整远程支持仍在开发中**

### 1.4 与 MCP 的分工：互补，不是竞争

| | 规范什么 |
|---|---|
| **ACP** | Client ↔ Agent 的**交互边界**：任务下发、流式渲染、权限管控 |
| **MCP** | Agent ↔ 外部数据源的**工具边界** |

握手方式很优雅：**ACP initialize 阶段，Agent 通过 `mcpCapabilities` 声明支持的 MCP 传输类型；
Client 在创建会话时传入 MCP Server 配置，Agent 据此自行建立与外部工具的连接。**

### 1.5 生态落地

- **2026-01 起 ACP Agent Registry 已在 JetBrains IDE 和 Zed 中上线**
- Registry 中并列：**Codex CLI、Claude Code、Gemini CLI、GitHub Copilot CLI**
- 通过注册表安装，可让 IDE 用户获得**由同一 harness 驱动的图形界面**
- 社区维护的 `codex-acp`（Rust）把 Codex 运行时桥接到任意 ACP 兼容客户端
- **国产：Kimi CLI 完整支持 ACP**（Zed 可直接把它当 AI 后端），Deep Agents 也有 ACP 集成

### 1.6 ⚠️ ACP 是歧义缩写

| 缩写 | 全称 | 主导 | 内容 |
|---|---|---|---|
| ACP | **Agent Client Protocol** | Zed | 编辑器 ↔ 编码 agent，本文讨论的这个 |
| ACP | **Agent Communication Protocol** | IBM Research | 多 agent 通信，继承 FIPA-ACL，定义 agent 角色、消息类型、协商模式，支持 propose/accept/reject/counter 等 performative |

**检索与引用时必须区分。**

---

## 2. MCP：纵向的工具协议，以及它的结构性局限

MCP 是 agent-to-tool 的**纵向**协议，采用 **hub-and-spoke** 结构。

其结构性局限常被忽略：

- LLM 能看到所有工具，但**两个 MCP Server 之间无法通信**
- **没有任务委派的概念**
- **没有点对点能力协商的概念**

→ 这正是 A2A 存在的理由。

---

## 3. A2A：远程 agent 之间的委派层

- **2026-04 发布 v1.0**，超过 150 家组织支持
- Google 捐给 Linux Foundation，已集成进 AWS / Microsoft / Google 云平台
- 成为**企业场景中 agent 间通信的事实标准**

设计要点：**显式区分 message 与 artifact**，并主张结果应作为 **task artifact** 返回，
而不是塞回聊天消息。这对做工程化编排很关键——聊天消息是给人看的，artifact 是给系统用的。

---

## 4. 工程建议（来自 2026 Modern Agent Harness Blueprint）

一份广为流传的「2026 现代 agent harness 蓝图」把运行时、状态模型、上下文系统、工具层、
子 agent 编排、审批、协议与可观测性列为结构化要点。其协议选型结论：

1. **ACP 已经足够成熟，可作为默认的 IDE/编辑器适配目标**
   （Kimi CLI 支持，Deep Agents 有集成）
2. **即使暂时不对外暴露 ACP，内部事件总线也应设计成 ACP 风格**
3. **A2A 更适合远程 agent 间的委派与任务交换层**

---

## 5. 治理缺口（值得单独立项）

这是一个公认但尚未解决的问题：

> **A2A 的四个官方扩展示例（Secure Passport、Timestamp、Traceability、Agent Gateway Protocol）
> 均未涉及治理。**

已有专门研究以《Governance Gaps in Agent Interoperability Protocols:
What MCP, A2A, and ACP Cannot Express》为题探讨三者**无法表达**的内容。

结合 [04-openclaw-security.md](04-openclaw-security.md) 里的间接提示注入问题看，
**协议层没有表达"这段内容是数据不是指令"的能力**，是当前整个 agent 生态最底层的缺陷。

---

## 6. 对选型的直接影响

| 你的诉求 | 看哪个协议 |
|---|---|
| 想随时换 Agent 但不换编辑器 | **ACP**（Zed / JetBrains Registry） |
| 想让 Agent 接自己的内部系统 | **MCP** |
| 想做多 agent 编排 / 企业级委派 | **A2A** |
| 国产工具里想要 ACP | **只有 Kimi CLI** |

> 本文档的可信度：协议规范与状态来自官方站点与协议文档（✅）；
> 「蓝图」类建议来自社区 gist（⚠️ 个人观点，非标准）。
