---
name: sql-migration-audit
description: 审计数据库迁移脚本的锁表与不可逆风险：逐条判断 PostgreSQL / MySQL 的 DDL 锁级别、全表重写代价、数据回填与回滚路径。当用户要上线新迁移、评审 migrations 目录、或遇到迁移期间的锁超时与主从延迟时使用。
runas: inline
---

# SQL 迁移风险审计

## 何时使用

- 用户提交了 `migrations/`、`db/migrate/`、`*.sql` 变更，要求合并前评审。
- 迁移期间出现锁超时、主从延迟、应用报连接耗尽。
- 用户问「这个 DDL 能不能在线跑」。

不负责查询性能调优与索引选型，只审迁移本身的锁、重写、回滚风险。

## 执行步骤

1. 列出本次迁移涉及的表，以及每张表的行数量级。拿不到就写「未知」，不要用「应该不大」猜测。
2. 逐条 DDL 判定锁级别（见检查清单），标出是否触发全表重写。
3. 判定可逆性：能否写出无损 `down`；不能就要求备份点并明确声明「不可逆」。
4. 判定回填方式：应用层双写、分批 UPDATE、影子表三选一，写清批大小与暂停条件。
5. 给出上线顺序：先加可空列 → 回填 → 加约束；先建新索引 → 切查询 → 删旧索引。
6. 给出验证手段：迁移前后 `EXPLAIN`、行数核对、观察窗口时长。

## 检查清单

### PostgreSQL

- `ADD COLUMN ... DEFAULT <常量>`：PG 11+ 不重写；`DEFAULT now()` 或表达式会重写全表，大表禁止直接加。
- `ALTER COLUMN TYPE`：几乎总是全表重写 + ACCESS EXCLUSIVE，必须走「新列 + 回填 + 切换 + 删旧列」。
- `ADD FOREIGN KEY`：先 `NOT VALID` 加约束，再 `VALIDATE CONSTRAINT`（只取 SHARE UPDATE EXCLUSIVE）。
- `CREATE INDEX`：必须 `CONCURRENTLY`，且不能包在事务里；失败会留下 INVALID 索引，需人工清理。
- `SET NOT NULL`：先加 `CHECK (col IS NOT NULL) NOT VALID`，验证通过后再切 NOT NULL。
- `ALTER TABLE SET DEFAULT`：只改元数据，旧行仍保留旧值，别当成已回填。
- 重命名列/表：旧代码会瞬间报错，必须先双读后切换。
- `VACUUM FULL` / `CLUSTER`：排他锁，大表改用 `pg_repack`。
- `CREATE INDEX CONCURRENTLY` 期间复制延迟上升，必须约定观察窗口与回滚阈值。

### MySQL 8

- `ADD COLUMN`：仅末尾追加且未强制 `ALGORITHM=COPY` 时才 INSTANT；指定位置或生成列会退化为 COPY。
- 修改列类型、字符集、排序规则：COPY，重建表并阻塞写入。
- 加索引默认 INPLACE，大表仍吃 IO；显式 `ALGORITHM=INPLACE, LOCK=NONE` 让不满足条件时快速失败。
- 加外键：先建列与索引，再单独加约束，避免一次 DDL 里做两件事。
- 主键/分表键变更：无法在线完成，必须停机或双写迁移。

## 判定规则

- 锁级别达到 ACCESS EXCLUSIVE 且表行数 > 100 万 → **阻断上线**，必须拆步。
- 无 `WHERE` 的全表 `UPDATE` → 阻断；改分批，每批 ≤ 5000 行并单独提交。
- 缺少可执行的 `down`，或 `down` 会丢数据 → 标记不可逆，要求备份点。
- 同一迁移文件混用 DDL 与 DML → 拆成两个迁移（事务语义不同）。
- PG 的 `CREATE INDEX` 未加 `CONCURRENTLY` → 中等风险，大表升级为阻断。
- 迁移脚本里出现硬编码密码、主机名、环境分支 → 阻断。

## 输出格式

| # | 迁移文件 | 语句摘要 | 锁级别 | 行数量级 | 风险 | 修复 |
| --- | --- | --- | --- | --- | --- | --- |

风险分三档：**阻断** / **需拆步** / **可接受**。每条修复给出最小改法，不重写整个迁移体系。

## 纪律

- 不执行 `migrate up`，不连生产库，只读脚本与 schema 定义。
- 行数量级未知时保留「未知」，不要虚构数字。
- 不给服务器参数调优建议，除非用户提供当前配置与硬件规格。
