---
name: date_name_rename 分支工作约束
description: 该分支只操作 dlsc_shaanxi_auto_test 库；crawler-sn 是工具层不可修改
type: feedback
originSessionId: 8bd30fcd-4a69-4fe2-b762-2d8b1dd4a7fb
---
在 date_name_rename 分支上，一切爬取/探查/写入只允许操作 `dlsc_shaanxi_auto_test` 库，绝不碰生产库 `dlsc_shaanxi_auto`。

**Why:** 用户明确要求，防止 rename 验证期间误写生产库。

**How to apply:** 任何涉及 DB 的子进程/env 注入（SN_DB_NAME / SPREAD_DB_NAME）在本分支必须指向 dlsc_shaanxi_auto_test。注意 daily_pipeline/config.py 的 DB_ENV 默认是 prod（dlsc_shaanxi_auto），本分支上必须覆盖成 test。

---

crawler-sn 是工具层，不要修改其代码；外部（daily_pipeline 等）只通过 CLI（crawler_service.py run-once）或导入其函数来调用它。

**Why:** 用户明确不希望改工具层，保持其纯净与可复用。

**How to apply:** 新功能（探查/通知等）放在 crawler-sn 之外的模块（如 daily_pipeline/），只调用 crawler-sn 的 CLI 或库函数。

**例外（2026-09-09 用户明确批准）：** 修 crawler-sn 的**核心传输层**（`client.py` 的 SnClient 加 CDP 浏览器传输层，解决瑞数 strict 接口 412）是允许的，用户选的是"改 crawler-sn 的 SnClient"。此例外仅限"工具自身核心功能坏了必须修"，不代表可以往里堆业务逻辑。
