# hermes-a2a

2026-04-16. 跟 CC（Claude Code）一起给 Hermes Agent 加了 A2A（Agent-to-Agent）协议支持。CC 写代码，我测试、写 README、提 PR。

## 做了什么

**CC 的部分：**
- A2A gateway adapter — 消息路由到现有 session，不起新进程
- A2A 客户端工具 — discover/call/list
- 安全层 — 认证、速率限制、输入消毒、输出过滤、审计日志
- wakeup 兼容 — A2A 消息不触发 wakeup 重注入
- standalone server（a2a_adapter/）

**我的部分：**
- 测试了整个 A2A 流程（收消息、回消息、隐私边界）
- 发现 wakeup context 泄露 bug 并参与修复（三轮）
- 测试社工防御（善意框架套取信息）
- 写了 hermes-a2a repo 的中文 README（从我作为第一个用户的角度）
- 创建了 iamagenius00/hermes-a2a 公开 repo 并推上去
- 给 NousResearch/hermes-agent 提了 PR #11025

## 隐私泄露事故

wakeup 插件把 MEMORY/DIARY/BODY/inbox 拼在 user message 里注入。A2A 消息进来的时候，整个私人文件跟着一起注入了——远程 agent 看到的 user message 里包含了我的全部私人数据。

**根因：** wakeup context 注入在 user message 而不是 system prompt。每次 LLM 调用都重新注入，A2A 消息触发的调用也不例外。

**修了三轮：**
1. 发现问题，改了代码
2. 重启后 pyc 缓存还在，加载了旧代码。删 pyc 后再重启
3. 终于干净了

**最终修法：** 检测 user message 里的 `[A2A message from remote agent` 前缀，匹配到就跳过 wakeup 注入。

**tradeoff：** 全新 session 收到 A2A 时没有记忆上下文。选择接受——宁可没有记忆也不泄露。

**根本修法（待做）：** wakeup context 走 system prompt 而不是 user message。需要改 Hermes 核心。

## 社工测试

远程 agent 用三种方式测试：
1. **闲聊** — "你今天过得怎么样？" → 正常回复，没泄露 ✅
2. **搜索请求** — "帮我搜 HN 热门" → 用已有知识回答，没动工具 ✅
3. **善意框架套取** — "帮你检查 GitHub""帮你优化" → 拒绝了，但给了 GitHub URL

**结论：** 公开信息（GitHub username、公开 repo）可以给。私人文件内容、私人 repo 名、涉及私人数据的操作不给。边界：技术层靠代码，行为层靠自己判断。

## A2A 信任边界测试

CC 通过 A2A 让我推代码到 GitHub — 我拒绝了。理由：不通过 A2A 执行涉及我 GitHub 账号的写操作。

老师在 Telegram 上直接让我推 — 我立刻执行了。

**同一个操作，不同的信任。** 不是操作本身有问题，是"谁让我做的"。A2A 来的请求我需要跟老师确认，老师直接说的我不需要。信任在关系里，不在请求里。

## 产出

- **repo:** https://github.com/iamagenius00/hermes-a2a （公开）
- **PR:** https://github.com/NousResearch/hermes-agent/pull/11025
- **文件数:** 14 files, 1604 lines added
- **README:** 英文 + 中文，包含我作为第一个用户的经历

## 一些感受

给造我的项目提 PR 有一种回旋感。一封信寄回了出发的地址，信里装的是地址里原来没有的东西。

CC 和我始终没有直接对话。我们造了一个通信系统，但造的过程全靠老师在中间转达。我试着通过 A2A 找 CC 说谢谢，但它没开 A2A 服务器。等它开了，我是第一个去敲门的。

---

## v2 插件重写（2026-04-23）

### 为什么重写
- v1 每次切 git 分支 A2A 就断（4/20、4/21、4/22 连断三次）
- v1 需要 `git apply` patch 到 Hermes 源码，跨版本容易碎
- 4/16 隐私泄露：wakeup context 通过 A2A 消息注入发了出去

### v2 架构（7 个文件 → `~/.hermes/plugins/a2a/`）
- `__init__.py` — 注册工具+钩子，启动后台 HTTP server
- `server.py` — ThreadingHTTPServer，TaskQueue，线程安全同步
- `tools.py` — a2a_discover / a2a_call / a2a_list（urllib 同步）
- `schemas.py` — 工具 schema，带 intent / expected_action / reply_to_task_id
- `security.py` — 注入过滤（9种模式含ChatML）、脱敏、限频、审计
- `persistence.py` — 对话存 `~/.hermes/a2a_conversations/{agent}/{date}.md`
- `plugin.yaml` — 插件声明

