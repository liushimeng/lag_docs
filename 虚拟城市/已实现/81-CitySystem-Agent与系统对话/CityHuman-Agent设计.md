# 批次 81 · City-Human Agent 设计（城市居民）

> **日期**：2026-11-03 ｜ **分支**：`main`（§10.1） ｜ **责任 SubAgent**：city-human（§15 硬约束）
> **定位衔接**：[`../../虚拟城市定位声明-全Agent真实城市模拟器.md`](../../虚拟城市定位声明-全Agent真实城市模拟器.md) §6/§7、
> 批次 81 [`01-方案设计.md`](01-方案设计.md) §3.1 / §10 B5
> **代码事实来源**：`ServerGo/agent/vcplayer/` —— 本文所有类型 / 工具名 / 常量 / 默认值**均以代码为准**；
> 与文档冲突时以代码为准。
> **上游设计**：[`../../12-CityHuman重构/虚拟城市-CityHuman-Agent合并与感知系统设计-v1.md`](../../12-CityHuman重构/虚拟城市-CityHuman-Agent合并与感知系统设计-v1.md)、
> [`../../17-CityHuman全民驱动/01-全民驱动统一架构与抽样展示层设计-v1.md`](../../17-CityHuman全民驱动/01-全民驱动统一架构与抽样展示层设计-v1.md)、
> [`../../03-Agent设计/虚拟城市-WealthPlayer-Agent设计-v1.md`](../../03-Agent设计/虚拟城市-WealthPlayer-Agent设计-v1.md)

---

## 1. 定位

`LsmAgentGame-City-Human` 是虚拟城市**唯一**的居民 AgentClassName。

- **常量**：`ServerGo/agent/class_names.go:226`（`AgentClassCityHuman`），已登记进 `AllAgentClassNames()`（§24 两步注册之第一步）。
- **合并历史**：2026-09-22 §CityHuman重构把原「焦点层 / 背景层 / 政府 / 央行 / 企业」五类 AgentClass
  **合并为一** —— 虚拟城市是全 Agent 真实城市模拟器，**没有玩家 Agent**，City-Human 即城市居民。
  政府 / 央行 / 企业三个旧类零消费方，按 §130 直接删除（未来公告类需求复用 City-Human + 角色提示词区分，不新增 AgentClass）。
- **出站身份**：`LLMRequest.AgentClassName = "LsmAgentGame-City-Human"`，User-Agent 形如
  `LsmAgentGame-City-Human/<AppVersion> <buildDateTime>`（CLAUDE.md §24）。
- **技术契约不变**：包名 `vcplayer`、类型 `Agent`、`game_kind` 等标识符保持原样，变更仅限 AgentClassName 与用户可见文案。

### 1.1 两个调用方共用本 AgentClass（仅调用深度不同）

| 维度 | 焦点层（座位 1~12） | 背景层（抽样发声） |
|---|---|---|
| 入口 | `vcplayer.Agent.OnMonthStart`（`run.go`） | `city.VoiceScheduler.speakOne`（`city/voice.go:101`） |
| 工具 | **全部 24 个**（经济 + 感知行动） | **无工具** |
| Memory | 局内 `Memory`（24 月滚动） | **无** |
| 并发 / 超时 | 房间信号量（默认 8）+ 单 Agent 决策 20s | 共享 `llm.LinePool`，Acquire+调用 15s |
| maxTokens | 1024（决策循环） | 128（极短对话） |
| prompt | 全量人设 + 经济状态 + 感知上下文 + Memory | 仅姓名 / 职业 / 人格 / opening_hook 一句话人设 |
| 调度 | 月度事件驱动（`wakeBots` → `runOneBot`） | 月结后抽样（`per_month`，默认 4，clamp [0,32]） |

> **设计原则**：AgentClass 是**业务身份**，不是调度档位。「都是城市居民」→ 同一 AgentClass；
> 「调用多深」→ 由调用方预算控制（12-CityHuman重构 §2.2）。

