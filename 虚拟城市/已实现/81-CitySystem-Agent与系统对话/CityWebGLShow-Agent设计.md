# 批次 81 · City-WebGL-Show Agent 设计（`LsmAgentGame-City-WebGL-Show`）

> 日期：2026-11-03 ｜ 分支：`main`（§10.1）｜ 职责线：`city-webgl-show`（§13.1 城市显示）
> 上游方案：[`01-方案设计.md`](01-方案设计.md) §3.1（总线）、§3.3（本包）、§6.1（WS 契约）
> 中性 wire 契约镜像：`ServerGo/agent/citysystem/display.go`（与本包逐字一致，互不 import）
>
> 本文是 City-WebGL-Show 的**单一设计说明**；实现事实以 `ServerGo/agent/citywebgl/` 代码为准。

---

## 1. 定位与职责边界

City-WebGL-Show 是虚拟城市的**渲染显示 Agent**：把「看哪里 / 怎么显示」
翻译成**结构化显示指令**（`DisplayCommand`），经 WS 推给前端 3D 引擎执行。

**只翻译、不渲染**：

- 服务端不做任何图形工作（不建场景、不动相机、不画标注）——指令由
  前端 `cityShowAdapter` 消费；
- 不做 LLM 决策之外的图形工作：LLM 只负责「自然语言 → 显示原语」的翻译，
  参数齐备时甚至不过 LLM（结构化路径）；
- 不改城市引擎状态、不参与月结 RNG、不采集数据。

**调用方**：City-System 主 Agent 的 `focus_camera` 工具（经 `citybus` 房间级
寻址 `LsmAgentGame-City-WebGL-Show`）；未来人类/其它 Agent 的显示请求同路。

---

## 2. AgentClassName 与登记

| 项 | 值 |
|---|---|
| AgentClassName | `LsmAgentGame-City-WebGL-Show` |
| 常量 | `ServerGo/agent/class_names.go::AgentClassCityWebGLShow`（已登记 `AllAgentClassNames()`） |
| 实现 | `ServerGo/agent/citywebgl/agent.go::Agent`（每房间一个） |
| User-Agent | `LsmAgentGame-City-WebGL-Show/<AppVersion> <buildDateTime>`（§24） |
| 提示词三段头 | `sysprompt.EnsureHead(..., AgentClassCityWebGLShow)`（§14.3，自动注入） |
| 总线注册 | **房间级**（每房一个，`citybus.RegisterRoom` / `UnregisterRoom`） |

`citysystem/display.go` 内的字面量镜像常量 `agentClassCityWebGLShow`
仅作 citybus 寻址 to 值（避免 citysystem → citywebgl 编译期依赖）；
正式登记以 `class_names.go` 为准。

---

## 3. 两种调用路径

| 路径 | 入口 | 是否过 LLM | 产物 | 时延取向 |
|---|---|---|---|---|
| **结构化** | `Agent.Structured(req)` | **不过 LLM** | 1-2 条指令 | 视角跟随要快（零模型往返） |
| **LLM** | `Agent.Run(ctx, req)` | ReAct ≤ `MaxSteps`（缺省 3） | 指令序列（逐条 `emit` 实时下发） | 自然语言翻译 |

**路径选择**（`ws/game_service_virtual_city_system.go` 房间级 handler）：

```
req.District != "" || req.X != nil || req.Z != nil  ⇒  Structured
否则（纯 text 自然语言）                          ⇒  Run
```

**Structured 组合规则**：district 优先于 `(x,z)`（`focus_camera` 契约二选一）；
`mode` 合法（orbit|fly）时追加一条 `set_mode`；全部参数无效 → 空切片。
不 `emit` —— 返回值由调用方 `citysystem.focusCamera` 统一带 `trace_id` 下发。

**Run**：每条指令都经 `emit` 实时下发（WS show_command 发帧路径），
返回值是本次全部指令（出错时已产出指令随切片返回）。

---

## 4. 6 个显示原语（工具 → 指令 → 前端动作）

