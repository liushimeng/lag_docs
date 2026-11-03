# 批次 81 · City-System Agent 设计（`LsmAgentGame-City-System`）

> 日期：2026-11-03 ｜ 分支：`main`（§10.1）｜ 职责线：`city-system`（§13.1 B2）
> 上游方案：[`01-方案设计.md`](01-方案设计.md) §3.2（主 Agent）、§5（场景 40）、§6（数据契约）、§8（风险）
> 依赖：`ServerGo/agent/citybus/`（总线）、`ServerGo/game/virtual_city/`（引擎只读桥 + 场景目录）、
> `ServerGo/llm/`（Provider / sysprompt 三段式）
>
> 本文是 City-System 的**单一设计说明**；实现事实以 `ServerGo/agent/citysystem/` 代码为准。

---

## 1. 定位

City-System 是虚拟城市的**系统主 Agent**：人类的**唯一自然语言控制面**，
替代旧「场景推演台」表单（批次 68/80）。人类说「帮我注入一个强度 3 的地震，
持续半年」，Agent 理解意图 → 选场景 → 调工具 → 控制模拟 → 把 3D 视角拉到事发地。

**职责边界**：
- **只被人类对话触发**（无自主行动）；自动注入仍走批次 80 规则引擎（确定性、零 RNG）。
- **工具只调既有房间 API**（`ApplyScenario` / `Pause` / `LaunchSurvey` /
  `PredictionReport` / `ScenarioCatalog`）+ 只读桥（`SystemSummaryText` /
  `WorldEventLogView`）——不新增平行账本、不改经济语义。
- **不做图形**：显示走 `City-WebGL-Show`（结构化显示指令），渲染由前端 3D 引擎执行。

---

## 2. AgentClassName 与注册

| 项 | 值 |
|---|---|
| AgentClassName | `LsmAgentGame-City-System` |
| 常量 | `ServerGo/agent/class_names.go::AgentClassCitySystem`（已登记 §24） |
| 实现 | `ServerGo/agent/citysystem/`（`Agent`，每房间一个实例，WS 层持有） |
| User-Agent | `LsmAgentGame-City-System/<AppVersion> <buildDateTime>`（§24，`AgentClassName` 出站拼装） |
| 提示词三段头 | `sysprompt.EnsureHead(..., AgentClassCitySystem)`（§14.3，单一注入点，Agent 不自拼） |
| 总线角色 | 只**发起**方（Send 三目标），自身不注册 handler |

---

## 3. 架构（对话 → ReAct → 工具 → 引擎 / 总线 → 显示）

```
人类自然语言 ──WS game.virtual_city_system_message──► Agent.Run(ctx, userID, text)
                                                        │
     ┌──────────────────────────────────────────────────┘
     ▼
 system = EnsureHead(角色段 + 步数纪律 + 场景目录40)      ← prompt.go(三段式 §14.3)
     ▼
 for step < MaxSteps:                                    ← agent.go runLoop
     callLLM(system, messages)                           ← sanitizeForWire 闸口(§14.1)
         ├─ ChatStreamAccumulate(§197 流式续命)            ← text_delta → 前端打字机
         └─ 节流槽 acquireLLM(llmSema 模式)
     if 无 tool_use → 结束
     for tu in ToolUses:
         Dispatch(ctx, userID, tu.Name, tu.Input)         ← dispatch.go
            ├─ inject_scenario …  → room.ApplyScenario(副作用)
            ├─ query_* …          → 房间只读桥(format.go 渲染)
            ├─ control_simulation → room.Pause(35011 房主校验)
            ├─ focus_camera       → citybus → City-WebGL-Show → show_command 事件
            ├─ collect_realworld_data → citybus → City-Chrome
            └─ send_agent_message / list_agents → citybus(任意已注册类)
         结果 = "[step i/N] " + 截断文本 → tool_result 回灌 + tool_result 事件
 持久化会话 → agent_end 事件(含最终文本 / 收尾说明)
```

