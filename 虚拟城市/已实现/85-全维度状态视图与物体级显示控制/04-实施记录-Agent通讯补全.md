# 批次 85 · 实施记录 —— Agent 通讯补全(§3.3)

> 职责线 3(`backend-dev` · Agent 通讯线)。方案契约:[`01-方案设计.md`](01-方案设计.md) §1.3 / §3.3 / §7.1。
> 上一批边界来源:[`../84-全维度事件体系与Agent双向对话/03-实施记录.md`](../84-全维度事件体系与Agent双向对话/03-实施记录.md) §8 第 10、13 条。

## 0. 一句话结论

居民对话与游戏内 Agent 已共享 LLM 令牌桶(`AgentForSeat` 只读句柄 → 装配层
优先复用游戏内实例,对话消耗计入房间 LLM 统计);房间级旧 `Register` 迁移
`RegisterWith` 补能力描述 —— City-WebGL-Show 已迁,**City-System 的迁移越出
本线工作面(§29.1 Agent 包禁区),已留精确迁移清单交接**。

## 1. `AgentForSeat` 只读契约与边界

**新文件**:`ServerGo/game/virtual_city/room_agent_handle.go`(该包本批唯一改动;
Go 方法跨文件同包定义,不动 `room.go` 等在跑文件)。

```go
// AgentForSeat 返回座位上的 City-Human Agent(未启动/越位/非居民返回 nil)。
// **只读句柄访问器**:仅取既有实例指针,不新建、不启动/停止 Agent、不改
// r.agents 生命周期、不改任何游戏状态;调用方不得用它改游戏状态。
func (r *VirtualCityRoom) AgentForSeat(seat int) *vcplayer.Agent
```

| 边界 | 行为 | 实测 |
| --- | --- | --- |
| `r == nil` | 返回 nil,不 panic | `TestAgentHandle_AgentForSeatContract` |
| 房间未启动(`r.World == nil`) | **返回 nil 且不 panic**(即使 Agent 已装配 —— hydrate 窗口) | 同上(跨线实测反馈钉死:未开局不放行句柄) |
| seat 越界(`<0` / `>= MaxSeats`) | 返回 nil | 同上 |
| 非居民(人类座位)/ 空位 / Agent 未装配(`r.agents` 无条目) | 返回 nil | 同上 |
| 命中 | 非 nil,**两次调用同一指针**(== `r.agents[seat]`,证明不新建) | 同上 |

- 持 `r.mu` 只读、**禁止回调房间**(§92a);返回后调用方在锁外使用句柄安全
  (`vcplayer.Agent` 方法自带锁)。
- **只读性实测**:`TestAgentHandle_AgentForSeatReadOnly` —— 调用前后
  `World.StateHash()`(既有世界快照,批次 71)+ transcript 快照 + 房间字段
  摘要 + `r.agents` 指针表**逐字段相等**(覆盖已开局/未开局两种状态)。
- 零新增依赖方向:`vcplayer` 已是 `game/virtual_city` 既有依赖(room.go 已 import)。

## 2. 共享桶取用顺序(`main.go::cityHumanAnswerer`)

`ResidentAnswer` 取 Agent 顺序(§3.3.2):

1. **先 `room.AgentForSeat(seat)`** —— 命中即用**游戏内实例**:与月度决策
   **共用同一令牌桶**(`SetLLMRateLimit` 装配的 `TokenBucket`)与**同一
   `BindLLMCallHook`**(`manager.go::ensureAgentsWithPool` 绑定
   `r.noteSeatLLMCall`)⇒ 对话消耗计入房间 LLM 统计(修 §1.3 双桶双口径)。
   **只调 `AnswerQuestion`** —— 不重复 `SetLLMRateLimit`/`BindRegistry`/
   `BindLLMCallHook` 等装配调用(实测:`SetLLMRateLimit` 会 `NewTokenBucket`
   **重置令牌桶为满**,Bind* 会覆盖既有配置;房间创建时的装配即权威)。
2. 未命中**回退既有缓存实例**(`agentFor` 按 (房间,座位) 缓存)——
   **行为与批次 84 逐字一致**(零回归:该分支代码未动)。

**零回归证据**:

- `TestCityHumanAnswerer_FallbackCacheWhenNoInGameAgent`:AgentForSeat 未命中
  ⇒ 走缓存实例(同键复用、令牌桶跨调用生效)+ 两条错误路径(未知房间 /
  无可受访存活居民)与批次 84 一致;
- `TestCityHumanAnswerer_SharedInGameInstanceFirst`:命中 ⇒ 回复来自游戏内
  实例的注册表(缓存注册表 0 调用),且缓存 map **零写入**;
