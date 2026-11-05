---
name: ezdxf-drawing-script
description: 生成 ezdxf 脚本：按参数批量创建 DXF 图元、图层、文字与简单标注，或读回既有 DXF 做实体统计。当用户要脚本化出图、批量改图层、核对图纸内容时使用。本技能只产出脚本，运行需用户自备 Python 环境与 ezdxf 库，以及 AutoCAD/中望/DXF 查看器。
runas: inline
---

# ezdxf 脚本：读写 DXF

## 何时使用

- 同一张图要按参数批量生成（护栏、桁架、标准件布置）。
- 既有图纸要统计图元数量、核对图层与线型是否规范。
- 手工改 50 张图的标题栏不现实，需要脚本批处理。

**能力边界**：本技能不连接 CAD 软件、不渲染图纸、不自带几何内核。产出的是 Python 脚本；用户需自行安装 Python 与 `ezdxf`（`pip install ezdxf`），生成的文件用 CAD 或 DXF 查看器打开验证。本仓库不代为安装依赖。

## 执行步骤

1. 确认目标 DXF 版本（`R2018` 兼容性最好，老设备可能需要 `R2000`）。
2. 建立图层表：名称、颜色（ACI 索引）、线型（`CONTINUOUS` / `DASHED` / `CENTER`）、用途。命名遵守项目或国家标准，不要用默认 `Layer1`。
3. 写图元：线段、圆、多段线用 `msp.add_line` / `add_circle` / `add_lwpolyline`；文字用 `add_text` 或 `add_mtext`；尺寸标注用 `add_linear_dim` 后 `dim.render()`。
4. 单位与坐标：明确图纸单位（mm），`doc.header["$INSUNITS"] = 4` 表示毫米。
5. 读回统计：遍历 `msp.query("*")`，按 `dxftype()` 与 `dxf.layer` 汇总。
6. 给出验证方式：用 DXF 查看器打开、检查图元数、检查图层与线型。

## 可运行示例：参数化栏杆平面图

```python
import ezdxf
from ezdxf.enums import TextEntityAlignment

DOC_PATH = "rail_plan.dxf"
LENGTH = 3000.0          # mm
POST_SPACING = 500.0
POST_SIZE = 50.0

doc = ezdxf.new("R2018", setup=True)
doc.header["$INSUNITS"] = 4          # 4 = mm
msp = doc.modelspace()

doc.layers.add("RAIL_POST", color=7, linetype="CONTINUOUS")
doc.layers.add("RAIL_MAIN", color=5, linetype="CONTINUOUS")
doc.layers.add("DIM", color=2, linetype="CONTINUOUS")

# 主梁
msp.add_lwpolyline(
    [(0, 0), (LENGTH, 0), (LENGTH, 100), (0, 100)],
    close=True, dxfattribs={"layer": "RAIL_MAIN"},
)

# 立柱阵列
count = int(LENGTH // POST_SPACING) + 1
for i in range(count):
    x = i * POST_SPACING
    msp.add_lwpolyline(
        [(x, 0), (x + POST_SIZE, 0), (x + POST_SIZE, 100), (x, 100)],
        close=True, dxfattribs={"layer": "RAIL_POST"},
    )

# 尺寸标注（线性标注需要 render() 才真正生成几何）
dim = msp.add_linear_dim(base=(0, -200), p1=(0, 0), p2=(LENGTH, 0),
                         dxfattribs={"layer": "DIM"})
dim.render()

msp.add_text("RAIL PLAN 1:10", height=80,
             dxfattribs={"layer": "DIM"}).set_placement((0, 300), align=TextEntityAlignment.LEFT)

doc.saveas(DOC_PATH)
print(f"saved {DOC_PATH}, posts={count}")
```

## 读回统计片段

```python
import ezdxf
doc = ezdxf.readfile("rail_plan.dxf")
msp = doc.modelspace()
summary = {}
for e in msp.query("*"):
    key = (e.dxftype(), e.dxf.get("layer", "0"))
    summary[key] = summary.get(key, 0) + 1
for (t, layer), n in sorted(summary.items()):
    print(f"{t:12s} {layer:12s} {n}")
```

## 判定规则

- 图元坐标必须是实数，字符串坐标会写入失败。
- 标注必须 `render()`，否则 DXF 里只有标注定义没有几何。
- 图层名超过 255 字符或含非法字符 → 写入失败。
- 线型未在 `doc.linetypes` 中加载就引用 → 回退为 `CONTINUOUS`，需显式加载。
- 读旧图时先检查 `doc.dxfversion`，`R12` 不支持轻量多段线 `LWPOLYLINE`。

## 输出格式

```markdown
## ezdxf 脚本：<用途>

**DXF 版本**：<R2018>
**单位**：<mm>
**图层**：<名称/颜色/线型/用途 表>

```python
<完整脚本>
```

### 运行与验证
1. `python script.py`
2. 用 CAD 打开，核对 <图元数/图层/标注>
3. 未验证：<需用户实测项>
```

## 纪律

- 不给伪代码；脚本必须是完整可运行文件（含 import 与保存）。
- 不虚构生成结果，只描述脚本应当产生什么。
- 覆盖既有图纸前提示备份；读写路径由用户提供。
