---
description: 用 Power Query / M 语言清洗大表，并给出百万行 Excel 的容量与性能方案
argument-hint: [数据源与表结构] [目标：清洗/汇总/建模]
---

为 `$1` 设计 Power Query 清洗方案，目标是 `$2`。**只产出查询与步骤，需你在 Excel 的「数据 → 获取数据」里粘贴执行。**

**先判断容量**

| 场景 | 上限/代价 | 建议 |
| --- | --- | --- |
| 单个工作表 | 1,048,576 行 × 16,384 列 | 超限必须拆分或改用数据模型 |
| xlsx 文件本身 | 单文件建议 < 100 MB | 更大改 CSV/parquet 或数据库 |
| 百万行加载到工作表 | 打开与保存都很慢 | **只加载到数据模型**，不落到工作表 |
| 公式列 | 全表逐格计算 | 改用 Power Query 计算列 |
| VLOOKUP 百万行 | O(n²) 级别 | 改用合并查询（Merge）或 XLOOKUP |

**可粘贴的 M 查询**

```text
let
    Source = Excel.Workbook(File.Contents("C:\data\sales.xlsx"), null, true),
    Sheet = Source{[Item="Sheet1", Kind="Sheet"]}[Data],
    Promoted = Table.PromoteHeaders(Sheet, [PromoteAllScalars=true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"日期", type date},
        {"区域", type text},
        {"金额", type number},
        {"数量", Int64.Type}
    }),
    DroppedNull = Table.SelectRows(Typed, each [金额] <> null and [数量] <> null),
    Trimmed = Table.TransformColumns(DroppedNull, {
        {"区域", each Text.Trim(Text.Upper(_)), type text}
    }),
    Grouped = Table.Group(Trimmed, {"区域", "日期"}, {
        {"总额", each List.Sum([金额]), type number},
        {"总数量", each List.Sum([数量]), Int64.Type},
        {"行数", each Table.RowCount(_), Int64.Type}
    }),
    Sorted = Table.Sort(Grouped, {{"日期", Order.Ascending}, {"总额", Order.Descending}})
in
    Sorted
```

要点：`File.Contents` 读文件、`Excel.Workbook` 取工作表或命名区域、`Table.PromoteHeaders` 提升表头、`Table.TransformColumnTypes` 显式定类型（避免「任意」列）、`Table.Group` 做聚合。

**多表合并（替代 VLOOKUP）**

```text
let
    Orders = Excel.CurrentWorkbook(){[Name="Orders"]}[Content],
    Prices = Excel.CurrentWorkbook(){[Name="Prices"]}[Content],
    Merged = Table.NestedJoin(Orders, {"SKU"}, Prices, {"SKU"}, "P", JoinKind.LeftOuter),
    Expanded = Table.ExpandTableColumn(Merged, "P", {"单价"}, {"单价"}),
    WithAmount = Table.AddColumn(Expanded, "金额", each [数量] * [单价], type number)
in
    WithAmount
```

**执行步骤**

1. 先看数据源：Excel 工作簿、CSV、数据库还是文件夹合并。
2. 定类型：日期、数值、文本必须显式声明，禁止留「任意」。
3. 清洗顺序：删空行 → 去重 → 修文本 → 补缺失 → 类型转换（转换放最后，避免错误值扩散）。
4. 聚合尽量在 Power Query 里做，不用工作表公式。
5. 加载方式：`仅创建连接` + 勾选「将此数据添加到数据模型」，用数据透视表读模型。
6. 性能：删掉中间步骤、避免整表排序、能用 `Table.SelectRows` 先过滤就不过滤后加载。

**输出格式**

```markdown
## Power Query 方案：<数据源>

**容量判断**：<行数/列数/是否超限>
**加载方式**：工作表 / 仅数据模型

### M 查询
```text
<完整 let ... in>
```

### 步骤说明
| 步骤 | 作用 | 可删性 |
| --- | --- | --- |

### 性能与容量
- 过滤后行数估计：<...>
- 是否启用查询折叠：<是/否>
```

**判定规则**

- 数据超过 1,048,576 行仍要求加载到工作表 → 阻断，改数据模型或 CSV。
- 类型列出现「任意」→ `高`，会导致后续聚合报错。
- 用 VLOOKUP 处理十万行以上 → 要求改合并查询。
- 有重复主键但未去重 → 明确提示 `Table.Distinct`。
- 查询步骤超过 20 步且无中间命名 → 建议拆成多个查询。

**纪律**

- 不虚构文件路径与工作表名；按用户实际结构给。
- 不承诺具体耗时，只给相对优化建议。
- 涉及覆盖源文件的操作必须先备份。