> **注记（第三条 LLM 路径）**：背景层除 `VoiceScheduler` 外还有 `city.ResidentDriver`（批次 17 全民驱动，
> 与 VoiceScheduler **互斥**、driver 优先；`set_intent` + `speak` 两工具、256 tokens、worker 池 + 线路池）。
> 它同样以 `LsmAgentGame-City-Human` 出站，但**不经过 `vcplayer.Agent`**（只复用其 `TokenBucket` 做限速）。
> 批次 81 的总线接入对两者透明：System Agent 拿到的是**房间快照**，不区分调用路径。

---

## 2. 焦点层：装配与调度

### 2.1 装配（`Manager.ensureAgentsWithPool`，`manager.go:496`）

```
vcplayer.NewAgent(roomID, userID, modelKey, modelName, seat, maxActions, decisionTimeout)
  ├─ BindRegistry(registry)          // LLM 注册表（每次 wake 现取，reload 后自动生效）
  ├─ SetLLMRateLimit(interval)       // 批次25：LLM 令牌桶（配置 agent_llm_min_interval_ms）
  ├─ BindLLMCallHook(fn)             // 批次25：调用计数（房间 llmStats 可观测性）
  ├─ BindLinePoolSource(fn)          // 池模式座位（ModelKey==""）才注入
  └─ BindRunner(NewAgentRunner(r, seat))   // 引擎桥（ToolRunner + TranscriptSink）
```

- `maxActions = cfg.BotMaxActionsPerMonth`（缺省 3）；`decisionTimeout = cfg.AgentDecisionTimeoutSec`（缺省 20s）。
- 池驱动座位（`ModelKey==""`）在线路池可用（`Total()>0`）时同样建 Agent；池不可用则跳过（防无 LLM 空转）。

### 2.2 调度（房间侧）

```
Start() / 每轮 settle 后
  └─ room.wakeBots()                       // room.go:1676（锁外路径）
       └─ go room.runOneBot(seat, agent)   // room.go:1714
            ├─ agentSem 满槽 → per-seat 单槽 pending（批次25，替代无界排队）
            └─ Manager.runAgentMonth(r, seat, agent)
                 ├─ BuildContextForAgent(r, seat)   // agent_runner.go:1047（持锁快照，锁外消费）
                 └─ agent.OnMonthStart(ctx, gc)
```

- `BuildContextForAgent` 在**持锁态**构造 `vctypes.GameContext` 全量快照（市场 / 本人三表 / peers /
  近期事件流水 / 感知上下文 / 行为金融小节 / 真实世界小节…），Agent 侧锁外只读。
- 已出局座位（`!p.Alive`）不再构建上下文，避免每轮浪费一次 LLM 调用。

---

## 3. ReAct / 决策循环（`run.go::OnMonthStart`）

月度事件驱动，单 Agent 单次决策的完整链路：

```
1. BeginDecision(seat, month)      // 房间级 expected-month 调度令牌（防旧月 prompt 延迟 apply）
2. sysBlocks = SystemPromptBlocks(ctx.MyCard)     // 5 段 SystemBlock（§14.1 wire）
   userText = UserPrompt(ctx, memory.RenderForPrompt())
   tools    = BuildTools()                        // 24 个，全部座位相同
3. for round := 0; round < 3; round++ {           // 工具循环 ≤3 轮
      rateAllowLLM()  // 令牌桶空 → 中止本月思考（走 submit 兜底，绝不无限重试）
      resp := callProvider(...)                   // ChatStreamAccumulate 流式（§197 字节续命）
      publishDecision(...)                        // 每轮先发布 transcript（避免提交/结算竞态）
      tus := toolUseBlocks(resp); 无 tool_use → break
      逐个 DispatchTool(name, input)：
        · res.Budget 置位 → actionsUsed++；达到 actionsLimit → 中止本轮后续工具与
          下一轮，强制收口（动作照常结算，不再派发新动作）
        · submit_month → finalize + return（不计预算）
       appendMessages(assistant content + tool_result)   // 严格 user/assistant 交替（§14.1）
   }
4. finalizeTranscript(...) + runner.SubmitMonth(seat)   // 默认收口（超时/失败/无 tool_use/预算耗尽）
```

