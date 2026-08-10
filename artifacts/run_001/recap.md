# 复盘知识条目

**产出方**：独立复盘 Agent（recapper）  
**运行 ID**：run_20260810_001  
**上游产物**：art_fix_report_i9j0k1l2  
**状态**：✅ 闭环完成  
**时间**：2026-08-10 08:23 → 08:25（耗时 120s）

---

## 缺陷摘要

service-cache 组件中 `Store.set()` 对共享 map 的并发写入未受互斥锁保护，v2.8.0 引入并发读路径后暴露 data race，API 偶发返回过期/不一致数据。根因：**设计假设变更（set() 从单 goroutine 调用变为并发调用）未及时加固**。

## 根因模式

**单线程设计假设失效**：原设计假定 set() 仅在单个 Refresh goroutine 中调用，系统演进为并发读写时该假定被打破，但未被显式审查或测试捕获。

## 触发条件

- 后台 Refresh 与前台 Read 并发执行
- map 写入未受互斥锁保护
- Refresh() 的锁区间未完全覆盖所有写操作

## 解决方案

为 Store 增加 `sync.RWMutex`，所有读写路径统一保护。修复仅涉及 1 个文件，race test ×500 零警告，无性能退化。

## 有效证据回顾

| 证据 | 为何有效 |
|---|---|
| EV_H1_S1（代码审查：无锁写入点） | 最小化、最精确的证据类型 |
| EV_FIX_004（race test ×500） | 统计显著性保证，远高于单次测试 |

## 误导信号

| 信号 | 为何误导 |
|---|---|
| git blame → v2.5.0 即无锁（"陈年代码"） | 代码未变但调用上下文变了；blame 会误导为"一直没问题所以不用改" |

## 失败假设

| 假设 | 失败原因 |
|---|---|
| H3：数据结构循环引用导致竞态 | race detector 精确指向 map 写入；sync.Map 替换后竞态消失 |

## 回归教训

本次缺陷的逃逸路径：单 goroutine 安全 → 多 goroutine 不安全，但**缺乏并发安全门禁**。任何涉及共享状态的代码变更必须伴随 race detector 回归测试，尤其是调用方并发模型变化时。

## 新增回归测试

| 测试 | 类型 | 覆盖点 |
|---|---|---|
| TestConcurrentReadWrite | unit | 并发读+写的 data race 检测 |
| TestRefreshLockCoverage | unit | Refresh 锁区间覆盖所有写操作 |

## 可回灌规则

### → T1（审查 Team）
当缺陷堆栈指纹包含 `concurrent map writes` 时，自动标记为并发安全类缺陷，调查范围应包括调用方并发模型变更历史。
> **理由**：本缺陷从上报到去重仅用了堆栈指纹，但未在审查阶段识别并发模型演进风险。

### → T2（调查 Team）
git blame 显示"陈年代码无锁"时，必须额外检查调用方并发模型的变更历史（如新增 goroutine、并发读路径），防止"一直没问题"的认知偏差。
> **理由**：H1 调查中 blame 误导了时间线——代码未变但调用上下文变了。

### → T3（修复 Team）
涉及共享状态变更的修复，验收标准必须包含 `race detector × N（N ≥ 100）` 统计显著性测试，不满足不得通过。
> **理由**：单次 race test 假阴性率高；本缺陷 ×100 测试中约 12% 触发，×500 提供足够置信度。

## 标签

`concurrency` `data-race` `cache` `sync.RWMutex` `race-detector` `design-assumption` `regression-testing`

## 后续行动

- 全仓库扫描：对所有使用 map 且存在并发 goroutine 的模块执行 race test 审计
- CI 门禁增强：pre-commit hook 强制 race detector（至少 -count=10）
- 文档更新：service-cache 的 CONTEXT.md 记录并发安全约束
