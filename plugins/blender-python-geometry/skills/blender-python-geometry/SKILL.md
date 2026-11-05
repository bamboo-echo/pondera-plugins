---
name: blender-python-geometry
description: 生成 Blender bpy 脚本：用 Python 创建参数化几何体、阵列、镜像与随机散布，输出可粘贴到 Blender 脚本编辑器直接运行的代码。当用户要用脚本重复生成模型、做批量布局，或想知道某个 bpy API 怎么写时使用。注意：本技能只产出脚本，实际建模需用户已安装 Blender 并自行运行。
runas: inline
---

# Blender bpy 几何脚本

## 何时使用

- 需要按参数批量生成几何体（楼梯、格栅、栏杆、阵列件、散布颗粒）。
- 想在 Blender 里用脚本替代重复手工操作。
- 已有模型，但需要脚本化重建以便版本管理。

**能力边界**：本技能不会连接 Blender、不执行脚本、不自带建模内核。产出的是 `.py` 脚本；用户需在 Blender 的 Scripting 工作区粘贴或 `Alt+P` 运行。API 以 Blender 4.x 为基准，3.x 需注意 `bpy.ops` 参数差异。

## 执行步骤

1. 明确输入参数：尺寸、数量、间距、随机种子、目标集合（collection）名称。
2. 选择构建方式：`bpy.ops.mesh.primitive_*_add` 快速建模；`bmesh` 精确控制拓扑；修改器（Array/Mirror/Solidify）用于后续可调。
3. 写脚本时固定三段结构：清理场景中本脚本产物 → 参数区 → 构建与收尾。
4. 每个对象设置 `obj.name`、放入指定 collection、必要时设置 `obj["gen_param"]` 便于后续筛选。
5. 给出运行说明与预期结果（对象数量、面数量级），并列出可能的 API 变化点。

## 可运行示例：参数化格栅阵列

```python
import bpy
import math

# ---- 参数区（改这里）----
COLLECTION_NAME = "Gen_Grille"
COUNT_X = 8
COUNT_Y = 6
BAR_LEN_X = 1.2
BAR_LEN_Y = 0.08
HEIGHT = 0.2
GAP = 1.0
# -------------------------

def get_collection(name):
    col = bpy.data.collections.get(name)
    if col is None:
        col = bpy.data.collections.new(name)
        bpy.context.scene.collection.children.link(col)
    return col

def clear_generated(col):
    for obj in list(col.objects):
        bpy.data.objects.remove(obj, do_unlink=True)

col = get_collection(COLLECTION_NAME)
clear_generated(col)

for ix in range(COUNT_X):
    for iy in range(COUNT_Y):
        bpy.ops.mesh.primitive_cube_add(size=1.0, location=(ix * GAP, iy * GAP, 0))
        obj = bpy.context.active_object
        obj.name = f"Bar_{ix:02d}_{iy:02d}"
        obj.scale = (BAR_LEN_X, BAR_LEN_Y, HEIGHT)
        obj["gen_param"] = {"ix": ix, "iy": iy, "seed": 0}
        # primitive_add 会把对象放进当前活动 collection，需要迁移到目标集合
        for c in list(obj.users_collection):
            c.objects.unlink(obj)
        col.objects.link(obj)

print(f"generated {COUNT_X * COUNT_Y} bars in {COLLECTION_NAME}")
```

## API 要点

- `bpy.ops.mesh.primitive_cube_add(size=..., location=...)` 新增对象后要用 `bpy.context.active_object` 取回，别假设它在 `scene.objects[-1]`。
- 单位：默认 1 Blender Unit = 1 m，导入引擎前确认缩放约定。
- 随机散布用 `random.Random(seed)`，不要用全局 `random`，否则不可复现。
- 修改器阵列比复制对象轻量：`mod = obj.modifiers.new("Array", 'ARRAY')`，再设 `mod.count`、`mod.relative_offset_displace`。
- 曲线倒角用 `bpy.ops.curve.primitive_bezier_curve_add` + `bevel_depth`，别手工生成圆柱拼接。

## 判定规则

- 脚本依赖未启用插件（如 extra objects）→ 在开头加 `bpy.ops.preferences.addon_enable(module=...)` 并注明。
- 每次重复运行前必须清理同名 collection，否则对象叠加。
- 面数超过 10 万的对象建议改用实例化（`collection.instance_offset` / Geometry Nodes）。
- 涉及 `bpy.ops` 时必须在正确的 context 下调用；后台模式用 `bpy.ops` 受限制，优先用 `bmesh` 或数据 API。

## 输出格式

```markdown
## Blender 脚本：<用途>

**运行方式**：Blender 4.x → Scripting → 粘贴 → Run Script
**预期结果**：<对象数、面数量级、collection 名>
**依赖**：<内置 / 需启用插件>

```python
<完整脚本>
```

### API 注意点
- ...

### 未验证项
- <需在用户 Blender 版本上实测的点>
```

## 纪律

- 不给伪代码；所有脚本必须能在 Blender 内置 Python 环境直接跑。
- 不承诺「自动生成成品模型」；脚本只是执行手段，最终效果由用户运行与调整。
- 修改用户现有场景的脚本必须先做备份提示。
