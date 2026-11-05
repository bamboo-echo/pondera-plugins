---
name: pandas-cleaning-template
description: 生成 pandas 数据清洗脚本：读入、类型转换、缺失值与重复处理、异常值标记、日期解析与清洗报告，输出可运行代码。当用户拿到脏 CSV/Excel 需要先清洗再用，或要建立可复用的清洗流程时使用。
runas: inline
---

# pandas 清洗模板

## 何时使用

- 原始 CSV/Excel 字段类型混乱、含空值、重复行、异常值。
- 需要一份可复现、可交接的清洗脚本，而不是手工在 Excel 里改。
- 要先把数据质量量化出来，再做建模或分析。

图表选型与建模不在本技能范围；清洗完的产物再交给下游。

## 执行步骤

1. **先探查再动手**：`df.info()`、`df.describe(include="all")`、每列唯一值数量与样例，记录原始行列数。
2. **类型转换**：数值列去千分位与货币符号后 `pd.to_numeric(errors="coerce")`；日期用 `pd.to_datetime(format=...)`，失败项单独列出。
3. **缺失值决策**：逐列判定处理方式——删除行、填充（均值/中位数/前值填充）、保留并标记。必须写清理由，不要一律 `dropna()`。
4. **重复处理**：先按业务主键去重（`subset=[...]`），再检查整行重复；保留最新用 `keep="last"` 并说明依据。
5. **异常值**：用 IQR 或业务阈值标记，不直接删除；新增 `is_outlier` 列供下游选择。
6. **一致性**：类别字段统一大小写/全半角/别名映射（`Categorical` 或映射字典）；单位统一。
7. **输出报告**：每个动作前后行列数、影响比例、被丢弃样本量。
8. **保存**：`to_parquet`（保留类型）或 `to_csv(index=False, encoding="utf-8-sig")`。

## 可运行模板

```python
import pandas as pd

RAW = "raw.csv"
OUT = "clean.parquet"

df = pd.read_csv(RAW, dtype=str, keep_default_na=True)
print("raw:", df.shape)

# 1) 数值列：去千分位与货币符号
for col in ["amount", "qty"]:
    if col in df.columns:
        df[col] = pd.to_numeric(
            df[col].str.replace(r"[,\s¥$]", "", regex=True), errors="coerce"
        )

# 2) 日期：统一解析，失败项单独记录
df["date"] = pd.to_datetime(df["date"], format="%Y-%m-%d", errors="coerce")
bad_dates = df[df["date"].isna()]
print("unparsed dates:", len(bad_dates))

# 3) 缺失值：按列策略
df["amount"] = df["amount"].fillna(df["amount"].median())
df = df.dropna(subset=["id"])
df["note"] = df["note"].fillna("")

# 4) 重复：按业务主键保留最新
df["date"] = df["date"].fillna(pd.Timestamp("1970-01-01"))
df = df.sort_values("date").drop_duplicates(subset=["id"], keep="last")

# 5) 异常值标记（IQR 法，不删除）
q1, q3 = df["amount"].quantile([0.25, 0.75])
iqr = q3 - q1
df["is_outlier"] = ((df["amount"] < q1 - 1.5 * iqr) | (df["amount"] > q3 + 1.5 * iqr))
print("outliers:", int(df["is_outlier"].sum()))

# 6) 类别归一
df["channel"] = df["channel"].str.strip().str.lower()

print("clean:", df.shape)
df.to_parquet(OUT, index=False)
```

## 判定规则

- 主键列缺失值 >> 其他列 → 先查导入编码/分隔符问题，不要直接 `dropna`。
- 数值列 `errors="coerce"` 后空值比例 > 20% → 阻断，先修原始格式。
- 日期解析失败 -> 打印样例（前 5 条原始值），不要静默丢弃。
- 删除行比例 > 5% → 必须在报告中说明，并给出去除前后的分布对比。
- 类别别名映射必须显式列出映射表，禁止用「看起来一样」判断。
- 输出格式必须可复算：脚本 + 输入文件哈希 + 行数。

## 输出格式

```markdown
## 清洗报告：<数据集>

**原始**：<行 × 列>　**清洗后**：<行 × 列>　**删除比例**：<...>

| 步骤 | 列 | 动作 | 影响 | 理由 |
| --- | --- | --- | --- | --- |

### 清洗脚本
```python
<完整脚本>
```

### 遗留问题
- <无法自动判断、需业务确认的项>
```

## 纪律

- 不覆盖原始文件，输出到新文件。
- 不编造数据规模；以用户数据或实际 `df.shape` 为准。
- 清洗决策必须记录理由，避免「黑箱式清洗」。
