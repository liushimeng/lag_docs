# 批次 81 · City-Chrome Agent 设计（`LsmAgentGame-City-Chrome`）

> 日期：2026-11-03 ｜ 分支：`main`（§10.1）｜ 职责线：`city-chrome`（§13.1 B4）
> 上游方案：[`01-方案设计.md`](01-方案设计.md) §3.1（总线）、§10（分工）
> 既有实现：[`../../65-CityChrome实时数据通道与实时看板/01-方案设计.md`](../../65-CityChrome实时数据通道与实时看板/01-方案设计.md)、
> [`../../77-CityChrome规划执行能力增强/01-方案设计.md`](../../77-CityChrome规划执行能力增强/01-方案设计.md)、
> [`../../78-CityChrome采集确定性与可观测诊断/01-方案设计.md`](../../78-CityChrome采集确定性与可观测诊断/01-方案设计.md)、
> [`../../79-CityChrome采集Agent可选LLM线路/01-实施记录.md`](../../79-CityChrome采集Agent可选LLM线路/01-实施记录.md)
>
> 本文是 City-Chrome 的**单一设计说明**；实现事实以 `ServerGo/citychrome/` 代码为准。

---

## 1. 定位

City-Chrome 是虚拟城市的**数据采集员** Agent：用 ReAct 循环操作 Chrome
（经 `go-web-debug-tool` 的 CDP，服务端只走 HTTP，不直接依赖 chromedp），
访问**真实公开网站**，采集行情与新闻入 `FeedStore`，供三条出口消费：

1. **实时看板**（前端「外部实时 → 实时看板」）；
2. **City-Human 工具** `realworld_info`（op = overview/quotes/news/history/sources）；
3. **城市引擎联动**（批次66 外盘情绪锚、批次80 真实世界事件自动注入）。

**职责边界**：只采集、只读；不做经济决策、不改引擎状态、不参与月结 RNG。
数据通道是 **server 级单例**（与房间无关），`enabled=false` 或底座不可达时
整体空转（channel=offline），绝不阻塞对局。

---

## 2. AgentClassName 与注册

| 项 | 值 |
|---|---|
| AgentClassName | `LsmAgentGame-City-Chrome` |
| 常量 | `ServerGo/agent/class_names.go::AgentClassCityChrome`（已登记） |
| 实现 | `ServerGo/citychrome/agent.go::Agent` |
| User-Agent | `LsmAgentGame-City-Chrome/<AppVersion> <buildDateTime>`（§24） |
| 提示词三段头 | `sysprompt.EnsureHead(..., AgentClassCityChrome)`（§14.3，自动注入） |
| 总线注册 | **类级**（全局唯一，`citybus.Register`），非房间级 |

---

## 3. ReAct 循环（`Agent.RunMission`）

```
Mission{Category, TargetURL, Sites, Goal, Hints}   ← BuildMission(category)（sites.go）
  │
  ├─ system = sysprompt.EnsureHead([{roleText}], AgentClassCityChrome)   ← 三段式头
  ├─ user   = missionUserPrompt(m, mlog)
  │            = mission JSON + 输出 schema + 备选站点 + 跨 mission 经验
  │
  └─ for step := 0; step < MaxStepsPerMission(默认 14); step++ {
         resp := callLLM(...)            ← ChatStreamAccumulate 流式续命(§197)
         if len(resp.ToolUses()) == 0 { break }        ← 纯文本作答 → JSON 兜底
         for _, tu := range resp.ToolUses() {
             text, isErr := toolset.Dispatch(ctx, tu.Name, tu.Input)
             text = "[step i/N] " + text              ← 步数预算回灌,促收敛
             results = append(results, tool_result{tu.ID, text, isErr})
         }
         if toolset.Done != nil { return toolset.Done }  ← web_done 校验通过即 break
         messages += assistant(resp.Content) + user(results)   ← §14.1 交替回灌
     }
  └─ 无 web_done ⇒ payloadFromText(lastText) 文本 JSON 兜底;仍失败 ⇒ mission 失败
```

