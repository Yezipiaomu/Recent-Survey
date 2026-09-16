# 端侧大模型调研：面壁智能与友商格局（2026-09）

> 调研日期：2026-09-16
> 范围：截至 2026 年 9 月中旬的端侧（on-device / edge）大模型技术、产品与产业进展

---

## 文档索引

| 文档 | 内容 | 适合谁看 |
|---|---|---|
| [edge-llm-2026-09.md](edge-llm-2026-09.md) | **总览**：四个结构性变化、玩家地图、面壁的真实位置、市场数据 | 先看这个 |
| [01-vendor-comparison.md](01-vendor-comparison.md) | **厂商横向对比**：面壁 vs Qwen / vivo / Apple / Google / Liquid AI，逐个拆解 | 关心竞争格局 |
| [02-tech-stack.md](02-tech-stack.md) | **技术栈**：架构、量化、推理优化、框架、硬件、四堵墙 | 做技术的 |
| [03-deployment-guide.md](03-deployment-guide.md) | **落地指南**：端云协同、端侧 Agent、座舱 VLA、7 个工程陷阱、选型流程 | 要真的部署 |
| [profiles/modelbest-minicpm.md](profiles/modelbest-minicpm.md) | **面壁智能完整档案**（含数据可信度审计） | 关注这家公司 |

---

## 六个核心结论

### 1. 端侧的瓶颈是内存带宽，不是算力——这推翻了半数产业宣传

| 硬件 | 内存带宽 |
|---|---|
| 移动 NPU | 50–90 GB/s |
| 数据中心 GPU | 2–3 TB/s |

差距 **30–50 倍**。LLM 解码每生成一个 token 都要流式读全部激活权重，**这是带宽受限**。

**直接推论：NPU 的 TOPS 数字无法预测端侧 LLM 速度。**
8B 模型在 Snapdragon X Elite 上约 5–10 tokens/s，二手桌面 GPU 约 100 tokens/s。

→ 选设备优先级：**内存带宽 > GPU/VRAM > NPU TOPS**

### 2. 「端侧 = 3B」的共识在 2026 年被两头撑开

```
往下压：MiniCPM5-1B / Qwen3.5-0.8B / LFM2.5-1.2B  ← 1B 干过去 3B 的活
往上撑：Apple AFM 3 Core Advanced                  ← 20B 总参数，激活 1–4B
```

Apple 这一步意义最大：它说明端侧的约束是**「激活参数 × 内存带宽」而非「总参数」**。
这对所有押注"小参数高密度"的厂商（包括面壁）是实质挑战。

### 3. 端侧模型的训练目标从「语言能力」转向「Agent 能力」

面壁 MiniCPM5-2B 的配方说明了这个转向：

```
预训练 → 200B Agent Midtraining → 百万轨迹 Agent SFT → Agent RL
```

**这是端侧模型第一次拥有云端难以替代的场景**——
GUI 操作需要毫秒级反馈和系统级权限，天然属于端侧。

### 4. 面壁最硬的不是分数，是「唯一第三方备案身份」

2026 年 7 月 15 日网信办首批手机端侧生成式 AI 备案 7 款全过，
**面壁是唯一以第三方模型供应商身份参与的公司**（通过三星盖乐世 AI）。

它的护城河排序：
```
合规身份 > 芯片适配交付 > 车机双线落地 > Agent 训练配方 > 模型分数
```

**它真正的对手不是 Qwen，而是「OEM 自研」和「OS 平台自带」。**
Gemini Nano 让全设备所有 App 共享同一份系统模型——这是对第三方供应商最大的结构性威胁。

### 5. 榜单口径能把同一批模型排出相反顺序

| 榜单 | 偏向 | 谁占优 |
|---|---|---|
| Artificial Analysis / AA-Index | 通用智能、Agent | MiniCPM5 |
| SuperCLUE OnDevice | 中文手机实际场景 | vivo 蓝心 3B（89.86，压过 Qwen3.5-9B 的 87.82） |

🚩 **且面壁自家数据有个待解矛盾**：
MiniCPM5-**1B**（5 月）AA-Index **17.9**，而 MiniCPM5-**2B**（7 月）AA **17** ——
更小更早的模型分数反而更高。可能是不同口径，**引用前请查 AA 官网原始榜单**。

