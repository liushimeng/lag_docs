# 虚拟城市 — 城市 Agent 规模化与 LLM 线路池总体方案 v1

> 状态：实现契约（2026-09-21）。本文是本轮重构的**总纲**：目标、架构、全触点遍历、兼容性。
> 细节契约见 02（LLM 线路池）/ 03（城市背景模拟）/ 04（建房与前端 UI）/ 05（验收）。

## 1. 背景与问题

### 1.1 现状（重构前）

| 维度 | 现状 | 问题 |
|---|---|---|
| 城市 Agent 规模 | 每房固定 12 座（`MaxSeats=12`，全 Agent 模式 12 bot） | 与「城市有 1~10 万居民」的产品设定相差 4 个数量级 |
| Agent ↔ LLM 绑定 | 建房时每个座位**显式挑选 model_key**（Fisher-Yates 随机去重），`wealthplayer.Agent` 终生固定该模型 | 模型管理行的语义被误读为「一个 Agent」；模型与居民身份强耦合 |
| `LLM 模型管理` | 每行有必填唯一 `agent_name`，管理页以「Agent 名称」为主列 | 行数 ≠ Agent 数；`agent_name` 与城市居民重名冲突，概念错位 |
| 并发控制 | provider 层**无任何并发概念**；只有房间级信号量（狼人杀 4 / 财商流 8） | 「多少条线路 = 多少并发」的产品诉求无处落地 |

### 1.2 目标（用户需求直译）

1. 虚拟城市容纳 **1 ~ 100,000+** 名 Agent 居民，模拟每个城市人类角色的真实活动；
   居民数据由 `lag_docs/虚拟城市/玩家职业设计/`（100,174 张职业卡，Schema v1.1）加载。
2. `LLM 模型管理` **不再使用 Agent Name**；**启用行的线路总数 = Agent 调用大模型的并发数**。
3. 居民层（1~10 万，职业卡驱动）与 LLM 线路层（1~N，并发池）**彻底分开、分别设计**。
4. 遍历所有相关游戏界面、Agent 驱动、建房流程、后台逻辑同步改造。

## 2. 双层解耦架构

```
┌─ 居民层（Identity & Simulation）──────────────────────────────┐
│ 城市居民 1..100,000+（resident_count，建房参数）                  │
│   ├─ 背景居民：ResidentLight 轻量结构（≈40B/人），逐月数值模拟      │
│   │   数值初值 ← 职业卡校准表（10 万卡异步抽样 → 26 行业域分布）     │
│   └─ 焦点座位 ≤12：既有全量引擎（职业卡全字段 + 39 工具 + Memory）  │
│ 身份唯一来源：职业卡（Card.Name/Title/收入/人格/开局钩子…）         │
├─ 认知调度层（Cognition Scheduling）───────────────────────────┤
│ 需要深认知的居民（焦点座位决策 / 城市之声抽样）统一入队             │
│ 全局并发 = LinePool 令牌总数 M（阻塞排队，ctx 超时→规则兜底）       │
├─ LLM 线路层（Lines & Concurrency）───────────────────────────┤
│ t_lsm_game_llm_provider 每行 = 一个模型配置 + concurrency_lines  │
│ M = Σ enabled(provider).concurrency_lines（默认行=1，范围 1..64） │
│ LinePool.Acquire() 加权轮询发线路令牌；Release 归还               │
│ agent_name 从管理语义废弃（UI 移除；API 可选自动派生）             │
└──────────────────────────────────────────────────────────────┘
```

**为什么背景居民不能全量走 LLM**：100,000 人 × 每月 1 次决策 × 单次 3~8s，
即使 64 条线路也要 1.3~3 小时/月。因此认知分层——
背景居民默认数值脑，LLM 线路按月抽样升级「城市之声」（见 03 §5）；
焦点座位每月决策必走线路池（既有超时兜底 `submit_month` 不变）。

## 3. 全触点遍历清单（改什么、在哪）

### 3.1 游戏界面（ClientWeb）

