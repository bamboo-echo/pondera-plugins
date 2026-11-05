---
description: 检查 Blender 模型的 UV 展开质量，并给出可运行的 bpy 展开与打包脚本
argument-hint: [模型/网格名称] [用途：游戏低模/影视高模/贴图绘制]
---

检查 `$1` 的 UV 展开，目标用途为 `$2`。**本命令只产出脚本与检查清单，实际执行需你已安装 Blender 并在其中运行；不宣称自带建模能力。**

**检查项**

1. 接缝（Seam）：接缝是否放在视角不易察觉处（后脑、腋下、物体背面），是否避开高曲率与法线突变的边。
2. 展开方法：硬表面用 angle-based unwrap；有机体用 smart project 或先 mark seam 再 unwrap；建筑类可配合 cube project。
3. 重叠：同一 UV 空间是否有多块 island 重叠；对称部件是否故意镜像共享（需注明）。
4. 拉伸：检查 UV 网格与 3D 面的面积比，拉伸超过 2:1 的区域要重新展开。
5. 图素密度（texel density）：全模型每米像素数是否一致，UI 与主视觉是否统一。
6. 打包：island 间距是否 ≥ 2px（1024 贴图），是否留有边距避免采样溢出。
7. 朝向：UV 是否与 3D 朝向一致，避免贴图旋转 90° 后文字或纹理方向错误。
8. 边界：UV 是否超出 0–1 范围（tiling 贴图除外），超出会导致采样被 clamp 或重复。

**可运行脚本（粘贴到 Blender Scripting 编辑器）**

```python
import bpy

obj = bpy.context.active_object
if obj is None or obj.type != "MESH":
    raise RuntimeError("请先选中一个网格对象")

bpy.ops.object.mode_set(mode="EDIT")
bpy.ops.mesh.select_all(action="SELECT")

# 硬表面：角度法展开；有机体可改用 smart_project
bpy.ops.uv.unwrap(method="ANGLE_BASED", margin=0.001)

# 打包 island，旋转以提升填充率
bpy.ops.uv.pack_islands(rotate=True, margin=0.001)

bpy.ops.object.mode_set(mode="OBJECT")

me = obj.data
layer = me.uv_layers.active
if layer is None:
    raise RuntimeError("展开后仍无 UV 层")
uvs = [d.uv for d in layer.data]
us = [v.x for v in uvs]
vs = [v.y for v in uvs]
print(f"uv islands bbox u=({min(us):.3f},{max(us):.3f}) v=({min(vs):.3f},{max(vs):.3f})")
print(f"uv layers={len(me.uv_layers)} loops={len(uvs)}")
```

**输出格式**

```markdown
## UV 检查：<对象>

**用途**：<游戏/影视/贴图绘制>　**结论**：通过 X/8

| 检查项 | 现状 | 判定 | 修法 |
| --- | --- | --- | --- |

### 展开脚本
```python
<完整可运行脚本>
```

### 复检
- 在 UV Editor 打开棋盘格贴图，查看拉伸与接缝
- 统计 island 数与覆盖面积占比
```

**纪律**

- 不声称能代替人工判断接缝位置；脚本只做展开与打包。
- 贴图尺寸未给出时按 1024 估算间距，并注明假设。
- 执行前提醒保存 .blend，脚本会修改 UV 层。
