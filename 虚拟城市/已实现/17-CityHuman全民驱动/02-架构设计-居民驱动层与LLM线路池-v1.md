# 虚拟城市 — 居民驱动层与 LLM 线路池（架构设计）v1

> **批次**：17-CityHuman 全民驱动 ｜ **日期**：2026-09-22 ｜ **性质**：实现契约（后端）
> **目标**：虚拟城市中的**每个居民都是 `LsmAgentGame-City-Human`**：拥有自己的系统提示词
> （档案人格）、Context 内容（月度状态）与通用 Tools 定义；由**驱动层**统一驱动运行——
> 上层是**线程池**（worker pool），底层是 **LLM 线路池**（并发 = Σ线路数）。
> 关联：[`02-LLM线路池` 历史契约（git 1a62ba44e2 取源）](../../../财商流游戏.md)（已删目录，见 README §断链说明）、
> [`../03-Agent设计/虚拟城市-城市居民人物卡档案锚定设计-v1.md`](../03-Agent设计/虚拟城市-城市居民人物卡档案锚定设计-v1.md)。
> 本文档中所有类型名 / 配置键 / JSON 字段 = 实现契约。

---

## 1. 分层总览

```
┌─ 深度层（引擎内部,固定 12,无用户概念）────────────────────────┐
│ 12 名深度居民: wealthplayer.Agent 全量 46 工具月度决策循环      │
│ 房间信号量 agentSem = max(LinePool.Total(),1)                  │
├─ 驱动层（本批新增 ResidentDriver）────────────────────────────┤
│ 每月预算内抽取背景居民 → 线程池 worker 执行「居民轮次」:        │
│   System=档案人格 persona / User=月度状态 Context              │
│   Tools=通用工具 set_intent + speak（单轮解析,无二轮循环）      │
│ 每次 LLM 调用 LinePool.Acquire(ctx) —— 全局并发 = Σ lines     │
├─ 城市之声（并入驱动层）───────────────────────────────────────┤
│ driver 启用: speak 产出 = VoiceRecord(城市之声事件+环形缓冲)   │
│ driver 关闭: 回退既有 VoiceScheduler（零回归）                 │
├─ 数值层 ─────────────────────────────────────────────────────┤
│ Backdrop.TickMonth: 全体居民数值演化（消费 set_intent 加成）   │
└──────────────────────────────────────────────────────────────┘
LLM 线路池 llm.LinePool: M = Σ enabled(provider).concurrency_lines
= 「Agent 调用大模型的并发数」;所有调用方(深度/驱动/之声/其他游戏)共享。
```

**成本算术**（为什么驱动层是抽样预算制而非全员逐月）：10 万居民 × 每月 1 次 × 3~8s/次，
即使 64 条线路也需 1.3~3 小时/月。驱动层以 `per_month` 预算 + 跨月轮转保证
「每个居民都会被驱动到，且城市规模与线路数解耦」；深度层 12 名不受此限。

## 2. ResidentDriver（`ServerGo/game/wealth/city/driver.go` 新文件）

```go
// DriverConfig 驱动层配置(config [wealth] 段注入)。
type DriverConfig struct {
    Enabled  bool // city_driver_enabled,缺省 true
    Workers  int  // city_driver_workers,线程池大小,缺省 4,clamp [1,16]
    PerMonth int  // city_driver_per_month,每月驱动居民数,缺省 8,clamp [0,64]
    AcquireTimeoutMS int // 线路租约等待,缺省 15000
}

// LinePoolSource 与 voice.go 同名接口复用(池 Reload 后指向新池)。
type LinePoolSource interface{ LinePool() *llm.LinePool }

// ResidentDriver 居民驱动层:线程池 + 线路池 + 跨月轮转游标。
type ResidentDriver struct {
    cfg    DriverConfig
    pool   LinePoolSource
    cursor uint64 // 跨月公平轮转游标(原子)
    mu     sync.Mutex
    lastDriven int // 最近一个月实际完成轮次数(Snapshot 下发)
}

// RunMonth 驱动一个月的居民轮次(月结后异步调用,绝不阻塞月结)。
// picks = b.PickDriverCandidates(cfg.PerMonth, &d.cursor)
// 任务chan → cfg.Workers 个 worker(线程池) → runOne → WaitGroup。
func (d *ResidentDriver) RunMonth(b *Backdrop, month int, onVoice func(VoiceRecord))
```

### 2.1 居民轮次 runOne（单居民单月）

