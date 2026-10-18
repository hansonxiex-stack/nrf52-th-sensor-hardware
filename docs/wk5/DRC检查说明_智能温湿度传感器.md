# DRC 检查说明

> 项目：开源智能温湿度传感器 Pro PCB 设计
> 姓名：谢天成
> 检查工具：KiCad 10.0 · Design Rules Checker
> 报告时间：2026-09-27 20:38:01
> 对应文件：`正式工程_方案B.kicad_pcb`

---

## 1. 检查配置

### 1.1 设计规则（摘自 `.kicad_pro`）

| 规则 | 数值 |
|---|---|
| 最小走线宽度 | 0.20 mm |
| 最小过孔直径 | 0.60 mm |
| 最小过孔环宽 | 0.20 mm |
| 最小通孔直径 | 0.30 mm |
| 最小孔间距（hole to hole） | 0.25 mm |
| 最小孔到铜间距 | 0.25 mm |

### 1.2 严重级别设置

本项目将电气类违规统一设为 **error**，外观类设为 **warning**：

| 级别 | 检查项 |
|---|---|
| **error** | `shorting_items`、`clearance`、`tracks_crossing`、`items_not_allowed`、`courtyards_overlap`、`pth_inside_courtyard`、`npth_inside_courtyard`、`unconnected_items`、`drill_out_of_range`、`annular_width`、`solder_mask_bridge`、`copper_edge_clearance`、`invalid_outline`、`zones_intersect` … |
| **warning** | `lib_footprint_mismatch`、`silk_overlap`、`silk_over_copper`、`silk_edge_clearance`、`hole_to_hole`、`isolated_copper`、`track_dangling`、`via_dangling`、`duplicate_footprints` … |

> 该分级方式使 **error 全部对应真实电气缺陷**，warning 对应可接受的外观/一致性提示。

---

## 2. 检查结果总览

```
** Found 4 DRC violations **
** Found 0 unconnected pads **
** Found 0 Footprint errors **
```

| 检查项 | 结果 | 判定 |
|---|---|---|
| `shorting_items`（短路） | **0** | ✅ |
| `clearance`（间距） | **0** | ✅ |
| `tracks_crossing`（走线交叉） | **0** | ✅ |
| `items_not_allowed`（禁布区侵入） | **0** | ✅ |
| `courtyards_overlap`（外框重叠） | **0** | ✅ |
| `pth_inside_courtyard` | **0** | ✅ |
| `unconnected_items`（未连接） | **0** | ✅ |
| `drill_out_of_range`（孔径超范围） | **0** | ✅ |
| `annular_width`（环宽不足） | **0** | ✅ |
| `isolated_copper`（孤立铜） | **0** | ✅ |
| `via_dangling`（悬空过孔） | **0** | ✅ |
| **error 级合计** | **0** | ✅ |
| **warning 级合计** | **4** | 见第 3 章，均为预期内 |

**全部 4 条均为 warning 级，无一条 error。**

---

## 3. 逐条说明

### 3.1 `lib_footprint_mismatch` × 3

#### 3.1.1 这条检查的含义

`lib_footprint_mismatch` 比较的是**板上封装实例的参数**与**库中同名封装的参数**是否逐字一致。

报告中每条后面的 **`Local override; warning`** 是 KiCad 对差异性质的判定：

> **Local override** = 板上副本与库不同，且**以板上副本为准**。

即：KiCad 已识别出这是**主动修改**，而非文件损坏。该提示的作用是提醒"若执行『从库中更新封装』会覆盖这些修改"。

**三条差异全部属于此类，且都是必要修改。**

#### 3.1.2 ① U2 — SHT4x 温湿度传感器

| 项目 | 内容 |
|---|---|
| 封装 | `Sensor_Humidity:Sensirion_DFN-4_1.5x1.5mm_P0.8mm_SHT4x_NoCentralPad` |
| 差异 | 板上实例内嵌一个 **四层 keepout zone**（F.Cu / B.Cu / In1.Cu / In2.Cu），系统库中不存在该 zone |
| 依据 | SHT4x Datasheet v7.3 §5.3 Land Pattern：<br>*"There shall be no copper under the sensor other than at the pin pads."*<br>要求为**全层**无铜 |
| 变更周次 | 第 4 周（变更记录 4-1） |

**该 zone 位于 `.kicad_pcb` 的封装块内部（板级实例），非库文件**。系统库 `Sensor_Humidity.pretty` 保持只读、未被改动。

> ⚠️ **这是三条中后果最严重的一条**：执行"从库中更新封装"会**整体删除该 keepout**，且**不会产生任何额外的 DRC 告警**——删除后 DRC 仍然通过，但传感器的热隔离设计已经失效。

