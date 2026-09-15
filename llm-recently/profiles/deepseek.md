# DeepSeek 公司档案

> 调研日期：2026-09-15 · [返回索引](../README.md)
> 定位：**架构效率的定义者**。四家里唯一一家其架构创新被竞争对手和 NVIDIA 直接采用的。

---

## 1. 模型谱系（2026）

| 时间 | 版本 | 规模 | 关键点 |
|---|---|---|---|
| 2025-12 | V3.2 | — | DSA 首发；相对 V3.1-Terminus **唯一**改动就是引入 DSA |
| 2026-04-24 | **V4 Preview** | V4-Pro 1.6T/49B · V4-Flash 284B/13B | Hybrid Attention（CSA+HCA）、mHC、Muon 优化器、32T token 预训练、1M 上下文 |
| 2026-08-13 | **V4-Pro GA** | 1.6T/49B | 主打 agent；输出上限 384K token |
| 2026-09-10 | **V4.1-Flash** | 552B 骨干 | MIT 开源；CED 架构；**首个非实验性原生多模态** |

**节奏**：约每 63 天一个版本，下一个预计在 2026-11-12 前后（V4.1-Pro）。

### 版本迁移状态（部署方注意）

- V4-Flash、V4-Flash-Vision-Exp **已退役**，请求重定向到 V4.1-Flash
- 2026-09-14 04:00 UTC 起，**V4-Pro 请求也重定向到 V4.1-Flash**（按 V4.1-Flash 费率计），直到 V4.1-Pro 发布
- → 目前 DeepSeek 实际只有一个在服务的旗舰：V4.1-Flash

---

## 2. 技术路线

完整拆解见 [01-sparse-attention-comparison.md §2.1](../01-sparse-attention-comparison.md)。要点：

### 演进逻辑：选择 → 压缩 → 拓扑重构

1. **DSA（V3.2）**：lightning indexer（多头 ReLU 门控点积、低秩投影、FP8，比主 MLA 便宜约一个数量级）
   + top-k=2048 token 选择。复杂度 O(L²) → O(Lk)。
2. **CSA + HCA（V4）**：DSA 解决了「少算」，但 KV cache 还在。
   CSA 沿序列维压缩 KV 后再跑 DSA；HCA 更激进压缩。
   → **1M 处 KV cache −90%、推理 FLOPs −73%**。
   配套 mHC（流形约束超连接）增强残差、Muon 优化器提升训练稳定性。
3. **CED（V4.1-Flash）**：40 层 = 20 层因果编码器 + 20 层解码器；
   **解码器全局 KV cache 从编码器末层隐状态投影而来**，而非各层自算。
   → prefill 每 token 仅激活 **8B**、decode 16B；
   配合 SWA Bounded Replay，常驻 KV cache 降到 V4-Flash 的 **1/8**；
   配合层次化索引，**decode 计算量在 1M 上下文下几乎恒定**。

### V4.1-Flash 的多模态设计

- **DeepSeek-ViT 从零自研**：2D-RoPE 编码空间位置 + 3×3 pixel-unshuffle 下采样
  （每个视觉 token 代表合并后的 3×3 像素块）→ 两层 MLP 投影器接入语言模型
- **图文从预训练第一步就联合处理**——无冻结视觉塔、无独立适配阶段
- MoE：1 共享专家 + 384 路由专家，每 token 选 6 个
- 训练：45T token 多模态语料从头训练；稀疏注意力在 64K 长度训练，34T token 处扩展到 1M
- 后训练：标准 SFT → RL → on-policy distillation，**无算法改动，变化集中在数据管线**
- **推理强度连续可调（整数 1–100）**，成本/精度可交换

---

## 3. 生态位：DSA 已成行业基础设施

这是 DeepSeek 最被低估的资产：

| 采用方 | 形式 |
|---|---|
| **NVIDIA** | cuDNN 内置 DSA kernel（CuTe-DSL；Hopper SM90 / Blackwell SM100+，压缩 logits + Top-K 合并路径仅 SM100） |
| **智谱** | GLM-5 / 5.1 **直接采用**，HuggingFace 架构名即 `glm_moe_dsa`；GLM-5.2 在其上加自研 IndexShare |
| **学术界** | HISA、MISA、IndexCache、SAC 均以 DSA 为基线 |

→ 竞争对手的模型跑在 DeepSeek 设计的注意力机制上，这在四家里是独一份。

---

## 4. 商业

