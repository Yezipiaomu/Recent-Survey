# OpenAI Codex 档案

> 调研日期：2026-09-16 · [返回索引](../README.md)
> 定位：**多入口 + 内核级沙箱 + token 效率**。它的差异化在「面」而不在单点能力——
> 五个surface 共用一套 App Server，加上云端按任务起容器，工作流是"派活 → 去干别的 → 回来 review diff"。

---

## 1. 基本情况

| 项 | 值 |
|---|---|
| 厂商 | OpenAI |
| 开源 | CLI 部分开源（**Rust** 编写） |
| 默认模型 | **GPT-6-Astra**（2026-09-03 起为推荐默认）；GPT-5.6 "Sol" 为无权限账户的 fallback |
| 项目指令文件 | **`AGENTS.md`**（就近优先：离工作目录越近优先级越高） |
| 配置 | `config.toml` |
| 参考版本 | v0.153.4（2026-09-04） |

---

## 2. 五个 surface，一套内核

> **五个 surface 共享同一套底层智能：CLI、桌面应用、IDE 扩展、云任务、浏览器扩展。**

2026-02 起 OpenAI 用单一 **App Server** 架构同时驱动 CLI、VS Code 扩展、Web 应用、
macOS 桌面应用，以及 JetBrains、Xcode 等第三方 IDE 集成。

**这是它与 Claude Code 最本质的产品差异**：Claude Code 是"和你一起坐着改"，
Codex 是"派活、异步跑、回来 review"。

---

## 3. 沙箱与审批（最强项）

### 3.1 三级审批

| 模式 | 行为 |
|---|---|
| **Suggest** | 提出改动，未经批准不确认任何事 |
| **Auto Edit** | 自动写文件，但执行 shell 命令前询问 |
| **Full Auto** | 整个循环不间断运行，**范围限定在当前目录** |

### 3.2 沙箱层级

> CLI 模式下沙箱在本地：**OS 级限制把文件写入限定在工作区，并默认禁用网络。**
> **Codex 在操作系统内核层执行限制，模型无法绕过。**

对比 Claude Code 的应用层策略 —— 这是「敢不敢开 Full Auto」的分水岭。

### 3.3 权限 profile

v0.133.0（2026-05-21）起：**goals 默认开启，权限 profile 成为一等管理面。**

---

## 4. Subagents 与云端

### 4.1 Subagents

- 可要求 Codex 把工作的独立部分委派给 subagents
- 当前本地版本在**你直接要求时**，或 **AGENTS.md / skill 指令要求时**委派
- **Subagents 继承你当前的沙箱策略**和 composer 下方选择的权限模式
- 有 background-agent 面板：查看状态、停止活动 subagent、打开 subagent 线程

### 4.2 云沙箱

> 按任务**provision 一个预装你仓库的隔离容器**；agent 在其中跑命令与测试，
> 返回 **diff、日志和产物**；**任务并行运行，不阻塞你的编辑器**。

⚠️ **Codex Cloud 与 code mode 仍标记为实验性**；
而 **核心 CLI、沙箱、AGENTS.md、config.toml、Skills、hooks、多 agent 工具和 plugins 已属稳定**。

---

## 5. Hooks（纠正一条流传说法）

> **「Codex 没有 hooks」的说法已经过时。**

现状：

- 可用 `/hooks` 浏览与接线
- 支持 **AfterAgent / AfterToolUse**
- 结合权限配置与沙箱做**执行前控制**

客观表述：**两者都有可编程治理 hooks——Claude Code 的更广更成熟，
Codex 的则搭配了同类中最强的沙箱。**

---

## 6. 按六职责定位

| 职责 | 实现 | 评价 |
|---|---|---|
| **Observation** | 文件 + 命令 + 云端仓库快照 | 云端是差异点 |
| **Context** | — | ✅ **token 效率最优**，同任务比 Claude Code 少 2–4 倍 |
| **Control** | subagents（继承沙箱策略）+ background 面板 | 单层但有可视化管控 |
| **Action** | MCP + 内置工具 | 成熟 |
| **State** | AGENTS.md + 云任务产物 | AGENTS.md 是跨工具可移植的 |
| **Verification** | AfterAgent/AfterToolUse hooks + **官方 GitHub Action**（`codex exec` 带沙箱控制进 CI） | CI 集成是强项 |

---

## 7. 生态

| 项 | 内容 |
|---|---|
| 分发优势 | **随 ChatGPT 捆绑**——每个订阅者无需额外设置即可使用 |
| CI | 官方 GitHub Action 跑 `codex exec`，带沙箱控制 |
| 反向集成 | 官方插件可**在 Claude Code 里调用 Codex** 做代码审查与任务委派 |
| ACP | 社区 `codex-acp`（Rust adapter）桥接到任意 ACP 客户端；Codex CLI 在 ACP Registry 中 |
| 生态规模 | 社区收录 150+ 工具 / skills / subagents / plugins |

---

## 8. 商业与安全

### 计费

**2026-04 起 Business 与 Enterprise 计划转为基于 token 的信用消耗。**

### 安全事件

> 2026-03，研究人员发现一个**已修复**的漏洞：
> **恶意的 GitHub 分支名可在任务设置阶段注入命令，并窃取 GitHub 认证 token。**

→ 与 [../04-openclaw-security.md](../04-openclaw-security.md) 同源：
**间接提示注入不挑产品，只要 agent 会读取外部内容就存在。**

---

## 9. 适合谁

| 适合 | 不适合 |
|---|---|
| 要把任务派出去、异步 review diff | 想和 AI 一起逐步推演复杂重构 |
| 需要强制性执行边界（内核级沙箱） | 需要跨模型厂商切换 |
| 重度 CI/自动化 | — |
| 在意 token 账单 | — |
| 团队用多种 IDE（AGENTS.md 可移植） | — |