| 工具名 | 参数（必填 / 可选） | 产出 `DisplayCommand.type` | 前端动作（`cityShowAdapter`） |
|---|---|---|---|
| `focus_district` | `district` / — | `focus_district` | 相机飞向/环绕该城区锚点 |
| `focus_point` | `x`,`z`（同给） / — | `focus_point` | 相机聚焦世界坐标 (x,z) |
| `set_mode` | `mode`（orbit\|fly） / — | `set_mode` | 切换相机运动风格（环绕/飞行） |
| `highlight_district` | `district` / `color` | `highlight_district` | 城区高亮描边/染色（缺省主题色） |
| `annotate` | `text`（≤40 字） / `district` | `annotate` | 叠简短标注（district 空=全局） |
| `clear_display` | — / — | `clear` | 清除高亮/标注（不动视角） |

注意：工具名 `clear_display` ≠ 指令类型 `clear`（契约常量 `DisplayClear`），
映射只此一处不对称，前端以指令 `type` 为准。

**派发纪律**（`tools.go::dispatchTool`，纯函数）：
- 缺参/非法参 → `tool_result{is_error:true}` 回灌模型自纠，**不 emit 不收集**；
- `mode` 只接受 orbit|fly（`NormalizeMode`，大小写不敏感）；
- `annotate` 文本超 40 字自动截断（纪律口径与 prompt 一致）；
- `district` 只接受请求方给的 id —— prompt 级禁止编造地名，派发层不去重校验
  城区是否存在（前端消费端兜底）。

---

## 5. Wire 契约（与 `citysystem/display.go` 逐字一致）

```jsonc
// DisplayRequest（bus Message.Payload，citysystem.displayRequest 镜像）
{"text":"...","district":"d1","mode":"orbit|fly","x":0.0,"z":0.0}

// DisplayCommand（返回值 []DisplayCommand + WS show_command 的 command 段）
{"type":"focus_district|focus_point|set_mode|highlight_district|annotate|clear",
 "district":"d1","x":0.0,"z":0.0,"mode":"orbit|fly",
 "text":"...","color":"...","trace_id":"tr-..."}
```

- **双侧镜像**：`citywebgl/command.go` 与 `citysystem/display.go` 各持一份
  结构（互不 import，防包耦合）；字段名任何一侧改动必须双侧同步，
  `command_test.go` 用字面 JSON 锚点 + 逐字镜像结构钉死。
- **跨包传输**统一 JSON 往返（`ParseDisplayRequest` / `parseDisplayCommands`），
  兼容 struct / map / `[]byte` 三种载荷形态。
- **WS 帧**（契约 §3.5/§6.1）：`game.virtual_city_show_command`
  `{room_id, command}`，另随 `game.virtual_city_system_event`（kind=`show_command`）双帧下发；
  `trace_id` 串联事件流与显示指令（Run 路径由本包生成 `tr-<unixNano>`，
  与 `citysystem.NewTraceID` 同格式）。

---

## 6. ReAct 循环与 LLM 调用纪律

```
DisplayRequest{Text, District?, Mode?, X?, Z?}
  │
  ├─ system = sysprompt.EnsureHead([{roleText}], AgentClassCityWebGLShow)  ← §14.3
  ├─ user   = userPrompt(req)  = 「显示请求」文本 + 已带结构化参数提示行
  │
  └─ for step := 0; step < MaxSteps(缺省 3); step++ {
         resp := callLLM(...)              ← ChatStreamAccumulate 流式续命(§197)
         if len(resp.ToolUses()) == 0 { break }     ← 纯文本作答 = 翻译结束
         for _, tu := range resp.ToolUses() {
             cmd, text, isErr := dispatchTool(tu.Name, tu.Input, trace)
             if !isErr { emit(cmd); cmds = append(cmds, *cmd) }
             results = append(results, tool_result{tu.ID, text, isErr})
         }
         messages += assistant(resp.Content) + user(results)   ← §14.1 严格交替
     }
     return cmds
```

**LLM 调用纪律**（对齐 CLAUDE.md §14/§14.1/§197/§24）：