**要点**

- **modelKey 入口解析一次**（`resolveLineKey`，批次79），同一 mission 内恒定，
  避免多步跨模型漂移；线路名落遥测 `MissionRecord.Model`。
- **LLM 并发节流**：`acquireLLM`（Service 注入，cap 1 + 5s 超时），防手动刷新
  与自动轮转竞态。
- **资源卫生**：`defer` best-effort 回收 `Toolset.OpenedPages` 全部页面
  （`CloseChromePage` 对 2000 幂等）。
- **遥测**：入口 `Append(running)`，出口同 ID 覆盖为 `ok`/`fail`。

---

## 4. 工具集（11 个，`mcp_tools.go::BuildTools`）

| # | 工具 | 一句话职责 |
|---|---|---|
| 1 | `web_open` | 打开新 Chrome 页面，返回 page_id/title/final_url |
| 2 | `web_control` | 页面交互（白名单 action，含 `wait` 等元素；点击/开新页返回 spawned_page_id） |
| 3 | `web_eval` | BatchEval 批量 DOM 提取（1–64 条 eval_js，优先用它减少往返） |
| 4 | `web_look` | 只读观察页面（console/network/elements/dom/storage/cookies/performance/screenshot） |
| 5 | `web_list` | 列出全部受管页面（页面索引，2000 后对齐用） |
| 6 | `web_close` | 关闭页面回收资源 |
| 7 | `web_plan` | **本地**：提交结构化规划（goal? + steps 1–8），mission 第 1 步必须调用 |
| 8 | `web_read` | 一步 = 打开 + 等元素（wait_selector/wait_ms）+ 读正文 + 确定性解析（parse） |
| 9 | `web_search` | 必应/百度 SERP 只读提取，固定源全失败后的发现兜底，自动关页；反爬识别 |
| 10 | `web_parse` | **本地**：对任意文本做确定性解析（配方见 §5） |
| 11 | `web_done` | **唯一使命出口**：提交结构化 payload，服务端校验（quotes 上限/数值合法/未知类目拒绝） |

> 工具名常量集中在 `mcp_tools.go`（`ToolWeb*`）；`Toolset.Dispatch` 是唯一派发入口。

---

## 5. 确定性解析（`recipes.go`）

**目标**：行情数字**不由 LLM 手工切分**，配方是纯函数、无 IO，涨跌幅**服务端自算**。

| recipe | 适用正文 | 字段口径 |
|---|---|---|
| `auto`（缺省） | 任意 | 依次 `tencent_quote` → `tencent_hf`，首个命中 ≥1 条即用 |
| `tencent_quote` | `v_sh000001="1~上证指数~000001~3658.03~3713.03~…"` | name=[1] code=[2] price=[3] prev=[4] time=[30] |
| `tencent_hf` | `v_hf_GC="4036.75,-0.58,…,纽约黄金"` | price=[0] pct=[1] name=[13]，symbol = 变量名去 `hf_` |
| `sina_quote` | 新浪 `hq_str_*` | **仅显式指定**，不在 auto 链 |

- **涨跌幅**：一律 `(price-prev)/prev*100`；`prev<=0` 才回退站点字段，
  顺序 `[32]→[31]→[33]`，取首个 `|·|≤20` 的可解析值。
- **校验**：`price>0` 且有限；`change_pct` 钳制 `[-100,100]`；单次 ≤64 行。
- **出口形态**：`web_read` 返回「结构化块在前、正文在后」，结构化块不被截断；
  `web_parse` 未命中返回 `matched=false` + 换配方提示。
- **纪律**：命中时**不要**再手工切分；未命中才回退 `web_eval`。

---

## 6. 调度与遥测

### 6.1 Scheduler（`scheduler.go`）

- **分级轮转**：8 类目各自独立计时 —— 行情 6 类 `refresh_seconds`（默认 1800s）、
  `news` 600s、`macro` 86400s；每 tick（5s）取首个到期类目入队。
