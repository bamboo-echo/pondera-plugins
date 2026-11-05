---
description: 体检 Nginx 配置文件，列出 location 匹配、代理超时、安全头、TLS 与缓存问题
argument-hint: [nginx.conf 或站点配置路径] [关注点：性能/安全/可用性]
---

对 `$1` 做静态体检，重点关注 `$2`。**只读分析并给修改片段，不代跑、不重启 Nginx。**

**取证命令（只读）**

```bash
nginx -t -c $1            # 语法检查
nginx -T -c $1            # 展开全部 include 后的完整配置
nginx -V 2>&1 | tr ' ' '\n' | grep -E 'with-|openssl'   # 编译模块与 TLS 版本
```

**检查项**

1. `location` 匹配顺序：精确 `=` → 前缀 `^~` → 正则 `~`/`~*` → 普通前缀；确认没有正则抢先命中静态资源。
2. `try_files` 是否兜底到 `index.html`（SPA）或返回 404，而不是落到代理。
3. 反向代理：`proxy_set_header Host`、`X-Real-IP`、`X-Forwarded-For`、`X-Forwarded-Proto` 是否透传。
4. 超时：`proxy_connect_timeout`、`proxy_read_timeout`、`proxy_send_timeout` 与后端实际耗时匹配；默认 60s 是否过长。
5. 缓冲：大文件/流式接口是否需 `proxy_buffering off`；`client_max_body_size` 是否限制上传。
6. 连接复用：`upstream` 是否配 `keepalive`，是否缺 `proxy_http_version 1.1`。
7. 安全头：`Strict-Transport-Security`、`X-Content-Type-Options`、`X-Frame-Options` 或 CSP、`Referrer-Policy`。
8. TLS：是否 `ssl_protocols TLSv1.2 TLSv1.3`、是否有 session 缓存、证书链是否完整、是否开了 OCSP stapling。
9. 缓存：静态资源是否 `expires`/`Cache-Control`，HTML 是否 `no-cache`。
10. 压缩：`gzip` 类型是否覆盖 `text/css`、`application/json`、`image/svg+xml`；`gzip_vary on`。
11. 日志：`access_log` 格式是否含上游耗时 `$upstream_response_time`；错误日志级别是否合理。
12. 隐藏文件：`.git`、`.env`、备份文件是否被 deny。
13. 限流：`limit_req`/`limit_conn` 是否对登录、搜索等接口生效。
14. HTTP→HTTPS：80 端口是否只有跳转，跳转是否用 `$host$request_uri` 保留路径。

**输出格式**

```markdown
## Nginx 体检：<配置文件>

**结论**：通过 X/14，高危 N 项
**取证**：<nginx -t 结果；有报错先贴原文>

| # | 检查项 | 现状 | 判定 | 影响 |
| --- | --- | --- | --- | --- |

### 高危项与修正片段
```nginx
<可直接合并的片段>
```

### 验证
```bash
nginx -t -c <配置路径>
# 通过后再 reload（由你自行执行）
```
```

**判定与纪律**

- 语法错误优先于一切，先贴 `nginx -t` 原文。
- 缺少 `proxy_set_header Host` → `高`（后端收到错误 Host）。
- 默认 60s 读超时 + 长任务后端 → `中`，需按接口区分。
- 安全头缺失 → `高`（面向公网）。
- 不给整份重写配置，只给最小 diff；说明每段的插入位置（server/http 块）。
- 不虚构 `nginx -T` 输出，未取证项写「需现场确认」。
