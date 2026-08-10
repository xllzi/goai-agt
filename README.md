# goai-agt

面向软件测试场景的多 Agent 团队：以**并行执行**换效率、**独立推理**去偏见，形成「审查 → 根因定位 → 修复验证 → 复盘沉淀」的测试闭环。设计详见 [design.md](./proposal/design.md)。

## 结构

```
teams/               并行执行单元（Team 内联角色：N×worker + Leader 聚合）
  review/            T1 代码审查：N×审查 Agent + 聚合 Agent(Leader)
  investigate/       T2 根因定位：N×调查 Agent + 聚合 Agent(Leader)
  fix/               T3 修复验证：修复 Agent(Leader) + N×测试 Agent
workflow/            全程工作流
sop-schema/          Team 间交接的 SOP 契约（JSON Schema + 示例产物）
  review-report.schema.json  T1 → T2 审查报告契约
  fix-plan.schema.json       T2 → T3 修复方案契约
  fix-report.schema.json     T3 → Recapper 修复报告契约
  recap.schema.json          Recapper 知识沉淀契约
artifacts/           各次闭环执行的结构化产物
  <run_id>/
    review-report.json / .md
    fix-plan.json / .md
    fix-report.json / .md
    recap.json / .md
    evidence/         独立附件（日志/diff/测试报告/覆盖率），JSON 通过 uri 引用
standalone-agents/   独立于 Team 流程的 Agent
  recapper/          流程收尾，知识沉淀
skills/              技能包（待定）：code-review / issue-prd-writing / testing / debug
tools/               工具（待定）：codebase / test-runner / static-analysis
eval/                评估指标和方式
```

## 流程

```
Issues → T1 审查 → review-report → T2 根因定位 → fix-plan → T3 修复验证 → fix-report
                                                              ├─ 成功 → recapper 复盘沉淀
                                                              └─ 全部无效 → 回退 T1 重审
```

