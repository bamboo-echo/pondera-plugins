---
name: systemd-unit-audit
description: 审查 systemd service/timer 单元：重启与失败处理、依赖顺序、权限最小化、资源限制、日志与超时，输出问题清单和可直接替换的 unit 片段。当用户要部署守护进程、排查开机启动失败、或给现有 unit 做安全加固时使用。
runas: inline
---

# systemd 单元审查

## 何时使用

- 新写了一个 `*.service`，想在上线前确认没有隐患。
- 服务重启后起不来、频繁重启、日志看不出原因。
- 要给已有 unit 做最小权限加固与资源限制。

## 审查维度

| # | 维度 | 检查点 |
| --- | --- | --- |
| 1 | `[Unit]` | `Description`、`After`/`Requires`/`Wants` 是否正确表达依赖 |
| 2 | 启动类型 | `Type=simple/exec/notify/forking` 是否与进程行为一致 |
| 3 | 用户 | 是否用 `User=` + `Group=` 降权，`DynamicUser=` 是否更合适 |
| 4 | 工作目录 | `WorkingDirectory=` 是否显式，相对路径是否会因 cwd 漂移出错 |
| 5 | 重启策略 | `Restart=on-failure` + `RestartSec=`，避免无限快速重启 |
| 6 | 退出码 | `SuccessExitStatus=` 是否覆盖正常退出码（如 143/SIGTERM） |
| 7 | 超时 | `TimeoutStartSec` / `TimeoutStopSec` 是否匹配实际启动与优雅退出耗时 |
| 8 | 停止信号 | `KillSignal=`、`KillMode=` 是否让进程能优雅收尾 |
| 9 | 资源 | `MemoryMax`、`CPUQuota`、`TasksMax`、`LimitNOFILE` |
| 10 | 加固 | `NoNewPrivileges`、`ProtectSystem=strict`、`ProtectHome`、`PrivateTmp`、`PrivateDevices`、`ReadWritePaths` |
| 11 | 网络 | `RestrictAddressFamilies`、`IPAddressAllow` 是否过宽 |
| 12 | 日志 | 是否走 journald（不重定向到文件），日志级别是否可调 |
| 13 | 环境 | `Environment=`/`EnvironmentFile=` 是否含密钥；密钥不该放 unit 文件里 |
| 14 | 定时器 | timer 是否配 `Persistent=true`、`RandomizedDelaySec`，避免同时触发 |

## 执行步骤

1. 先取证：`systemctl cat <unit>`、`systemctl show <unit> -p Restart,User,ExecStart,TimeoutStopUSec`、`journalctl -u <unit> -n 100 --no-pager`；只读，不改服务。
2. 对照上表逐项判定 `通过` / `风险` / `缺失`，每项写清现值和期望值。
3. 给出改进片段：只给需要改的键，不整份重写，避免用户误覆盖。
4. 说明验证方式：`systemd-analyze verify <unit文件路径>`、`systemctl daemon-reload`、再 `systemctl restart` 后看 `systemctl status`。
5. 高风险项单独置顶（无限重启、以 root 运行、无超时、密钥内联）。

## 可套用模板

```ini
[Unit]
Description=<服务名>
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=simple
User=<专用用户>
Group=<专用组>
WorkingDirectory=/opt/<服务名>
ExecStart=/opt/<服务名>/bin/<可执行文件> --config /etc/<服务名>/config.yaml
Restart=on-failure
RestartSec=5
SuccessExitStatus=143
TimeoutStartSec=30
TimeoutStopSec=30
KillSignal=SIGTERM
EnvironmentFile=/etc/<服务名>/env
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ReadWritePaths=/var/lib/<服务名>
MemoryMax=512M
TasksMax=256
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

验证：

```bash
systemd-analyze verify /etc/systemd/system/<服务名>.service
systemctl daemon-reload
systemctl restart <服务名> && systemctl status <服务名> --no-pager
journalctl -u <服务名> -n 50 --no-pager
```

## 输出格式

```markdown
## systemd 审查：<unit>

**结论**：通过 X/14，高风险 N 项
**取证**：<systemctl cat / show 的关键输出>

| 维度 | 现状 | 判定 | 建议 |
| --- | --- | --- | --- |

### 高风险项
- **<项>**：<后果> → <改法>

### 改进片段
```ini
<只含需修改的键>
```

### 验证步骤
```

## 纪律

- 不修改用户机器上的 unit 文件，只输出片段。
- 不虚构 `systemctl show` 的输出；未取证就写「需现场确认」。
- 加固建议必须同时说明副作用（例如 `ProtectSystem=strict` 会让写路径失效，需配 `ReadWritePaths`）。
- 不推荐 `Restart=always` 搭配无 `StartLimit*` 的组合。
