# 11-城市扩张 — 目录索引（v2.12）

> **已实现 ✅ commit ccf321b81f (2026-09-21)**（阶段 2+3；阶段 1 文档对应 commit bba37f562d）
> 城市扩张专项：8 城区 40×40 地图 → 16 城区 80×80 地图 + 宏观金融深化（央行货币政策工具箱）。
> 上级索引：[`../README.md`](../README.md)（虚拟城市实现档案总入口）。
> 规约：CLAUDE.md §3（Markdown ≤ 800 行）/ §4（代码 ≤ 1800 行）/ §13（SubAgent 分工）。

---

## 1. 子目录与文档

| 子目录 | 文档 | 类型 | commit |
|--------|------|------|--------|
| `01-架构与规模化/` | `虚拟城市-城市扩张v2.61-总体架构升级.md` | 设计 | bba37f562d（阶段 1 前置方案） |
| | `虚拟城市-城市扩张v2.12-8大方向分阶段路线.md` | 设计 | bba37f562d |
| | `虚拟城市-城市扩张v2.12-200背景居民架构.md` | 设计 | bba37f562d |
| | `虚拟城市-城市扩张v2.12-200居民LLM调度策略.md` | 设计 | bba37f562d |
| | `虚拟城市-城市扩张v2.12-动态并发与自适应月窗.md` | 实施（阶段 1） | bba37f562d |
| | `虚拟城市-城市扩张v2.12-16城区地图扩展.md` | 设计 | bba37f562d |
| `02-地图与3D/` | [`虚拟城市-城市扩张v2.12-16城区地图扩展实施记录.md`](02-地图与3D/虚拟城市-城市扩张v2.12-16城区地图扩展实施记录.md) | **实施（阶段 2）** | ccf321b81f |
| | [`虚拟城市-城市扩张v2.12-16城区静态表契约.md`](02-地图与3D/虚拟城市-城市扩张v2.12-16城区静态表契约.md) | **契约** | ccf321b81f |
| `03-货币政策/` | [`虚拟城市-城市扩张v2.12-货币政策工具箱实施记录.md`](03-货币政策/虚拟城市-城市扩张v2.12-货币政策工具箱实施记录.md) | **实施（阶段 3）** | ccf321b81f |
| | [`虚拟城市-城市扩张v2.12-菲利普斯曲线与利率传导实施记录.md`](03-货币政策/虚拟城市-城市扩张v2.12-菲利普斯曲线与利率传导实施记录.md) | **实施（阶段 3）** | ccf321b81f |
| | [`虚拟城市-城市扩张v2.12-央行公告与季度节奏.md`](03-货币政策/虚拟城市-城市扩张v2.12-央行公告与季度节奏.md) | **实施（阶段 3）** | ccf321b81f |

阶段 1（动态并发公式 + 3 AgentClassName 注册）实施记录见 `01-架构与规模化/虚拟城市-城市扩张v2.12-动态并发与自适应月窗.md`（commit bba37f562d）。

## 2. 阶段 2 + 3 提交快照（ccf321b81f，2026-09-21）

### 阶段 2 — 16 城区地图扩展（面积 ×4）

- 城区静态表 8→16：后端 `ServerGo/game/wealth/districts.go`（`DistrictCount=16`、`DistrictDefs` 追加 8 区）、前端 `ClientWeb/src/types/wealth.ts`（`WEALTH_DISTRICTS`）、城市背景层 `city/calibration.go` + `city/backdrop.go`、职业卡 `profession/card.go` —— 四方 id/顺序逐字一致（契约文档）。
- 地图 40×40 → 80×80：`WealthCityMap.tsx` `WORLD_SIZE=80` 派生常量块（贴图/雾/相机/光照等比）；`WealthMinimap.tsx` WORLD 同源 import；`cityScale.ts` `DISTRICT_FLOORS` 追加 8 区楼层区间。
- 适配：`StreetPropsLayer.tsx` props 化（16 区 ≈92 mesh ≤ 2500 护栏）；`CityStatsPanel` >12 区紧凑模式（top10 + 其他 N 区聚合）；i18n zh-CN/en/ja + wealthKeys 9 新键。
- 守卫：`seats12_test.go` 契约升级（16 区 + 前 8 P0 id/顺序冻结）。

### 阶段 3 — 央行货币政策工具箱

- `policy_toolbox.go`（251 行）：六件套工具（SLF/MLF/降准/升准/OMO/信贷窗口）+ `ApplyInstrument`（nil 守卫、步长量化 0.25%、RRR clamp [5%,20%]、12 月历史、事件播报）。
- `phillips_curve.go`（122 行）：CPI+失业率 4 规则 → 政策倾向（hawkish/dovish/neutral）+ 利率建议（clamp ±5%），nil 安全纯函数。
- `interest_transmission.go`（201 行）：4 步传导链 MLF→SHIBOR→资金成本→零售→实体经济 + `TraceTransmission` 留痕 + `ConsumerLoanRate` 新方法。
- `central_bank.go`（+29 行）：`SLFRate`/`NIM`/`CreditWindowFactor` 三字段 + `MonthlyDecision` 步骤 l 季度公告（每 3 月，文本占位，LLM 接入阶段 4+）。
- `central_bank_v212_test.go`（308 行 12 测试全 PASS）。

### 验证门禁

`go build` ✅ ｜ `go test ./...` 30 包全 ok ✅ ｜ `tsc --noEmit` ✅ ｜ `npm run build` ✅ ｜ `rebuild_restart_app` ✅ ｜ §130 grep 16 区四方对齐 ✅

## 3. 后续阶段入口

路线图（`01-架构与规模化/虚拟城市-城市扩张v2.12-8大方向分阶段路线.md` §5 起）：阶段 4 财政税收（房产税/个税/地方债 + 央行行长 Bot LLM 接入）→ 阶段 5+ 按路线推进。**未实现内容一律以路线图为设计稿，落地后在本目录新增「实施记录」文档并更新本索引。**
