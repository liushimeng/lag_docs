# 虚拟城市 — 建房解耦与前端 UI 改造设计 v1

> 状态：实现契约（2026-09-21）。上游总纲见 01；线路池契约见 02；城市层契约见 03。
> 本篇定义建房 API 变更与全部前端改造。

## 1. 建房 API 契约（`POST /api/games/wealth/rooms`）

### 1.1 请求体（新）

```json
{
  "name": "我的城市",
  "agent_seats": [ {"seat": 1, "model_key": ""}, {"seat": 2, "model_key": ""} ],
  "resident_count": 10000,
  "wealth": { "month_ms": 8000, "pool": "docs", "seed": 42 },
  "full_agent": true
}
```

| 字段 | 变更 | 规则 |
|---|---|---|
| `resident_count` | **新增** | int，缺省 0=不启用城市背景层（纯 12 座旧形态，向后兼容）；负数 400；`(0, max_residents]`，超上限 clamp 到 `cfg.Wealth.MaxResidents`（默认 100000） |
| `agent_seats[].model_key` | 语义放宽 | **wealth 允许空串 = 线路池驱动**（座位不绑定模型）；非空仍走固定模型（兼容存量）；werewolf 建房校验**不变**（仍强制合法 key） |
| `agent_seats[].profession` | 不变 | 座位职业偏好（存量字段，HTTP 路径未填充的既有行为保留） |

### 1.2 校验分流（`ws/game_service.go::ValidateAgentSeats`）

- 新增按 kind 分流：`kind == "wealth"` 时 `model_key == ""` 直接放行（不查 registry）；
  非空 key 行为不变（未知/禁用/占位 → 400）。
- 其余游戏（werewolf 等）走原路径，**零变化**。

### 1.3 房间响应 / 状态下发

- 房间列表与详情：新增 `"resident_count"`。
- `ClientGameState` 新增（`view.go`，`omitempty`）：

```json
"city": {
  "resident_count": 10000, "employment_rate": 0.93, "median_income": 6800,
  "total_savings": 12345678.9, "avg_age": 36.4, "stressed_rate": 0.18,
  "districts": [{"id":0,"name":"城东","population":1250}],
  "voices": [{"month":12,"name":"A1024·华东城东","text":"…","model":"DeepSeek"}]
}
```

- 座位字段：池驱动座位 `model_display = "LLM线路池"`；事件流新增 `EventRecord.type="city_voice"`。

### 1.4 座位注册与昵称（后端）

- `registerWealthAgentSeats`：空模型座位照常创建 bot 用户 + 注册座位；
  `SeatModelKeys[i]` 存空串。`Manager.EnsureAgents` 改为「非空 key **或** 池可用」均创建
  `wealthplayer.Agent`（`ModelKey=""` 即池模式）。
- 昵称策略：注册时先 `AI·居民<seat>号`；`Start()` 抽卡后升级为 **`AI·<Card.Name>`**
  （真实职业卡人名，如 `AI·陆荷`），并按既有用户更新机制广播。
- 全 Agent 模式判定（seats≥10 / 35013 拒人类 / 观战路由）全部不变。

## 2. 前端改造

### 2.1 `components/wealth/WealthCreateRoomModal.tsx`

| 项 | 契约 |
|---|---|
| 删除 | 每座位模型 `<select>`、`shuffledRoundRobinModels`、🎲 重新分配按钮及相关 state（R1-R3 随机分配方案整体退役） |
| 新增：城市居民数量 | 数字输入 + 预设档按钮 `12 / 1千 / 1万 / 10万`；clamp 1..100000；默认 **10000**；`0` 不可选（UI 语义：至少 1 名居民；纯旧形态走 agent 数=0 的 12 座房） |
| 新增：线路池信息 | `listModels()` → Σ`concurrency_lines` → 展示 `LLM 线路池：N 条线路（Agent 并发数）`；N=0 时黄色警示（无可调模型） |
| Agent 数 | 保留 0..12 档位按钮（默认 12 全 Agent 语义不变） |
| 提交体 | `agent_seats` 的 `model_key` 一律送空串；新增 `resident_count` |
| 保留 | month_ms 预设/滑杆、pool curated/docs 选择与统计、seed、职业卡一览、MinSeats 门控 |

### 2.2 `pages/ModelAdminPage.tsx`（详见 02 §6）

