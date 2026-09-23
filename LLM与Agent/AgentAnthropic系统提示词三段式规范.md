# Anthropic 三段式 system 提示词规范（全部 Agent）

> 规则标记：**CLAUDE.md §14.3**。本文件是该规则的唯一事实来源。
> 适用范围：本仓库**所有**经由 `llm.LLMProvider` 调用大模型的 Agent —— 狼人杀玩家 / 法官 / 解说、
> 辩论玩家 / 裁判 / 解说、德州扑克玩家、虚拟城市居民、记忆迭代器、上下文压缩器，以及后续新增的任何 Agent。

## 1. 目标

每个出站 LLM 请求的 `system[]` **最前面固定三段**，与 Claude Code 的 Anthropic 协议一致：

| 序 | 名称 | 职责 | 是否影响模型行为 |
|----|------|------|------------------|
| ① | 计费元数据头 | 承载 SDK 版本 / 调用入口 / 子代理标识等链路信息，供上游计费与调用追踪 | 否（纯追踪字符串） |
| ② | 基础身份声明 | 声明代理的基础身份属性；带 `ephemeral` 缓存控制，实现提示片段复用 | 是（弱） |
| ③ | 核心行为规则 | Claude Code CLI Agent 的完整指令集：工具使用边界 / 任务执行准则 / 输出规范 / 权限约束 | 是（强） |

设计要点：**三段是协议层常量，不是某个 Agent 的私有 prompt**。因此它们不散落在各 Agent 的 prompt builder
中，而是由 Provider 在出站前统一注入 —— 这样「全部 Agent 都升级」由**结构**保证，而不是靠 17 个调用点逐个改对。

## 2. 权威文本与 wire 形状

文本常量位于 `ServerGo/llm/sysprompt/texts.go`（**逐字节稳定**，禁止改写）：

```
system[0] = {"type":"text","text":"x-anthropic-billing-header: cc_version=2.1.278.acd; cc_entrypoint=server; cc_is_subagent=true; cc_agent_name=<AgentClassName>;"}
system[1] = {"type":"text","text":"You are a Claude agent, built on Anthropic's Claude Agent SDK.","cache_control":{"type":"ephemeral"}}
system[2] = {"type":"text","text":"<核心行为规则全文>","cache_control":{"type":"ephemeral"}}
system[3..n] = Agent 自有块（原样保留，顺序不变）
```

- **权威来源**：Claude Code 实际下发的协议用例 `CluadeCode的Anthropic协议-RequestBody-数据用例01.json`（项目根目录）。
- 与用例的唯一差异：用例末尾的动态行 `<total_tokens>N tokens left</total_tokens>` **不收编** ——
  那是 Claude Code 会话级 token 池的实时余量；本仓库 Agent 无会话级池（预算由上下文字节预算 + 阶段超时管理），
  写死假余量会污染模型判断。
- 常量：`CCVersion = "2.1.278.acd"`、`EntrypointServer = "server"`（Claude Code 自身为 `cli`，本仓库全部 Agent
  由 Go 服务端进程内驱动，故统一声明 `server`）。

## 3. 注入点（单一收口）

```text
Agent 代码（prompt builder）──► LLMRequest{System: 自有块}
                                      │
                  Provider 序列化前 ──┼──► sysprompt.EnsureHead(req.System, req.AgentClassName)
                                      │
                                      ▼
                出站 wire:  [① ② ③] + Agent 自有块
```

| Provider | 注入位置 | 协议落点 |
|----------|----------|----------|
| `ServerGo/llm/anthropic/anthropic.go` | `Chat()` / `ChatStream()` 构造 `anthropicRequest` 处 | `system[]` 数组前三项 |
| `ServerGo/llm/openai/convert.go` | `buildRequest()` 的 `system[] → system message` 转换处 | 单条 `role:"system"` 消息的**前缀**（`\n\n` 拼接） |

**硬约束**：

- Agent 侧代码**禁止**手工拼接三段（新增调用点一律不得复制 `x-anthropic-billing-header` 字面量）；
  由 `sysprompt` 包外零字面量 lint 守门（`head_lint_test.go`）。
- 新增 Provider（新协议）时**必须**同样调用 `sysprompt.EnsureHead`，否则走该协议的 Agent 会漏掉三段；
  `head_lint_test.go` 的注入点清单需同步登记。

## 4. 字段归一化规则

