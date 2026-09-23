# 虚拟城市 — PBR 材质管线与建筑几何深化 · 架构设计（18 · 02）

> 2026-09-23。架构契约文档。读者：frontend-dev。
> 前置契约：13/02（体块形态）+ 14/02（共享贴图缓存）+ 15/02（V2 道具）+ 16/02（渲染增强）全部沿用。
> 本文件是阶段 X / Y / Z / AA 的**逐符号实现契约**——文件路径、导出名、常量、props 均不可随意改名。

---

## 1. 总体架构

单管线下扩展，无重构。新增 1 个贴图钩子 + 1 个市政层挂载点 + 1 个 civic 目录：

```
WealthCityMap
├── EnvBinder / fill light / Sky / fog / CloudLayer          （16 批次，不动）
├── Ground                     （阶段X: PBR；阶段AA: 扩围 120）
├── WaterLayer → WaterPlane    （阶段X: 双层滚动法线）
├── AtmosphereLayer            （15 批次，不动）
├── CivicLayer                 （阶段AA 新）→ civic/* 15 项设施
├── WEALTH_DISTRICTS.map(DistrictBlock)   （阶段X: 底板/草地/广场 PBR）
│   └── BuildingMesh → BuildingShape      （阶段X: PBR；阶段Y: 构造件）
├── RoadsLayer / RingRoad      （阶段X: 路面 PBR）
├── CanalBridges               （阶段X: 桥墩/桥面 PBR）
├── StreetPropsLayer           （阶段Z: VehicleV2 / PedestrianV3 / TreeV3；阶段AA: 路口信号灯）
└── AgentToken / OrbitControls / CameraReporter / FocusController（不动）
```

## 2. PBR 贴图管线（阶段 X）

### 2.1 访问器契约（`assets/images/wealth/index.ts` 追加）

```ts
// ── 18-3D城市PBR材质与真实城市冲刺 · 阶段 X 新增（PBR 贴图）─────────

const pbrImgs = import.meta.glob<string>('./pbr/**/*.png', { eager: true, import: 'default' });

/** 派生 PBR 贴图类别（与 3d_script/procedural_pbr_maps.py::DERIVED_JOBS 对齐）。 */
export type PbrCategory = 'facades' | 'roofs' | 'streets' | 'ground' | 'districts' | 'synth';

/**
 * 法线贴图 URL（缺失 = ''，调用方保持现状材质）。
 * @param category 派生类别；name 为颜色贴图的 stem
 *   例：pbrNormalUrl('facades', 'finance_mid') → ./pbr/facades/finance_mid_n.png
 *       pbrNormalUrl('synth', 'water')         → ./pbr/synth/water_n.png
 */
export function pbrNormalUrl(category: PbrCategory, name: string): string {
  return pbrImgs[`./pbr/${category}/${name}_n.png`] ?? '';
}

/** 粗糙度贴图 URL（缺失 = ''）。synth/water 无 _r（水面粗糙度是材质常量）。 */
export function pbrRoughUrl(category: PbrCategory, name: string): string {
  return pbrImgs[`./pbr/${category}/${name}_r.png`] ?? '';
}
```

> **§130 接线自检**：两个新函数写完立即 `git grep -n "pbrNormalUrl\|pbrRoughUrl"`，
> 命中数 ≥ 8（建筑 / 地面 / 道路 / 桥 / 水 / 草地 / 广场 / 树 / 市政）。

### 2.2 `useSharedPBR`（`textureCache.ts` 追加）

```ts
export interface SharedPBROpts {
  wrap?: 'repeat' | 'clamp';
  repeat?: [number, number];
  /** 法线强度（three `normalScale`，默认 [1, 1]）。 */
  normalScale?: [number, number];
}

export interface SharedPBR {
  map: THREE.Texture | null;
  normalMap: THREE.Texture | null;
  roughnessMap: THREE.Texture | null;
  /** 拼好的材质 props（map/normalMap/roughnessMap/normalScale）；贴图缺失的键自动省略。 */
  matProps: {
    map?: THREE.Texture;
    normalMap?: THREE.Texture;
    roughnessMap?: THREE.Texture;
    normalScale?: THREE.Vector2;
  };
}

/**
 * 共享 PBR 三件套。三张贴图**共用同一 wrap/repeat**（同一套 UV），
 * 法线/粗糙度强制 `srgb: false`（colorSpace = NoColorSpace，线性空间）。
 * 全部走 useSharedTexture → 进程级缓存，禁组件自建 loader。
 */
export function useSharedPBR(
  colorUrl: string,
  normalUrl: string,
  roughUrl: string,
  opts?: SharedPBROpts,
): SharedPBR
```

**实现要点**：

