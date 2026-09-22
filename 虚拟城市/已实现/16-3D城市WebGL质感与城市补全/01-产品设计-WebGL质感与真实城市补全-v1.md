# 虚拟城市 — WebGL 质感与真实城市补全 · 产品设计（16 · 01）

> 2026-09-22。前置：15-3D城市全面真实感深化（阶段 K–Q 已交付：行人 V2 / 树木 V2 /
> 街道家具 V2 / 建筑细节 / 道路标线 / 氛围层 / UI 密度）。
> 契约文档：02 架构设计 / 03 资产方案 / 04 UI 布局 / 05 路线图。
> 目标一句话：**从「街道有细节」迈向「城市有骨架」**——
> 补齐环形路网、跨河桥梁、双向车道等功能性城市结构，同时把 WebGL 材质质感
> （环境反射 / 各向异性过滤 / 楼宇色彩分化）与天空氛围（云层）拉满。

---

## 1. 现状差距（15 轮交付后距「真实城市」仍缺）

| 维度 | 15 轮已达成 | 距「真实城市」仍缺 |
|---|---|---|
| **路网结构** | 放射主干道（15 条）+ 车道箭头 + 斑马线 | 无环路 / 无城区间联系道路——真实城市必有「环线 + 跨河桥」骨架 |
| **跨水交通** | 运河 + 港池水面 | 教育/医疗城道路直接「压」在运河上（无桥）——车在水上开 |
| **交通流** | 每主干道 1 车沿中线行驶 | 车压中线、单方向——真实城市双向车道、右侧通行 |
| **材质质感** | meshStandard + 立面贴图 + ACES | 无环境反射（玻璃幕墙死黑）；地面贴图各向异性糊；楼群同贴图同色一片单调 |
| **天空** | drei Sky 天空穹顶 | 万里无云——真实城市天空有云 |
| **公园** | 草地 + 树群 + 落叶粒子 | 无喷泉 / 无园路 / 无花坛——中央公园缺「可游」感 |
| **城市生活** | 街道家具 + 底商 | 无停车场、无施工工地（塔吊是城市天际线标配）、住宅屋顶无太阳能板 |
| **UI** | Tab 6 个 + 折叠面板 | 小地图常驻不可收起（小屏遮挡视野）；侧栏滚动区未显式收口 |

## 2. 产品目标（可感知的 8 条）

1. **环线成形**：CBD 外围出现环形道路，放射干道与之相交，「环线 + 放射」骨架一目了然。
2. **有桥**：运河上两座桥（教育园区 / 医疗城方向），桥面、栏杆、桥墩齐全，车不再开进水里。
3. **双向车道**：车辆右侧通行、对向车流并行，同向多车不叠影。
4. **玻璃会反光**：金融城幕墙反射天空色（PMREM 环境贴图），不再是死黑面片。
5. **天上有云**：5–7 团慢速漂移的云，地平线不再空旷。
6. **公园可游**：喷泉 + 十字园路 + 花坛，中央公园从「草地」变「公园」。
7. **城市在生长**：文创区旁出现塔吊施工工地；金融/商业有划线停车场；郊区屋顶有太阳能板。
8. **楼群有表情**：同贴图楼栋带确定性色相微差（±8%），打破「复制粘贴」感。

## 3. 功能规格

### 3.1 渲染管线质感（阶段 R · RenderQuality）

| 项 | 规格 |
|---|---|
| 环境反射 | PMREMGenerator.fromScene(Sky) 生成 env map → `scene.environment`；玻璃/金属材质（crown / billboard / 车身）自动获得反射；仅生成 1 次（帧=1），组件卸载 dispose |
| 填充光 | 反方向冷色平行光 `#b8cce8` intensity 0.30 不投影——抬亮背光面，消除「阴阳脸」 |
| 各向异性 | textureCache 对 repeat 贴图设 `anisotropy = 8`（三内部 clamp 到 GPU 上限）——路面/地面掠射角清晰 |
| 楼宇色相分化 | BuildingShape 按占地 (w,d) 确定性 hash → `color` 乘 0.92–1.08 明度微差；不引随机源 |
| envMapIntensity | 立面材质 0.5 / 金属件 0.8 / 车身 0.6；贴图缺失降级链不变 |

### 3.2 路网骨架（阶段 S · RoadNetwork）

