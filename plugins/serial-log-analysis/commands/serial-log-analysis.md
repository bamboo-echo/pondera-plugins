---
description: 解析串口日志与示波器导出数据，定位丢帧、超时、复位与总线错误
argument-hint: [串口设备或日志文件] [波特率/协议]
---

分析 `$1`（串口设备或日志文件），参数 `$2`。**本命令只产出分析脚本与结论模板，需你本机已安装 Python 与 pyserial；不代为安装任何依赖。**

**执行步骤**

1. 先确认链路：波特率、数据位、校验、停止位、是否硬件流控；波特率与设备配置不一致会表现为乱码而非丢帧。
2. 采集：用下面的脚本抓带主机时间戳的原始行，同时保留设备自身时间戳（若有），两者用于区分「设备卡住」与「USB 转串口丢数据」。
3. 解析：按正则提取 `[时间] [级别] [模块] 消息`，统计相邻同模块消息间隔，找出超过阈值的空档。
4. 关联：把空档与复位标志（`Reset cause`、`HardFault`、看门狗 `IWDG`）对齐，判断空档前是否有中断风暴。
5. 总线：I2C/SPI 出现 `NACK`、`BUSY`、`OVR` 时，核对时钟频率与上拉电阻，不要先怀疑芯片。
6. 示波器：导出的 CSV 里看上升沿、毛刺宽度、总线空闲电平；与串口时间戳对齐到同一时间基准。

**采集脚本（pyserial，可运行）**

```python
import serial, time, sys, re

PORT = "COM3"          # Linux: /dev/ttyUSB0
BAUD = 115200
DURATION = 30          # 秒

pat = re.compile(r"\[(?P<dev>[^\]]+)\]\[(?P<lvl>[A-Z]+)\](?P<msg>.*)")

with serial.Serial(PORT, BAUD, timeout=0.2) as ser, open("serial.log", "w", encoding="utf-8") as f:
    t0 = time.monotonic()
    n_err = 0
    while time.monotonic() - t0 < DURATION:
        raw = ser.readline().decode("utf-8", errors="replace").rstrip()
        if not raw:
            continue
        host_ms = int((time.monotonic() - t0) * 1000)
        f.write(f"{host_ms}\t{raw}\n")
        m = pat.search(raw)
        if m and m.group("lvl") in {"ERR", "WARN", "FATAL"}:
            n_err += 1
            print(f"[{host_ms}ms] {m.group('lvl')} {m.group('msg').strip()}")
    print(f"done: errors={n_err}", file=sys.stderr)
```

**间隙分析（离线，可运行）**

```python
import re

THRESHOLD_MS = 500
rows = []
with open("serial.log", encoding="utf-8") as f:
    for line in f:
        host_ms, rest = line.rstrip("\n").split("\t", 1)
        rows.append((int(host_ms), rest))

gaps = [(rows[i][0] - rows[i-1][0], rows[i-1][1], rows[i][1])
        for i in range(1, len(rows)) if rows[i][0] - rows[i-1][0] > THRESHOLD_MS]
for gap, before, after in gaps:
    print(f"gap {gap}ms\n  before: {before}\n  after : {after}")
print(f"total gaps: {len(gaps)}")
```

**输出格式**

```markdown
## 串口日志分析：<设备>

**配置**：<端口/波特率/时长>　**样本**：N 行　**错误**：N 条　**空档**：N 处

| 时间(ms) | 级别 | 模块 | 消息 | 判定 |
| --- | --- | --- | --- | --- |

### 关键空档
| 空档起点 | 持续 | 前一条 | 后一条 | 最可能原因 |
### 复位与异常
### 建议
- 抓取方式改进（加设备侧时间戳、提高波特率、加环形缓冲）
- 待验证项（需示波器/逻辑分析仪确认）
```

**判定规则**

- 空档前后都有心跳 → 大概率主机侧丢数据（USB/驱动），不是设备死机。
- 空档后紧跟 `Reset cause` → 设备复位，查看门狗与 HardFault。
- 乱码且长度固定 → 波特率不匹配。
- 偶发 `NACK` 且重试成功 → 总线时序临界，先降速再查上拉。
- 示波器毛刺 < 100ns 且不影响功能 → 记录，不建议改硬件。

**纪律**

- 不虚构采集结果；脚本需用户在自己机器上运行。
- 不宣称能自动修复固件；只给定位与建议。
- 日志摘录逐字，脱敏后保留时间戳。
