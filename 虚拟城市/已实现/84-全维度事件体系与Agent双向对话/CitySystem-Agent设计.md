# City-System Agent 设计（批次 84 · 全维度事件体系与 Agent 双向对话）

> 日期：2026-11-05 ｜ 分支：`main`（§10.1）
> 工作面：`ServerGo/agent/citysystem/`、`ServerGo/agent/citybus/`
> 上游衔接：批次 81《CitySystem-Agent设计.md》（基础版）、批次 83《系统对话窗口优化与场景目录扩充》（紧凑索引）、
> 批次 84《01-方案设计.md》§4.2 / §4.3 / §4.5 / §4.6、CLAUDE.md §29.1（每类 City-* Agent 独立 SubAgent）。
>
> 本文档是 `LsmAgentGame-City-System` 在批次 84 的**权威设计文档**，取代 81 号文档中
> 与本批冲突的部分（工具面 11→17、目录内联改维度索引、Agent 双向通讯）。81 号文档
> 中不冲突的部分（ReAct 主循环、线路策略、会话裁剪）继续有效。

---

## 1. Agent 定位与职责边界

**`LsmAgentGame-City-System`（`agentroot.AgentClassCitySystem`）是虚拟城市的系统主 Agent**，
是「人类 ↔ 城市」的**唯一自然语言控制面**。人类是观察者 + 条件触发者，通过与它对话来
查看状态、注入场景、控制模拟、驱动 3D 显示、与其它 Agent 协作。

### 1.1 职责

| 职责 | 说明 |
| --- | --- |
| 状态查询 | 城市状态 / 指标历史 / 城区 / 调研 / 预测报告 / 世界事件 |
| 场景注入 | 检索目录 → 注入场景（kind / 强度 / 持续 / 范围） |
| 模拟控制 | 暂停 / 恢复 / 查询运行状态（房主权限校验在房间侧） |
| 显示驱动 | 视角聚焦 / 高级显示（时间、天气、流速、图层、实体、机位、路径） |
| 真实数据 | 触发 City-Chrome 采集 |
| Agent 协作 | `ask_agent`（双向提问）/ `send_agent_message`（单向通知）/ `list_agents` |
| 来信消费 | 接收其它 Agent 上行消息 → 前端卡片 + 下一轮上下文 |

### 1.2 硬约束

1. **只能通过工具影响城市** —— 不得凭空描述未由工具返回的指标。
2. **禁止编造数据** —— 所有数字来自工具结果；子系统未启用时明确返回「未启用」，绝不凑数。
3. **kind 一律检索后注入** —— 目录规模可达万条，`inject_scenario.kind` **不带 enum**
   （批次 84 §2.1 阻塞点 A），模型必须先 `search_scenarios` 定位 kind。
4. **不代替人类做不可逆操作** —— 本批工具均为可逆或可观测。
5. **`sync.Mutex` 不可重入（§92a）** —— 本包只经房间的**只读** API 取数
   （`SystemSummaryText` / `SystemMetricHistory` / `ListSurveys` / `MuLock+EngineLocked`），
   持锁区间内不回调房间方法。

---

## 2. 工具面：11 → 17

> 契约：批次 84《01-方案设计.md》§4.2。`ToolNames()` 单测钉死 **17** 个，且与
> `BuildTools()` 顺序逐位对齐（防「声明了却从不接线」§130）。

