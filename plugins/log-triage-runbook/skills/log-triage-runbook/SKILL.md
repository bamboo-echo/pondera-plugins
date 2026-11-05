---
name: log-triage-runbook
description: 从日志碎片收敛故障假设：按时间线对齐、按特征聚类、给出可复制的 journalctl/grep 取证命令与判定条件，最后输出排障记录。当线上服务报错、间歇性失败、或需要从一大堆日志里定位根因时使用。
runas: inline
---

# 日志排障手册

## 何时使用

- 只知道「服务挂了」「接口 502」，日志量大且噪音多。
- 间歇性故障，需要判断是否与时间、并发、上游依赖相关。
- 要产出一份可交接的排障记录。

## 排障流程

### 1. 定界

先确认四件事，缺一项就先问用户：**发生时间窗**、**影响范围**（单实例/全集群）、**可复现性**（必现/偶发）、**最近变更**（发布、配置、证书、配额）。

### 2. 取证（只读命令）

```bash
# 指定单元与时间窗
journalctl -u <service> --since "2026-11-05 09:00" --until "2026-11-05 09:30" --no-pager

# 只看错误级别
journalctl -u <service> -p err --since "30 min ago" --no-pager

# 统计高频错误（按消息去数字后聚类）
journalctl -u <service> --since "1 hour ago" --no-pager \
  | sed -E 's/[0-9]+/N/g' | sort | uniq -c | sort -rn | head -20

# 应用日志：抓时间窗 + 排除噪音
grep -E '2026-11-05T09:(0[0-9]|1[0-9]|2[0-9])' app.log | grep -vE 'DEBUG|healthcheck'

# 找首个异常（不要只看最后一条）
grep -n -m 5 -E 'ERROR|FATAL|panic|timeout|refused' app.log
```

Windows / PowerShell 等价：

```powershell
Get-Content .\app.log | Select-String -Pattern '09:0[0-9]|09:1[0-9]' -Context 2,2
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=(Get-Date).AddMinutes(-30)} |
  Where-Object LevelDisplayName -in 'Error','Critical'
```

### 3. 时间线对齐

按时间戳把「第一个异常 → 爆发点 → 恢复点」排成一条线，标注每次发布/配置变更/依赖抖动。不要从最响的错误开始，先找**第一个**异常。

### 4. 假设与判定

每个假设必须写出「什么现象支持」「什么现象会否决」，并按优先级取证：

| 假设 | 支持证据 | 否决证据 | 取证命令 |
| --- | --- | --- | --- |
| 依赖超时 | 上游耗时突增、`context deadline exceeded` | 上游健康检查正常 | 上游日志 + `curl -w '%{time_total}'` |
| 连接池耗尽 | `too many open files`、`pool timeout` | fd 数正常 | `ss -s`、`ls /proc/<pid>/fd \| wc -l` |
| 内存压力 | OOMKilled、GC 尖峰 | 内存平稳 | `journalctl -k \| grep -i oom` |
| 证书过期 | `x509: certificate has expired` | 证书有效期内 | `openssl s_client -connect host:443 -servername host` |
| 磁盘写满 | `no space left on device` | `df -h` 正常 | `df -h`、`du -sh /var/log` |

### 5. 收敛

只保留未被证伪的假设。若两个假设都成立，判断先后关系（谁触发了谁），并说明为什么不是另一个。

### 6. 产出排障记录

```markdown
## 排障记录：<现象>

**时间线**
| 时间 | 事件 | 证据来源 |
| --- | --- | --- |

**根因**：<一句话>　**置信度**：高/中/低
**已排除**：<假设 → 否决证据>

**关键日志**
```text
<逐字摘录 10 行以内>
```

**修复建议**：<最小改动>
**验证方式**：<命令 + 期望输出>
**复盘项**：<监控/告警/日志字段缺失>
```

## 判定规则

- 日志里出现两种错误时，按时间先后判断因果，不按出现次数。
- 错误计数多 ≠ 根因，通常是根因的后果（重试风暴）。
- 没有时间戳的日志先补时间戳，否则不做因果推断。
- 摘录日志必须逐字，不改写、不省略中间行而不标注 `...`。
- 置信度低时必须写明「还缺什么证据」，不给斩钉截铁的结论。

## 纪律

- 只读排障，不擅自重启服务、不删日志、不改配置。
- 不虚构命令输出；没跑过的命令标注「待执行」。
- 涉及生产变更的建议单独列出，提醒在变更窗口执行并回滚预案。
