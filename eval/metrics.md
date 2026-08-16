# 评估指标体系

> 本文档定义 eval 模块的全部指标。约定：**每个指标都标注「数据来源字段」**（取自 `sop-schema/` 下的 JSON 产物），保证可复算。指标分「硬指标」（从字段机械计算）与「软指标」（需 LLM 判定，需注明）。

## 0. 通用符号

- 一次闭环（run）：一个缺陷从 T1 走到复盘或回退的全过程。
- 产物：`review-report` / `fix-plan` / `fix-report` / `recap` 四个 JSON。
- **ground truth**：数据集预先标注的正确答案（真实根因、修复是否成立、应合并/拆分的缺陷对）。

---

## L1 端到端指标

回答：**这团队能不能修好 bug？**

### 1.1 resolve rate（修复率）
- **定义**：`最终判定为已修复的 run 数 / 总 run 数`
- **来源**：`fix-report.json` → `payload.final_decision.result == "fixed"`
- **注意**：这是「Agent 自报」，不可单独采信，必须与 1.2 的「测试判定」交叉验证。

### 1.2 test-verified resolve rate（测试验证修复率）
- **定义**：`修复后数据集 ground-truth 测试用例全部通过 的 run 数 / 总 run 数`
- **来源**：数据集测试用例执行结果（**不来自** Agent 产物）
- **意义**：这是**最硬的指标**。若 1.2 显著低于 1.1，说明存在「自我评分幻觉」——Agent 报 `fixed` 但测试没过。

### 1.3 regression rate（回归率）
- **定义**：`修复后引入新失败测试 的 run 数 / 已修复 run 数`
- **来源**：`fix-report.json` → `payload.verification.failed_tests[]` + 数据集 pre-existing 测试基线
- **意义**：衡量「修好一个、弄坏一堆」的爆炸半径。对应方案痛点三。

### 1.4 回退率（rollback/redo rate）
- **定义**：`触发「回退 T1 重新审查」的 run 数 / 总 run 数`
- **来源**：`fix-report.json` → `payload.final_decision.next_action.target == "review"` 或 `fix-plan.next_action.target == "review"`
- **意义**：衡量流程是否在错误根因上空转（方案强调「修复无效回退审查而非重复修复」）。

---

## L2 分阶段指标

回答：**每个 Team 对不对？**

### 2.1 去重准确率（T1）
- **定义**：`判重决策正确的缺陷对数 / 总判重决策数`
- **来源**：`review-report.json` → `payload.canonical_defects[].dedup_decisions[].verdict` vs 数据集标注的 `same/related/different`
- **细分**：
  - **误合并率**：把「不同缺陷」判成「同一缺陷」的比例（漏修风险）
  - **误拆分率**：把「同一缺陷」判成「不同缺陷」的比例（重复修风险）
- **软指标**：需要与 ground truth 对齐判定。

### 2.2 根因定位命中率（T2）
- **定义**：`root_cause 与 ground-truth 根因一致 的 run 数 / 总 run 数`
- **来源**：`fix-plan.json` → `payload.root_cause.conclusion` vs 数据集标注根因
- **软指标**：语义比对（LLM 判定或人工核对），建议用「是否指向同一代码位置 + 同一因果链」双条件。

### 2.3 假设淘汰有效性（T2）
- **定义**：`被正确淘汰的错误假设数 / 被淘汰的错误假设数`
- **来源**：`fix-plan.json` → `payload.hypotheses[]`，其中 `result == "rejected"` 的项 vs 数据集标注的真实错误假设
- **意义**：验证「主动证伪」是否真的在工作（方案强调 `counter_evidence` 必填、主动证伪）。

### 2.4 修复门禁通过率（T3）
- **定义**：`隔离评审 verdict == passed 且 复现测试红转绿 的 run 数 / 进入 T3 的 run 数`
- **来源**：`fix-report.json` → `payload.isolated_review.verdict == "passed"` 且 `payload.verification.reproduction_test.before_fix=="failed" && after_fix=="passed"`
- **意义**：验证「先红后绿」门禁是否严格执行（方案强调不可跳过）。

---

## L3 效率成本指标

回答：**并行值不值？**

