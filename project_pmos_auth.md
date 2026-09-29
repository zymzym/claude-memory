---
name: pmos 瑞数 412 的成因与绕行（IPv6 出口 / CDP 浏览器会话，二选一）
description: pmos.sn.sgcc.com.cn 的 412 是瑞数按源 IP 信誉拦；可用 IPv6 出口，或借已登录 Chrome 的 CDP 页面会话发 fetch
type: project
originSessionId: d27211b1-9c61-4796-b52b-da1731092f74
---
pmos.sn.sgcc.com.cn（陕西电力交易）的 `HTTP 412` 是**瑞数信息(RiverSecurity)动态安全反爬**，按**源 IP 信誉**打分，默认放行、命中才拦。

## 两条可用通道（按当前条件二选一）

| 通道 | 何时可用 | 代价 |
|---|---|---|
| **IPv6 出口** + 静态 cookie + 纯 requests | 本机确实有**可用**的 IPv6 出口时 | 出口一变就全废 |
| **CDP 借会话**：附着已登录 Chrome，在页面上下文 `fetch` | 有常开的已登录 pmos Chrome 时 | 需保活 Chrome + 登录态 |

**⚠️ 2026-09-12 实测：IPv6 出口被堵，CDP 是当时唯一可用通道。**
`pmos` 的 AAAA 能解析（`240e:659:130:1ff::186:1`），但 TCP 连不上（超时）；IPv4 `202.100.20.80:443` TCP 能连（15ms）却被瑞数 412。移动出口（`2409:`）到电信段（`240e:`）整段不通，`ping -6`/`tracert -6` 均中断。所以**别再假设「有 IPv6 就一定能过」**，先实测。

## 结论（2026-09-10/11 实测，推翻旧结论）

**当时真因是出口 IP 协议族，不是 cookie、不是 header、不是 TLS 指纹：**

| 出口 | 结果 |
|---|---|
| IPv4（家宽 / 公司 CGNAT） | **412**（瑞数拦） |
| IPv6（手机热点 移动 IPv6 / 家宽 IPv6） | **200 + 正常数据** |

同一台机器、同一运营商、同一 cookie、同一套 header，只换协议族，结果就翻转。复现矩阵：默认→200 / 强 IPv4→412 / 强 IPv6→200。

**旧结论「多次请求要实时重算滚动 token，纯 requests 过不了，必须浏览器上下文 fetch」是错的**：一个 4 小时前的静态 cookie 快照连打 14 次严防护接口全部成功。瑞数 cookie **不是**每次请求滚动。

## 怎么强制走 IPv6

`requests`/`urllib3` **不做 Happy Eyeballs**：`getaddrinfo` 先返回哪个协议族就用哪个，无法从 requests 层指定。只能在 socket 层打补丁 —— 仓库根 **`ipv6_prefer.py`**（导入即安装，幂等）：

- 把 `pmos.sn.sgcc.com.cn` 的解析钉到 `AF_INET6`，**只在确实有 AAAA 记录时**才用，没有则原样回落（服务器只有 IPv4 时同一份代码不会炸）。
- **补丁函数参数名必须叫 `family`**：urllib3 是按关键字传 `family` 的，改名会 `TypeError: got multiple values for argument 'family'`。
- **必须 `session.trust_env = False`**：一旦请求交给 HTTP 代理（本机 `HTTP(S)_PROXY=http://127.0.0.1:7897`，且 `NO_PROXY` 不含 pmos），是**代理**去解析域名，上面的补丁完全不生效，出口变成代理的、不可控。

已经接上补丁的入口：`crawler-sn/crawler_sn/client.py`（导入即生效，覆盖 crawler-sn 全部接口）、`all_unit_clearing_price/陕西实时电价爬取.py`、`陕西日前电价爬取.py`、`cookie保活0907.py`。

**运维前提：这台机器必须有真正的 IPv6 出口**（手机热点 / 开了 IPv6 的家宽）。只有 IPv4 时全接口 412。快速自检：有没有全局 IPv6 地址（不是只有 `fe80::`）、`ping -6 2400:3200::1` 通不通。`%{remote_ip}` 是**服务端** IP，不是出口，别拿它当出口看。

## 请求头（严防护接口）

| Header | 缺失时 |
|---|---|
| `ClientTag: OUTNET_BROWSE` | 200 但 `401 请求未授权` |
| `Content-Type: application/json;charset=UTF-8` | 200 但 `data:null` |
| `X-Ticket` | 401 |
| `CurrentRoute: <前端路由>?date=<ms>` | 200 但 `40302 您当前没有该接口访问权限` |