| 项 | 规格 |
|---|---|
| 环形路 | 16 边形近似圆，半径 5.6（CBD 8×8 底板外缘 4.05 之外），路宽 1.0；沥青贴图 + 中线虚线段；y=0.017 防 z-fighting |
| 跨河桥 | 运河 z=17 与 edu/medical 放射干道交点 (-6.2, 17) / (6.2, 17) 各 1 座：桥面（长 4.6 宽 2.0，y=0.05）+ 两侧栏杆 + 4 桥墩；桥面元素确定性布点 |
| 双向车道 | 车辆 `laneOffset` prop：垂直行进方向右偏 0.32（右侧通行）；每主干道正向 1 辆 + len>15 的反向加 1 辆 |
| 环路元素 | 环路与放射干道交点外侧停止线（白条，确定性 8 处以内） |

### 3.3 功能地块与地标（阶段 T · Landmarks）

| 地块 | 形态 | 布点 |
|---|---|---|
| 中央喷泉 | 双层圆池（石色）+ 内水盘（water_tile）+ 中心柱 + 水花 Sparkles | central_park 正中 (0,-22) |
| 园路 | 十字 2 条（宽 0.5，plaza_tile）贯穿公园，与喷泉对中 | central_park |
| 花坛 | 4 组彩色小花簇（球体 × 3，红/黄/紫）沿园路 | central_park 四象限 |
| 施工工地 | 围挡 4 面 + 裸土面 + 塔吊（格构柱 + 起重臂 + 平衡臂 + 吊钩） | (28, -1)，文创区与交通枢纽之间空地 |
| 停车场 | 8 条白色划线 + 3 辆静态车（复用 Vehicle 色板、独立简化几何） | commerce 东北角 (12.4, -4.9) |
| 太阳能板 | 倾斜 20° 深蓝板 + 边框，郊区/老城区屋顶确定性布 1–2 块 | suburb / oldtown 楼顶 |

### 3.4 天空氛围（阶段 U · Sky）

| 项 | 规格 |
|---|---|
| 云层 | 6 团云，每团 3–4 片 Billboard 云朵（`sky/cloud_puff.png`，缺失 → 白色扁球 opacity 0.30）；高度 y=14–20；速度 0.15–0.3 单位/s 沿 +x 漂移，出界回绕 |
| reduced-motion | `prefers-reduced-motion` 时云静止 |
| 降级链 | 贴图缺失 → 纯几何白雾球；两者皆可关闭（组件级） |

### 3.5 UI 布局加固（阶段 V · UI）

| 项 | 规格 |
|---|---|
| 小地图可折叠 | 右上角「⤫/⤢」toggle；折叠态缩为「🗺 地图」pill 按钮；localStorage `wealth.ui.minimap` |
| 侧栏滚动收口 | `.wealth-sidebar` flex 布局下 tabs 固定 + panels 区 `min-height:0; overflow-y:auto`——窄屏不把聊天面板挤出视口 |
| 事件流防遮 | MonthTicker 高度上限 + 内滚动（已有则验证） |
| 空态 | 等待棋盘 spinner 居中（阶段 J 已做，验证不回归） |

### 3.6 性能护栏

| 指标 | 上限 |
|---|---|
| 全城 mesh 数 | ≤ 3000（15 轮末约 900 + 环路 32 + 桥 2×9 + 地标 ~70 + 云 ~22 ≈ 1160，充裕） |
| 实时光源 | 1 日光 + 1 填充光 + 半球 + 环境；广告牌 pointLight 保持 w>1.6 稀疏上限 |
| PMREM | 仅启动时 1 次（cubeUV 尺寸 256），无每帧开销 |
| 云更新 | useFrame 每帧仅改 group position（无材质重建） |
| 帧率 | 1080p 中端 GPU ≥ 45fps |

## 4. 非目标（本轮不做）

- 昼夜循环 / 夜景模式（另立专项）
- 树 / 路灯 InstancedMesh 化（15 轮 §9.1 已论证收益有限，维持）
- 发光窗贴图 `*_emit.png`（AI 立面窗口位置不可对齐，做出来必然错位——技术方案否决，见 03 §4）
- 雨雪天气 / 路面湿滑
- 后端 / WS / 经济引擎改动（纯前端 + 3d_script 资产）

## 5. 验收体感标准

### 5.1 视觉验收

