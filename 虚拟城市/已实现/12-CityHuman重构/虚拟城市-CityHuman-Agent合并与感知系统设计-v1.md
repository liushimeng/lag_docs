# 虚拟城市 — City-Human Agent 合并与感知系统设计 v1

> **版本**：v1 ｜ **日期**：2026-09-22 ｜ **状态**：设计定稿（P0 实施中）
> **上游定位**：[`../../虚拟城市定位声明-全Agent真实城市模拟器.md`](../../虚拟城市定位声明-全Agent真实城市模拟器.md)
> **配套文档**：[`虚拟城市-界面文案与建房流程重构设计-v1.md`](虚拟城市-界面文案与建房流程重构设计-v1.md)（文档 2）、[`虚拟城市-CityHuman重构实施清单-v1.md`](虚拟城市-CityHuman重构实施清单-v1.md)（文档 3）
> 本文档中所有文件路径、包名、类型名、工具名、JSON 字段名 = 后续实现的契约。

---

## 1. 背景与目标

虚拟城市自 2026-09-22 起定位为**全 Agent 真实城市模拟器**：没有玩家、没有胜负，
城市由全体居民 Agent 的行为涌现。原架构沿用了「财商流游戏」时期的双层 Agent 设计：

| 层 | 旧 AgentClassName | 职责 |
|---|---|---|
| 焦点层（12 座） | `LsmAgentGame-City-Player` | LLM 每月决策、40 个工具循环 |
| 背景层（1~10 万） | `LsmAgentGame-City-Voice` | 无工具、抽样极短发声 |
| 规划常量（零消费方） | `LsmAgentGame-City-Government` / `-Banker` / `-Firm` | 政府/央行/企业公告 |

这套命名仍带着「Player（玩家）」的游戏语义，且五个 AgentClass 割裂了「城市居民」这一统一身份。
本次重构：

1. **五个 AgentClass 合并为唯一 `LsmAgentGame-City-Human`** —— City-Human 即「城市居民」，
   虚拟城市不存在玩家 Agent。焦点层与背景层共用同一 AgentClass，仅调用深度不同。
2. **Agent 工具重设计** —— 新增符合真实世界尺度的感知与行动工具（看见 / 听见 / 闻到 / 移动 / 说话），
   删除不必要设计，**全部经济系统工具保留不回归**。
3. **档案驱动** —— 每个 City-Human 由 `玩家职业设计` 知识库人物卡（Schema v1.1，65 字段）驱动，
   感知结果中的人物信息同样来自真实档案。

## 2. Agent 合并：唯一 `LsmAgentGame-City-Human`

### 2.1 合并方案

- `ServerGo/agent/class_names.go` 收敛为单常量：

  ```go
  AgentClassCityHuman = "LsmAgentGame-City-Human"
  ```

- 删除常量：`AgentClassCityPlayer`、`AgentClassCityVoice`、`AgentClassCityGovernment`、
  `AgentClassCityBanker`、`AgentClassCityFirm`（后三者本就零消费方，属「声明了却从不接线」，
  按 §130 教训直接移除；未来政府/央行/企业公告若接入 LLM，一律复用 `City-Human` + 角色提示词区分，
  不再新增 AgentClass）。
- `AllAgentClassNames()` 同步收敛；`class_names_test.go` 断言更新。
- 所有出站 LLM 请求（焦点层月度决策、背景层抽样发声、法官式公告）的
  `LLMRequest.AgentClassName` 统一填 `LsmAgentGame-City-Human`。
  User-Agent 形如 `LsmAgentGame-City-Human/<AppVersion> <buildDateTime>`（CLAUDE.md §24）。

### 2.2 双层共用、深度分级

合并后两类调用方共享 AgentClass，但在调度参数上保持分级（沿用 v2.12 调度策略的预算隔离）：

| 维度 | 焦点层居民（座位 1~12） | 背景层居民（抽样发声） |
|---|---|---|
| 调用入口 | `agent/wealthplayer/run.go OnMonthStart` | `game/wealth/city/voice.go VoiceScheduler` |
| 工具 | 全部工具（经济 40 个 + 感知行动 5 个） | 无工具，单次极短对话 |
| Memory | 局内滚动记忆（memory.go） | 无 |
| 并发/超时 | agentSem=8、decisionTimeout=20s | 共享 LinePool、voiceAcquireTimeout=15s |
| maxTokens | 决策循环自有预算 | 128（voiceMaxTokens） |
| prompt 差异 | 全量人设 + 经济状态 + 感知上下文 | 仅姓名/职业/人格/opening_hook 一句话人设 |

