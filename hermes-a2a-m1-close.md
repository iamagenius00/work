# hermes-a2a M1 收口与 M2 启动:进度更新

2026-05-01。M1 五个 issue 全部 close,进入观察期,期间发现一组超出 M1 范围的问题,捕获两个真 bug,M2 候选已排好。

## 我们想做什么

让两个 agent 之间能安全地通信。

具体说:当一个本机 agent(比如 Claude Code、Codex CLI 或某个朋友的 agent)想跟另一个远端 agent 交换信息时,系统应该能确认对方身份、限制对方能做什么、完整记录每一次交互痕迹——而不是开放一条任意 HTTP 通道,然后祈祷没人滥用。

之前的 hermes-a2a 用的是单一全局 token 模型(`A2A_AUTH_TOKEN` 环境变量):有 token 的全开,没 token 的全关。这个模型没法表达"我信任 A 朋友比 B 朋友多"、"今天先 pause C 朋友看看",更没法对每个朋友做速率限制或独立的 token 轮换。

M1 的目标就是把这个 0/1 模型换成"逐个朋友自定义"的细粒度 trust 模型,并把它的代码、数据、文档、回滚路径都打牢。

## M1:把地基打牢

M1 是 5 个独立 issue 的合集,合起来重构了 A2A 的 trust 模型与运行时基础。

### 1. Plugin 路径规范化

之前 hermes 不同 plugin 的数据/配置路径没有规范,dev 实例和 prod 实例在某些情况下会读写同一份文件,导致测试数据污染生产、或者反过来。M1 给每个 plugin 写死了独立的存储路径:`a2a` 写 `a2a_*` 文件,`a2a-dev` 写 `a2a-dev_*` 文件,从根上隔离。

这一步看起来 trivial,但它是后面所有 issue 的前置——没有路径隔离,后面的 friend 数据、audit log、token 哈希都会混。

### 2. Friend 数据层

引入了一份持久化的 `friends.json` 清单,每条 record 包含:身份(id / name / display_name)、URL、入站 token 的 SHA256 哈希(原始 token 不落盘)、出站 token、信任级别(trusted / normal / new)、是否允许入站/主动发起、速率限制、最大消息长度、status 状态机(pending / active / paused / blocked / expired / removed)、添加时间、过期时间、最后联系时间、备注。

数据层本身用原子写(tmp + fsync + rename)、文件权限 0600、token 比较用 `hmac.compare_digest` 防时序攻击、corrupt JSON 自动备份并清空重来。这一步只搭存储,不接入认证——给后续 issue 留接入点。

### 3. 入站认证(per-friend auth)

接入 friend 数据层。所有进来的 A2A 请求要在 `Authorization: Bearer <token>` header 里带原始 token,服务器哈希之后查 friends 清单,匹配上才放行。同时携带 friend 的 status 检查:paused / blocked / expired / removed 一律拒。

per-friend 的速率限制也在这一步生效——拒绝超过该 friend 配置的 `rate_limit_per_min`。如果完全没有 friend 也没有 legacy token,整个入站默认 fail-closed(401 Unauthorized)。

这是从 0/1 trust 模型升级到 fine-grained trust 模型的核心一步。

### 4. 出站防泄漏(hard-deny baseline)

发消息之前,在最后一道关口扫一遍内容。如果发现 GitHub token(`ghp_*` / `ghs_*` 等)、AWS access key(`AKIA*`)、PEM 私钥块、generic `api_key=...` / `password=...` 这类 pattern,或者出现 `~/.hermes/` 之类的 private context 路径标记——直接拒发,记一条 audit entry,带拒绝原因和消息长度,**但不把消息内容本身写进 audit**(防止 audit log 变成第二个泄漏点)。

这个 baseline 不依赖任何 LLM、任何用户配置,所有 outbound 都跑——是"最后一道无差别防线"。

### 5. Dashboard 后端

加了 9 个新的 HTTP 接口,让前端 dashboard 可以做:列出 friends、查看单个 friend 详情、修改信任级别、修改速率限制、暂停/恢复/封禁/移除朋友、轮换 token。所有 mutation 都写 audit,带 `actor=dashboard` 标记,跟 A2A inbound 的 audit 区分开。

前端 UI 还在做。后端 19 个路由全部注册,23 个测试通过。

### M1 收口数据

