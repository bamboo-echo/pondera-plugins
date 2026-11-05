---
description: 诊断 Dockerfile 体积与缓存失效，输出可对比的瘦身改造方案
argument-hint: [Dockerfile 路径] [目标镜像上限，例如 200MB]
---

对 `$1` 做体积与构建效率诊断，目标体积 `$2`（未给出时按「能删的全删，保留运行必需」处理）。

**执行步骤**

1. 读取 Dockerfile 与 `.dockerignore`，报告基础镜像、阶段数、每层安装内容、最终阶段暴露的文件。
2. 按四类问题定位，每类给行号与证据：
   - **基础镜像过大**：`ubuntu`/`debian` 全量 → `debian:*-slim` 或 `alpine`（注意 glibc 差异）。
   - **构建依赖混入运行镜像**：编译工具链、包管理器缓存未在多阶段中隔离。
   - **层未合并 / 缓存失效**：`COPY . .` 早于依赖安装；`RUN apt-get update && install` 分离；未清理 `/var/cache`。
   - **多余文件**：`.git`、测试数据、文档、本地构建产物未通过 `.dockerignore` 排除。
3. 给出改造后的 Dockerfile 草案：多阶段（builder + runtime）、依赖层先复制锁文件、非 root 用户、固定基础镜像 digest、`--mount=type=cache` 语法（BuildKit）标注为可选。
4. 给出验证命令，让用户自行执行（不要替用户构建）：
   ```bash
   docker build --progress=plain -t <tag> .
   docker images --format '{{.Repository}}:{{.Tag}} {{.Size}}'
   docker history <tag> --no-trunc
   ```
5. 估算收益：能说明来源的（例如 apt 缓存约数百 MB）给区间；不能量化的写「需构建后实测」。

**输出格式**

```markdown
## Dockerfile 诊断：<路径>

**现状**：基础镜像 <image>，<N> 个阶段，<M> 条 RUN
**目标**：<体积上限>

### 问题清单
| # | 行号 | 问题 | 影响 | 改法 |

### 改造草案
```dockerfile
<完整草案>
```

### 预期收益
| 项 | 现状 | 改造后 | 依据 |

### 需要你知道的取舍
- alpine vs slim：<musl 兼容性风险>
- 固定 digest 与自动升级：<...>
```

**纪律**

- 不执行 `docker build`/`pull`；需要拉取镜像时提示用户自行操作。
- 不为了瘦身牺牲可调试性（例如删除所有 shell），必须说明代价。
- 草案保持原项目的构建约定（包管理器、入口命令），不做无关重构。