```
1. brief := b.driverBrief(idx)          // 锁内快照:档案人格 + 月度状态 + 邻居/氛围
2. ctx 15s → lease := pool.LinePool().Acquire(ctx)   // ErrAllLinesBusy → 丢弃本条
3. req := LLMRequest{
     Model: lease.ModelKey, AgentClassName: AgentClassCityHuman,
     System: personaBlocks(brief),      // §3.1
     Messages: [{user: contextText(brief)}],  // §3.2
     Tools: commonToolDefs(),           // §3.3 set_intent / speak
     MaxTokens: 256,                    // driverMaxTokens
   }
4. resp := chatViaLease(ctx, lease, req)  // 复用 voice.go 的租约调用路径(流式优先)
5. 解析 resp tool_use 块(单轮,不发 tool_result 二轮):
   · set_intent → b.ApplyIntent(idx, intent, targetDistrict)   // 锁内,§4
   · speak(text ≤60字) → VoiceRecord{Month,Name(真实档案名),Text,ModelKey,ResidentID,Occupation}
     → onRecord 回调(房间层 emit city_voice 事件 + 环形缓冲,复用 launchCityVoices 现有管线)
6. defer lease.Release();完成计数 → d.lastDriven(锁内)
```

失败语义与 voice.go 一致：单条失败静默丢弃（Debug 日志），不重试、不阻塞月结、不影响其他 worker。

### 2.2 抽样与公平：PickDriverCandidates

```go
// backdrop.go 新增:
// PickDriverCandidates 返回本月经线路池驱动的居民下标。
// 规则: stressed/失业居民优先(与 PickVoiceCandidates 同权重),其余按 cursor
// 全域轮转(跨月公平:N=10万、预算8/月 → 约 416 个月覆盖全员一圈);
// 同月不重复;返回数 = min(n, len(residents))。
func (b *Backdrop) PickDriverCandidates(n int, cursor *uint64) []int
```

## 3. 每个居民的 Agent 定义（persona / Context / 通用 Tools）

> 满足「每个居民都有自己的系统提示词、Context 内容、以及通用的 Tools 定义」：
> 三者皆由驱动层**按居民惰性构造**（档案锚定后即用真实档案；未锚定时降级代号，
> 与 voice.go 现行降级链一致）。

### 3.1 System 提示词（personaBlocks，档案人格）

```
你是虚拟城市居民「{姓名}」，{age}岁，{occupation}（{domain}，住在{district}）。
性格：{personality}。{opening_hook} 你的 5 年目标：{goal}。
婚姻{marital}，健康档 {health_grade}。请始终以这名居民的身份思考与说话。
```

（锚定前降级：`你是虚拟城市的一名普通居民,代号 {codename},从事 {domain} 行业,住在{district}…`）

### 3.2 User Context（contextText，月度状态）

```
■ 本月状态（第 {month} 月）
收入 {income} 元/月（{employed|失业中}）；支出 {expense} 元；储蓄 {savings} 元（约 {months_runway} 个月开支）{stressed}。
所在城区：{district}。城区氛围：{smells}/{sounds}。
附近居民：{neighbor 姓名(职业) × ≤3}。
请调用 set_intent 设定你本月的打算；如果想对街坊说句话，再调用 speak。
```

### 3.3 通用工具（commonToolDefs，Anthropic wire 四键约束 §14.1）

| 工具 | InputSchema | required | 语义 |
|---|---|---|---|
| `set_intent` | `{intent: "job_seeking"\|"frugal"\|"consume"\|"socialize"\|"move_out", target_district?: string(城区 id,move_out 必填)}` | intent | 本月意图：写入居民 intent 位，下个 TickMonth 消费（§4）后清除 |
| `speak` | `{text: string(1..60 字)}` | text | 对同城区说一句话 → 城市之声（事件 + 环形缓冲 + hear 可闻） |

## 4. 意图消费（backdrop.go TickMonth 扩展）

`resident` 增加 `intent uint8`（0=none 1=job_seeking 2=frugal 3=consume 4=socialize 5=move_out）+
`moveTarget uint8`（目标城区，仅 move_out）。TickMonth 内按位消费后**清除**（意图只生效一个月）：

| intent | 数值效果 |
|---|---|
| job_seeking | 失业者本月再就业概率 15% → 30% |
| frugal | 本月 expense ×0.9 |
| consume | 本月 expense ×1.25（拉动消费品市场口径不变，仅居民侧） |
| socialize | 本月 stress 解除概率 +20%（stressed 位清退加成） |
| move_out | 本月末迁移至 moveTarget 城区（无校验失败则忽略） |

