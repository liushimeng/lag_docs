# 批次 84 · City-WebGL-Show Agent 设计（`LsmAgentGame-City-WebGL-Show`）

> 日期：2026-11-05 ｜ 分支：`main`（§10.1）｜ 职责线：`backend-dev`（Agent 线 · §13.1 职责线 8 / §29.1）
> 上游方案：[`01-方案设计.md`](01-方案设计.md) §2.6（现状诊断 F）、§4.5（显示原语 6 → 14）、§7.1（WS 契约）
> 中性 wire 契约镜像：`ServerGo/agent/citysystem/display.go`（与本包逐字一致，互不 import）
> 上一版：[`../81-CitySystem-Agent与系统对话/CityWebGLShow-Agent设计.md`](../81-CitySystem-Agent与系统对话/CityWebGLShow-Agent设计.md)
>
> 本文是 City-WebGL-Show 的**单一设计说明**；实现事实以 `ServerGo/agent/citywebgl/` 代码为准。

---

## 0. 本批变更摘要

| 项 | 批次 81 | 批次 84 |
|---|---|---|
| 显示原语（工具） | 6 | **14** |
| 显示指令类型（`DisplayCommand.type`） | 6 | **14** |
| `DisplayCommand` 字段 | 8 | **19**（新增 11 个） |
| `DisplayRequest` 字段 | 5 | **16**（新增 11 个） |
| Structured 快路径上限 | 2 条 | **5 条** |
| 提示词纪律 | 6 原语 | 14 原语 + 7 组枚举 |

**零回归承诺**：批次 81 的 6 个原语**产出逐字不变**（同 `type`、同字段、同值），
既有单测全部保留并继续通过；新增能力全部是**加字段、加工具、加分支**，不改既有语义。

---

## 1. 定位与职责边界

City-WebGL-Show 是虚拟城市的**渲染显示 Agent**：把「看哪里 / 怎么显示」
翻译成**结构化显示指令**（`DisplayCommand`），经 WS 推给前端 3D 引擎执行。

**只翻译、不渲染**：

- 服务端不做任何图形工作（不建场景、不动相机、不画标注）——指令由
  前端 `cityShowAdapter` 消费；
- 不做 LLM 决策之外的图形工作：LLM 只负责「自然语言 → 显示原语」的翻译，
  参数齐备时甚至不过 LLM（结构化路径）；
- 不改城市引擎状态、不参与月结 RNG、不采集数据；
- 不查实体是否存在：`entity_id` / `district` 只做**非空 + 格式**校验，
  真实性由 prompt 纪律 + 前端消费端兜底。

**调用方**：City-System 主 Agent 的 `focus_camera` 工具（经 `citybus` 房间级
寻址 `LsmAgentGame-City-WebGL-Show`）；未来人类/其它 Agent 的显示请求同路。

**本包不碰的东西**（其它职责线负责，本包只依赖其契约）：

| 事项 | 归属 |
|---|---|
| `citysystem/display.go` 的镜像结构 | 职责线 2（City-System Agent 线） |
| `citybus` 扩展 / 双向通讯 | 职责线 2 |
| 前端 `cityShowAdapter.ts` 14 type 实现 | 前端职责线 |
| `errcode` 35062–35065 | 后端职责线 1（`errcode` 包） |

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

---

## 3. 两条调用路径

| 路径 | 入口 | 是否过 LLM | 产物 | 时延取向 |
|---|---|---|---|---|
| **结构化** | `Agent.Structured(req)` | **不过 LLM** | 1–5 条指令 | 视角跟随要快（零模型往返） |
| **LLM** | `Agent.Run(ctx, req)` | ReAct ≤ `MaxSteps`（缺省 3） | 指令序列（逐条 `emit` 实时下发） | 自然语言翻译 |

### 3.1 Structured 快路径（零延迟）

实现：`structured.go::buildStructuredCommands`。

**优先级顺序**（超出上限时按此顺序保留靠前的）：

