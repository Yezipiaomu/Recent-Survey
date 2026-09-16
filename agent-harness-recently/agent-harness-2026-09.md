# Agent Harness 全景对比（2026 年 9 月）

> 调研日期：2026-09-16 · [返回索引](README.md)
> 一句话结论：**「Agent = 模型 + Harness」已经成为学术与工程的共识框架；
> 而在头部模型分数打平（61/60/60）之后，harness 设计本身——尤其是 Context 与 Verification
> 这两个职责——才是剩下的全部差异，也是 70 倍成本差的来源。**

---

## 0. 分析框架：Harness 的六职责分解

arXiv 2606.20683《From Question Answering to Task Completion: A Survey on Agent System
and Harness Design》提出的模型—harness 视角，是目前最完整的分析骨架。

它的核心提问是：**瓶颈到底在 foundation model、在 execution harness、还是在二者的耦合？**

并把 harness 拆成**六个相互耦合的运行时职责**：

```
              ┌── Observation  怎么看环境（文件、诊断、外部信号）
              │
              ├── Context      什么进窗口、怎么压   ← 成本差 70 倍的根源
              │
  Harness ────┼── Control      循环与编排（子 agent、计划）
              │
              ├── Action       工具与执行（MCP 是共同底座）
              │
              ├── State        记忆与持久化
              │
              └── Verification 怎么知道做对了   ← 最被忽视，下一个分水岭
```

该综述的结论值得直接引用：

> agent 质量（成功率、效率、安全、泛化）**涌现自「模型能力 × 运行时基础设施 × 任务结构 × 评测设计」
> 的交互**，而不是"带了工具的模型"。

