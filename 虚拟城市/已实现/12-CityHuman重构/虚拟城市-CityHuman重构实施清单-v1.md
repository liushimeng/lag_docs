# 虚拟城市 — City-Human 重构实施清单 v1

> **版本**：v1 ｜ **日期**：2026-09-22 ｜ **状态**：实施执行依据
> **设计契约**：[`虚拟城市-CityHuman-Agent合并与感知系统设计-v1.md`](虚拟城市-CityHuman-Agent合并与感知系统设计-v1.md)（文档 1）、[`虚拟城市-界面文案与建房流程重构设计-v1.md`](虚拟城市-界面文案与建房流程重构设计-v1.md)（文档 2）
> 本清单按文件列出改动点与验收标准；执行顺序 = 后端契约 → 后端实现 → 前端 → 联调验收。

---

## 1. 后端改动清单（`ServerGo/`）

### 1.1 AgentClass 合并

| 文件 | 改动 | 验收 |
|---|---|---|
| `agent/class_names.go` | 新增 `AgentClassCityHuman = "LsmAgentGame-City-Human"`；删除 CityPlayer/CityVoice/CityGovernment/CityBanker/CityFirm 五常量；`AllAgentClassNames()` 同步 | `go build` 通过；旧常量 grep 零命中 |
| `agent/class_names_test.go` | 断言 CityHuman 值；删除旧五常量断言 | `go test ./agent/` 通过 |
| `agent/wealthplayer/agent.go`、`run.go`、`run_llm.go` | `LLMRequest.AgentClassName` 填 `AgentClassCityHuman` | 单测断言 `!= ""` 且 == CityHuman |
| `game/wealth/city/voice.go` | 发声请求的 AgentClassName 改为 `AgentClassCityHuman`（注释同步） | `city_test.go` 通过 |
| `agent/wealthtypes/context.go` | `GameContext.AgentClass` 注释/默认值改 City-Human | grep `City-Player` 零命中 |

### 1.2 感知与行动工具（核心新增）

| 文件 | 改动 | 验收 |
|---|---|---|
| `agent/wealthplayer/tools_sense.go`（**新建**） | 5 个工具的 Anthropic wire 定义 + 派发：`see`/`hear`/`smell`/`move`/`speak(private)`；每月限次计数 | 新单测 `tools_sense_test.go` 全绿 |
| `agent/wealthplayer/tools.go` | `ToolRunner` 接口新增 `See/Hear/Smell/Move/SpeakTo`；`speak` 工具 schema 加 `scope/target_seat`；`move_district` 保留为兼容别名（派发改写为 move+bus） | `tools_test.go` 更新后通过 |
| `agent/wealthtypes/context.go` | 新增 `NeighborBrief` / `AmbianceBrief` / `SenseResult`；`GameContext` 加 `Surroundings` / `Ambiance` | 编译通过 + JSON 键名与文档 1 §3.3 一致 |
| `game/wealth/agent_runner.go` | 实现 5 个新 ToolRunner 方法；感知结果确定性构造（同区座位 + Backdrop 抽样档案 + 挂单 + 当月事件过滤） | `agent_observability_test.go` 等回归通过 |
| `game/wealth/districts.go` | 16 城区各加 `AmbianceBase{Smells, Sounds}` 静态表 | 表长 16，单测断言非空 |
| `game/wealth/events.go` | 事件类型 → 气味/声响标签映射表（失业潮/暴雨/集市/施工/救护等） | 单测覆盖映射命中 |
| `game/wealth/view.go` | `ClientGameState` 加 `my_local_pos`、`city.ambiance`（`omitempty`）；感知结果写入 `bot_contexts[].last_senses` | 回放兼容（旧帧无新字段不报错） |
| `game/wealth/actions.go` | `move` 三模式计费/精力差分（taxi>metro>bus；区内 walk/run 只改 local_pos 不进 Ledger） | 单测三模式结算断言 |
| `errcode/errcode.go` | 新增 35100–35103（ErrWealthSenseInvalid/SenseLimit/MoveForbidden/WhisperTarget） | 错误码表无冲突 |
| `ws/game_service_wealth.go` | `speak scope=private` 定向聊天帧（`whisper:true`，仅目标+观战者） | 参照狼人杀 WhisperFromBot 路径 |