| # | 工具 | 状态 | 参数 | 返回 | 失败语义 |
| --- | --- | --- | --- | --- | --- |
| 1 | `inject_scenario` | **改** | `kind`(string,**无 enum**) / `magnitude`1-5 / `duration_months`1-24 / `target` / `note` | 注入记录 + 即时效果列表 | kind 缺失 → 失败；kind 不在目录 → `[35055]`；参数越界 → `[35056]`；生效场景超上限 → `[35057]` |
| 2 | `search_scenarios` | **新** | `query`(可空) / `group`(可空) / `limit`(缺省30,上限200) | kind+名称+分组+默认强度+参数+描述摘要 | 未知分组 → `[35064]`；无命中 → 明确「检索无命中」 |
| 3 | `list_scenarios` | **改** | `group`(可空) / `limit` | **无 group ⇒ 维度索引**（13 行：维度名 · 条目数 · 代表 kind）；有 group ⇒ 该组条目（缺省 60，上限 200） | 未知分组 → `[35064]` |
| 4 | `query_city_state` | **扩** | `scope`(13 个) | 状态摘要文本 | 子系统未启用 → 明确「未启用」；room 未绑定 → 失败 |
| 5 | `query_metric_history` | **新** | `metric`(cpi/unemployment/wage/stock/gold/house/treasury/satisfaction) / `months`(1-60) | 时间序列（旧→新，带量纲） | 非法 metric → 失败；无历史 → 明确「尚无历史采样」 |
| 6 | `query_districts` | **新** | `district`(id 或名称前缀，可空) | 城区目录：id/名称/人口/房价指数/产业/设施说明 | 无匹配 → 失败「未找到匹配的城区」 |
| 7 | `query_world_events` | 保留 | — | 事件流（标题/来源/是否已注入/即时效果/次月结果） | — |
| 8 | `query_prediction_report` | 保留 | `run_id`(可空=最新) | 文本摘要 + 报告卡片事件 | run_id 未命中 → `[35058]` |
| 9 | `query_surveys` | **新** | — | 调研列表 + 各选项得票数与占比 + 理由摘录 | 无调研 → 明确「暂无」 |
| 10 | `control_simulation` | 保留 | `action`(pause/resume/status) | 状态文本 | 非房主暂停 → `[35011]`；未知 action → 失败 |
| 11 | `launch_survey` | 保留 | `question` / `options`(2-6) | 调研 id + 截止月 | 选项数非法 → 失败；已有进行中 → `[35017]`；月上限 → `[35018]` |
| 12 | `focus_camera` | 保留 | `district` \| `(x,z)` / `mode` | 「已下发视角指令: …」+ show_command 事件 | 总线目标未注册 → `[35061]` |
| 13 | `apply_display` | **新** | `phase`/`hour`/`weather`/`speed`/`layer`+`visible`/`entity_id`+`kind`/`category`/`preset`/`points` | 「已下发显示指令: …」+ show_command 事件 | 未知枚举 / points 越界 / 参数不全 → 失败文本 + `isErr` |
| 14 | `collect_realworld_data` | 保留 | `category`(可空) | 采集摘要 | 总线目标未注册 → `[35061]` |
| 15 | `ask_agent` | **新** | `target_agent` / `message` | **对方回包文本** | 目标未注册 → `[35061]`；超时 30s → `[35062]`；回包超长 → `[35063]` + 截断 |
| 16 | `send_agent_message` | 保留(**改**) | `target_agent` / `message` | 「已向 X 发送消息(单向通知)」 | 目标未注册 → `[35061]`；内容为空 → 失败 |
| 17 | `list_agents` | **扩** | — | **类级 + 房间级**目录：类名/作用域/支持的消息类型/一句话能力描述 | 总线未绑定 → 失败 |

### 2.1 `inject_scenario.kind` 移除 enum（阻塞点 A）

`scenarioKindEnum()` 会在 schema 里内联全部 kind：101 条 ≈ 2.6 KB 尚可，
**10000 条 ≈ 260 KB** 会直接撑爆 Anthropic 请求体。批次 84 起 kind 是普通
`string`，描述引导「先用 `search_scenarios` 检索」。单测钉死：`properties.kind`
**没有** `enum` 键。

### 2.2 `list_scenarios` 的两种形态

- **无 `group`** ⇒ 返回**维度索引**：每维度一行「维度名 · 条目数 · 代表 kind」，
  **体积恒定**（13 行 ≈ 800 B，目录怎么扩容都不变）。
- **有 `group`** ⇒ 返回该组条目（`limit` 缺省 60，上限 200），含描述与参数。

