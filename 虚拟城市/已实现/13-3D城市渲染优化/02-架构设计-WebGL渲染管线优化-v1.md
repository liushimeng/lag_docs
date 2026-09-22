# 虚拟城市 — WebGL 渲染管线优化 · 架构设计（13-3D城市渲染优化 · 02）

> 2026-09-22。职责线：frontend-dev（工作面 `ClientWeb/src/components/wealth/**`、
> `ClientWeb/src/pages/WealthGamePage.tsx`、`ClientWeb/src/styles/wealth*.css`）。
> 产品设计见 `01-产品设计-真实城市渲染升级-v1.md`；资产见 `03-资产生成方案-v1.md`。

## 1. 渲染管线（阶段 C）

### 1.1 Canvas 级

`WealthCityMap.tsx` `<Canvas>` 增强：

```tsx
<Canvas
  shadows                    // 保留；three 0.169 默认 PCFShadowMap → 显式改 PCFSoft
  dpr={[1, 2]}               // 保留
  camera={{ position: CAMERA_START, fov: 45 }}
  gl={{ antialias: true }}
  onCreated={({ gl }) => {
    gl.toneMapping = THREE.ACESFilmicToneMapping;   // 胶片曲线，高光不过曝
    gl.toneMappingExposure = 1.05;
    gl.shadowMap.type = THREE.PCFSoftShadowMap;     // 柔和阴影边缘
  }}
>
```

### 1.2 天空与背景

- 用 drei `<Sky />`（drei 9.114 已含，`three-stdlib` 依赖已随 drei 安装）：
  `sunPosition` 与主方向光方向一致（`[20, 32, 16]` 归一化 ×100），
  `turbidity={6} rayleigh={1.2}`，傍晚偏暖。
- `<color attach="background">` 删除（Sky 接管背景）；`<fog>` 颜色改为
  天际线色 `#aeb8c6`（与 Sky 地平线接近，远景自然消隐），`FOG_NEAR/FAR` 不变。
- Sky 不参与雾（Sky shader 自带），无冲突。

### 1.3 光照与阴影

| 项 | 现状 | 目标 |
|---|---|---|
| directionalLight | 1024 shadow map，无 bias | 2048，`shadow-bias={-0.0002}` `shadow-normalBias={0.02}`，阴影相机 `left/right/top/bottom = ±WORLD_SIZE*0.6` 覆盖全城 |
| ambientLight | 0.7 | 0.45（ACES 下压环境光，让阴影有层次） |
| hemisphereLight | 0.35 | 0.5（天空蓝/地面暖反弹） |
| 太阳色 | 默认白 | `#fff2e0` 暖白（傍晚） |

阴影相机必须显式给 `shadow-camera-*`，否则默认 ±5 只能罩住原点一小块
（现状 80×80 地图上大部分楼实际无阴影——隐形缺陷）。

## 2. 建筑形态系统（阶段 B）

### 2.1 Archetype 分派

`BuildingMesh` 拆分为体块组合器。新增文件
`ClientWeb/src/components/wealth/building_shapes.tsx`（与 BuildingMesh 同目录同职责，
防止单文件超 §4 上限）：

```ts
export type BuildingArchetype =
  | 'tower'      // 高层塔楼：裙楼(podium) + 塔身(tower) + 顶部收分(crown)
  | 'slab'       // 多层板楼：单 box + 檐口线
  | 'house'      // 坡屋顶别墅：box + 三棱柱屋顶（prism）
  | 'shed'       // 工业厂房：大跨平顶 + 山墙 + 烟囱/水塔
  | 'pavilion';  // 公园景观低层：平顶小品

export const DISTRICT_ARCHETYPE: Record<WealthDistrictId, BuildingArchetype> = {
  finance: 'tower', riverside: 'tower', medical_city: 'tower',
  tech: 'slab', hightech_park: 'slab', residential: 'slab',
  commerce: 'slab', edu_district: 'slab', transport_hub: 'slab',
  cultural_creative: 'slab',
  oldtown: 'house', suburb: 'house',
  industry: 'shed', industrial_park: 'shed', logistics_port: 'shed',
  central_park: 'pavilion',
};
```

### 2.2 体块构成（世界单位，尺寸经 `cityScale.u()` 换算）

- **tower**：裙楼高 = `u(10)`（约 3 层商业裙楼，占地 w×d 满铺）；
  塔身 = 总高 − 裙楼 − 收分，占地 0.8×；收分块高 `u(6)`，占地 0.55×。
  立面贴图：裙楼用 `facade_base`，塔身用 `facade_mid`，收分用纯色金属。
- **slab**：单 box + 顶部 0.05 高檐口条（深色），保留现有 4 面贴图逻辑。
- **house**：box 主体（高 ×0.7）+ 三棱柱坡屋顶（`prismGeometry` 自构 BufferGeometry，
  或用 `cylinderGeometry(r, r, depth, 3)` 旋转 90° 取三角棱柱），屋顶用 roof 贴图/红瓦色。
- **shed**：大跨 box（高 ×0.8，w×1.2）+ 山墙三角 + 1-2 根烟囱（细圆柱，高 ×1.3）。
- **pavilion**：低矮平顶 box（高 ≤ `u(9)`）+ 大挑檐。

### 2.3 立面 emissive（繁荣度夜景接口）

