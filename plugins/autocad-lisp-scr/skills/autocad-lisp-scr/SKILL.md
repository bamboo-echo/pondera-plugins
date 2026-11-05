---
name: autocad-lisp-scr
description: 生成 AutoCAD 原生脚本：AutoLISP 批量改图层/统计图元，以及 .scr 命令序列，并处理换行空格陷阱与 FILEDIA/CMDDIA。当用户要在 AutoCAD 里批量整理图纸、按图层统计、或无人值守跑脚本时使用。本插件不自带 CAD 能力，只产出脚本，执行依赖用户已安装 AutoCAD。
runas: inline
---

# AutoCAD 原生脚本（AutoLISP + .scr）

## 何时使用

- 把散落在多个图层的图元归并到标准图层。
- 统计每类图元的数量、按图层汇总。
- 需要在多张图纸上无人值守执行同一串命令。

**定位说明**：本插件产出的是 AutoLISP 与 `.scr` 文本，不包含 AutoCAD 运行时；脚本效果取决于图纸与版本，必须在用户自己的 AutoCAD 中加载验证。

## 一、AutoLISP：统计图层

```lisp
(defun c:ZZCOUNT (/ ss i e ed lay tally pair)
  (setq ss (ssget "X" '((410 . "Model"))))
  (if ss
    (progn
      (setq tally nil i 0)
      (repeat (sslength ss)
        (setq e   (ssname ss i)
              ed  (entget e)
              lay (cdr (assoc 8 ed))
              i   (1+ i))
        (setq pair (assoc lay tally))
        (if pair
          (setq tally (subst (cons lay (1+ (cdr pair))) pair tally))
          (setq tally (cons (cons lay 1) tally))))
      (foreach p tally
        (princ (strcat "\n" (car p) ": " (itoa (cdr p)))))
      (princ (strcat "\nTotal: " (itoa (sslength ss))))
      (princ))
    (princ "\nNo entities in model space."))
  (princ))
```

要点：`ssget "X"` 用过滤表；`410 . "Model"` 排除布局；`assoc` 取组码 8（图层）；`subst` 更新计数；`princ` 输出到命令行。

## 二、AutoLISP：批量改图层

```lisp
;; 先选中对象，再执行 ZZMOVELAYER
(defun c:ZZMOVELAYER (/ ss i e ed)
  (setq ss (ssget "I"))
  (if ss
    (progn
      (setq i 0)
      (repeat (sslength ss)
        (setq e  (ssname ss i)
              ed (entget e))
        (if (assoc 8 ed)
          (progn
            (setq ed (subst (cons 8 "A-WALL") (assoc 8 ed) ed))
            (entmod ed)))
        (setq i (1+ i)))
      (command "_.REGEN")
      (princ (strcat "\nMoved " (itoa (sslength ss)) " objects to A-WALL.")))
    (princ "\nNothing selected."))
  (princ))
```

`ssget "I"` 取当前选择集（隐含选择）。改图层用 `entmod` 而不是 `command "_.CHPROP"`，避免命令回显污染脚本与批量执行时的确认提示。

## 三、`.scr` 命令序列模板

```text
FILEDIA
0
CMDDIA
0
_.-LAYER
_N
A-WALL
_LA
A-DOOR
_LA
A-TEXT
_
_.-PURGE
_ALL
_
_.QSAVE
FILEDIA
1
CMDDIA
1
```

执行方式：`SCRIPT` 命令选择该 `.scr`，或在启动参数里 `acad.exe /b script.scr`。

## 陷阱清单

1. **空行就是回车**：`.scr` 里空行表示「按 Enter 结束当前命令」，少一行会卡在等输入；多一行会重复上一命令。
2. **空格是分隔符**：命令名与选项用空格分隔，含空格的路径必须加引号，否则被拆成多个参数。
3. **命令前加下划线**：`_.` 前缀保证在非英文版 AutoCAD 上也能识别。
4. **`FILEDIA` 与 `CMDDIA`**：批量执行前设为 `0` 关闭文件/命令对话框，否则脚本会卡在弹窗；脚本末尾务必恢复为 `1`。
5. **`-LAYER` 带减号**：`_.-LAYER` 是命令行版；`_.LAYER` 会弹图层对话框，脚本无法继续。
6. **`_` 空参数**：`-LAYER` 的 `_` 表示接受默认值，用来结束选项菜单。
7. **文件名编码**：`.scr` 用 ANSI/UTF-8 无 BOM 保存；中文路径建议避开。
8. **版本差异**：`_.PURGE` 的选项顺序在不同版本可能不同，先在单张图上试跑。

## 输出格式

```markdown
## AutoCAD 脚本方案：<任务>

**定位**：只产出脚本，需在已安装 AutoCAD 中运行

### AutoLISP
```lisp
<完整 defun>
```

### .scr
```text
<完整命令序列，含 FILEDIA/CMDDIA 开关>
```

### 陷阱与验证
- [ ] 单张图纸试跑
- [ ] 检查图层名是否存在
- [ ] 脚本结束后 `FILEDIA=1`
- [ ] UNDO 标记：脚本首行加 `_.UNDO _M`
```

## 纪律

- 不声称脚本已在用户机器上运行成功；只声明语法与命令序列。
- 修改图元前建议 `_.UNDO _M` 打标记，便于一次撤销。
- 不硬编码图层名之外的业务规则；图层不存在时先建层或跳过并报告。
