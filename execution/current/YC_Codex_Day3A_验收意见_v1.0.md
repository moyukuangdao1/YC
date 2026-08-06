---
document_id: yc-codex-day3a-acceptance
title: YC Codex Day 3A HumanTask核心｜验收意见
version: 1.0
status: ACCEPTED_PENDING_MANUAL_SMOKE_AND_CANCEL_INVARIANT
updated_at: 2026-08-06
release:
  commit: e55c999
  tag: day3a-human-task-core
  alembic_revision: 0004_day3a_human_task_core
  full_test_result: 27 passed
  demo_result: 3 passed
---

# YC Codex Day 3A HumanTask核心｜验收意见 v1.0

## 1. 报告级结论

根据Codex提交报告，Day 3A的主要目标已经完成：

- 多角色RBAC与稳定Membership UUID；
- HumanTask、HumanTaskSubmission、ResumeSignal；
- 已有WAITING_HUMAN数据自动补建任务；
- Assignee互斥、同租户约束和活动任务去重；
- `EXTERNAL_ACTION_COMPLETED`、`EXTERNAL_ACTION_NOT_EXECUTED`、`KEEP_PAUSED`；
- HTTP提交与Worker应用人工结论分离；
- 原Write/Reconcile Attempt历史不被修改；
- 人工结论标记为MANUAL；
- 27项测试和3条Demo路径通过；
- Commit、annotated Tag、Release和干净工作区；
- 未越界进入Day 3B。

因此本阶段在报告层面通过。

最终验收仍需：

1. 创始人本地Smoke；
2. 确认取消唯一有效HumanTask不会制造“WAITING_HUMAN但无人可处理”的死状态。

---

## 2. 必须确认的取消不变量

Codex报告说明：

```text
Cancel不改变Exception、Run或业务结论
```

方向正确，但还需确认：

> 如果当前HumanTask是某个WAITING_HUMAN流程唯一的有效任务，系统不能直接将它取消后什么都不做。

必须满足以下至少一种语义：

### 推荐

```text
取消唯一有效HumanTask
→ 409 ACTIVE_HUMAN_TASK_REQUIRED
```

### 或

```text
取消任务与创建替代HumanTask
→ 同一事务完成
```

禁止出现：

```text
Run / Step = WAITING_HUMAN
Exception = OPEN
但没有任何有效HumanTask
```

如果当前代码未覆盖该不变量，应增加一个极小Day 3A.1修订和测试，再最终验收。

---

## 3. 手工Smoke

在项目目录运行：

```powershell
git status --short
git rev-parse --short "day3a-human-task-core^{}"
.\.venv\Scripts\alembic.exe current
.\.venv\Scripts\alembic.exe check
.\.venv\Scripts\python.exe -m pytest -q
powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\demo_day3a_human_task.ps1
```

预期：

- 工作区干净；
- Tag指向`e55c999`；
- Alembic为`0004_day3a_human_task_core (head)`；
- 27项测试通过；
- Demo 3项通过。

人工观察：

### 已完成

- 原Write Attempt仍为`RESULT_UNKNOWN`；
- HumanTask Submission保留人工结论；
- 人工事实标记为MANUAL；
- Run/Step/Track完成；
- Exception解决；
- ResumeSignal单次消费。

### 未执行

- 原Command恢复`PENDING`；
- 下一次生成更高Generation；
- 旧Attempt历史保留。

### KEEP_PAUSED

- 同一个HumanTask继续有效；
- 可以保存多条Submission；
- 不创建ResumeSignal；
- 不创建后继HumanTask；
- 不创建新Generation；
- Run/Step保持`WAITING_HUMAN`。

---

## 4. 当前成熟度

| 能力 | 状态 |
|---|---|
| Fake纵向执行 | VALIDATED_FOR_FAKE |
| RESULT_UNKNOWN/Reconcile | VALIDATED_FOR_FAKE |
| HumanTask人工结果确认 | VALIDATED_FOR_FAKE |
| HumanTask多次Submission | VALIDATED_FOR_FAKE |
| 人工恢复与安全重试 | VALIDATED_FOR_FAKE |
| QR登录与系统二次验证 | NOT_IMPLEMENTED |
| 消息通知 | NOT_IMPLEMENTED |
| 企业差异配置 | NOT_IMPLEMENTED |
| 真实Connector | NOT_IMPLEMENTED |
| Browser Service正式集成 | NOT_IMPLEMENTED |
| 真实业务价值 | NOT_VALIDATED |

---

## 5. 下一阶段授权

Smoke与取消不变量通过后，允许进入：

> Day 3B：认证二维码、临时安全材料、Fake/In-App通知和系统二次验证。

禁止直接进入真实飞书通知、Browser Service、云仓或抖音写入。
