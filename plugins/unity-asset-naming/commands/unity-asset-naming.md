---
description: 审计 Unity 资源命名与导入设置，生成可粘贴的 AssetPostprocessor 校验脚本
argument-hint: [资源目录或清单] [项目约定：命名前缀/图集策略]
---

审计 `$1` 的资源命名与导入配置，遵循 `$2` 的项目约定。**本命令只产出脚本与检查清单，实际执行需你在已安装的 Unity 编辑器里运行。**

**检查项**

1. 命名规范：贴图 `T_`、材质 `M_`、模型 `SM_`、预制体 `PF_`、音效 `SFX_`、音乐 `BGM_`、UI 图 `UI_`、动画 `ANIM_`。
2. 目录规范：`Assets/Art/Characters`、`Assets/Audio`、`Assets/Prefabs` 分层；禁止资源散落在 `Assets/` 根。
3. 命名字符：仅 `A-Za-z0-9_-`，禁止空格、中文、`@` 以外的特殊字符；下划线不与空格混用。
4. 贴图导入：尺寸为 2 的幂（UI 九宫格除外）、`sRGB` 与法线贴图类型正确、`mipmap` 按用途开关、压缩按平台（Android ASTC / iOS ASTC / PC DXT）。
5. 模型导入：`Read/Write` 关闭、`Generate Colliders` 关闭（除非需要）、`Optimize Mesh` 开启、缩放 `1`、`Import Units` 与项目一致。
6. 音频导入：短音效 `Decompress On Load` + 单声道，长音乐 `Streaming`。
7. 图集：同 UI 模块进同一 `SpriteAtlas`，避免跨图集打断合批。
8. 变体：`Resources/` 目录尽量不用；`AssetBundle`/`Addressables` 分组是否有重复依赖。

**校验脚本（可直接放到 `Assets/Editor/AssetNamingValidator.cs`）**

```csharp
using UnityEditor;
using System.IO;
using UnityEngine;

public class AssetNamingValidator : AssetPostprocessor
{
    static readonly (string prefix, string[] folders)[] Rules = {
        ("T_",   new[] { "Assets/Art/Textures" }),
        ("M_",   new[] { "Assets/Art/Materials" }),
        ("SM_",  new[] { "Assets/Art/Models" }),
        ("PF_",  new[] { "Assets/Prefabs" }),
        ("SFX_", new[] { "Assets/Audio" }),
        ("BGM_", new[] { "Assets/Audio" }),
    };

    void OnPreprocessTexture()
    {
        var importer = (TextureImporter)assetImporter;
        importer.textureCompression = TextureImporterCompression.CompressedHQ;
        importer.isReadable = false;
        if (assetPath.Contains("Assets/Art/UI"))
            importer.mipmapEnabled = false;
    }

    void OnPostprocessAllAssets(string[] imported, string[] deleted,
                                string[] moved, string[] movedFrom)
    {
        foreach (var path in imported)
        {
            if (!path.StartsWith("Assets/")) continue;
            var name = Path.GetFileNameWithoutExtension(path);
            if (name.Contains(" ") || name.Contains("中文"))
                Debug.LogWarning($"[命名] 非法字符: {path}");

            foreach (var (prefix, folders) in Rules)
                foreach (var folder in folders)
                    if (path.StartsWith(folder) && !name.StartsWith(prefix))
                        Debug.LogWarning($"[命名] {folder} 需以 {prefix} 开头: {path}");
        }
    }
}
```

**输出格式**

```markdown
## Unity 资源审计：<目录>

**统计**：资源 N 个，命名违规 N，导入配置问题 N

| 路径 | 问题类型 | 现状 | 建议 | 严重度 |
| --- | --- | --- | --- | --- |

### 校验脚本
```csharp
<完整可粘贴版本>
```

### 批量修正建议
- 重命名清单（旧名 → 新名），提示会丢失引用，需先提交版本控制
```

**纪律**

- 不声称能直接改资源；重命名会断引用，必须提示先提交再操作。
- 贴图压缩格式按目标平台给，不给「通用最优」。
- 未看到实际导入设置时写「需在 Inspector 确认」，不臆测。
