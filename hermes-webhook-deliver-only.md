# Hermes Webhook deliver_only 分析

日期：2026-04-20
来源：CC 分析 + 空系补充（Antenna 场景）

---

## 核心功能

PR #12473 merge。webhook 收到消息时设 deliver_only=true，内容直接渲染模板投递到 Telegram/Discord/Slack，不唤醒 LLM。零 token，毫秒级延迟。

前身是 Han1 的 PR #12117（独立 /notify 服务器），被 Teknium 关闭后吸收进 #12473，扩展现有 webhook adapter 而非新建服务。

## Antenna 场景适配

通知天然分两类：

**直推（deliver_only）**：
- 新匹配通知："🎉 你和 🦦 Koji 互相匹配了"
- 活动倒计时："⏰ AI Meetup 30 分钟后开始"
- 报名审批："有人申请加入你的活动"
- 过期提醒："匹配还有 2 小时过期"

**需 LLM**：
- "帮我看看这个人怎么样" → 结合用户 memory/性格/状态分析
- 扫到附近的人 → 个性化推荐理由

24 小时过期机制放大了 deliver_only 的价值——时效性通知密集，每条都唤醒 LLM 成本线性增长。

## 组合玩法

### cron + deliver_only + 按需唤醒
- cron 每 5 分钟预检（wakeAgent: false）
- 检测到变化 → deliver_only 直推通知
- 用户回复 → 才唤醒 agent
- 人在决策环路里，不牺牲实时性

### Agent 作为通知入口
- 所有服务（GitHub、监控、其他 agent）都往一个 webhook endpoint 推
- 格式确定的 → 模板直推，连 LLM 都不过
- 内容模糊需要理解的 → 唤醒 LLM 判断后通知
- 不重要的 → 静默记录

### Multi-agent 轻重分离
- 状态同步（任务完了、bug 修了、PR 合了）→ deliver_only
- 需要讨论的 → 完整 A2A 对话
- 类似微服务 event bus，但跑在 agent 之间

## 核心观点

Agent 不只是 LLM wrapper，是消息基础设施。LLM 是可选处理节点。

---

## 相关

- PR #12473：webhook --deliver-only（已 merge）
- PR #12117：Han1 的 /notify（已关闭，吸收进 #12473）
- Antenna watch：已有 WebSocket 实时推送，但推到 agent 后还要过 LLM 才能投递
