# Claude Code 档案

> 调研日期：2026-09-16 · [返回索引](../README.md)
> 定位：**推理深度优先的垂直整合 harness**。扩展体系最完整（六类原语），
> 但 token 消耗是 Codex 的 3–4 倍，且沙箱在应用层而非内核层。

> ⚠️ 本档案由 Claude 撰写，涉及 Anthropic 自家产品。不利事实已主动列出并加粗。

---

## 1. 基本情况

| 项 | 值 |
|---|---|
| 厂商 | Anthropic |
| 开源 | ❌ 闭源 |
| 运行位置 | **本机运行** |
| 模型耦合 | 绑 Claude 家族 |
| 项目指令文件 | `CLAUDE.md` |
| 运行形态 | CLI（终端）/ 桌面应用（Mac、Windows）/ Web（claude.ai/code）/ IDE 扩展（VS Code、JetBrains） |
| 协议 | MCP 原生；在 ACP Agent Registry 中 |

---

## 2. 分层扩展体系（六类原语）

Claude Code 已从单纯的终端助手演进为**分层 agentic 系统**，
底层把 memory、hooks、skills、subagents、plugins 和 MCP 分离为不同层级，
**每一层改变模型能看到什么或能做什么**。

```
Harness（运行时）
      │
      └── Main Agent
            ├── Skills        同窗口内按需加载的指令包
            ├── Subagents     独立 context window，单向回传摘要
            └── Agent Teams   独立进程，双向通信

      Plugins / Hooks  ←── 横跨所有层级
```

| 原语 | 作用 | 关键机制 |
|---|---|---|
| **CLAUDE.md** | 项目级常驻记忆 | 每次会话加载 |
| **Skills** | 基于文件夹的指令包 | 目录含 `SKILL.md` + 可选脚本/资源；**每次会话只读文件夹名和描述，任务匹配时才拉入正文** |
| **Subagents** | 隔离 worker | 在自己的 context window 中运行，**只返回摘要**，内部过程不污染主会话 |
| **Agent Teams** | 多 agent 协作 | 每个成员是**独立的 Claude Code 进程**，双向通信 |
| **Hooks** | 确定性控制点 | 生命周期事件触发，**20+ 事件** |
| **Plugins** | 打包分发 | 把 skills / subagents / hooks / output styles 打成一个可安装单元 |
| **MCP servers** | 外部工具 | 见 [../01-protocol-layer.md](../01-protocol-layer.md) |

### 2026 年的变化

- 新增**内置 skills**：`/doctor`、`/code-review`、`/verify`、`/run`
- **自定义 commands 已并入 Skills**
- 推出 **Agent Skills 开放标准**

### 常见 hooks 用法

| Hook | 用途 |
|---|---|
| `PreToolUse` | 阻止仓库外写入 |
| `PostToolUse` | 编辑后跑 prettier |
| `Stop` | 结束时跑测试套件 |
| shell 命令时 | 脱敏密钥 |

---

## 3. 选型误区：在错误的层级用了正确的工具

> 问题根源通常是"在错误的层级用了正确的工具"：
> - **CLAUDE.md 不断膨胀成流程文档**，会拖累每次会话
> - **本该用 skill 的任务却派生 subagent**，只会带来不必要的开销和无谓的上下文隔离

判断标准：

| 需求 | 用 |
|---|---|
| 一段"遇到 X 就这么做"的指令 | **Skill**（按需加载，不占常驻窗口） |
| 一段大量读取但只需要结论的工作 | **Subagent**（隔离 context，回传摘要） |
| 确定性的、必须发生的动作 | **Hook**（不依赖模型判断） |
| 团队/项目间共享一致配置 | **Plugin** |
| 几个独立方向并行推进 | **Agent Teams**（⚠️ plan mode 下约 **7 倍** token） |

---

## 4. 按六职责定位

| 职责 | 实现 | 评价 |
|---|---|---|
| **Observation** | 文件 + 命令 | 常规 |
| **Context** | 全盘累积为主，**自动启用缓存** | ⚠️ **最大短板**，见下 |
| **Control** | Skills / Subagents / Agent Teams **三层** | ✅ 最完整 |
| **Action** | MCP + 内置工具 | 成熟 |
| **State** | CLAUDE.md + 会话 | 常规 |
| **Verification** | `/verify`、`/code-review`、Stop hook 跑测试 | 相对领先但仍非一等公民 |

---

## 5. 不利事实（主动列出）

### 5.1 token 消耗

| 对比 | 数据 |
|---|---|
| vs Codex 每任务 token | **Claude Code 约为 Codex 的 3–4 倍** |
| 中文实测最耗组合 | **Claude Code + GLM-5.1 = 4.8M/任务**，是 Cursor CLI + Opus 4.7（1.5M）的 3 倍多 |
| 外部模型横评（Kimi K3） | **Claude Code 花钱最多** |
| 但分数 | Claude Code (Opus 4.7) = **60**，Cursor CLI + Opus 4.7 = **61** |

→ **3 倍的成本换来 -1 分。** 在外部模型配置下尤其不划算。

⚠️ 公允补充：Claude Code 在自家模型上自动启用缓存（同一 bugfix $0.54 有缓存 vs $1.35 无缓存），
外部模型横评的结论不能直接外推到 Claude 模型上。

### 5.2 沙箱层级

> **Claude Code 在应用层执行安全策略**（20+ hook 事件，需自行接线）；
> **Codex 在操作系统内核层执行，模型无法绕过限制。**

这直接决定「敢不敢开 Full Auto」。

### 5.3 闭源

无法审计、无法自托管、无法改。对合规要求高的场景是硬门槛。

### 5.4 harness 效应对它不利的一面

> 同一个 Opus 模型：在 **Claude Code 得 77%**，在 **Cursor 得 93%** ——**差 16 个百分点**。

这说明 Claude Code 的 scaffold 并非在所有任务类型上都最优。

---

## 6. 优势面

- **推理深度**：适合深度重构、代码审查、"为什么改 / 怎么拆 / 哪里危险"的判断
- **可编程治理**：hooks 覆盖面最广、最成熟
- **编排层级最完整**：Skills / Subagents / Agent Teams 三层是目前独有的
- **Agent Skills 开放标准**：把扩展格式标准化，而非厂商私有

---

## 7. 实战建议（社区共识）

> 团队通常两者都用，但**别让它们抢同一块代码**：
> **Claude Code 负责判断"为什么改、怎么拆、哪里危险"；
> Codex 负责执行边界清晰的任务、检查、复审或自动化。**

另：**即使只用 Claude Code，也顺手写一份 `AGENTS.md`**，
20 分钟就能让项目对 Codex、Cursor、Copilot 用户开放。

---

## 8. 安全提醒

> **安装第三方资源前请先检查其权限、脚本、hooks 和服务连接——
> 很多 Claude Code 资源可以读文件、执行命令或向外部服务发送数据。**

参见 [../04-openclaw-security.md §6](../04-openclaw-security.md)。