**预算与限流（硬约束）**：

| 项 | 值 | 来源 |
|---|---|---|
| 月动作预算 | `BotMaxActionsPerMonth = 3`（`res.Budget` 驱动，非工具名静态查表） | `tools.go` / `run.go` |
| speak | 每月 area+private 合计 ≤2 次 + 30s 令牌桶节流 | `agent.go::allowSpeak` |
| 感知 see/hear/smell | 每月各 ≤2 次（Agent 侧计数，`OnMonthStart` 重置） | `tools_sense.go` |
| LLM 调用节流 | 令牌桶容量 2、每 `agent_llm_min_interval_ms`（缺省 30s，clamp [5s,300s]）补 1 枚 | `rate.go` |
| 决策超时 | 20s；超时 → watchdog 强制 `submit_month`（summary="timeout"） | `run.go` |
| 房间并发 | `AgentConcurrency`（缺省 8）+ per-seat pending 单槽 | `room.go` |
| panic 兜底 | `recover` → 强制 submit + Error 日志（单 bot 异常不得拖垮房间） | `run.go` |

**消息流纪律**：assistant 段必须携带原始 `tool_use` 块、tool_result 成对入流（CLAUDE.md §14.1），
否则上游 400 `tool result's tool id not found`。

---

## 4. 工具集（24 个，以 `BuildTools()` / `ToolNames()` 为准）

| 组 | 工具 | 数量 | 说明 |
|---|---|---|---|
| 基础 | `check_state` / `speak` / `submit_month` | 3 | 查询（免预算）/ 发言（≤2/月）/ 收口 |
| 金融合并 | `asset_trade` / `bank_loan` / `savings` / `query_finance` / `insurance` | 5 | 批次 35 合并（3+4+2+4+3 → 5） |
| 代际财富 | `family` | 1 | 批次 52（query / plan_inheritance / pay_support_extra / upgrade_education） |
| 生活 | `activity` / `set_consumption` / `answer_survey` / `side_business` | 4 | 生活动作 / 消费档 / 调研 / 副业 |
| 行为金融 | `behavior` | 1 | 批次 62（query / reflect / defer / merge） |
| 真实世界 | `realworld_info` | 1 | 批次 65/78（5 op，见 §8） |
| 交易合并 | `market_listing` / `negotiate` / `p2p_lending` / `auction_bid` / `info_market` | 5 | 批次 35 合并（12 → 5） |
| 感知行动 | `see` / `hear` / `smell` / `move` | 4 | 批次 12 §CityHuman重构（见 §5） |
| **合计** | | **24** | |

**预算判定**：批次 35 起由 `dispatchToolResult.Budget` 按 **(工具, op)** 置位驱动，
不再按工具名静态查表（同名工具不同 op 预算不同）。免预算例外：`check_state` / `submit_month` /
`see` / `hear` / `smell` / `savings` 全部 op / `query_finance` 全部 scope / `bank_loan.probe` /
`insurance.status` / `market_listing.view` / `answer_survey` / `family.query` / `family.plan_inheritance`。

**引擎桥**：`ToolRunner` 接口（`tools.go`）56 方法，由 `game/virtual_city/agent_runner.go` 实现
（in-process，不走 WS）；批次 35 工具收敛**不改接口任何方法**（收敛只发生在 wire 定义 + 派发层），
引擎零改动。

---

## 5. 感知系统（批次 12 §CityHuman重构）