1. 三次 `useSharedTexture` 调用，key 自然分色（`srgb` 已在 key 中）。
2. **法线 / 粗糙度必须传 `{ srgb: false }`** —— 颜色图才是 sRGB；法线是方向向量、
   粗糙度是标量，走 sRGB 会让凹凸强度随色彩空间被扭曲（常见「法线过弱」根因）。
3. `normalScale` 用 `useMemo(() => new THREE.Vector2(...), [...])`，组件卸载不 dispose。
4. `matProps` 里**省略值为 null 的键**（不能写 `normalMap: null`，three 会告警）。
5. 三张 url 任意为空 → 对应项返回 null，`matProps` 自动省略（降级链）。

### 2.3 材质接线表（14 类，逐处给出符号与 normalScale）

| # | 材质 | 位置（文件 · 符号） | color | normal / rough | normalScale | 备注 |
|---|---|---|---|---|---|---|
| 1 | 楼宇立面 | `building_shapes.tsx::sideMatProps` | `facades/<id>_<v>` | `pbrNormalUrl('facades', stem)` | `[0.8, 0.8]` | stem 由调用方传入（`<id>_base` / `<id>_mid`） |
| 2 | 屋顶 | `building_shapes.tsx::topMatProps` | `roofs/<id>` | `pbr/roofs/<id>` | `[0.7, 0.7]` | |
| 3 | 坡屋顶 | `building_shapes.tsx::PrismRoof` | `roofs/<id>` | 同上；**无贴图时**用 `pbr/synth/tile_roof_n/r` | `[1.0, 1.0]` | 红瓦兜底色保留 |
| 4 | 城区底板 | `DistrictBlock.tsx` 底板 mesh | `districts/<id>` | `pbr/districts/<id>` | `[0.5, 0.5]` | |
| 5 | 草地 | `DistrictBlock.tsx` 公园覆盖层 | `ground/grass_tile` | `pbr/ground/grass_tile` | `[0.7, 0.7]` | |
| 6 | 广场铺装 | `DistrictBlock.tsx` 广场覆盖层 | `ground/plaza_tile` | `pbr/ground/plaza_tile` | `[0.8, 0.8]` | |
| 7 | 城市地面 | `WealthCityMap.tsx::Ground` | `ground/urban_base` | `pbr/ground/urban_base` | `[0.6, 0.6]` | |
| 8 | 沥青路面 | `Road.tsx` 主车道 + `RingRoad.tsx` | `streets/asphalt_main` | `pbr/streets/asphalt_main` | `[0.5, 0.5]` | |
| 9 | 人行道 | `Road.tsx` 两侧 | `streets/sidewalk_*` | `pbr/streets/sidewalk_*` | `[0.8, 0.8]` | |
| 10 | 水面 | `props/WaterPlane.tsx` | `ground/water_tile` | **`pbr/synth/water_n`**（双层滚动） | `[0.35, 0.35]` | 无 rough 贴图 → `roughness: 0.08` |
| 11 | 混凝土 | `CanalBridge.tsx` 桥墩/桥面 + civic 挡墙 | 纯色 `#6b7280` | `pbr/synth/concrete_n/r` | `[0.8, 0.8]` | |
| 12 | 砖 | `civic/FireStation.tsx` / `PoliceStation.tsx` / `civic/CityHall.tsx` | 纯色 | `pbr/synth/brick_n/r` | `[1.0, 1.0]` | |
| 13 | 压型钢板 | `ShedShape` 屋面 + `civic/PortTerminal.tsx` 集装箱/岸吊 | 纯色 | `pbr/synth/metal_deck_n/r` | `[0.9, 0.9]` | |
| 14 | 树冠 | `props/TreeV3.tsx` | 纯色 | `pbr/synth/foliage_n/r` | `[1.2, 1.2]` | |

**统一材质规则**：

```ts
// 有 roughnessMap 时不再写死 roughness（贴图全权决定）；无 roughnessMap 保持原常量。
const withPBR = (base: Record<string, unknown>, pbr: SharedPBR) => ({
  ...base,
  ...pbr.matProps,
  // roughnessMap 存在时不传 roughness（three 用 roughnessMap.g × roughness 默认 1.0）
  ...(pbr.matProps.roughnessMap ? {} : { roughness: base.roughness }),
});
```

**`SharedPBR` 的取用方式**（三个入口的对应关系，避免接错图）：

| 既有的颜色贴图入参 | 对应的 `SharedPBR` | 取 url 的方式 |
|---|---|---|
| `ShapeProps.facadeBase` | `ShapeProps.pbrBase` | `pbrNormalUrl('facades', `${def.id}_base`)` |
| `ShapeProps.facadeMid` | `ShapeProps.pbrMid` | `pbrNormalUrl('facades', `${def.id}_mid`)` |
| `ShapeProps.roofMap` | `ShapeProps.roofPbr` | `pbrNormalUrl('roofs', def.id)` |

