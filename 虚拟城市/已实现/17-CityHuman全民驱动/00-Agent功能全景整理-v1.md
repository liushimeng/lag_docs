# 虚拟城市 — CityHuman Agent 功能全景整理 v1

> **批次**：17-CityHuman 全民驱动 ｜ **日期**：2026-09-22 ｜ **性质**：Agent 功能实现全景索引（本轮整理）
> **定位上游**：[`../../虚拟城市定位声明-全Agent真实城市模拟器.md`](../../虚拟城市定位声明-全Agent真实城市模拟器.md)
> 本文回答「虚拟城市的 Agent 都有哪些功能、在哪个文件、由什么驱动」。实现契约见本目录 01–04。

---

## 1. AgentClassName 登记（CLAUDE.md §24）

| AgentClassName | 调用方 | 文件 |
|---|---|---|
| `LsmAgentGame-City-Human` | **唯一**城市居民身份：深度居民月度决策 / 驱动层居民轮次 / 城市之声 | `ServerGo/agent/class_names.go`（`AgentClassCityHuman`）；消费点 `agent/wealthplayer/{run,run_llm}.go`、`game/wealth/city/{voice,driver}.go` |
| `LsmAgentGame-Werewolf-*` | 狼人杀（Player/Judge/MemoryIter） | 不在本文范围 |

- 出站 UA：`LsmAgentGame-City-Human/<AppVersion> <buildDateTime>`（`llm/anthropic/anthropic.go::userAgentFor`）。
- 五旧类（City-Player/Voice/Government/Banker/Firm）已于 12 批次合并删除。

## 2. 深度居民 Agent（座位层，引擎内部固定 12）

> 17 批次起「焦点居民」概念退役：这 12 名居民是引擎内部的**深度轨迹层**，
> 服务端建城时自动注册，不再暴露给建房界面（契约见 01 §3）。

| 功能面 | 实现 | 文件 |
|---|---|---|
| Agent 结构 | registry / linePoolSource / ToolRunner / Memory / speak+sense 限次 | `agent/wealthplayer/agent.go` |
| 月度决策循环 | GameContext 快照 → prompt → 流式 LLM → 工具循环(≤3 轮) → submit_month/watchdog | `agent/wealthplayer/run.go` |
| LLM 调用 | 直连 `registry.Get` 或池模式 `LinePool.Acquire`→`lease.Provider`→`ChatStreamAccumulate` | `agent/wealthplayer/run_llm.go` |
| 工具定义 | 46 工具（经济 35 + P2 交易 12 + 感知 5 − 复用计数），Anthropic wire 四键约束 | `agent/wealthplayer/{tools,tools_sense,tools_trade}.go` |
| 感知行动 | see/hear/smell（各 ≤2/月）+ move（walk/run/bus/metro/taxi）+ speak(area/private ≤2/月) | `game/wealth/agent_sense.go`（ToolRunner 实现） |
| 记忆 | 24 月滚动 MonthDecision + 20 KeyFacts + 规则式压缩（无 DB 持久化） | `agent/wealthplayer/memory.go` |
| 提示词 | System 5 段（身份/职业卡/人格风险/目标/规则）+ User 月度快照（含周边环境与氛围段） | `agent/wealthplayer/prompt.go` |
| 装配 | `Manager.ensureAgentsWithPool`：每 bot 座位 NewAgent + BindRegistry + BindLinePoolSource + BindRunner | `game/wealth/manager.go` |
| 房间信号量 | `agentSem` 容量 = `max(LinePool.Total(),1)`（Start 锁内 `resizeAgentSemLocked`） | `game/wealth/{room,room_city}.go` |

## 3. 背景居民与城市层（1 ~ 100,000）

| 功能面 | 实现 | 文件 |
|---|---|---|
| 紧凑居民世界 | `Backdrop`（≈20B/人）+ `TickMonth` 月度演化 + `Snapshot` 聚合（100K tick <100ms） | `game/wealth/city/backdrop.go` |
| 档案锚定 | 建城后异步 `DrawPaths → HydrateBatch(worker pool) → AnchorProfiles`，10 万卡 ≈100s 不阻塞开局 | `game/wealth/{room_city,profession/loader_batch}.go`、`city/profile.go` |
| 城市之声 | 每月抽样居民经线路池发一句极短话（AgentClass=City-Human，失败静默丢弃） | `game/wealth/city/voice.go` |
| 校准表 | 抽样 512 卡 → 26 域 × 16 区分布；兜底链 docs → synthetic | `game/wealth/city/calibration.go` |
| 邻居/氛围 | 同城区邻居抽样 + 城区气味/声响基底 + 事件叠加 | `game/wealth/city/neighbors.go`、`districts.go`、`events.go` |

