---
name: image-batch-naming
description: 批量处理图片并统一命名：ImageMagick 与 Pillow 两套可运行脚本，覆盖缩放、水印、EXIF 剥离、序号与日期命名、失败回滚。当用户要整理成百上千张素材、导出电商图、或把相机原图改成规范文件名时使用。本插件只产出脚本，执行依赖用户已安装 ImageMagick 或 Pillow。
runas: inline
---

# 图片批处理与命名

## 何时使用

- 相机/手机导出的 `IMG_1234.JPG` 需要统一成 `20261105_001.jpg` 这类可排序名称。
- 需要批量缩放、加水印、剥离 EXIF 后上传。
- 处理到一半失败，需要能回滚到原状。

## 先定命名规则

推荐 `日期_序号_用途.扩展名`：

```text
2026-11-05_001_hero.jpg
2026-11-05_002_detail.jpg
```

规则要求：只用 `A-Za-z0-9._-`；序号从 1 开始且补零到 3 位；日期用文件 `EXIF DateTimeOriginal` 优先、缺失时用文件修改时间；扩展名统一小写。

## 方案 A：ImageMagick（命令行，适合快速批处理）

```bash
# 0) 预演：只打印将要处理哪些文件，不落盘
magick identify -format "%f %wx%h %[EXIF:DateTimeOriginal]\n" *.{jpg,JPG,jpeg,png}

# 1) 统一方向、剥离 EXIF、缩放到长边 1920、转 sRGB
magick mogrify -path ./out -auto-orient -strip \
  -resize "1920x1920>" -colorspace sRGB -quality 82 \
  -format jpg *.jpg

# 2) 水印：批量在右下角叠加，边距 20px
mkdir -p ./watermarked
for f in ./out/*.jpg; do
  magick "$f" watermark.png -gravity southeast -geometry +20+20 \
    -compose over -composite "./watermarked/$(basename "$f")"
done

# 3) 校验：输出数量必须等于输入数量
echo "in=$(ls -1 *.jpg | wc -l) out=$(ls -1 ./out/*.jpg | wc -l)"
```

注意 `-strip` 会移除 EXIF（含拍摄时间与 GPS）；如果命名还需要拍摄时间，**先提取再剥离**。

## 方案 B：Pillow（Python，适合需要按 EXIF 改名的场景）

```python
from pathlib import Path
from datetime import datetime
from PIL import Image, ImageOps, ImageDraw

SRC = Path("raw")
OUT = Path("out")
OUT.mkdir(exist_ok=True)

def taken_at(img, path):
    exif = img.getexif()
    raw = exif.get(36867)  # DateTimeOriginal
    if raw:
        return datetime.strptime(raw, "%Y:%m:%d %H:%M:%S")
    return datetime.fromtimestamp(path.stat().st_mtime)

records = []
for i, path in enumerate(sorted(SRC.glob("*.jpg")), start=1):
    with Image.open(path) as img:
        img = ImageOps.exif_transpose(img).convert("RGB")
        day = taken_at(img, path).strftime("%Y-%m-%d")
        name = f"{day}_{i:03d}.jpg"
        img.thumbnail((1920, 1920))
        img.save(OUT / name, "JPEG", quality=82, optimize=True)
        records.append((path.name, name))

for src, dst in records:
    print(f"{src} -> {dst}")
print(f"total={len(records)}")
```

要点：`ImageOps.exif_transpose` 会把方向转正；`save` 不写 `exif` 参数即等于剥离元数据；`thumbnail` 保持宽高比且不放大。

## 失败与回滚

1. 先写清单（`manifest.tsv`：原路径 → 新路径 → 状态），成功一行写一行。
2. 所有输出写到独立目录（`out/` 或 `out_tmp/`），不要就地覆盖。
3. 任一文件失败：记录到 `failed.log`，继续处理其余文件，最后统一报告。
4. 回滚：输出目录整体删除即可；若已就地重命名，按 manifest 逆序改回，并校验文件数。
5. 重命名前先 `git status` 或打 zip 快照，避免不可逆。

## 检查清单

- [ ] 输入数量 = 输出数量 + 失败数量
- [ ] 文件名唯一（重名时加序号后缀）
- [ ] 无空格、无中文、无大写扩展名
- [ ] 长边不超过目标尺寸，未被放大
- [ ] 方向正确（竖图不横躺）
- [ ] EXIF 已剥离或按需保留
- [ ] 水印不遮挡主体关键区域
- [ ] 输出体积在目标区间（如 200–400 KB）

## 输出格式

```markdown
## 图片批处理：<目录>

**规模**：输入 N / 成功 N / 失败 N
**命名规则**：<日期_序号>
**工具**：ImageMagick / Pillow

### 脚本
```bash
<可运行命令> 或 <可运行 Python>
```

### 失败与回滚
- 失败文件：<清单>
- 回滚方式：<删除 out/ 或按 manifest 逆向>
```

## 纪律

- 不就地覆盖原图；所有写入落到新目录。
- 不凭空设定目标尺寸与质量，按用途给建议（网页/电商/印刷）。
- 执行前提示先备份；脚本只给可运行版本，不承诺已在本机验证。