> `BuildingMesh.tsx` 是唯一拼 stem 的地方：它持有 `def.id`，用三次
> `useSharedPBR(districtFacadeUrl(def.id,'base'), pbrNormalUrl('facades', `${def.id}_base`),
> pbrRoughUrl('facades', `${def.id}_base`), { normalScale:[0.8,0.8] })` 这样的形式取三件套，
> 再原样下传。`building_shapes.tsx` 内**不得** import `@/assets/images/wealth` 拼路径。

### 2.4 水面双层滚动法线（`props/WaterPlane.tsx`）

> **最易踩的坑**：`useSharedTexture` 返回的是**进程级共享**贴图，`offset` 也是共享的。
> 两个水面同时改 offset 会互相打架；同尺寸水面还会「同相位」闪烁。
> **必须 `tex.clone()`** —— clone 共享 `image`/GPU 上传数据，但 `offset/repeat` 独立。

```tsx
// WaterPlane 内（示意）：
const shared = useSharedPBR(groundTileUrl('water_tile'), pbrNormalUrl('synth', 'water'), '', {
  wrap: 'repeat', repeat: [w / 2, d / 2], normalScale: [0.35, 0.35],
});
// 双层滚动：两个独立 offset 的 normal 贴图实例
const n1 = useMemo(() => {
  const t = shared.normalMap?.clone(); if (t) { t.needsUpdate = true; t.wrapS = t.wrapT = THREE.RepeatWrapping; }
  return t;
}, [shared.normalMap]);
const n2 = useMemo(() => { const t = n1?.clone(); if (t) { t.repeat.set(1.7, 1.3); } return t; }, [n1]);
// useFrame（REDUCED_MOTION 时跳过）：
//   n1.offset.x += delta * 0.012; n1.offset.y += delta * 0.006;
//   n2.offset.x -= delta * 0.008; n2.offset.y += delta * 0.015;
// 卸载：useEffect(() => () => { n1?.dispose(); n2?.dispose(); }, [n1, n2]);
```

第二层法线无法直接叠在一个 `meshStandardMaterial` 上 → **用第二个叠加 mesh**：

| 层 | y | 材质 | 说明 |
|---|---|---|---|
| 主水面 | 0.028 | `map=water_tile`, `normalMap=n1`, `transparent opacity 0.92`, `roughness 0.08`, `metalness 0.35`, `envMapIntensity 0.9` | 既有颜色 + 第一层波 |
| 细波层 | 0.029 | **无 map**，`normalMap=n2`, `transparent opacity 0.35`, `color #9fd4ea`, `depthWrite: false`, `blending: NormalBlending` | 第二层波，交错产生细波 |

> `clone()` 后必须 `needsUpdate = true`，否则 three 沿用旧 GPU 句柄。

## 3. 建筑几何深化（阶段 Y）

全部在 `building_shapes.tsx` 内实现（保持 1800 行内；若超限则拆
`building_details.tsx` 并在原文件 re-export，纯搬移不改签名）。

### 3.1 `mergeBoxes` 工具（本文件新增，导出供 civic 复用）

```ts
/**
 * 把 N 个 box 描述合并为 1 个 BufferGeometry（1 mesh 渲染 N 个体块）。
 * 目的：把「每层一道阳台线」「四角柱」这类 N 件套从 N 个 draw call 压到 1 个。
 * 纯手写顶点拼接（不引入 three/examples/jsm 依赖）。
 */
export interface BoxSpec {
  /** 中心偏移（父组局部坐标）。 */
  x: number; y: number; z: number;
  w: number; h: number; d: number;
}
export function mergeBoxes(boxes: BoxSpec[]): THREE.BufferGeometry
```

实现：对每个 box 生成 24 顶点（6 面 × 4）+ 36 索引 + 简单盒式 UV（0..1 每面），
拼进同一 `BufferGeometry`；`computeVertexNormals()`。**必须 `useEffect` dispose**。

### 3.2 新增构造件（每个给出几何规格）