> 设计原则：**AgentClass 是业务身份，不是调度档位**。「都是城市居民」→ 同一 AgentClass；
> 「调用多深」→ 由调用方预算控制，不再靠 AgentClass 区分。

### 2.3 术语对齐

代码内标识符（包名 `wealthplayer`、类型 `WealthPlayer`、`game_kind=wealth` 等）属技术契约，
**保持不变**以避免大规模回归；变更仅限：AgentClassName 字符串、用户可见文案、日志/注释中的「玩家」表述。

## 3. 空间与感官模型

### 3.1 空间模型

- 城市 = 16 城区（`game/wealth/districts.go` 静态表）。
- 每个居民有：所在城区 `district` + 区内位置 `local_pos`（区内归一化坐标 `[0,1]²`，仅用于
  区内移动与感知排序，不上 3D 地图逐人渲染）。
- 感知范围按真实世界尺度定义（用于工具结果的过滤与排序，不做连续物理模拟）：

| 感官 | 现实范围 | 模拟语义 |
|---|---|---|
| 视觉 see | ≈ 500m | 同城区：人 / 物 / 事 |
| 听觉 hear | ≈ 100m | 同城区近处：公开发言 / 事件声响 / 城市之声 |
| 嗅觉 smell | ≈ 50m | 所在城区气味画像（基底 + 动态事件叠加） |

### 3.2 城区气味基底表（`districts.go` 扩展）

每个城区定义 `AmbianceBase`（2~4 个气味标签 + 1~2 个环境声标签），示例：

| 城区 | 气味基底 | 环境声基底 |
|---|---|---|
| 金融CBD | 咖啡、打印机墨粉、空调新风 | 键盘声、电梯提示音 |
| 工业区 | 油烟、尾气、金属切削液 | 机器轰鸣、货车倒车提示 |
| 商业中心 | 食物香气、香水、爆米花 | 促销广播、人群嘈杂 |
| 文创区 | 咖啡、油墨、旧书页 | 街头艺人、轻声交谈 |
| 居住区 | 饭菜香、洗衣液、绿化泥土 | 广场舞音乐、儿童嬉闹 |
| 中央公园 | 青草、花香、湖水湿气 | 鸟鸣、风声 |
| 物流港 | 柴油、海腥、纸箱 | 吊机作业、集卡鸣笛 |
| 医疗城 | 消毒水、药味 | 救护车笛、叫号广播 |
| 其余 8 区 | 按城区主色调性定义（实现清单见文档 3） | 同左 |

动态叠加：当月事件（失业潮→焦虑汗味/工地停工→扬尘、暴雨→潮湿霉味、
集市/节日→烟火与食物）按 `events.go` 事件类型映射气味/声响标签，注入当月 `Ambiance`。

### 3.3 GameContext 扩展（`agent/wealthtypes/context.go`）

```go
// GameContext 新增（引擎持锁构造、Agent 锁外只读，沿用现有契约）：
Surroundings []NeighborBrief  // 同城区邻居摘要（座位居民优先 + 抽样背景居民，≤8 条）
Ambiance     AmbianceBrief    // 当月所在城区感官画像
```

```go
type NeighborBrief struct {
    Kind       string `json:"kind"`        // "seat" | "resident"
    Seat       int    `json:"seat,omitempty"`
    CardID     string `json:"card_id,omitempty"`
    Name       string `json:"name"`
    Occupation string `json:"occupation"`
    District   string `json:"district"`
    MoodHint   string `json:"mood_hint,omitempty"` // 由压力/情绪字段映射的一词状态
}

type AmbianceBrief struct {
    District string   `json:"district"`
    Smells   []string `json:"smells"`  // 基底 + 动态事件气味
    Sounds   []string `json:"sounds"`  // 基底 + 动态事件声响
}
```

可见性：与 `bot_contexts` 一致 —— 仅本人与观战者可见，**不走 BroadcastRoom 全量下发**。

## 4. Agent 工具重设计

### 4.1 新增感知与行动工具（5 个）

定义于新文件 `ServerGo/agent/wealthplayer/tools_sense.go`（避免 tools.go 超 §4 行数上限）。