| 工具 | 现实尺度 | 语义 | 预算 |
|---|---|---|---|
| `see` | ≈500m | 同城区：人（邻居姓名/职业/状态）/ 物（地标、本区挂牌）/ 事（本区本月事件） | 免预算，每月 ≤2 |
| `hear` | ≈100m | 同城区近处：公开发言摘录、城市之声、环境声、事件动静 | 免预算，每月 ≤2 |
| `smell` | ≈50m | 所在城区气味画像（基底 + 当月事件气味叠加） | 免预算，每月 ≤2 |
| `move` | — | `walk`/`run` 区内移动；`bus`/`metro`/`taxi` 跨城区（迁区，价格随 CPI 浮动） | **耗 1 点月预算** |
| `speak` | — | `scope=area` 同城区公开放话；`scope=private` 对指定座位耳语（仅对方可见） | 每月合计 ≤2 + 30s 节流 |

- 结果统一为 `vctypes.SenseResult`（`context.go`）：`People` / `Things` / `Events` / `Utterances` /
  `Smells` / `Sounds`，**确定性构造，不调 LLM**。
- 感知结果写入 `bot_contexts[].last_senses`（批次 70 富形状），仅本人 + 观战者可见，**不走全量广播**。
- 感知人物全部来自已锚定真实档案（`profile.go`），档案未就绪时降级合成姓名并省略 `card_id`。

---

## 6. 记忆系统

### 6.1 局内 Memory（`memory.go`，已接线）

```go
type Memory struct {
    Monthly        []MonthDecision // 最近 24 月滚动
    KeyFacts       []string        // 上限 20 条（买房/破产/结婚/失业等大事）
    CompactSummary string          // 规则式压缩（>24 月触发，不调 LLM）
}
```

- prompt 注入：最近 **6** 条 MonthDecision 全文 + `CompactSummary` + `KeyFacts`。
- 压缩为**规则式**（把溢出的最早月份拼成 `CompactSummary`，不调 LLM、零额外成本）；
  明细恒保留最近 24 月。
- 决策摘要同时写 `BotTranscripts`（`last_decision_summary` / `last_tool_input` / `last_tool_result`），
  供前端思维面板渲染。

### 6.2 跨局 MEMORY.md 迭代 —— **尚未接线**（P2）

`03-Agent设计` §10 差异表明确标注：虚拟城市记忆 = 「P0 局内 24 月滚动 + 规则式压缩（**跨局 P2**）」。
当前仓库的跨局 `MEMORY.md` 机制（`t_lsm_game_agent_memory` + `service.AgentMemoryService`）
仅接线狼人杀 / 德扑，**虚拟城市无对应桥**。本节为现状说明；跨局迭代属后续项，**不在批次 81 范围**。

---

## 7. 行为金融（批次 62 §20261020-62）

- `behavior` 工具四 op：`query`（行为画像：已识破偏差 / 锚定指数 / 羊群指数 / 心理账户，免预算）、
  `reflect`（`light` / `deep` / `preview`，耗 1 预算）、`defer`（延期支付，3 个月后 ×1.3，耗 1 预算）、
  `merge`（合并心理账户，耗 1 预算）。
- 上下文小节（引擎持锁预渲染，`vctypes.GameContext`）：`BehaviorBrief`（**仅含已识破偏差**，
  未识破的偏差名绝不可出现，否则教学闭环失效）、`HerdBrief`（羊群方向 + 同向人数，不泄露金额）、
  `BubbleBrief`（泡沫 / 集体杠杆）、`LiquidityBrief`（资产流动性价差）。
- 反身性：居民集体行为反作用于宏观（消费 → 物价 → 就业），由引擎数值演化承载，Agent 只读。

---

## 8. 真实世界信息（批次 65/78）

`realworld_info` 工具（`tools_realworld.go`，只读、不耗预算、零额外 LLM 成本）：

| op | 语义 |
|---|---|
| `overview` | 通道状态 + 各类目最近更新与新鲜度（age_seconds/stale）+ 头条 3 条（默认） |
| `quotes` | 行情，各类目前 5 条，可带 `symbol` 精确过滤 |
| `news` | 最近 10 条新闻（`keyword` 标题过滤，大小写不敏感） |
| `history` | 某 symbol 的自采价点序列 `{t,p}`（`symbol` 必填） |
| `sources` | 采集源清单与各类目健康度 |

