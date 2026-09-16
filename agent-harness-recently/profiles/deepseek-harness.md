# DeepSeek Harness (dsh) 档案

> 调研日期：2026-09-16 · [返回索引](../README.md)
> 姊妹档案：[../../llm-recently/profiles/deepseek.md](../../llm-recently/profiles/deepseek.md)（模型侧）
> 定位：**这批里架构最激进的一个**。它不想做"又一个 Coding 客户端"，
> 而想做"组装 Agent 的通用方式"——一切皆插件。代价是仍处 Developer Preview。

---

## 1. 基本情况

| 项 | 值 |
|---|---|
| 厂商 | DeepSeek |
| 命令行名 | **`dsh`** |
| 发布 | **2026-08-13 晚**，v0.1 开发者预览版 |
| 协议 | **MIT** |
| 仓库 | `deepseek-ai/deepseek-harness` |
| 文档 | deepseek-harness.github.io |
| 配套模型 | 同日开源 **DeepSeek-V4-Pro-0813**（MoE，总参 **1.6T** / 激活 **490B**，1M 上下文，最大输出 **384K**） |
| 状态 | ⚠️ **仍标 Developer Preview**（截至 2026-08-18），版本升级出现不兼容属正常 |
| 负责人 | 崔添翼（Tianyi Cui），浙大计算机背景，曾在 Jane Street 任职约 9 年 |

> ⚠️ 模型参数与 [../../llm-recently/profiles/deepseek.md](../../llm-recently/profiles/deepseek.md) 记录的
> "V4-Pro GA 1.6T/49B" 存在出入（490B vs 49B），两处来源不一致，**以官方仓库 README 为准**。

---

## 2. 核心理念

### 2.1 Model + Harness = Agent

这与 [六职责综述](../agent-harness-2026-09.md) 的学术框架**完全同构**——
一个产品和一篇综述独立收敛到同一个公式，本身就说明这是 2026 年的共识。

官方定位对标 OpenAI Codex 与 Anthropic Claude Code，但强调：

> 它想做的**不是"一个固定的 Coding 客户端"，而是"组装 Agent 的通用方式"**。
> 不是再套一层聊天页面，而是一套能读写工作区、运行命令、维护计划、
> 调用工具并持续执行任务的 **Agent Harness**。

### 2.2 一切皆插件

> **模型适配器、工具注册、技能、会话、沙箱、存储、Agent 循环、任务调度、UI ——
> 所有能力均由插件组合而成，开发者无需改源码即可定制。**

对照六职责：**dsh 把六个职责全部插件化了**，而其他三家都是硬编码其中大部分。

---

## 3. 技术架构：Cordis 插件元框架

dsh 建立在 **Cordis** 插件元框架之上，理念源自北大与 DeepSeek 联署论文
《A Programming Paradigm for Spatiotemporal Composability》。

### 插件树的层叠顺序

```
①  官方组合包（打底）
      ↓
②  profile 的 patch 层
      ↓
③  机器级 patch
      ↓
④  命令行 --patch 覆盖
      ↓
   越靠后优先级越高
```

**与 Claude Code 的对比**：

| | Claude Code | dsh |
|---|---|---|
| 扩展单位 | 六类固定原语（Skills/Subagents/Hooks/Plugins/MCP/CLAUDE.md） | **统一的插件** |
| 组合方式 | 各原语各有机制 | **单一的 patch 层叠** |
| 能改的范围 | 模型看到什么、能做什么 | **连 Agent 循环和 UI 都能换** |
| 心智负担 | 要判断"用哪一层" | 要理解插件树 |

---

## 4. 运行方式

支持四种形态：**Web UI / TUI / Headless / 编程 API**。

### 最简上手

```bash
# 1. 申请 DeepSeek API Key，装好 Node.js
npx @deepseek-ai/dsh web
# → 默认 http://127.0.0.1:3080
```

### 从源码构建

```bash
git clone <repo>
pnpm install
pnpm run build
pnpm dsh web
```

---

## 5. 按六职责定位

| 职责 | 实现 |
|---|---|
| **Observation** | 插件决定 |
| **Context** | 插件决定 |
| **Control** | **插件树层叠 patch**（Agent 循环本身可替换） |
| **Action** | 工具注册即插件 |
| **State** | 存储插件 |
| **Verification** | 插件 |

→ **六项全为"插件决定"。** 这既是最大优势（完全可定制）也是最大风险
（**没有默认的最佳实践，质量取决于你自己拼得好不好**）。

---

## 6. 路线图

| 时间 | 计划 |
|---|---|
| 2026-Q3 | 插件市场、任务编排、代码补全 |
| 2026-Q4 | Agent 协作、**自进化插件**、视觉交互 |
| 2027-Q1 | 多模态编程、3D 环境交互 |

---

## 7. 风险与适用性

### 风险

- ⚠️ **Developer Preview**，破坏性升级是常态
- ⚠️ 现阶段**更适合开发者而非普通聊天用户**
- ⚠️ 插件开发需要 **TypeScript / JavaScript 基础**
- ⚠️ 迭代极快，任何第三方教程都可能很快过期——**以官方 README 和文档为准**
- ⚠️ 生态从零开始，没有 Claude Code/Codex 那样的现成 skills 库

### 适合谁

| 适合 | 不适合 |
|---|---|
| 想自己**拼一个 Agent** 而不是用别人的 | 要开箱即用 |
| 需要替换 Agent 循环本身 | 生产环境、稳定性优先 |
| 研究 harness 设计 | 不写 TS/JS |
| 要完全自主可控 + MIT | 依赖现成生态 |

---

## 8. 战略意义

它是**唯一一个由模型厂商主导、且把 harness 本身作为可组合基础设施来做**的项目。
DeepSeek 一贯的路数（见姊妹档案：架构创新被竞争对手和 NVIDIA 直接采用）是
**把底层做成公共品**——如果 Cordis 的插件范式被采纳，
影响会超出 dsh 这个产品本身。

值得跟踪的信号：**2026-Q3 的插件市场是否真的形成生态**。
