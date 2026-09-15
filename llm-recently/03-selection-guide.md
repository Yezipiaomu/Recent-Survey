# 选型落地建议：四家怎么选、怎么测、踩哪些坑

> 调研日期：2026-09-15
> 一句话结论：**agent 类负载别看 token 单价，看 cache read 单价——
> 这一项 DeepSeek 与 Kimi 差 50–100 倍，足以盖过所有其他因素。**

---

## 1. 价格表（每百万 token，标准档）

| 模型 | 输入 | 输出 | **缓存命中输入** | 备注 |
|---|---|---|---|---|
| **DeepSeek V4.1-Flash** | $0.30 峰 / **$0.15 离峰** | $1.20 / **$0.60** | **$0.006 / $0.003** | 峰值时段：工作日 01:00–04:00、06:00–10:00 UTC；其余半价 |
| **MiniMax M3** | **$0.30** | **$1.20** | — | 标价 $0.60/$2.40，永久五折，≤512K 输入有效；>512K 恢复 $0.60/$2.40 |
| **GLM-5.3-Flash** | $0.15 | $0.50 | — | 最便宜的「够用」档 |
| **GLM-5.3** | $1.40 | $4.40 | $0.26 | |
| **Kimi K3** | $3.00 | $15.00 | $0.30 | 四家最贵 |

**粗口径对照**（1M 输入 + 1M 输出）：

```
GLM-5.3        $5.80
Kimi K3       $18.00
Claude Opus 5 $30.00
GPT-5.6 Sol   $35.00
```

---

## 2. ⚠️ 三个会让预算翻车的陷阱

### 陷阱一：缓存单价的量级差异（最重要）

agent 负载的特征是**同一段上下文被反复读取**——系统提示、工具定义、代码库快照、对话历史。
这类负载里 **cache read 往往占总 token 的 80% 以上**。

看这一列的差距：

| 模型 | cache read 单价 | 重缓存场景相对成本 |
|---|---|---|
| DeepSeek V4.1-Flash（离峰） | **$0.003** | **1×** |
| DeepSeek V4.1-Flash（峰值） | $0.006 | 2× |
| GLM-5.3 | $0.26 | ~87× |
| Kimi K3 | $0.30 | **~100×** |

具体化：同一个重缓存读场景，**DeepSeek 离峰约 $0.15，Kimi K3 约 $15**。

> **这一项的量级差距足以盖过模型能力差异。**
> 如果你在做编码 agent 且没算过 cache 命中率，先去算，再谈选型。

### 陷阱二：token 单价 ≠ 每任务成本

Artificial Analysis 的实测：

| 模型 | token 单价 | Intelligence Index 每任务成本 |
|---|---|---|
| GLM-5.2 | 与 5.3 **完全相同** | $0.44 |
| GLM-5.3 | 与 5.2 **完全相同** | **$0.68（+55%）** |

原因：GLM-5.3 明显更啰嗦。
**升级到「更强」的模型，账单涨了 55%，而 token 价目表一个字没变。**

→ 一定要测 **cost per completed task**，不是 prompt tokens × 标价。

### 陷阱三：MiniMax 的 512K 价格断崖

M3 的 $0.30/$1.20 是**永久五折后**的价格，且**仅在输入 ≤512K 时有效**。
超过 512K 直接回到 $0.60/$2.40，**翻倍**。

如果你的场景就是奔着 1M 上下文去的，**M3 的实际价格是 $0.60/$2.40，不是 $0.30/$1.20**——
此时它相对 DeepSeek 的价格优势消失。

---

## 3. 按场景选型

### 场景 A：编码 / agent，高 cache 命中率（最常见）

**首选：DeepSeek V4.1-Flash**

理由：cache read $0.003–0.006 的量级优势（陷阱一）+ CED 架构本身就是为 prefill 重的 agent 负载设计的
（prefill 每 token 仅激活 8B）+ decode 计算量在 1M 上下文下几乎恒定。

**但必须知道的风险**：
- V4.1-Flash **实质上仍处于 beta**：有 429 错误报告、**20 请求并发上限**
- **无任何第三方评分**（见 [02-benchmark-credibility-audit.md](02-benchmark-credibility-audit.md) §1.1）——
  它的能力目前只有 DeepSeek 自己说了算