| # | 触点 | 变更 | 文件 |
|---|---|---|---|
| F1 | LLM 模型管理列表 | 删「Agent 名称」列；新增「线路数」列；头部显示「总线路数（Agent 并发上限）」 | `pages/ModelAdminPage.tsx` |
| F2 | LLM 模型新增/编辑表单 | 删 agentName 输入；新增 concurrency_lines（1-64，默认 1） | `pages/ModelAdminPage.tsx` |
| F3 | 模型详情页 | agent_name 行 → 线路数行 | `pages/ModelDetailPage.tsx` |
| F4 | 财商流建房弹窗 | **移除每座位模型选择器 + 随机分配 + 🎲 重摇**；新增「城市居民数量」（1~100000 + 预设档）；显示「LLM 线路池：N 条线路」 | `components/wealth/WealthCreateRoomModal.tsx` |
| F5 | 对局页 | 新增城市统计面板（人口/就业率/收入中位数/居民之声） | `pages/WealthGamePage.tsx` + `components/wealth/CityStatsPanel.tsx`（新） |
| F6 | 房间列表 | 房间行显示「🏙 居民 N」徽标 | `components/wealth/RoomListTable.tsx` |
| F7 | 类型与 API 契约 | `concurrency_lines`、`resident_count`、`city` 块、`city_voice` 事件 | `types/model.ts` `api/llm.ts` `api/wealth.ts` `types/wealth.ts` |
| F8 | i18n | 三语新增 ~14 键；删 `modelAdmin.colAgentName/fieldAgentName` 2 键 | `i18n/{zh-CN,en,ja}.ts` + `i18n/types.ts` |

**不动的界面**（`agent_name` 仍由 API 派生值兼容展示）：狼人杀/德州建房模型选择器、
GameChatPanel 🤖 徽标、WerewolfTable 座位卡、雷达图/排行榜——本轮范围是财商流，
狼人杀并发模型不变（房间级信号量），仅文档记录。

### 3.2 Agent 驱动大模型（ServerGo）

| # | 触点 | 变更 | 文件 |
|---|---|---|---|
| B1 | 线路池 | 新增 `LinePool`（令牌通道 + 加权轮询 + 指标） | `llm/linepool.go`（新） |
| B2 | registry | `registeredProvider` 增加 lines；`TotalLines()`；Reload 重建池 | `llm/registry.go` |
| B3 | Agent 线路池模式 | `wealthplayer.Agent.ModelKey` 允许空 = 池驱动；`run_llm.go` 分支 Acquire/Release；Acquire 失败走既有超时兜底 | `agent/wealthplayer/{agent,run_llm}.go` |
| B4 | 城市之声 Agent | 新 AgentClassName `LsmAgentGame-Wealth-CityVoice`（注册 + 单测） | `agent/class_names.go` |
| B5 | 焦点座位并发 | `wakeBots` 房间信号量容量 → `max(LinePool.Total(),1)`（池缺失时回退 `AgentConcurrency`） | `game/wealth/room.go` |
| B6 | 城市背景包 | 校准表 + ResidentLight 数组 + 月度演化 + 聚合快照 + 城市之声调度 | `game/wealth/city/`（新包：backdrop/calibration/voice） |

### 3.3 创建游戏房间

| # | 触点 | 变更 | 文件 |
|---|---|---|---|
| R1 | 建房请求体 | 新增 `resident_count`（0~MaxResidents，默认 0=不启用城市层）；`agent_seats[].model_key` **对 wealth 允许空串**=池驱动（狼人杀仍强制校验） | `api/room_api.go` |
| R2 | 座位校验 | `ValidateAgentSeats` 按 kind 分流：wealth 空串放行，werewolf 行为不变 | `ws/game_service.go` |
| R3 | 座位注册 | 空模型座位照常建 bot 用户与 Agent；昵称改为职业卡名 `AI·<Card.Name>`（Start 抽卡后升级），无卡回退 `AI·居民<seat>号` | `ws/game_service_wealth_bot.go` + `game/wealth/room.go` |
| R4 | 全 Agent 模式 | 语义不变（seats≥10 自动置位/35013 拒人类）；`model_key` 为空不影响 | `game/wealth/room.go` |
| R5 | 房间视图 | `ClientGameState` 新增 `city` 块（omitempty）；座位 `model_display` 池驱动时 = `LLM线路池` | `game/wealth/view.go` |

