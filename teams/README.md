# teams/ — 并行执行单元

Team 内联角色：N×Worker 并行执行 + Leader 聚合裁决。每个角色目录下的 `ROLE.md` 是该角色的设计文档（职责、输入输出契约、边界、失败语义）。

| Team | 角色 | 职责一句话 | 产物 |
|---|---|---|---|
| `review/` | `reviewer/` ×3 + `aggregator/` | 多判据并行判重，按证据质量裁决 | review-report |
| `investigate/` | `investigator/` ×3 + `aggregator/` | 多假设并行取证、主动证伪，按证据链收敛根因 | fix-plan |
| `fix/` | `fixer/` + `tester/` ×3 | 最小修复 + 隔离评审 + 并行交叉验证 | fix-report |

（流程收尾的复盘 Agent 不属于 Team，见 `standalone-agents/recapper/`。）

## 通用编排约定

- **视角固化**：Worker 的判据/假设/验证 lens 在任务分配阶段强制指定，独立性来自视角与（可选的）模型差异，不依赖 LLM 输出随机性。
- **信息隔离**：Worker 之间互不通信、互不可见结论；T3 评审输入被裁剪（只见缺陷描述 + 修复目标 + diff，不见修复推理）。
- **证据门禁**：结论必须附可核对证据；支持证据 / 反驳证据 / 复现步骤缺一即淘汰。聚合按证据质量收敛，禁止多数投票。
- **交接契约**：Team 间以 `sop-schema/` 的 JSON Schema 交接，真源 JSON + 自动渲染 `.md` 视图；Schema 校验是流程门禁，关键字段缺失不得进入下一阶段。

## 回退语义

- T3 修复无效 → **回退 T1 重新审查**，不原地重复修复，防止在错误根因上空转；
- T2 根因未收敛 → 补证或回退 T1；
- 任一阶段信息不完备 → `inconclusive` 挂起，不降低标准放行。