- 删「Agent 名称」列与表单项；新增「线路数」列/表单项（1-64）；头部总线路数提示条。

### 2.3 `pages/ModelDetailPage.tsx`

- 基本信息卡：agent_name 行 → 「线路数」行。

### 2.4 `pages/WealthGamePage.tsx` + `components/wealth/CityStatsPanel.tsx`（新）

- 新组件挂右侧栏（与 WealthBotPanel 同列）：
  - 头部：`🏙 城市 · N 人`；
  - 指标行：就业率 / 收入中位数 / 居民储蓄合计 / 压力率（百分比格式化）；
  - 城区人口迷你条形（8 区，纯 CSS 宽度百分比）；
  - 「居民之声」列表：最近 voices（月份 + 代号 + 一句话 + 服务模型小徽标）。
- `city` 缺省（旧房）时整面板不渲染。
- `city_voice` 事件进入既有事件流 UI（无需新组件，EventRecord 既有渲染）。

### 2.5 `components/wealth/RoomListTable.tsx`

- 房间行 resident_count>0 时显示 `🏙 N` 徽标（title=城市居民数）。

### 2.6 类型与 API 契约（`ClientWeb/src`）

| 文件 | 变更 |
|---|---|
| `types/model.ts` | `LlmProvider.concurrency_lines: number`；`LlmProviderCreate.concurrency_lines?: number` |
| `api/llm.ts` | `ModelInfo.concurrency_lines: number` |
| `api/wealth.ts` | 建房 body 增 `resident_count?: number` |
| `types/wealth.ts` | `WealthCitySnapshot`（含 DistrictPop/VoiceRecord）；`ClientGameState.city?: WealthCitySnapshot`；事件类型增 `city_voice` |

## 3. i18n 键表（三语 zh-CN / en / ja 同步，`i18n/types.ts` 同步声明）

| 键 | zh-CN | 用途 |
|---|---|---|
| `modelAdmin.colConcurrencyLines` | 线路数 | 管理页列 |
| `modelAdmin.fieldConcurrencyLines` | 并发线路数 | 管理页表单 |
| `modelAdmin.totalLinesHint` | 总线路数 {n} 条 = Agent 调用大模型的并发数 | 管理页提示条 |
| `modelAdmin.linesRangeError` | 线路数须在 1-64 之间 | 表单校验 |
| `wealth.residentCount` | 城市居民数量 | 建房弹窗 |
| `wealth.residentCountHint` | 居民由 10 万职业卡生成，背景居民逐月模拟 | 建房弹窗说明 |
| `wealth.linePoolInfo` | LLM 线路池：{n} 条线路（Agent 并发数） | 建房弹窗 |
| `wealth.linePoolEmpty` | 当前无可用 LLM 线路，请先在模型管理配置 | 警示 |
| `wealth.cityTitle` | 城市 | 面板标题前缀 |
| `wealth.cityPopulation` | 居民 | 面板指标 |
| `wealth.cityEmployment` | 就业率 | 面板指标 |
| `wealth.cityMedianIncome` | 收入中位数 | 面板指标 |
| `wealth.citySavings` | 居民储蓄 | 面板指标 |
| `wealth.cityStress` | 压力率 | 面板指标 |
| `wealth.cityVoices` | 居民之声 | 面板列表标题 |
| `wealth.cityVoiceOfMonth` | 第 {month} 月 | 声音条目前缀 |
| 删除 | `modelAdmin.colAgentName` / `modelAdmin.fieldAgentName` | — |

en/ja 由对应 locale 文件同步翻译（"Lines" / "Concurrent Lines"；「路線数」/「同時接続路線数」）。

## 4. 错误展示（CLAUDE.md §7.1 合规）

- 建房失败（含 resident_count 非法）：`WealthCreateRoomModal` 内联红条（不关弹窗）。
- `listModels()` 失败：弹窗内联提示 + `reportGlobalError` 兜底（两者其一，推荐都做）。

## 5. 测试要求

| 层 | 用例 |
|---|---|
| 后端 | `resident_count` 边界（0/1/100000/100001→clamp/负→400）；空 model_key 建房成功且 12 Agent 池驱动；werewolf 空 key 仍 400 |
| 后端 | 昵称升级：Start 抽卡后座位昵称= `AI·<Card.Name>` |
| 前端 | `tsc --noEmit` + `npm run build` 通过；i18n 三语键完备（既有键校验机制） |