内存预算：resident 20B → +2B ≈ 22B，仍满足 ≤40B 契约；100K 常驻增量 <0.3MB。

## 5. 房间接线（`game/wealth/room_city.go`）

| 接线点 | 契约 |
|---|---|
| `startCityLocked` | `driver.cfg.Enabled` → `NewResidentDriver(cfg, poolSource)` 存房间；否则 `NewVoiceScheduler`（现状）。两者互斥，常量 `driverMinPerMonth=0` 时语义=关闭 |
| 月结后 | 现状 `launchCityVoices` 改为 `launchCityDriver`：driver 启用走 `RunMonth(b, month, onVoice=emitCityVoiceEvent)`；关闭走旧 `VoiceScheduler.Run`（零回归） |
| onVoice | 复用现有 `emitCityVoiceEvent`：`game.event{type:"city_voice"}` + 环形缓冲 + hear 全城可闻（speak 的公共语义） |
| Snapshot | `Backdrop.Snapshot()` 增 `Driver *DriverSnapshot`（`omitempty`）：`{enabled, workers, per_month, driven_last}`，经 `game.state.city` 下发 |
| 重启恢复 | Driver 无持久状态（cursor 重建从 0 起，漏几个月轮转可接受）；Backdrop 确定性重建不变 |

## 6. 前端可见性（最小契约）

- `types/wealth.ts` `WealthCitySnapshot` 增 `driver?: { enabled: boolean; workers: number; per_month: number; driven_last: number }`。
- `CityStatsPanel.tsx` 指标网格追加一行：`本月驱动 {driven_last} 名居民`（键 `wealth.cityDriven`）；
  `driver` 缺省（旧帧/关闭）时该行不渲染。
- 无新增 WS 帧；`city_voice` 事件渲染不变。

## 7. 配置（`config/config.go` → `[wealth]` 段 + `LsmAgentGame.conf.example`）

| 键 | 缺省 | clamp | 说明 |
|---|---|---|---|
| `city_driver_enabled` | true | — | 驱动层总开关（false 回退 VoiceScheduler） |
| `city_driver_workers` | 4 | [1,16] | 线程池 worker 数 |
| `city_driver_per_month` | 8 | [0,64] | 每月驱动居民数（0=仅深度层） |

零值归一与 `MaxResidents` 同款（`config.go:1090` 区域）。

## 8. 并发与锁纪律（§92a / §197）

1. Driver 全程**不持有 Backdrop.mu 跨 LLM 调用**：brief 锁内快照、ApplyIntent 锁内短临界区——与 voice.go 同款。
2. worker 数与 agentSem（深度层）互不影响：驱动层不经 agentSem，仅经全局 LinePool。
3. LLM 调用流式优先（`ChatStreamAccumulate`），15s 租约超时 + 单轮 256 max tokens——无长循环续命需求。
4. `cursor` 用 `atomic.Uint64`；`lastDriven` 用互斥锁保护（与 Snapshot 读竞争）。

## 9. 测试要求

| 用例（`city/driver_test.go` 新建） | 断言 |
|---|---|
| 预算 | fake pool 注入：PerMonth=5 → 恰 5 次 LLM 调用；5 条 speak → 5 条 VoiceRecord |
| 线程池 | Workers=4 + PerMonth=16 → 峰值并发 ≤4（fake pool 计数器） |
| 线路全忙 | Acquire 恒 ErrAllLinesBusy → 不 panic、不阻塞、lastDriven=0 |
| 轮转公平 | 两月 PerMonth=2、N=4 → 两月 picks 并集 = 全员、无重复 |
| 意图演化 | set_intent(job_seeking) 后 TickMonth：失业者再就业率显著上升；次月 intent 清零 |
| move_out | ApplyIntent(move_out, target) → TickMonth 后 district==target |
| 回退 | Enabled=false → VoiceScheduler 行为零变化（既有 city_test 回归） |
| 驱动开关下发 | Snapshot.Driver 字段 omitempty；关闭时前端不渲染 |

**总门禁**：`go build` + `go test ./...`（city 包含 100K 性能断言不回归）。

---

## 更新日志

- v1（2026-09-22）：首版。ResidentDriver 契约：线程池/线路池/persona/Context/通用工具/意图消费/接线/配置/测试。
