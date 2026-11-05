---
name: go-concurrency-review
description: 审查 Go 代码的并发安全，逐项检查数据竞争、死锁、goroutine 泄漏、context 取消、锁粒度与同步原语误用，并输出可执行的修复建议。
runas: inline
---

# Go 并发安全审查

## 何时使用

- 用户要求 review 含 `go`、`chan`、`sync`、`context`、`atomic` 的 Go 代码。
- 用户报告偶发卡死、CPU 持续飙高、`data race detected`、`all goroutines are asleep`。
- 用户想优化锁粒度或减少锁竞争。

不要用于纯业务逻辑审查；并发问题只输出与并发相关的发现。

## 执行步骤

1. **划定并发边界**：列出所有 goroutine 发起位置（`go func`、`go method`、`errgroup.Go`、`time.AfterFunc`、库回调）。标出每个 goroutine 的启动者、退出条件、是否可被取消。
2. **找出共享状态**：列出被两个及以上 goroutine 读写的变量、map、slice 底层数组、结构体字段、全局变量与包级缓存。逐个确认由哪种机制保护（mutex / channel / atomic / 不可变 / 单 goroutine 独占）。
3. **逐项过下面的检查清单**，命中即记录：位置（文件:行）、风险等级、触发条件、修复方案。
4. **静态与动态验证**：给出可执行的验证命令，不要凭猜测下结论。
   ```bash
   go vet ./...
   go test -race -count=20 ./<受影响的包>
   go test -run TestX -race -timeout 30s ./...
   ```
5. **输出结论**：按 `严重 / 中等 / 建议` 三级分组；每条给出最小修复 diff 思路，不做无关重构。

## 检查清单

### 1. 数据竞争

- 同一变量在多 goroutine 中至少一处为写，且无同步 → 竞争。
- `map` 并发读写：Go 运行时会直接 `fatal error: concurrent map writes`。改用 `sync.Mutex` 保护的 map，或 `sync.Map`（仅当读多写少且键集合稳定）。
- 切片扩容竞争：`append` 到同一 slice 头会替换底层数组，务必用锁或按索引预分配后写入不同位置。
- 结构体字段被 `go func` 闭包捕获后仍在外层被改写。
- 循环变量捕获：Go 1.22 起每次迭代有新变量，Go 1.21 及更早必须 `v := v`。检查 `go.mod` 的 go 指令版本再判断。

### 2. goroutine 生命周期与泄漏

- 每个 `go func` 都必须有明确退出路径：完成、返回错误、context 取消、channel 关闭、`sync.WaitGroup` 归零。
- 向无缓冲 / 无人接收的 channel 发送会永久阻塞 → 泄漏。检查发送侧是否有 `select { case ch <- v: case <-ctx.Done(): }`。
- 常见泄漏点：`time.Tick`（不可停止，改用 `time.NewTicker` + `Stop`）、HTTP body 未关闭、`select` 里缺少 `done` 分支、启动 goroutine 后外层直接 `return`。
- 泄漏验证：`go.uber.org/goleak`（若项目已依赖）或测试结束时对比 `runtime.NumGoroutine()`。

### 3. channel 使用

- 只由**发送方**关闭 channel；多个发送方时用 `sync.Once` 或由一个协调 goroutine 关闭后广播。
- 向已关闭的 channel 发送会 panic；接收侧用 `v, ok := <-ch` 区分关闭与零值。
- 有界并发用带缓冲 channel 作信号量；注意缓冲容量与 worker 数量的关系。
- `select` 多个 case 同时就绪时随机选择，不要依赖顺序；需要优先级时用嵌套 `select`。
- `nil` channel 上的收发永久阻塞——可用于禁用某个分支，但也常是“忘了初始化”的信号，需确认意图。

### 4. context 与取消

- 每个跨调用边界的阻塞操作都应接受 `context.Context`，并把它作为第一个参数。
- 不要把 `context.Background()` 塞进请求路径绕过取消；不要在结构体里存 context。
- 派生时必须 `defer cancel()`，否则泄漏 timer 与父 context 引用。
- 子 goroutine 内的长任务要 `select { case <-ctx.Done(): return ctx.Err() ... }`。
- 用 `context.WithTimeout` 时确认超时值来自配置而非硬编码魔法数。

### 5. 锁

- `defer mu.Unlock()` 之后若还持有锁调用外部/可能阻塞的函数（网络、channel 发送、回调）→ 死锁或长时间阻塞风险。缩小临界区：先取数据，释放锁，再做 I/O。
- 同一 goroutine 重复加不可重入的 `sync.Mutex` → 自死锁。
- 加锁顺序：多把锁必须全局统一顺序，否则 A→B 与 B→A 会死锁。
- 读多写少用 `sync.RWMutex`，但注意 `RWMutex` 写饥饿与递归读锁风险（`RLock` 不可重入）。
- `sync.Once` 内的函数若 panic，Once 视为已执行，后续调用直接返回零值——初始化失败要显式记录错误。
- 复制含锁结构体（`go vet` 的 `copylocks` 会报）会复制锁状态，必须用指针或 `noCopy`。

### 6. WaitGroup 与 errgroup

- `wg.Add` 必须在 `go` 之前调用，避免 `Add` 与 `Wait` 竞争。
- `wg.Wait` 后不得再 `Add`（Go 1.25 起会 panic）。
- `errgroup.Group` 默认在第一个错误后取消派生 context；需要收集全部错误时用 `errgroup.WithContext` + 显式聚合，或 `errors.Join`。
- 需要限制并发时用 `errgroup.SetLimit(n)`，不要自己再包一层 channel 信号量。

### 7. 原子操作与内存模型

- `sync/atomic` 只能用在同一地址、同宽度的操作上；32 位平台上 `atomic` 的 int64 字段必须是结构体首字段（或使用 `atomic.Int64` 类型）。
- 用 `atomic.Bool` / `atomic.Pointer[T]` 等类型化 API，避免裸 `atomic.LoadUint32` 造成的类型不一致。
- 原子操作只保证单次读写的原子性，不保证多字段一致性；需要多字段一致时仍要用锁。
- `atomic` 不能替代 `mutex` 保护复合操作（读-改-写要用 `CompareAndSwap` 循环或锁）。

### 8. 其他常见陷阱

- 在 `init` / 包级变量里启动 goroutine，导致测试与关闭顺序不可控。
- 关闭 channel 后仍有 goroutine 写入 → panic；关闭信号应用 `close(done)` 而非发送值。
- `sync.Map` 当作普通 map 用（每次 `Load` 后类型断言）通常比加锁 map 更慢。
- 测试里用 `t.Parallel()` 却共享可变全局状态。
- `time.Sleep` 替代同步（等待另一个 goroutine 完成），在慢机器上必现 flaky。

## 输出格式

```markdown
## 并发审查结论

**范围**：<包/文件清单>
**验证**：go vet <结果>；go test -race <结果>

### 严重（会 panic / 死锁 / 数据竞争）
1. `path/file.go:42` — <问题>。触发条件：<...>。修复：<最小改动>。

### 中等（资源泄漏 / 性能退化）
1. ...

### 建议
1. ...

**未覆盖**：<需要运行期才能确认的点，如需要真实流量复现>
```

## 纪律

- 不修改代码，除非用户明确要求修复；先给结论再给 diff。
- 不把“可能有问题”写成确定结论；无法静态确认时标明“需 `-race` 复现”。
- 不为了报告数量凑条目；没有发现就明确写“未发现并发问题”。
