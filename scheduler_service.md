# 调度器后台常驻：看门狗 + 防休眠 + 自愈

2026-08-11 落地。`orchestrator.py schedule` 不再手动起，由看门狗托管。

## 组成
- **`D:\auto_pipeline\daily_pipeline\scheduler_watchdog.py`** — 看门狗：
  - 拉起/监控 `orchestrator.py schedule` 子进程（CREATE_NO_WINDOW，无窗口）。
  - 进程退出 → 30s 内重启；心跳停更超 5 分钟 → 判僵死 → terminate/kill → 重启。
  - 自身每 30s 调 `SetThreadExecutionState(ES_CONTINUOUS|ES_SYSTEM_REQUIRED)` 防空闲休眠（进程级，改的是电源请求不是全局计划）。
  - 日志 `logs/scheduler_watchdog.log`；心跳 `logs/scheduler_heartbeat.txt`。
  - CLI：`--install` 写 HKCU Run 键自启 / `--uninstall` 删除 / `--status` 查状态 / 无参 = 前台跑。
- **`orchestrator.py`** 新增：`_keep_awake_loop` 保活线程 + `_write_scheduler_heartbeat` apscheduler 心跳 job（每 60s）+ main() stdout None 兜底。

## 关键事实
- 自启走 **HKCU `Software\Microsoft\Windows\CurrentVersion\Run` 键 `AutoPipelineScheduler`**（pythonw 跑看门狗），与 OneDrive/Notion 同机制。**非管理员**，`schtasks /sc onlogon` 被策略拒（Access Denied），故不用计划任务。
- 机器是**笔记本**（Chassis 10），电源「平衡」AC/DC 空闲 15/10 分钟睡眠——保活只挡空闲休眠，**手动合盖/手动睡眠不挡**（用户已知悉）。
- 保活不写全局电源计划，服务停止即恢复。
- 看门狗重启调度器后，apscheduler 对「已过时刻」的 cron 排到下次未来时间（如 08:00 已过则明早再跑）；观测到的僵死多在 morning 后 06:08，距 wx 08:00 有近 2h，能自愈不丢窗。

## 常用命令
```
cd /d/auto_pipeline/daily_pipeline
python scheduler_watchdog.py --status   # 自启项/心跳/进程
python scheduler_watchdog.py --install | --uninstall
```
调度器调度时刻：morning 06:00 / wx 08:00 / probe 08:40 / thermal_rt 15:00 / thermal_da 23:00。