1. 环路绕 CBD 一周，与 ≥ 6 条放射干道相交，路面沥青材质与主干道一致。
2. 运河上 2 座桥，车辆过桥不沉入水面。
3. 同一主干道可见对向 2 车并行，各行其道（不压中线）。
4. 金融城塔楼幕墙映出天空色（旋转相机时高光随视角移动）。
5. 天空至少 4 团云缓慢漂移。
6. 中央公园：喷泉居中 + 十字园路 + 彩色花坛。
7. 文创区旁可见塔吊；商业中心旁可见划线停车位与静态车；郊区至少 1 块屋顶太阳能板。
8. 同城区相邻两栋同贴图楼颜色深浅略有差异。

### 5.2 UI 验收

1. 1280×800 下小地图可折叠/展开，折叠后不再遮挡地图视野。
2. 侧栏 Tab 常显，面板区独立滚动，聊天面板入口始终可见。
3. 全部控件不重叠、不出界、无大面积留白。

### 5.3 性能验收

1. `tsc --noEmit` + `npm run build` 通过。
2. 云层 / 车辆动画在 `prefers-reduced-motion` 下静止。
3. PMREM 仅初始化 1 次（Performance 面板无重复 GPU 上传毛刺）。

## 6. 阶段划分（共 5 阶段 + 1 整合）

| 阶段 | 名 | 核心交付 | 工作量 |
|---|---|---|---|
| R | 渲染质感 | PMREM env + 填充光 + anisotropy + 楼宇色相分化 | 中 |
| S | 路网骨架 | 环形路 + 2 跨河桥 + 双向车道 | 大 |
| T | 功能地标 | 喷泉/园路/花坛 + 塔吊工地 + 停车场 + 太阳能板 | 大 |
| U | 天空云层 | CloudLayer + cloud_puff 贴图 + 降级 | 小 |
| V | UI 加固 | 小地图折叠 + 侧栏滚动收口 | 小 |
| W | 整合验收 | 资产生成 + 全量 build + 视觉过检 | 中 |

## 7. 文件落点总览（§2.1 / §4 约束）

| 文件路径 | 动作 | 阶段 |
|---|---|---|
| `components/wealth/WealthCityMap.tsx` | 填充光 + EnvBinder 注入 | R |
| `components/wealth/EnvBinder.tsx` | 新：PMREM 环境绑定（一次性） | R |
| `components/wealth/textureCache.ts` | anisotropy 选项 | R |
| `components/wealth/building_shapes.tsx` | 楼宇色相分化 + envMapIntensity | R |
| `components/wealth/RingRoad.tsx` | 新：16 边环路 | S |
| `components/wealth/CanalBridge.tsx` | 新：跨河桥（可复用 2 处） | S |
| `components/wealth/props/Vehicle.tsx` | laneOffset prop | S |
| `components/wealth/StreetPropsLayer.tsx` | 双向车流编排 | S |
| `components/wealth/props/Fountain.tsx` | 新：喷泉 | T |
| `components/wealth/props/ParkGrounds.tsx` | 新：园路 + 花坛 | T |
| `components/wealth/props/ConstructionSite.tsx` | 新：塔吊工地 | T |
| `components/wealth/props/ParkingLot.tsx` | 新：划线停车场 + 静态车 | T |
| `components/wealth/props/SolarPanel.tsx` | 新：屋顶太阳能板 | T |
| `components/wealth/props/CloudLayer.tsx` | 新：云层 | U |
| `components/wealth/WealthMinimap.tsx` | 折叠 toggle | V |
| `styles/wealth-city3d.css` | 侧栏滚动收口 + minimap 折叠样式 | V |
| `3d_script/procedural_city_textures.py` | 扩展：sky/cloud_puff + ground/urban_base | U |
| `assets/images/wealth/index.ts` | sky / urban_base 访问器 | U |

## 8. 硬约束

- 单文件 ≤ 1800 行（§4）；Markdown ≤ 800 行（§3）
- 纯前端，ServerGo/ 不动
- 确定性伪随机（hash+mulberry32），禁 Math.random——刷新布局稳定
- 同源贴图走 textureCache.ts；禁组件自建 loader
- 降级链完整：云贴图缺失 → 几何兜底；urban_base 缺失 → asphalt_main → 纯色
- `prefers-reduced-motion` 全局兼容（云 / 车 / 喷泉水花静止）
- 所有新 helper grep 验证接线（§130）
