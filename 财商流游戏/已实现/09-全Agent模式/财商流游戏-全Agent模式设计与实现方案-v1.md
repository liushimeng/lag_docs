# 财商流游戏 — 全 Agent 模式设计与实现方案 v1

> **定位**：财商流游戏（game_kind=`wealth`）为**全 Agent 运行**的游戏，人类用户**不能参与**对局，
> 仅可以**观战者**身份观看。本文件定义全 Agent 模式的完整设计方案，涵盖后端、前端、
> API 协议、房间创建、加入限制等全链路改动。
>
> **创建日期**：2026-09-19 ｜ **版本**：v1 ｜ **状态**：实现契约

---

## 目录

1. [背景与目标](#1-背景与目标)
2. [核心设计原则](#2-核心设计原则)
3. [后端改动](#3-后端改动)
4. [前端改动](#4-前端改动)
5. [API 协议变更](#5-api-协议变更)
6. [房间创建流程](#6-房间创建流程)
7. [加入限制逻辑](#7-加入限制逻辑)
8. [观战者流程](#8-观战者流程)
9. [错误码定义](#9-错误码定义)
10. [测试验证方案](#10-测试验证方案)

---

## 1. 背景与目标

### 1.1 背景

财商流游戏是一个**教育类财商沙盘模拟**，核心玩法是多个 AI Agent 在模拟经济环境中
进行人生决策（投资、消费、学习、社交等），通过月度 tick 机制推进游戏。

### 1.2 目标

- **全 Agent 运行**：房间内所有参与者必须是 Agent（AI 玩家），人类不能占据游戏座位
- **人类仅观战**：人类用户可以进入房间观看 Agent 对局，但不能参与决策
- **自动开局**：房间创建后自动填充 Agent 并自动开始，无需人类操作
- **模型多样化**：多个 Agent 尽量使用不同的 LLM 模型，提升对局多样性

### 1.3 设计约束

- 保持现有房间架构不变（WealthRoom + Manager + Agent Runner）
- 保持观战者架构不变（Spectator 机制）
- 保持 WebSocket 广播机制不变
- 全 Agent 模式通过房间级标志位控制，不影响其他游戏

---

## 2. 核心设计原则

| 原则 | 说明 |
|------|------|
| **单一事实来源** | 全 Agent 标志位存储在 `WealthRoom.FullAgentMode` |
| **后端强制** | 人类加入请求在后端被硬性拒绝，前端仅做辅助展示 |
| **向后兼容** | 现有接口保持兼容，新增字段均为可选 |
| **最小侵入** | 仅修改 wealth 相关代码，不影响其他游戏 |

---

## 3. 后端改动

### 3.1 WealthRoom 新增字段

**文件**：`ServerGo/game/wealth/room.go`

```go
type WealthRoom struct {
    // ... 现有字段 ...
    
    // FullAgentMode 标记该房间为全 Agent 模式（人类不能参与对局）。
    // 2026-09-19 §全Agent模式 新增：创建时由 agent_seats 满 MinSeats 自动置位，
    // 或前端显式请求 full_agent=true 置位。
    FullAgentMode bool
}
```

### 3.2 JoinGame 方法修改

**文件**：`ServerGo/game/wealth/room.go`

```go
// JoinGame 加入房间（人类玩家）。全 Agent 模式下拒绝人类加入。
func (r *WealthRoom) JoinGame(userID string, nickname string) (seat int, full bool, err *errcode.Error) {
    r.mu.Lock()
    defer r.mu.Unlock()
    
    // 全 Agent 模式：拒绝人类加入
    if r.FullAgentMode {
        return -1, false, errcode.Code(errcode.ErrWealthFullAgentRoom)
    }
    
    // ... 现有逻辑 ...
}
```

### 3.3 CreateRoomWithAgents 中的自动置位

**文件**：`ServerGo/service/room_service_crud.go`

在 `CreateRoomWithAgents` 函数中，当 `agent_seats` 数量 >= MinSeats(10) 时，
自动将房间标记为全 Agent 模式。

### 3.4 GameJoiner 接口扩展

**文件**：`ServerGo/service/room_service.go`

```go
type GameJoiner interface {
    SyncSeat(gameKind, roomID, userID string) (started bool, err *errcode.Error)
    // SetFullAgentMode 设置房间的全 Agent 模式标志（仅 wealth 生效）。
    SetFullAgentMode(gameKind, roomID string, enabled bool) *errcode.Error
    // IsFullAgentRoom 查询房间是否为全 Agent 模式。
    IsFullAgentRoom(roomID string) (bool, *errcode.Error)
    // ... 其他方法 ...
}
```

### 3.5 WS 层实现 SetFullAgentMode / IsFullAgentRoom

**文件**：`ServerGo/ws/game_service.go`

```go
func (s *GameService) SetFullAgentMode(gameKind, roomID string, enabled bool) *errcode.Error {
    if gameKind != "wealth" { return nil }
    if s.wealthMgr == nil { return nil }
    r := s.wealthMgr.Get(roomID)
    if r == nil { return nil }
    r.SetFullAgentMode(enabled)
    return nil
}

func (s *GameService) IsFullAgentRoom(roomID string) (bool, *errcode.Error) {
    if s.wealthMgr == nil { return false, nil }
    r := s.wealthMgr.Get(roomID)
    if r == nil { return false, nil }
    return r.IsFullAgentMode(), nil
}
```

### 3.6 JoinRoom 中的人类加入拦截

**文件**：`ServerGo/service/room_service_crud.go`

在 `JoinRoom` 函数中，查询到房间为 wealth 全 Agent 模式时，拒绝人类加入。

---

## 4. 前端改动

### 4.1 创建房间弹窗（WealthCreateRoomModal.tsx）

- **默认全 Agent 模式开启**：所有座位自动填充 Agent
- **Agent 数量固定为 12**（全 Agent 模式）
- **模型选择区域**：每个座位独立选择 LLM 模型

### 4.2 大厅房间列表

- 全 Agent 房间显示「🤖 全 Agent」标签
- 人类点击「加入」按钮时，自动跳转观战

### 4.3 房间详情页

- 全 Agent 模式下，「加入游戏」按钮替换为「观战」按钮

### 4.4 对局页面

- 全 Agent 模式下，隐藏所有人类操作按钮（ActionPanel）
- 显示「观战模式」标识

---

## 5. API 协议变更

### 5.1 创建房间 POST /api/games/wealth/rooms

**新增请求字段**：`full_agent`（可选，默认 true）
**新增响应字段**：`full_agent`（标记是否为全 Agent 模式）

### 5.2 加入房间 POST /api/rooms/:id/join

**新增错误码**：35013 — 全 Agent 房间拒绝人类加入

### 5.3 房间详情 GET /api/rooms/:id

**响应新增字段**：`full_agent`

---

## 6. 房间创建流程

```
用户点击「创建房间」
  → 弹出创建房间弹窗（默认全 Agent 模式）
  → 选择模型、月节拍速度、卡池
  → 点击「创建」
  → POST /api/games/wealth/rooms { full_agent: true, agent_seats: [...] }
  → 后端 CreateRoomWithAgents：
      1. 创建 DB 房间记录（capacity=12）
      2. 注册 bot 座位到 WealthRoom
      3. 标记 FullAgentMode=true
      4. bot 数量 >= MinSeats → 自动 Start
  → 响应返回 { full_agent: true, ... }
  → 前端自动以观战者身份进入房间
```

---

## 7. 加入限制逻辑

### 7.1 后端拦截

JoinRoom 中查询 IsFullAgentRoom，返回 35013 错误。

### 7.2 前端适配

房间列表中全 Agent 房间显示「观战」按钮，加入失败自动跳转观战。

---

## 8. 观战者流程

### 8.1 观战入口

- 房间列表中的「👁 观战」按钮
- 创建成功后的自动跳转
- 加入失败（全 Agent 拒绝）后的自动跳转

### 8.2 观战权限

- 观战者可以实时观看游戏状态
- 观战者不能发送 game.wealth_action（后端硬拒，errcode 30011）

---

## 9. 错误码定义

**文件**：`ServerGo/errcode/errcode.go`

```go
ErrWealthFullAgentRoom = 35013
```

---

## 10. 测试验证方案

### 10.1 后端测试

| 测试项 | 验证内容 |
|--------|---------|
| 创建全 Agent 房间 | 房间 FullAgentMode=true |
| 人类加入全 Agent 房间 | 返回 35013 错误 |
| 人类观战全 Agent 房间 | 成功，接收广播消息 |
| 全 Agent 自动开局 | bot >= MinSeats 后自动 Start |

### 10.2 前端测试

| 测试项 | 验证内容 |
|--------|---------|
| 创建弹窗默认全 Agent | 默认开启，Agent 数=12 |
| 全 Agent 房间列表标签 | 显示「🤖 全 Agent」标签 |
| 观战按钮 | 全 Agent 房间显示「观战」按钮 |

---

## 附录：改动文件清单

### 后端

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `ServerGo/game/wealth/room.go` | 修改 | 新增 FullAgentMode 字段及相关方法 |
| `ServerGo/service/room_service.go` | 修改 | GameJoiner 接口扩展 |
| `ServerGo/service/room_service_crud.go` | 修改 | CreateRoomWithAgents / JoinRoom 逻辑 |
| `ServerGo/ws/game_service.go` | 修改 | SetFullAgentMode / IsFullAgentRoom 实现 |
| `ServerGo/ws/game_service_wealth_bot.go` | 修改 | 注册 bot 后置位 FullAgentMode |
| `ServerGo/errcode/errcode.go` | 修改 | 新增 35013 错误码 |

### 前端

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `ClientWeb/src/components/wealth/WealthCreateRoomModal.tsx` | 修改 | 全 Agent 开关 + 模型选择 |
| `ClientWeb/src/pages/WealthLobbyPage.tsx` | 修改 | 全 Agent 标签 + 观战按钮 |
| `ClientWeb/src/pages/WealthGamePage.tsx` | 修改 | 观战模式 UI 适配 |
| `ClientWeb/src/api/wealth.ts` | 修改 | API 类型扩展 |
| `ClientWeb/src/types/wealth.ts` | 修改 | 新增 full_agent 字段 |
