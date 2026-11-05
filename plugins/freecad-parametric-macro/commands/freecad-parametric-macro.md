---
description: 生成 FreeCAD 参数化宏（Part + Spreadsheet 驱动），可直接粘贴到宏编辑器
argument-hint: [零件描述与关键尺寸] [参数来源：硬编码或表格名]
---

为 `$1` 写一段 FreeCAD 宏，参数来自 `$2`（为空则用脚本顶部常量）。**本命令只产出宏代码，不在 FreeCAD 中执行；需你已安装 FreeCAD 并在「宏 → 编辑」里运行。**

**执行步骤**

1. 明确建模步骤：草图/拉伸/切除/阵列/倒角，以及每步依赖的参数。
2. 优先 `Part` 模块（`Part.makeBox`、`Part.makeCylinder`、`Part.makeCompound`）——无需 GUI 草图，宏在后台也能跑。
3. 参数集中定义：要么脚本顶部常量，要么读取 Spreadsheet 单元格 `sheet.get("B1")`，并校验类型与范围。
4. 布尔运算用 `base.cut(tool)` / `base.fuse(tool)`，结果对象设置 `Label` 与 `ViewObject.ShapeColor`。
5. 文档结构：每次运行前删除同名对象，避免重复叠加；最后 `doc.recompute()`。
6. 给出运行方式与验证清单（体积、包围盒、导出 STEP 命令）。

**可运行宏模板**

```python
import FreeCAD as App
import Part

DOC_NAME = "ParamPlate"
LENGTH = 120.0   # mm
WIDTH = 80.0
THICK = 6.0
HOLE_D = 10.0
HOLE_MARGIN = 15.0

# 已打开的文档也可用 App.ActiveDocument 直接取到
doc = App.getDocument(DOC_NAME) or App.newDocument(DOC_NAME)
doc.recompute()

# 幂等：删除上次运行生成的对象
for name in ("Plate", "HolePattern"):
    if name in doc.Objects:
        doc.removeObject(name)

plate = doc.addObject("Part::Feature", "Plate")
plate.Shape = Part.makeBox(LENGTH, WIDTH, THICK)

# 四个角孔（用 Compound 合并后一次 cut）
holes = []
for x in (HOLE_MARGIN, LENGTH - HOLE_MARGIN):
    for y in (HOLE_MARGIN, WIDTH - HOLE_MARGIN):
        cyl = Part.makeCylinder(HOLE_D / 2.0, THICK)
        cyl.translate(App.Vector(x, y, 0))
        holes.append(cyl)

pattern = doc.addObject("Part::Feature", "HolePattern")
pattern.Shape = Part.makeCompound(holes)
plate.Shape = plate.Shape.cut(pattern.Shape)

doc.recompute()

print("bbox:", plate.Shape.BoundBox)
print("volume:", plate.Shape.Volume)
# 导出 STEP（取消注释后使用）
# Part.export([plate], r"C:/exports/plate.step")
```

**输出格式**

```markdown
## FreeCAD 宏：<零件>

**依赖**：FreeCAD 0.20+/1.0，Part 模块
**参数**：<表：名称/值/单位/范围>

```python
<完整宏>
```

### 运行与验证
1. 宏 → 新建 → 粘贴 → 执行
2. 核对包围盒 <...>、体积 <...>
3. 导出 STEP/STL 后由用户复核

### 未验证项
- <GUI 相关行为、特定 FreeCAD 版本差异>
```

**纪律**

- 不使用 `Gui` 相关调用，保证后台 `freecadcmd` 也能运行。
- 不虚构体积与包围盒数值；模板里 print 的是运行后真实值，输出中不要预填。
- 覆盖同名对象前先删除，避免用户文档里出现重复实体。