分组集由 `virtual_city.ScenarioGroupOrder()` 动态给出（13 组 = 既有 6 + 批次 84 新增 7），
**不硬编码 enum**（分组随目录扩容可变）；未知组名 ⇒ `[35064]` 失败文本回灌模型。

### 2.3 `query_city_state` 的 13 个 scope

既有 6 个（`overview`/`market`/`labor`/`treasury`/`election`/`all`）仍走房间侧
`SystemSummaryText`（**零回归**）；批次 84 新增 7 个由本包渲染（`renderExtendedScopeLocked`）：

| scope | 数据源 | 未启用时 |
| --- | --- | --- |
| `infrastructure` | 产业链 `SupplyChain` + 产业集群 `Clusters` | 「产业链子系统未启用」 |
| `public_health` | `PublicSvc`（医疗/教育/养老/住房/就业） | 「公共服务(医疗)子系统未启用」 |
| `environment` | 引擎无环境生态子系统 | **恒**「环境生态子系统未启用」 |
| `security` | 引擎无治安子系统（`Regulators` 是市场监管） | **恒**「公共安全(治安)子系统未启用」 |
| `transport` | 交通类商品价格指数 `Goods.Items["transport"]` + 产业链节点数 | 「交通类商品价格未启用」 |
| `culture` | 教育质量 `PublicSvc.EduQuality` + 教育类商品价格指数 | 「公共服务(教育)子系统未启用」 |
| `daily` | 消费篮子 8 类价格指数 + 社会结构 `Society` | 「商品市场子系统未启用」/「社会结构统计未启用」 |

> **绝不编造**：`environment` / `security` 两个维度在当前引擎**没有对应子系统**，
> 恒定返回「未启用」+ 可注入对应维度场景的提示，而不是拿别的数据凑数。

---

## 3. 系统提示词结构

`BuildSystemBlocks(maxSteps int, incoming []IncomingMessage)` 产出（经
`sysprompt.EnsureHead` 注入三段式头，§14.3 **单一注入点**，本包不自己拼标准头）：

```
① 三段式标准头（sysprompt.EnsureHead 注入；逐字节稳定，prompt cache 前缀）
② 角色段 roleText（身份 / 城市背景 / 能力边界 8 条）
③ 步数纪律（单轮 ≤ maxSteps 步 + [step i/N] 进度前缀）
④ 场景维度索引（恒定体积；13 行「维度 · 条目数 · 代表 kind」）
⑤ 来信段（可选；≤5 条，时间倒序；来信为空不加）
```

### 3.1 维度索引取代全目录内联（阻塞点 B）

批次 83 的 `ScenarioCatalogIndex()` 把**全目录**内联进 system：101 条 ≈ 3 KB 尚可，
1000+ 条 ≈ 30 KB —— 直接撑爆请求体，且与 §14.3「三段式头逐字节稳定（全体 Agent 共享
prompt cache 前缀）」冲突：目录一变，前缀就变，cache 全失效。

批次 84 起 system 内联 **`ScenarioDimensionIndexText()`**（维度索引）：

```
共 1131 个场景 · 13 个维度;每行「维度 · 条目数 · 代表 kind」:
- 市场与经济 · 21 个 · 代表:goods_price_shock / policy_shock / market_crash
- …（13 行）
需要某个具体 kind 时用 search_scenarios(query=关键词,group=维度名) 检索。
```

- **体积恒定**：每组 ≤3 个代表 kind，13 组 ≈ 800 B —— 目录扩容到万条也不变。
- **`ScenarioCatalogIndex()` 保留为 Deprecated**：仅供调试/对照全目录，**不再进 system**
  （注释已注明；单测 `TestBuildSystemBlocks_NoFullCatalog` 钉死哨兵 kind 不出现在 system）。
- 具体 kind 一律经 `search_scenarios` 检索（`virtual_city.ScenarioCatalogSearch`）。

### 3.2 来信段

