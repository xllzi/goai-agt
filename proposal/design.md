# 设计：面向软件测试场景的 Agent Team

## 1. 多 Agent 系统的优势

- 多 Agent **并行执行**带来的高效率
- 多 Agent **独立推理**无锚定效应 / 去偏见

适用任务：可分解、上下文独立、无前后依赖的任务；或答案多解、错误代价高的决策任务。

## 2. 角色设计（5 个角色类型）

| 角色 | 职责 | 出现位置 |
|---|---|---|
| 聚合 Agent | 缺陷聚合 + 去偏见审查 | T1/T2 的 Leader（聚合多个 worker 的结论） |
| 根因定位 Agent | 多假设并行调查 | T2 的调查 worker |
| 修复 Agent | 方案生成 + 独立评审 | T3 的 Leader |
| 测试验证 Agent | test-based 交叉校验 | T3 的测试 worker |
| 复盘 Agent | 知识沉淀 | 流程收尾（A13） |

## 3. 协同关系设计

```mermaid
flowchart LR
 subgraph s1["输入"]
        Issue["Issue"]
        UserFeedback["用户反馈"]
        Log["日志"]
        Codebase["codebase"]
  end
 subgraph SOP["中间产物/输出"]
        Note["沉淀笔记"]
        Report["修复报告"]
        Review["审查报告"]
        Investigation["调查报告和修复方案"]
  end
 subgraph Team1["审查Team: 多方面并行审查代码库"]
        ReviewWorker1["审查Worker1"]
        ReviewWorker2["审查Worker2"]
        ReviewWorker3["审查Worker3"]
        ReviewLeader["聚合Leader"]
  end
 subgraph Team2["调查Team: 多假设并行调查，定位根因"]
        InvestigateWorker1["调查Worker1"]
        InvestigateWorker2["调查Worker2"]
        InvestigateWorker3["调查Worker3"]
        InvestigateLeader["聚合Leader"]
  end
 subgraph Team3["修复Team: 并行测试提高测试覆盖和反馈速度"]
        FixLeader["修复Leader"]
        TestWorker1["测试Worker1"]
        TestWorker2["测试Worker2"]
        TestWorker3["测试Worker3"]
  end

    ReviewWorker1 --> ReviewLeader
    ReviewWorker2 --> ReviewLeader
    ReviewWorker3 --> ReviewLeader
    ReviewLeader --> Review
    Review --> Team2
    InvestigateWorker1 --> InvestigateLeader
    InvestigateWorker2 --> InvestigateLeader
    InvestigateWorker3 --> InvestigateLeader
    InvestigateLeader --> Investigation
    Investigation --> Team3
    TestWorker1 --> FixLeader
    TestWorker2 --> FixLeader
    TestWorker3 --> FixLeader
    FixLeader --> Report
    Report --> Decision
    Decision -- 是 --> Manager["Manager"]
    Manager --> Team1
    Decision["所有修复方式无效"] -- 否 --> ReviewWorker["复盘Worker(Standalone)"]
    ReviewWorker --> Note
```