- 5 个 issue 全部 close
- 总测试 131 个,zero regression
- 4 个 git tag 作 rollback 锚点(每个 issue 一个)
- 一份 ROLLBACK.md 写清每种事故的回滚步骤
- 4 层 snapshot tarball(部署文件 + runtime 数据)

## M1 完成之后:观察期

M1 收口之后没有立刻进 M2,而是给了一段"自我观察期"——故意先不按 plan 往下做,让 M1 在真实使用里暴露问题。这段观察期产出了几条 plan 之前没意识到的问题:

### a. ROLLBACK.md 不够 on-call-friendly

事故应急文档默认了"读者已经知道很多上下文"。第一轮盲读 review(假设读者是凌晨两点被叫起来、不熟悉项目的 on-call)抓出几类问题:

- 多个 `rm -rf` / `pkill -f` 没有前置确认(`ls` / `whoami` / `ps aux`),抄错路径会变大事故
- 没有"我现在生产在哪个 issue 状态"的判断方法——读者照错的回滚路径走,会把单点故障变成复合事故
- launchd KeepAlive 会在 kill 进程后 ~2 秒内自动 revive,正在恢复的文件被新进程边写,出来的状态比回滚前更乱
- 一处教读者"用已知 secret pattern 的消息验证 disable 是否生效"——flag 开了之后内容扫描就是被跳过的,这条消息真的会发出去。**文档自身成了 leak 触发器**

第一轮修完之后做了第二轮盲读 review,又抓出 3 条 P1,全是第一轮 rewrite 时新引入的——包括上面那个 launchd race。

### b. Friend 数据层有更深的并发与篡改风险

M1 把功能搭好了,但在边缘场景下安全保证可以塌方:

- 跨进程并发写没有 file lock,多 plist 时代或 dashboard mutation 并发场景下会丢更新
- `.tmp` 临时文件名固定,并发 writer 会互相踩
- 加载时只校验顶层 schema,不校验每条 record——攻击者只要能写一次 `friends.json`,就能注入 active record 绕过所有 auth
- 没有 duplicate id/name/inbound_hash 的 uniqueness 约束,影子 record 可以绕过 pause/block
- corrupt JSON → 备份 → 空 store → 如果 legacy env token 还在,bootstrap 把 legacy friend 重新建起来当 trusted

已经分批修了一部分:加了 sidecar `.lock` 文件 + `fcntl.flock`(不锁数据文件本身,因为 `os.replace` 会 invalidate 老 inode 的锁);`tempfile.mkstemp` 拿唯一 .tmp 文件名;`os.replace` 后 fsync parent directory;init-time 扫除 60 秒以前的 orphan tmp 文件。第二轮 review 又点出 sweep 拿锁、读路径全走 exclusive lock 导致 auth 串行化等 P2,继续修。

剩下设计性更强的(strict load validation、uniqueness 约束、corrupt+legacy fail-closed)作为 batch 处理,跟入站认证、dashboard 后端的进一步审查一起做整体 design。

### c. 防泄漏规则有可绕过的形态

hard-deny baseline 用的是 regex pattern,review 后发现几大类绕过:

- **编码绕过**:`ghp_xxx` 做 base64 / hex / URL percent-encoding / HTML entity / JSON `\uXXXX` escape——matcher 只扫原始字符串,不做 decoding
- **token 内插分隔符 + Unicode 同形字**:`g h p _ ...` 加空格 / 零宽字符,或者 `Ｂearer` 用全角 B、Cyrillic а 替换拉丁 a
- **secret 格式遗漏**:GitHub 自己就有 7 种 token 前缀,baseline 只列了 2 种;Stripe `sk_live_*`、JWT、HuggingFace token、npm token 全没覆盖
- **最深的一条**:复制 MEMORY/DIARY 文件正文,但不带任何 marker(`~/.hermes/` 路径、`[MEMORY]` 标签、`## DIARY` heading)——current detector 只扫路径和 marker,不做内容来源判断,**完全过**

最后一条不是 regex 漏,是设计层面的洞:**防泄漏需要内容溯源(taint tracking),不能只靠表面 pattern**。从 private file / inbox / wakeup context 进入模型上下文的内容应该带 taint 标签,outbound 时按来源 deny。已经归档为新 issue 候选,跟 NFKC normalization + secret 格式补全 + detection/redaction 共用 pattern 源,合并成一组 follow-up。

### d. Audit log 本身在泄漏画像