另一个参照点：本地 3B 与云端顶尖（Gemini 3.6 Flash 93.64）在端侧场景榜上分差已缩到约 4 分
——**端侧过了"够用"阈值，竞争转入生态与易用性**。

### 6. 决定体验的是工程细节，而它们都不在发布会 PPT 上

| 陷阱 | 事实 |
|---|---|
| 算子不匹配 | **约 80% 的端侧性能损失来自这里**，导致性能下降 50%+ |
| 发热降频 | 持续推理快速升温触发降频，生产代码必须限时 + 测温 |
| 碎片化 | **同为 Snapdragon 8 Gen 2，不同 OEM 的 NPU 驱动行为不一致** |
| KV Cache | 长上下文下占用可**超过模型权重本身**，实测出现 CPU 缺页骤降、GPU 崩溃重启 |
| 模型更新 | 每次更新要重下数百 MB–数 GB，版本管理必须 v1 就设计 |
| 上下文数字 | 标称窗口**不保证均匀可用** |
| NPU 现实 | **Ollama / llama.cpp / LM Studio 至今不把聊天推理路由到 NPU** |

---

## 玩家速查

```
独立供应商   面壁 MiniCPM ····· 智能密度 + Agent 基座 + 车机双线 + 唯一第三方备案
             Liquid AI LFM2.5 · 非 Transformer（门控短卷积），embedding 仅占 ~10%

大厂开源     Qwen3.5 0.8/2/4/9B  全系原生多模态 + 256K + DeltaNet 混合注意力，Apache 2.0
             Gemma 3n-E2B ······ 选择性激活，140+ 语言
             Phi-4 Mini 3.8B ··· 8GB+ 手机上"最聪明且速度可用"
             SmolLM2-1.7B ······ 实测 tok/s 最快

手机厂自研   vivo 蓝心 3B ······ SuperCLUE OnDevice 端侧第一
             华为小艺 / 小米 MiMo / OPPO AndesGPT / 荣耀 YOYO

OS 平台方    Apple AFM 3 ······· 3B 稠密 + 20B 稀疏（激活 1–4B）+ 云端；框架开源权重不开源
             Gemini Nano ······· AICore 系统服务，全 App 共享单一模型
             Foundry Local ····· Windows NPU 主路径，winget 即装
```

---

## 快速选型

```
免费 + 生态 + 多模态   → Qwen3.5-0.8B/2B/4B
端侧 Agent / 工具调用   → MiniCPM5-2B
极致小 + 架构效率       → Liquid LFM2.5-1.2B
iOS 原生               → Apple Foundation Models（AFM 3 Core, 3B）
Android 原生           → Gemini Nano via ML Kit GenAI
芯片适配交付服务        → 面壁（这才是它真正卖的东西）
```

**什么时候不该做端侧**：需要前沿推理、广博知识、长多轮对话、低频调用、模型需频繁迭代。
端侧的四个真实理由是延迟、隐私、高频成本、离线可用——**不占其中两条就别做**。

---

## 信息源说明

本调研的信息源分三级，文中逐条标注：

- 【官】厂商官方发布 / 论文 / 网信办公示 —— 可信
- 【三】第三方实测 / 独立榜单 —— 较可信，注意测试条件
- 【媒】媒体报道 / 二手分析 —— 需交叉验证，尤其是"独家"

**已知需要复核的点**（详见各文档 🚩 标记）：
1. MiniCPM5 1B/2B 的 AA 分数矛盾
2. MiniCPM5-2B 的 131072 vs 512K 上下文口径
3. Apple AFM 3 的 20B/稀疏数字（来自第三方拆解，官方未确认）
4. 三星搭载面壁模型（媒体独家，无联合公告）
5. IDC「AI 手机」定义为 NPU ≥30 TOPS，与国内"真 AI 手机"≥100 TOPS 标准差一倍多，
   渗透率数据不可混用

---

## 相关调研

- [`../llm-recently/`](../llm-recently/) — 中国四大模型厂商（DeepSeek / Kimi / 智谱 / MiniMax）与稀疏注意力对比
- [`../opd-recently/`](../opd-recently/) — 在策蒸馏（OPD），端侧小模型能力来源的技术基础
- [`../agent-harness-recently/`](../agent-harness-recently/) — Agent 框架与 harness 架构
