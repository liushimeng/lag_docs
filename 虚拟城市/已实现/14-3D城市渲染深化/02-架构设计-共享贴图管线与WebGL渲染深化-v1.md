# 虚拟城市 — 共享贴图管线与 WebGL 渲染深化（14-3D城市渲染深化 · 02）

> 2026-09-22。实现契约。读者：frontend-dev。
> 前置契约：13/02-架构设计-WebGL渲染管线优化-v1.md（archetype 体块 / 光照常量不变）。

## 1. 共享贴图缓存（阶段 H 核心）

### 1.1 问题

`BuildingMesh` / `Road` / `DistrictBlock` / `Vehicle` / `Pedestrian` / `Tree` / `Sign` / `RooftopAcc` / `StreetLight` / `Ground`
各自 `new THREE.TextureLoader().load(...)`。16 区 × 5 楼 × 3 贴图 ≈ **240 次重复 GPU 上传**同源 30 张 PNG，
且组件卸载各自 `dispose()`（共享源被误销毁风险）。

### 1.2 方案：`components/wealth/textureCache.ts`（新）

```
loadSharedTexture(url, opts): THREE.Texture | null
  opts: { wrap?: 'repeat' | 'clamp'; repeat?: [rx, ry]; srgb?: boolean }
```

- **模块级 Map 缓存**，key = `${url}|${wrap}|${rx}|${ry}|${srgb}`。
- 命中即同步返回；未命中返回 null 并异步填充后 `notify`（实现上用 React hook `useSharedTexture` 暴露 state）。
- **禁止组件侧 dispose 共享纹理** —— 缓存随页面生命周期存活（游戏页面切换即整页卸载，量级可接受）。
- 失败 → 缓存哨兵 `FAILED`，`useSharedTexture` 返回 null（各组件既有降级链不动）。

### 1.3 hook 形态

```ts
export function useSharedTexture(
  url: string,
  opts?: SharedTextureOpts,
): THREE.Texture | null
```

- 各组件删掉私有 `useTexture` / `useStreetTile` / 手写 loader，统一换 hook。
- `url === ''`（资产缺失）→ 返回 null，零副作用（§9 降级链不变）。

## 2. 渲染管线微调（阶段 H）

| 项 | 现状 | 改为 | 理由 |
|---|---|---|---|
| OrbitControls | 无阻尼 | `enableDamping` `dampingFactor={0.08}` + `enableDamping` 需在 useFrame 有效（drei 内置） | 漫游手感 |
| dpr | [1, 2] | [1, 1.75] | 高分屏填充率；Retina 上肉眼差 <3% |
| gl | `{ antialias: true }` | + `powerPreference: 'high-performance'` | 独显调度 |
| 光照/雾/ACES/阴影 | 13 阶段 C 交付 | **不改** | 契约沿用 |

## 3. 新增场景组件

### 3.1 `props/WaterPlane.tsx`（新）

| props | 类型 | 说明 |
|---|---|---|
| x, z | number | 水体中心世界坐标 |
| w, d | number | 世界单位尺寸 |
| rotation? | number | 绕 Y 旋转（默认 0） |

- geometry：`planeGeometry [w, d]` 旋转 -π/2 贴 y=0.028。
- 材质：`water_tile.png` RepeatWrapping，repeat=(w/2, d/2)；`useFrame` 内 `map.offset.y += dt * 0.02`（REDUCED_MOTION 静止）。
- `meshStandardMaterial` roughness 0.15 metalness 0.35 → 轻微镜面感；无环境贴图时不追加 reflection（避免第二渲染通道）。
- 缺贴图 → `#1a3a52` 纯色水面（§9 降级）。

### 3.2 `props/BusStop.tsx`（新）

- 几何公交站台：2 圆柱（r=u(0.08), h=u(2.8)）+ 顶棚 box（u(2.4)×u(0.12)×u(1.0)）+ 灯箱 box（u(1.6)×u(1.0)×u(0.08)），
  灯箱 emissive `#ffe9b8` intensity 0.5。
- props: `x, z, rotation`。米制全部经 `cityScale.u()`。

### 3.3 `props/StreetFurniture.tsx`（新）

- `bench`：座板 + 靠背 2 box（木色 #6b4f3a）。
- `hydrant`：圆柱 r=u(0.12) h=u(0.55)（红 #c0392b）+ 顶半球省略（box 盖）。
- props: `x, z, variant: 'bench' | 'hydrant', rotation`。

## 4. 建筑临街细节（`building_shapes.tsx` 扩展）

### 4.1 底商雨棚 + 灯带（TowerShape 裙楼 / SlabShape）

