---
name: flutter-layout-audit
description: 审查 Flutter 布局代码与构建日志：找 RenderFlex 溢出、无界高度约束、ListView 嵌套、重复重建与硬编码尺寸，输出带代码的修正方案。当用户遇到黄黑溢出条、布局在真机上错位、滚动卡顿或列表掉帧时使用。
runas: inline
---

# Flutter 布局审查

## 何时使用

- 控制台出现 `A RenderFlex overflowed by N pixels`。
- 某页面在窄屏/大字号/横屏下错位。
- 列表滚动掉帧，怀疑重复 build 或布局过深。

## 取证

```bash
flutter analyze
flutter run --profile                     # 真机或模拟器观察掉帧
# 运行中按 v 打开 Widget Inspector，或：
# debugDumpApp() / debugDumpRenderTree() 在代码中临时调用
```

在代码里临时插入：

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  debugDumpRenderTree();   // 输出 RenderObject 树，看约束与尺寸
  runApp(const MyApp());
}
```

## 常见病灶与修法

| 症状 | 根因 | 修法 |
| --- | --- | --- |
| 横向 `Row` 溢出 | 子项宽度之和超过父约束 | 用 `Expanded`/`Flexible` 包住文本，或 `Wrap` 换行 |
| `Column` 内 `ListView` 报无界高度 | 可滚动组件在无界父里 | 用 `Expanded` 包 `ListView`，或改 `shrinkWrap: true`（仅数据量小） |
| 文本被截断 | 未设 `overflow`/`maxLines` | `Text(..., maxLines: 2, overflow: TextOverflow.ellipsis)` |
| 大字号下溢出 | 固定 `height` | 改 `IntrinsicHeight` 或去掉固定高度，用 padding |
| 图片拉伸 | 未约束宽高比 | `AspectRatio(aspectRatio: 16/9, child: Image(...))` |
| 嵌套滚动抢手势 | 两个 `ListView` 同向嵌套 | 合并为一个 `CustomScrollView` + `SliverList` |

安全写法示例：

```dart
Column(
  crossAxisAlignment: CrossAxisAlignment.stretch,
  children: [
    Expanded(
      child: ListView.builder(
        itemCount: items.length,
        itemBuilder: (context, i) => ListTile(title: Text(items[i].title)),
      ),
    ),
    Padding(
      padding: const EdgeInsets.all(16),
      child: Row(
        children: [
          Expanded(
            child: Text(
              '合计：${total}',
              maxLines: 1,
              overflow: TextOverflow.ellipsis,
            ),
          ),
          const SizedBox(width: 12),
          const FilledButton(onPressed: null, child: Text('提交')),
        ],
      ),
    ),
  ],
)
```

响应式尺寸示例：

```dart
LayoutBuilder(
  builder: (context, constraints) {
    final isWide = constraints.maxWidth >= 600;
    return isWide ? _buildWide() : _buildNarrow();
  },
)
```

## 审查维度

1. **溢出**：`Row`/`Column` 内文本与图标是否都能收缩；有无固定宽高。
2. **约束**：`ListView`/`SingleChildScrollView` 是否位于无界高度父级。
3. **滚动嵌套**：同向嵌套是否合并为 sliver。
4. **重建**：`build` 里是否 new 了 `TextEditingController`、`AnimationController`、`Theme.of` 大对象；是否缺 `const`、缺 `ValueListenableBuilder`。
5. **尺寸**：是否硬编码 `MediaQuery` 宽高比例，是否考虑 `textScaleFactor`。
6. **安全区**：是否 `SafeArea`，刘海与手势条是否遮挡。
7. **无障碍**：可点区域 ≥48dp，`Semantics` 标签是否缺失。

## 输出格式

```markdown
## Flutter 布局审查：<页面>

**结论**：溢出 N 处 / 无界约束 N 处 / 重建隐患 N 处

| 位置 | 症状 | 根因 | 修法 | 风险 |
| --- | --- | --- | --- | --- |

### 修改片段
```dart
<最小改动>
```

### 验证
- `flutter analyze` 无新增告警
- 窄屏（320dp）、大字号（textScale 1.3）、横屏三档回归
```

## 纪律

- 不给整页重写，只给最小 diff。
- 不推荐 `shrinkWrap: true` 用于长列表，必须说明性能代价。
- 不编造报错文本，以用户实际日志为准；未取证写「待复现」。
- 修改后必须给出验证档位，不接受「应该好了」。