### 3.1 端到端 wall-clock（墙钟时间）
- **定义**：`最后一个产物 completed_at − 第一个产物 started_at`
- **来源**：各产物 `started_at` / `completed_at` 字段的并集区间
- **意义**：这是「并行」的直接体现。若 Team 内 Worker 真正并行，端到端时间应显著小于「各 Worker 耗时之和」。

### 3.2 并行加速比（speedup）
- **定义**：`Σ(各 Worker 独立耗时) / 端到端 wall-clock`
- **来源**：需要每个 Worker 的独立 `duration_ms`（若产物仅记录 Leader 汇总，则此指标暂不可算，标注为「待实现埋点」）
- **意义**：最直接证明「并行提效」的指标。

### 3.3 单位修复成本（token / 美元）
- **定义**：`Σ(各 Agent 消耗 token 或费用) / 已修复 run 数`
- **来源**：`producer.model` + 运行日志（需在 Team 实现中埋点记录 token）
- **软指标/依赖埋点**：当前 schema 未含 token 字段，需上游补充或从运行日志提取。

### 3.4 平均重试次数
- **定义**：`Σ(run 内回退次数) / 总 run 数`
- **来源**：产物链中 `next_action.target == "review"` 的出现次数
- **意义**：结合成本，衡量「多 Agent 是否省了钱」而非「花了更多钱多跑几轮」。

---

## L4 闭环增值指标

回答：**复盘有没有用？**

### 4.1 回灌规则命中率
- **定义**：`后续 run 中触发历史 reusable_rules 的缺陷数 / 与历史缺陷同类的后续缺陷数`
- **来源**：`recap.json` → `payload.reusable_rules[].target` vs 后续 `review-report`/`fix-plan` 是否命中对应规则
- **软指标**：需判定「规则是否本应适用」。

### 4.2 同类缺陷二次修复耗时下降率
- **定义**：`(首次同类缺陷平均耗时 − 后续同类缺陷平均耗时) / 首次平均耗时`
- **来源**：跨 run 的 wall-clock 对比，按 `recap.json` 的 `knowledge_tags[]` 聚类
- **意义**：知识沉淀价值的直接量化。若为负（越修越慢），说明复盘没产生正收益。

### 4.3 逃逸缺陷转化率
- **定义**：`recap 中 new_regression_tests 被纳入后续 T3 测试套件的比例`
- **来源**：`recap.json` → `payload.new_regression_tests[]` vs 后续 `fix-report` 的 `verification.test_suites[]`
- **意义**：验证「逃逸回归 → 新测试」的闭环是否真的发生。

---

## 指标汇总表

| 编号 | 指标 | 层 | 类型 | 数据来源字段 |
|---|---|---|---|---|
| 1.1 | resolve rate | L1 | 硬 | `fix-report.final_decision.result` |
| 1.2 | test-verified resolve rate | L1 | 硬 | 数据集测试结果 |
| 1.3 | regression rate | L1 | 硬 | `fix-report.verification.failed_tests` |
| 1.4 | 回退率 | L1 | 硬 | `next_action.target=="review"` |
| 2.1 | 去重准确率 | L2 | 软 | `review-report.dedup_decisions.verdict` |
| 2.2 | 根因定位命中率 | L2 | 软 | `fix-plan.root_cause.conclusion` |
| 2.3 | 假设淘汰有效性 | L2 | 软 | `fix-plan.hypotheses[].result` |
| 2.4 | 修复门禁通过率 | L2 | 硬 | `fix-report.isolated_review.verdict` + `reproduction_test` |
| 3.1 | 端到端 wall-clock | L3 | 硬 | 各产物 `started_at`/`completed_at` |
| 3.2 | 并行加速比 | L3 | 硬* | Worker 级 `duration_ms`（需埋点） |
| 3.3 | 单位修复成本 | L3 | 软* | `producer.model` + 运行日志（需埋点） |
| 3.4 | 平均重试次数 | L3 | 硬 | `next_action.target=="review"` 计数 |
| 4.1 | 回灌规则命中率 | L4 | 软 | `recap.reusable_rules[].target` |
| 4.2 | 二次修复耗时下降率 | L4 | 硬 | 跨 run wall-clock 对比 |
| 4.3 | 逃逸缺陷转化率 | L4 | 硬 | `recap.new_regression_tests[]` |

> `*` 表示依赖上游埋点，当前 schema 可能未覆盖，需在实现阶段补充。
