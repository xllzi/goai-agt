# 审查报告

**产出方**：T1 审查 Team（review-leader）  
**运行 ID**：run_20260810_001  
**状态**：✅ 通过  
**时间**：2026-08-10 08:00 → 08:03（耗时 225s）

---

## 缺陷概要

| 字段 | 值 |
|---|---|
| 主缺陷 ID | DEF_CANON_001 |
| 标题 | 缓存并发写入导致数据竞争，偶发返回过期/不一致数据 |
| 严重程度 | 高 |
| 优先级 | P0 |
| 影响版本 | v2.8.0, v2.8.1 |
| 影响组件 | service-cache, api-gateway |

## 缺陷描述

API 服务在并发请求下对共享缓存 map 的写入未持有互斥锁，导致数据竞争。表现为用户刷新页面偶见价格不一致、订单列表与详情页数据矛盾，重启或缓存过期后自动恢复。

## 去重决策

对多源上报的缺陷执行判重，结论如下：

| 对比缺陷 | 来源 | 判定 | 证据 |
|---|---|---|---|
| FB-8892 | 用户反馈 | 同一缺陷 | 堆栈指纹一致（store.go:87）；触发路径相同；版本重叠 |

### 被合并的缺陷
- **PROJ-4201**（Issue）→ DEF_CANON_001
- **FB-8892**（用户反馈）→ DEF_CANON_001
- **prod-20260809-error.log**（日志）→ DEF_CANON_001

## 堆栈指纹

```
fatal error: concurrent map writes
goroutine 42 [running]:
internal/cache.(*Store).set(...)
    /app/internal/cache/store.go:87
```

## 复现线索

- `go test -race -count=100 ./internal/cache/...` 可稳定触发竞态检测
- 并发场景：100 goroutine 并发 Read + 1 goroutine 后台 Refresh

## 调查范围

- `internal/cache/store.go` — 并发写入路径的锁机制
- `internal/cache/refresh.go` — 后台刷新 goroutine 的同步策略
- `go.mod` — 是否已使用 sync.Map 或并发安全库

## 下一阶段

→ T2 调查 Team：根因定位与修复方案