```
【来自其它 Agent 的消息】
- 来自 LsmAgentGame-City-Chrome:已采集到今日 A 股行情
（以上消息来自其它 Agent 的主动上行,供你参考;需要追问请用 ask_agent。）
```

- 最多 5 条，**时间倒序**（最新在前）；
- 单条正文按 rune 截断到 200（防撑爆 system）；
- **来信为空时不加该段**。

### 3.3 体积断言（单测钉死）

- system 全文 **≤ 8 KB**；
- **不含全目录 kind**：用一个「非代表 kind」的哨兵断言不出现（`TestBuildSystemBlocks_NoFullCatalog`）。

---

## 4. Context 管理

沿用批次 81 的 `ConversationStore`（`context.go`）：

| 项 | 策略 |
| --- | --- |
| 键 | `roomID + "/" + userID`（每用户每房一条独立会话） |
| 值 | `[]llmtypes.Message`（**不含 system**；system 每轮重建） |
| 裁剪 | 超 `MaxMessages`(40) 保留最近 N-10 + 前 10；切口不拆散 tool_use/tool_result 对 |
| 压缩 | `Compact` 接口已实现，**默认不触发** |
| 出站闸口 | `sanitizeForWire`：块白名单 / 合并相邻同角色 / tool 配对校验（§14.1） |

### 4.1 来信队列（批次 84 新增）

`ConversationStore` 新增**独立来信队列**（不进 messages 历史，避免污染对话历史与
前端历史展示）：

| 项 | 值 |
| --- | --- |
| 键 | `incomingKey(roomID) = roomID + "/#incoming"`（**房间级共享收件箱**） |
| 容量 | 环形 ≤ **20**（超出丢最旧） |
| 取用 | `TakeIncoming(roomID, 5)`：最近 5 条，**时间倒序**，取走出队 |
| 注入 | 下一轮 `BuildSystemBlocks` 的「【来自其它 Agent 的消息】」段 |
| 消费语义 | **同一条来信只注入一次**（取走出队；房间内任一用户开一轮即消费） |

> 房间级而非用户级的理由：总线来信是发给「本房系统 Agent」的（不区分用户），
> 放在用户维度会重复派发给同房每个用户。取走出队保证不重复注入；若需要
> 跨轮保留，未来可改为「按用户标记已读」（已知边界 §7）。

---

## 5. Agent 双向通讯协议

### 5.1 消息形态

所有 `message` 类型载荷统一为 `map[string]any{"text": "<正文>"}`；兼容批次 81 的
裸字符串载荷（`busMessageText` 两种都解析）。消息类型常量 `BusTypeMessage = "message"`
与 citychrome / vcplayer / citywebgl 逐字一致。

### 5.2 `ask_agent`（双向，等待回复）

```
System ──hub.Send(roomID, from=City-System, to=TARGET, "message", {text})──▶ TARGET
System ◀─────────────────────────── 回包文本 ──────────────────────────── TARGET
```

| 情形 | 回灌文本 |
| --- | --- |
| 目标未注册 | `[35061] agent not registered: …` |
| 注册但 30s 内未回 | `[35062] 目标 X 未在时限内回应` |
| 回包超长（> `tool_result_limit`） | `[35063] 回包超长,已截断` + 截断正文 |
| 正常 | 回包文本原样回灌 |

- 等待实现：`hub.Send` 在独立 goroutine 执行（handler 跑在调用方 goroutine），
  `select` 限时返回 —— 目标 handler 挂住不拖垮本轮。
- **防回环**：目标为 `LsmAgentGame-City-System` 自身时直接拒绝（`不能向自己提问`）。
- 超时 30s（`askAgentTimeout`）；单测可注入更短的 `a.askTimeout`。

### 5.3 `send_agent_message`（单向通知）

```
System ──hub.Post(roomID, from=City-System, to=TARGET, "message", payload)──▶ TARGET
```