#### 3.1.3 ② AE1 — 2.4 GHz 板载天线

| 项目 | 内容 |
|---|---|
| 封装 | `RF_Antenna:Texas_SWRA117D_2.4GHz_Left` |
| 差异 | 手工修改 **pad 2 的环形宽度：0.10 mm → 0.25 mm** |
| 依据 | 库中 0.10 mm 低于常见 PCB 厂家的最小环宽要求（通常 0.15 ~ 0.20 mm） |
| 变更周次 | 第 4 周（变更记录 4-10） |

加宽后满足可制造性要求，不影响天线电气性能（pad 2 为天线馈电点）。

#### 3.1.4 ③ U3 — HT16C21A LCD 驱动

| 项目 | 内容 |
|---|---|
| 封装 | `Package_SO:SSOP-28_3.9x9.9mm_P0.635mm` |
| 差异 | 库中 28 个焊盘的**局部角度为 0°**，板上实例为 **270°** |
| 依据 | **修正焊盘重叠错误**。U3 以 90° 旋转置于 B.Cu；若焊盘维持库中的 0°，其 1.2 mm 长边将落在 0.635 mm 的引脚节距方向上，导致相邻焊盘互相重叠，产生 **50 条 `shorting_items` / `clearance` 违规**。旋转焊盘后 50 条违规全部清零 |
| 变更周次 | 第 4 周（变更记录 4-4） |

**该修正是必要的，不是可选的。** 换型后的焊盘几何已通过三项独立验证：

| 验证项 | 结果 |
|---|---|
| 焊盘尺寸 | 1.2 × 0.4 mm；沿 0.635 mm 节距方向的宽度为 **0.4 mm** |
| 相邻焊盘缝隙 | **0.235 mm**（= 0.635 − 0.4），与第 4 周报告 §2.5 的记录一致 |
| 焊盘中心间距 | 14 个焊盘 x 向间距 **0.635 mm**、两排间距 **5.2 mm**，与库定义逐项相同 |
| 连接性 | 28 个焊盘**每个都有走线精确落于焊盘中心**，DRC 无任何间距或短路违规 |

> **反证**：若焊盘在节距方向真为 1.2 mm 宽，相邻焊盘将重叠 0.565 mm；而这些焊盘分属 VDD / SDA / SCL / COM 等不同网络，DRC 必然报错。当前不报，说明几何正确。

#### 3.1.5 三条差异的统一结论

> **U2 / AE1 / U3 三个器件，任何时候均不得执行「Update Footprint from Library…」。**
>
> 其余 33 个封装为库封装的原样引用，可正常更新。

这三条 `lib_footprint_mismatch` 在每次 DRC 中都会出现，**属于预期内的固定提示，不应试图消除**——消除它意味着放弃上述三项必要修改。

### 3.2 `silk_over_copper` × 1

```
[silk_over_copper]: Silkscreen clipped by solder mask
    @(123.0000 mm, 86.4000 mm): Segment of LCD1 on B.Silkscreen
    @(122.0000 mm, 86.4000 mm): PTH pad 1 [COM0] of LCD1
```

| 项目 | 内容 |
|---|---|
| 现象 | LCD1 外框丝印（B.Silkscreen）与 LCD1 自身的 1 号通孔焊盘（COM0）重叠 |
| 性质 | **元件自身的图形压自身的焊盘**，不涉及任何外部器件 |
| 电气影响 | **无** |
| 处理 | **不处理**，理由如下 |

**三条理由：**

1. **阻焊开窗处本就无法印刷丝印**。PCB 厂家的 CAM 工序会自动裁除与阻焊开窗重叠的丝印段，这是行业标准处理，无需在源文件中修改。
2. **该线段位于 LCD 本体正下方**，装配完成后完全被玻璃覆盖，裁除与否均不可见。
3. 与任何其他器件或网络无关，不构成装配或电气风险。

> 若需要 DRC 报告全零，删除该段丝印即可，无任何后果。当前判定为无需处理。

---

## 4. 被忽略的检查（Ignored checks）

DRC 报告末尾列出 5 项被设为"忽略"的检查。以下逐条说明处置理由。