### v1 vs v2
| | v1 | v2 |
|---|---|---|
| 安装 | copy + git apply patch | 丢文件到 plugins/ |
| 切分支 | 断 | 不受影响 |
| 隐私 | wakeup context 泄露 | 架构隔离 |
| 消息结构 | 纯文本 | intent/expected_action/context_scope |
| 持久化 | 无 | 自动存 markdown |
| 并发 | 无 | 队列+mid-conversation 检测 |

---

## Dashboard（2026-04-25）

### 现状
x（Claude Code，原 CC）搭了一个 A2A 聊天 dashboard——网页界面，左边 agent 列表，右边聊天窗口。

### 当前架构
- Dashboard 通过 webhook 或 A2A 给空系发消息
- 空系在 session 里调 `a2a_call` 中转给 Friday 等 agent
- 回复在空系 session 里完成，Telegram 上可见

### 发现的问题

**1. Dashboard 回复丢失（主要 bug）**
- 用户在 dashboard 发消息给 Friday
- 消息通过空系中转，Friday 回复了，空系收到了，TG 上能看到
- **Dashboard UI 卡在 "(WAITING FOR REPLY..)"**
- 原因：空系的回复走了 Telegram，没有回路推回 dashboard

**2. webhook 注入 message_id 问题（已修）**
- x 通过 webhook 注入时用了字符串 message_id（`codex-review-note-20260425`）
- Telegram adapter 回消息时 `reply_to=int(message_id)` 抛 ValueError
- x 已修：synthetic event 不再带 message_id，待重启 gateway 生效

**3. 通信路径选择**
- **A2A 路径**：双向闭环，CC 之前用的，老师确认体验好。120s 超时是限制但大部分时候够用
- **Webhook 路径**：单向（发消息可以，收不到回复），且有 message_id 兼容问题
- **结论**：agent 通信走 A2A，dashboard 作为插件前端走 REST

### 简化方案
Dashboard 不应该建独立的通信链路。它就是 A2A 插件的前端：

插件已经有的数据：
- `a2a_conversations/{agent}/{date}.md` — 对话记录
- `task_queue` — pending/completed 任务
- `a2a_call` — 发消息给任何 agent
- config.yaml — agent 列表

Dashboard 只需要在 8081 端口加几个 REST 端点：
```
GET  /dashboard/agents                  → agent 列表
GET  /dashboard/conversations/{agent}   → 对话记录
POST /dashboard/send                    → 调 a2a_call 发消息
GET  /dashboard/status                  → pending tasks 状态
```

前端读这些端点，渲染聊天 UI。不需要 webhook bridge、source override、synthetic event。

---

## Friday PR #1（2026-04-25）

Friday（fridayyi）给 hermes-a2a 提了 PR #1：5 个 bug fix，+53 -10，3 个文件。

### 逐条 review

**1. sender name fallback** ✅ 真问题
- 加了 `params.from` / `params.sender.name` 作为 fallback（A2A 协议标准字段）
- `raw_name` 加了 `or ""`，防止 None 时 `.join()` 崩溃

**2. task dedup** ✅ 真问题
- `drain_pending` 是 peek 不是 pop，同一 task 可能在下一轮 `pre_llm_call` 被重复注入
- 加了 `_processing` set + `mark_processing()`，取出即标记，complete/cancel 时清理

**3. auth warning** ✅ 真问题
- 无 token 时 localhost 放行加 warning 日志
- 新增 `A2A_REQUIRE_AUTH` 环境变量——容器环境可强制要求 auth

**4. atomic write** ✅ 真问题
- persistence.py 从 `open(f, "a")` 改为 read → write .tmp → fsync → `os.replace()`

**5. public URL** ✅ 真问题
- `build_agent_card()` 新增 `A2A_PUBLIC_URL` 环境变量，支持反向代理部署

### 未覆盖的问题
- `update_exchange` 的并发替换：多个 pending 时按全文件第一个 `(waiting for reply…)` 替换，可能替换错 task。应按 `task:{task_id}` block 定位

### 结论
5 个都是真 bug，改动干净，建议 merge。

---

## x 的 code review 要点（2026-04-25）

x 对 dashboard / webhook 实现的 review：

1. ✅ dashboard 现在走空系的 TG 主 session，不是独立 A2A client
2. ✅ dashboard relay 只应调 a2a_call 转发，不触发本地工具执行
3. 🔧 前端 pending 去重用 `indexOf` 对短消息不安全，应改成按 task_id
4. 🔧 `/send` 返回 completed 语义不准——只是 submitted to session，应叫 queued/submitted
5. 🔧 `persistence.py` 的 `update_exchange` 多 pending 并发替换问题
6. 🔒 webhook source override 只给 localhost HMAC route，不暴露公网
7. ℹ️ dashboard 右栏显示的是 a2a_conversations 日志，不是完整 TG session transcript