**关键决策**：
1. **对话即控制面**：场景参数语义全部进系统提示词 + 工具 schema，人类不学机器语言。
2. **显示与决策分离**：`focus_camera` 经总线拿中性 JSON（`display.go` 中性 wire 契约），
   **不 import citywebgl** —— 两个包可独立演进。
3. **身份显式传递**：`userID` 随 `Run → runLoop → Dispatch` 调用链传递，不在 Agent
   上共享字段 —— 多用户并发对话时 `control_simulation` 房主校验不会串号。
4. **出站唯一闸口**：`callLLM` 入口 `sanitizeForWire`（§6），存储保留原始留痕。

---

## 4. 系统提示词设计（`prompt.go`）

**三段式结构**（`BuildSystemBlocks(maxSteps)`）：

```
① 计费元数据头  ─ sysprompt.EnsureHead 注入(§14.3,cc_agent_name=…City-System)
② 基础身份声明  ─ sysprompt.EnsureHead 注入
③ 核心行为规则  ─ sysprompt.EnsureHead 注入
─────────────────────────────────────────────────
角色段(roleText) + 步数纪律 + 场景目录(ScenarioCatalogText)
```

**角色段要点**（为什么这么写）：
| 要点 | 文本位置 | 动机 |
|---|---|---|
| 身份 = `LsmAgentGame-City-System`，系统主 Agent | 开头 | 计费/审计口径与 §24 一致 |
| 城市背景：全 Agent 模拟（非游戏），人类 = 观察者 + 条件触发者 | 【城市背景】 | 防止模型把城市当游戏乱改 |
| **只能**用工具影响城市、**禁止编造数据**、没依据说「暂未取到」 | 【能力边界】1-2 | Agent 编造指标是方案 §8 头号风险 |
| 注入前确认 kind 与强度档（L1-L5），缺省按目录默认并**先告知** | 【能力边界】3 | 不可逆操作保护 |
| 输出规范：先说**要做什么** → 调工具 → **做了什么 + 关键指标** | 【能力边界】4 | 用户实时看到「在干什么 / 处理方法」 |
| 查询一律走工具；`collect_realworld_data` / `focus_camera` 引导 | 【能力边界】7 | 减少幻觉路径 |
| 步数纪律：最多 N 步、`[step i/N]` 进度前缀、步尽直接总结 | 【步数纪律】 | 促收敛，控成本 |

**场景目录内联**（`ScenarioCatalogText()`，40 个场景分组渲染）：省一次
`list_scenarios` 模型往返；`list_scenarios` 工具保留作按需刷新。目录源 =
`virtual_city.ScenarioCatalog()` **动态生成**（`tools.go::scenarioKindEnum`
同源），目录扩缩两侧零同步成本。

**纪律**：三段式文本只经 `sysprompt.EnsureHead` 注入（幂等），prompt.go
**不自拼**标准头（§14.3 单一注入点）。

---

## 5. 11 个工具表（`tools.go` wire 定义 + `dispatch.go` 派发）

| 工具 | 参数（schema 约束） | 返回（回灌文本） | 引擎副作用 | 错误码（原样回灌） |
|---|---|---|---|---|
| `inject_scenario` | `kind`(enum=目录 40) / `magnitude` 1-5 / `duration_months` 1-24 / `target?` / `note?` | 注入记录 + 即时效果 | `room.ApplyScenario`（进 `World.Scenarios`） | 35055 kind 未知 / 35056 参数非法 / 35057 活动场景满 |
| `list_scenarios` | — | 40 场景目录文本（分组/kind/名称/默认强度/描述/参数） | 无（只读） | — |
| `query_city_state` | `scope`=overview\|market\|labor\|treasury\|election\|all | `SystemSummaryText(scope)` | 无（持锁只读） | — |
| `query_world_events` | — | 事件流文本（批次 80 反馈链） | 无（`WorldEventLogView` 快照） | — |
| `query_prediction_report` | `run_id?` | 文本摘要 + **报告卡片**（`report` 事件下发前端） | 无 | 35058 run 不存在 |
| `control_simulation` | `action`=pause\|resume\|status | 状态文本 | `room.Pause`（月结时钟冻结/恢复） | 35011 非房主 |
| `launch_survey` | `question`(≤100字) / `options[]`(2-6) | 调研 ID + 截止月 | `room.LaunchSurvey`（既有调研引擎） | 既有调研错误 |
| `focus_camera` | `district?` / `x?`+`z?`（二选一） / `mode?=orbit\|fly` | 指令摘要 | 经总线 → City-WebGL-Show → `show_command` 事件 | 35061 目标未注册 |
| `collect_realworld_data` | `category?=a_share\|hk\|us\|metal\|futures\|crypto\|macro\|news` | 采集回执 | 经总线 → City-Chrome（异步入 FeedStore） | 35061 |
| `send_agent_message` | `target_agent` / `message` | 发送确认 | 经总线单向投递目标 Agent | 35061 目标未注册 |
| `list_agents` | — | 已注册类级 Agent 清单 | 无（`hub.Agents()`） | — |