- 既有测试全绿(`ws/`、`agent/vcplayer/`、`api/`、`game/virtual_city` 全量
  `-skip MC`,见 §4)。

**共享桶生效的机制链**(§7.1-3):

1. 装配层命中游戏内实例(`TestCityHumanAnswerer_SharedInGameInstanceFirst`
   断言实例选择);
2. 该实例的 `AnswerQuestion` 触发 `BindLLMCallHook` → `noteSeatLLMCall` →
   `llmStats.seatCalls`(`TestAgentHandle_SharedInstanceFeedsRoomLLMStats`
   断言增量 = provider 真实调用数 = 2);
3. 共享令牌桶容量语义在该实例上生效(容量 2,第 3 次调用节流报错,provider
   调用恰 2 次)。

## 3. `Register` → `RegisterWith` 迁移清单(§3.3.3)

grep 口径:`hub.Register(` / `RegisterRoom(` 全量实测(`ws/handler.go:310` 的
`hub.Register(client)` 是 WS 连接 Hub,非 citybus,不相关)。

| Agent | 作用域 | 注册调用点 | 迁移状态 | MessageTypes(**实测来源**) | Description |
| --- | --- | --- | --- | --- | --- |
| `LsmAgentGame-City-Chrome` | 类级 | `main.go` 装配 | 批次 84 **已迁** `RegisterWith`,本批零改动 | `citychrome.BusMessageTypes = ["collect","message"]`(`citychrome/dialogue_bus.go` 的 `HandleBusMessage` 按 Type 分支实测) | `citychrome.BusCapabilityDesc`(既有常量) |
| `LsmAgentGame-City-Human` | 类级 | `main.go` 装配 | 批次 84 **已迁** `RegisterWith`,本批零改动 | `vcplayer.BusMessageTypes = ["message"]`(`vcplayer/dialogue_bus.go` 的 `HandleBusMessage`;其余类型回居民摘要) | `vcplayer.BusCapabilityDesc`(既有常量) |
| `LsmAgentGame-City-WebGL-Show` | 房间级 | `ws/game_service_virtual_city_system.go` | **本批迁移** `RegisterRoom` → `RegisterRoomWith` | `["", "run_display", "message"]` —— 实测该 handler 的 `switch msg.Type`:`""`/`"run_display"` 显示直通、`"message"` 对话式翻译、其余明确中文失败说明(`""` 是 run_display 的缺省别名,如实列入) | 「渲染显示 Agent:把结构化/自然语言显示请求翻译为 DisplayCommand 显示原语(run_display 缺省零延迟直通,message 走对话式翻译并逐条下发前端,回中文摘要)。」 |
| `LsmAgentGame-City-System` | 房间级 | `agent/citysystem/bus_handler.go` | **未迁移 —— 越出本线工作面**(§29.1 Agent 包归 City-System 线;详见 §5-1) | `["message"]` —— 实测 `handleBusMessage` 的 `msg.Type != BusTypeMessage` 单一分支 | 约定取值:「城市系统主 Agent:接收其它 Agent 的上行来信(message 类型;入队 + 前端来信卡片 + 回执确认,不代答),下一轮系统对话注入上下文。」 |

**不做**(按方案):`Register` / `RegisterRoom` 语义零改动(向后兼容,旧入口
仍可用)。

**验收口径**:`hub.Describe(roomID)` 对上述 4 条登记全部返回非空
`MessageTypes` + `Description`(`TestBusDescribe_RegisteredAgentsCarryMetadata`
钉死取值 + Describe 契约);**生产接线**中 City-WebGL-Show 已真实生效,
City-System 待 §5-1 交接完成后由 City-System 线单测钉死。

## 4. 单测与实测结果

| 断言(方案 §7.1) | 测试 | 结果 |
| --- | --- | --- |
| `AgentForSeat` 越界/未启动返回 nil,命中非 nil、两次同指针 | `game/virtual_city/room_agent_handle_test.go::TestAgentHandle_AgentForSeatContract` | PASS |
| `AgentForSeat` 只读(前后状态逐字段相等) | `TestAgentHandle_AgentForSeatReadOnly`(StateHash + transcript + 房间字段 + agents 指针表) | PASS |
| 共享桶生效(对话消耗计入房间 LLM 统计,`BindLLMCallHook` 触发) | `TestAgentHandle_SharedInstanceFeedsRoomLLMStats`(seatCalls 增量 = 2)+ `test_city_human_answerer_test.go::TestCityHumanAnswerer_SharedInGameInstanceFirst`(装配层命中游戏内实例) | PASS |
| 零回归(未命中时与批次 84 一致) | `TestCityHumanAnswerer_FallbackCacheWhenNoInGameAgent` + 既有测试全绿 | PASS |
| `list_agents` / `hub.Describe` 全部非空描述 | `TestBusDescribe_RegisteredAgentsCarryMetadata` | PASS |