- **单飞**：全局同时只跑 1 个 mission（队列 cap 8，满则拒绝）。
- **熔断**：连续 3 次失败 ⇒ channel=offline + 指数退避（5m→10m→20m→封顶 30m），
  任一成功即复位 online。
- **空闲降载**：Hub 内无 `virtual_city` 房间 ⇒ 暂停自动轮转（手动刷新仍可用）。
- **同站点克制**：同 host 访问间隔 ≥10s。
- **手动刷新**：同类目 60s 限流（`ErrCityChromeRefreshTooFrequent`，429 语义）；
  通道 disabled ⇒ `ErrCityChromeChannelOffline`（35053）。

### 6.2 遥测与出口

| 结构 | 说明 |
|---|---|
| `MissionLog` | 容量 20 环形缓冲（新→旧，mutex，同 ID 覆盖更新） |
| `MissionRecord` | id/category/started_at/duration_ms/status/source/steps_used/max_steps/plan/quotes/news/error/visited/model |
| `MissionStats` | total/ok/fail/running/success_rate/avg_steps/avg_duration_ms/last_error |
| `FeedStore` | server 级单例：报价替换式合并 + 新闻 md5 去重 + 自采价环 ≤48 点 |

**出口**：WS `game.realtime_feed`（只含 feed）；HTTP
`GET /api/virtual_city/realtime_board` 的 `data.{feed,missions,stats}`（missions
只随 HTTP 快照/手动刷新回读，不进 WS 帧）。

---

## 7. 线路策略（批次79）

`Service.SetLinePolicy(line)` 透传到 `Agent`，取值：

| 值 | 语义 |
|---|---|
| `""`（缺省） | 沿用服务端 `city_chrome.model_key`（旧行为，零回归） |
| `"random"` | 每个 mission 从线路池按 `concurrency_lines` 加权随机取线；池空回退 `model_key` |
| 其它 | 固定 model key |

- **驱动链**：建房 body `city_chrome_line` → `main.go` 装配
  `cityChromeSvc.SetLinePolicy(opts.CityChromeLine)`（server 级单例，后建房覆盖）。
- **一次 mission 内固定线路**（入口 `resolveLineKey` 抽一次）。
- **遥测**：`MissionRecord.model` 落盘，看板「采集」子页可回溯。

---

## 8. 总线接入（批次81 新增）

### 8.1 薄委托入口（`citychrome/system_bus.go`）

```go
// BusTrigger 由 main.go 装配层注入(真正触发一次 mission 的逻辑)。
var BusTrigger func(ctx context.Context, category string) (string, error)

// SystemBusPayload 是 LsmAgentGame-City-System 经 citybus 调用本 Agent 的
// 薄委托入口。category 为空 = 全部类目。
func SystemBusPayload(ctx context.Context, category string) (string, error)
```

- `BusTrigger == nil` ⇒ 返回可读说明「采集服务未接入总线（BusTrigger 未注入）」，
  **不返回 error**（避免 System Agent 侧误判为故障）。
- 触发逻辑**不在本包反向依赖** `ws` / `main` / `citysystem` / `game/virtual_city`。

### 8.2 Service 侧触发实现（复用既有 API）

```go
func (s *Service) SystemBusTrigger(ctx context.Context, category string) (string, error)
```

复用 `Service.Refresh`（既有手动刷新入口）入队一次 mission，把结果格式成人读摘要：

| 情形 | 返回 |
|---|---|
| 服务未启动 | 「采集服务未启动，无法触发采集。」 |
| 未启用 | 「采集数据通道未启用（city_chrome.enabled=false）…」 |
| 类目非法 | 「未知采集类目 %q（可选：a_share/hk/us/metal/futures/crypto/macro/news；留空=自动选择）。」 |
| 限流 / 入队失败 | 「未能触发「x」采集：[35054] …」 |
| 成功入队 | 「已触发「x」采集任务：采集 Agent 将异步访问真实站点，结果产出后可在实时看板查看。」 |

