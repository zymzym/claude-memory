---
name: 记忆仓库（跨机同步）
description: 项目记忆目录本身是一个私有 git 仓（zymzym/claude-memory），工作机与家用机之间靠 commit/push/pull 同步
type: reference
originSessionId: 65170c82-5665-4c3b-a87f-564ca86d5b3d
---
`~/.claude/projects/D--auto-pipeline/memory/` **本身就是一个 git 仓**：`origin = git@github.com:zymzym/claude-memory.git`（私有，`main` 分支，2026-09-29 建）。

**Why:** 用户有两台机器（工作机 + 家里各一个 Claude Desktop），想跨机延续同一批会话的记忆。桌面端会话 transcript **不跨机同步**（日志里只有 `[SkillsPlugin] skills sync`，无 session sync），claude.ai 登录也搬不过去（`oauth:tokenCache` 是本机 DPAPI 加密），所以只有「代码走 `auto_pipeline` 仓 + 记忆走这个仓」是可复制的部分。

**How to apply:**
- 每次写完/改完记忆文件，提醒用户提交：`cd ~/.claude/projects/D--auto-pipeline/memory && git add -A && git commit -m "..." && git push`；在另一台机器开工前先 `git pull`。
- 家用机的记忆目录必须放在**该机 harness 自己算出的路径**下（由 Windows 用户名 + 项目路径决定 slug），路径可能和工作机不同；让用户直接问家里的 Claude「你的 memory 目录在哪」再 clone 进去。
- 记忆仓只覆盖 `D--auto-pipeline` 这一个项目；`~/.claude/projects/` 下另有 8 个项目的记忆目录（`D--my-product-*`、`D--clean-space-*` 等）**未纳入同步**。
- 仓是私有的，但含半敏感信息（飞书 APP_ID/chat_id、内网主机名、3p 网关地址）——**别改成 public**。
- 代码远程仓是 `git@github.com:zymzym/auto_pipeline.git`，工作分支 `crawler-failure-taxonomy`（已推 origin）。