1. `AgentClassName = AgentClassCityWebGLShow` 必填（UA 拼装 + 计费区分）；
2. `ChatStreamAccumulate` 优先（§197 流式续命），无流式能力回退 `Chat`；
3. `thinking` 按 `registry.GetThinkingEnabled(modelKey)` 注入；
4. `tool_use` 派发后**成对**回灌 `tool_result`（user/assistant 严格交替，禁连续 user）；
5. system 经 `EnsureHead` 幂等注入三段式头（单一注入点在 provider 层，双保险）；
6. `MaxTokens=2048`（显示指令体量小）；`MaxSteps` 缺省 3 —— 指令精准克制，
   一次聚焦 ≤3 条（prompt 纪律 + 步数上限双重收敛）。

**角色段要点**（`prompt.go::roleText`）：只产出结构化显示指令、不做图形渲染；
mode 只能 orbit|fly；annotate ≤40 字；不编造地名（district 只来自请求参数）；
完成即停不闲聊。

**线路策略**（批次79 同构）：`SetLinePolicy(""|"random"|固定 key)` 建房链路
按 `room.CitySystemLine()` 透传；`resolveLineKey` 每次 Run 解析一次。

---

## 7. 总线接入（房间级注册/注销）

| 生命周期点 | 位置 | 动作 |
|---|---|---|
| 注册 | `ws/game_service_virtual_city_system.go::getOrCreateSystemRoomAgent` | `citybus.Default().RegisterRoom(roomID, AgentClassCityWebGLShow, handler)`（先 `UnregisterRoom` 强制重绑，防同房重建/测试复用 roomID 时旧 handler 残留） |
| 注销 | 同文件 `removeCitySystemAgent`（房间销毁路径） | `citybus.Default().UnregisterRoom(...)` |

handler 行为：

1. `citywebgl.ParseDisplayRequest(msg.Payload)` 解码（citysystem 发出的
   `displayRequest` 镜像 struct / map 均可）；
2. 路径选择（§3）：结构化参数齐备 ⇒ `Structured`；否则 `Run`（自然语言）；
3. 返回 `[]DisplayCommand` —— citysystem 侧 `parseDisplayCommands` JSON 往返解析。

`emit` 复用 `emitCitySystemEvent` 的 show_command 发帧路径（`kind=show_command`
→ `game.virtual_city_show_command` 双帧）。registry 复用 `s.vcSystemRegistry`
（`citysystem.LLMRegistry` 与 `citywebgl.LLMRegistry` 方法集兼容，直接传入）。

**双发语义**：Run 路径逐条 `emit`（实时）+ citysystem `focusCamera` 对返回值
再 `emitCommands`（补 trace_id）——同 `trace_id` 的重复帧前端按幂等消费
（focus/highlight 重放无副作用）；Structured 路径只发一次（调用方下发）。

---

## 8. 扩展点（新增显示原语）

1. `tools.go`：工具名常量 + `BuildTools()` 一条 `ToolDef` + `ToolNames()` 一行 +
   `dispatchTool` 一个 case（产出新 `DisplayCommand.Type`）；
2. `command.go`：新指令类型常量（若需新字段，**双侧同步** `citysystem/display.go`，
   并更新 `command_test.go` 字面 JSON 锚点）；
3. 前端 `ClientWeb/src/components/virtualCity/cityShowAdapter.ts`：消费新 `type`
   （前端职责线，独立 SubAgent）；
4. 测试：`tools_test.go` 各补 1 正 1 负（§130）。

角色段 `roleText` 如需提及新原语，保持「精准克制」总纪律不变。

---

## 9. 测试锚点

| 文件 | 覆盖 |
|---|---|
| `command_test.go` | wire 形状与 citysystem 镜像逐字一致（全量/稀疏 JSON + 字面键集锚点）；`ParseDisplayRequest` 三形态；路径判据 |
| `tools_test.go` | 6 工具 schema（名实一致 + required 钉死）；9 正例字段精确匹配；11 负例 isErr；annotate 截断；nil input |
| `agent_test.go` | `Structured` 9 组合；`Run` emit/返回/交替回灌/AgentClassName/三段头；派发失败不 emit；空文本/nil registry/LLM 出错部分产出；MaxSteps 边界；线路策略 |

验证命令：`cd ServerGo && go test ./agent/citywebgl/... ./agent/... ./ws/... -count=1`。
