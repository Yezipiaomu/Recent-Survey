# 智谱 / Z.ai 公司档案

> 调研日期：2026-09-15 · [返回索引](../README.md)
> 定位：**全球首家上市的基础模型公司**。四家里唯一不自研底层架构的——
> 把资源全押在后训练与垂直场景，这既是清醒的分工判断，也意味着架构层面没有护城河。

---

## 1. 模型谱系（2026）

| 时间 | 版本 | 规模 | 关键点 |
|---|---|---|---|
| **2026-02-11** | **GLM-5** | 744B / 40B 激活 | 200K 上下文；**华为昇腾训练**；SWE-bench Verified 77.8、BrowseComp 62.0 |
| 2026-04-08 | GLM-5.1 | 744B / ~40B | 开源；202K 上下文；同时 **API 涨价约 10%**（年内第二次） |
| 2026-06-13 | GLM-5.2 | 743B MoE | MIT；**1M 上下文**；High/Max 推理档；自研 **IndexShare** |
| **2026-08-14** | **GLM-5.3** | 同 5.2 底座 | 纯 post-training 提升；low/high/max 三档（默认 max） |
| 2026-08-26 | GLM-5.3 权重 | | MIT，延后两周才放 |

### 一个值得记的营销动作

GLM-5 发布前在 OpenRouter 上以匿名代号 **"Pony Alpha"** 隐身测试；
身份揭晓当日智谱股价**单日涨 26%，盘中一度 +34%**。

---

## 2. 技术路线：不自研，复用 DeepSeek

**这是本次调研中最需要明确的一点。**

### 事实依据