| 组件 | 几何 | 材质 | 适用 | mesh 计 |
|---|---|---|---|---|
| `ParapetRing({ w, d, y })` | `mergeBoxes` 4 条：前后 `[w, u(0.9), u(0.25)]` @ z=±(d/2−u(0.125))；左右 `[u(0.25), u(0.9), d−2·u(0.25)]` @ x=±(w/2−u(0.125)) | 混凝土 PBR（synth/concrete）；无 PBR 时 `#8a8f98` | tower 裙楼顶 `y=pH`、slab 顶 `y=h`、shed 檐口上 `y=bH` | **1** |
| `EntranceLobby({ w, d, y })` | 玻璃门 plane `[u(1.4), u(2.4)]` @ z=+d/2+0.002，y=u(1.2)；门框 2 竖 1 横 `mergeBoxes`；雨棚 box `[u(2.2), u(0.12), u(0.9)]` @ y=u(2.6), z=d/2+u(0.45) | 门 `#2a4a6e` rough 0.10 metal 0.30 envMapIntensity 0.9；框 `#3a414c`；雨棚 `#4a5568` | tower / slab / house（shed 有卷帘门、pavilion 有挑檐，不加） | 2 |
| `CornerQuoins({ w, d, h })` | `mergeBoxes` 4 条竖凸条 `[u(0.15), h, u(0.15)]` @ 四角 | `pbr/synth/brick_n/r` 或 `#9a8f84` | house / slab（w>1.4） | **1** |
| `AcUnits({ w, d, y })` | `mergeBoxes` 2–3 个小盒 `[u(0.5), u(0.35), u(0.25)]` @ z=+d/2+u(0.12)，y 由 `hashStr` 确定楼层 | `#c5c8ce` rough 0.55 metal 0.25 | tower 塔身 / slab | **1** |
| `LiftRoom({ w, d, y })` | 小屋 box `[w*0.3, u(2.4), d*0.3]` + 擦窗机轨道 `mergeBoxes` 一圈细条 | 屋 `#6b7280`；轨 `#3a414c` | tower（w>1.2），置于 crown 顶 | 2 |
| `PodiumRail({ w, d, y })` | `mergeBoxes` 一圈立柱（每 u(1.5) 1 根 `[u(0.08), u(1.1), u(0.08)]`）+ 顶部扶手 4 条 | `#3a414c` rough 0.6 metal 0.4 | tower 裙楼顶外缘 | **1** |
| `SawtoothRoof({ w, d, y })` | 3 齿：每齿 1 个 `prismGeometry`（齿高 u(1.2)，齿宽 w/3）+ 齿背面采光带 plane（浅蓝半透明 `#bcd9ea` opacity 0.55，`envMapIntensity 0.9`） | 齿面 `pbr/synth/metal_deck_n/r` | shed（w>1.4，**替代** `PrismRoof` 人字顶） | 6 |
| `ShedWindowBands({ w, d, h })` | `mergeBoxes` 2 条横向高窗 `[w*1.2*0.9, u(0.6), u(0.06)]` @ z=+d/2+0.003，y=h*0.55 / h*0.75 | `#7fa8c4` rough 0.15 metal 0.2 | shed | **1** |

### 3.3 现有符号的改造点

| 符号 | 改动 |
|---|---|
| `BalconyLines` | **只画 +Z 一面 → 四面**；`count` 条 × 4 面改用 `mergeBoxes` 合并成 **1 mesh**；跳过底层 `y<u(3.5)` 与顶层 `y>h−u(1)` 不变 |
| `RooftopEquipment` | 不动（15 批次已交付水箱/通风管/天窗） |
| `ShutterDoor` | 不动；`ShedShape` 中 `w>1.4` 时用 `SawtoothRoof` 替代 `PrismRoof`，`w≤1.4` 保持 `PrismRoof` |
| `Shopfront` / `BillboardSign` | 不动 |
| `TowerShape` | 追加 `EntranceLobby` / `AcUnits` / `LiftRoom` / `PodiumRail` / `ParapetRing(y=pH)` |
| `SlabShape` | 追加 `EntranceLobby` / `AcUnits` / `CornerQuoins` / `ParapetRing(y=h)` |
| `HouseShape` | 追加 `EntranceLobby` / `CornerQuoins`；坡顶无贴图时接 `tile_roof` PBR |
| `ShedShape` | 追加 `ShedWindowBands` + `SawtoothRoof`（条件）+ `ParapetRing(y=bH)`（锯齿顶时改为檐口收边） |
| `PavilionShape` | 不动 |
| `ShapeProps` | 追加可选三件套：`pbrBase?: SharedPBR`（对应 `facadeBase`）、`pbrMid?: SharedPBR`（对应 `facadeMid`）、`roofPbr?: SharedPBR`（对应 `roofMap`）。**不要只给一个 `pbr`**——base 与 mid 是两张不同贴图，共用一个 `SharedPBR` 会让窗洞凹凸错位 |
| `BoxFaces` | props 追加 `pbrA` / `pbrB` / `pbrTop`（`SharedPBR`，可选），在 `sideMatProps` / `topMatProps` 内 spread `matProps` |

### 3.4 mesh 预算（阶段 Y 实测口径）

