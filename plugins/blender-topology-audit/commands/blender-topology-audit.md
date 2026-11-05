---
description: 生成 Blender 拓扑审计脚本，检查非流形、N-gon、松散点与面数预算
argument-hint: [目标对象或 collection 名] [面数预算，可选]
---

为 `$1` 生成一份 Blender 拓扑审计脚本，面数预算 `$2`（为空则只报告不判定）。**本命令只产出脚本与判读标准，需你在 Blender 里运行。**

**执行步骤**

1. 确认审计对象：单个 object 名、collection 名，或「选中对象」。
2. 生成脚本，用 `bmesh` 统计以下指标，不要用 `bpy.ops` 交互式选择：
   - 总面数/顶点数/边数
   - 非流形边（`edge.is_manifold == False`）
   - 边界边（`edge.is_boundary`）
   - N-gon（面顶点数 > 4）
   - 松散顶点/孤立边
   - 重复顶点（按坐标去重后差值）
3. 输出分级判定：阻断（非流形 > 0 且用于布尔/3D 打印）、警告（N-gon 出现在变形区域）、提示（面数超预算）。
4. 给出修复建议：`bmesh.ops.remove_doubles`、三角化、重建环切、分离松散几何。
5. 说明运行方式与预期输出（每个对象一行统计）。

**可运行脚本模板**

```python
import bpy
import bmesh

TARGET_COLLECTION = "Gen_Grille"   # 改成你的 collection；留空表示选中对象
FACE_BUDGET = 0                     # 0 表示不判定

def target_objects():
    if TARGET_COLLECTION:
        col = bpy.data.collections.get(TARGET_COLLECTION)
        return list(col.all_objects) if col else []
    return list(bpy.context.selected_objects)

for obj in target_objects():
    if obj.type != 'MESH':
        continue
    bm = bmesh.new()
    bm.from_mesh(obj.data)
    faces = len(bm.faces)
    verts = len(bm.verts)
    edges = len(bm.edges)
    non_manifold = sum(1 for e in bm.edges if not e.is_manifold)
    boundary = sum(1 for e in bm.edges if e.is_boundary)
    ngons = sum(1 for f in bm.faces if len(f.verts) > 4)
    loose_verts = sum(1 for v in bm.verts if not v.link_edges)
    over = faces > FACE_BUDGET if FACE_BUDGET else False
    print(f"{obj.name}: faces={faces} verts={verts} edges={edges} "
          f"non_manifold={non_manifold} boundary={boundary} ngons={ngons} "
          f"loose_verts={loose_verts} over_budget={over}")
    bm.free()
```

**输出格式**

```markdown
## 拓扑审计：<对象/collection>

| 对象 | 面数 | 顶点 | 非流形 | 边界边 | N-gon | 松散点 | 判定 |

### 判定标准
- 阻断：<...>
- 警告：<...>
- 提示：<...>

### 修复脚本（可选）
```python
<remove_doubles / triangulate 片段>
```
```

**纪律**

- 只读统计，脚本默认不修改网格；修复片段单独给出并标注「会改数据」。
- 必须在用户 Blender 版本上实测；脚本里的 API 变化点要写明。
- 不宣称本插件能自动修复拓扑。
