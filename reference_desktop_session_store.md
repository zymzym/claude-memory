---
name: Claude Desktop 左侧会话的存储与跨机迁移
description: 左侧列表=Claude-3p\claude-code-sessions\<acct>\<org>\local_*.json（仅576K）；正文=~/.claude/projects/<slug>/<cliSessionId>.jsonl（722M）；跨机要拷这两处且 cwd 必须存在
type: reference
originSessionId: 65170c82-5665-4c3b-a87f-564ca86d5b3d
---
**Desktop 左侧会话列表 ≠ 对话内容**，两处拼起来：
- **列表项**：`%APPDATA%\Claude-3p\claude-code-sessions\<accountId>\<orgId>\local_*.json`，每条约 500B，字段 `title`（左侧显示的标题）/ `cliSessionId`（=正文文件名）/ `cwd` / `model` / `isArchived` / `lastActivityAt`。2026-09-29 实测 **229 条 / 共 576K**。Cowork（agent 模式）另存 `local-agent-mode-sessions\<accountId>\<orgId>\`（1.1M / 5 条）。
- **正文**：`~\.claude\projects\<slug>\<cliSessionId>.jsonl`，slug = cwd 把 `:` 和 `\` 各换成 `-`（`D:\auto_pipeline\dws-sn` → `D--auto-pipeline-dws-sn`）。2026-09-29 总计 **722M**，头部集中在 `D--auto-pipeline` **478M**（含 20–33MB 巨型会话）与 `D--clean-space-rolling-backtest` 165M。

**Why:** 2026-09-29 用户有两台机器（工作机 + 家里各一个 Claude Desktop），想回家继续左侧这批会话。会话正文**不跨机自动同步**（日志里只有 `[SkillsPlugin] skills sync`，无 session/transcript sync），只能手动搬。任务中途暂停（用户选「暂时停下」），**未实施任何拷贝**。

**How to apply:**
- 复现左侧列表 + 能续聊 = 拷 (a) 元数据（全拷仅 <2M，列表原样出现）+ (b) 选中的正文 jsonl（按 `cliSessionId` 精确挑，通常几 M）。**别全搬 722M。**
- 三个必须对齐的点：① 目录名 `<accountId>\<orgId>` 要与家里一致——同一 claude.ai 账号才同名（accountId 从 `Claude-3p\logs\main.log` 的 `accountId=` 取；3p 下 orgId 恒为占位 `00000000-0000-4000-8000-000000000001`）；② `cwd` 家里必须存在——229 条里约 127 条指向 `D:\my_product\*` / `D:\clean_space\*` 这批**合并进 auto_pipeline 之前的老树**，家里没有就是点开空壳；③ slug 路径一致。
- jsonl 里含密钥（库配置有飞书 token / DB 凭据 / 网关 key），**别进公开仓**；跨机走云盘或 U 盘。
- 无 `jq`；有 `node`（v22.22.3）可写挑选脚本。