```
1. focus        district 优先于 (x,z)          → focus_district | focus_point
2. 相机运动风格  mode ∈ orbit|fly              → set_mode
3. 环境         phase(+hour) / weather / speed → set_time_of_day / set_weather / set_time_speed
4. 实体/图层/筛选 entity_id(+kind) / layer+visible / category(+mode)
                                               → focus_entity / set_layer_visibility / set_filter
5. 预设/路径     preset / points               → camera_preset / fly_path
```

**硬上限**：`maxStructuredCommands = 5`（防指令风暴）。批次 81 时代上限为 2，
批次 84 放宽到 5 —— 与「一次显示类请求 ≤5 条指令」的提示词纪律同口径。

**与 LLM 路径的语义差异**（**仅在非法取值上**）：

| 场景 | LLM 路径（`dispatchTool`） | Structured 路径 |
|---|---|---|
| 缺参 / 非法枚举 / 越界数值 | `tool_result{is_error:true}` 回灌模型自纠，**不产指令** | **静默忽略该参数**，不产该条指令 |
| 合法参数 | 产对应指令 | 产**语义相同**的指令 |

理由：Structured 路径**没有错误回传通道**（返回值就是指令切片），
宁可少一条指令，也不可产出一条非法指令。

**`mode` 字段的双语义**（上下文相关，由 `type` 区分，**不加嵌套**）：

- 指令 `type = set_mode` ⇒ `mode ∈ orbit|fly`（相机运动风格）；
- 指令 `type = set_filter` ⇒ `mode ∈ highlight|dim`（筛选呈现方式）；
- 请求级 `mode` 命中 `highlight|dim` 时，Structured 路径理解为**筛选模式**
  （不产 `set_mode`）；命中 `orbit|fly` 时理解为**相机风格**。

### 3.2 Run（LLM 路径）

