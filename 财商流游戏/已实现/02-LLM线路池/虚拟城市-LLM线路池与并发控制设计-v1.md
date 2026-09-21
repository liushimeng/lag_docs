# 虚拟城市 — LLM 线路池与并发控制设计 v1

> 状态：实现契约（2026-09-21）。上游总纲见 01；本篇定义 provider 表、LinePool、
> 管理 API 与 `agent_name` 废弃路径的精确契约。

## 1. 语义定义

- **线路（line）**：一条可并行发起 LLM 调用的通道。provider 行的 `concurrency_lines`
  表示该模型可同时占用的线路数（1~64，默认 1）。
- **总线路数 M** = Σ `enabled=true` 行的 `concurrency_lines`。
  **M 即 Agent 调用大模型的全局并发上限**——同一时刻全进程最多 M 个在途 LLM 请求。
- **Agent Name 废弃**：provider 行不再承载「Agent」语义；管理页不出现该字段。
  列 `agent_name` 物理保留（唯一索引 + 存量展示面依赖），值由后端自动派生。

## 2. 数据库变更（`models/t_lsm_game_llm_provider.go`）

```go
// 新增字段（AutoMigrate 自动补列，存量行默认 1）
ConcurrencyLines int `gorm:"column:concurrency_lines;type:int;not null;default:1" json:"concurrency_lines"`
```

- `AgentName` 列保留原状（uniqueIndex 不动），注释标注 `// Deprecated: 管理语义已废弃，仅作兼容展示；空值由 API 派生`。
- 校验（API 层执行，DB 层只存）：`concurrency_lines ∈ [1,64]`。

## 3. LinePool（`ServerGo/llm/linepool.go` 新文件）

### 3.1 对外契约

```go
// ErrAllLinesBusy 在 Acquire 等待期间 ctx 被取消/超时返回；调用方应走规则兜底。
var ErrAllLinesBusy = errors.New("llm: all lines busy")

type LineLease struct {
    ModelKey string              // 服务本次调用的模型 key
    Info     types.ModelInfo     // key-free 投影（含 concurrency_lines）
    Provider types.LLMProvider   // 已构造的 provider（含 endpoint 覆盖）
    APIKey   string              // 解密后明文 key（仅本次调用内传递）
    release  func()
}
func (l *LineLease) Release()    // 幂等；归还线路令牌

type LinePool struct { /* ... */ }
func (p *LinePool) Total() int                // M；无可用行返回 0
func (p *LinePool) Acquire(ctx context.Context) (*LineLease, error)
func (p *LinePool) Stats() []LineStat         // 管理页/健康检查用
type LineStat struct {
    ModelKey string `json:"model_key"`
    Lines    int    `json:"lines"`
    InFlight int    `json:"in_flight"`
}
```

### 3.2 实现：预填充令牌通道

- 内部 `tokens chan lineToken`，容量 M，**初始化时按各行 lines 数量预填充**
  `lineToken{modelKey, prov}`（即 A×3、B×2 → AABAB 轮询序），天然加权轮询。
- `Acquire`：`select { case <-ctx.Done(): return ErrAllLinesBusy; case t := <-tokens: ... }`。
- `Release`：`tokens <- t`（容量必有空位，因令牌被本租约持有）。
- 令牌内缓存已构造的 provider 实例（与 `registry.Get` 的懒建路径复用同一构造函数），
  key 为解密明文，仅在租约生命周期内存在。
- **Reload 语义**：`Registry.Reload()` 重建行表后**新建** LinePool 并原子替换指针；
  在途租约持旧池令牌，Release 归还旧池（旧池随引用归零被 GC），不泄漏、不阻塞。

### 3.3 Registry 集成（`llm/registry.go`）

- `registeredProvider` 增加 `lines int`（DB 行 `concurrency_lines` clamp [1,64]）。
- 新增 `func (r *Registry) LinePool() *LinePool`（原子读，Reload 后自动指向新池）。
- 新增 `func (r *Registry) TotalLines() int` = `LinePool().Total()`。
- `HealthCheck`/`List` 等既有方法不动；`Get(modelKey)` 保留原语义（固定模型场景）。
- `llm/defaults.go`：`DefaultProviderSeed` 每行补 `ConcurrencyLines: 1`。