- **改用 `hub.Post`**（批次 84）：不等回复、不阻塞本轮；handler 返回值丢弃。
- 目标未注册仍回 `[35061]`（`Post` 对未注册目标是**静默丢弃**，不能给确定反馈，
  故先经 `hub.Describe` 查注册表）。

### 5.4 上行来信（Agent → System → 人类）

`(*Agent).RegisterBusHandler(hub)` 把 City-System 注册为**房间级**总线 handler：

```
任意 Agent ──hub.Send/Post(roomID, from=X, to=City-System, "message", {text})──▶ City-System
City-System:
  1. emit SystemEvent{Kind: "agent_message", Data: {from,text,ts}}   → 前端「📨 来自 X」卡片
  2. store.AppendIncoming(incomingKey(roomID), {from,text,ts})        → 来信队列(环形 ≤20)
  3. 回执「已收到…将在下一轮系统对话上下文中参考」(**不代答**)
```

- **下一轮** `Run` 构造 system 时 `TakeIncoming` 取最近 5 条注入来信段。
- 防御分支：`Type != "message"` 不入队；`From == City-System` 自我发信忽略（防回环）；
  空正文忽略。均返回明确回执，不 panic。
- 回执**不代答**对方的问题（City-System 的回答走下一轮系统对话，由人类触发）。

### 5.5 `list_agents`（双层目录）

经 `hub.Describe(roomID)` 返回**类级 + 该房间的房间级** Agent：

```
- LsmAgentGame-City-Chrome | 类级 | 消息类型 collect/message | 采集真实世界行情与新闻
- LsmAgentGame-City-WebGL-Show | 房间级 | 消息类型 run_display | 渲染显示 Agent
```

能力描述经 `RegisterWith` / `RegisterRoomWith` 登记；旧 `Register` 注册的 Agent
这两项为空（不影响发现，显示「未声明」/「未登记描述」）。

---

## 6. 错误码（35061 – 35065）

| 码 | 常量 | 触发条件 | 触发工具 |
| --- | --- | --- | --- |
| 35061 | `ErrVirtualCityAgentNotFound` | 目标 Agent 未注册（本房不可达） | `ask_agent` / `send_agent_message` / `focus_camera` / `collect_realworld_data` |
| 35062 | `ErrVirtualCityAgentNoReply` | 目标 Agent 在 30s 时限内未回应（超时或正在忙） | `ask_agent` |
| 35063 | `ErrVirtualCityAgentReplyTooLong` | 目标 Agent 回包超长（已截断后回灌） | `ask_agent` |
| 35064 | `ErrVirtualCityScenarioGroupUnknown` | 场景分组未知（不在 13 组分组表内） | `list_scenarios` / `search_scenarios` |
| 35065 | `ErrVirtualCityDisplayCommandInvalid` | 显示指令参数非法（字段缺失或取值非法） | `apply_display` |

> 35055–35058（场景未知 / 参数非法 / 生效上限 / 预测运行不存在）与 35011（非房主）
> 沿用批次 68/81 的口径，**零回归**。错误码随 `errcode.Error()` 文本回灌给模型，
> 便于模型自纠（如 35055 → 去 `search_scenarios` 重查 kind）。

---

## 7. 装配层接入说明

装配层（`ServerGo/ws/`，**不在本包工作面**）在房间级 City-System Agent 建好后接线：

### 7.1 注册上行 handler

```go
// ws/game_service_virtual_city_system.go · getOrCreateSystemRoomAgent
ag := citysystem.NewAgent(r, cfg, s.vcSystemRegistry, citybus.Default(), store, emit)
ag.SetLinePolicy(r.CitySystemLine())
if err := ag.RegisterBusHandler(citybus.Default()); err != nil {
    logger.L().Warn("citysystem register bus handler failed",
        zap.String("room_id", roomID), zap.Error(err))
}
```

### 7.2 注销（房间销毁路径）