配套论文集：[Awesome-Agent-Engineering](https://github.com/ggjy/Awesome-Agent-Engineering)。

**本报告后续所有横向对比，均按这六轴展开。**

---

## 1. 先分层：四类东西不要混着比

用户口中的「大模型框架」实际横跨四个抽象层，直接横评会得出错误结论。

| 层级 | 代表 | 本质 | 交互范式 |
|---|---|---|---|
| **① 编程型 Harness** | Claude Code、Codex CLI、OpenCode、DeepSeek Harness | 本地「模型 + 工具循环」 | 你敲一句它动一下 |
| **② 通用助理框架** | OpenClaw、ArkClaw、CoPaw、QClaw、AutoClaw、DuClaw、Linclaw | 常驻进程 + 主动触发 + IM 渠道 | **不等你说话就干活** |
| **③ 办公任务型产品** | WorkBuddy、豆包任务模式、Kimi Work、QoderWork、TRAE Work、Marvis | 面向非技术用户的成品 | 说人话的封装 |
| **④ 模型** | 豆包 Seed、DeepSeek-V4、混元 Hy3、GLM-5.x、Kimi-K2.7 | **不是框架** | — |

### 「豆包」的定位必须澄清

火山引擎官方自己给了明确对比：

| | 定位 | 能力 |
|---|---|---|
| **豆包大模型** | **交互型** | 对话、多模态内容生成、代码辅助、知识问答；需结合外部工具才能落地任务 |
| **ArkClaw** | **执行型** | 自主任务执行、事务闭环、定时任务、飞书/钉钉机器人、TOS 文件快传 |

二者是**协同不是竞争**：ArkClaw 依托火山方舟可无缝集成豆包 API 作为推理引擎，
在管理界面一键切换推理模型，实现「思考 + 执行」双能力。

---

## 2. ①类：四个编程 Harness 的真实分歧

### 2.1 总表

| | **Claude Code** | **Codex CLI** | **OpenCode** | **DeepSeek Harness (dsh)** |
|---|---|---|---|---|
| 厂商 | Anthropic | OpenAI | 开源社区 | DeepSeek |
| 设计哲学 | 薄壳 + 强模型垂直整合 | 多入口统一 App Server | 模型中立 + 开源 | **一切皆插件** |
| 模型耦合 | 绑 Claude 家族 | 绑 GPT 系（默认 GPT-6-Astra，2026-09-03 起） | **75+ provider** | 主推 DeepSeek-V4-Pro，适配器亦为插件 |
| 项目指令文件 | `CLAUDE.md` | `AGENTS.md`（就近优先） | 自有配置 | profile / patch 层叠 |
| 扩展机制 | Skills / Subagents / Agent Teams / Hooks / Plugins / MCP | Skills / hooks / subagents / plugins / MCP | MCP + **LSP 诊断回灌** | Cordis 插件树 |
| 沙箱层级 | **应用层**策略（20+ hook 事件，需自行接线） | **操作系统内核层**（模型无法绕过），默认断网 | 依赖配置，自托管 | 沙箱本身是插件 |
| 审批模型 | 权限模式（默认/接受编辑/plan/bypass） | 三级：Suggest / Auto Edit / Full Auto | 配置 | 插件 |
| 运行形态 | CLI / 桌面 / Web / IDE | **CLI / 桌面 / IDE / 云任务 / 浏览器插件**（五面共内核） | **client-server**：Go(TUI) + JS/Bun，可接任意 HTTP 前端 | Web UI / TUI / Headless / 编程 API |
| 云端异步 | 有（Claude Code on web） | **最强**：按任务起容器、并行跑、回 diff+日志 | 无原生 | Headless 可接 CI |
| 开源 | ❌ | CLI 部分开源（Rust） | ✅ **MIT，~150K star** | ✅ **MIT** |
| 成熟度 | 生产级 | 生产级（Cloud/code mode 仍实验） | 生产级 | **Developer Preview** |

### 2.2 按六职责看差异

| 职责 | Claude Code | Codex | OpenCode | dsh |
|---|---|---|---|---|
| **Observation** | 文件+命令 | 文件+命令+云端仓库快照 | **唯一把 LSP 诊断喂回模型** | 插件决定 |
| **Context** | 全盘累积为主，自动缓存 | **token 效率最优**（同任务少 2–4 倍） | 居中 | 插件决定 |
| **Control** | Skills / Subagents / Agent Teams **三层** | subagents（继承当前沙箱策略与权限模式） | 单层 | **插件树层叠 patch** |
| **Action** | MCP + 内置工具 | MCP + 内置工具 | MCP | 工具注册本身是插件 |
| **State** | CLAUDE.md + 会话 | AGENTS.md + 云任务产物 | 配置 | 存储插件 |
| **Verification** | 内置 `/verify`、`/code-review`、Stop hook 跑测试 | AfterAgent/AfterToolUse hooks + CI Action | 靠 LSP 诊断闭环 | 插件 |

### 2.3 三个值得单独拎出来的差异点

**① Codex 的差异化在「面」而不在能力。**
2026-02 起 OpenAI 用单一 App Server 架构同时驱动 CLI、VS Code 扩展、Web、macOS 桌面
以及 JetBrains/Xcode 第三方集成。加上云沙箱按任务起容器、并行跑不阻塞编辑器，
它的工作流是「派活 → 去干别的 → 回来 review diff」。
**Claude Code 更像「和你一起坐着改」。**

**② 沙箱层级的差异是决定性的。**
Claude Code 在**应用层**执行安全策略；Codex 在**操作系统内核层**执行，模型无法绕过限制。
这直接决定「敢不敢开 Full Auto」。
（注：**「Codex 没有 hooks」的说法已过时**——Codex 现已有 `/hooks`、AfterAgent/AfterToolUse。
客观表述是：两者都有可编程治理 hooks，**Claude Code 的更广更成熟，Codex 的搭配同类最强沙箱**。）

**③ dsh 是这批里架构最激进的。**
它建立在 Cordis 插件元框架上（理念源自北大与 DeepSeek 联署论文
《A Programming Paradigm for Spatiotemporal Composability》），主张：

> **Model + Harness = Agent**，且**一切皆插件**——模型适配器、工具注册、技能、会话、
> 沙箱、存储、Agent 循环、任务调度、UI 全部由插件组合，不改源码即可定制。

启动时按序叠加插件树：官方组合包 → profile patch → 机器级 patch → `--patch`，越靠后优先级越高。
它想做的**不是"又一个 Coding 客户端"，而是"组装 Agent 的通用方式"**。
代价：2026-08-13 才发 v0.1，至今标 Developer Preview，破坏性升级是常态。

---

## 3. ②类：OpenClaw 及其国产分支

**OpenClaw 与上面四个不是同一物种。** 关键差异有三个：

1. **常驻 + 主动**：中央 Gateway 进程单端口多路复用 WS/HTTP，靠 **heartbeat 心跳守护**
   定时醒来巡查本地文件与远程数据库——**不等你说话就干活**。
2. **入口是 IM 不是终端**：WhatsApp / Telegram / 微信 / 飞书 + macOS/iOS/Android 语音。
3. **记忆是本地的**：不用云向量库，而是**本地 Markdown 文件 + SQLite FTS5 全文索引**两层结构。

架构分四层：access → routing → business → storage。26 个内置工具，MCP 原生，
技能通过 **ClawHub** 分发（「AI Agent 的 npm」）。

**代价是安全**——见 [04-openclaw-security.md](04-openclaw-security.md)。

### 国产分支速览

| 分支 | 厂商 | 核心取向 | 可信度 |
|---|---|---|---|
| **ArkClaw** | 字节火山引擎（2026-03-09） | **云上 SaaS 版 OpenClaw**，零配置；定时任务、飞书/钉钉 Bot、TOS 快传；关机任务继续跑；不暴露公网端口 | 有官方页面 |
| **CoPaw** | 阿里通义（基于 AgentScope） | 本地+云双部署、一键脚本+国内镜像；原生钉钉/飞书/QQ/Discord/iMessage；**ReMe 长期记忆** | 自媒体为主 |
| **QClaw** | 腾讯电脑管家 | **唯一真·基于 OpenClaw 的腾讯产品**；只走微信 | 自媒体为主 |
| **AutoClaw** | 智谱（代号"澳龙"） | 纯本地、隐私最好、50+ 预装技能、**按积分计费**；易用性与文件操作是短板 | 自媒体为主 |
| **DuClaw** | 百度智能云 | 官网仍跳错误页，预发布状态，**建议观望** | 存疑 |
| **Linclaw** | 七牛云 | — | 竞品方自评，慎引 |

> ⚠️ 以上横评几乎全部来自自媒体或竞品方（实在智能、七牛云等），**倾向性明显**。
> 唯一相对可靠的共性判断是：**工具调用能力（尤其浏览器控制与第三方服务集成）国产普遍落后原版**；
> 反向地，原版只支持本地部署、依赖 Node+Docker、国内 IM 需第三方适配。

---

## 4. ③类：办公任务型产品——以及一处必须纠正的流传说法

### 4.1 「所有龙虾都是 OpenClaw 套壳」是错的

> **只有 QClaw 基于 OpenClaw 开源框架构建。WorkBuddy 是基于腾讯自研的 CodeBuddy 智能体架构，
> 底层代码与 OpenClaw 无关**；它被叫"龙虾"只是因为**兼容 OpenClaw 的技能包和 MCP 协议**，
> 目的是复用社区已有资源、降低迁移成本。

腾讯的完整布局是**四产品共用一套四层解耦架构**：

```
Layer 4 应用层
  WorkBuddy(办公) │ CodeBuddy(编程) │ Marvis(系统管家) │ OPC Platform(全栈自动化)
                        ↓  共享同一底座  ↓
        CodeBuddy 沉淀两年多的 Coding Agent 内核（Harness 层）
```

WorkBuddy 自身 = **专家 / 技能 / 插件**三模块 + **「本地优先」架构**
（自动化操作本地执行、文件数据不上云）；内置混元 Hy3 / DeepSeek-V4 / GLM-5.2 / Kimi-K2.7-Code
等 11 种国产模型可切换；支持企微/QQ/飞书/钉钉**远程控制办公电脑**。

商业数据：2026-06 月访问量 2097 万（为第二三名之和），月活 2000 万 / 日活 1300+ 万；
马化腾在财报会上称其为"中国使用最广的效率智能体服务"。
起源颇草根——源自一个十余人的 AI 代码助手团队，2026-01 中旬一个周末熬两个通宵做出 0.01 版。

> ⚠️ 上述架构描述来自开发者社区文章与媒体分析，**并非腾讯官方架构白皮书**。
> 已知内部隐忧：WorkBuddy 自称"Agent 时代的操作系统""工作生活总入口"，
> 与企业微信的办公入口、腾讯文档的文档入口重叠，而三者分属不同事业群。

### 4.2 同赛道格局

2026 年以来密集上线：阿里 QoderWork、腾讯 WorkBuddy / Marvis、Kimi Work、
字节 TRAE Work、豆包任务模式。策略分野明显：
**腾讯靠办公生态整合，阿里从编程工具转型，Kimi 把长文本优势延伸到任务执行。**

行业背景：IDC 数据显示中国企业级 AI 智能体市场 2025 年 212 亿元，2026 年预计 449 亿元，
相关服务商已突破 300 家。

---

## 5. 国产编程 CLI 四强

| 工具 | 厂商 | 开源 | 差异点 |
|---|---|---|---|
| **Kimi CLI** | 月之暗面 | ✅ | **完整支持 ACP**（Zed 可直接当 AI 后端）+ MCP 深度集成；定位 **bash+AI+IDE 三位一体**，`Ctrl+K` 切 Shell/AI 模式；运行时已从 Python 迁至 Node.js（旧教程 `uv tool install kimi-cli` 已非推荐） |
| **Qwen Code** | 阿里 | ✅ | Qwen-Coder 底座，中文场景优化 |
| **iFlow CLI** | 心流（阿里系） | ❌ | 免费接多模型（Kimi K2 / Qwen3-Coder / DeepSeek）；SubAgents + MCP 一键装；`/init` 扫库生成文档；YOLO 模式；**iflow-cli-action** 可进 GitHub Actions |
| **CodeBuddy CLI** | 腾讯 | ❌ | `generate` / `deploy` / `architect` 命令式子功能；混元+DeepSeek 双模型；微信小程序开发 |

IDE 侧「御三家」为字节 Trae 国内版、阿里 Qoder、腾讯 CodeBuddy，功能接近，
共同优势是**国内直连无障碍**。其中 Qoder 的 **Repo Wiki**（自动生成项目知识图谱）
对理解大型代码库有实际价值；Trae 国内版核心 AI 编程功能**免费无限制**（需排队），
主打 SOLO 智能体模式与 Builder 模式。

---

## 6. 五个判断

### 判断 1：模型分数已经区分不出 harness

Cursor CLI + Opus 4.7 = **61**，Codex (GPT-5.5) = **60**，Claude Code (Opus 4.7) = **60**。
继续卷性能分数的边际意义快速递减，**竞争已转移到成本、速度和生态**。

### 判断 2：Context 是当前最大的工程洼地

主流产品仍在用**全盘累积**：每次查询、每份文件、每条工具返回都追加，下一轮原封不动重喂。
这是 70 倍成本差的直接来源，也意味着**谁先做对上下文压缩，谁就拿走 40–70% 的成本优势**。

### 判断 3：Verification 是下一个分水岭

六职责里唯一还没有明确赢家的。Claude Code 有 `/verify`、`/code-review` 和 Stop hook；
Codex 有 AfterAgent hooks 和 CI Action；OpenCode 靠 LSP 诊断闭环。
但**没有任何一家把"怎么知道做对了"做成一等公民**。

### 判断 4：协议层押注比产品选择更重要

MCP / ACP / A2A 三分已成定局。选 harness 可能三个月就换，
但**选择支持 ACP 的工具意味着你不换编辑器就能换 Agent**——这是 LSP 级别的解耦。
国产里只有 Kimi CLI 到位。

### 判断 5：常驻型 agent 的安全问题会限制 ②类的天花板

CNCERT 预警 + 政府系统限用，已经不是技术问题而是合规问题。
**间接提示注入在架构上无解**——注入指令与用户请求落在同一上下文窗口，
模型没有可靠办法区分二者。这决定了 ②类短期内难以进入企业核心场景。

---

## 附：关键事实的可信度标注

| 事实 | 来源等级 |
|---|---|
| 六职责框架、模型-harness 视角 | ✅ arXiv 综述 |
| MCP/ACP/A2A 分工与状态 | ✅ 官方文档 + 协议站 |
| SWE-bench Pro 三套数字、Terminal-Bench 2.1 修订 | ✅ 第三方榜单方 |
| 70 倍 token 差 | ⚠️ 仅 10 个任务，方向性参考 |
| Claude Code ≈ Codex 的 3–4 倍 token | ⚠️ 社区对比 |
| OpenClaw 安全事件 | ✅ 安全厂商 + 官方文档 + CNCERT |
| WorkBuddy 底层为 CodeBuddy 内核 | ⚠️ 腾讯云社区文章，非官方白皮书 |
| 国产 Claw 分支横评 | ❌ 自媒体/竞品方，倾向性明显 |
| ArkClaw 功能与定价 | ✅ 火山引擎官方 |