> 所有**可预期运营态都返回可读文本 + nil error**（是状态而非故障）；
> 仅未来真正意外的内部错误才返回 error。

### 8.3 main.go 装配（集成 SubAgent 收口）

`main.go` 已有 TODO 桩（`register City-Chrome bus handler failed` 上方），
收口需两步：

```go
// ① 在 cityChromeSvc 创建后注入触发器（一次性,启动期）
citychrome.BusTrigger = cityChromeSvc.SystemBusTrigger

// ② 把桩 handler 体替换为薄委托（msg.Payload 形如 {"category":"news"}）
if err := citybus.Default().Register(string(agentroot.AgentClassCityChrome),
	func(ctx context.Context, msg citybus.Message) (any, error) {
		category := ""
		if m, ok := msg.Payload.(map[string]any); ok {
			category, _ = m["category"].(string)
		}
		return citychrome.SystemBusPayload(ctx, category)
	}); err != nil {
	logger.L().Warn("register City-Chrome bus handler failed", zap.Error(err))
}
```

### 8.4 System Agent 侧调用（`citysystem/dispatch.go`）

工具 `collect_realworld_data{category?}` →
`hub.Send(ctx, roomID, City-System, City-Chrome, "collect", {"category": category})`
→ 返回文本回灌给模型。System Agent **不直接 import** `citychrome`。

---

## 9. 纪律（角色段 `roleText`，逐字稳定）

1. 只用工具操作浏览器，**绝不臆造数据**；每条数据必须来自页面真实 DOM 文本。
2. **禁止登录、禁止提交表单、禁止绕过验证码**，只读公开页面。
3. 遇拦截/验证码/超时 → 换备选站点（最多 3 次），仍失败则终止。
4. 同站点访问保持克制（调度器 ≥10s）；不点广告、不触发下载。
5. `payload.source` 只填站点名（≤40 rune 截断兜底），不带链接说明。
6. 优先 `web_eval` / `web_read(parse)` 减少往返；`web_search` 仅作发现兜底。
7. 完成后必须调 `web_done`（唯一使命出口）。

---

## 10. 文件清单（本批新增）

| 文件 | 行数 | 职责 |
|---|---|---|
| `ServerGo/citychrome/system_bus.go` | 79 | `BusTrigger` + `SystemBusPayload` + `Service.SystemBusTrigger` |
| `ServerGo/citychrome/system_bus_test.go` | 150 | 薄委托三态 + Service 触发分支单测 |

> 未改动 `Service` 核心调度逻辑（`Refresh` / `Scheduler` 逐字未动）。

---

## 11. 测试

`ServerGo/citychrome/system_bus_test.go`：

| 用例 | 断言 |
|---|---|
| `TestSystemBusPayload_NilTrigger` | `BusTrigger == nil` ⇒ 可读说明 + nil error |
| `TestSystemBusPayload_ErrorPassthrough` | `BusTrigger` 返回 error ⇒ `errors.Is` 透传 |
| `TestSystemBusPayload_NormalSummary` | category 原样下传 + 摘要返回 |
| `TestServiceSystemBusTrigger_NilService` | 未启动 ⇒ 可读文本 |
| `TestServiceSystemBusTrigger_Disabled` | 未启用 ⇒ 可读文本 |
| `TestServiceSystemBusTrigger_UnknownCategory` | 非法类目 ⇒ 可读文本且不入队 |
| `TestServiceSystemBusTrigger_Accepted` | 正常入队 ⇒ 摘要 + 队列长度 1 |
| `TestServiceSystemBusTrigger_RateLimited` | 60s 限流 ⇒ 可读文本 + nil error |

**验证命令**

```bash
cd ServerGo
gofmt -l citychrome/system_bus.go citychrome/system_bus_test.go   # 空 = 干净
go build ./...
go test ./citychrome/... -count=1
go test -race ./citychrome/ -run 'SystemBus' -count=1
```
