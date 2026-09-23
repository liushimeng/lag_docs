# 虚拟城市 — PBR 法线粗糙度与程序化材质 · 资产方案（18 · 03）

> 2026-09-23。读者：art-designer / frontend-dev。
> 生成器：[`3d_script/procedural_pbr_maps.py`](../../../../3d_script/procedural_pbr_maps.py)（**已交付并实测**）。
> 前置资产方案：13/03（城区贴图）+ 14/03（地表铺装）+ 16/03（云层与底色）。

---

## 1. 资产总账

| 轨道 | 产出 | 数量 | 体积 | 状态 |
|---|---|---|---|---|
| **轨道 2 · 程序化**（本批唯一轨道） | `pbr/**` 法线 + 粗糙度 | 161 张 | **9.2 MB** | ✅ 已生成 |
| 轨道 1 · AI 图像（`python-generate-image-tool`） | 本批**不产出** | 0 | 0 | 见 §5 取舍 |

### 1.1 目录结构（落在 `ClientWeb/src/assets/images/wealth/pbr/`）

```
pbr/
├── facades/    <id>_{base,mid}_n.png   256×512   32 张   楼宇立面法线
│               <id>_{base,mid}_r.png   128×256   32 张   立面粗糙度（灰度）
├── roofs/      <id>_n.png              256×256   16 张   屋顶法线
│               <id>_r.png              128×128   16 张   屋顶粗糙度
├── streets/    <name>_n.png            256×256    6 张   路面法线（人行道源 512×256 → 256×128）
│               <name>_r.png            128×128    6 张
├── ground/     <name>_n.png            256×256    4 张   地表法线
│               <name>_r.png            128×128    4 张
├── districts/  <id>_n.png              256×256   16 张   城区底板法线（源 1024² 降 1/4）
│               <id>_r.png              128×128   16 张
└── synth/      water_n.png             512×512    1 张   水面波纹法线（无 _r）
                concrete_{n,r}.png      512×512    2 张   清水混凝土
                brick_{n,r}.png         512×512    2 张   砖墙
                metal_deck_{n,r}.png    512×512    2 张   压型钢板
                tile_roof_{n,r}.png     512×512    2 张   瓦屋面
                asphalt_wear_{n,r}.png  512×512    2 张   轮胎磨耗
                foliage_{n,r}.png       512×512    2 张   树冠起伏
```

合计 161 张（64 + 32 + 12 + 8 + 32 + 13）。

## 2. 派生算法（颜色贴图 → 高度 → 法线 / 粗糙度）

> **核心决策**：法线与粗糙度**不独立绘制**，而是从既有颜色贴图求高度场派生。
> 好处：窗洞 / 砖缝 / 标线与颜色图**像素级天然对齐**，杜绝「凹凸错位」这一类
> AI 生成贴图最常见的硬伤；且对 16 城区 × 2 段共 32 张来源各异（AI 轨道 / 程序轨道）
> 的立面统一有效。

### 2.1 高度场 `height_from_color(img, mode)`

```
lum    = 0.299R + 0.587G + 0.114B          # alpha 低的像素按中性灰 0.5 混合
smooth = box_blur(lum, r=2)                # 可平铺盒式模糊（两趟，周期边界）
detail = lum − smooth                      # unsharp 细节（窗框 / 砖缝 / 颗粒）

base   = normalize01(smooth)  或  1 − normalize01(smooth)   # 由 mode 决定
height = normalize01(base + sharp · detail · 4)
```

| mode | 用途 | 明暗语义 | sharp |
|---|---|---|---|
| `wall` | 立面 | 亮墙 = 高、暗窗 = 低（**内凹**） | 1.15 |
| `street` | 路面 | 亮标线 = 略低（漆膜比沥青薄）→ `base = 1 − lum` | 0.45 |
| `roof` | 屋顶 | 亮设备 / 瓦楞 = 高 | 0.75 |
| `ground` | 地面 / 城区底板 | 缝隙暗 = 低 | 0.55 |

### 2.2 法线 `normal_from_height(height, strength)`

标准 Sobel 求梯度（**周期边界**，保证可平铺无缝）：

```
dx = Sobel_X(height) ; dy = Sobel_Y(height)
n  = normalize(−dx·strength, −dy·strength, 1)
RGB = n × 0.5 + 0.5                # 切线空间法线，A=255
```

