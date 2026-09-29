---
name: market_price_board 盘口表位置与档位口径
description: market_price_board 在 dlsc_shaanxi 库；实时价在生产库 dlsc_shaanxi_auto；price_level=1 是最高价档而非"最优价"
type: reference
originSessionId: 826beea1-c050-4073-8008-05a754cd20b6
---
`market_price_board`(中长期撮合盘口快照)**在 `dlsc_shaanxi` 库**,不在生产库 `dlsc_shaanxi_auto`;而实时价 `dws_rt_price_d`(0~95 点、含最新数据)在 `dlsc_shaanxi_auto`。跨库查询用全限定名即可(同一 RDS 实例)。

**盘口结构**:UI 是二维表,行=价位档(卖1…卖s / 买1…买b),列=包1…包N(每档内第 N 笔挂单);DB 只存每档汇总,无"包"维度。列:`side`(1买/2卖)、`price_level`、`price`、`sy_energy`(该档剩余量)、`nlk_type`(小时 1~24,=period_label 00:00-01:00…23:00-24:00)、`delivery_date`(交割日)、`snapshot_time`(约每 5 分钟一个快照,单(交割日,小时)常有 ~200 个)、`trade_date`。

**关键口径坑(2026-09 二次实测,已确证)**:`price_level=1` 是**最高价档**,`price` 随 `price_level` 递增**递减**。sanity check(2026-09-01~07, 12218 个卖盘快照):**99.5% 快照 price_level 升→price 非增**;`level1 > level2` 占 63.1%,`level1 < level2` 占 **0.0%**;mean(level1−level2)=+4.53。pooled 相关系数是 +0.18 的正值——那是时间混杂的 Simpson 悖论,勿用 pooled corr 判断方向,必须 **within-snapshot** 比。要"最优卖价"(最低卖价)取每个快照内 sell 侧 `min(price)`。这与 `docs/申报知识库/04_数据字典.md` 里"price_level=深度档位(1=最优)"的表述不符,文档疑似有误。

**UI↔DB 命名风险**:盘口 UI 行名是 卖s..卖1 / 买1..买b(`j.name="买"+A`),UI 的"卖1"若按常规是**最优(最低)卖价**,但 DB 的 `price_level=1` 是**最高卖价**——**两者可能不是同一档**。用户口头的"卖方包1/卖1"务必回问是指 UI 行次还是 DB `price_level`,否则会取到相反的价格。(另:UI 的"包1..包N"是同一档位内的第 N 笔挂单,是 UI 列,DB 无此维度。)

**Why**:用盘口做尖峰时段分析时,口径选错会得到完全不同的数(本次 163 个实时价>500 的点:取 price_level=1 全部快照均价=531.98,取最低卖价=462.43)。
**How to apply**:任何"盘口第1档/包1/卖一"的需求,先确认是 price_level=1(最高价档)还是 min price(最优卖价),并确认快照口径(全部均价/最后/最早)。
