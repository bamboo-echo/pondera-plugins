---
description: 生成 Blender glTF/OBJ 导出脚本，控制减面、纹理与材质数量
argument-hint: [目标 collection 或对象] [导出路径与体积目标]
---

为 `$1` 生成导出优化方案与脚本，导出到 `$2`。**本命令只产出脚本与参数建议，不执行 Blender 导出。**

**执行步骤**

1. 先做导出前检查：面数、材质数、贴图分辨率、是否含动画/形态键、坐标轴朝向（Y-up 还是 Z-up）。
2. 减面策略：静态道具用 Decimate（`COLLAPSE`，ratio 按目标面数反推）；建筑/硬表面优先保留轮廓，避免在 UV 接缝处塌陷。
3. 纹理策略：贴图统一为正方形 PNG/JPEG，尺寸 512/1024/2048；合并通道（ORM）以减少贴图数；颜色贴图用 sRGB，数据贴图用 Non-Color。
4. 材质合并：同一着色模型的多材质尽量合并为一张图集，减少 draw call。
5. 导出参数：`export_format='GLB'`、`export_apply=True`（应用修改器）、`export_yup=True`、`export_animations` 按需、`export_image_format='AUTO'`。
6. 给出体积核对方式：导出后检查文件大小与贴图总大小，超预算时回到第 3 步。

**可运行脚本模板**

```python
import bpy

SOURCE_COLLECTION = "Gen_Grille"      # 改成你的 collection
EXPORT_PATH = r"C:/exports/model.glb"  # 改成你的路径
DECIMATE_RATIO = 0.5                   # 1.0 表示不减面

col = bpy.data.collections.get(SOURCE_COLLECTION)
if col is None:
    raise RuntimeError(f"collection not found: {SOURCE_COLLECTION}")

for obj in col.all_objects:
    if obj.type != 'MESH':
        continue
    if DECIMATE_RATIO < 1.0:
        mod = obj.modifiers.get("ExportDecimate") or obj.modifiers.new("ExportDecimate", 'DECIMATE')
        mod.decimate_type = 'COLLAPSE'
        mod.ratio = DECIMATE_RATIO

for obj in bpy.context.view_layer.objects:
    obj.select_set(obj.name in {o.name for o in col.all_objects})

bpy.ops.export_scene.gltf(
    filepath=EXPORT_PATH,
    export_format='GLB',
    use_selection=True,
    export_apply=True,
    export_yup=True,
    export_animations=False,
    export_image_format='AUTO',
)
print(f"exported to {EXPORT_PATH}")
```

**输出格式**

```markdown
## 导出优化：<目标>

**当前**：面数 <n>，材质 <m>，贴图 <k> 张（最大 <px>）
**目标**：<体积上限 / 目标平台>

### 优化项
| # | 项 | 现状 | 改法 | 预估收益 |

### 脚本
```python
<完整脚本>
```

### 导出后核对
- 文件大小：<...>
- 材质/贴图数：<...>
- 用 glTF Validator 或引擎导入验证：<需用户执行>
```

**纪律**

- 修改器会实际改变网格，脚本运行前提示用户备份 `.blend`。
- 不虚构减面后的面数；给的是目标与参数，实测由用户完成。
- 材质合并涉及贴图重烘焙，成本高时明确说明，不只提「合并材质」四个字。