### 3.4 后台逻辑

| # | 触点 | 变更 | 文件 |
|---|---|---|---|
| G1 | GORM 模型 | `t_lsm_game_llm_provider` 新增 `concurrency_lines int default 1`；`agent_name` 列保留但语义废弃 | `models/t_lsm_game_llm_provider.go` |
| G2 | 管理 API | create/update 增 `concurrency_lines`（1..64）；`agent_name` 变可选，空则按 model 派生（冲突加序号后缀保唯一索引） | `api/model_admin_api.go` |
| G3 | 模型列表 API | `/api/llm/models` 条目增 `concurrency_lines`（`agent_name` 字段保留为兼容展示，标注 deprecated） | `api/llm_api.go` |
| G4 | 种子默认 | 8 家默认 seed 每行 `concurrency_lines=1` | `llm/defaults.go` |
| G5 | 配置 | `cfg.Wealth` 增 `max_residents/city_voice_enabled/city_voice_per_month/city_calib_sample_size` | `config/config.go` + `LsmAgentGame.conf.example` |
| G6 | 装配 | wealth manager 注入 LinePool 与城市配置；校准表后台预热 | `main.go` |

## 4. 兼容性策略

| 存量面 | 策略 |
|---|---|
| DB 既有 provider 行 | AutoMigrate 补列默认 `concurrency_lines=1` → 8 行启用即 8 条线路，恰好等于旧 `AgentConcurrency=8`，行为零变化 |
| 旧请求显式带 `model_key` 的建房 | 仍支持（座位固定模型，不经池），存量测试与狼人杀路径不受影响 |
| `agent_name` 展示面（狼人杀聊天昵称、雷达、排行榜） | API 派生值（= model 或 model-2）继续下发，功能不回归；管理页不再出现该概念 |
| `registry.Get(model_key)` | 保留（法官/解说/德州等固定模型场景），线路池是新增并行能力 |
| 存量进行中房间（重启 hydrate） | `SeatModelKeys` 已有值 → 原路径；空值 → 池驱动，两态幂等 |

## 5. 验收标准（摘要，详见 05）

1. `go build` + `go test ./...` 全绿；`tsc --noEmit` + `npm run build` 全绿。
2. LinePool：并发上限 = Σ线路数（并发单测断言）；Acquire 阻塞/Release 归还/Reload 重建。
3. 建房 `resident_count=100000` 创建耗时 < 1s、常驻内存增量 < 32MB、月度背景 tick < 100ms。
4. 管理页无任何「Agent 名称」；总线路数展示正确；`agent_name` 缺省可创建 provider。
5. 池驱动建房 → 12 焦点座位全部产生决策（BotTranscripts 活跃）；城市之声每月产出 ≤ 配置条数且消耗线路池。
6. 狼人杀/德州/辩论全量测试不回归。

## 6. 决策记录

| # | 决策 | 理由 |
|---|---|---|
| D1 | 行 = 模型配置 + 线路数字段，而非「一行一条线路」 | 避免同模型 N 行重复密钥配置；`model` 唯一索引无需破坏；UI 仍以「总线路数」呈现并发语义 |
| D2 | `agent_name` 软废弃（列保留、API 可选派生），不物理删除 | 唯一索引与狼人杀等展示面依赖；物理删除波及面大且无收益 |
| D3 | 背景居民数值脑 + LLM 抽样「城市之声」 | 100K 全量 LLM 物理不可行（见 §2 算术）；抽样保留「每个居民都是活人」的观感 |
| D4 | 焦点座位仍 ≤12、全量引擎不变 | 本轮目标是规模化与解耦，不是重写引擎；12 焦点 + 10 万背景 = 产品语义「城市里的主角与芸芸众生」 |
| D5 | 狼人杀不切线路池 | 并发模型（房间级信号量）是其既有契约；列入后续观察项 |
