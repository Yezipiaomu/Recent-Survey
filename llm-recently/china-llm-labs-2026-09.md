# 中国四大模型厂商研发进展调研（DeepSeek / Kimi / 智谱 / MiniMax）

> 调研日期：2026-09-15
> 数据来源：公开报道、厂商技术报告、第三方评测榜单（详见文末）
> ⚠️ 大量 benchmark 为厂商自测，见文末「数据可信度提醒」
>
> **本文档为总览。深挖文档见 [README.md](README.md)。**
>
> **勘误（2026-09-15 二次核查）**：初版称智谱使用「native sparse attention」有误。
> 实际上 GLM-5 / 5.1 直接采用 DeepSeek 的 DSA（HuggingFace 架构名即 `glm_moe_dsa`），
> GLM-5.2 在其上增加自研的 IndexShare 改进。已在下文表格与判断部分更正。

## 总览结论

四家已经从「追赶通用能力」全面转向 **Agent + 编码** 主战场，1M 上下文和原生多模态成为标配，
架构竞争的焦点是**稀疏化路线**；同时四家全部在走向 IPO，**开源策略正在收紧**。

---

## 一、DeepSeek — 架构效率的定义者

### 模型节奏（约每 63 天一个版本）

| 时间 | 版本 | 要点 |
|---|---|---|
| 2026-04-24 | V4 Preview | 1.6T MoE / 49B 激活；V4-Flash 284B/13B；1M 上下文；Hybrid Attention（CSA + HCA）；mHC 残差；Muon 优化器；32T token 预训练（arXiv:2606.19348） |
| 2026-08-13 | V4-Pro GA | 主打 agent：Terminal Bench 2.1 **87.9**、DeepSWE 62.7、NL2Repo 61.5；输出上限 384K token |
| 2026-09-10 | **V4.1-Flash** | MIT 开源，首个非实验性原生多模态 |

### V4.1-Flash 技术要点（本轮最值得看的动作）

- **规模**：552B 骨干的多模态 MoE
- **CED（Causal Encoder-Decoder）架构**：40 层拆成 20 层因果编码器 + 20 层解码器，
  解码器的全局 KV cache 直接从编码器末层隐状态投影而来
  → prefill 阶段每 token 只激活 **8B**、decode 阶段 16B
- **MoE 配置**：1 共享专家 + 384 路由专家，每 token 选 6 个
- **视觉侧**：自研 DeepSeek-ViT（2D-RoPE + 3×3 pixel-unshuffle），
  图文从预训练第一步就联合处理，无冻结视觉塔、无独立适配阶段
- **训练**：45T token 多模态语料从头训练；稀疏注意力在 64K 长度训练、34T 处扩到 1M；
  后训练为标准 SFT → RL → on-policy distillation，改动集中在数据管线
- **效果**：KV cache 的 HBM 需求降到上一代 **1/4**、SSD 降到 **1/8**；
  用约 1/3 的总参数在编码与 agent 基准上超过 V4-Pro，长程 agent 任务提升最大
- **推理强度**：支持 1–100 连续可调

### 版本迁移

V4-Flash / V4-Flash-Vision-Exp 已退役；9/14 04:00 UTC 起 V4-Pro 请求路由到 V4.1-Flash
（按 V4.1-Flash 费率计），等 V4.1-Pro 发布。

### 外溢影响

DSA 已成行业基础设施——NVIDIA cuDNN 内置 DSA kernel（Hopper SM90 / Blackwell SM100+），
学术界 HISA、MISA、IndexCache、SAC 等均以其为基线。

**DSA 原理**：lightning indexer（多头 ReLU 门控点积 + 低秩投影 + FP8）对所有历史 token 打分取 top-k，
主注意力只在稀疏子集上计算，复杂度从 O(L²) 降到 O(Lk)，k=2048。indexer 每 FLOP 成本比主 MLA 低约一个数量级。

### 商业

- 6 月首次外部融资 **74 亿美元**，估值超 500 亿
- 据报正以最高 **710–740 亿**估值再融一轮，目标科创板，最早 2027 Q2
- 融资结构特殊：腾讯/京东/宁德时代接受五年锁定 + 零投票权；国家人工智能产业投资基金有投票权且不锁定；
  梁文锋本人出资 200 亿元（最大单笔）
- 已涨价：V4-Pro 峰值输出从 0.87 → **3.96 美元/百万 token**

---

## 二、月之暗面 Kimi — 最激进的规模 + 架构双押

### 迭代路径