- 降采样在**求高度之后**做（先 512² 求高度 → 缩到 256² → 求梯度），
  比「先缩图再求梯度」更稳：缩图的锯齿边缘不会被误判成陡坡。
- `strength × n_scale` 补偿降采样带来的梯度衰减。
- 实测强度（mean |nx| 偏离 128 的均值）：立面 28.8 / 瓦 57.1 / 压型钢板 71.1 /
  砖 17.0 / 沥青 3.4 / 水 7.5 —— 高频材质强、平面材质弱，符合物理直觉。

### 2.3 粗糙度 `roughness_from_color(img, mode)`

单通道灰度 PNG（three 采 G 通道）：

| mode | 公式 | 物理含义 |
|---|---|---|
| `wall` | `0.28 + 0.58·lum` | 玻璃（暗）滑 0.28 ～ 墙体（亮）糙 0.86 |
| `street` | `0.92 − 0.42·lum` | 沥青糙 0.92 ～ 标线漆面 0.50 |
| `roof` | `0.55 + 0.30·lum` | 0.55 ～ 0.85 |
| `ground` | `0.72 + 0.22·lum` | 0.72 ～ 0.94 |

> 合成材质的粗糙度由高度反推：`rough = rough_hi − (rough_hi − rough_lo)·height`
> （凸起磨亮更滑、凹陷积灰更糙）。

## 3. 合成材质（`pbr/synth/`，与颜色贴图无关的可平铺通用材质）

| 名 | 高度构造 | rough 区间 | 挂到 |
|---|---|---|---|
| `water_n` | 三层正弦波（`3u+v` / `5v−2u` / `8u+7v`）+ fbm 扰动 | —（材质常量 0.08） | 水面双层滚动 |
| `concrete` | fbm 5 octave + 每 128px 十字模板缝（缝处 ×0.72） | 0.62–0.88 | 桥墩 / 码头 / 挡墙 / 女儿墙 |
| `brick` | 错缝砖列（行高 32 / 砖宽 64 / 灰缝 3）+ 单砖微凸面 | 0.70–0.90 | 消防站 / 警局 / 市政厅 / 角柱 |
| `metal_deck` | 纵向瓦楞（每 32px 一棱）+ 每 128px 横向搭接缝 | 0.22–0.48 | 厂房屋面 / 集装箱 / 岸吊 |
| `tile_roof` | 筒瓦行列（每 40px 半圆棱）+ 每行叠瓦缝 | 0.55–0.85 | 别墅坡顶（无贴图时） |
| `asphalt_wear` | fbm + 双轮迹光带（x=0.28 / 0.72 处 −18%） | 0.55–0.88 | 主干道补强（可选叠加） |
| `foliage` | fbm 5 octave，`base^1.4` 起伏 | 0.72–0.95 | 树冠 |

**可平铺保证**：`_tile_noise` 用 `cells×cells` 随机控制点 + **周期闭合**（第 0 行/列复制到第 cells 行/列）
再 smoothstep 双线性；正弦项在 `0..2π` 上周期整数倍；Sobel 用 `% w / % h` 环绕取样。
三重保障使任意 repeat 下无接缝。

## 4. 确定性与体积控制

### 4.1 确定性

| 机制 | 说明 |
|---|---|
| seed | `FNV-1a(文件名)`（`fnv1a`），与 `procedural_city_textures.py` 同源实现 |
| 禁 `random` 全局态 | 合成材质的噪声走自实现 LCG（`state = 1103515245·state + 12345`）或 `random.Random(seed)`，不调用 `random.random()` |
| 幂等 | `skip-if-exists`；`--force` 全量重生成 |
| **验收断言** | `--force` 重跑两次 → `md5sum` 全量比对一致（已实测通过） |

### 4.2 体积 / 显存控制（实测值）

| 策略 | 效果 |
|---|---|
| 法线 **半分辨率**（`DERIVED_N_SCALE=2`） | 立面 512×1024 → 256×512 |
| 城区底板法线 **1/4 分辨率**（`DERIVED_N_SCALE×2=4`） | 1024² → 256²，16 张从 6.8 MB → 2.3 MB |
| 粗糙度 **1/4 分辨率 + 灰度 `L` PNG** | 单文件 ≈ 3–8 KB |
| 合成材质 512²（本就要平铺细节） | 908 KB / 13 张 |
| **合计** | **9.2 MB 磁盘 / ≈ 24 MB GPU**（未做半分辨率前为 28 MB / 64 MB） |