> **错误码透传纪律**：`*errcode.Error` 的 350xx 经 `Error()` 以 `[35055] 消息`
> 形态**原样回灌**给模型（不吞成泛化错误）；总线「agent not registered」经
> `busFailure()` 映射为 `[35061]`。模型据此自我纠正（换 kind / 换目标）。
> 房间未绑定 / 参数缺失等局部校验返回「失败:…」文本（isErr=true）。

**wire 定义与派发一致性**：`ToolNames()`（11 个常量）供单测钉死（§130）；
`Dispatch` switch 全量覆盖，未知工具名回「失败:未知工具」。

---

## 6. ReAct 主循环（`agent.go`）

```
Run(ctx, userID, text):
  key = roomID + "/" + userID
  store.Append(key, user(text)); messages = store.Get(key)
  system = BuildSystemBlocks(cfg.MaxSteps)
  emit(agent_start)
  rctx = WithTimeout(ctx, cfg.RequestTimeout)          // 默认 180s(WS 层同源)
  finalText, newMsgs = runLoop(rctx, userID, system, messages)
  store.Append(key, newMsgs…)；若有 finalText 再 append assistant(finalText)
  err ≠ nil → emit(error)；emit(agent_end{text})

runLoop: for step := 0; step < MaxSteps; step++ {
  resp = callLLM(system, messages)                     // sanitizeForWire + 节流 + 流式
  if len(resp.ToolUses()) == 0 { return resp.Text() }   // 纯文本作答
  for tu in ToolUses:
    emit(tool_start); text, isErr = Dispatch(ctx, userID, …)
    text = "[step {step+1}/{N}] " + truncateRunes(text, ToolResultLimit)   // 促收敛
    emit(tool_result)
  messages += [assistant(resp.Content), user(tool_results…)]   // 成对回灌
}
// 步数用尽:finalText 为空补收尾说明「已达到单轮最大步数 N 步…」
// ⇒ 会话恒以 assistant 结束(下一轮 user 不与悬空 tool_result 相邻)
```

**纪律**（CLAUDE.md §14.1 / §197）：
- user/assistant **严格交替**；`tool_result` 与 `tool_use` **成对**回灌（同轮全部
  tool_use 的结果合入下一条 user）。
- `[step i/N]` 前缀先截断后拼接（前缀恒完整）；与 citychrome 同款约定。
- `ChatStreamAccumulate` + `text_delta` 流式下发（§197 字节续命）；
  thinking 开启时注入 `ThinkingConfig`（`GetThinkingEnabled`）。
- 单轮出错**整体不落地**（不持久化半截 tool 对）；工具级失败作为 isErr 结果
  回灌模型，不中断循环。

**出站消息修整 `sanitizeForWire`（`sanitize.go`，callLLM 唯一闸口）**：
llm/anthropic 层不修交替与配对，本包自证合法：
1. 内容块白名单 text/tool_use/tool_result（thinking 等剔除）；
2. 相邻同角色合并（失败轮残留相邻 user 的 400 防线）；
3. 首条必须 user（前置 assistant 丢弃）；
4. tool_result 只在**紧跟**持有其 id 的 assistant 时保留（孤儿/重复剔除）；
   无收尾的 tool_use 剔除（消息清空则整条剔除）；
