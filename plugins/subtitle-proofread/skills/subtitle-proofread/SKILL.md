---
name: subtitle-proofread
description: 字幕校对：检查 SRT/ASS 的时间轴重叠、单行长度、每秒字数（CPS）、断句与术语一致性，并给出逐条修正与自动化脚本。当用户有字幕文件要校对、字幕与语音对不上、或需要统一术语时使用。
runas: inline
---

# 字幕校对

## 何时使用

- 拿到机器翻译或人工翻译的字幕，需要可读性与同步性检查。
- 观众反馈「字幕太快」「一句话跨两行很怪」。
- 多集成片中同一术语译法不一致。

不负责配音与混音；视频剪辑命令用 `ffmpeg-edit-recipes`。

## 执行步骤

1. 解析字幕结构：条数、时长、每条的起止时间与文本；先确认编码为 UTF-8。
2. 时间轴检查：相邻条目重叠、间隔过短（< 2 帧）、单条时长过短（< 1 秒）或过长（> 7 秒）。
3. 阅读速度检查：CPS = 字符数 / 显示秒数。中文建议 ≤ 9 CPS，英文 ≤ 17 CPS；超出需拆条或延长。
4. 断句检查：一条内不宜跨两个完整句；行末不留单字孤行；中文每行 ≤ 16–18 字，英文 ≤ 42 字符。
5. 内容检查：与语音是否一致（听不出时标「需听校」）、数字与单位是否统一、标点是否规范（中文用全角）。
6. 术语一致性：同一概念全文唯一译法，对照术语表；人名地名首次出现是否给原文。
7. 输出逐条修正表 + 可选的自动检查脚本。

## 自动检查脚本

```python
import re

PATH = "zh.srt"
MAX_CPS = 9.0
MAX_LINE_CHARS = 18
MIN_DURATION = 1.0
MAX_DURATION = 7.0

def ts_to_sec(ts):
    h, m, rest = ts.split(":")
    s, ms = rest.replace(",", ".").split(".")
    return int(h) * 3600 + int(m) * 60 + float(s) + float(ms) / 1000

blocks = re.split(r"\n\s*\n", open(PATH, encoding="utf-8-sig").read().strip())
prev_end = 0.0
for block in blocks:
    lines = [l for l in block.splitlines() if l.strip()]
    if len(lines) < 3:
        continue
    idx, timecode, text = lines[0], lines[1], "\n".join(lines[2:])
    start_s, end_s = [x.strip() for x in timecode.split("-->")]
    start, end = ts_to_sec(start_s), ts_to_sec(end_s)
    duration = end - start
    chars = len(text.replace("\n", ""))
    cps = chars / duration if duration > 0 else 999
    flags = []
    if start < prev_end:
        flags.append("overlap")
    if duration < MIN_DURATION or duration > MAX_DURATION:
        flags.append(f"duration={duration:.2f}s")
    if cps > MAX_CPS:
        flags.append(f"cps={cps:.1f}")
    if any(len(l) > MAX_LINE_CHARS for l in text.splitlines()):
        flags.append("line-too-long")
    if flags:
        print(f"[{idx}] {flags} :: {text[:40]}")
    prev_end = end
```

## 判定规则

- 重叠 > 0.1 秒 → 必须修（播放器会截断或闪跳）。
- CPS 超阈值 20% 以上 → 拆条，不要仅缩短时长。
- 单条含两个以上句号 → 拆条。
- 译文与原文长度比 > 1.3（中文对英文）→ 检查是否漏译或过度意译。
- 术语不一致 → 以术语表为准，全文统一。
- 无法判断听感与语义的条目 → 标「需听校」，不要静默改词。

## 输出格式

```markdown
## 字幕校对报告：<文件>

**条目**：<n>　**总时长**：<...>　**编码**：<...>

### 结构问题
| 条目 | 类型 | 现状 | 建议 |

### 逐条修正
| 条目 | 原文 | 修正后 | 理由 |

### 术语一致性
| 概念 | 出现译法 | 建议统一为 | 出现次数 |

### 需听校
- [ ] 条目 <...>
```

## 纪律

- 只改字幕文本与时间轴，不替用户重新翻译内容。
- 修改前提示备份原文件；脚本输出修正版到新文件。
- 不虚构 CPS 数值；以实际解析结果为准。