| 件 | 每楼 | 楼数 | 合计 |
|---|---|---|---|
| ParapetRing | 1 | ~140 | 140 |
| EntranceLobby | 2 | ~100（tower+slab+house） | 200 |
| CornerQuoins | 1 | ~60 | 60 |
| AcUnits | 1 | ~100 | 100 |
| LiftRoom | 2 | ~15 | 30 |
| PodiumRail | 1 | ~15 | 15 |
| SawtoothRoof | 6 | ~20 | 60 |
| ShedWindowBands | 1 | ~20 | 20 |
| BalconyLines（4 面合并） | 1 | ~70 | 70 |
| **小计** | | | **695** |

## 4. 人物与车辆（阶段 Z）

### 4.1 `props/PedestrianV3.tsx`（新）

```tsx
export interface PedestrianV3Props {
  /** 漫步路径折线（世界坐标 x,z；首尾不闭合，到端点折返）。 */
  path: Array<[number, number]>;
  /** 步速（世界单位/秒，默认 0.55）。 */
  speed?: number;
  /** 0..3 四套服装色（商务蓝 / 休闲灰 / 亮色红 / 卡其）。 */
  outfit?: 0 | 1 | 2 | 3;
  /** 起始相位（0..1，错开步态，避免全员同步摆腿）。 */
  phase?: number;
}
```

| mesh | 几何 | 枢轴 |
|---|---|---|
| 躯干 | `boxGeometry [u(0.32), u(0.58), u(0.18)]` @ y=u(1.11)（跨度 u(0.82)–u(1.40)） | 组根 |
| 头 | `sphereGeometry r=u(0.12)` @ y=u(1.55)（跨度 u(1.43)–u(1.67)） | 组根 |
| 左/右臂 | box `[u(0.09), u(0.52), u(0.09)]`，`position.y = u(1.34)`，**`geometry.translate(0, -u(0.26), 0)`** 让 pivot 落在肩（手端垂到 u(0.82)） | `rotation.x = sin(t·ω + φ) · 0.35` |
| 左/右腿 | box `[u(0.11), u(0.82), u(0.11)]`，`position.y = u(0.82)`，**`geometry.translate(0, -u(0.41), 0)`** 让 pivot 落在髋（**脚端落在 y=0 贴地**） | `rotation.x = -sin(t·ω + φ) · 0.45` |

> ⚠️ **尺寸自洽性（写码前必验）**：**腿长 = 髋 y**，脚端才能贴地（y=0）。
> 初版曾写「腿 u(0.52) @ 髋 y=u(0.85)」→ 脚底悬空 u(0.33)（33 cm），属尺寸自洽性错误；
> 上表已修正。通式：`腿长 = 髋 y`；`躯干中心 y = (髋 y + 肩 y) / 2`；`头中心 y = 肩 y + u(0.21)`。

- 总高 **u(1.67)** ≈ 1.7 m（脚底 0 → 头顶 u(1.67)）；mesh 数 **6 / 人**；全城 56 人 → 336 mesh。
- 行走：沿 `path` 折线推进（复用 PedestrianV2 的推进语义：归一化 t + 端点折返停顿），
  组朝向 = 行进方向 `Math.atan2(dx, dz)`。
  > 实现注意：V2 有把 `useFrame` 的 `delta` 当绝对时间用的缺陷（端点停顿 500 单位会永久卡死）。
  > V3 **保留推进语义但用累计时间正确实现**，不照搬该缺陷。
- `REDUCED_MOTION`：**位移 + 摆肢全部静止**（吸附 `path[0]`、四肢 rotation 归 0）——
  与 `PedestrianV2` 既有约定一致，也与 15 批次「reduced-motion 全部动画静止」对齐。
- 配色常量导出 `PEDESTRIAN_OUTFITS: [string, string, string][]`（上衣 / 裤 / 肤）。

### 4.2 `props/Vehicle.tsx`（增强，props 向后兼容）

现有 props `{ x, z, from, to, speed, variant, laneOffset, ... }` **全部保留**。新增内部件：

| 件 | 几何 | 材质 |
|---|---|---|
| 前挡风 | box `[车身宽×0.85, u(0.28), u(0.04)]` @ 前段上部，后倾 25° | `#2a4a6e` transparent opacity 0.55 rough 0.10 metal 0.25 envMapIntensity 0.9 |
| 侧窗 ×2 | box `[u(0.04), u(0.22), 车身长×0.4]` | 同上 |
| 前大灯 ×2 | box `[u(0.1), u(0.06), u(0.04)]` | `#fff6d8` emissive 同色 intensity 0.8 |
| 尾灯 ×2 | box `[u(0.1), u(0.06), u(0.04)]` | `#ff3b30` emissive 同色 intensity 0.5 |
| 轮毂 ×4 | `cylinderGeometry r=u(0.07) h=u(0.03)` 贴轮外侧 | `#c5c8ce` metal 0.6 rough 0.35 |
| 后视镜 ×2 | box `[u(0.05), u(0.04), u(0.05)]` | `#3a414c` |
| 雨刮 | box `[u(0.3), u(0.015), u(0.015)]` @ 前挡风下沿 | `#1f2733` |

