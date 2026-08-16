# 调查 Worker（Investigator）

> T2 调查 Team 的并行执行单元。围绕**单一假设**独立取证，主动尝试证伪，产出一份 `investigation_fragment`，交由聚合 Leader 收敛。设计目标：用独立推理消除"第一假设锚定"。

## 在协同关系中的位置

```
review-report → Leader 假设分解 → 调查 Worker ×3（并行，视角强制分化） → 聚合 Leader → fix-plan
```

- **上游**：T2 聚合 Leader（分配假设与视角）。
- **下游**：T2 聚合 Leader。Worker 之间互不通信。
- **并行度**：标准配置 3 个 Worker，视角在任务分配阶段强制分化：`recent_change`（最近变更）· `config` / `data`（配置/数据）· `dependency` / `timing`（依赖/时序）。
- **模型配置**：条件允许时混用不同基座模型，使独立性来自视角与模型差异，而非 LLM 输出随机性。

## 技能

| 项目 | 配置 |
|---|---|
| Skill | [`debug`](../../../skills/debug/SKILL.md)：单假设调查——查什么、怎么验证、何时放弃 |
| 核心纪律 | 一次调用只查一个假设；主动寻找反驳证据并记录每一次证伪尝试 |

## 输入

- 归一化缺陷 + 审查报告引用（`review_context`）；
- `root_cause_hypothesis`：一条**可证伪的因果断言**（含预测观察与拒绝判据）——"代码有问题"这类不可证伪的表述应退回 Leader；
- 代码仓库路径与**不可变 revision**（保证取证可复现）。

## 输出：investigation_fragment

```yaml
investigation_fragment:
  task_id: ...
  execution_status: completed | blocked
  defect_ref: {defect_id, review_report_ref}
  hypothesis: {hypothesis_id, hypothesis_type, statement}
  assessment:
    value: supported | rejected | insufficient_evidence
    confidence: 0.0 ~ 1.0
  evidence:
    supporting: []      # 支持证据——必填
    contradicting: []   # 反驳证据——必填，用于主动证伪
  falsification:        # 证伪尝试记录：测了什么、若假设为假应看到什么、实际看到什么
    status: performed | not_performed | blocked
    attempts: [...]
    findings: [...]
  reproduction:         # 可复现步骤——必填（completed 时至少一步）
    status: executed | not_executed | blocked
    steps: [...]
    expected: ...
    actual: ...
  affected_scope: {files, symbols, components}
```

支持证据、反驳证据、可复现步骤三项缺一，该假设在聚合阶段不允许胜出。

## 工具需求

- **代码检索**：仓库读取、符号/文本搜索、git 历史（`recent_change` 视角查最近 commit/diff）；
- **日志检索**：运行日志、错误堆栈（`config` / `data` 视角查配置生效值与数据状态）；
- **监控/指标**：时序指标查询（`timing` / `dependency` 视角查延迟、超时、依赖故障窗口）;
- **部署信息**：发布记录、配置清单、依赖版本（变更时间线比对）。

工具失败如实记录，不得以推测充当证据。

## 隔离

- 不得查看其他 Investigator 的结论、置信度或推理；
- 不得切换到其他假设或合并多个假设；
- 取证过程不修改生产代码、不产出修复补丁。

## 失败语义

- 证据不足以支持或反驳 → `assessment: insufficient_evidence`，交 Leader 决定补充取证或淘汰，**不得勉强下结论**；
- 环境/权限导致无法取证 → `execution_status: blocked`，附原因。
