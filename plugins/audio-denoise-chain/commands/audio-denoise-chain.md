---
description: 用 ffmpeg 的 afftdn/arnndn 加 loudnorm 生成降噪命令链，并给出过度降噪的判据与试听校验
argument-hint: [输入音频路径] [环境噪声类型：底噪/风扇/键盘/多人]
---

为 `$1` 生成降噪与响度归一命令，噪声类型 `$2`。**本命令只产出命令与判据，需你本机已安装 ffmpeg；`arnndn` 的 `.rnnn` 模型文件需自备，插件不附带。**

**先探测源文件**

```bash
ffprobe -hide_banner -show_streams -show_format "$1"
```

关注采样率、声道数、码率与是否已压缩。已强压过的 MP3/AAC 再降噪会更明显掉细节，优先要原始 WAV。

**方案 A：afftdn（频域降噪，无需模型，适合稳态底噪）**

```bash
ffmpeg -i "$1" \
  -af "afftdn=nr=12:nf=-50:tn=1,loudnorm=I=-16:TP=-1.5:LRA=11" \
  -ar 48000 -ac 1 -c:a pcm_s16le out_denoise.wav
```

参数含义：`nr` 降噪量（dB），`nf` 噪声下限，`tn=1` 自动跟踪噪声轮廓。**从 `nr=8` 起步，每次加 2–4**，不要一上来给 20。

**方案 B：arnndn（RNN 降噪，需要模型文件）**

```bash
ffmpeg -i "$1" \
  -af "arnndn=m=/path/to/std.rnnn:mix=0.8,loudnorm=I=-16:TP=-1.5:LRA=11" \
  -ar 48000 -ac 1 -c:a pcm_s16le out_rnn.wav
```

`m=` 指向你自备的 `.rnnn` 模型（本项目不提供模型）；`mix` 是干湿比，0.8 保留少量原声可减少「水下感」。模型与人声语言不匹配时效果会明显变差。

**过度降噪判据（满足任一即回调）**

1. 听感出现金属感、水下感或「闷住」→ `nr` 降低 4，`mix` 降到 0.7。
2. 词首辅音（p/t/k/s）被削掉，字头变软 → 回调 `nr`，并加 `highpass=f=60` 分离低频噪声。
3. 尾音与呼吸声被完全抹平 → 过强，保留 30% 原始噪声更自然。
4. 频谱图在 2–8 kHz 出现大片空白或齿音消失 → 过强。
5. `nr > 20` 且素材本身信噪比 > 15 dB → 基本属于过度处理。

**试听与量化校验**

```bash
# 生成 A/B 对照：左原声右处理后，便于盲听
ffmpeg -i "$1" -i out_denoise.wav -filter_complex \
  "[0:a]volume=1.0[a];[1:a]volume=1.0[b];[a][b]amerge=inputs=2" -ac 2 ab_compare.wav

# 客观指标：电平、噪声底、峰值
ffmpeg -i out_denoise.wav -af "astats=metadata=1:reset=0" -f null - 2>&1 | grep -E "RMS level|Peak level|Noise floor"
```

判据：处理后 RMS 不应比原声低超过 3 dB；峰值不得触顶（True Peak ≤ -1.5 dBTP）；噪声底下降 6–12 dB 属正常，超过 15 dB 多半削了人声。

**输出格式**

```markdown
## 降噪方案：<文件>

**源信息**：<采样率/声道/码率>
**噪声类型**：<稳态/非稳态>　**方案**：A afftdn / B arnndn
**参数**：nr=<N> nf=<N> tn=<0|1> mix=<N>

```bash
<可直接执行的完整命令>
```

### 判据与回调
| 现象 | 判据 | 回调动作 |
| --- | --- | --- |

### 校验步骤
1. A/B 盲听 30 秒
2. 看 astats 的 RMS/峰值
3. 抽 3 段词首辅音单独试听
```

**纪律**

- 不承诺「一键去噪」；参数必须给区间与回调规则。
- 模型文件缺失时明确报错提示，不伪造路径。
- 不覆盖原文件，输出统一带 `_denoise` 后缀。
- 不把降噪与混响消除混为一谈，后者需要专门算法。