- GLM-5 / GLM-5.1 采用 **DeepSeek 式 MLA + DSA** 架构
- HuggingFace / NVIDIA NeMo AutoModel 中的架构标识符就叫 **`glm_moe_dsa`**
- GLM-5 技术报告（[arXiv:2602.15763](https://arxiv.org/html/2602.15763v1)）把「采用 DSA」列为
  降低训练/推理成本的架构创新——**采用，不是发明**

### 唯一的自研改进：IndexShare（GLM-5.2）

上到 1M 上下文时加入：

> **跨注意力头共享稀疏索引**。indexer 开销按组大小因子下降，
> KV cache 条目可加载一次、在整个 head group 内复用，缓解显存带宽压力。
> 声称 1M token 下有效计算量比「每头独立索引」**少约 2.9×**。

这个改进本身是合理且有价值的——DSA 原版每头独立跑 indexer 在多头下确实浪费。
但它是**在别人的地基上加一层**。

### GLM-5.3 的路线：零预训练改动

明确沿用 GLM-5.2 的**同一个 743B 底座**，全部提升来自：
**更多 RL 环境 + 更广任务覆盖 + 更多 RL 算力**。

### 战略解读

底层架构正在变成公共品（DeepSeek 开源 DSA → NVIDIA 做进 cuDNN → 智谱直接用）。
智谱的选择是：**架构投入压到最低，差异化放在后训练和垂直场景。**

考虑到它是四家中最早上市、财务压力最直接的，这个选择有其合理性。
**但风险是**：若 DeepSeek 上市后停止开源，智谱的架构来源会直接断掉。

---

## 3. 差异化：网络安全垂直

这是智谱在四家中最独特的一步棋。

- GLM-5.3 用**漏洞挖掘的数据和环境**训练，公司称模型「开始跨多个利用阶段进行推理」
- 配合中国的安全团队，报告在 **269 个项目中发现 2436 个漏洞**，部分项目有 40 年历史
- CyberGym 自报 84.5%（超 Mythos 5 的 83.8、GPT-5.6 Sol 的 83.6）
- ExploitBench 54.4%——**仍明显落后前沿模型**

⚠️ 上述 CyberGym 数字是**单次 pass@1**（跨 1507 任务）、**无方差报告**，
与竞品 1 分以内的差距完全在噪声范围内。见 [02-benchmark-credibility-audit.md §2.2](../02-benchmark-credibility-audit.md)。

---

## 4. 第三方评价（这部分比自报数据可信得多）

| 榜单 | 结果 |
|---|---|
| **Artificial Analysis v4.3** | **44 分，开源并列第一**（与 Kimi K3）；GLM-5.3-Flash **42 分开源第三** |
| **Vals Index（综合）** | 57.0，**开源第二**（K3 第一）、总榜第 13（GLM-5.2 曾为第 18） |
| **Vals 编码类目** | **64.3，开源第一**（高于 DeepSeek V4-Flash 61.4、Kimi K3 60.6） |
| **Vals Terminal-Bench 2.1** | 71.5%，开源第二（K3 为 80.9%）；GLM-5.2 为 67.8% |
| **GDPval（AA 评分）** | **1769，高于两家闭源领先者** |

**总体判断**：智谱的第三方成绩是扎实的——**开源第一梯队，编码类目甚至压过 Kimi K3**。
问题不在实力，在于**自报数字的方法学**（见下）。

### ⚠️ 一个负面的独立发现

Artificial Analysis 估算：

| 模型 | token 单价 | 每任务成本 |
|---|---|---|
| GLM-5.2 | 与 5.3 **完全相同** | $0.44 |
| GLM-5.3 | 与 5.2 **完全相同** | **$0.68（+55%）** |

原因是 GLM-5.3 明显更啰嗦。**升级后账单涨 55%，而价目表一字未改。**
这个信息不会出现在发布材料里。

### 自报数字的方法学问题（集中）

- **用竞品 harness 测自己**：CyberGym、ExploitGym、ExploitBench、Terminal Bench 等
  **在 Claude Code 2.1.207 内运行**
- **单次运行无方差**（CyberGym）
- **自选对手分数**
- **发布时零外部复现**
- **基准版本的叙事选择**：自报 Terminal-Bench **3.0** 从 4.6→28.3（6.2×），
  而 Vals 用 **2.1** 测出 71.5%——版本不可比，但 3.0 的低基数让「6.2 倍」格外醒目

---

## 5. 商业

### 上市

- **2026-01-08 港股上市**，全球首家上市的基础模型公司
- 募资约 **5.58–5.6 亿美元**，估值约 **67–71 亿**
- 股价一度涨约 1500–1600%，**从 3 月高点回撤近 3/4，年内仅 +5%**
- 当前市值约 **160 亿美元**（不同来源口径差异大，有一处称近 660 亿，需以最新披露为准）
- 2026-07 进行约 40 亿美元的股份出售

### 收入与毛利（最有信息量的一组数据）

| 项 | 数据 |
|---|---|
| 2026 上半年收入 | **1.36 亿美元** |
| 整体毛利 | 约 **40%** |
| **API 业务毛利** | **仅 0–10%** |
| 利润来源 | **类 Palantir 的客户本地化部署**（在客户硬件上部署） |

→ **这是理解整个行业商业模式的关键数据点**：卖 API 几乎不赚钱，赚钱的是交付。

### GLM Coding Plan

- 订阅制 **$18–168/月**，Team **$88/座**
- 按 **5 小时 / 每周配额**计费，不按 token
- 支持 **Claude Code、Cline、OpenCode** 等第三方客户端
- GLM-5.2 / GLM-5-Turbo 的配额消耗：峰值 3×、离峰 2×；离峰 1× 促销持续到 2026-09

⚠️ **历史信用记录**：2026-02-12 随 GLM-5 发布重构定价——取消首购优惠、涨幅 30% 起，
当日售罄；**2026-02-21 公开向开发者道歉**，承认规则透明度不足、GLM-5 灰度过慢、
老用户升级机制粗糙。

### 其他

- 美国实体清单对象
- **训练使用国产芯片（华为昇腾）而非 NVIDIA**——这在四家中是独特的

---

## 6. ⚠️ 开源策略的松动

GLM-5.3 是智谱**首次打破同步开源传统**：

- 8/14 API 上线，权重**延后两周**
- 官方理由：「史上最广泛的风险评估」
- 期间只有 GLM-5.3-Flash（MIT）可用
- 8/26 旗舰权重才以 MIT 放出

结合 Kimi 加收入门槛、MiniMax 设 2000 万美元红线，
**四家全部走向公开市场后，开源正从战略变成战术。**

---

## 7. 值得继续跟的

1. **DeepSeek 若停止开源 DSA，智谱如何应对**——这是它最大的单点依赖
2. **纯 post-training 路线的天花板**：GLM-5.2 → 5.3 已经是同底座第二次迭代，还能榨几次
3. **网络安全垂直能否变现**——2436 个漏洞是好故事，但没看到对应收入
4. **API 毛利 0–10% 的结构性问题**如何解决；本地化部署能否规模化
5. 股价从高点回撤 3/4 后的**融资能力**

---

## 一手资料

- [arXiv:2602.15763 — GLM-5 技术报告](https://arxiv.org/html/2602.15763v1)
- [NVIDIA NeMo — glm_moe_dsa 架构文档（证明复用 DSA）](https://docs.nvidia.com/nemo/automodel/latest/model-coverage/large-language-models/glm-5-moe-dsa)
- [MindStudio — GLM-5.2 架构：IndexShare 与稀疏注意力](https://www.mindstudio.ai/blog/glm-5-2-architecture-index-share-sparse-attention)
- [The Decoder — GLM-5.3 发布](https://the-decoder.com/zhipu-ai-releases-glm-5-3-claims-its-the-strongest-open-weights-coding-model/)
- [AI News — GLM-5.3 基准方法学质疑](https://www.artificialintelligence-news.com/news/zhipu-glm-5-3-benchmarks-explained/)
- [MLQ News — 权重延后两周](https://mlq.ai/news/zhipu-releases-glm-53-through-its-coding-service-with-weights-still-two-weeks-away/)
- [VentureBeat — GLM-5.3 定价](https://venturebeat.com/technology/glm-5-3-hits-the-api-at-1-4-4-4-per-million-tokens)
- [Maxime Labonne — GLM-5 分析](https://medium.com/@mlabonne/glm-5-chinas-first-public-ai-company-ships-a-frontier-model-a068cecb74e3)
- [Layer3 Labs — GLM Coding Plan](https://www.layer3labs.io/guides/glm-coding-plan-explained)
- [Wikipedia — Z.ai](https://en.wikipedia.org/wiki/Z.ai)