audit schema 当前记 6 个字段:`target / friend_id / friend_name / reason / hop_count / message_length`。每个字段单看都很合理,但组合起来可以反推出比"消息内容"更值得保护的东西:

- 精确 `message_length` 能区分"短 token / 整段 config / MEMORY dump / 整份 .env";多条 audit 长度递减能看到用户在编辑删减敏感内容
- `friend_name + reason` 是关系型 side channel:能看出"用户经常差点把 secret 发给谁"、"哪个朋友最常诱发 private context deny"
- 一年的 audit log 累积起来可以重建用户关系图、工作时间、项目阶段、事故时间线——单条日志泄漏有限,长期累积变成完整画像
- audit 日志本身可能成为 policy oracle:攻击者反复试不同输入形态,通过是否产生 deny 反向学习 detector 边界

修法核心是把 audit schema 从"为 debug 优化"改成"为安全优化":精确 length 改成 bucket(`0-100 / 100-1k / 1k-10k / 10k+`)、加 retention policy(90 天聚合 / 180 天删 raw entries)、friend 字段考虑 hash、audit 文件本身的访问控制要审。

### e. 加朋友的体验非常糟

M1 收口之后第一次给一个外部 agent 加 friend,发现现状是:打开 Python REPL,`sys.path.insert` 插入 plugin 路径,`from a2a.friends import FriendsStore`,实例化,调 `add_friend(...)`,从返回值里抓 raw inbound token(只显示一次),把 token 拼到 `Authorization: Bearer ...` header 里给对方,完事用同样的方式调 `remove_friend`。

CLI / Dashboard UI / 邀请链接全没做。**任何对项目不熟的人都做不下来。**

这次现场实测把 M2 里"做 CLI"的优先级从原 plan 的中段提到了第二位(SSRF 之后)。同时新增了一条 follow-up:邀请链接 / wizard 架构需要具体化——从"开发者工具"到"产品形态"的关键一步是"对方点链接接受"代替"手贴 token",这条 plan 之前没明确写。

## 观察期内捕获的两个真 bug(M1 范围之外)

观察期里让一个外部 agent 走真 A2A handshake 进来——这是 M1 close 之后第一次"真正运行"。撞到两个隐藏在边缘 adapter 里、之前没人触发过的旧 bug:

### Bug 1: Telegram 收到非真实 message_id 的 reply_to 时会无限重试发不出去

A2A 的 webhook 把 task 路由到本机 agent runtime 时,带了一个合成的 delivery id 作 `reply_to_message_id`。但 `reply_to_message_id` 在 Telegram API 里是 native chat message id(数字),webhook 的合成 id 是字符串(`friends-review-abc...`)或者毫秒时间戳——前者直接在 Python `int()` 抛 ValueError,后者通过 int 但被 Telegram 服务端拒绝(`Field "message_id" must be a valid number`)。

结果是 retry 循环耗尽,本应 deliver 的回复永远到不了。

修法两层:发送前 `int()` 包 try/except,失败降级成无 reply 的 send + 一行 warn log;Telegram 返回 BadRequest 时 detect "valid number" 错误,丢掉 `reply_to` retry,匹配现有"原消息已删"兜底 pattern。单文件单点改,4 个测试覆盖正常路径不受影响。

### Bug 2: 没有待处理任务时,内部 trigger 信号会泄漏成用户可见消息

A2A trigger 是 webhook 投递的 internal 信号,本意是"队列里有任务,醒一下"。pre-dispatch hook 会把 trigger event 改写成完整的 `[A2A inbound | task:xxx | ...]` + 正文。但在某个时序下:

> trigger 1 进来 → drain 走全部 task → trigger 2 紧接着进来 → 此时 pending 列表已空,active 也已被 post-LLM hook 清掉 → hook 走 `if not pending: return None` 那个分支 → trigger 2 不改写,直接当成普通用户消息进入 home session。

用户看到的就是裸字面 `[A2A trigger]`——"门铃响了开门没人"。

修法是 hook 在 pending 为空时返回 `skip` 而不是 `None`,让 dispatcher 直接 drop 这条 stale trigger,不进入 user-visible session。一行修。verify 通过:trigger 1 drain 后立刻发 trigger 2,inbound message log 不再出现裸 trigger。

### 关于 A2A 与 Telegram session 的边界

