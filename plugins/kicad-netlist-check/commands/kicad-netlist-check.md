---
description: 导出并检查 KiCad 网表，核对单端网络、电源引脚与去耦
argument-hint: [KiCad 工程路径或 .kicad_sch 文件] [目标板名，可选]
---

检查 `$1` 的网表完整性。**需要你已安装 KiCad；本命令给出导出命令与解析脚本，由你执行。**

**执行步骤**

1. 导出网表（KiCad 7/8 用 `kicad-cli`；KiCad 6 及更早用原理图界面「导出网表」）：
   ```bash
   kicad-cli sch export netlist --format kicadsexpr -o board.net board.kicad_sch
   ```
2. 用下面的脚本解析 `board.net`，报告：
   - 单端网络（只连接 1 个引脚的 net），典型是漏连线或标签写错。
   - 电源网络（`+5V`、`GND` 等）上缺少去耦电容。
   - 未连接引脚（`no_connect` 标记与实际悬空不一致）。
   - 连接器引脚与网名映射（确认 Pin 1 定义）。
3. 与原理图 ERC 结果交叉核对：ERC 报「引脚未连接」但网表里该引脚挂在网络上 → 符号引脚类型定义错。
4. 输出问题清单，按「会导致板子不工作 / 影响可靠性 / 仅规范问题」三档。

**解析脚本**

```python
import re
from collections import defaultdict

NET_FILE = "board.net"

text = open(NET_FILE, encoding="utf-8").read()
blocks = re.findall(r"\(net\s+\(code[^)]*\)\s+\(name\s+\"([^\"]+)\"\)(.*?)\n\s*\)", text, re.S)
nets = defaultdict(list)
for name, body in blocks:
    refs = re.findall(r"\(ref\s+\"([^\"]+)\"\)\s*\(pin\s+\"([^\"]+)\"\)", body)
    nets[name] = refs

single = {n: r for n, r in nets.items() if len(r) == 1}
print("single-pin nets:", single)

power = {n: r for n, r in nets.items() if n.upper() in ("GND", "+5V", "+3V3", "VCC")}
for n, refs in power.items():
    caps = [ref for ref, pin in refs if ref.upper().startswith("C")]
    print(f"{n}: pins={len(refs)} caps={len(caps)}")

all_pins = [(ref, pin) for refs in nets.values() for ref, pin in refs]
print("total net pins:", len(all_pins))
```

**输出格式**

```markdown
## 网表检查：<板名>

**导出命令**：<...>　**网表版本**：<...>
**统计**：网络 <n>，单端 <m>，引脚 <k>

| # | 类别 | 网络/器件 | 问题 | 判定 | 建议 |

### 电源与去耦
| 电源网络 | 引脚数 | 去耦电容数 | 结论 |

### 需人工核对
- <封装引脚定义、连接器方向 等>
```

**纪律**

- 不修改原理图；只读网表。
- 单端网络不等于错误（测试点可能合法），但必须逐条说明原因。
- 不虚构 ERC 结果；用户未提供 ERC 输出时标「未提供」。