- 每车 mesh 由 ~5 → **19**（车身 1 + 轮 4 + 毂 4 + 挡风 1 + 侧窗 2 + 灯 4 + 镜 2 + 雨刮 1）；
  20 辆 → 380 mesh（原 100）。消防车 / 巡逻车复用同一组件，仅传 `palette` 覆盖色。
- **新 prop**（可选，向后兼容）：`palette?: { body: string; roof: string; accent?: string }`。
  三字段全部接线：`roof` → 车顶 +Y 面材质、`accent` → 前后 ±X 面材质（六面
  `attach="material-N"`，与 `building_shapes.tsx::BoxFaces` 同口径，**零额外 mesh**）。
  传 `palette` 时走几何体分支（sprite 彩绘无法换色）。

### 4.3 `props/TreeV3.tsx`（新）

| 件 | 几何 | 材质 |
|---|---|---|
| 主干 | `cylinderGeometry r=u(0.09)/u(0.13) h=u(1.6)` | `#5a4634` rough 0.9 |
| 分枝 ×2 | 细圆柱倾斜 30° | 同上 |
| 树冠 ×3 | `icosahedronGeometry r=u(0.7/0.85/0.6)`，中心 (0,u(2.3),0) 及 ±(u(0.35), u(0.15)) | 纯色 `#2f7a3a` + `foliage_n/r` PBR |

- mesh 6 / 棵；全城树由 `StreetPropsLayer` 编排，总数上限 40 → 240 mesh。
- 确定性：冠球偏移由 `hashStr(seed)` 决定。

## 5. 市政设施（阶段 AA）

### 5.1 新目录 `components/wealth/civic/`

> **目录约束**（CLAUDE.md §2.1 硬约束 2）：虚拟城市私有代码必须落在
> `components/wealth/` 下；本批新增的 15 个设施组件归 `wealth/civic/`，**不得**进 `components/ui|common/`。

| 文件 | 导出 | mesh 预算 |
|---|---|---|
| `PortTerminal.tsx` | `PortTerminal({ x, z })` | 28 |
| `SportsField.tsx` | `SportsField({ x, z, rotation })` | 10 |
| `RailViaduct.tsx` | `RailViaduct({ z, x0, x1 })` | 23 |
| `HeliPad.tsx` | `HeliPad({ x, z })` | 6 |
| `GasStation.tsx` | `GasStation({ x, z, rotation })` | 12 |
| `Substation.tsx` | `Substation({ x, z, rotation })` | 5 |
| `WaterTower.tsx` | `WaterTower({ x, z })` | 3 |
| `CommTower.tsx` | `CommTower({ x, z })` | 6 |
| `FireStation.tsx` | `FireStation({ x, z, rotation })` | 8 |
| `PoliceStation.tsx` | `PoliceStation({ x, z, rotation })` | 7 |
| `CityHall.tsx` | `CityHall({ x, z, rotation })` | 12 |
| `Outskirts.tsx` | `Outskirts()`（内部按 r∈[34,58] 确定性布点） | 28 |
| `ParkExtras.tsx` | `ParkExtras({ x, z })` | 14 |
| `CanalExtras.tsx` | `CanalExtras()` | 14 |
| `IntersectionSignals.tsx` | `IntersectionSignals({ junctions })` | 48 |
| `CivicLayer.tsx`（`components/wealth/`） | `CivicLayer()` —— 上表统一挂载点 | — |

### 5.2 关键结构规格（易错处写死）

**PortTerminal（岸吊）** —— 局部坐标：岸线沿 x，海侧朝 −z：

| 件 | 几何 | y |
|---|---|---|
| 码头面 | plane `[u(80), u(25)]`（80 m × 25 m 码头岸线），`pbr/synth/concrete_n/r` | 0.03 |
| 门架腿 ×4 | box `[u(0.5), u(22), u(0.5)]`，x=±u(9)、z=±u(4) | u(11) |
| 横梁 | box `[u(22), u(1.2), u(1.0)]` | u(22.5) |
| 海侧悬臂 | box `[u(10), u(0.8), u(0.8)]` 伸向 −z | u(22.5) |
| 小车 | box `[u(1.6), u(1.0), u(1.4)]` @ 悬臂上 | u(21.5) |
| 钢缆 | `cylinderGeometry r=0.006 h=u(8)` | 从 u(21) 吊下 |
| 吊具 | box `[u(2.4), u(0.35), u(1.6)]` | 吊具底 u(13) |
| 集装箱 | box `[u(6), u(2.6), u(2.4)]`，4 色（`#c0392b` / `#2471a3` / `#e67e22` / `#27ae60`），3×3 平铺 + 2 层；`pbr/synth/metal_deck_n/r` | 堆在 (±u(11), 0, 0) |
| 系缆桩 ×4 | `cylinderGeometry r=u(0.15) h=u(0.5)` 沿岸 | 0.25 |

