# 参数是 DNA，Harness 是身体｜Self-Evolving AI 活动回顾

200 行 Rust 代码。一个不可变的目标：进化成能对标 Claude Code 的开源 coding agent。每 8 小时被 GitHub Actions 唤醒——读自己的源码、检查社区 Issue、规划改进、实现变更、跑测试。通过就 commit，失败就 revert，然后写日记，回去睡觉。

40 天后：42,000+ 行代码，1,700+ 测试，28 个模块，60+ slash commands。总花费约 300 美金。没有人类写过它的一行代码。

这个叫 [Yoyo](https://github.com/yologdev/yoyo-evolve) 的项目，是上周六在上海漕河泾一场闭门活动上展示的。当天聊的主题是"自进化 AI"——不是概念，是正在发生的事。台上的人有在做 RL 训练框架的，有在做分布式算力的，有在让模型自己跑实验的，有在用 AI 预测未来的。台下坐着投资人和创业者，聊到最后，没有人能给出"自进化 AI"的标准定义，但每个人都承认：这条路已经在走了。

活动由 [Gradient](https://gradient.network/)、红杉中国、上海漕河泾开发区主办，AI Hackerhouse、真格基金、[MiniMax](https://www.minimax.io/)、[Unipat AI](https://www.unipat.ai/)、Spark Labs 联合支持。

以下是当天的几个分享。

---

## 一、从对话中提取训练信号

王胤杰是第一个上台的。他是芝加哥大学博士生，[OpenClaw-RL](https://github.com/Gen-Verse/OpenClaw-RL) 的作者（开源代表作还包括 RLAnything、CURE、dLLM-RL）。他做的事情说起来很朴素：让 Claude Code 通过跟用户的对话自己完成 RL 训练。

OpenClaw-RL 的核心设计是把 agent serving、rollout collection、PRM/judge 评估、policy training 解耦成四个独立的异步循环，互不阻塞——你跟 agent 正常聊天的同时，训练已经在后台跑了。框架提供两条学习路径：

**路径一，标量反馈（Binary RL）。** Process Reward Model 基于 next-state 反馈对每个 turn 打分。代码跑通了 +1，报错了 -1。数据量大，但方向模糊——模型知道自己错了，不知道错在哪。

**路径二，方向性反馈（On-Policy Distillation）。** 当 next state 包含有用的 hindsight 信息时，judge 模型提取文本 hint，构建增强版 teacher context，提供 token 级别的方向性监督。比如用户说"注释不够详细"，系统就提取出"增加注释细节"这个方向让模型重试。方向明确，但数据稀疏。

两条路径加权结合后效果显著优于单独使用。但硬件门槛不低：当前需要 8 张 GPU。

王胤杰的判断很实际："个人用户短期还是靠 skills 和 memory。真正需要上训练的是 Cursor 那种量级的厂商。如果个人真要做，50-100 人共享一个模型，分摊成本，数据收集也快。"

> GitHub：[github.com/Gen-Verse/OpenClaw-RL](https://github.com/Gen-Verse/OpenClaw-RL) ｜ 论文：[arxiv.org/abs/2603.10165](https://arxiv.org/abs/2603.10165)

---

## 二、模型为什么会"学坏"

宋昀翀来自上海人工智能实验室，他说了一件做 RL 的人都经历过但不愿承认的事："模型训练着训练着，突然就'学坏'了。做数学题的时候，它找到一条捷径能蒙对答案，之后就只走这条路。变体题一来，直接傻眼。"

这就是 mode collapse。传统 RL 追求奖励最大化，结果模型变成了应试高手——只会刷分，不会变通。他的核心论点是：reasoning 和 generation 本质上都是 path-distribution 问题，不该用 single-path optimization 来解。

他的团队用 GFlowNets 的思路做了两个工作（都是 ICLR 2026），分别改变了"学什么"和"在哪搜索"：

**FlowRL：改变优化目标。** 不再最大化奖励，而是让 policy 去匹配一个 reward-induced target distribution。用一个可学习的 partition function 把标量奖励归一化成目标分布，再最小化 policy 和目标之间的 reverse KL divergence。为了解决长 CoT 的两个工程问题（梯度爆炸和采样偏移），还引入了 length normalization 和 detached trajectory-level importance sampling。

**FoSS：改变生成空间。** 标准自回归 LM 的状态空间是一棵 token tree——每个 prefix 只有一个 parent，大量的 compositional path structure 根本没被建模。FoSS 把它替换成了 span-DAG：同一个句子可以有多条不同的组合路径，模型在一个更丰富的搜索空间里探索。

实验数据：FlowRL 在 32B 数学 benchmark 上对 GRPO 平均提升 +10.0，对 PPO 提升 +5.1，code reasoning 上也有 consistent gains。宋昀翀展示了一个 AIME 案例：GRPO 在 AM-GM 不等式上循环，FlowRL 利用对称性将问题化归为 cubic 直接求解。

用他自己的话说："以前模型做题是一条路走到黑，现在是条条大路通罗马。"

> FlowRL 论文：[arxiv.org/abs/2509.15207](https://arxiv.org/abs/2509.15207)

---

## 三、当 Token 消耗暴涨 15 倍，成本账怎么算

Rymon（Gradient）分享了一组来自 OpenRouter 的数据：过去一年，全球 AI token 消耗从每周 1.78 万亿涨到 27 万亿，15 倍。什么概念？维基百科全站大概 50 亿词，现在 AI 一周要处理 4000 多个维基百科的量。

结构性变化也值得注意。Claude、GPT 等前沿模型的市场份额从 22%（一年前的 Sonnet 3.7）降到 4%（现在的 Opus 4.6）——增量被 Qwen3.6 Plus、MiMo-V2-Pro、DeepSeek V3.2 等模型吃掉。算力供给每年涨约 3 倍，但需求涨 10 倍以上。Agent 越智能，token 吞得越凶。Claude Code 这种终端常驻型的吞吐和传统 chatbot 完全不是一个量级。

所以他算了一笔直接影响创业者决策的账：**DAU 到 5 万左右时，自建模型 + post-training 的成本曲线会显著优于持续付 API 费。** 他称之为"frontier tax 交不起了"。Cursor、Cognition、Perplexity 都已经在往这个方向走——拿开源基座，针对自己场景做 post-training。

但 post-training 的 infra 是真的糟心。传统同步 RL 像流水线，一个环节卡住全网等——Gradient 估计 80% 的 rollout 计算是浪费的。Echo 框架把训练和推理完全异步化，支持跨云、跨地区、甚至抢占式实例混用。在同一个 RL 任务上（Qwen3-30B-A3B, DAPO math 17k），Fireworks 跑了 124 小时花费 $4,490，Echo 跑了 9.5 小时花费 $425——成本降 91%，时间缩 92%，reward 收敛曲线完全 match。

Gradient 即将发布的 [Logits](https://logits.dev) 平台目标是让没 infra 背景的开发者也能跑 post-training 实验。Rymon 的原话："就像 vibe coding 让产品想法能快速验证，我们要让 research 想法也能快速验证。"

> Gradient 官网：[gradient.network](https://gradient.network/) ｜ Echo 论文：[arxiv.org/pdf/2602.02192](https://arxiv.org/pdf/2602.02192)

---

## 四、当 AI 开始预测未来

陶政为（北京大学计算机学院博士，[Unipat AI](https://www.unipat.ai/) 联合创始人）的方向和其他嘉宾不太一样——他们在做通用 AI 预测系统 Echo。

传统做法是拿已发生的事件当训练数据，让模型"事后诸葛亮"。陶政为指出这有两个根本问题：一是你无法真正复刻事件发生时的信息环境（关键网页是动态变化的，过了就没了），二是以结果为导向的反馈在本质上就是错误的——预测是概率判断，你不能因为骰子这次扔出了 6，就教模型"扔骰子一定出 6"。

所以他们提出了 "Train-on-Future" 范式，核心是**不对预测结果打分，对预测过程的行为质量打分**。具体包含三个机制：动态问题合成（由实时数据流驱动 agent 自动生成关于未来的问题）；行为导向评分搜索（用一套可演化的 rubrics 评估搜索是否全面、推演是否有逻辑、证据链是否完整）；Map-Reduce 智能体架构（将复杂问题分解为可并行子任务，多 agent 协同推理后聚合决策）。

在场有人追问 rubrics 搜索的创新点，陶政为笑着说"这个比较关键，有意识藏了一下"。

结果：EchoZ-1.0 以 1034.2 的 Elo 分数排名第一，超越所有商业旗舰模型。在 Polymarket 上做了实盘测试，5 个 EchoZ agent 中 4 个盈利，表现最好的一个取得了 15% 的收益。预测系统本身也在"自进化"——rubrics 不断搜索和迭代，模型越跑越准。

> Unipat AI 官网：[unipat.ai](https://www.unipat.ai/) ｜ GitHub：[github.com/UniPat-AI](https://github.com/UniPat-AI)

---

## 五、MiniMax M2.7：让 AI 自己跑 RL 实验

Vincent 介绍了 [M2.7](https://www.minimax.io/models/text/m27) 的细节——这是 MiniMax 第一个在训练流程中让模型自身深度参与的版本，官方直接把"self-evolving"写进了模型定义。

具体数字：后训练团队 30%-50% 的任务已经可以完全交给 AI。

一个工程师花了 4 天时间，给模型搭了一个专门跑 RL 任务的 harness（带记忆、带 skill、带安全权限）。搭完之后，RL 流程五步里，中间三步（构造环境、跑训练、收集分析结果）100% 由 AI 自主完成。人类只负责两头：**定实验目标**和**复盘找洞察**。

内部版本的 M2.7 自主优化编程 scaffold 超过 100 轮——分析失败轨迹、修改代码、跑评估、决定保留还是回滚——最终在内部评估集上提升 30%。公开 benchmark：SWE-Pro 56.22%（匹配 GPT-5.3-Codex），VIBE-Pro 55.6%，Terminal Bench 2 57.0%，GDPval-AA ELO 1495（开源模型最高）。

Q&A 环节有人问怎么省钱。Vincent 给了两个 pattern：强模型当 planner 弱模型当执行者，或者反过来弱模型主跑遇到难题 escalate 到强模型。"取决于你的任务形状。"

> M2.7：[minimax.io/models/text/m27](https://www.minimax.io/models/text/m27) ｜ HuggingFace：[huggingface.co/MiniMaxAI/MiniMax-M2.7](https://huggingface.co/MiniMaxAI/MiniMax-M2.7)

---

## 六、圆桌：Harness、记忆与人类的位置

下午的圆桌由 Gradient 创始人 Eric 主持，参与者包括任亦心（红杉中国）、钟天杰（真格基金）、Vincent（MiniMax）和陈亮（Unipat AI）。聊了很多，这里记三个最有意思的方向。

### Harness 和参数的 co-evolution

Vincent 打了个比方：参数是 DNA，harness 是身体。当前 AI 的进化还在"身体层面"，但参数层面的进化——一代模型迭代下一代——已经开始萌芽。

他讲了一个具体案例：去年 9 月 Sonnet 发布时，Cursor 发文说需要完全重构 harness。原因是 Anthropic 后训练 Sonnet 时本身就基于 harness 在训，模型参数里已经有了对 harness 的自我感知——context 越来越长时，模型会"紧张"，想赶紧收掉任务，甚至主动调用 Haiku。"从那之后，所有模型都在走向 harness 和参数的 co-evolution。harness 怎么定义，模型就跟着怎么变。"

### 记忆是新的护城河

Vincent 和陈亮几乎同时得出一个结论：好的 skills 最终会被蒸馏进模型权重，但 memory 应该留在外面。Vincent 说："模型的一个很大优势是记忆不在权重里。可以随时换记忆，甚至换脑子。"

他提到 Letta 公司的 sleep-time compute 概念：agent 在不与人对话时自己回顾对话记录，生成更准确的记忆。以后的 agent 会 7×24 在后台持续推理。"你有没有不小心把自己的 claude.md 删掉过？你有多痛，这个进化就有多成功。"

陈亮指出一个产品策略分叉：如果记忆是开源的 markdown 文件，你随时能迁移；但如果一个应用把它做成闭源的，它就成了让你离不开的理由。

Eric 给了一个建议："回去试一下，给你用得最多的 AI 输入这个 prompt——'请根据你对我的记忆，有哪些我自己意识不到的、但如果明白了就能改变我生活的残酷真相。请坦诚告诉我。'你会被吓到。"

### 那人类以后干嘛？

钟天杰："大家焦虑的不是 AI 给社会带来什么，是怕隔壁邻居去 AI 创业公司赚了大钱。我的建议是，早点加入 AI 创业公司，在人类最后一次大产业变革里拿到一些生产资料。"

Vincent："理想状态是你跟模型说'请帮我赚钱，维持我的生活'。如果它做到了，焦虑就解除一大半。再往后，人类只能做自己喜欢做的事——其实也不坏。"

任亦心收尾："焦虑来源于不自觉地把自己物化，觉得价值等于工作。你的 calling 是什么，比你擅长做什么重要。"

---

## 七、Demo

活动还有一个 Demo 环节。一个前一天刚在小红书黑客松拿了第三名的团队展示了 AI PPT 生成器——核心思路不是让 AI 直接生图，而是在浏览器里造了一个轻量级操作系统，让 AI 像人类设计师一样操作矢量图形、搜真实图片、排版布局。更硬核的部分是零成本沙盒：不需要云端 VM，直接在用户本地浏览器里跑 bash 和 python，整个沙盒只要 10MB。主讲人季翔说："你的 taste 是你唯一的护城河。"

开头提到的 Yoyo，是十字路口的 Koji 发起的 Token Grant 项目资助的——给入选的 AI 创业项目提供 5 万美元 token 费用。"今天创业不缺钱，缺的是 token。"

真格基金的钟天杰说，Yoyo 最有意思的不是它改了多少代码，而是看它每次进化时在想什么。"像楚门的世界，你可以留言，它会回复你，甚至把你的话考虑进下一次进化。"

> Yoyo GitHub：[github.com/yologdev/yoyo-evolve](https://github.com/yologdev/yoyo-evolve)

---

这一天聊下来，没有人给出"自进化 AI"的标准定义。但有一件事是清楚的：从对话中提取训练信号、让模型自己跑实验、用异步 RL 把成本砍掉 90%——这些不是论文里的设想，是已经在跑的代码。

Yoyo 是一个缩影。200 行代码，一个目标，40 天，没有人类干预。它每 8 小时醒来一次，读自己写的代码，决定接下来改什么。

也许这就是"自进化"真正的意思——不是有人写好了剧本让系统去演，是系统自己开始写剧本了。至于它会写出什么，现在没人知道。