| 字段 | 规则 |
|------|------|
| `cc_version` | 常量 `CCVersion`；仅作追踪字符串，上游不做版本校验 |
| `cc_entrypoint` | `server`（本仓库全部 Agent 为服务端进程内驱动）；预留 `cli` |
| `cc_is_subagent` | `req.AgentClassName != ""` ⇒ `true`（该调用由服务端引擎以 task 形态驱动，语义等价 Claude Code subagent）；健康探针 / 后台自检等无 AgentClassName 的调用 ⇒ `false` |
| `cc_agent_name` | 仅 `AgentClassName` 非空时输出，取 `ServerGo/agent/class_names.go` 常量，便于上游区分同一局内的不同 Agent |
| 键序 | `cc_version` → `cc_entrypoint` → `cc_is_subagent` → `cc_agent_name`，**禁止重排** |
| 幂等 | `EnsureHead` 检测首位块是否以 `x-anthropic-billing-header:` 开头；已带头则原样返回（不重复注入） |
| text 块收敛 | 三段均为纯 `text` 块，不得携带 `id` / `name` / `input` / `tool_use_id` / `is_error`（§14.1 wire 收敛规则同样适用于 `system[]`） |
| HTTP 头同源 | 出站请求头 `x-anthropic-billing-header` 与 ① 段共用 `sysprompt.BillingHeaderText`（main.go 注入 registry → provider），避免两处口径漂移 |

## 5. 缓存与成本

- ②③ 段带 `cache_control: {"type":"ephemeral"}`：三段文本全局常量 ⇒ 跨 Agent、跨局、跨轮次字节稳定，
  作为 prompt cache 前缀可跨 Agent 复用，把"每 Agent 每轮多付 2.5KB"压到近乎零边际成本。
- ① 段**不带**缓存控制：其 `cc_agent_name` 逐 Agent 类变化，作前缀会击穿命中。
- Anthropic 每请求最多 4 个 cache 断点：三段占 2 个，Agent 自有块（如狼人杀玩家 14KB 规则段）仍可保留 1 个，
  合计 ≤ 3，不触碰上限。
- Agent 自有块文本**未改动**：本规范只做前缀追加，不重排、不合并、不裁剪既有 prompt。

## 6. 字节预算（隐形开销）

三段约 **2.5KB**，属 Provider 侧注入、Agent 侧不可见的开销。历史上（§20260810-14）正是"只算 messages、
漏算 system/tools"导致小窗口模型（DouBao 128K-256K）实际超限而预算未触发。因此：

- Agent 侧估算 payload 字节时须计入三段：`sysprompt.HeadBytes(agentClass)`；
  参考实现 `ServerGo/agent/wwplayer/memory.go` 的 `approxSystemToolsBytes`。

## 7. 代码地图

| 文件 | 角色 |
|------|------|
| `ServerGo/llm/sysprompt/texts.go` | 三段权威文本 + `BillingHeaderText()` |
| `ServerGo/llm/sysprompt/sysprompt.go` | `Head()` / `EnsureHead()` / `HasHead()` / `HeadBytes()` |
| `ServerGo/llm/anthropic/anthropic.go` | 协议注入点（非流式 + 流式） |
| `ServerGo/llm/openai/convert.go` | OpenAI 协议注入点（转换为 system message 前缀） |
| `ServerGo/main.go` | HTTP 头 `x-anthropic-billing-header` 同源注入 |
| `ServerGo/agent/wwplayer/memory.go` | 字节预算计入三段 |

## 8. 验证

| 测试 | 断言 |
|------|------|
| `llm/sysprompt/sysprompt_test.go` | 三段文本锚点、wire 键收敛、头顺序、幂等、`cc_is_subagent` 判定；`TestTextsAreByteStable` 锁定关键文本 |
| `llm/anthropic/sysprompt_head_test.go` | **最终出站请求体**（httptest 上游抓捕）在流式/非流式两条路径上都带三段；调用方自有块在其后且字节不变；已带头不重复 |
| `llm/openai/sysprompt_head_test.go` | OpenAI 协议下三段拼进 `role:"system"` 消息前缀，顺序正确，幂等 |
| `llm/sysprompt/head_lint_test.go` | ① 计费头文本仅 `sysprompt` 拥有（Agent/Provider 侧零字面量）；② 两个 Provider 都调用 `EnsureHead` |

```bash
cd ServerGo && go test ./llm/... && go test ./...
```

## 9. 变更纪律

1. **三段文本逐字节稳定**：它是全体 Agent 的共享缓存前缀，任何改写都会让全量 Agent cache 命中失效。
   改文本必须同步更新 `sysprompt_test.go` 的锚点断言，并在本文件登记新版本号（`cc_version`）。
2. **禁止 Agent 侧拼接**：新 Agent 无需任何改动即自动获得三段；如需自定义 `cc_entrypoint`，
   必须在设计文档中说明并由 `sysprompt` 提供参数，不得在 Agent 内复制字面量。
3. **禁止重排 `system[]`**：三段永远在最前，Agent 自有块永远在其后。