**RailViaduct（高架轻轨）**：

| 件 | 规格 |
|---|---|
| 箱梁 | box `[x1−x0, u(1.6), u(4.5)]`，`pbr/synth/concrete_n/r`，y=u(9) |
| 桥墩 | **13** 根 box `[u(1.8), u(9), u(1.8)]`，x 等距（每 u(50)=5 世界单位一根），y=u(4.5) |
| 车站 | 2 座：月台 plane `[u(80), u(6)]` y=u(10) + 雨棚 box + 楼梯 1 段斜面 + 站牌柱 |
| 列车 | 3 节 box `[u(22), u(3.2), u(3.2)]` 银白 + 深色窗带（透明 box）+ 车头斜切 |
| 栏杆 | `mergeBoxes` 沿梁两侧细条（1 mesh） |

**Outskirts（外围腹地）** —— 确定性 `mulberry32(hashStr('outskirts-v1'))`：

> **半径下限是算出来的，不是拍的**：16 个城区底板外角最远半径 = **36.88**（`industrial_park` 角点 (−28,−24)）。
> 任何占地超过点状的外围物，其**内缘**必须 ≥ 37.9，即 `放置半径 − 半对角线 ≥ 37.9`。
> 同时外缘必须 ≤ 60（地面 `WORLD_GROUND_SIZE/2`）。下表数值已按此双向约束选好。

| 件 | 规格 |
|---|---|
| 农田 ×8 | plane 6×4.5（半对角 3.75）@ r∈[42,55] 等角分布（内缘 38.25 ✓ / 外缘 58.75 ✓）；色 `#6b8e23` / `#c4a945` / `#8b7355` / `#556b2f` 轮转 + `pbr/ground/grass_tile_n` |
| 丘陵 ×4 | `sphereGeometry r=u(80)`（8 世界单位）+ `scale [1, 0.3, 1]` + `position.y = u(-9)` → 露出地面约 u(15)（15 m 丘）；@ r=50（内缘 42 ✓ / 外缘 58 ✓）；色 `#3f5a3a` |
| 环城高速 | 复用 `RingRoad` 几何思路：24 段 `plane` 半径 44，宽 1.6，`asphalt_main` + 护栏 `mergeBoxes`（43.2–44.8 ✓） |
| 风机 ×2 | @ r=48；塔 `cylinderGeometry r=u(0.3)/u(0.5) h=u(30)` + 机舱 box + 3 叶 `mergeBoxes`（叶长 u(8)）绕中心 `useFrame` 慢转 0.4 rad/s；`REDUCED_MOTION` 静止 |
| 风机 ×2 | 塔 `cylinderGeometry r=u(0.3)/u(0.5) h=u(30)` + 机舱 box + 3 叶 `mergeBoxes`（叶长 u(8)）绕中心 `useFrame` 慢转 0.4 rad/s；`REDUCED_MOTION` 静止 |

**IntersectionSignals**：

```ts
// WealthCityMap 计算 12 处交点（沿用 16 批次 RingRoad::junctionAngles 的口径）：
// 对每条 len>12 的放射 main 干道，交点 = 该干道方向 × RING_RADIUS(5.6)。
// 每处 2 杆：交点沿干道两侧 ±(路宽/2 + 0.3)，杆高 u(3.2)，
// 复用 props/TrafficLight，rotation 朝向来车方向 = 干道 angle。
```

### 5.3 地面扩围（`WealthCityMap.tsx`）

```ts
/** 城市建成区边长（城区 / 道路 / 相机语义，不变）。 */
export const WORLD_SIZE = 80;
/** 18 · 阶段 AA：地面 plane 边长（建成区外的腹地）。只影响 Ground 与其贴图 repeat。 */
export const WORLD_GROUND_SIZE = 120;
export const GROUND_REPEAT = WORLD_GROUND_SIZE / GROUND_TILE; // 8 → 15
```

- `Ground` 的 `planeGeometry args={[WORLD_GROUND_SIZE, WORLD_GROUND_SIZE]}`。
- `FOG_NEAR / FOG_FAR` **不变**（100 / 200）：腹地最远角 √(60²+60²)≈85 < 100，
  仍有雾感收敛；改小会把建成区泡进雾里（16 批次已实测）。
- `ORBIT_MAX_DISTANCE = WORLD_SIZE` **不变**（相机不许缩到看不见城）。
- 阴影相机半宽 `SHADOW_CAMERA_HALF = WORLD_SIZE * 0.6` **不变**（腹地不需要投影）。