```go
// ws/game_service_virtual_city_system.go · removeCitySystemAgent
func (s *GameService) removeCitySystemAgent(roomID string) {
    s.vcSystemAgents.Delete(roomID)
    citybus.Default().UnregisterRoom(roomID, string(agentroot.AgentClassCityWebGLShow))
    citybus.Default().UnregisterRoom(roomID, string(agentroot.AgentClassCitySystem)) // 新增
}
```

> 同房重复注册会失败（总线同房同类不允许重复）；重建同 roomID 的测试房须先
> `UnregisterRoom` 再注册（与批次 81 的 City-WebGL-Show 同口径）。

### 7.3 其它 Agent 的接线（各自责任线）

| Agent | 侧 | 说明 |
| --- | --- | --- |
| City-Chrome | `citychrome.HandleBusMessage` | `message` 分支：提问 → 采集意图 → 结果摘要上行（`NotifySystem`） |
| City-Human | `vcplayer.HandleBusMessage` | `message` 分支：居民口吻回应；主动汇报走 `ReportToSystem` |
| City-WebGL-Show | `citywebgl` | `run_display` 分支：显示请求 → `DisplayCommand[]` |

> **零改动接入**：未来新增 Agent 只需 `hub.Register*` 并登记 `types`/`desc`，
> `list_agents` 立刻可见、System 可呼叫、其上行消息自动进对话窗（§130 合规）。

### 7.4 模型线路

沿用批次 81 的 `city_system.model_key` + `city_system_line`（`""` 沿用配置 /
`"random"` 加权随机 / 固定 key）。批次 84 不改线路语义。

---

## 8. 显示链路（`display.go` 中性 wire）

`citysystem/display.go` 与 `citywebgl/command.go` 各持一份**逐字一致**的镜像结构
（两包互不 import，经 citybus JSON 往返）：`displayRequest` / `displayCommand` /
`displayPoint`，字段名 / JSON tag / **字段顺序** 任一侧改动都必须双侧同步
（两侧各有 JSON 锚点单测钉死字面 JSON）。

- 显示指令类型常量 **14 个**（批次 81 的 6 个 + 批次 84 的 8 个）：
  `focus_district` / `focus_point` / `set_mode` / `highlight_district` / `annotate` / `clear`
  + `set_time_of_day` / `set_weather` / `set_time_speed` / `focus_entity` /
  `set_layer_visibility` / `set_filter` / `camera_preset` / `fly_path`。
- **只加字段、不加嵌套**：一条指令 = 一个 type + 一组扁平字段（旧前端遇未知 type 静默忽略）。
- `focus_camera` 与 `apply_display` 同构：构造中性 `displayRequest` → 经总线
  `run_display` 转发 City-WebGL-Show → `parseDisplayCommands` 解析 → `emitCommands`
  统一下发（**show_command 单一发射点**，批次 81 纪律）。
- `apply_display` **本侧严格校验**枚举与边界（非法 ⇒ isErr 失败文本，不转发）——
  渲染侧 Structured 路径对非法取值是「静默忽略」，不拦截会变成「什么都没发生」。

---

## 9. 已知边界