- 位置：面向 -Z（街景默认朝向）y = h_shop = min(u(3.6), pH 或 h*0.3)。
- 雨棚：box `[w*0.9, u(0.35), u(1.2)]`，position z = -(d/2 + u(0.5))，色 `#8a5a44`。
- 灯带：box `[w*0.85, u(0.18), u(0.06)]` 贴雨棚下沿，emissive `#ffd9a0` intensity = emissive*1.2（上限 0.45）。
- pavilion / house / shed 不加（低层住宅/厂房无沿街商业，契约 01 §3.3）。

### 4.2 广告牌（TowerShape，w > 1.4 时）

- 双杆 cylinder（r=u(0.06), h=u(2.0)）立于 crown 顶 y=pH+bodyH+cH。
- 面板 box `[w*0.5, u(1.4), u(0.08)]`，emissive `#ffd9a0` intensity 0.5，metalness 0.4。
- 确定性：由 w 决定是否出现（同楼同形，§13 伪随机约束：不引 Math.random）。

## 5. 地表系统（`DistrictBlock` / `WealthCityMap` 扩展）

| 组件 | 改动 |
|---|---|
| `DistrictBlock` | ① curb：底板四边 4×box `[8.1, u(0.5), u(1.2)]` 或等效窄条，色 `#3a414c`；② `def.id==='central_park'` 时叠加 grass 覆盖层（y=0.026）；③ `transport_hub` 南半 + `finance` 中心 plaza 覆盖层（y=0.025） |
| `WealthCityMap` | 新增 `<WaterLayer />`（内部渲染运河 + 港池 2 块 `<WaterPlane>` + 两岸草皮收边 2 条） |

- 覆盖层 plane 一律 `position y` 递增 0.001 级别防 z-fighting（地面 0.02 → plaza 0.024 → grass 0.026 → curb 顶 0.05）。

## 6. 行人漫步（`props/Pedestrian.tsx` 改造）

- 增 props `pathR?`（默认 1.5）：`useFrame` 内 t += dt*0.12，位置 = 圆心 + [cos(t), sin(t)]×pathR。
- 原 x/z 语义改为中心；`StreetPropsLayer` 布点不变（多传 pathR 0.8–2.0 确定性）。
- `REDUCED_MOTION` → 固定在 (x, z) 不动（比现状「微抖」更符合规范）。
- 微上下浮动保留（0.005 sin）。

## 7. 布点扩展（`StreetPropsLayer.tsx`）

| 新增 | 布点规则 | 预算 |
|---|---|---|
| BusStop | 每条主干道（len≥12）t=0.35 路侧偏移（与路灯同侧公式） | ≤ 12 |
| StreetFurniture | 每城区 1 件（idx 偶=bench，奇=hydrant），树池旁半径 2.6 | 16 |
| 漫步行人 | 每城区 1 人改漫步；central_park 加第 2 人 | 17 |

- mesh 预算核算：现状 ≈ 300（楼 ~80×3 体块 + props ~120 + 道路层）；新增 curb 64 + 水 6 + 站台 12×4 + 家具 16 + 雨棚/灯带 ~160 + 广告牌 ~30 ≈ **+290 → 总 ≈ 590 < 2500 护栏**。

## 8. 文件落点（§2.1 / §4 约束）

| 文件 | 动作 | 行数预估 |
|---|---|---|
| `components/wealth/textureCache.ts` | 新 | ~110 |
| `components/wealth/props/WaterPlane.tsx` | 新 | ~90 |
| `components/wealth/props/BusStop.tsx` | 新 | ~80 |
| `components/wealth/props/StreetFurniture.tsx` | 新 | ~80 |
| `components/wealth/building_shapes.tsx` | 扩展（+~90） | 324→~415 |
| `components/wealth/BuildingMesh.tsx` | 换共享缓存（-30） | 112→~95 |
| `components/wealth/Road.tsx` | 换共享缓存 + 虚线 | 229→~225 |
| `components/wealth/DistrictBlock.tsx` | curb/grass/plaza | 201→~250 |
| `components/wealth/WealthCityMap.tsx` | WaterLayer + damping/dpr | 343→~390 |
| `components/wealth/StreetPropsLayer.tsx` | 站台/家具/漫步参数 | 284→~340 |
| `components/wealth/props/*.tsx`（既有 7 个） | 私有 loader → useSharedTexture | 各 -15~25 |
| `assets/images/wealth/index.ts` | ground 类别导出 | +25 |
| `styles/wealth-city3d.css` | UI 阶段 J 增补 | 133→~175 |

- 全部文件 ≤ 1800 行（§4）；纯前端，`ServerGo/` 不动。
- 新 helper 全部 grep 验证接线（§130）：`useSharedTexture|WaterPlane|BusStop|StreetFurniture|pathR`。