| 项 | 数据 |
|---|---|
| 首次外部融资 | 2026-06，**74 亿美元**，估值 >500 亿 |
| 正在进行 | 据报以最高 **710–740 亿**估值再融一轮 |
| IPO | 目标**上海科创板**，最早 2027 Q2 |
| 定价变化 | V4-Pro 峰值输出从 $0.87/M **涨到 $3.96/M** |
| 当前 API 价格 | V4.1-Flash：$0.30/$1.20 峰值，$0.15/$0.60 离峰，**cache read $0.003–0.006** |

### 融资结构值得单独看

- **商业投资方**（腾讯、京东、宁德时代）：接受**五年锁定期 + 零投票权**
- **国家人工智能产业投资基金**：有投票权、**不锁定**
- **创始人梁文锋本人出资 200 亿元人民币**，是最大单笔

→ 这个结构显示：创始人对控制权的保护力度极强，国家队拿到了唯一的外部话语权。

---

## 5. ⚠️ 风险与未解问题

### 5.1 最新一代没有任何第三方评分

- **V4.1-Flash 未进入 Artificial Analysis 榜单**，当前跟踪的仍是 V4-Pro 0813
- V4-Pro 在 AA v4.3 只有 **36 分，开源第五**，落后 GLM-5.3 / Kimi K3 **8 分**
- V4.1-Flash 自报「编码与 agent 基准超过 V4-Pro」——**这条链路完全无法独立验证**

### 5.2 自报分数超过第三方全球最高 🚩

DeepSeek 自报 V4-Pro **Terminal Bench 2.1 = 87.9%**；
Vals AI 独立 harness 测出的最高是 Kimi K3 的 **80.9%**、Claude Fable 5 的 80.52%。

**不一定是造假**（该基准对脚手架极敏感），但意味着两组数字不在同一测量体系，不可同表比较。
详见 [02-benchmark-credibility-audit.md §2.1](../02-benchmark-credibility-audit.md)。

### 5.3 服务稳定性

V4.1-Flash **实质仍处 beta**：有 429 错误报告，**20 请求并发上限**。

### 5.4 蒸馏指控（2026-02）

Anthropic 于 2026 年 2 月指控 DeepSeek、Moonshot、MiniMax 进行「蒸馏攻击」：
约 24000 个假账号、1600 万次与 Claude 的对话，通过商业代理绕过中国访问限制。
其中 340 万次追溯到 Moonshot。**针对 DeepSeek 的具体数量未见披露。**

---

## 6. 值得继续跟的

1. **V4.1-Pro**——CED 架构能否扩到大规模，是这套架构的真正考验。DeepSeek 明确说 V4.1-Flash 是「新架构族里最小的模型」。
2. **IPO 后是否继续开源**。DSA 的开源是其生态位的来源，但上市后有收紧压力。若停止开源，**智谱的架构来源会直接断掉**。
3. **第三方何时给 V4.1-Flash 评分**——这是目前最大的信息缺口。

---

## 一手资料

- [arXiv:2512.02556 — DeepSeek-V3.2（DSA 原始论文）](https://arxiv.org/abs/2512.02556)
- [arXiv:2606.19348 — DeepSeek-V4（CSA/HCA/mHC/Muon）](https://arxiv.org/abs/2606.19348)
- [alphaXiv — DeepSeek-V4.1-Flash（CED）](https://www.alphaxiv.org/abs/2609.deepseek-v4-1-flash)
- [HuggingFace — DeepSeek-V4.1-Flash](https://huggingface.co/deepseek-ai/DeepSeek-V4.1-Flash)
- [Baseten — V4.1-Flash prefill 效率分析](https://www.baseten.co/blog/deepseek-v41-flash-more-efficient-prefill-for-coding-agents/)
- [NVIDIA cuDNN — DSA kernels](https://docs.nvidia.com/deeplearning/cudnn/latest/fe-oss-apis/dsa.html)
- [MIT Technology Review — V4 为何重要](https://www.technologyreview.com/2026/04/24/1136422/why-deepseeks-v4-matters/)
- [VentureBeat — V4.1-Flash 定价与基准](https://venturebeat.com/technology/deepseek-v4-1-flash-debuts-with-0-003-1m-off-peak-cached-input-rate-and-benchmarks-eclipsing-gpt-5-6-sol-claude-opus-5)
- [Neowin — V4.1-Flash 发布](https://www.neowin.net/news/deepseek-launches-v41-flash-multimodal-reasoning-model/)
- [Fortune — 中国 AI IPO 潮](https://fortune.com/2026/07/23/moonshot-deepseek-great-chinese-ai-ipo-rush/)
