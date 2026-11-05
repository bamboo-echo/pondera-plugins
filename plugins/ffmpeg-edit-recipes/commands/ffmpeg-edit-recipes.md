---
description: 生成可直接运行的 ffmpeg 命令，完成裁剪、压缩、转码与批量处理
argument-hint: [输入文件或通配符] [目标：分辨率/时长/体积/格式]
---

为 `$1` 生成 ffmpeg 命令，目标为 `$2`。**本命令只产出命令，需你本机已安装 ffmpeg 并自行执行；不代为下载或安装任何软件。**

**执行步骤**

1. 先探测源文件：`ffmpeg -i input.mp4` 记录编码、分辨率、帧率、码率、音轨数与时长。
2. 按目标选择命令模板，明确每个参数的作用，不要堆砌无意义参数。
3. 给出体积估算方法：`目标码率 ≈ (目标体积MB × 8 × 1024) / 时长秒数`，音轨预留 128k。
4. 批量处理给出 PowerShell / bash 循环，并说明文件名冲突与覆盖风险。
5. 说明验证方式：`ffprobe` 检查输出编码与时长，抽帧对比画质。

**常用模板**

```bash
# 无损裁剪（重编码，帧精确）
ffmpeg -i input.mp4 -ss 00:01:30 -to 00:03:00 -c:v libx264 -crf 18 -c:a aac -b:a 128k out.mp4

# 快速裁剪（不重编码，切点对齐关键帧，可能有偏差）
ffmpeg -ss 00:01:30 -to 00:03:00 -i input.mp4 -c copy out.mp4

# 压缩到约 100MB（时长 600 秒）
ffmpeg -i input.mp4 -c:v libx264 -preset medium -crf 23 -maxrate 1300k -bufsize 2600k \
  -c:a aac -b:a 128k out.mp4

# 缩放并统一帧率
ffmpeg -i input.mp4 -vf "scale=1920:-2,fps=30" -c:v libx264 -crf 20 -c:a copy out.mp4

# 抽帧（每 2 秒一张）
ffmpeg -i input.mp4 -vf "fps=1/2" -q:v 3 frames/frame_%04d.jpg

# 烧录字幕
ffmpeg -i input.mp4 -vf "subtitles=zh.srt:force_style='FontSize=22'" -c:v libx264 -crf 20 -c:a copy out.mp4

# 仅提取音频
ffmpeg -i input.mp4 -vn -c:a aac -b:a 192k out.m4a
```

**输出格式**

```markdown
## ffmpeg 方案：<目标>

**源信息**：<编码/分辨率/帧率/时长/码率>
**目标**：<...>

### 命令
```bash
<可直接运行>
```

### 参数说明
| 参数 | 作用 | 为什么这么选 |

### 预计体积
<计算过程与结果区间>

### 验证
- `ffprobe -v error -show_format -show_streams out.mp4`
- 抽 3 帧人工对比
```

**纪律**

- 不给出未标注是否重编码的命令；`-c copy` 的切点偏差必须提示。
- 批量覆盖前必须提示备份或改输出目录。
- 不虚构编码后的实际体积，给区间并注明「需实测」。
