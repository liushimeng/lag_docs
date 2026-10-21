# 批次 69 · 虚拟城市 · 实时看板统一重构

> **日期**：2026-10-21 ｜ **分支**：`main`（CLAUDE.md §10.1）
> **一句话**：把虚拟城市右侧栏的 **9 个平级 Tab** 重构为**单一「虚拟城市 - 实时看板」**——
> 3 个一级分组 / 11 个维度 Tab / 目录双源（前端静态兜底 + 后端下发校验）/ 通道离线时骨架仍可用。

---

## 文档索引

| # | 文档 | 定位 | 主要读者 |
|---|------|------|---------|
| 1 | [`01-方案设计.md`](01-方案设计.md) | **权威来源**：背景与目标、7 条设计原则、3 组 11 Tab 信息架构、维度目录（11 条 × 7 字段）与数据源登记册（22 条 × 4 字段）、`GET /api/virtual_city/dashboard` 字段级契约、前端外壳规范、城市总览 39 项 KPI 取数表、i18n 规范、16 条风险控制、验收清单、**8 条实现偏离（D1–D8）** | backend-dev、frontend-dev、integration-tester、game-designer |
| 2 | [`02-实施记录.md`](02-实施记录.md) | 落地文件清单（后端 / 前端 / 文档三类）、契约落点表、**验收执行结果（待填）**、实现取舍、遗留与后续（L1–L6） | 主 Agent、integration-tester |

## 阅读顺序

1. **实现方（backend-dev / frontend-dev）**：先读 `01` 的 §3（信息架构）、§4（数据体系）、§5（接口契约）——
   这三节是**字段级契约**，前后端据此并行实现，字段名/路径不允许漂移。
2. **验收方（integration-tester）**：读 `01` 的 §10（验收清单）逐条勾选；已知偏离与已知遗留见 `01` §11–§12。
3. **接手方**：先读 `02` 的 §6（遗留与后续），避免重复踩坑。

## 核心速查

| 项 | 值 |
|---|---|
| 一级分组 | `city` 城市运行（6）/ `market` 市场金融（3）/ `external` 外部实时（2） |
| 维度 Tab id（11） | `city-overview` `finance` `ledger` `economy` `behavior` `macro` `market` `market-trade` `insurance` `realtime` `datasources` |
| 既有 9 个 id | **零改名** ⇒ `onTradeTab('market-trade')` / `onViewLedger()` 天然继续有效 |
| 新端点 | `GET /api/virtual_city/dashboard`（需登录，与 `realtime_board` 同组同鉴权） |
| 响应体 | `data.dashboard` = `schema_version`(int) / `generated_at`(unix 秒) / `dimensions`(11) / `data_sources`(22) / `realtime`(≡ `citychrome.Feed`) |
| 离线语义 | `code=35053`（复用批次 65）但 **`data.dashboard` 仍完整** ⇒ 骨架可渲染 |
| 数据源分类 | 9 类（综合财经 / A股 / 港股 / 美股 / 贵金属大宗 / 期货 / 加密货币 / 宏观 / 数据接口），共 22 条 |
| i18n 命名空间 | `virtualCity.dash.*`（36 键 × 3 语言；**注意与 P2 的 `virtualCity.dashboard.*` 区分**） |
| 新增 CSS | `styles/virtualCity-dashboard.css`（`.virtualCity-dash*`），`globals.css` **链尾追加** |

## 上游与相关批次

- 批次 65《CityChrome 实时数据通道与实时看板》—— 外部实时通道与 `RealtimeBoard`（8 子 Tab）。
- 批次 66《真实行情市场联动》—— 外盘情绪锚（`market.exo_anchor`）与其零回归契约。
- 批次 68《预测运行报告》—— `game.state.prediction`（已纳入城市总览「宏观与风险」组）。
- [`../../虚拟城市定位声明-全Agent真实城市模拟器.md`](../../虚拟城市定位声明-全Agent真实城市模拟器.md) —— 定位与术语基准。