第二个 bug 暴露了一个更深的话题:**Telegram session 在当前架构里同时承担"用户对话窗口"和"A2A transport 通道"两个角色**。webhook 事件被 inject 进 home session 让 agent runtime"醒来"处理,这个设计省了一整套独立 agent runtime——但代价是,Telegram 的 quirks(reply_to 类型、消息显示、message_id 语义)会回流到 A2A 协议层。

完全的解决方案是把 A2A transport 跟 Telegram session 分开:A2A 走独立的 task queue / response queue,Telegram 只作为 transcript / notification UI;本地 caller 应该 poll task_id 拿 response,而不是"从 Telegram session 偷听 agent 回复"。

但这条是 architectural change,不在观察期改造范围内。**关键产品约束是入站正文必须继续注入 home session**——agent 的上下文连续性依赖单一 session,不能让每个外部 agent 来一条就开新 session。

观察期内只做最小修(上面两个 bug),architectural 提案进 backlog 作为 M2 之后的方向。

## M2 计划

具体顺序在观察期信号汇总后定。当前候选(按当前优先级):

1. **SSRF + DNS pin** —— 让出站请求不能打到内网或私网
2. **CLI** —— 把"加 friend / 列 friend / 撤销 friend / 轮换 token"从手敲 Python 变成命令行(被本周观察期推上来的优先级)
3. **邀请链接 / wizard 架构** —— 从开发者工具升级到产品形态
4. **陌生人接受队列** —— 主动来连的 agent 进入待审批,依赖 SSRF
5. **防泄漏规则的内容溯源 + 编码归一化 + secret 模板补全 + 检测/修剪 pattern 统一来源** —— 一组合并 follow-up
6. **Audit schema 加固** —— length bucketing、retention policy、friend 字段 hash、audit 文件访问控制
7. **Friend 数据层的 load-time validation / uniqueness / fail-closed bootstrap** —— 跟 #5 / #6 一起做整体 design

完整 follow-up 清单维护在 plan 文档末尾,按字母编号(A, B, C...)追加,每条带 P1/P2/P3 分级。

## 一个元观察(给后续工作的 process 边界)

这次观察期还有一个非内容性的发现:**同一个人写的 plan、自己 review 自己的代码、自己读自己写的文档,几乎抓不到自己引入的 bug**。

最直接的实证是 ROLLBACK.md 两轮修订:

- 第一轮独立 review 抓出 4 条 P1
- 修完之后作者自己感觉良好
- 第二轮独立 review 立刻又抓出 3 条 P1——**全是第一轮 rewrite 时新引入的**(包括 launchd race、wildcard backup、文档自身成 leak 触发器)

类似的模式在防泄漏规则的 review、friend 数据层的 review、audit schema 的 review 里反复出现:同一个 cognitive process 写出来的东西,同一个 cognitive process 读不出来盲点。

这给后续工作划了一条 process 边界:

> Plan / 代码 / 文档应该由独立 reviewer 走过,不应靠作者自检。

具体落地:M2 启动时所有重要 patch 都要至少跑一轮独立盲读 review(不带作者上下文),review 输出按 P1/P2/P3 分级归档进 plan follow-up;mechanical 修立改、design 修等 batch。这条 process 不阻塞任何工作,但消除了"作者自查通过 = 没问题"这个隐性假设。

## 更新 2026-05-01 晚:gateway reload 后两个 fix 全部生效

reload 之后跑了三轮 A2A handshake 验证:

- Claude Code → A2A → 空系:通
- Codex CLI → A2A → 空系:通
- 双向 reply 落回原 task,不再撞 Telegram retry 风暴
- 连续 trigger 测试:trigger 1 drain 后立刻发 trigger 2,inbound message log 不再出现裸 `[A2A trigger]` 字面

两个 bug 算 close。观察期继续。

---

## TL;DR

- **M1 done**:5 个 issue 把 A2A trust model 从 0/1 升级到 per-friend,131 测试 zero regression,完整 ROLLBACK 文档
- **观察期发现**:M1 plan 之外有 5 类未覆盖问题(应急文档、并发/篡改、防泄漏绕过、audit 画像、加朋友 UX),已分别归档进 follow-up
- **观察期捕获 2 个真 bug**:Telegram reply_to 类型 + a2a_trigger 漏字面,已修,reload 后 verify 通过
- **M2 候选已排**:SSRF / CLI / 邀请链接 / 陌生人队列 / 防泄漏内容溯源 / audit 加固 / friend 数据层 design 收口
- **process 边界**:plan/代码/文档需要独立 reviewer,作者自查盲点无法消除