路由映射见 `client.py` 的 `_CURRENT_ROUTE_BY_PATH`（值与前端页面路由一致即可，只需合法）。Cookie 本身可选。

## 凭证

统一入口**仓库根 `auth.env`**。**两个票，两套账号**：

| 键 | 账号 | 用于 |
|---|---|---|
| `AUTH_COOKIE` / `AUTH_X_TICKET` | **售电侧** | crawler-sn 全部接口、申报（`queryUserDeclareCurve`） |
| `AUTH_X_TICKET_FA` | **发电侧（发侧）** | 大火电**机组出清**接口（日前/实时 96 点出清量价） |

**机组出清接口只认发侧票**（2026-09-11 实测）：
- 同一 route（`…/spotgoodsProvince/…`）+ 售电票 → 一律 `40301 您当前没有该菜单访问权限`；
- 换成发侧票 → 日前/实时都出 96 点（`Success`），且**不带 cookie 也通**。
- 所以之前 thermal_rt/thermal_da 天天「0 行」不是 route 写错、不是 cookie 过期，是**票的账号不对**。
  route 本身没错，见 `all_unit_clearing_price/*.py` 的 `_CURRENT_ROUTE_BY_PATH`（至今保持 spotgoodsProvince）。

`陕西实时电价爬取.py` / `陕西日前电价爬取.py` 的 `build_headers()` 已改为**优先用 `AUTH_X_TICKET_FA`**、缺失才回落 `AUTH_X_TICKET`。crawler-sn 仍用售电票（够用，且申报那步必须售电账号）。人工从已登录浏览器复制一次即可；cookie 约 7 天过期，两票过期后各自换新写回 `auth.env`。

## 已下线的东西（2026-09-11，仅限 D:\auto_pipeline）

`D:\auto_pipeline` 下的 CDP/浏览器传输层已按用户要求全部移除，恢复「静态 cookie + 纯 requests」：

- `crawler_sn/client.py`：删 `_post_via_browser` / `_ensure_browser` / `close()` / `_CDP_URL`；`config.py` 删 `transport` 字段与 `SN_TRANSPORT`；`pipeline.py` 删 `close()` 调用。
-  orchestrator 删掉每阶段开头的 `_refresh_auth()`（原来是跑 `refresh_auth.py` 从 9223 Chrome 捞 cookie，Chrome 没起就整阶段失败）。
- `pmos_browser.py` 已删除（其 docstring 断言「纯 requests 永远 412」是错的，留着误导）。
- `refresh_auth.py` 留在磁盘上但**不在流程里**，仅备手动捞 cookie 用。

**注意：该「下线」只在 auto_pipeline。** `D:\my_product\trade-crawler` 在 2026-09-12 又**反向装回了 CDP 通道**（见下），两边结论相反不要混用。

**⚠️ 2026-09-13 修正：auto_pipeline 的 crawler-sn 也重新装回了 CDP。**
`D:\auto_pipeline\crawler-sn\crawler_sn\client.py` 现在有 `transport`（默认 `cdp`）/`cdp_url=http://127.0.0.1:9223`，`_post_cdp` 走页面内 fetch；`config.py` 有 `SN_TRANSPORT`/`SN_CDP_URL` 等；`ServiceConfig.from_env` 调 `_load_central_auth()` 读 `auth.env`。所以上面那段「已下线」描述已过时——**两个项目现在都走 CDP，共用同一个 9223 Chrome**。

## trade-crawler（D:\my_product\trade-crawler）走 CDP —— 2026-09-12 接入

该项目的盘口爬取（`POST /sn-match-maker/D3Day/dwxx`，即 `crawler/http_client.py`）原先用「IPv6 直连 + 硬编码静态令牌」，IPv6 被堵后全天 0 数据 → 套利监控无盘面 → 飞书**不推送**。已改为 **CDP 通道**：

- `config.HttpConfig.transport`（env `TRANSPORT`，默认 `cdp`）/ `cdp_url`（`http://127.0.0.1:9223`）；`http` 通道保留作兜底。
- `crawler/http_client.py` 双通道：`_fetch_cdp` 用 `page.evaluate` 在 pmos 同源页面里 `fetch`，**每轮 `run_once()` 结束调 `close()` 释放 playwright 驱动**（下一轮自动重连；`sync_playwright` 不可跨线程复用，所以按轮建拆）。
- 页面选择：优先 URL 含 `dashboard` 的 pmos 同源页，否则任意 pmos 页，都没有就新开标签导航 `https://pmos.sn.sgcc.com.cn/#/dashboard`。
- `crawler/logger.py` 加了 `logs/crawler.log`（轮转 5MB×5）——常驻进程用 `pythonw` 无控制台，日志文件是唯一可观测手段。