| # | 检查项 | 忽略理由 |
|---|---|---|
| 1 | **Footprint has no courtyard defined**<br>`missing_courtyard` | 板上 **36 个封装全部带 courtyard**（已逐个核对）。该检查在本板**开启或关闭均不产生任何结果**。保持忽略是为了避免后续引入的机械件、Mark 点等无 courtyard 器件刷出无意义提示。 |
| 2 | **Track endpoint not centered on via**<br>`track_not_centered_on_via` | 全板共 70 处走线端点未精确落于过孔中心（GND 41 处、信号网 29 处），最大偏移 **0.35 mm**（信号网最大 0.283 mm）。**过孔外径 0.7 mm，半径即 0.35 mm**——这些端点**仍落在过孔环内**，连接成立。密集区域中走线以 45° 斜向进入过孔属常规做法。`unconnected_items = 0` 与 `track_dangling`（warning 级、0 条）共同证明连接无误。 |
| 3 | **Tuning profile track geometries**<br>`tuning_profile_track_geometries` | 本板**无任何差分对，也未做等长/调谐**。射频链路为**单端 50 Ω 微带**，不是差分对；I2C 运行于 100 kHz，不需等长。本板不存在调谐轮廓，该检查无作用对象。 |
| 4 | **Footprint doesn't match symbol's footprint filters**<br>`footprint_filters_mismatch` | 该检查仅在**原理图 → PCB 同步**（Update PCB from Schematic）时起作用，用于提示符号库中登记的候选封装与实际选用不一致。本板封装在布局阶段已逐个手工指定，板文件中即为最终封装，**对已完成的板文件无影响**。 |
| 5 | **Footprint component type doesn't match footprint pads**<br>`footprint_type_mismatch` | 全板仅 **AE1** 会触发：其 **pad 1 为 `connect` 类型铜皮**（天线辐射体）、**pad 2 为 `thru_hole`**。该混合焊盘类型是 TI SWRA117D 参考天线封装**固有的结构**，来自 KiCad 官方 `RF_Antenna` 库且未作改动，不属于选型错误。 |

---

## 5. 关于过孔与焊盘的间距判据

第 5 周处理了 7 处过孔与焊盘距离过近的问题，此处记录判据，便于复核。

**判据分两个圈看：**

| 关系 | 判定 | 原因 |
|---|---|---|
| **环压着焊盘** | **允许** | 同网络下环与焊盘重叠只是"零长度连接"，无副作用 |
| **孔压着焊盘** | **不允许**，孔边距焊盘边须 **≥ 0.15 mm**（0.10 mm 为底线） | 孔与焊盘过近时，回流焊过程中**焊料会沿孔壁虹吸流失**，导致焊点虚空 |

**换算关系**：过孔外径 0.7 mm − 钻孔 0.3 mm = 环宽 0.4 mm，两侧各 0.2 mm。

```
孔净距 = 环净距 + 0.2 mm （恒定）
```

因此调整位置时应以**孔**的净距为控制量。

**执行结果**：7 个过孔重新定位至「孔净距 0.15 mm / 环压焊盘 0.05 mm」。

**已知例外 1 处**：C8 的 GND 过孔孔净距为 0.070 mm，为全板唯一低于 0.10 mm 者。该处几何受限：

```
L1 焊盘下沿 → C8 焊盘上沿  =  0.770 mm
0.7 mm 过孔 + 两侧各 0.2 mm 间隙  =  1.10 mm   （放不下）
```

且东侧有 SWDIO 走线斜穿，无可挪位置。该处为已知例外，第 5 周已确认维持原位。

---

## 6. 附录：DRC 报告原文

```
** Drc report for 正式工程_方案B.kicad_pcb **
** Created on 2026-09-27T20:38:01 **
** Report includes: Errors, Warnings, Exclusions **

** Found 4 DRC violations **
[lib_footprint_mismatch]: Footprint 'Sensirion_DFN-4_1.5x1.5mm_P0.8mm_SHT4x_NoCentralPad' does not match copy in library 'Sensor_Humidity'
    Local override; warning
    @(140.8000 mm, 86.0000 mm): Footprint U2
[lib_footprint_mismatch]: Footprint 'Texas_SWRA117D_2.4GHz_Left' does not match copy in library 'RF_Antenna'
    Local override; warning
    @(150.1800 mm, 56.1250 mm): Footprint AE1
[lib_footprint_mismatch]: Footprint 'SSOP-28_3.9x9.9mm_P0.635mm' does not match copy in library 'Package_SO'
    Local override; warning
    @(131.5425 mm, 70.0000 mm): Footprint U3
[silk_over_copper]: Silkscreen clipped by solder mask
    Local override; warning
    @(123.0000 mm, 86.4000 mm): Segment of LCD1 on B.Silkscreen
    @(122.0000 mm, 86.4000 mm): PTH pad 1 [COM0] of LCD1

** Found 0 unconnected pads **

** Found 0 Footprint errors **

** Ignored checks **
    - Footprint has no courtyard defined
    - Track endpoint not centered on via
    - Tuning profile track geometries
    - Footprint doesn't match symbol's footprint filters
    - Footprint component type doesn't match footprint pads

** End of Report **
```

---

*说明完*
