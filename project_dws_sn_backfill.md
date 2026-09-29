---
name: dws-sn 宽表补跑流程与依赖链
description: 漏跑 dws-sn 时怎么补——依赖 crawler-sn 先补源数据；RUN_DATE 逐日跑；两类表的差异
type: project
originSessionId: a6075795-7a57-400c-9bbd-845f60122ff6
---
用户日常在 `dws-sn/` 跑 `bash run.sh` 生成宽表。漏跑后补数**必须先看上游**：

**依赖链：crawler-sn（抓数）→ dws-sn（宽表）。两者写同一个库 `dlsc_shaanxi_auto`。**
dws-sn 的 7 个 job 全部读 crawler-sn 落下的 `dws_*` 源表。所以"漏跑一天"往往意味着**那天 crawler-sn 也没跑**，源表有洞，直接跑 run.sh 会静默少数据（inner join 直接丢日期，job 仍报成功）。

**排查源表有没有洞**：查各 `raw_*` 表的 `DISTINCT trade_date`。crawler-sn 的 `trade_date` = 执行日，所以某个 trade_date 在所有 raw 表里都缺席 = 那天爬虫没跑。

**补跑顺序**：

1. **先补 crawler-sn**。backfill 模式会把 `fore_date` 塌缩成 `trade_date`（忽略 `day_offsets`），所以它**无法直接重建"某天发布的 D-1 预报"**。要填源表 `date=X`，就 backfill `--start-date X --end-date X`：
   - `日前电价` off=[0] → `dws_da_price_d@X`
   - `机组状态` off=[0] → `dwd_unit_status_d@X`（capacity_output 的输入）
   - `负荷预测日`/`非市场总出力机组预测`/`电力电量供需平衡预测`/`省间联络曲线输电预测`/`水电总出力预测`/`发电总出力预测` off=[1] → `dws_pred_*_d@X`
   - 用 `--interface NAME`（可重复）精确限定，避免无谓覆盖历史。`--dry-run` 先验证能返回数据。
   - 真实时间价 `dws_rt_price_d@X` 要等 X+1 06:00 才发布，查不到是正常的。

2. **再逐日跑 dws-sn，从旧到新**：`RUN_DATE=YYYY-MM-DD bash run.sh`（`RUN_DATE` 覆盖 `get_run_date()` 的 D）。**必须每个漏跑日都跑一次**，因为：
   - 训练集类表（`da_price_wide` / `da_rt_spread_wide` / `price_forward_wide` / `bidding_space_wide` / `capacity_output`）从 `get_max_date()` 自动续增到 `d(0)` 或 `d(-1)`，单跑最新一天就能补齐历史洞。
   - **但 `pred_price_feature_wide_d` 和 `pred_price_feature_forward_wide_d` 是 as_of/D+1 驱动的**（前者取 `d(+1)`，后者按 `as_of_date` 分区写 D+2~D+5），跳过的执行日不会自动补上。漏几天就必须跑几次。

**Why:** 用户的口径是"那天该有的数据"。源表的洞不补，宽表就静默缺行；而 as_of 驱动的表不逐日补，会永久缺那几天的推理集切片。

**How to apply:** 用户说"这两天没跑 run.sh / 帮我把数据补全"时，先按上面第 1 步查并补 crawler-sn 源表，再按第 2 步从旧到新逐日 `RUN_DATE=... bash run.sh`。跑完抽查各宽表的 `DISTINCT date` 是否连续、以及新日期的关键列有无意外 NULL（`bidding_space_wide_d` 里 D 日的 `rt_price`/`da_rt_spread` 为 NULL 是设计如此）。