保留 `prosperity → emissiveIntensity` 语义，但 emissive 色统一 `#ffd9a0`（暖窗光），
强度上限 0.35，避免 ACES 下过曝。贴图存在时 `emissiveMap` 复用 facade 贴图
（窗格自带亮度差，近似夜景窗灯），无贴图时才用城区色 emissive。

### 2.4 中央公园特化

`DistrictBlock` 对 `central_park`：
- 楼群数量公式改为 1-2 栋 pavilion；
- `StreetPropsLayer` 对该区树数量 2 → 10（`treesForDistrict` 按 archetype 分支），
  树 scale 0.8~1.2，覆盖底板 >60% 视觉面积；
- 底板贴图 `districts/central_park.png` 为绿地+园路（阶段 A 生成）。

## 3. 街道细节（阶段 D）

### 3.1 crosswalk 接线（修复 G4 / §130）

`Road.tsx` 在**靠近 from 端（城区入口）**处加斑马线：
`position` 沿道路方向 `t = 0.08`，平面宽 = roadWidth、长 0.5，
贴图 `streetTileUrl('crosswalk')`，`transparent`，y = ROAD_Y + 0.002。
主干道与次干道都加。

### 3.2 红绿灯 prop

新增 `props/TrafficLight.tsx`：立杆（高 `u(6.5)`）+ 横臂 + 三色灯头
（三个 emissive 小球，红/黄/绿，静态绿亮）。布点：每条主干道 `t=0.12` 处路侧。
贴图可选（props/trafficlight/），无贴图纯几何即可接受。

### 3.3 车辆升级

`props/Vehicle.tsx`：车身 box 下加 4 个车轮（黑色扁圆柱 `u(0.35)` 半径），
车头 2 个暖白 emissive 小方块（前灯）、车尾 2 个红色 emissive（尾灯）。
尺寸遵守 cityScale（轿车 4.5×1.8×1.5m）。

### 3.4 行道树增密

`StreetPropsLayer` 每城区树 2 → 4（非公园区），布点沿城区边缘（半径 3.6±0.3），
避开路口 sign 位。mesh 预算复核见 §5。

## 4. UI 布局（阶段 E，规范见 04 文档）

- 右侧栏 9 Tab 分两行分组：「数据」（财务/行情/流水/经济/调研）+「交易」（挂单/借贷/信息/保险），
  `flex-wrap: wrap`，tab 行 `max-height` 两行，超出滚动。
- `wealth.css` 1738 行已逼近 §4 上限 → **新增样式一律进 `wealth-city3d.css`**，
  `globals.css` 在 wealth 系列末尾追加 `@import`（顺序追加，不改既有顺序）。
- 面板容器统一 `max-height` + `overflow-y: auto`；`Html` hover 卡加
  `zIndexRange` 上限防盖住小地图。

## 5. 性能预算与护栏

| 指标 | 现状估算 | 目标上限 |
|---|---|---|
| mesh 总数 | ~600 | ≤ 1500 |
| 贴图总数（首屏） | ~40 | ≤ 80（512-1024px，显存 < 300MB） |
| shadow map | 1024 | 2048（单张，可接受） |
| draw call | ~700 | ≤ 1800 |

- 树 ≥ 40 棵后仍用独立 mesh（每棵 2 mesh × 48 ≈ 96，预算内）；**不引入 instancing**
  （确定收益不足、增加复杂度，留作后续优化项）。
- 所有伪随机继续走 mulberry32 + hashStr（seed 稳定，§3 风格）。
- 每帧 React 重渲染禁令不变（动画只在 useFrame 内改 ref）。

## 6. 改动文件清单

| 文件 | 阶段 | 改动 |
|---|---|---|
| `3d_script/procedural_city_textures.py`（新） | A | 程序化纹理生成（PIL，无 API 依赖） |
| `python-generate-image-tool/generate_wealth_city_assets.py` | A | FACADES/ROOFS/DISTRICTS 清单 +8 城区 |
| `components/wealth/building_shapes.tsx`（新） | B | archetype 体块组合 |
| `components/wealth/BuildingMesh.tsx` | B | 改调 building_shapes，保留贴图加载 |
| `components/wealth/DistrictBlock.tsx` | B | central_park 特化、heightBase 适配 |
| `components/wealth/WealthCityMap.tsx` | C | Sky/ACES/阴影/光照 |
| `components/wealth/Road.tsx` | D | crosswalk 接线 |
| `components/wealth/props/TrafficLight.tsx`（新） | D | 红绿灯 |
| `components/wealth/props/Vehicle.tsx` | D | 车轮/车灯 |
| `components/wealth/StreetPropsLayer.tsx` | B/D | 树增密、公园特化、红绿灯布点 |
| `pages/WealthGamePage.tsx` | E | Tab 分组 |
| `styles/wealth-city3d.css`（新）+ `globals.css` | E | 布局样式 |

## 7. 风险与回退

- **Sky 与 fog 冲突**：Sky 不受雾影响是预期行为；若地平线突兀，调 `FOG_FAR` 至
  `WORLD_SIZE*1.8` 或 `rayleigh` 加大。
- **ACES 下旧贴图变暗**：`toneMappingExposure` 1.05→1.15 区间内微调，不许超过 1.2。
- **贴图缺失降级链不变**：任何新贴图缺失 → 纯色/简化几何兜底（§9 策略不变）。
- 每阶段独立可回退：A 纯增量文件；B/C/D/E 按文件边界拆分提交。
