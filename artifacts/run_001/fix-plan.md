# 根因调查报告与修复方案

**产出方**：T2 调查 Team（investigate-leader）  
**运行 ID**：run_20260810_001  
**上游产物**：art_review_report_a1b2c3d4  
**状态**：✅ 通过  
**时间**：2026-08-10 08:05 → 08:12（耗时 450s）

---

## 目标缺陷

**DEF_CANON_001**：缓存并发写入导致数据竞争，偶发返回过期/不一致数据

## 复现条件

| 项目 | 内容 |
|---|---|
| 环境 | Go 1.22, Linux amd64, 4 vCPU, race detector enabled |
| 前置条件 | 缓存已预热（1 万条商品价格）；后台 Refresh 每 30s 触发 |
| 复现步骤 | 100 goroutine 并发 Get() + 同时触发 Refresh()，`go test -race -count=100` |
| 期望行为 | 无竞态警告，数据一致 |
| 实际行为 | store.go:87 报告 concurrent map writes，约 12% 运行触发 |

---

## 调查假设

### H1：近期重构移除了写锁（✅ 确认）

**视角**：最近变更  
**断言**：近期重构将写锁从 Refresh 方法中移除或缩小了临界区  
**支持证据**：`Store.set()` 方法在 L87 直接对 map 赋值，无任何锁保护（代码审查确认）  
**反驳证据**：git blame 显示 set() 自 v2.5.0 起即无锁——非近期引入；v2.8.0 新增后台 Refresh goroutine 后才暴露  
**实验**：在 set() 中添加 sync.Mutex → race test 通过

### H2：读锁未覆盖写路径（✅ 确认）

**视角**：时序/并发  
**断言**：sync.RWMutex 的读锁未正确覆盖 Refresh 期间的写路径  
**支持证据**：Refresh() 在 L52 释放写锁后，L55 仍调用 set()——写操作落到了锁外  
**反驳证据**：即使修复 Refresh 锁区间，set() 自身无锁仍存在问题——H2 是部分原因

### H3：数据结构本身导致竞态（❌ 排除）

**视角**：数据  
**断言**：缓存数据存在循环引用导致竞态  
**反驳证据**：race detector 定位精确指向 map 写入；sync.Map 替换后竞态消失

---

## 根因结论

> **Store.set() 对共享 map 的写入操作未受任何互斥锁保护，在后台 Refresh goroutine 与前台 Read goroutine 并发时产生 data race。Refresh() 的写锁区间存在缺陷（L55 set() 在 Unlock 之后调用），加剧了竞态窗口。**

**因果链**：
1. v2.5.0: set() 仅在单 goroutine Refresh 中调用，未加锁（当时安全）
2. v2.8.0: 引入高性能 Read 路径，Get() 与 Refresh() 并发
3. Refresh() sync.Mutex 在 L52 Unlock，L55 set() 暴露给并发 Get()
4. set() 无锁，直接 map 赋值 → data race

**置信度**：高

---

## 修复方案

### 选定方案 —— FIX_001（P0）

为 Store 增加 sync.RWMutex，set() 获取写锁，Get() 获取读锁

| 项目 | 内容 |
|---|---|
| 目标文件 | internal/cache/store.go |
| 预期改动 | Store 新增 mu sync.RWMutex；set() 加写锁；Get() 加读锁；简化 Refresh() 锁逻辑 |
| 影响面 | Store 所有读写方法受锁保护；RWMutex 允许并行读，读并发度不受影响 |
| 风险 | 需审计 Store 方法间互调，防止读写锁死锁 |
| 回滚策略 | 回退 commit，恢复局部 sync.Mutex 并确保 set() 在锁区间内 |
| 验收标准 | race test ×500 零警告；QPS 下降 ≤ 5%；现有单元测试全通过 |

### 备选方案 —— FIX_002（P1）

将 map 替换为 sync.Map，移除手动锁（改动面更大，类型安全性下降）

---

## 下一阶段

→ T3 修复 Team：按 FIX_001 实施修复并验证