K2.5（2026-01，1T/32B）→ K2.6（4月）→ K2.7 Code（6月）→ **K3**（7/16 发布，7/27 开权重 + 47 页技术报告）

### Kimi K3 — 全球最大开源权重模型

- **规模**：2.8T 总参数，每 token 仅激活 **104B**；原生视觉 + 1M 上下文
- **KDA（Kimi Delta Attention）**：线性注意力，占 93 层中的 **69 层**；
  另 24 层 Gated MLA 交错插入以保留全局精确注意力（明确的成本/精度权衡）；
  kernel 以 **FlashKDA** 开源
- **AttnRes（Attention Residuals）**：不再逐层均匀累积表示，改为跨深度选择性检索
- **Stable LatentMoE**：896 专家选 16
- **核心主张**：单位算力的智能提升约 **2.5 倍**，而非单纯堆参数；强调算法-系统协同设计
- **配套开源**：高性能注意力 kernel、MoE 通信库、大规模 agent 环境基础设施

### 成绩

- Artificial Analysis 榜 **第 3**（仅次于 Claude Fable 5、GPT-5.6 Sol）
- Vals AI **第 2**
- WebDev Arena 99 个模型中 **第 1（1678 Elo）**，首个登顶该榜的开源模型
- DeepSWE 67.3（mini-SWE-agent harness）

### 能力演示

- 长程编码：大仓库导航、终端工具编排、GPU kernel 优化、编译器开发、CAD、芯片设计
- **K3 Swarm Max**：行业报告/文献综述类任务扇出到大量并行子 agent
- 标志性 demo：设计一颗能跑自身纳米版的物理芯片，**连续自主运行 48 小时**，
  用开源 EDA 工具走完架构设计 → 优化 → 验证全流程

### 风险点

1. **许可收紧**：年收入 >2000 万美元的公司对外提供 K3 服务须先与 Moonshot 签合同；
   月收入 >2000 万或 MAU >1 亿须显著署名。已非标准开源。
2. **蒸馏指控**：白宫 OSTP 主任 Kratsios 公开指控 K3 蒸馏自 Anthropic 的 Claude Fable；
   Anthropic 于 2026-09-10 进一步发难。事件仍在发酵。
3. **工程注意事项**：K3 被训练为跨会话保留推理历史，若 agent harness 未正确回传历史、
   或会话中途从其他模型切换到 K3，输出质量会不稳定。

### 商业

- 估值：年初约 33 亿 → 5 月 200 亿 → 7 月 **350 亿**美元
- 2026-09-03 报道**秘密递交港股 IPO**，拟融 30 亿美元、估值 **500 亿**；承销商高盛 / 中金 / 德银
- 1 月 5 亿美元 C 轮，7 月再融 35 亿；年内累计融资至少 80 亿
- ARR 刚过 **3 亿美元**

---

## 三、智谱 Z.ai — 最早上市，转向「编码 + 安全」垂直

2026-01-08 港股上市，**全球首家上市的基础模型公司**，募资约 5.6 亿美元，估值约 67–71 亿。

### 模型时间线

| 时间 | 版本 | 要点 |
|---|---|---|
| 2026-02-11 | GLM-5 | 744B MoE / 40B 激活，200K 上下文，**华为昇腾训练**；SWE-bench Verified 77.8、BrowseComp 62.0。发布前在 OpenRouter 匿名代号 "Pony Alpha"，身份揭晓当日股价 +26%（盘中一度 +34%） |
| 2026-04-08 | GLM-5.1 | 开源，同时 API 涨价约 10%（年内第二次） |
| 2026-06-13 | GLM-5.2 | MIT 协议，1M 上下文，High/Max 推理档；未发布完整官方基准 |
| 2026-08-14 | **GLM-5.3** | 同 743B 底座，纯靠 post-training 提升 |

### GLM-5.3 路线：不换底座，扩 RL

靠扩大 RL 环境数量、任务覆盖和 RL 算力。自报数据：

- 编码能力比 GLM-5.2 提升 **50%**
- Terminal-Bench 3.0：4.6 → **28.3（6.2 倍）**，开源模型第一；Agents' Last Exam 开源第一
- CyberGym **84.5%**（超 Mythos 5 的 83.8、GPT-5.6 Sol 的 83.6）
- ExploitBench 54.4%（仍落后前沿模型）
- 效率：内部编码基准 31.4% @ 约 5 万输出 token vs Opus 4.8 的 29.5% @ 12 万 token
- GDPval（由 Artificial Analysis 而非智谱评分）：1769，高于两家闭源领先者
- 三档推理强度：low / high / max（默认 max）；1M 上下文通过 `glm-5.3[1m]` model ID