- 前代 V4-Pro 在 Artificial Analysis 只有 36 分（开源第五）

**备选：GLM-5.3**（Vals AI 编码类目开源第一 = 64.3）或
**Kimi K3**（Terminal-Bench 2.1 开源第一 = 80.9%）——两者都有扎实的第三方背书，但贵得多。

### 场景 B：要最强开源能力，成本次要

**并列首选：Kimi K3 / GLM-5.3**（Artificial Analysis v4.3 开源并列第一，均 44 分）

区分建议：

| 你的负载 | 选 |
|---|---|
| 终端 / 长程 agent、前端 Web | **Kimi K3**（Terminal-Bench 2.1 80.9% 开源第一；WebDev Arena 总榜第一） |
| 一般编码类目 | **GLM-5.3**（Vals 编码类目 64.3 开源第一，高于 K3 的 60.6） |
| 预算敏感但要接近的能力 | **GLM-5.3-Flash**（AA 42 分，仅比第一低 2 分，$0.15/$0.50） |

> GLM-5.3-Flash 是本次调研里**性价比最突出的单品**：
> AA 开源第三（42），价格是 GLM-5.3 的约 1/9。

### 场景 C：包月订阅式开发（个人/小团队）

**GLM Coding Plan**，$18/月起（$18–168，Team $88/座）。

- 按 5 小时 / 每周配额计费，不按 token
- 支持 Claude Code、Cline、OpenCode 等第三方客户端
- ⚠️ 历史信用记录：2026-02 改价翻车（取消首购优惠、涨幅 30% 起、灰度过慢），2/21 公开道歉

### 场景 D：多模态 / 视频 / 音频生成

**MiniMax，无竞品**——另外三家在这条线上基本没有对应产品。

- **H3（Hailuo 3.0）**：4–15 秒多镜头、最高 2K(1440p)、**原生立体声单次生成**、自然语言改片
- **Speech 2.8**：7 种逐句可控情感、约 10 秒样本克隆
- ⚠️ **Music 3.0 已于 2026-08-20 起不对新用户开放**（老付费客户可继续）——
  如果音乐生成是你的需求，**现在已经进不去了**

### 场景 E：纯成本优先

1. **GLM-5.3-Flash** $0.15/$0.50 — 能力最好的廉价档（AA 42）
2. **DeepSeek V4.1-Flash 离峰** $0.15/$0.60 — 若能把批处理排到非峰值时段
3. **MiniMax M3** $0.30/$1.20 — 但注意 512K 断崖

---

## 4. 合规与许可风险（采购前必看）

### 4.1 许可协议对比

| 厂商 | 协议 | 商用门槛 |
|---|---|---|
| **DeepSeek** | MIT | ✅ 无限制 |
| **智谱 GLM** | MIT | ✅ 无限制（但 GLM-5.3 权重延后两周才放） |
| **MiniMax** | MIT（M3）/ Community License（H3） | ⚠️ H3：年收入 <$2000 万可商用 + 署名 |
| **Kimi K3** | **自定义** | 🚩 **年收入 >$2000 万的公司对外提供 K3 服务须先与 Moonshot 签合同**；月收入 >$2000 万或 MAU >1 亿须显著署名 |

→ **如果你的公司年收入超过 2000 万美元且打算把 K3 作为服务对外提供，这不是「开源随便用」，需要走法务。**

### 4.2 蒸馏争议对采购的影响

Kimi K3 面临美国政府层面的指控，且财政部**已威胁制裁**。客观梳理见
[profiles/kimi-moonshot.md](profiles/kimi-moonshot.md) §风险。

对采购决策的实际含义：

- **公开证据不足**：白宫与 Anthropic 均未公开访问日志、训练数据指纹或取证材料；
  多位独立研究者（Snorkel AI 的 Braden Hancock、分析师 Nathan Lambert）认为时间线不成立
  （Fable 公开仅 15 天 K3 就发布）
- **但制裁风险是真实的**：若指控被认定成立，Treasury 的制裁会直接影响可用性
- **建议**：美国主体 / 受美国出口管制影响的企业，把 K3 的**供应连续性风险**计入选型；
  技术评估与地缘风险评估应分开做，不要因为指控未证实就忽略制裁的尾部风险

### 4.3 数据驻留