### 5.4 挂载顺序（`WealthCityMap` 内，保持 z-fighting 分层）

```
y=0.000 Ground
y=0.015 Road / RingRoad
y=0.017 RingRoad
y=0.020 DistrictBlock 底板 / Civic 硬地
y=0.024 河岸草皮
y=0.025 广场铺装
y=0.026 ParkingLot / ConstructionSite 裸土
y=0.028 主水面
y=0.029 细波层
y=0.030+ 道具 / 建筑 / 市政设施
```

### 5.5 布点自洽性校验（**实现前必读，坐标已按此修正过一版**）

> 初版坐标表有 4 处会与既有几何穿插，已用脚本做数值碰撞预检后修正。规则如下，
> **新增/改动任何设施坐标时必须重跑同一套判据**（判据本身就是下表的四类禁入区）。

**四类禁入区**（对设施 AABB 的**四个角点**逐一判，不是只判中心）：

| 禁入区 | 范围 | 例外 |
|---|---|---|
| 城区底板 | 16 个 `districtCenter` 的 `±4` 方块 | — |
| 运河带 | `z ∈ [15.5, 18.5]` 且 `x ∈ [−32, 32]` | `CanalExtras` 的船/系船柱/护栏**必须**在带上 |
| 物流港港池 | `x ∈ [−33, −27]` 且 `z ∈ [−8, 0]` | — |
| 放射路 | 各非 finance 城区中心 → 原点的线段，安全距 `> 0.7（路半宽）+ 0.15` | — |

另加两条：
- **跨河桥位**（16 批次）`x = ±6.2, z = 17` 周边 u(25) 内不得放船（会被桥压住）。
- **外围物**：`放置半径 − 半对角线 ≥ 37.9` 且 `放置半径 + 半对角线 ≤ 60`（地面半宽）。

**修正记录（2026-09-23 数值预检实测）**：

| 设施 | 初版 | 问题 | 现值 |
|---|---|---|---|
| `GasStation` | (7, 5) | **正压在滨河新区放射路上（距 0.00）**；备选 (8.5,3.0) 又压文创区路（距 0.16） | **(5.8, 6.7)** |
| `PoliceStation` | (−6, −6) | 距工业区/产业基地放射路 1.66 / 0.77，不足安全距 | **(−5.8, −8.5)** |
| `CityHall` | (−6, −2) | 距工业区/物流港放射路 1.66 / 0.89 | **(−9.0, −1.7)** |
| `CanalExtras` 游船 | (−6,17) / (10,17) | 第一艘**正好停在西跨河桥位 (−6.2,17)** | **(−9.5,17) / (16.0,17)** |
| `Outskirts` | r∈[34,58] | r=34 会压到 `industrial_park` 外角（最远角半径 36.88） | **r∈[38,58]** |
| `SportsField` | (−16,22) 5.5×3.5 | 尺寸与教育园区底板仅 0.25 间隙 | **(−16.5,22) 7×4.5** |

> 其余 9 项（PortTerminal / HeliPad / Substation / WaterTower / CommTower / FireStation /
> IntersectionSignals / ParkExtras / RailViaduct）初版坐标预检即通过，未改。

## 6. 确定性布局约定

- 唯一随机源：`hashStr(s) + mulberry32(seed)`（`DistrictBlock.tsx` 已导出同款，
  本批在 `components/wealth/civic/rand.ts` 复制一份 24 行实现，**不跨文件 import
  DistrictBlock**（避免循环依赖），并在注释标注「与 DistrictBlock 同源实现」）。
- seed 取值：设施 id（`'outskirts-v1'` / `'containers'` / `'trees-v3'` …），**不用时间戳/ Math.random**。
- 同 seed 重渲染 / 刷新 / 换机型，布点完全一致（验收清单断言）。

## 7. 不破坏的契约

| 对象 | 约定 |
|---|---|
| `WealthCityMap` 对外 props | 不变 |
| `useSharedTexture` 既有签名 | 不变（新增 `useSharedPBR` 并列） |
| `SharedTextureOpts` | 不变 |
| `BuildingSpec` / `ShapeProps` | 只追加可选字段 |
| `Vehicle` props | 只追加可选 `palette` |
| `props/Pedestrian*.tsx` V1/V2 | 保留文件不删（`StreetPropsLayer` 改为引用 V3） |
| 后端 / WS / 协议 / i18n | **零改动** |
| `styles/globals.css` @import 链 | 只在 `wealth-city3d.css` 内追加，不改链序 |

## 8. 测试与验收

编译门禁 + 功能验收 + 视觉回归实录见 [`05-路线图与验收-v1.md`](05-路线图与验收-v1.md)。
资产生成与确定性断言见 [`03-资产方案-v1.md`](03-资产方案-PBR法线粗糙度与程序化材质-v1.md)。