| # | 边界 | 影响 | 归属 |
| --- | --- | --- | --- |
| 1 | **WS 层结构化判据未覆盖批次 84 新参数** —— `game_service_virtual_city_system.go` 的内联判据仍是 `req.District != "" \|\| req.X != nil \|\| req.Z != nil`；仅带 `phase`/`weather` 等新参数的 `run_display` 会落到 LLM 路径（空 text 报错）。citywebgl 侧已提供 `HasStructuredParams()` 供 WS 层改用 | `apply_display` / `focus_camera` 仅带新参数时可能走 LLM 路径 | **WS 装配层**（不在本包工作面） |
| 2 | **来信队列为房间级共享收件箱** —— 取走出队，同房首个开轮的用户消费；同房其它用户看不到同一条来信 | 多用户同房时来信只注入一次 | 本包（可按用户标记已读扩展） |
| 3 | **`apply_display` 的 `entity_id` 无法校验真实性** —— 实体 id 只能来自上下文（前端实体注册表）；本侧只校验非空，不校验存在性 | 可能下发对不存在实体的聚焦指令 | 前端 `cityEntityResolver` |
| 4 | **`query_districts` 的「人口」是当前所在区口径** —— 按 `World.Players[].District` 统计存活居民，非常住人口 | 口径与「常住人口」不同 | 引擎（如需常住口径需新增字段） |
| 5 | **`query_city_state` 的 `environment` / `security` 恒「未启用」** —— 引擎无对应子系统 | 这两个维度只能看场景注入效果，无独立状态 | 引擎（如需建模） |
| 6 | **`ask_agent` 超时后目标 handler 仍在运行** —— goroutine 不回收，仅本轮不等待 | 慢目标的 handler 最终会返回但被丢弃 | 总线（如需取消语义需 handler 尊重 ctx） |
| 7 | **`list_agents` 的能力描述依赖注册方登记** —— 旧 `Register` 注册的 Agent 描述为空 | 描述缺失显示「未登记描述」 | 各 Agent 责任线（逐步补 `RegisterWith`） |

---

## 10. 单测锚点（硬性）

| 断言 | 测试 |
| --- | --- |
| `ToolNames()` 恰为 **17** 个且与 `BuildTools` 逐位对齐 | `TestToolNames` |
| `inject_scenario.kind` 无 enum | `TestInjectScenarioKindNoEnum` |
| system 全文 **≤ 8KB** 且不含全目录 kind（哨兵断言） | `TestBuildSystemBlocks_NoFullCatalog` |
| `ScenarioCatalogIndex()` 不再进 system（Deprecated 注释） | `TestScenarioCatalogIndex_Deprecated` |
| `apply_display` 合法参数产出对应 type 的命令 | `TestApplyDisplay_Valid`（7 分支） |
| `apply_display` 非法参数返回失败文本且 `isErr=true` | `TestApplyDisplay_Invalid`（6 分支） |
| `ask_agent` 未注册 → 35061 | `TestAskAgent_NotRegistered` |
| `ask_agent` 超时 → 35062 | `TestAskAgent_Timeout` |
| `ask_agent` 回包超长 → 35063 + 截断 | `TestAskAgent_ReplyTooLong` |
| `send_agent_message` 走 `Post`（不等待） | `TestSendAgentMessage_Post` |
| `list_agents` 类级 + 房间级 + 描述 | `TestListAgents_Extended` |
| 上行来信 → `agent_message` 事件 + 入队 + 下一轮 system 含来信段 | `TestBusHandler_IncomingMessage` / `TestBusHandler_SystemInjectedInSection` |
| `display.go` 镜像结构 JSON 锚点（字段名逐字） | `TestDisplayRequestJSONAnchor` / `TestDisplayCommandJSONAnchor` |
| `citybus.Describe` 覆盖类级 + 房间级；`RegisterWith` 不影响 `Agents()` | `TestDescribe_ClassAndRoom` / `TestRegisterWith_ZeroRegression` |

---

## 11. 相关文档

- [`01-方案设计.md`](01-方案设计.md) —— 批次 84 总方案（§4.2 工具表 / §4.3 双向通讯 / §4.5 显示原语 / §4.6 错误码）
- [`02-全维度事件体系对账表.md`](02-全维度事件体系对账表.md) —— 164 条用户清单对账
- [`CityWebGLShow-Agent设计.md`](CityWebGLShow-Agent设计.md) —— 渲染显示 Agent（14 显示原语 + 镜像 wire）
- [`../81-CitySystem-Agent与系统对话/CitySystem-Agent设计.md`](../81-CitySystem-Agent与系统对话/CitySystem-Agent设计.md) —— 基础版（ReAct 主循环 / 线路策略 / 会话裁剪继续有效）
- [`../83-系统对话窗口优化与场景目录扩充/01-方案设计.md`](../83-系统对话窗口优化与场景目录扩充/01-方案设计.md) —— 紧凑索引（批次 84 改维度索引）