编译与套件(2026-11-05 实测):

- `go build -o LsmAgentGame main.go` —— OK;
- `go test ./game/virtual_city/ -run 'TestAgentHandle|TestScenario|TestDimension' -count=1` —— ok;
- `go test . ./agent/vcplayer/ ./ws/ ./api/ ./agent/citybus/ ./agent/citysystem/ ./citychrome/ -count=1` —— 全 ok;
- `go test ./game/virtual_city/ -skip 'MC' -count=1` —— ok(MC 为存量失败,见批次 84 §7.2,非本批引入)。

§130 接线验证:`AgentForSeat` 定义 `game/virtual_city/room_agent_handle.go`,
唯一消费点 `main.go::cityHumanAnswerer.ResidentAnswer`(grep 全量命中 2 文件)。

## 5. 已知边界(后续批次 / 跨线交接)

1. **City-System 的 `RegisterRoom` 迁移待 City-System 线应用**
   (`agent/citysystem/bus_handler.go` 是 §29.1 Agent 包 + 方案 §6 职责线 1
   工作面,本线不动)。精确改法:`hub.RegisterRoom(a.RoomID, class, handler)`
   → `hub.RegisterRoomWith(a.RoomID, class, []string{BusTypeMessage},
   <上表 Description>, handler)`(建议同 citychrome/vcplayer 形态在
   citysystem 包内补 `BusMessageTypes`/`BusCapabilityDesc` 常量)。应用后
   `list_agents` 即对全部 Agent 给出能力描述(批次 84 §8-10 闭环)。
2. `TestBusDescribe_RegisteredAgentsCarryMetadata` 对房间级两条是**取值 +
   契约镜像钉死**(citysystem/ws 的生产注册无法从 `game/virtual_city` /
   `main` 包触达);City-WebGL-Show 的生产接线由 `ws` 既有 bus 测试覆盖
   (注册走 `getOrCreateSystemRoomAgent` 真实路径)。
3. City-WebGL-Show 的 `MessageTypes` 含 `""`(run_display 缺省别名,实测
   handler 接受)—— 如消费方(前端 list_agents 展示)不接受空串,由前端线
   过滤展示,**不得**回改实测口径。
4. 未开局窗口(World nil,hydrate 装配后未 Start)对话问答走**缓存实例**
   (批次 84 行为)—— 生产 `ResidentAnswer` 必经 `BuildContextForAgent`
   (要求 World 非 nil)后才取句柄,该窗口实际不可达,零行为差异。

## 6. 方案偏差登记(诚实记录)

| # | 方案 / brief 原文 | 实际 | 原因 |
| --- | --- | --- | --- |
| 1 | 工作面清单未列 `ws/game_service_virtual_city_system.go` | 该文件动 **1 个 hunk**(City-WebGL-Show `RegisterRoom` → `RegisterRoomWith`) | 方案 §6 将「Register 迁移」划给职责线 3(本线),且 §3.3.3 要求迁移**全部**旧调用点;该文件不属任何并行线工作面、不在「不要改」清单 |
| 2 | 新测试文件仅列 `room_agent_handle_test.go` | 增补 `ServerGo/test_city_human_answerer_test.go`(package main) | §7.1-3/4/5 断言装配层 `cityHumanAnswerer` 与 `hub.Describe`,须在 main 包内测试;命名按 §12 `test_<topic>_test.go` 规约 |
| 3 | §3.3.1 契约注释「未启动/越位/非居民返回 nil」 | 「未启动」明确为 **`World == nil`(房间未开局)**或 Agent 未装配,两种都返回 nil | 跨线(职责线 1)实测反馈钉死;对生产零行为差异(见 §5-4) |

## 7. 相关文档

- [`01-方案设计.md`](01-方案设计.md) §1.3 / §3.3 / §7.1 —— 本批契约
- [`../84-全维度事件体系与Agent双向对话/03-实施记录.md`](../84-全维度事件体系与Agent双向对话/03-实施记录.md) §6 / §8 —— 装配层现状与已知边界 10/13
- [`../../81-CitySystem-Agent与系统对话/01-方案设计.md`](../../81-CitySystem-Agent与系统对话/01-方案设计.md) §3.1 —— citybus 总线契约
