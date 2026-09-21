# 虚拟城市 — 城市 Agent 与线路池验收报告 v1

> 验收日期：2026-09-21。验收对象：契约 01–04 全部条目。
> 环境：Linux 6.8，`main` 分支单工作区，`go test -count=1 ./...` + 前端 `tsc --noEmit` / `npm run build`。

## 1. 编译与全量测试门禁

| 门禁 | 结果 |
|---|---|
| `go build -o /tmp/final_check ./`（ServerGo） | ✅ 通过 |
| `go test -count=1 ./...` | ✅ **30 个包全部 ok，0 FAIL**（werewolf 11.1s / wealth 0.5s / wealth/city 0.2s / profession 16.9s / debate 7.9s / api / llm / ws / service / agent/* 等） |
| `go test -race ./game/wealth/ ./game/wealth/city/ ./agent/wealthplayer/` | ✅ 全绿（城市之声 goroutine / 池租约无数据竞争） |
| `npx tsc --noEmit`（ClientWeb，独立复核两次） | ✅ 零输出 |
| `npm run build` | ✅ `✓ built in 28.21s`（仅既有 >500kB chunk advisory，与本次无关） |

## 2. 契约落地核对（01 §5 验收标准逐条）

| # | 标准 | 结果 | 证据 |
|---|---|---|---|
| 1 | 编译 + 全量测试 + 前端门禁 | ✅ | §1 |
| 2 | 线路池并发上限 = Σ线路数 | ✅ | `llm/linepool_test.go`（291 行）：M=3 恰 1 阻塞 + Release 放行；加权轮询 A2/B1→A4B2；ctx 超时 `ErrAllLinesBusy` 且令牌数不变；Release 幂等 |
| 3 | 100K 居民创建 <1s / 内存 <32MB / 月度 tick <100ms | ✅ | `game/wealth/city/city_test.go`：100K 初始化 <200ms（实测居民结构 20B/人，100K≈2MB）、tick <100ms（CI 放宽 2× 余量充足）；同 seed 同城确定性断言 |
| 4 | 管理页无 Agent Name、总线路数正确、缺省可建 | ✅ | ModelAdminPage 列/表单已删 agent_name，新增线路数列与总线路数提示条；`model_admin_api_test.go`：无 agent_name 创建成功、派生 `model-2/-3` 唯一、越界 400 |
| 5 | 池驱动 12 焦点座位全决策 + 城市之声走线路池 | ✅ | `room_city_test.go`：池驱动 12 Agent 决策完成全链路、混合座位兼容；city 包 fake pool 发声条数 = 配置、Acquire 失败不 panic 不阻塞 |
| 6 | 其他游戏零回归 | ✅ | werewolf/debate/xiangqi/texas 全量测试 ok；`registry.Get` 原语义未动；werewolf 空 model_key 仍拒绝（`TestValidateAgentSeats_EmptyKeyRejected` 保留） |

## 3. 交付物清单

### 3.1 ServerGo（新 10 文件 + 改 33 文件）

| 模块 | 文件 | 行数 |
|---|---|---|
| LLM 线路池 | `llm/linepool.go`（新） | 270 |
| 线路池测试 | `llm/{linepool,registry_linepool}_test.go`（新） | 291+203 |
| 城市背景 | `game/wealth/city/{backdrop,calibration,voice}.go`（新） | 458+303+165 |
| 城市测试 | `game/wealth/city/city_test.go`（新） | 498 |
| 房间城市接线 | `game/wealth/room_city.go` + `room_city_test.go`（新） | 187+测试 |
| 职业域抽取 | `game/wealth/profession/domain.go`（新，`DrawWithDomain`） | 41 |
| Agent 池模式 | `agent/wealthplayer/run_llm_pool_test.go`（新） | — |
| 建房参数测试 | `api/room_api_resident_test.go`（新） | — |
| 核心修改 | `models/t_lsm_game_llm_provider.go`（+concurrency_lines）、`llm/{registry,defaults,types}`、`api/{model_admin_api,llm_api,room_api}`、`game/wealth/{room,manager,view}`、`agent/wealthplayer/{agent,run_llm}`、`agent/class_names(+test)`、`service/room_service*`、`ws/game_service*`、`config/config.go`、`main.go`、`LsmAgentGame.conf.example` | — |

### 3.2 ClientWeb（新 2 文件 + 改 17 文件）

- 新：`components/wealth/CityStatsPanel.tsx`（123 行）+ `CityStatsPanel.css`（177 行，组件内 import，未动 globals.css）。
- 改：ModelAdminPage（1649→1672 行，余量 128）、ModelDetailPage、WealthCreateRoomModal（515→478 行，净减）、WealthGamePage、WealthLobbyPage、RoomListTable、ModelAnalyticsPanels、types/{model,wealth,api}、api/llm.ts、i18n 三语 + types + wealthKeys、lobby.css。

### 3.3 i18n

新增 16 键 / 删除 2 键（`modelAdmin.colAgentName/fieldAgentName`），zh-CN/en/ja/types/wealthKeys 同步；tsc Dict 类型检查即完备性证明。

## 4. 实现偏差记录（均已注释说明，需知悉）

| # | 偏差 | 理由 |
|---|---|---|
| 1 | 城市月结 cpi 取 `World.Goods.CPIMom`（消费篮子月环比）而非央行 CPI 年率 | 月度 tick 需要月率量纲；查不到回落 0.002 |
| 2 | `city_voice_per_month=0` 经 conf 不可达（零值归一为 4） | 与 `economy_enabled` 等既有配置同款取舍；关闭走 `city_voice_enabled=false`；代码内 clamp [0,32] 保留 |
| 3 | `resident_count` 不持久化（重启后城市配置丢失，座位/对局恢复不受影响） | 查证 `month_ms/pool/seed` 现状即仅存内存 `pendingOpts`，遵循现状不新建表；Backdrop 本身确定性可重建，后续补房间选项持久化后即闭环 |
| 4 | 居民之声代号的城区名由 city 包独立同步 wealth `DistrictDefs` 8 中文名 | 避免 city→wealth 反向 import |
| 5 | 显式 `model_key` 座位昵称保持 `AI·<model_key>` 旧形态 | 存量兼容；仅池驱动座位升级为 `AI·<职业卡人名>` |

## 5. 后续观察项（不在本轮范围）

- 狼人杀是否切换线路池并发模型（01 §6 D5）。
- 房间选项（month_ms/pool/seed/resident_count）统一持久化机制。
- 城市背景层与劳动力/商品市场的深度耦合（当前仅统计快照 + CPI 传导）。
- 10 万职业卡池的 L3 归并（v5.0 计划 dry-run 未落盘）对校准抽样的影响。