```
DisplayRequest{Text, District?, Mode?, X?, Z?, +批次84 的 11 个新字段}
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

`MaxSteps` 缺省仍是 **3**（零回归）。`userPrompt` 的 `structuredHint`
把请求里已有的 **14 个** 结构化参数逐条列出，让模型**优先直译**
而不是从自然语言里猜（零歧义）。

---

## 4. 14 个显示原语（工具 → 指令 → 校验 → 前端消费方）

> 工具名常量集中在 `tools.go`；批次 84 新增 8 个的 schema + 派发在
> `tools_batch84.go`。指令类型常量集中在 `command.go`。
> ⚠ 工具名 `clear_display` ≠ 指令类型 `clear`（`DisplayClear`）—— **唯一**一处不对称。

| # | 工具名 | 参数（必填 / 可选） | 产出 `type` | 校验规则 | 前端应做什么（`cityShowAdapter`） |
|---|--------|----------------------|-------------|----------|-------------------------------|
| 1 | `focus_district` | `district` / — | `focus_district` | 非空（去空格后） | 相机飞向 / 环绕该城区锚点 |
| 2 | `focus_point` | `x`,`z`（同给） / — | `focus_point` | 两值均为数值，缺一即失败 | 相机聚焦世界坐标 (x,z) |
| 3 | `set_mode` | `mode` / — | `set_mode` | 只能 `orbit\|fly`（大小写不敏感） | 切换相机运动风格 |
| 4 | `highlight_district` | `district` / `color` | `highlight_district` | `district` 非空；`color` 可空 | 城区高亮描边 / 染色（缺省主题色） |
| 5 | `annotate` | `text` / `district` | `annotate` | `text` 非空；**超 40 字自动截断** | 叠简短标注（`district` 空 = 全局） |
| 6 | `clear_display` | — / — | `clear` | 无参 | 清除高亮 / 标注（不动视角） |
| 7 | `set_time_of_day` | `phase` / `hour` | `set_time_of_day` | `phase ∈ dawn\|day\|dusk\|night`；`hour` 可选，须为 **0–24 有限数**（越界即失败） | `engine3d/DayNightSky` 覆盖时段（+可选精确小时） |
| 8 | `set_weather` | `weather` / — | `set_weather` | `weather ∈ clear\|rain\|snow\|fog\|haze\|sandstorm\|storm` | `engine3d/WeatherFX` 覆盖天气 |
| 9 | `set_time_speed` | `speed` / — | `set_time_speed` | `speed ∈ pause\|slow\|normal\|fast\|xfast` | `freeViewStore` 的 timeRatio 通道 |
| 10 | `focus_entity` | `entity_id` / `kind` | `focus_entity` | `entity_id` 非空；`kind` 可选，须 ∈ `building\|resident\|vehicle`（给了非法值即失败） | 对象注册表（批次 28 `objectInfo/`）+ `focusRef` |
| 11 | `set_layer_visibility` | `layer`,`visible` / — | `set_layer_visibility` | `layer ∈ buildings\|trees\|vehicles\|pedestrians\|props\|labels\|water`；`visible` **必须是布尔**（缺失或非布尔即失败） | 场景层图层注册表切 `visible` |
| 12 | `set_filter` | — / `category`,`mode` | `set_filter` | `category` 可空（**空 = 取消筛选**，仍产指令）；`mode` 可选，默认 `highlight`，须 ∈ `highlight\|dim` | 按类别筛选高亮 / 变暗 |
| 13 | `camera_preset` | `preset` / — | `camera_preset` | `preset ∈ bird\|eye\|street\|top\|free` | `FreeViewControls` 的 mode + 目标点映射 |
| 14 | `fly_path` | `points` / — | `fly_path` | `points` 为 **2–8** 个 `{x,z}`，每点 x/z 须为**有限数** | 相机沿折线插值掠过 |

**派发纪律**（`dispatchTool` / `dispatchToolBatch84`，纯函数）：

- 缺参 / 非法枚举 / 越界数值 ⇒ `tool_result{is_error:true}` 回灌模型自纠，
  **不 emit 不收集**（`cmd == nil`）；
- 枚举取值统一走 `NormalizeEnum`（小写 + 去首尾空格后命中白名单）；
- **可选参数给空串 / nil 视作「未给」**（走缺省值），不判失败
  （`argPresent`：键存在 + 非 nil + 非空串）；
- `visible` 例外：**必须是真正的布尔**（字符串 `"true"` / `"1"` 一律判非法），
  因为「显示 / 隐藏」语义不能靠猜；
- `annotate` 文本超 40 字自动截断（纪律口径与 prompt 一致）；
- 成功统一 `mk(cmd)` ⇒ 回执「已下发显示指令 `<type>`」+ 补 `trace_id`；
- `district` / `entity_id` **只做非空校验**，不校验实体是否存在
  （prompt 级禁止编造，前端消费端兜底）。

---

## 5. Wire 契约（与 `citysystem/display.go` 逐字一致）

### 5.1 `DisplayCommand`（WS `game.virtual_city_show_command` 的 `command` 段）

```jsonc
{
  "type": "focus_district|focus_point|set_mode|highlight_district|annotate|clear"
          "|set_time_of_day|set_weather|set_time_speed|focus_entity"
          "|set_layer_visibility|set_filter|camera_preset|fly_path",

  // ── 批次81 字段(顺序不变)──
  "district": "d1",
  "x": 0.0,
  "z": 0.0,
  "mode": "orbit|fly",          // set_mode 时为运动风格;set_filter 时为 highlight|dim
  "text": "...",
  "color": "...",
  "trace_id": "tr-...",

  // ── 批次84 新增字段(统一追加在既有字段之后,顺序见下表)──
  "phase": "dawn|day|dusk|night",
  "hour": 12,
  "weather": "clear|rain|snow|fog|haze|sandstorm|storm",
  "speed": "pause|slow|normal|fast|xfast",
  "entity_id": "...",
  "kind": "building|resident|vehicle",
  "layer": "buildings|trees|vehicles|pedestrians|props|labels|water",
  "visible": true,
  "category": "...",
  "preset": "bird|eye|street|top|free",
  "points": [{"x": 0, "z": 0}, {"x": 1, "z": 1}]
}
```

### 5.2 字段表（`DisplayCommand`）

| # | Go 字段 | JSON 键 | 类型 | omitempty | 适用 type |
|---|---|---|---|---|---|
| 1 | `Type` | `type` | string | 否（恒有） | 全部 |
| 2 | `District` | `district` | string | 是 | focus_district / highlight_district / annotate |
| 3 | `X` | `x` | float64 | 是 | focus_point |
| 4 | `Z` | `z` | float64 | 是 | focus_point |
| 5 | `Mode` | `mode` | string | 是 | set_mode（orbit\|fly）/ set_filter（highlight\|dim） |
| 6 | `Text` | `text` | string | 是 | annotate |
| 7 | `Color` | `color` | string | 是 | highlight_district |
| 8 | `TraceID` | `trace_id` | string | 是 | 全部（串联事件流） |
| 9 | `Phase` | `phase` | string | 是 | set_time_of_day |
| 10 | `Hour` | `hour` | **\*float64** | 是 | set_time_of_day（指针 ⇒ `0` 与「未设」可区分） |
| 11 | `Weather` | `weather` | string | 是 | set_weather |
| 12 | `Speed` | `speed` | string | 是 | set_time_speed |
| 13 | `EntityID` | `entity_id` | string | 是 | focus_entity |
| 14 | `Kind` | `kind` | string | 是 | focus_entity |
| 15 | `Layer` | `layer` | string | 是 | set_layer_visibility |
| 16 | `Visible` | `visible` | **\*bool** | 是 | set_layer_visibility（指针 ⇒ `false` 与「未设」可区分） |
| 17 | `Category` | `category` | string | 是 | set_filter（空 = 取消筛选） |
| 18 | `Preset` | `preset` | string | 是 | camera_preset |
| 19 | `Points` | `points` | **[]DisplayPoint** | 是 | fly_path（`{x,z}` 数组） |

`DisplayPoint`：`{ "x": float64, "z": float64 }`（无 omitempty —— 0 坐标合法）。

### 5.3 `DisplayRequest`（bus `Message.Payload`）

字段与 `DisplayCommand` 的新增部分**同名同 tag 同顺序**：
`text` / `district` / `mode` / `x` / `z` + `phase` / `hour` / `weather` /
`speed` / `entity_id` / `kind` / `layer` / `visible` / `category` / `preset` /
`points`。

### 5.4 双侧同步铁律

- `citywebgl/command.go` 与 `citysystem/display.go` 各持一份镜像结构
  （互不 import，防包耦合），**字段名、JSON tag、类型、声明顺序**必须逐字一致；
- Go 的 `encoding/json` 按**声明顺序**序列化 ⇒ 顺序不同则 JSON 字节不同；
  两侧的锚点测试（`command_test.go` / citysystem 侧 `display_test.go`）
  用**字面 JSON** 双向钉死；
- 批次 84 新增字段**统一追加在既有字段之后**（`phase → hour → weather →
  speed → entity_id → kind → layer → visible → category → preset → points`），
  两侧照此顺序追加；
- 跨包传输统一 JSON 往返（`ParseDisplayRequest`），兼容 struct / map / `[]byte`
  三种载荷形态。

### 5.5 向后兼容

**只加字段、不加嵌套** ⇒ 旧前端 adapter 遇到未知 `type` 会静默忽略
（`cityShowAdapter.ts` 既有 default 分支），既有 6 个 type 的 JSON 形态
**逐字不变**。

---

## 6. 系统提示词纪律（`prompt.go::roleText`）

角色段经 `sysprompt.EnsureHead(..., AgentClassCityWebGLShow)` 注入三段式头
（§14.3，**单一注入点**，本包不自行拼接任何计费 / 身份头）。

本批新增的纪律条目：

1. **14 个原语清单 + 7 组取值枚举**（枚举值与 `command.go` 的
   `DisplayPhases` / `DisplayWeathers` / `DisplaySpeeds` / `DisplayEntityKinds`
   / `DisplayLayers` / `DisplayFilterModes` / `DisplayCameraPresets` 逐字对应）；
2. **条数纪律**：一次聚焦类请求 ≤3 条指令；一次显示类请求（环境 / 图层 /
   路径等）≤5 条指令；
3. **不编造实体与图层**：`entity_id` / `district` 只能来自请求参数或上下文中
   出现过的真实 id；`layer` 只能用枚举里的图层名；绝不用占位符或「示例 id」充数；
4. **布尔与空值语义**：`visible` 必须是 true/false；`set_filter` 的 `category`
   留空表示取消筛选。

既有纪律（批次 81，零回归）：只产出结构化显示指令、不做图形渲染；
`mode` 只能 orbit|fly；`annotate` ≤40 字、不写链接/Markdown；不编造地名；
清除用 `clear_display`；完成即停不闲聊。

`structuredHint` 扩展到 14 行提示（district / point / mode / phase / hour /
weather / speed / entity_id / kind / layer / visible / category / preset /
points），让模型**优先直译**而不是从自然语言里猜。

---

## 7. 与前端 `cityShowAdapter` 的对接（前端由独立职责线实现）

| 指令 type | 前端应做什么 | 既有引擎落点 |
|---|---|---|
| `focus_district` | 相机飞向 / 环绕该城区锚点 | `focusRef` / `freeViewApi` |
| `focus_point` | 相机聚焦世界坐标 | `focusRef` |
| `set_mode` | 切相机运动风格 | `freeViewStore` |
| `highlight_district` | 城区高亮描边 / 染色 | `cityShowStore` 高亮区 |
| `annotate` | 叠一条短标注 | `cityShowStore` 标注文本 |
| `clear` | 清高亮 + 标注（不动视角） | `cityShowStore` |
| `set_time_of_day` | 覆盖昼夜时段（可选精确小时） | `engine3d/DayNightSky` |
| `set_weather` | 覆盖天气 | `engine3d/WeatherFX` |
| `set_time_speed` | 时间流速（pause…xfast） | `engine3d/controls/freeViewStore` 的 timeRatio |
| `focus_entity` | 聚焦建筑 / 居民 / 车辆 | `components/virtualCity/` 对象注册表 + `focusRef` |
| `set_layer_visibility` | 切图层 `visible` | 各类 instanced mesh（需场景层汇总图层注册表） |
| `set_filter` | 按类别筛选高亮 / 变暗 | 场景层筛选通道（**本批可能缺现成入口**） |
| `camera_preset` | 机位预设 | `engine3d/controls/FreeViewControls.tsx`（mode + 目标点映射） |
| `fly_path` | 沿折线插值飞行 | `FreeViewControls` 目标插值（**需新增「路径插值」模式**） |

> 若某能力在既有引擎中**没有现成入口**，方案文档 §5.2 允许先实现为
> 「接受并记账」（adapter 记录最后一条指令并在 UI 上提示「该能力本批未接引擎」），
> 但必须逐条列出，**不得静默吞掉**。

---

## 8. 测试锚点

| 文件 | 覆盖 |
|---|---|
| `command_test.go` | 批次 81 wire 形状（零回归）+ 镜像锚点结构**扩到 19 字段**；`ParseDisplayRequest` 三形态；路径判据；`NormalizeMode` |
| `command_batch84_test.go` | 批次 84 的 11 个 JSON 键名**逐字**锚点；全量 / 稀疏两种形态；镜像字节一致；`ParseDisplayRequest` 三形态解出全部新字段；`NormalizeEnum`；`HasStructuredParams` 扩展；`NormalizeMode` 零回归 |
| `tools_test.go` | 批次 81 的 6 工具 schema + 正例字段精确匹配 + 负例 isErr + annotate 截断 + nil input（**因 `DisplayCommand` 含切片字段，`==` 比较改为 `reflect.DeepEqual`**） |
| `tools_batch84_test.go` | `ToolNames()` **恰 14**；`BuildTools()` 14 且与 ToolNames 一致；8 个新工具的 `required` / `enum` / `type` 钉死；`fly_path` 的 minItems/maxItems；`visible` 为 boolean；正例字段精确匹配；负例 isErr 且 `cmd == nil` |
| `structured_batch84_test.go` | 11 个新参数各自的产出；组合顺序；条数上限 5；批次 81 组合零回归 |
| `agent_test.go` | `Structured` 组合；`Run` emit/返回/交替回灌/AgentClassName/三段头；派发失败不 emit；空文本 / nil registry / LLM 出错部分产出；MaxSteps 边界；线路策略 |

验证命令：

```bash
cd ServerGo && go build -o LsmAgentGame main.go && go test ./agent/citywebgl/... -count=1
```

---

## 9. 已知边界与待接线

| # | 事项 | 归属 / 说明 |
|---|---|---|
| 1 | **ws 层路径选择未覆盖批次 84 新参数** | `ws/game_service_virtual_city_system.go` 当前用内联判据 `req.District != "" \|\| req.X != nil \|\| req.Z != nil` 选 Structured / Run。**仅带 `phase`/`weather`/`entity_id` 等新参数的请求仍会走 LLM 路径**（功能正确，只是失去零延迟）。建议改为 `req.HasStructuredParams()`（本包已扩展该方法）。`ws/` 不在本职责线工作面，未修改。 |
| 2 | `citysystem/display.go` 镜像结构尚未同步 | 职责线 2 负责；字段名 / tag / 类型 / 顺序须与 §5.2 逐字一致，否则跨包 JSON 往返**静默丢字段**（`omitempty` + 不匹配 ⇒ 字段消失，无报错）。 |
| 3 | `set_filter` / `fly_path` 的引擎入口可能缺失 | 前端职责线负责「接受并记账」的兜底提示（§7）。 |
| 4 | 错误码 35065 `ErrVirtualCityDisplayCommandInvalid` 未在本包使用 | 本包的非法参数走 `tool_result{is_error:true}`（模型自纠通道），不产生 WS 错误码；35065 供 citysystem 侧对外报错。 |
| 5 | 实体 / 城区真实性不校验 | 本包只做非空 + 枚举校验；`entity_id` / `district` 是否真实存在由 prompt 纪律 + 前端消费端兜底（批次 81 既有口径，零回归）。 |
| 6 | `mode` 字段双语义 | 由 `type` 区分（§3.1），**不加嵌套**是 §4.5.2 的硬约束；新增前端 type 时不得再引入第二个 mode 字段。 |
| 7 | `hour` / `visible` 用指针 | 指针是「0 / false 与未设可区分」的唯一手段；跨包 JSON 往返保持指针语义（`null` ⇔ 缺失）。 |
| 8 | `set_filter` 取消筛选依赖「category 缺失/为空」 | `Category` 用 `omitempty`（与 §4.5.2 结构逐字一致）⇒ 空 category 在 wire 上表现为**键缺失**。前端 `cityShowAdapter` 已按「空 = 取消筛选」处理（`layerApi.setFilter('', 'highlight')`），两侧语义一致。 |

---

## 10. 相关文档

- [`01-方案设计.md`](01-方案设计.md) —— 本批上游方案（§2.6 / §4.5 / §7.1）
- [`02-全维度事件体系对账表.md`](02-全维度事件体系对账表.md) —— 164 条用户清单对账
- [`../81-CitySystem-Agent与系统对话/CityWebGLShow-Agent设计.md`](../81-CitySystem-Agent与系统对话/CityWebGLShow-Agent设计.md) —— 上一版（6 原语）