**关键：`CurrentRoute` 头必须与接口的菜单匹配**，否则 200 + `40302 您当前没有该接口访问权限`。盘口接口对应 `/pxf-match-maker/upCadeTrading?date=<ms>`（由 `config.HttpConfig.page_route()` 生成）。缺 `CurrentRoute` → 40302；填 `/dashboard` → 40302；填对 → 200 + 真实盘口。

**运维前提（不满足就不推送）**：① `D:\auto_pipeline\启动爬虫Chrome.bat` 拉起的 9223 Chrome 必须**常开且处于登录态**（`X-Ticket` 是会话 cookie，Chrome 一关就失效）；② 调度器 `run_scheduler.py` 得活着——它**没有**自启/保活，是手工 `Start-Process pythonw` 拉起的。

**How to apply:** 抓不到数据先看 412 还是 401/40302/40301 —— 412 是瑞数拦（当前出口协议族不行，或 CDP 通道没走通/没登录）；401/40301/40302 是令牌或 `CurrentRoute` 菜单路由不对。trade-crawler 的日志直接看 `D:\my_product\trade-crawler\logs\crawler.log`。

### 2026-09-13 补：trade-crawler CDP 驱动「毒死」的坑（已修）

现象：盘口抓取跑到某个月份中途**卡住不动**（日志停在某时段不再输出），随后的每一轮都秒失败：
`无法附着爬虫 Chrome: It looks like you are using Playwright Sync API inside the asyncio loop`。

真因：`page.evaluate` 若挂起（页面被瑞数反爬搞成主线程忙等，或页面内 fetch 永不 resolve），
`_fetch_cdp` 的 except 会调 `_reset_cdp()`，**它把 `self._pw` 置空却不 `.stop()`** → 旧 playwright
驱动没停，下一轮 `sync_playwright().start()` 再起一个就报上面那句，**整个常驻进程永久毒死**
（`HttpClient` 是单例，只能重启 `run_scheduler.py`）。

修法（`crawler/http_client.py`）：
- `_reset_cdp(stop_driver=False)` 加开关；**中途请求失败只丢 `_browser/_page`、保留 `_pw`**
  （驱动可复用，重连 CDP 即可），只有 `close()` / 驱动本身不可用才 `stop_driver=True`。
- 页面内 fetch 加 `AbortController` 超时（`HTTP.cdp_eval_timeout`，20s），超时按**可重试的传输
  错误**（`TimeoutError`）抛，不计入令牌熔断。

排障手段（有用，可复用）：页面卡死时 `page.evaluate` 也会一起挂 → 用另一个 CDP 连接
`page.reload(wait_until="commit", timeout=15000)` 重载该页即可解冻（会打断挂起中的轮次，
让 `run_once` 继续/收敛）。注意 python 重定向到文件时 stdout 是块缓冲，诊断脚本要 `python -u`。

## 周出力预测接口（crawler-sn 的 `*_w` 表）—— 2026-09-13 打通

`params_config.yaml` 里 5 个周接口：`负荷预测周`/`风电出力预测周`/`光伏出力预测周`/`新能源出力预测周`/`水电出力预测周`
（表 `raw/dwd/dws_pred_{load,wind,pv,re,hydro}_w`；负荷/水电走 `selectByTable`，风电/光伏/新能源走
`/sn-et-information-out/newEnergyWeeklyTotalOutput/queryByDateTime`）。

**风电/光伏/新能源三个接口光有 ClientTag 会 200 + `40302`**，必须补两样（2026-09-13 用户给 curl 打通）：
1. `CurrentRoute: /pxf-retail-out/output/TotalOutputForecast?date=<ms>`（已加进 `_CURRENT_ROUTE_BY_PATH`）；
2. 两个静态头 `secret: 7532b9e62d914d21b82cc6dcbb60bd43` 和 `check_value: 04E2B130…19560`（240 hex）。

为此给 `client.py::_build_headers` 加了 **`extra_headers`** 支持（读 `params_config.yaml` 里各接口的
`extra_headers` 字典，后写可覆盖同名头）。`check_value` 疑为**会话级静态签名**（同一值对不同 body
都成功），但**可能会随登录态轮换**——届时 40302/报错就从浏览器 Network 重新复制这两个头写回 yaml。

口径：`day_offsets=[0..6]` 抓 D~D+6；负荷/水电是 `monday_offsets=[0..6]`，**只在周一**才展开整周，
其余日子只抓当天（所以周日跑完 负荷/水电 只到当天，属正常）。命令：
`SN_DB_NAME=dlsc_shaanxi_auto python crawler_service.py run-once --mode incremental --interface 风电出力预测周 …`