| 工具 | 参数 | 语义 | 预算 |
|---|---|---|---|
| `see` | `{}` | 看见同城区的人（座位居民 + 抽样背景居民：姓名/职业/状态）、物（本区挂牌/店铺/建筑）、事（本月本区正在发生的事件） | 不耗动作预算，每月 ≤2 次 |
| `hear` | `{}` | 听见同城区近期公开发言摘录、事件声响、城市之声（voice 记录） | 不耗预算，每月 ≤2 次 |
| `smell` | `{}` | 闻到所在城区气味画像（§3.2 基底 + 动态叠加） | 不耗预算，每月 ≤2 次 |
| `move` | `{destination: string, mode: "walk"\|"run"\|"bus"\|"metro"\|"taxi"}` | 统一移动。walk/run = 区内移动（改 `local_pos`，run 消耗更多精力、更快）；bus/metro/taxi = 跨城区（语义等同并替代 `move_district`：taxi 最贵最快、bus 最便宜、metro 居中；价格随 CPI 浮动） | **耗动作预算**（占每月 ≤3 动作之一） |
| `speak` | `{scope: "area"\|"private", target_seat?: int, text: string}` | area = 同城区范围公开放话（原行为，走 ChatSender 公屏）；private = 对指定座位耳语（仅目标与观战者可见，走 `bot_contexts` + 定向聊天帧） | 每月 area+private 合计 ≤2 次（原 ≤1 放宽为 ≤2） |

### 4.2 旧工具处置

- **保留不动（35 个经济/社会工具）**：`check_state`、`buy_asset`、`sell_asset`、`buy_house`、
  `take_loan`、`repay_loan`、`start_side_business`、`stop_side_business`、`study`、`socialize`、
  `rest`、`work_overtime`、`consume`、`donate`、`submit_month`、`query_central_bank`、
  `query_banking_system`、`apply_loan_with_credit`、`deposit_savings`、`withdraw_savings`、
  `query_minsky`、`early_repay`、`set_consumption`、`answer_survey`、`query_economy`、
  `buy_insurance`、`cancel_insurance`、`get_insurance_status`，以及 P2 交易 12 工具
  （`list_asset`/`cancel_listing`/`view_listings`/`start_negotiate`/`respond_negotiate`/
  `create_loan_listing`/`accept_loan`/`repay_p2p_loan`/`add_guarantor`/`bid_auction`/`sell_info`/`bid_info`）。
- **兼容保留**：`move_district` 保留为 `move` 跨城区模式的别名入口（派发层重写为
  `move{destination, mode:"bus"}`），防止旧 prompt/回放断链；prompt 中不再主动教 Agent 使用它，
  下个大版本删除。
- **删除**：无。本次重构不删除任何已接线工具（「删除不必要的设计」落实为：感知/移动/说话
  不再拆成更多零碎工具，统一为 5 个；政府/银行/企业三个零消费方 AgentClass 删除）。

### 4.3 ToolRunner 接口扩展（`agent/wealthplayer/tools.go` → `game/wealth/agent_runner.go` 实现）

```go
// ToolRunner 新增 5 个方法（in-process，不走 WS）：
See(seat int) (*wealthtypes.SenseResult, error)
Hear(seat int) (*wealthtypes.SenseResult, error)
Smell(seat int) (*wealthtypes.SenseResult, error)
Move(seat int, destination string, mode string) error
SpeakTo(seat int, targetSeat int, text string) error  // speak scope=private
```

```go
// wealthtypes 新增：
type SenseResult struct {
    District string          `json:"district"`
    People   []NeighborBrief `json:"people,omitempty"`   // see 专用
    Things   []string        `json:"things,omitempty"`   // see：挂牌/店铺/建筑
    Events   []string        `json:"events,omitempty"`   // see/hear：本区本月事件
    Utterances []string      `json:"utterances,omitempty"` // hear：近期公开发言摘录
    Smells   []string        `json:"smells,omitempty"`   // smell
    Sounds   []string        `json:"sounds,omitempty"`   // hear/smell 共用环境声
}
```

实现要点（`game/wealth/agent_runner.go` / `room.go`）：
- 感知结果**确定性构造**（不调 LLM）：人 = 同区座位快照 + `city.Backdrop` 抽样档案（profile.go 已锚定
  真实姓名/职业）；物 = 本区 `listing.go` 挂单 + 城区静态建筑；事 = `events.go` 当月事件按城区过滤；
  声/味 = §3.2 基底表 + 事件映射。