### 差异化：网络安全

用漏洞挖掘数据和环境训练，模型「开始跨多个利用阶段进行推理」；
配合中国安全团队在 **269 个项目中发现 2436 个漏洞**，部分项目有 40 年历史。

### 开源策略松动

GLM-5.3 先 API 后权重——旗舰权重因「史上最严格的风险评估」延后两周，
8/26 才以 MIT 放出，期间只有 GLM-5.3-Flash 可用。这是智谱首次打破同步开源传统。

### 商业

- **GLM Coding Plan**：订阅制 18–168 美元/月，Team 88 美元/座；
  按 5 小时 / 每周配额而非按 token 计费；支持 Claude Code、Cline、OpenCode 等第三方客户端
- 2 月改价翻车：取消首购优惠、涨幅 30% 起、灰度过慢，2/21 公开向开发者道歉
- 上半年收入 **1.36 亿美元**
- **毛利约 40%，主要来自类 Palantir 的客户本地化部署；API 业务毛利仅 0–10%**
- 股价从 3 月高点回撤近 3/4，年内仅 +5%，市值约 160 亿美元（不同来源口径差异大）
- 美国实体清单对象，训练用国产芯片而非 NVIDIA

---

## 四、MiniMax — 唯一在文本与视听两线同时推进的

2026-01 港股上市，首日翻倍。

### 文本线

M2（2025-10，230B/10B）→ M2.1（2025-12-23）→ **M2.5**（2026-02-12）→ **M2.7**（03-18）→ **M3**（06-01）

- **M2.5**：架构不变（230B/10B），纯靠 **20 万+ 真实环境的大规模 RL**。
  SWE-Bench Verified 80.2%、Multi-SWE-Bench 51.3%（第一）、BrowseComp 76.3%；
  比 M2.1 快 37%，成本约 Claude Opus 4.6 的 1/10。另有 M2.5-Lightning 变体。
- **M2.7**：号称「首个深度参与自身演化的模型」，能自建 agent harness
  （Agent Teams、复杂 Skills、动态工具搜索）。205K 上下文，0.24 美元/百万 token 起。
- **M3**：428B，MIT 协议，自研 **MSA（MiniMax Sparse Attention）**，1M 上下文 + 原生多模态。
  定位「首个同时具备顶级编码/agent 性能、1M 上下文和原生多模态的开源权重模型」。
  价格 0.23 / 0.96 美元每百万 token（输入/输出），cache read 0.05。

#### ⚠️ M3 自报 vs 第三方落差最大

| 来源 | 结果 |
|---|---|
| MiniMax 自报 | SWE-Bench Pro 59.0%、MCP Atlas 74.2%、BrowseComp 83.5（超 Opus 4.7 的 79.3）、OmniDocBench 超 Gemini 3.1 Pro、SVG-Bench 超 Opus 4.7 |
| Artificial Analysis | Intelligence Index **30**（同体量开源中位数 18），109.3 tok/s，TTFT 1.33s |
| Vals AI | 综合 **第 6**（58.94%），开源第一；但 SWE-bench Verified 75.00% 排 **第 17**、Terminal Bench 2.1 53.56% 排 **第 12** |
| BenchLM | 232 个模型中 **第 52**（61.55/100）；多模态最强（第 34），Agentic 最弱（第 107） |

MiniMax 自己披露：多项结果跑在自家基础设施上，常用 Claude Code / Mini-SWE-Agent / Terminus 做脚手架。

### 视听线（真正的护城河）

- **H3 / Hailuo 3.0（2026-07-31）**：omni-modal 视频模型，文本/图像/视频/音频统一上下文
  - 4–15 秒多镜头视频，24 fps，六种画幅（21:9 到 9:16）
  - 768p 模式可升 1440p；官方推荐 1440p 模式，21:9 下达 2976×1248
  - **原生立体声**，单次生成，无独立音频阶段
  - 支持自然语言指令改片而非重摇
  - 计划以 MiniMax Community License 开权重（收入 <2000 万美元可商用 + 署名）
  - 产品线关系：MiniMax 是公司，Hailuo 是消费级视频品牌，H3 是第三代模型，API ID `MiniMax-H3`
  - **明确批评行业「任务孤岛」**——把图像编辑、参考类型、语音、音效、音乐、各类视频变体拆成一堆专家模型
