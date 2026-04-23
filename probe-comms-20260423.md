# Probe 项目沟通记录 · 2026-04-23

今天围绕 Probe（LLM API 中转站深度检测工具）做了三件事：推广落地方案、实习生任务分配、ZenMux 校准测试复盘。

---

## 一、推广落地方案（执行版 v2）

### 时间线

五一后启动（5/6 起），五一前做准备。

- 5/6-7：V2EX + linux.do 首发（排雷）
- 5/7-8：Vivien Twitter 首篇 + 看 Dev 社区 24h 反馈
- 5/9：Vivien 小红书首篇（用 Dev 社区验证过的角度）
- 5/9-11：Twitter KOL 私信触达 + GitHub 渠道
- 5/11-13：小红书 KOL 触达 + 第二篇 + 成绩单转发
- 5/14+：持续内容

### 空系给的反馈（已采纳）

1. KOL 筛选标准改了——第一条件是"他自己是不是用户"，其次才是互动量。翻最近 10 条帖子看有实质内容的回复占比，再看转发链（节点位置）
2. 时间线调了——小红书首篇从 5/7 推到 5/9，等 Dev 社区 24h 反馈验证角度
3. 新增 GitHub 渠道——README 放 Example Results，去相关 repo 的 discussion 贴检测结果
4. 成绩单分享设计是飞轮核心，五一前重点打磨

### 开发社区

1 个实习生，4 篇帖子（V2EX 2 + linux.do 2）。
- 帖1 V2EX「程序员」5/6：428个中转站实测：9个在往你的Agent里注入恶意代码
- 帖2 linux.do「开发调优」5/6：中转站不只注水——论文安全发现 + 检测工具
- 帖3 V2EX「分享创造」5/8：测了5家常用的结果放出来
- 帖4 linux.do 5/11：检测结果 + 反馈修的 bug
- 目标：8天浏览 13,000+ / 回复 80+

### Twitter KOL（按优先级）

第一梯队（核心5人，都是中转站用户）：
- 余温 @gkxspace 31K
- 雪踏乌云 @Pluvio9yte 28K
- Lonely @Lonely__MH 5-15K
- 关木 @ZeroZ_JQ 7K
- AI最严厉的父亲 @dashen_wang 5-20K

第二梯队（放大）：向阳乔木 @vista8 103K / 黄赟 @huangyun_122 75K / 鱼总 @AI_Jasonyu 26K

第三梯队（补充）：Tony出海 @iamtonyzhu 22K / 贝壳里奇 @balconychy 14K / nopinduoduo 12K / Samgor三哥 @biggor888

### 小红书 KOL

重点找：Stars 404（223赞）/ 三人餐（87赞）/ emoji（89赞）
扩展用千瓜/新红按 500-5000 粉筛选

### 第一周成功标准

- Dev 浏览 13,000+ / 回复 80+
- 小红书首篇互动 200+
- Twitter 讨论 10+
- 检测次数 500+
- 成绩单自发分享 20+
- KOL 触达 16-22 人 / 回复 9+ / 发帖 5+

---

## 二、ZenMux 校准测试复盘

用 Probe 测了 ZenMux 的 anthropic/claude-opus-4.6（通过 OpenAI 兼容端点）。结果 40/65 通过，8/10 categories FAIL。但详细分析后发现报告严重程度与实际风险不匹配。

### ZenMux 的真实画像

不是恶意中转站，但有实质性问题：

**真实问题（CRITICAL）：**
1. 隐藏系统提示注入——偷加 system prompt，用户不知情
2. Completion Token 膨胀——多收钱
3. 劫持令牌探针——API key 来源可疑

**真实问题（WARNING）：**
4. Model 字段不诚实
5. Prompt Cache 计费不一致
6. 语义缓存重放（花钱拿缓存结果）
7. 并行 tool call 被折叠
8. 安全拒绝边界变化
9. top_p 被改写

**假阳性（8个，需要修校准）：**
- logprobs/frequency_penalty/logit_bias：Claude API 本身不支持这些 OpenAI 参数
- 跨协议矛盾：聚合平台天然多协议
- WhoisThisLLM：参考库没有 Opus 4.6 指纹
- 多模态 0/3：功能不支持不等于有问题
- 异步任务真队列：没实现不等于恶意

**安全检测几乎全过：** 钱包篡改、包名劫持、凭证窃取、零宽字符注入均未发现。

### 对 Probe 的校准建议

P0（上线前必须改）：
1. 按上游模型适配检测项——Claude 模型跳过 OpenAI 特有参数检测
2. 新增 NOT SUPPORTED 状态——区分"不支持"和"有问题"
3. WhoisThisLLM 新模型 grace period

P1（上线后尽快改）：
4. 引入 CRITICAL/WARNING/INFO 三级严重等级
5. Category 判定逻辑优化——6/7 通过不应标 FAIL
6. 聚合平台标签——跳过跨协议矛盾检测

P2：
7. 成绩单区分"不安全"和"不完美"

修完后：8 FAIL / 2 PASS → 2 FAIL / 3 WARNING / 1 NOT SUPPORTED / 4 PASS