5. 剔除后再合并相邻同角色。存储层保留原始留痕（前端历史展示不受影响）。

**并发**：不同用户可在同一 Agent 并发 `Run`（WS 层每用户 busy 限流 1，
35060）；`store`/`linePolicy`/`emit` 各自加锁或由调用方保证，身份 userID
显式传递（无共享可变状态，`-race` 单测覆盖）。

---

## 7. Context 管理（`context.go`）

| 项 | 设计 |
|---|---|
| 键 | `roomID + "/" + userID`（每用户每房独立会话） |
| 值 | `[]llmtypes.Message`（**不含 system** —— 每轮重建，场景目录内联恒定） |
| 容量 | `Config.MaxMessages`（缺省 40） |
| 裁剪 | 超限保**前 min(10, max/2) 条**（保底开头意图）+ **最近 max-head 条**；切口不拆散 tool_use/tool_result 对（头不以 tool_use 收尾、尾不以 tool_result 开头） |
| 压缩 | `Compact(ctx, key, callLLM)`：最旧一半 → LLM 压成一条 `user` 摘要（`【前文摘要】…`）插回头部；切口同样守配对。**默认不触发**（省 LLM 开销），接口 + 单测就绪 |
| 重置 | `Agent.Reset(userID)` → `store.Reset(key)`（WS `system_reset` / HTTP reset 共用） |
| 历史 | `store.Get` 返回**副本**；WS `system_history` / `GET …/system_agent/history` 直出 |

---

## 8. 线路策略（`linepolicy.go`，三态）

| 取值 | 语义 | 来源 |
|---|---|---|
| `""` | 沿用 `cfg.ModelKey`（旧行为，零回归） | 建房缺省 |
| `"random"` | 每轮从 `registry.ListEnabled()` 按 `concurrency_lines` **加权随机**取线；池空回退 `cfg.ModelKey` | 建房弹窗默认 |
| 其它 | 固定 model key | 建房指定 |

- 与批次 79 `city_chrome_line` 完全同构；字段 `city_system_line` 独立，互不覆盖。
- 房间级策略 `room.CitySystemLine()` → `Agent.SetLinePolicy`（建房/进房装配时）。
- 并发安全：`lineMu`（建房写 / `resolveLineKey` 读），`-race` 单测覆盖。

---

## 9. 与其它 Agent 的协作（经 `citybus`）

| 工具 | 目标 | 消息 Type | 语义 |
|---|---|---|---|
| `focus_camera` | `LsmAgentGame-City-WebGL-Show`（房间级） | `run_display` | 中性 `displayRequest` → 结构化指令序列 → `show_command` 事件逐条下发（带 `trace_id`） |
| `collect_realworld_data` | `LsmAgentGame-City-Chrome`（类级） | `collect` | 触发一次采集 mission；结果异步入 FeedStore |
| `send_agent_message` | 任意已注册类 | `message` | 单向投递（不做自由双边聊天，防不可控 LLM 循环） |
| `list_agents` | `hub.Agents()` | — | 类级清单（不含房间级实例）；新 Agent 只需 `Register` 即自动可见（§130 反例） |

显示指令 wire 契约（`display.go` 中性结构，字段与方案 §3.3/§6.1 逐字一致）：
`{"type":"focus_district|focus_point|set_mode|highlight_district|annotate|clear",
"district","x","z","mode","text","color","trace_id"}`。

**事件流**（WS `game.virtual_city_system_event`，`kind` 枚举）：
`agent_start` / `text`(增量) / `thinking` / `tool_start` / `tool_result` /
`show_command`（另双发 `game.virtual_city_show_command`）/ `report`(预测报告卡片) /
`agent_end` / `error` / `cleared`。

---

## 10. 错误码（`errcode.go`，⚠ 与方案 §9 的偏差）