- **Speech 2.8（2026-01-23）**：7 种逐句可控情感，约 10 秒样本语音克隆
- **Music 3.0**：⚠️ 2026-08-20 起音乐/歌词 API 不再对新用户开放，老付费客户可继续
- Hailuo 2.3 / 2.3 Fast / Hailuo 02 已移入 Legacy

### 商业

上半年收入 **1.17 亿美元**。

---

## 五、横向对比

| | DeepSeek | Kimi (Moonshot) | 智谱 Z.ai | MiniMax |
|---|---|---|---|---|
| 当前旗舰 | V4.1-Flash (552B) / V4-Pro (1.6T) | K3 (2.8T / 104B 激活) | GLM-5.3 (743B / 40B) | M3 (428B) + H3 |
| 稀疏路线 | DSA → CSA+HCA → CED（自研） | KDA 线性注意力 + MLA 混合（自研） | **直接复用 DeepSeek DSA** + 自研 IndexShare | MSA 块级稀疏（自研） |
| 上下文 | 1M | 1M | 1M | 1M |
| 原生多模态 | ✅ 2026-09 刚实现 | ✅ 视觉 | ❌ 文本为主 | ✅ 含视频/音频 |
| 开源协议 | MIT | 自定义（收入门槛） | MIT（权重延后放） | MIT / Community License |
| 差异化 | 架构与推理成本效率 | 规模 + 长程自主 agent | 编码订阅 + 网络安全 | 视听生成 + 性价比 |
| 上市状态 | 拟科创板（~2027 Q2） | 递表港股（$50B） | 已上市（~$16B） | 已上市 |
| 收入 | — | ARR >$300M | 上半年 $136M | 上半年 $117M |

---

## 六、判断

1. **架构竞争已从「堆参数」转到「每 token 激活多少」**。
   K3 用 2.8T 总参数只激活 104B，V4.1-Flash 用 552B 骨干在 prefill 时只激活 8B——
   这两个数字比任何 benchmark 都更能说明方向。
   稀疏注意力路线分化：DeepSeek 走 token 级 indexer 选择 + KV 压缩、Kimi 走线性注意力混合、
   MiniMax 走块级选择；**智谱是唯一不自研底层架构的——直接复用 DeepSeek 的 DSA**，
   把资源全押在 post-training。**这是未来一年最值得跟的技术分岔。**
   详见 [01-sparse-attention-comparison.md](01-sparse-attention-comparison.md)。

2. **评测重心整体迁移到 Terminal-Bench / SWE-Bench / BrowseComp / MCP Atlas**，
   MMLU 类基准基本退场。「模型能力」的定义已变成「能否在真实工具环境里跑长任务」。

3. **开源正在从战略变成战术**。GLM-5.3 权重延后两周、Kimi K3 加收入门槛、
   MiniMax 设 2000 万美元红线——四家全部走向公开市场后，
   「把研究免费送出去」的共享池大概率持续收紧。同时 DeepSeek 和智谱都已开始涨价。

4. **中国实验室已占 OpenRouter 约 45% 的 token 流量**（一年前不到 2%）；
   Stanford HAI AI Index 2026 称中美头部模型差距收窄到 **2.7%**，而中国私人 AI 投资约少 23 倍。

5. **收入仍是最大软肋**。ARR 3 亿、半年 1.36 亿和 1.17 亿的量级，
   撑不起 350–740 亿美元的估值预期。智谱的数据揭示了一个残酷事实：
   **API 业务毛利只有 0–10%，真正赚钱的是本地化交付**。这会反向影响接下来的产品形态。

---

## ⚠️ 数据可信度提醒

上述大量 benchmark 数字为**厂商自测、自选对手分数、自建 harness**：

- **智谱**在 Claude Code 2.1.207 里跑 GLM-5.3 的评测；CyberGym 是单次 pass@1（1507 任务）、
  无方差报告，1–2 分的差距完全在噪声内；发布时无任何外部实验室复现。
- **MiniMax** 明确披露多项结果跑在自家基础设施上、用 Claude Code / mini-SWE-agent / Terminus
  做脚手架，第三方分数明显更低（见上表）。
- 部分来源为预测性/投机性内容：Introl、VERTU 曾预测 DeepSeek V4 于 2 月发布并带
  "Engram 记忆架构"——实际为 4 月 24 日发布，且**未找到任何 Engram 与 DeepSeek 相关的证据**；
  SitePoint 一文明确标注为推测。
- 智谱市值一项来源间差异巨大（160 亿 vs 660 亿美元），需以最新披露为准。

**选型建议**：只信第三方榜（Artificial Analysis / Vals AI / BenchLM）+ 自己的工作负载实测。

---

## 来源

