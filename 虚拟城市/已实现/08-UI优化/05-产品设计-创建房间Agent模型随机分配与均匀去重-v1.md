# 虚拟城市 — 创建房间 Agent 模型随机分配与均匀去重（v1）

> 2026-09-19 用户需求响应。
> 需求原文：创建房间时，分配 Agent（AI 玩家数量 > 1，例如 AI 1 – AI 7）应该是**随机分配**使用 Agent，
> 尽量不要使用相同的 AgentName；遵循**均匀随机分配**原则，尽量不重复。

## 1. 需求拆解

| # | 需求点 | 含义 |
|---|--------|------|
| R1 | 随机分配 | 每次创建房间（打开建房弹窗），各 AI 座位的模型（`agent_name`）组合应当**随机变化**，而不是固定的"AI1 永远=模型列表第 1 个" |
| R2 | 尽量不重复 | 模型数 ≥ 座位数时，各 AI 座位模型**互不相同**（零重复） |
| R3 | 均匀原则 | 模型数 < 座位数（如 8 模型 12 座位）时，每个模型被使用的次数差 ≤ 1（floor/ceil 均匀），且相邻座位尽量不同模型 |
| R4 | 可重摇 | 用户可以一键「🎲 重新分配」重新随机（对齐狼人杀/德州建房弹窗已有体验） |

## 2. 现状分析（改动前）

### 2.1 前端 `ClientWeb/src/components/wealth/WealthCreateRoomModal.tsx`

```tsx
// L103-136 useEffect([open]) 内，listModels() 成功后：
setSeatModels((prev) =>
  prev.length === 0                       // ← 仅首次初始化
    ? Array.from({ length: WEALTH_MAX_SEATS }, (_, i) => ms[i % ms.length]?.model ?? '')
    : prev,                               // ← 之后永远复用旧值
);
```

```tsx
// L141-149 useEffect([models]) 槽位补齐：
next.push(models.length > 0 ? models[next.length % models.length]?.model ?? '' : '');
```

**三个缺陷**：

| 缺陷 | 位置 | 后果 |
|------|------|------|
| D1 固定顺序轮询 | L116 / L145 的 `i % models.length` | 座位-模型映射是**确定性的**：模型列表顺序不变时，AI1 永远是第 1 个模型、AI2 永远是第 2 个……多次建房看到的组合完全一样，用户感知"没有随机" |
| D2 仅首次初始化 | L114 的 `prev.length === 0` 守卫 | 关闭弹窗再打开，仍复用上一次的分配，进一步固化 |
| D3 无重摇入口 | 座位区只有 per-seat `<select>` | 与狼人杀（`ww-create-modal__reshuffle`）/德州（`reshuffle`）弹窗体验不一致，用户无法主动重新随机 |

> 说明：当模型数 ≥ 座位数时，`i % length` 轮询确实做到了"零重复"（R2 恰好满足），
> 但 R1（随机）/R3（模型不足时的均匀性）/R4（重摇）均不满足。

### 2.2 后端 `ServerGo/service/room_service_crud.go`（已满足，无需改动）

`CreateRoomWithAgents`（L384-418）对**所有游戏类型**（含 wealth，`maxAgentSeats = 12`，L234）已有去重兜底：

- 触发条件：`len(agentSeats) > 1`；
- `alternateModelsLocked(seats)`（`room_service.go:545`）返回**排除已占用模型后 Fisher-Yates 洗牌**的候选池；
- 第一个出现的每个 model_key 保留（用户显式选择生效），**重复项**依次从洗牌池取未被占用的模型改写；池耗尽时轮询兜底；
- 这是防止客户端退化（CDP / eval_js 绕过 React state）的最后防线——**永远不信客户端**。

**结论：后端语义正确且通用，本次改动仅限前端。**

### 2.3 参考实现（狼人杀弹窗）

`ClientWeb/src/components/werewolf/RoomCreateModal.tsx`：
- L179-185 Fisher-Yates 洗牌 model keys → 新座位取未使用模型 → 用尽后 `fillCursor % shuffled.length` 轮询；
- L108 `shuffleNonce` 计数器 + L528-539「🎲 重新分配」按钮（清空 seats + nonce+1 触发整体重摇）；
- i18n：`werewolf.createModal.reshuffle` / `texasholdem.createModal.reshuffle`（zh-CN / en / ja 三语已有）。

## 3. 解决方案（前端 only）

### 3.1 核心：均匀随机分配纯函数

在 `WealthCreateRoomModal.tsx` 内新增文件私有纯函数（引用方 100% 属于 wealth，遵守 CLAUDE.md §2.1 硬约束 2）：

```ts
/** Fisher-Yates 洗牌（返回新数组，不改动入参）。 */
function shuffle<T>(arr: readonly T[]): T[];

/**
 * 均匀随机分配座位模型：ceil(slots / models.length) 轮洗牌拼接后截断 slots 个。
 * - models.length >= slots → 等价于随机排列取前 slots 个，座位间零重复（R2）；
 * - models.length <  slots  → 每个模型出现 floor(slots/len) 或 ceil(slots/len) 次，
 *   差 ≤ 1（R3 均匀）；再过一遍"相邻同模型修复"（向后找最近异值交换，
 *   len >= 2 时保证相邻座位不同模型，不改变各模型总次数）；
 * - 每次调用独立随机（R1）。
 */
function shuffledRoundRobinModels(models: ModelInfo[], slots: number): string[];
```

