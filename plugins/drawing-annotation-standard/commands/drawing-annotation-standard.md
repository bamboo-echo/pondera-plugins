---
description: 按制图规范审计图纸标注、公差、尺寸链与图层线型
argument-hint: [DXF/DWG 图纸路径或图纸描述] [适用标准：GB / ISO / 企标]
---

审计 `$1` 的制图规范性，适用标准 `$2`（默认 GB/T 4458 系列 + 企标图层约定）。**本命令只产出检查清单与读图脚本，不修改图纸；DWG 需先由用户转 DXF。**

**执行步骤**

1. 用 ezdxf 读图，导出图层清单、线型、图元统计与所有标注实体的测量值。
2. 按下列清单逐项判定，标出具体图元句柄或位置。
3. 尺寸链校验：闭合尺寸链误差应为 0（公差范围内）；开放链必须有基准；重复标注同一尺寸视为冗余。
4. 公差检查：配合尺寸必须给公差或配合代号；未注公差需在技术要求中声明等级（如 GB/T 1804-m）。
5. 图层/线型检查：轮廓、中心线、虚线、剖面线、文字各自独立图层；中心线用 `CENTER`，虚线用 `DASHED`，不得混用。
6. 输出问题清单与修改建议；无法从 DXF 判断的（如线宽打印效果）标「需人工核对」。

**读图脚本模板**

```python
import ezdxf

PATH = "drawing.dxf"
doc = ezdxf.readfile(PATH)
msp = doc.modelspace()

print("=== layers ===")
for layer in doc.layers:
    print(f"{layer.dxf.name:20s} color={layer.dxf.color} linetype={layer.dxf.linetype}")

print("=== entity counts ===")
counts = {}
for e in msp.query("*"):
    counts[e.dxftype()] = counts.get(e.dxftype(), 0) + 1
for t, n in sorted(counts.items()):
    print(f"{t:16s} {n}")

print("=== dimensions ===")
for dim in msp.query("DIMENSION"):
    measurement = dim.dxf.get("actual_measurement", None)
    print(f"handle={dim.dxf.handle} type={dim.dimtype} layer={dim.dxf.layer} measure={measurement}")

print("=== linetype usage ===")
used = {}
for e in msp.query("*"):
    lt = e.dxf.get("linetype", "BYLAYER")
    used[lt] = used.get(lt, 0) + 1
print(used)
```

**输出格式**

```markdown
## 制图规范审计：<图纸>

**标准**：<GB / ISO / 企标>
**统计**：图层 <n>，标注 <m>，线型 <k>

| # | 类别 | 位置（句柄/图层） | 问题 | 判定 | 修改建议 |
| --- | --- | --- | --- | --- | --- |

### 尺寸链
| 链 | 组成尺寸 | 理论闭合 | 实际闭合 | 差值 | 结论 |

### 需人工核对
- <打印线宽、字体大小、图框比例 等>

### 脚本
```python
<读图脚本>
```
```

**纪律**

- 不自动修改图纸；只给清单与脚本。
- 无法从数据判断的项不得写成确定违规。
- 涉及安全相关尺寸（间隙、壁厚）时提示由设计负责人复核。