### 1.3 限次与调度

| 文件 | 改动 |
|---|---|
| `agent/wealthplayer/run.go` | 月度循环重置感知计数（see/hear/smell 各 ≤2）；speak 预算 1→2（area+private 合计） |
| `agent/wealthplayer/prompt.go` | System prompt 更新：自我介绍为「城市居民」，教授 5 个新工具用法与感官范围语义；移除「玩家/对局」措辞 |

### 1.4 后端测试（新增/更新）

| 文件 | 内容 |
|---|---|
| `agent/wealthplayer/tools_sense_test.go`（新建） | 5 工具派发、参数校验、限次 35101、move 三模式、私聊定向 |
| `game/wealth/room_city_test.go` | 回归：建城/锚定/月结顺序不变 |
| `game/wealth/city/city_test.go`、`profile_test.go` | 回归 + voice 的 AgentClass 断言 |
| `api/room_api_resident_test.go`、`api/wealth_city_api_test.go` | 契约回归（字段名不变） |

**总门禁**：`go build -o LsmAgentGame main.go` + `go test ./...` 全绿（CLAUDE.md §4）。

## 2. 前端改动清单（`ClientWeb/`）

| 文件 | 改动 | 验收 |
|---|---|---|
| `i18n/locales/zh-CN.ts` | 文档 2 §1.1 全表逐键替换 | grep 零命中旧词 |
| `i18n/locales/en.ts` / `ja.ts` | 文档 2 §1.2/§1.3 同步 | 键集三语一致 |
| `i18n/wealthP2-{zh,en,ja}.ts` | `wealth.loan.subtitle`、`wealth.info.category.intel` 替换 | — |
| `components/wealth/WealthCreateRoomModal.tsx` | 标题「创建城市」；字段标签替换；删「创建者身份」行；agent_seats 档位改 1–12 默认 12；提交载荷不变 | 渲染截图核对 + 载荷契约单测（如有） |
| `pages/WealthLobbyPage.tsx` | banner 副标题、大厅提示新键值 | — |
| `pages/WealthGamePage.tsx` | 顶栏「我的视角/观察者」徽章；全 Agent 房隐藏「提前开始」 | — |
| `components/wealth/CityStatsPanel.tsx`（+css） | 「城市之声」标题；在区居民 | — |
| `components/wealth/WealthBotPanel.tsx` | 标题「居民思维」；新增「感知」三段式小节（读 `bot_contexts[].last_senses`） | 焦点居民决策后面板可见 |
| `components/wealth/MonthTicker.tsx` | 「模拟月」文案 | — |
| `components/wealth/GameOverModal.tsx` | 「模拟收官 / 人生结局评定」 | — |
| `components/wealth/ActionPanel.tsx:633` | 硬编码 tooltip 改「居民间借贷」 | grep 核对 |
| `types/wealth.ts` | 新增 `WealthSenseResult` 类型 + `bot_contexts.last_senses` / `city.ambiance` / `my_local_pos` 字段类型 | `tsc --noEmit` 通过 |

**总门禁**：`tsc --noEmit` + `npm run build` 通过（CLAUDE.md §4）；CSS 若新增类，同提交完成 JSX+CSS+`prefers-reduced-motion` 三件套（§26.3）。

## 3. 联调与发布

1. 建房链路改动 → 必跑 E2E 冒烟（含验证码登录、建房→自动开局→3 个月 tick→感知工具触发→私聊可见性）。
2. `./rebuild_restart_app.sh` 重新编译运行；确认服务健康（39001/39002）。
3. git 提交：lag_docs 子模块先 commit+push，主仓库再提交指针 + 代码改动（中文提交信息）。

## 4. 回滚预案

- 本次重构不改经济数值与帧结构，回滚 = revert 对应 commit。
- `move_district` 兼容别名保留一个大版本，旧回放/旧前端不断链。