## 4. 居民驱动层（本批次新增，契约见 02）

| 功能面 | 契约 |
|---|---|
| ResidentDriver | 每月预算内抽居民 → **线程池** worker 执行「居民轮次」：persona(System) + 月度状态(Context) + 通用工具 |
| 线程池 | `city_driver_workers`（默认 4，clamp 1..16）个常驻 worker goroutine 消费任务队列 |
| LLM 线路池 | 每次调用 `LinePool.Acquire(ctx 15s)`；全局并发 = Σ enabled 行 `concurrency_lines`；`ErrAllLinesBusy` → 本轮丢弃 |
| 通用工具 | `set_intent`（本月意图 → TickMonth 演化加成）+ `speak`（→ 城市之声）；单轮解析、无二轮循环 |
| 公平轮转 | 跨月 cursor 轮转 + stressed/失业优先；意图月末清除 |
| 与城市之声关系 | Driver 启用时 VoiceScheduler 停用（speak 输出即城市之声）；关闭时回退旧路径 |

## 5. 出站纪律（全 Agent 共用，违反即 400/零 token）

1. ContentBlock 按 Type 收敛（text 只 `{type,text}`；tool_use 四键齐；tool_result 禁 id/name/input）。
2. messages user/assistant 严格交替；content 为 block 数组；system 为 SystemBlock 数组。
3. 流式优先 `ChatStreamAccumulate`（§197 字节刷新续命）；5xx/429 由 anthropic provider 内建重试。
4. `LLMRequest.AgentClassName` 必填（单测断言非空）。

## 6. LLM 线路池（`llm/linepool.go`，全局）

- 一条线路 = 一个可并行 LLM 调用通道；`M = Σ enabled(provider).concurrency_lines`（每行 clamp [1,64]）。
- `Acquire(ctx)` 从预填充令牌通道取令牌（平滑加权轮询）；`LineLease.Release()` 幂等归还。
- `ErrAllLinesBusy`：ctx 超时/取消 → 调用方按一次失败走兜底（深度居民强制 submit_month；驱动层/之声丢弃本条）。
- `Registry.Reload` 原子换池；管理 API `concurrency_lines` 1..64；`GET /api/llm/models` 下发每行线路数。
- 前端建房弹窗展示 `LLM 线路池：N 条线路（Agent 并发数）`（Σ 计算）。

## 7. 功能全景一页速查

```
创建城市(resident_count 10..100000, 数值控件)
  └─ 服务端: 固定注册 12 深度居民(bot 座位,池驱动) + Backdrop(N 背景居民)
       ├─ 档案自动加载: 玩家职业设计 10 万卡 → 锚定到居民(异步,不阻塞开局)
       ├─ 深度层: 12 × City-Human Agent 每月 46 工具决策(记忆/感知/交易/发言)
       ├─ 驱动层: 每月 ≤PerMonth 名背景居民轮次(线程池 + 线路池, set_intent+speak)
       ├─ 城市之声: 驱动层 speak 产出 / 关闭驱动层时回退 voice 抽样
       └─ 经济层: TickMonth 数值演化(消费意图加成) + 世界系统(央行/财政/市场/监管)
```

## 8. 测试与验收索引

- 驱动层单测：`game/wealth/city/driver_test.go`（fake pool 预算/线程池并发/轮转公平/意图演化）。
- 建房契约：`api/room_api_resident_test.go`（10..100000 clamp）、`service` 建房 12 深度座位自动注册。
- 退役门禁：`grep -rn "CuratedCards\|ProfessionAPI\|焦点居民数" ServerGo/ ClientWeb/src/` 零命中（历史文档除外）。
- 总门禁：`go build` + `go test ./...`；`tsc --noEmit` + `npm run build`；E2E 建城冒烟。

---

## 更新日志

- v1（2026-09-22）：首版。17 批次整理：全景索引 + 驱动层契约摘要 + 速查图。
