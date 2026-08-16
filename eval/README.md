# eval 模块：评估指标体系与验证方案

> 负责人：Liu Xiao。本模块负责回答一个核心问题——**这套多 Agent 团队是否真的更高效、更少偏见、更可靠**，并产出 [初赛方案](../proposal/初赛方案.md) 中「运行验证」章节所需的量化证据。

## 1. 评估定位

整个项目的闭环是 `Issue → T1 审查 → T2 根因定位 → T3 修复验证 → 复盘`，每个 Team 通过 `sop-schema/` 中的 JSON 契约交接产物。评估模块**不改变任何 Team 的实现**，只做两件事：

1. **端到端度量**：给一个真实缺陷，看整套流程最终能否修复、修多快、花多少钱。
2. **分阶段度量**：逐层拆解每个 Team 的产出质量，定位「到底哪一环在拖后腿」。

设计原则：**JSON 为真源，指标从产物字段直接计算**，不依赖 LLM 主观打分（除非明确标注为「软指标」）。每个指标都标注了它读的是哪个 schema 的哪个字段，保证可追溯、可复算。

## 2. 评估对象与产物来源

| Team | 产物（真源） | 评估关注的字段 |
|---|---|---|
| T1 审查 | `review-report.schema.json` | `payload.canonical_defects[]`、`payload.merged_defects[]`、`payload.dedup_decisions[]` |
| T2 调查 | `fix-plan.schema.json` | `payload.hypotheses[]`、`payload.root_cause`、`payload.fix_candidates[]` |
| T3 修复 | `fix-report.schema.json` | `payload.verification`、`payload.final_decision`、`payload.isolated_review` |
| 复盘 | `recap.schema.json` | `payload.reusable_rules[]`、`payload.new_regression_tests[]` |

## 3. 指标分层

指标分四层：端到端、分阶段、效率成本、闭环增值。完整定义见 [metrics.md](./metrics.md)。

| 层 | 回答的问题 | 代表指标 |
|---|---|---|
| **L1 端到端** | 这团队能不能修好 bug？ | resolve rate（修复率）、end-to-end 成功率 |
| **L2 分阶段** | 每个 Team 对不对？ | 去重准确率、根因定位命中率、修复门禁通过率 |
| **L3 效率成本** | 并行值不值？ | 端到端 wall-clock、token/成本、并行加速比 |
| **L4 闭环增值** | 复盘有没有用？ | 回灌规则命中率、同类缺陷二次修复耗时下降率 |

## 4. 对照实验（Ablation）

单看一套流程的绝对值，无法证明「多 Agent」比「单 Agent」强。因此评估必须包含**对照实验**：控制变量，只改一个设计要素，比较同一批缺陷上的表现。设计见 [ablation/README.md](./ablation/README.md)。

| 实验 | 验证的卖点 | 关键变量 |
|---|---|---|
| A1 多 Agent vs 单 Agent | 并行 + 去偏见带来增量 | Team 数量 / 并发度 |
| A2 信息隔离开 vs 关 | 独立评审去偏见 | T3 评审者能否看到修复推理 |
| A3 有复盘回灌 vs 无 | 知识沉淀的价值 | 是否注入 `recap.reusable_rules` |
| A4 证据门禁开 vs 关 | 证据驱动的必要性 | 是否强制 `supporting/counter_evidence` 必填 |

## 5. 数据集

评估需要**带 ground truth 的真实缺陷**——既要有 issue 描述，也要有可自动判定的「修复是否成功」的测试用例。采用 SWE-bench 风格，但针对本项目「软件测试」场景做了裁剪。设计见 [swe-bench/README.md](./swe-bench/README.md)。

判定标准：**以测试用例是否通过为准，而非 Agent 自报 `fixed`**——这是避免「自我评分幻觉」的关键。

## 6. 评估产物

一次完整评估的输出，同样遵循项目的「JSON 真源 + md 视图」约定，契约见 [schema/eval-result.schema.json](./schema/eval-result.schema.json)。汇总结果直接喂给 [初赛方案](../proposal/初赛方案.md) 的「运行验证」章节。

## 7. 目录结构

```
eval/
  README.md                 # 本文件：定位与总纲
  metrics.md                # 指标定义（含字段级映射）
  schema/
    eval-result.schema.json # 评估结果契约（JSON 真源）
  swe-bench/                # 数据集设计
    README.md
  ablation/                 # 对照实验设计
    README.md
  reports/                  # 汇总报告（运行验证章节的原料）
```

## 8. 落地顺序（方案阶段）

1. **指标定稿**：先冻结 `metrics.md`，明确「什么算修好、什么算高效」。
2. **数据集定稿**：确定缺陷语料来源与 ground truth 标注方式。
3. **实验设计定稿**：确定 ablation 的对照组与样本量。
4. **跑样例**：用手工构造的 3~5 个缺陷（参考 `artifacts/run_001` 那种粒度）走通一遍，验证指标可算。
5. **规模化**：在真实缺陷集上跑全量对照实验，产出「运行验证」数据。

> 注：当前模块交付的是**方案与契约**，不强制要求流程可运行。第 4、5 步依赖上游 Team 实现落地后再执行。
