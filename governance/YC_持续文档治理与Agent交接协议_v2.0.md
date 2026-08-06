---
document_id: yc-continuous-documentation-and-handoff-protocol
title: YC持续文档治理与Agent交接协议
version: 2.0
status: ACCEPTED
updated_at: 2026-08-06
owner: YC Founder
supersedes:
  - YC_持续文档治理与Agent交接协议_v1.0.md
  - YC_文档持续沉淀与上下文治理规则_v1.0.md
purpose: 统一YC长期文档治理、里程碑记录、Agent交接和知识沉淀规则
---

# YC持续文档治理与Agent交接协议 v2.0

## 1. 核心原则

> 聊天是讨论层，正式文档是事实层，Git Release是实现事实层，真实业务结果是价值事实层。

任何结论必须明确属于以下哪一层：

1. **讨论候选**：尚未确认；
2. **正式决策**：已写入战略、产品、ADR或行动书；
3. **工程实现**：有Commit、Tag、Migration、测试和Demo；
4. **真实业务验证**：在真实企业连续运行；
5. **商业验证**：有付费、续费或明确采购承诺。

不得把“讨论过”“代码写完”“Fake测试通过”“真实业务有效”“客户愿意付费”混为一谈。

---

## 2. 文档状态

所有重要文件或结论使用以下状态：

| 状态 | 含义 |
|---|---|
| `IDEA` | 仅供畅想 |
| `HYPOTHESIS` | 有依据、待验证 |
| `PROPOSED` | 已形成方案、待确认 |
| `ACCEPTED` | 已确认并成为当前决策 |
| `IN_IMPLEMENTATION` | 正在实现 |
| `EXECUTION_BASELINE` | 当前Codex执行依据 |
| `VALIDATED_FOR_FAKE` | 已通过Fake/自动化测试 |
| `VALIDATED_FOR_PILOT` | 已通过真实试点 |
| `VALIDATED_FOR_COMMERCIAL` | 已出现付费和可复制证据 |
| `OPEN_QUESTION` | 信息不足，禁止Agent猜测 |
| `SUPERSEDED` | 已被新文件替代 |
| `REJECTED` | 明确不采用 |

---

## 3. 文档分层

### 3.1 文档中心

`docs-gpt/README.md`

作用：

- 当前状态；
- 权威文件；
- 阅读顺序；
- 当前唯一主任务；
- 已归档文件；
- 最近Release；
- 开放问题。

### 3.2 战略

`docs-gpt/strategy/`

只记录相对稳定内容：

- 公司定位；
- 商业模式；
- 现金流；
- 客户、团队、融资和IP；
- 长期护城河和阶段路线。

### 3.3 产品

`docs-gpt/product/`

记录：

- 产品边界；
- 用户、对象、流程和页面；
- P0/P1/P2；
- 试点与采购表达；
- 验收标准。

### 3.4 架构与ADR

`docs-gpt/architecture/`
`docs-gpt/adr/`

记录：

- YC Core / Solution / Tenant Deployment；
- Program / Agent / Human / Edge；
- 多租户、权限、安全和数据边界；
- Cell Runtime；
- Browser Service；
- HumanTask、Reconcile、配置和版本；
- 关键技术决策及取舍。

### 3.5 执行

`docs-gpt/execution/current/`
`docs-gpt/execution/history/`

`current`最多保留：

- 当前唯一行动书；
- 下一阶段已批准行动书；
- 当前验收意见。

历史Day/Week任务移入`history`，不再作为执行依据。

### 3.6 销售

`docs-gpt/sales/`

包括：

- 产品一页纸；
- Demo脚本；
- PPT；
- 采购话术；
- 试点方案。

### 3.7 上下文与开放问题

`docs-gpt/context/`
`docs-gpt/open-questions/`

包括：

- 当前状态快照；
- 新Agent接管上下文；
- OPEN_QUESTIONS；
- 决策门。

### 3.8 工程事实

工程仓库`yc/docs/`保存：

- API和开发说明；
- ADR的工程副本；
- Release；
- 测试和Demo产物；
- Runbook；
- Migration说明。

不要在`docs-gpt`和`yc/docs`重复保存同一份Release产物。`docs-gpt`只链接它。

---

## 4. 冲突优先级

发生冲突时：

1. 最新已接受ADR；
2. 最新Release及真实实现；
3. 当前`EXECUTION_BASELINE`行动书；
4. 当前产品技术方案；
5. 当前战略规划；
6. 当前状态快照与接管上下文；
7. 旧文件和历史聊天。

发现文档与实现冲突时，不得静默选择；必须记录差异并更新权威文件。

---

## 5. 每轮沟通后的动作

战略顾问必须判断是否需要：

- 更新战略；
- 新增ADR；
- 更新产品技术方案；
- 更新当前行动书；
- 新增验收记录；
- 更新OPEN_QUESTIONS；
- 更新当前状态快照；
- 更新新Agent接管上下文；
- 形成Decision Ledger / Wiki / Skill / Eval候选；
- 或明确“本轮只解释，无需更新”。

回复末尾固定输出：

```text
本轮文档动作
- 更新/新增：
- 新确认决策：
- 仍属假设：
- 新开放问题：
- 对Codex影响：
- 下一步唯一主任务：
```

---

## 6. 里程碑治理

每个Day/Week或真实Cell结束，必须记录：

- Git Commit；
- Git Tag；
- Alembic Revision；
- 测试结果；
- Demo记录；
- 当前边界；
- KNOWN_GAPS；
- 验收结论；
- 是否允许进入下一阶段。

重大里程碑还必须更新：

1. 文档中心；
2. 当前状态快照；
3. Agent接管上下文；
4. 下一阶段行动书。

---

## 7. Agent接管

新Agent必须按顺序阅读：

1. 文档中心README；
2. 当前状态快照；
3. 当前战略规划；
4. 当前产品技术方案；
5. 有效ADR/架构增量；
6. 当前行动书；
7. OPEN_QUESTIONS；
8. 新Agent接管上下文。

新Agent不能从旧聊天自行恢复事实，也不能根据模糊描述补充业务规则。

---

## 8. 知识沉淀

真实业务知识按以下路径处理：

```text
事件
→ Decision Ledger候选
→ 结果回流
→ Wiki候选
→ Skill候选
→ Eval
→ 人工确认
→ 发布
```

人类的一次临时选择、一次偶然成功或未经验证的专家观点，不直接成为生产规则。

---

## 9. 文档更新纪律

- 同一主题小调整：更新原文件；
- 新一级架构原则：ADR或架构附录；
- 新阶段实施：新行动书；
- 里程碑完成：Release/验收；
- 未决问题：OPEN_QUESTIONS；
- 真实决策与结果：Decision Ledger候选；
- 旧文件标记`SUPERSEDED`并移入历史目录；
- 不允许多个文件同时自称“当前唯一执行基线”。