| 码 | 常量 | 含义 | 触发层 |
|---|---|---|---|
| 35059 | `ErrVirtualCitySystemAgentDisabled` | 系统 Agent 未启用（`city_system.enabled=false`） | WS 层 |
| 35060 | `ErrVirtualCitySystemAgentBusy` | 该用户上一轮未结束（并发限流 1） | WS 层 |
| 35061 | `ErrVirtualCityAgentNotFound` | 总线目标 Agent 未注册（`send_agent_message` 等） | 派发层 `busFailure` |
| 35055/35056/35057 | 场景 kind / 参数 / 活动数上限 | 引擎注入校验 | 派发层原样回灌 |
| 35058 | `ErrVirtualCityPredictionRunNotFound` | 预测运行不存在（批次 68 占用） | 派发层原样回灌 |
| 35011 | 房主校验 | `control_simulation` pause/resume | 引擎 `room.Pause` |

> 方案 §9 原写 35058/35059/35060，但 35058 已被批次 68 占用 ⇒ 实际顺延
> **35059/35060/35061**（errcode 注释已注明）；前端契约以 errcode 常量为准。

---

## 11. 零回归契约

1. **不消耗 RNG**：工具只调既有确定性房间 API；`ApplyScenario` 复用批次 68
   `applyScenarioToWorld`（不触 `w.Rand`）。
2. **不参与月结**：System Agent 是人类触发旁路；`Pause` 只冻结时钟。
3. **固定 seed 双跑逐位相等**：无对话 ⇒ 不触碰 World 状态 ⇒ `SettleMonth`
   双跑相等；MC / calibration 对局（无 feed）零偏移。
4. **只读桥持锁自证**：`SystemSummaryText` / `WorldEventLogView` 持 `r.mu`
   只读、nil 安全（World 为 nil 返回可读文本），禁回调房间方法（§92a）。

---

## 12. 风险与回退

| 风险 | 缓解 |
|---|---|
| LLM 乱调工具 / 死循环 | `MaxSteps=8` + `RequestTimeout=180s` + `[step i/N]` 促收敛 + 步尽收尾说明 |
| LLM 成本 | 每轮 ≤8 步；会话 ≤40 条；工具结果 ≤4000 rune 截断；场景目录内联省往返 |
| Agent 编造指标 | 提示词禁止 + 指标一律经真实工具；`report` 卡片数据与文本同源 |
| 消息序列 wire 破损（相邻 user / 悬空对） | `sanitizeForWire` 出站闸口 + 裁剪/Compact 配对守卫 + 失败轮不落地 |
| 并发串号 | userID 显式传递（无共享身份字段），`-race` 覆盖 |
| 系统 Agent 影响存量对局 | 只读 + 复用既有注入通道；零 RNG；无事件零偏移 |
| 一键回退 | `[city_system] enabled=false` ⇒ WS 回 35059，前端隐藏「系统对话」按钮 |

---

## 13. 测试覆盖（`ServerGo/agent/citysystem/`）

| 域 | 用例 |
|---|---|
| 11 工具派发 | 每工具 ≥1 正 1 负（`dispatch_test.go`）；nil room 边界；错误码透传 `[35055]/[35056]/[35058]/[35011]/[35061]` |
| ReAct 主循环 | 纯文本 / 工具调用 / 流式 delta / 步数用尽（收尾 assistant）/ LLM 失败（不落地）/ 节流槽 / thinking 注入 / 失败轮后重试 |
| wire 修整 | 相邻同角色合并 / 孤儿 tool_result / 悬空 tool_use / 紧邻配对 / 多工具并行 / thinking 剔除 / 不改输入（`sanitize_test.go`） |
| 会话 | 裁剪（前 10 + 尾部）、副本语义、Reset、Compact（含配对守卫）、并发 Append/Get |
| 线路策略 | 缺省 / 固定 / 随机加权 / 池空回退 / 并发读写 |
| 提示词/工具 schema | 三段头幂等、步数纪律、目录 40 全覆盖（enum ≡ catalog）、数值上下界 |
| 并发 | 多用户并发 Run（`-race`） |