> 首次调仓实测：全分辨率 RGBA 法线 28 MB；灰度粗糙度 + 半分辨率后 14 MB；
> 城区底板再降 1/4 后 9.2 MB。**不要再把法线提回全分辨率**——256×512 在
> 1 单位 = 10 m 的标尺下已超出屏幕像素密度。

## 5. 为什么本批不用 `python-generate-image-tool`（轨道 1）

| 判据 | 结论 |
|---|---|
| 法线 / 粗糙度需要**与颜色图像素对齐** | AI 生成无法对齐既有窗格 → 必然凹凸错位；派生法完美规避 |
| 合成材质需要**可平铺无缝** | AI 生成的 tileable 保证弱；程序化正弦/周期噪声天然无缝 |
| 确定性重跑字节一致 | AI 轨道无法保证 |
| 网络 / API 依赖 | 程序轨道零依赖（纯 Pillow），符合 13/03「轨道 2」定位 |

**保留轨道 1 的适用场景**（本批不做，留作后续）：

- 16/05 §5 已知边界提到的「等距 AI 立面重绘」——那是**颜色图**重绘，属另一专项；
  需要新的窗格布局时，应同步重跑本脚本派生法线（一条命令）。
- 大厅 banner / 职业头像 / 招牌画面等**具象插画**，仍走 `generate_wealth_*.py`。

## 6. 生成器 CLI 契约

```bash
python3 3d_script/procedural_pbr_maps.py                 # 补齐缺失
python3 3d_script/procedural_pbr_maps.py --force         # 全部重生成
python3 3d_script/procedural_pbr_maps.py --only synth    # 只跑合成材质
python3 3d_script/procedural_pbr_maps.py --only derive   # 只跑派生材质
python3 3d_script/procedural_pbr_maps.py --only districts,facades   # 按派生子目录
```

| 参数 | 语义 |
|---|---|
| `--force` | 覆盖已存在文件 |
| `--only` | 逗号分隔：`derive` / `synth` / `facades` / `roofs` / `streets` / `ground` / `districts`。缺省 = 全部 |

**运行时长**：全量 ≈ 4–6 min（纯 Python Sobel）；`--only districts` ≈ 1.5 min；`--only synth` ≈ 20 s。
超时不要杀进程——是在跑像素循环，不是卡死。

**与 `procedural_city_textures.py` 的关系**：两脚本**并列维护**、互不 import
（共享的 `fnv1a` 各自复制 8 行，避免 `3d_script/` 变成包）。
新增颜色贴图时：**先跑 `procedural_city_textures.py`，再跑 `procedural_pbr_maps.py`**（派生依赖颜色图存在）。

## 7. 降级链

| 缺失 | 行为 |
|---|---|
| `pbr/**` 整目录 | 全城保持 16 批次的材质外观（纯色 / 单贴图 + 硬编码 roughness），零报错 |
| 单张 `_n` | 该材质无凹凸；颜色与粗糙度照常 |
| 单张 `_r` | 该材质用代码里的硬编码 roughness |
| `synth/water_n` | 水面退回单层静态（无滚动细波层） |
| 颜色贴图缺失（既有链） | 不变：立面退城区主色、地面退沥青 → 纯色 |

> 验收要求：`rm -rf ClientWeb/src/assets/images/wealth/pbr` 后 `npm run build` + 运行
> 无 console error、全城可见，再恢复目录。

## 8. 新增素材时的检查清单

1. 颜色贴图放对应目录（`facades/` / `roofs/` / …）或在 `synth` 新增生成函数。
2. 跑 `procedural_pbr_maps.py` 补齐（`--only` 指定子集）。
3. `assets/images/wealth/index.ts` 确认 `pbrNormalUrl` / `pbrRoughUrl` 能解析到新 stem。
4. 在 `02-架构设计` §2.3 材质接线表登记（含 `normalScale`）。
5. `git grep` 确认接线（§130）。
6. 视觉验收：低角度看该材质有凹凸明暗变化（不是平贴纸）。