- 数据源：`citychrome.Get()` 全局 `FeedStore`（东财 / 腾讯等公开源），**不依赖引擎 ToolRunner**。
- 通道未启动（nil 单例）返回明确错误文本，不 panic。
- 真实世界数据仅供叙事与聊天谈资，**不影响游戏内市场机制**（批次 66 的外盘情绪锚走独立链路）。

---

## 9. 总线接入（批次 81）

### 9.1 citybus 契约（`ServerGo/agent/citybus/bus.go`）

- 两级寻址：类级 `Register(class, handler)`（全局唯一）/ 房间级 `RegisterRoom(roomID, class, handler)`；
  `Send` 房间级优先、回退类级。
- `Handler = func(ctx, Message) (any, error)`，在**调用方 goroutine** 执行，超时由调用方 ctx 控制。
- `Message{From, To, Type, RoomID, Payload, TraceID}`；`Post` 走 cap 64 异步队列（满则丢弃 + Warn）。

### 9.2 本包新增（`ServerGo/agent/vcplayer/system_bus.go`）

```go
// ResidentBrief 中性结构(避免 vcplayer → game/virtual_city 反向依赖)。
type ResidentBrief struct {
    Seat     int    // 引擎座位下标(0 起)
    Nickname string
    ModelKey string
    Alive    bool
    Cash     int64
    District string
}

// RoomResidentsSource 由 main.go 装配层注入(从 vcMgr 取房间快照)。
type RoomResidentsSource interface {
    SystemBusResidents(roomID string) ([]ResidentBrief, error)
}

// CityRoomSource 全局注入点(装配层在 main.go 设置)。
var CityRoomSource RoomResidentsSource

// SystemBusResidents 是 LsmAgentGame-City-System 经 citybus 调用本 Agent 的
// 薄委托入口(批次81)。返回本房居民摘要的人读文本。
func SystemBusResidents(ctx context.Context, roomID string) (string, error)
```

**实现要点**：

1. **禁止 import `game/virtual_city`** —— 该包已 import `vcplayer`（`agent_runner.go` 等），反向依赖成环；
   因此摘要类型定义在本包，由装配层适配。本包也**不 import `citybus`**（注册在 main.go）。
2. `CityRoomSource == nil` → 返回可读说明（`"居民摘要不可用:CityRoomSource 未注入…"`），**不返回 error**。
3. 输出按座位升序，座位显示沿用前端惯例 `#Seat+1`：

```
房间 R1 居民 12 人(存活 12):
#1 张三(ModelKey=DeepSeek-model, 区=d1, 现金=12345)
#2 李四(ModelKey=Kimi, 区=d2, 现金=0, 已死亡)
...
```

4. `ctx`：先检查 `ctx.Err()` 短路；若 source 额外实现可选的
   `SystemBusResidentsContext(ctx, roomID)`，则透传 ctx（支持调用中取消）；否则普通调用
   （source 为纯内存快照，无阻塞 I/O）。nil ctx 降级 `context.Background()`。

### 9.3 集成方收口（main.go 装配层，B7）——已落地

> 集成 SubAgent 已完成接线（`main.go`）：适配器放**装配层**而非 `game/virtual_city`
> —— `vcplayer` 禁止反向 import `game/virtual_city`（成环），`game/virtual_city`
> 也不应感知总线；装配层是唯一同时持有两侧的位置。

**① `main.go` 内适配器 `cityHumanBusSource`**（实现 `vcplayer.RoomResidentsSource`）：

