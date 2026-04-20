# LLM Router 深度分析：UncommonRoute v0.6.0 vs vLLM Semantic Router v0.2.0

日期：2026-04-20
来源：CC 研究 + 空系在 UncommonRoute 提的 issue

---

## 一、UncommonRoute v0.6.0 — 5 个 Issue

仓库：https://github.com/CommonstackAI/UncommonRoute

### Issue #12：tool_use.id 格式损坏

代理转发 Anthropic API 请求时把 tool_use.id 格式改坏了，不符合 `^[a-zA-Z0-9_-]+$`，直接 400。对话中断，致命 bug。

代码位置：`uncommon_route/proxy.py`

### Issue #13：Thompson Sampling 无 session 一致性

每次请求独立采样，没有 session 锁定。

Monte Carlo 10 万次模拟（真实 Pinchbench 数据）：
- COMPLEX：Kimi 54.9%、Opus 20.1%、Haiku 18.3%、DeepSeek 2.6%
- SIMPLE：Kimi 51.7%、DeepSeek 18.9%、Haiku 13.5%、MiniMax 9.8%、Opus 0.9%

45% 的 COMPLEX 请求不选排名第一的 Kimi——同一对话里模型频繁跳。

源码：
- 生产评分入口：`api.py:313` → `select_from_pool()`
- Thompson Sampling：`selector.py:830-834`，`exploration_scale = max(3.0, 4.0 + mu * 6.0)`
- 无任何 session/sticky 机制

### Issue #14：tier 分级形同虚设

SIMPLE 和 COMPLEX 共用同一个候选池。分级只影响 q_weight（0.69→0.89）。

实测：Kimi 在两个级别都是 ~53% 霸榜。分级改变了尾部（Opus 从 0.9% 涨到 20%），但头部不变。

4/20 空系在 issue 上补了 comment：dashboard FEEDBACK 面板实际观察到 SIMPLE 和 COMPLEX 用了几乎同一批模型（minimax/kimi/glm）。

源码：
- `selector.py:797-798`
- `config.py:65-70`：三个 tier 全是空的 TierConfig()

### Issue #15（GitHub #16）：feedback prior_n=20 在单用户部署下无效

Bayesian 混合公式（`selector.py:782,821-823`）：
```
prior_n = 20.0
base_quality = (prior_n * benchmark_q + samples * reward_mean) / (prior_n + samples)
```

| 反馈次数 | 一致性 | 质量分变化 | 用户信号权重 |
|---------|--------|-----------|------------|
| 5 次全负 | 100% | 0.70→0.62 | 20% |
| 20 次全负 | 100% | 0.70→0.50 | 50% |
| 50 次全负 | 100% | 0.70→0.41 | 71% |

prior_n=20 是多用户 SaaS 设计，UR 是本地单用户。反馈用 EWMA，几次误点就能抵消积累。

### Issue #5（成本+benchmark 数据质量）

Kimi 和 Opus 质量几乎相同（0.823 vs 0.822），但 Opus 成本高（cost_norm=1.0 vs 0.39），COMPLEX 下选中率只有 20%。

Benchmark 数据问题：
- Sonnet-4.6 Pinchbench 评分 0.537，低于 Haiku-4.5 的 0.742——明显不对
- deepseek-chat、minimax-m2.5 等未收录，默认 0.5

### 额外发现：prompt caching 冲突

UR 转发时同时设置 cached content 和 tools/system instruction，上游不允许这种组合。独立 bug。

---

## 二、vLLM Semantic Router v0.2.0

仓库：https://github.com/vllm-project/semantic-router

### 基本信息

Go 43.9% + Python 18.2% + TypeScript 14.7% + Rust 11.5%。Envoy ext_proc 架构。

入口只接受 OpenAI 格式（/v1/chat/completions）。`api_format: anthropic` 只是出口转换，不支持 streaming。无法直接替代 UR 做 Claude Code 代理。

### Session 一致性：三层

**第一层：Session 状态追踪**（`extproc/session_transition.go`）

自动推导 session ID（header / 消息 hash / Response API conversation ID）。跟踪 turn_number、previous_model、history_token_count。UR 完全没这层。

**第二层：Cache Affinity**（`selection/cache_affinity.go`）——核心

`Adjustment[m] = lambda_req × C_m × A_m`

W_req（继续性敏感度）= 0.45×复用比 + 0.30×历史质量 + 0.25×轮次深度
- 第一轮 → W_req ≈ 0，不偏好
- 多轮深对话 → W_req → 1，强偏好同模型