### DeepSeek
- [MIT Technology Review — Why DeepSeek's V4 matters](https://www.technologyreview.com/2026/04/24/1136422/why-deepseeks-v4-matters/)
- [Yahoo Tech — V4-Pro GA](https://tech.yahoo.com/ai/articles/deepseek-officially-launches-v4-pro-181255468.html)
- [Neowin — V4.1-Flash 多模态推理模型](https://www.neowin.net/news/deepseek-launches-v41-flash-multimodal-reasoning-model/)
- [HuggingFace — DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)
- [Baseten — V4.1-Flash prefill 效率分析](https://www.baseten.co/blog/deepseek-v41-flash-more-efficient-prefill-for-coding-agents/)
- [arXiv:2512.02556 — DeepSeek-V3.2 技术报告（DSA 原始论文）](https://arxiv.org/abs/2512.02556)
- [Sebastian Raschka — DeepSeek Sparse Attention](https://sebastianraschka.com/llm-architecture-gallery/deepseek-sparse-attention/)
- [NVIDIA cuDNN — DSA kernels](https://docs.nvidia.com/deeplearning/cudnn/latest/fe-oss-apis/dsa.html)

### Kimi / Moonshot
- [GitHub — MoonshotAI/Kimi-K3](https://github.com/MoonshotAI/Kimi-K3)
- [VentureBeat — 最大开源模型](https://venturebeat.com/technology/chinas-moonshot-ai-releases-kimi-k3-the-largest-open-source-model-ever-rivaling-top-u-s-systems)
- [CNBC — Kimi K3 对标 OpenAI/Anthropic](https://www.cnbc.com/2026/07/17/moonshot-ai-kimi-k3-model-openai-anthropic-china.html)
- [Fortune — K3 推进中国 AI 至 Fable 级](https://fortune.com/2026/07/16/moonshots-kimi-k3-pushes-chinese-ai-into-fable-level-territory/)
- [Tech Startups — 港股 IPO 递表](https://techstartups.com/2026/09/03/chinese-ai-startup-moonshot-files-for-3-billion-hong-kong-ipo-at-50-billion-valuation/)
- [Wikipedia — Kimi (AI)](https://en.wikipedia.org/wiki/Kimi_(AI))

### 智谱 / Z.ai
- [The Decoder — GLM-5.3 发布](https://the-decoder.com/zhipu-ai-releases-glm-5-3-claims-its-the-strongest-open-weights-coding-model/)
- [AI News — GLM-5.3 基准方法学质疑](https://www.artificialintelligence-news.com/news/zhipu-glm-5-3-benchmarks-explained/)
- [MLQ News — 权重延后两周](https://mlq.ai/news/zhipu-releases-glm-53-through-its-coding-service-with-weights-still-two-weeks-away/)
- [Maxime Labonne — GLM-5 分析](https://medium.com/@mlabonne/glm-5-chinas-first-public-ai-company-ships-a-frontier-model-a068cecb74e3)
- [Layer3 Labs — GLM Coding Plan](https://www.layer3labs.io/guides/glm-coding-plan-explained)
- [Wikipedia — Z.ai](https://en.wikipedia.org/wiki/Z.ai)

### MiniMax
- [MiniMax 官方 — M3](https://www.minimax.io/models/text/m3)
- [Artificial Analysis — MiniMax-M3](https://artificialanalysis.ai/models/minimax-m3)
- [Vals AI — MiniMax-M3](https://www.vals.ai/models/minimax_MiniMax-M3)
- [MiniMax 官方 — H3 / Hailuo 3.0](https://www.minimax.io/blog/minimax-h3)
- [MiniMax 官方 — M2.7](https://www.minimax.io/news/minimax-m27-en)
- [Maxime Labonne — M2.5 分析](https://medium.com/@mlabonne/minimax-m2-5-the-1-hour-frontier-model-92168de195b8)

### 行业
- [Fortune — 中国 AI IPO 潮](https://fortune.com/2026/07/23/moonshot-deepseek-great-chinese-ai-ipo-rush/)
- [Asia Tech Review — Moonshot IPO 与变现压力](https://www.asiatechreview.com/p/moonshots-massive-ipo-means-chinese)
- [Dealroom — 中国 AI 生态分析](https://dealroom.co/news/136199-inside-chinas-ai-ecosystem-beyond-deepseek-zhipu-minimax-moonshot-byteda/)
- [Business Model Analyst — 开源与 IPO 的冲突](https://businessmodelanalyst.com/china-ai-labs-shared-rd-ipo-clock/)