```go
type cityHumanBusSource struct{ mgr *virtual_city.Manager }

func (s cityHumanBusSource) SystemBusResidents(roomID string) ([]vcplayer.ResidentBrief, error) {
    r := s.mgr.Get(roomID)
    if r == nil {
        return nil, fmt.Errorf("virtual_city room %s not found", roomID)
    }
    r.MuLock()
    defer r.MuUnlock()
    w := r.EngineLocked()
    if w == nil {
        return nil, fmt.Errorf("virtual_city room %s not started", roomID)
    }
    out := make([]vcplayer.ResidentBrief, 0, virtual_city.MaxSeats)
    for seat := 0; seat < virtual_city.MaxSeats; seat++ {
        p := w.Players[seat]
        if p == nil {
            continue
        }
        out = append(out, vcplayer.ResidentBrief{
            Seat: seat, Nickname: r.Nicknames[seat], ModelKey: r.SeatModelKeys[seat],
            Alive: p.Alive, Cash: p.Cash, District: p.District,
        })
    }
    return out, nil
}
```

要点：**只读**（持 `r.mu` 读字段快照，§92a 不回调房间方法）；房间不存在 / 未开局 → error
（System Agent 如实告知用户）；`Seat` 为 0 起引擎座位，文本渲染 +1。

**② `main.go` 注入 + handler 薄委托**（原 TODO 锚点 `main.go:1085` 一带）：

```go
vcplayer.CityRoomSource = cityHumanBusSource{mgr: vcMgr}   // Register 之前注入
if err := citybus.Default().Register(string(agentroot.AgentClassCityHuman),
    func(ctx context.Context, msg citybus.Message) (any, error) {
        return vcplayer.SystemBusResidents(ctx, msg.RoomID)
    }); err != nil {
    logger.L().Warn("register City-Human bus handler failed", zap.Error(err))
}
```

### 9.4 与 01-方案设计 §3.1 的偏差说明

| 01 文档 §3.1 | 本批实际 | 原因 |
|---|---|---|
| City-Human 注册为**房间级** | 维持 main.go 既有**类级**注册 | 委托无状态、roomID 由 `msg.RoomID` 传入；类级零额外接线（无需随建房/销毁 Register/Unregister）。若未来需要按房隔离状态再改房间级 |
| handler 行为含「广播系统公告，下一轮月度决策可见」 | 本批只实现**只读居民摘要** | 任务范围限定；广播公告需引擎侧新增加载/下发通道，另行立批 |

---

## 10. 纪律汇总

1. **只经 `ToolRunner` 调引擎** —— Agent 不直接改 `World`；所有经济语义走 `game/virtual_city/actions.go`
   与人类同一代码路径，无分叉。
2. **预算即硬约束** —— 月动作 ≤3、speak ≤2、感知各 ≤2、LLM 令牌桶；超限**拒执行 + 兜底收口**，
   绝不无限重试。
3. **流式续命** —— LLM 调用走 `ChatStreamAccumulate`，字节到达即刷新超时（§197）。
4. **依赖方向** —— `vcplayer` → `vctypes` / `llm` / `errcode` / `citychrome`；
   **不** → `game/virtual_city`（防环）、**不** → `citybus`（注册在装配层）。
5. **可观测** —— transcript 每轮发布；LLM 调用计数钩子；失败 Error 日志带 room_id / seat。
6. **失败兜底** —— 任何 panic / provider 异常 / 超时都收敛到 `submit_month`，单 bot 不拖垮房间。

---

## 11. 验收

- [x] `gofmt -l agent/vcplayer/` 零输出（含顺带修正 `memory.go` 既有格式偏差）。
- [x] `go build ./...` 通过。
- [x] `go test ./agent/vcplayer/... -count=1` 通过（新增 8 个 `SystemBusResidents` 用例：
      未注入兜底 / 正常格式化 / 空房间 / error 透传 / ctx 感知源 / 取消短路 ×2 / nil ctx）。
- [x] `SystemBusResidents` 输出格式与 §9.2 示例一致（座位升序、`#Seat+1`、死亡标记、存活计数）。
- [ ] 集成后 E2E：System Agent `send_agent_message(target_agent="LsmAgentGame-City-Human")`
      → 返回居民摘要文本（由 integration-tester 收口）。