要点：
- **多轮洗牌拼接**（而非单轮 + 轮询）：保证任何前缀中每个模型出现次数差 ≤ 1；
- **相邻修复**：拼接边界可能出现 `...A | A...`，交换后面最近的异值元素消除相邻重复；
  交换只移动位置不改变计数，均匀性不变；`len == 1` 时无模型可换，保持原样（物理上限）。

### 3.2 接线（4 处）

| # | 位置 | 改动 |
|---|------|------|
| W1 | `useEffect([open])` 的 `listModels().then` | 去掉 `prev.length === 0` 守卫，改为**每次弹窗打开且模型加载成功都** `setSeatModels(shuffledRoundRobinModels(ms, WEALTH_MAX_SEATS))`（R1：每次建房重新随机；用户上次的手动选择随之重置——自动分配值本来就是默认值，语义合理） |
| W2 | `useEffect([models])` 槽位补齐 | 简化为纯长度对齐（slice 到 `WEALTH_MAX_SEATS`）；补齐兜底统一走 W1 的随机分配（模型列表仅由 `listModels()` 写入，与 seatModels 同源同步，确定性轮询补齐不再需要） |
| W3 | 座位区标题条 | 新增「🎲 重新分配」按钮：`setSeatModels(shuffledRoundRobinModels(models, WEALTH_MAX_SEATS))`；`data-testid="wealth-create-reshuffle"`；样式复用 `wealth-tier-btn wealth-tier-btn--sm`（不新增 CSS，规避 §26.3 JSX 拼接零 CSS 规则风险） |
| W4 | i18n | `wealth.create.reshuffle` 三语（zh-CN `🎲 重新分配` / en `🎲 Reshuffle` / ja `🎲 再配分`，对齐 `texasholdem.createModal.reshuffle` 文案） |

附带：`agentCount > 0 && models.length > 0 && models.length < agentCount` 时在座位区下方显示
hint「模型数（N）少于座位数（M），将均匀复用」——复用现有 `wealth-create-form__hint` 样式，
提示用户重复是物理约束而非 bug（后端 `alternateModelsLocked` 仍会兜底去重）。

### 3.3 不改动的部分（明确边界）

- **后端零改动**：`CreateRoomWithAgents` / `alternateModelsLocked` / `ValidateAgentSeats` 保持原样；
- per-seat `<select>` 手动改选能力保留（用户显式选择优先于自动分配，与狼人杀语义一致，
  后端"第一次出现的 key 保留"正好尊重该语义）；
- `agentSeats` 提交结构（seat 0..N-1 + model_key）不变。

### 3.4 分配语义总览（改后）

```
打开弹窗 ──listModels 成功──> shuffledRoundRobinModels(models, 12) → 12 槽位随机均匀
   │                              │
   │ agentCount 档位(0..12)       └─ 渲染时截取前 agentCount 个座位
   │
   ├─ 「🎲 重新分配」 ──> 重新 shuffledRoundRobinModels 整体重摇
   ├─ per-seat 手动改选 ──> 覆盖该槽位（提交后后端尊重首个出现的 key）
   └─ 提交 agent_seats ──> 后端 len>1 时 alternateModelsLocked 对重复项随机改写（最后防线）
```

## 4. 验收标准

| # | 场景 | 预期 |
|---|------|------|
| V1 | 模型数 ≥ 8、Agent 数 = 7 | 打开弹窗：7 个座位模型互不相同；关闭重开：组合与上次**大概率不同**（随机排列） |
| V2 | 模型数 = 8、Agent 数 = 12 | 每个模型恰好出现 1 或 2 次（12 = 8+4：4 个模型 2 次、4 个模型 1 次）；相邻座位不同模型 |
| V3 | 点击「🎲 重新分配」 | 12 槽位整体重摇（agentCount 内的座位组合变化） |
| V4 | 手动改某座位 → 提交 | 该座位按用户选择提交；其余保持随机分配值 |
| V5 | 客户端被绕过（如全座位同 key 直发 API） | 后端 `alternateModelsLocked` 随机改写重复项（现有行为，回归确认） |
| V6 | 构建门禁 | `tsc --noEmit` 通过 + `npm run build` 成功 |

## 5. 涉及文件清单

| 文件 | 改动类型 |
|------|----------|
| `ClientWeb/src/components/wealth/WealthCreateRoomModal.tsx` | 新增 `shuffle` / `shuffledRoundRobinModels` 纯函数；W1/W2/W3 接线；头注释更新 |
| `ClientWeb/src/i18n/locales/zh-CN.ts` / `en.ts` / `ja.ts` | W4 新增 `wealth.create.reshuffle` 键 |
| 本文档 | 新增 |

## 6. 关联

- `lag_docs/狼人杀-Agent与系统/狼人杀Agent设计.md` §12.1（AI 玩家模型随机分配 — 后端 Fisher-Yates 兜底的原始出处）
- `lag_docs/虚拟城市/已实现/02-架构设计/虚拟城市-WS与HTTP协议契约-v1.md`（agent_seats 请求结构）
- 狼人杀参考实现：`ClientWeb/src/components/werewolf/RoomCreateModal.tsx:108,179-185,528-539`