四家的官方 API 均为中国境内托管。
**受监管行业（金融、医疗、政府）应评估数据驻留与合规问题**，
或走第三方托管（Together AI、Fireworks、OpenRouter 等已提供 M3、K3 等的境外托管）。

---

## 5. 推荐的实测方案

针对本次调研发现的坑，建议这样验证：

```
第 1 步：确定你的 cache 命中率
  - 跑 100 个真实任务，统计 cache read tokens / total input tokens
  - 若 >60%，cache 单价就是你的主导成本项 → 直接看 §2 陷阱一的表

第 2 步：固定 harness 做对比
  - 选一个脚手架（Claude Code / mini-SWE-agent / 自研）并锁定
  - 不要混用厂商各自的 harness 结果做横向对比（DeepSeek 87.9 vs Vals 80.9 的教训）

第 3 步：测「每完成任务成本」而非 token 单价
  - 记录：完成率、平均输出 token 数、平均工具调用轮数、总费用 / 完成任务数
  - GLM-5.3 的 +55% 成本涨幅完全来自输出啰嗦，token 价目表看不出来

第 4 步：多次运行报方差
  - 至少 3 次；1–2 分的差距视为噪声（智谱 CyberGym 单次 pass@1 的教训）

第 5 步：压测并发与稳定性
  - 特别是 DeepSeek V4.1-Flash：确认 20 并发上限与 429 是否影响你的场景
```

---

## 6. 决策速查表

| 如果你… | 选 | 主要理由 | 主要风险 |
|---|---|---|---|
| 做编码 agent，cache 命中高 | **DeepSeek V4.1-Flash** | cache read 便宜 50–100× | beta 状态、20 并发上限、无第三方评分 |
| 要最强开源，跑终端/长程 agent | **Kimi K3** | AA 并列第一、TB2.1 开源第一 | 最贵、许可门槛、制裁尾部风险 |
| 要最强开源，一般编码 | **GLM-5.3** | AA 并列第一、Vals 编码开源第一 | 每任务成本比 5.2 高 55% |
| 预算紧但要接近顶配 | **GLM-5.3-Flash** | AA 42 分，价格 1/9 | — |
| 包月订阅式开发 | **GLM Coding Plan $18/月起** | 支持 Claude Code 等客户端 | 厂商有改价前科 |
| 视频 / 语音生成 | **MiniMax H3 / Speech 2.8** | 无竞品 | Music API 已对新用户关闭 |
| 纯跑分最高、不限开源 | 不在本调研范围 | Claude Fable 5.1 / GPT-6 Astra 均 53 分 | 贵 5–20× |

---

## 来源

- [VentureBeat — GLM-5.3 定价 $1.4/$4.4](https://venturebeat.com/technology/glm-5-3-hits-the-api-at-1-4-4-4-per-million-tokens)
- [VentureBeat — DeepSeek V4.1-Flash 定价与基准](https://venturebeat.com/technology/deepseek-v4-1-flash-debuts-with-0-003-1m-off-peak-cached-input-rate-and-benchmarks-eclipsing-gpt-5-6-sol-claude-opus-5)
- [MindStudio — GLM-5.3 Flash 定价](https://www.mindstudio.ai/blog/glm-5-3-flash-pricing-api)
- [Tencent Cloud — GLM 5.3 / Kimi K3 / DeepSeek V4 Flash 如何选](https://www.tencentcloud.com/techpedia/147388)
- [Medium — V4.1 Flash vs Kimi K3 vs GLM 5.3 对比](https://medium.com/data-science-in-your-pocket/deepseek-v4-1-flash-vs-kimi-k3-vs-glm-5-3-best-open-weight-llm-right-now-f2dc15a0a137)
- [CostGoat — LLM API 定价对比（2026-09）](https://costgoat.com/compare/llm-api)
- [Morph — 12 家 API 按价格/限流/上下文对比](https://www.morphllm.com/llm-api)
- [Layer3 Labs — GLM Coding Plan](https://www.layer3labs.io/guides/glm-coding-plan-explained)
- [OpenRouter — MiniMax M3](https://openrouter.ai/minimax/minimax-m3)
- [Verdent — M3 用于编码 agent](https://www.verdent.ai/guides/minimax-m3-coding-agents)