## 4. 管理 API 变更（`api/model_admin_api.go`）

### 4.1 Create（`POST /api/admin/llm/providers`）

| 字段 | 变更 |
|---|---|
| `agent_name` | **可选**（原必填）。空串/缺省 → 派生：初值 = `model`；若与既有行冲突则 `model-2`、`model-3`… 直到唯一 |
| `concurrency_lines` | 新增，可选，默认 1，范围 [1,64]，越界 400（沿用 invalid-param 错误码） |

### 4.2 Update（`PUT /api/admin/llm/providers/:id`）

- 新增 `concurrency_lines *int`（nil=不改，非 nil 时 clamp 校验同上）。
- `agent_name *string` 保留可改（兼容），但前端不再提交。

### 4.3 providerView（响应投影）

- 新增 `"concurrency_lines"`；`agent_name` 继续下发（兼容存量 UI），文档标注 deprecated。

## 5. 模型列表 API（`api/llm_api.go`）

`GET /api/llm/models` 条目（需登录）：

```json
[{ "agent_name": "DeepSeek", "model": "DeepSeek", "provider_type": "anthropic",
   "concurrency_lines": 3 }]
```

- `concurrency_lines` 新增；`agent_name` 保留（狼人杀/雷达等展示面仍用，标 deprecated）。
- 前端「总线路数」= Σ `concurrency_lines`（仅 enabled 行参与——该接口本就只列可用模型）。

## 6. 管理页 UI（`ClientWeb/src/pages/ModelAdminPage.tsx`）

| 项 | 契约 |
|---|---|
| 列表列 | 删除「Agent 名称」列；「模型」列改为链接至详情；新增「线路数」列（`concurrency_lines`） |
| 头部摘要 | 新增提示条：`总线路数 N = Agent 调用大模型的并发数`（N=Σ启用行线路） |
| 表单 | 删除 agentName 输入与校验；新增 concurrency_lines 数字输入（1-64，默认 1，必填校验） |
| 编辑 diff | `buildUpdateBody` 纳入 concurrency_lines 变更检测 |
| 详情页 | `ModelDetailPage.tsx`：agent_name 行替换为「线路数」行 |

## 7. i18n（三语同步）

- 新增：`modelAdmin.colConcurrencyLines`（线路数）、`modelAdmin.fieldConcurrencyLines`、
  `modelAdmin.totalLinesHint`（总线路数 {n} = Agent 调用大模型的并发数）、
  `modelAdmin.linesRangeError`（线路数须在 1-64 之间）。
- 删除：`modelAdmin.colAgentName`、`modelAdmin.fieldAgentName`（zh-CN/en/ja/types 四处同步）。

## 8. 消费方接入（财商流；详见 03/04）

- 焦点座位 Agent（`ModelKey==""`）与城市之声：`LinePool.Acquire(ctx)` → 用 `lease.Provider`
  发起调用（沿用 `ChatStreamAccumulate` 流式 + §197 字节刷新）→ `defer lease.Release()`。
- Acquire 返回 `ErrAllLinesBusy`：等价于一次 LLM 失败，走既有超时兜底
  （焦点座位强制 `submit_month`；城市之声丢弃本条，下月再抽）。
- 房间级信号量容量 = `max(LinePool.Total(), 1)`，池不可用时回退 `cfg.Wealth.AgentConcurrency`。

## 9. 测试要求

| 用例 | 断言 |
|---|---|
| `llm/linepool_test.go`：容量上限 | M=3 时 4 个并发 Acquire，恰 1 个阻塞；Release 后放行 |
| 加权轮询 | A(lines=2)/B(lines=1)，6 次 Acquire → A 4 次 B 2 次 |
| ctx 取消 | 阻塞中 ctx 超时 → `ErrAllLinesBusy`，令牌数不变 |
| Reload | 重建后 Total 反映新行表；旧租约 Release 不 panic |
| `api/model_admin_api_test.go` | 无 agent_name 创建成功且派生唯一；concurrency_lines 越界 400；更新生效 |
| `/api/llm/models` | 条目含 concurrency_lines，总数正确 |
