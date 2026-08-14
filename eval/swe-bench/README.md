# 数据集设计（SWE-bench 风格）

> 评估需要**带 ground truth 的真实缺陷**。本目录定义数据集的构造方式与字段规范，供后续填充真实语料。

## 1. 为什么用 SWE-bench 风格

SWE-bench 的核心思路是：**用真实开源仓库的历史 issue + 对应的修复 commit + 回归测试，自动判定「修复是否成功」**。

判定不靠 Agent 自报，而靠**测试用例**——这是本模块最看重的一点（对应 [metrics.md](../metrics.md) 的 1.2「测试验证修复率」）。

## 2. 数据集条目结构

每个缺陷一个目录，结构如下：

```
swe-bench/
  dataset/
    <defect_id>/
      issue.md              # 缺陷描述（喂给 T1 的输入）
      ground-truth.json     # 标注真值（见下方 schema）
      repo/                 # 缺陷代码仓库（可复现的最小样例）
      tests/                # 判定修复是否成功的测试用例
      baseline/             # 修复后的正确 patch（用于生成 ground truth）
```

### ground-truth.json 字段

| 字段 | 类型 | 说明 |
|---|---|---|
| `defect_id` | string | 唯一 ID，如 `DEF_CACHE_001` |
| `true_root_cause` | object | 真实根因：`conclusion` + `causal_chain[]` + `location` |
| `dedup_groups` | array | 判重标注：哪些 source 属于同一缺陷（供 2.1 去重准确率判定） |
| `true_fix_files` | array | 正确修复涉及的文件 |
| `fail_to_pass_tests` | array | 修复前失败、修复后应通过的测试（判定 resolve 的核心） |
| `pass_to_pass_tests` | array | 修复前后都应通过的测试（判定 regression 的核心） |
| `difficulty` | enum | `easy / medium / hard`，用于分层报告 |
| `language` | string | `go / python / ...` |

## 3. ground truth 的关键：fail-to-pass 与 pass-to-pass

SWE-bench 判定 resolve 的标准是：

- **fail-to-pass（F2P）**：修复前失败、修复后通过的测试。全部通过才算「修复成功」。
- **pass-to-pass（P2P）**：修复前后都必须保持通过的测试。任一失败即算「引入回归」。

这恰好映射到本项目的两个硬指标：

| SWE-bench 概念 | 对应 eval 指标 |
|---|---|
| F2P 全部通过 | 1.2 test-verified resolve rate |
| P2P 出现失败 | 1.3 regression rate |

## 4. 与本项目「软件测试」场景的适配

SWE-bench 原版偏向「通用 bug 修复」。本项目聚焦**软件测试与缺陷管理**，因此数据集优先选择以下类型缺陷：

1. **并发/数据竞争类**（如 `artifacts/run_001` 演示的 `concurrent map writes`）——因为方案大量讨论了 race detector、隔离评审的价值。
2. **回归类**（历史修复引入的新 bug）——验证「回退审查」和「复盘回灌」的价值。
3. **多源重复上报类**——验证 T1 去重的价值（SWE-bench 原版没有这个维度，需自行标注 `dedup_groups`）。

## 5. 数据来源（方案阶段）

按优先级排序：

1. **手工构造的最小样例**（首选，可控）：像 `artifacts/run_001` 那样，构造 3~5 个带完整 ground truth 的缺陷，先走通评估流程。
2. **SWE-bench 官方子集**：从 [SWE-bench](https://www.swebench.com/) 的 `SWE-bench_Lite` 中筛 Go/Python 且带测试的条目，补充 `dedup_groups` 标注。
3. **团队内部真实缺陷**：如果团队有真实 bug 库，去敏后使用（最有说服力，但标注成本最高）。

## 6. 数据质量要求

- **可复现**：每个缺陷必须能在隔离环境跑测试，判定确定性（避免 flaky test 污染 resolve 判定）。
- **ground truth 独立**：标注者与流程运行者隔离，避免「照着答案跑」。
- **样本分层**：难/中/易各覆盖，避免只挑简单 bug 造成「修复率虚高」。

## 7. 与上游的接口约定

eval 的 runner 需要从数据集读取 `issue.md` 喂给 T1，并在流程结束后：

1. 拿到 `fix-report.json` 里的 `payload.implementation.changed_files[]` 和 `diff_ref`。
2. 在 `repo/` 上应用该 diff。
3. 运行 `fail_to_pass_tests` 与 `pass_to_pass_tests`。
4. 产出判定结果，写入 `eval-result.schema.json` 的 `end_to_end`。

> 注：runner 的具体实现依赖上游 Team 落地，方案阶段只定义数据集结构与判定标准。
