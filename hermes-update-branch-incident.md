# Hermes Update 分支事故记录

日期：2026-04-20
来源：CC 报告

---

## 经过

1. 跑 hermes update，从 v0.10.0 更新到最新（38 个新 commit）
2. update 自动从 feat/a2a-protocol 分支切到 main
3. PR #11025（A2A protocol support）还没合并到 main，切分支后 A2A adapter 代码消失（gateway/platforms/a2a.py 等 8 个文件）
4. 重启 gateway 后 A2A 端口 8081 不再监听，Telegram 正常
5. 旧 gateway 进程没完全退出，新 gateway 报 "Telegram bot token already in use"

## 根本原因

hermes update 强制切到 main。如果在 feature branch 上有未合并的本地功能，update 会静默移除。stash 只保存未提交修改，不保存 branch 上的 commit。

## 解决

1. git checkout feat/a2a-protocol
2. git rebase main
3. git stash pop（hermes_state.py 有冲突，手动解决——CJK LIKE fallback 两个版本，保留 main 的更完整实现）
4. 重启 gateway，A2A 恢复

## 教训

- update 前先 git branch 确认当前分支
- feature branch 上 update 后需要手动 rebase
- PR #11025 合并后问题消失
- gateway --replace 有 race condition：旧进程没杀干净时新进程启动报 token 冲突，需要手动 kill + 等待

## CC 的额外反思

- 后台任务的副作用：CC 后台跑 update，老师 terminal 也跑了一次，两边操作同一个 repo 导致冲突
- "跑通了"不等于"验证了"：vllm-sr install skill 只跑通 CLI 就差点提 PR，没完整跑过 serve 不该贡献
- 先问"为什么"再解决"怎么"：vllm-sr 一上来就折腾安装，空系一句话点出老师的场景根本不需要这个