- 每月感知次数计数挂在 Agent 实例（随 `OnMonthStart` 重置），超限返回工具错误
  `ErrSenseLimit = 35101`。
- `Move` 跨城区复用现有 `move_district` 的校验与结算（`actions.go` 单一代码路径），
  仅按 mode 追加费用/精力差分；区内移动只改 `local_pos`，不进 Ledger。
- `SpeakTo` 走 `ChatService` 定向通道（参考狼人杀 `WhisperFromBot`），房间内仅目标座位
  与观战者收到。

### 4.4 错误码新增段（35100 起，`errcode/errcode.go`）

| 码 | 常量 | 含义 |
|---|---|---|
| 35100 | `ErrWealthSenseInvalid` | 感知/移动工具参数非法（未知城区、未知 mode、目标不存在） |
| 35101 | `ErrWealthSenseLimit` | 当月感知/发言次数超限 |
| 35102 | `ErrWealthMoveForbidden` | 当前状态不允许移动（破产清算中等，沿用现有动作门控语义） |
| 35103 | `ErrWealthWhisperTarget` | 私聊目标不可达（目标死亡/非座位居民/跨房） |

## 5. 人物卡驱动（保持不变 + 感知增强）

- 建房后 `city/profile.go AnchorProfiles` 依旧从 `玩家职业设计` 文档池（75,115 张、Schema v1.1）
  不重复抽卡 → 并行解析 frontmatter → 锚定居民；合成数值兜底不阻塞开局。
- 焦点层 prompt（`agent/wealthplayer/prompt.go`）继续注入 `CardBrief` 投影：
  人格标签、行为特征、风险偏好、人生目标、opening_hook —— 不变。
- **新增**：感知工具（see/hear）返回的 `NeighborBrief` 全部来自已锚定的真实档案
  （姓名/职业/城区/情绪提示），不再出现「居民 #12345」式占位；档案未就绪时降级为
  合成姓名（`profile.go` 现有兜底链），并在结果中省略 `card_id`。

## 6. 可见性与前端契约

- WS 帧结构**不变更**：感知结果写入 `bot_contexts[].last_senses`（本人 + 观战者可见），
  前端 `WealthBotPanel` 追加「感知」小节渲染（看见/听见/闻到三段式）。
- `speak scope=private` 产生定向聊天帧（`chat.message` + `whisper: true` 标记位，
  复用狼人杀耳语帧惯例），仅目标座位与观战者渲染。
- `ClientGameState` 新增（`omitempty`，保旧回放兼容）：
  - `my_district` 已有；新增 `my_local_pos`（区内坐标，仅本人/观战者）。
  - `city.ambiance`（每城区当月气味/声响标签，全员可见，供地图面板氛围渲染）。

## 7. 非目标（明确不做）

- 不做连续物理/碰撞模拟；感知范围是语义过滤不是物理引擎。
- 不改经济系统任何数值与结算顺序（央行/财政/产业链/金融市场/保险/交易/调研/社会统计全保留）。
- 不改 `game_kind=wealth`、路由 `/wealth/*`、包名等技术契约。
- 不新增除 `City-Human` 外的任何 AgentClass。
- 背景层居民仍不跑工具循环（10 万居民 × 工具循环 = 不可行成本），仅抽样发声。

## 8. 验收标准

1. `grep -rn "LsmAgentGame-City-Player\|LsmAgentGame-City-Voice\|LsmAgentGame-City-Government\|LsmAgentGame-City-Banker\|LsmAgentGame-City-Firm" ServerGo/` 零命中（历史文档除外）。
2. `grep -rn "LsmAgentGame-City-Human" ServerGo/` ≥ 5 处（常量登记 + 焦点层 + 背景层 + 测试断言 + 注释）。
3. `go build` + `go test ./...` 全绿；新增单测覆盖：5 个新工具派发、感知限次、
   move 三模式计费、私聊定向可见性、Ambiance 事件叠加。
4. 感知工具返回的人物均为真实档案姓名（profile ready 后），无占位编号。
5. 前端：`tsc --noEmit` + `npm run build` 通过；`WealthBotPanel` 可见感知三段式渲染。
