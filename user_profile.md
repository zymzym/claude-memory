---
name: 用户画像
description: 用户是电力现货市场交易员（陕西 SGCC），维护 D:\auto_pipeline 下多组件交易流水线
type: user
originSessionId: 37440f79-9aa0-4b5b-953d-702fcf805337
---
电力现货市场交易员（陕西/山西，SGCC 平台），工作语言为中文。每天跑固定任务流（由 daily_pipeline/orchestrator.py 常驻调度）：06:00 morning（实时价回填→宽表→score 回填真值→昨日套利复盘→TOP10 发飞书 swettie）；08:00 wx（气象更新，入库 wx_forecast MySQL 库，不再复制 CSV）；08:40 probe（预报数据齐全即触发 day：日前增量→宽表→eff_re→价差推理 sd+wx × 线性/树模型→明日套利预测）。注意：编排器后台调度随 Claude 会话结束而终止，需 `bash run.sh` 手动常驻。

涉及仓库（均在 D:\auto_pipeline\）：`crawler-sn`（爬虫）、`dws-sn`（宽表）、`spread_forecast_wxfeat`（价差预测）、`declare-alpha-dev`（套利复盘/预测）、`wx-ec`（气象）、`daily_pipeline`（编排器，本项目）。

**Why:** 决定沟通方式——用他熟悉的流程术语（回填/宽表/infer/复盘/申报门控）交流，避免解释他已知的领域概念。
**How to apply:** 改动涉及上述组件时，说明对 06:00/08:00/09:00 链条的影响。日期一律按 Asia/Shanghai（D=运行日，D-1=昨日）。