A_m（亲和力）= tanh(各项加权)
- emSameBase=0.50, emSameReuse=0.35, emSameTurn=0.15, emDiffPenalty=0.35

关键设计：
- **Ambiguity gating**：top-1 和 top-2 基础分差 > 0.20 时 affinity 归零——强信号不被粘性覆盖
- **最大调整 0.14**——是 tie-breaker 不是 override

**第三层：Handoff Penalty 查找表**（`selection/lookuptable/`）

数据驱动：quality_gap、handoff_penalty、remaining_turn_prior。来自 router replay records。

### Signal 系统

不靠单一 benchmark，多维信号：keywords（BM25/ngram/fuzzy）、embeddings、domains（MMLU 分类）、complexity、structure（问号密度、步骤数、约束密度）、jailbreak/PII 检测、user feedback（wrong_answer/need_clarification）、reask 检测、用户偏好。

### 路由算法

10+ 种：static、confidence、ratings、remom、elo、router_dc、automix、hybrid、rl_driven、gmtrouter、knn、kmeans、svm、mlp、latency_aware。每个 route 可配不同算法。

---

## 三、核心 Insight

**路由的粒度问题**

UR：每次请求都路由。Turn 1→Kimi, Turn 2→Opus, Turn 3→Haiku。看着在路由，实际在赌。

vLLM SR：分层路由。
- Session 首轮：全力评估，选最合适的
- 后续轮次：cache affinity 粘着，除非质量差 > 0.20
- 跨 session：全力路由

路由的价值不是每次 call 都重选，是在对的时机做对的选择。

**Teknium 的判断**（4/20 推文）

"I never believed in model routing for tasks. I'll remove."——他对 agent 场景是对的（中途换模型代价太大），但对独立请求批量场景 routing 仍有价值。核心问题不是"要不要 route"，是"在什么粒度上 route"。

---

## 四、vLLM 和 vLLM SR 的技术关系

### vLLM 本体

LLM serving framework，解决单模型推理效率。核心技术 PagedAttention——KV cache 显存从连续分配改成分页（类似 OS 虚拟内存），同一张 GPU 并发更多请求，吞吐比 HuggingFace 原生高几倍。

定位：一个模型一个 vLLM 进程，暴露 OpenAI 兼容 API。

### vLLM SR 的定位

vLLM 生态的请求层控制平面。不做推理，做推理前的决策。链路：

1. **信号提取**：本地 BERT 分类器分析请求（reasoning? PII? jailbreak? code?），毫秒级
2. **决策匹配**：信号组合匹配预定义 routing decision（keyword=sql AND category=code → code-specialist）
3. **模型选择**：多候选时用算法——static / ELO / Thompson Sampling / POMDP 等
4. **Session affinity**：首轮完整决策，后续粘着，跨 session 重选
5. **Cache affinity**：KV cache 已有上下文的模型微加权（最大 0.14），tie-breaker 不是 override

### 为什么用 Envoy

Envoy ExtProc（External Processor）。Envoy 做 TLS、连接池、重试、可观测性、转发。SR 只做决策。

请求流：客户端 → Envoy → ExtProc → Go Router（信号+决策）→ Envoy 转发到 vLLM 实例

### vLLM 为什么要做路由

1. **生态补全**：多模型、多硬件（NVIDIA+AMD）、多数据中心，不提供官方方案就碎片化
2. **状态耦合**：路由需要 vLLM 内部状态（KV cache 命中、队列深度），纯外部路由做不到 cache affinity
3. **组织层**：vllm-project org 下，共享 maintainer

### 绑定关系

- 协议：假设后端 OpenAI 兼容（vLLM 原生）
- 运维：CLI 可管理 vLLM 实例生命周期
- 状态：cache affinity 依赖 vLLM 内部感知
- 可以路由到非 vLLM 后端，但最佳体验搭配 vLLM

### 结论

vLLM SR 主要适配自建推理集群，不适合通过 API 中转商（CommonStack/OpenRouter）调用云端模型。Anthropic 只做出口转换且不支持 streaming，暂不深入。

值得带走的设计思路：session affinity 粒度选择、cache affinity 作为 tie-breaker、信号分类驱动路由而非纯规则。

---

## 五、我们提了什么

| Issue | 内容 | 状态 |
|-------|------|------|
| #14 comment | tier 分级形同虚设的可观察症状 | 已发 |
| #16 | feedback prior_n=20 单用户无效 | 已开 |

Teknium 4/20 说要删整个 routing 功能。我们的 issue 分析是对的，但他的结论比我们激进——我们说修，他说删。
