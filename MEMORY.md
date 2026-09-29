# Memory Index

- [用户画像](user_profile.md) — 电力现货交易员，06:00/08:00/09:00 三时段任务流，涉及组件及日期口径
- [Windows 子进程/飞书/桌面端环境陷阱](env_quirks.md) — subprocess 裸 bash 被 WSL shim 劫持；飞书上传 file_type 须放 form；Claude Desktop 数据在 %APPDATA%\Claude-3p；DB 契约 SN_DB_NAME/SPREAD_DB_NAME=dlsc_shaanxi_auto
- [advisor 申报决策顾问](advisor_申报顾问.md) — 每天 9:20 前 LLM 判断是否申报；advisor/ 知识库+简报+判断记录+复盘的工作流与位置
- [调度器后台常驻](scheduler_service.md) — scheduler_watchdog.py 看门狗托管 orchestrator schedule；HKCU Run 自启 + 进程级防休眠 + 僵死自愈；--status/--install 命令
- [date_name_rename 分支工作约束](feedback_branch_constraints.md) — 本分支只操作 dlsc_shaanxi_auto_test 库；crawler-sn 是工具层不可修改
- [陕西数据源发布时刻表](project_data_publish_schedule.md) — 21:00 D+1 日前价 / 6:00 D-1 实时价 / 8:00 日前边界
- [pmos 瑞数 412 的成因与绕行](project_pmos_auth.md) — 两条通道二选一：IPv6 出口（ipv6_prefer.py 钉 socket + trust_env=False）或 CDP 借已登录 Chrome 页面会话；2026-09-12 IPv6 被堵、trade-crawler 已装回 CDP；CurrentRoute 决定 40302
- [dws-sn 宽表补跑流程](project_dws_sn_backfill.md) — 漏跑 run.sh 时先补 crawler-sn 源表，再按漏跑日从旧到新逐日 RUN_DATE=... bash run.sh
- [pmos 价格接口的可回溯范围](project_pmos_price_history.md) — 实时价 2025-01 起才有；日前价 2024 仅 116 天零星；2024 上半年 0 值口径存疑 + 两个占位日
- [盘口表位置与档位口径](reference_market_price_board.md) — market_price_board 在 dlsc_shaanxi（非生产库）；price_level=1 是最高价档不是最优价（99.5% 单调已确证）；包=UI 列、库无此维度
- [尖峰套利的核心是"残差"](project_spike_arb_edge.md) — T-2 判别实时价≥500；纯尖峰概率不稳健，edge=低入场价+高概率；规则 p≥0.08 且卖价≤350
