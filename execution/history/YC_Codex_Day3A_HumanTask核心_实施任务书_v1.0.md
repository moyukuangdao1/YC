---
document_id: yc-codex-day3a-human-task-core
title: YC Codex Day 3A｜HumanTask核心与人工恢复实施任务书
version: 1.0
status: EXECUTION_BASELINE
updated_at: 2026-08-06
depends_on:
  - day2-reliability
next_stage:
  - day3b-login-notification
---

# YC Codex Day 3A｜HumanTask核心与人工恢复实施任务书 v1.0

## 0. 唯一目标

把当前：

```text
WAITING_HUMAN + MANUAL_REQUIRED Exception
```

升级为正式的：

```text
HumanTask
→ 人提交结构化结论
→ Runtime创建后续Job
→ 校验和应用结论
→ ResumeSignal
→ 原流程继续、重试或保持暂停
```

Day 3A不做二维码、OTP、通知和安全临时Artifact。

---

## 1. 开工前检查

必须满足：

- 当前Commit基于`day2-reliability`；
- Git工作区干净；
- Alembic当前为`0003_day2_1_semantic_cleanup`；
- 全量16项测试通过；
- 不修改已冻结的0001—0003迁移；
- 新增迁移编号从0004开始。

---

## 2. 核心模型

### 2.1 HumanTask

建议字段：

```text
id
tenant_id
run_id
step_run_id
exception_id
task_type
status
assignee_type
assignee_id
title
instruction
input_schema
submitted_payload
submission_hash
due_at
expires_at
created_at
submitted_at
resolved_at
version
```

首批任务类型：

```text
EXTERNAL_RESULT_CONFIRMATION
MANUAL_EXCEPTION_RESOLUTION
```

状态：

```text
OPEN
WAITING_USER
SUBMITTED
VERIFYING
RESOLVED
CANCELLED
EXPIRED
FAILED
```

### 2.2 ResumeSignal

字段：

```text
id
tenant_id
run_id
step_run_id
human_task_id
signal_type
payload
status
created_at
consumed_at
```

状态：

```text
PENDING
CONSUMED
CANCELLED
```

---

## 3. 业务路径

### 3.1 Reconcile仍UNKNOWN

```text
Reconcile = SUCCEEDED + UNKNOWN
→ 原Write Command保持RESULT_UNKNOWN
→ Run/Step进入WAITING_HUMAN
→ 创建或复用MANUAL_REQUIRED Exception
→ 创建或复用HumanTask(EXTERNAL_RESULT_CONFIRMATION)
```

HumanTask允许三个选择：

```text
EXTERNAL_ACTION_COMPLETED
EXTERNAL_ACTION_NOT_EXECUTED
KEEP_PAUSED
```

### 3.2 人确认“外部动作已完成”

```text
提交HumanTask
→ HumanTask = SUBMITTED
→ 创建APPLY_HUMAN_DECISION Job
→ Worker验证权限、任务和状态
→ HumanTask = VERIFYING
→ 原Command = SUCCEEDED
→ Step/Run/Track按成功路径推进
→ Exception = RESOLVED
→ ResumeSignal = CONSUMED
→ HumanTask = RESOLVED
```

注意：

> 人工确认属于有责任主体的业务结论，不得篡改原Write Attempt的`RESULT_UNKNOWN`历史。

### 3.3 人确认“外部动作未执行”

```text
提交HumanTask
→ 后续Job验证
→ 原Write Command = PENDING
→ Run返回RUNNING
→ 下一次领取产生更高Generation Attempt
→ Exception解决
→ HumanTask解决
```

### 3.4 保持暂停

```text
KEEP_PAUSED
→ HumanTask继续WAITING_USER或新增确认记录
→ Run/Step保持WAITING_HUMAN
→ 不创建新的写Attempt
```

实现时选择一种明确语义并写入测试；不得把`KEEP_PAUSED`标记为RESOLVED后又没有开放任务。

---

## 4. 关键不变量

### 4.1 WAITING_HUMAN必须有有效任务

当Run或Step为`WAITING_HUMAN`时，必须至少存在一个：

```text
OPEN / WAITING_USER / SUBMITTED / VERIFYING
```

的HumanTask。

### 4.2 任务解决事务

以下内容在同一事务中完成：

- 更新HumanTask；
- 更新Exception；
- 创建/消费ResumeSignal；
- 创建后续Job或推进业务状态；
- 写ActivityEvent；
- 写AuditEvent。

### 4.3 人类提交不在HTTP中直接完成业务

`POST /human-tasks/{id}/submit`只做：

- 权限与租户校验；
- 语义校验；
- 持久化提交；
- 创建后续Job；
- 返回。

业务状态由Worker处理。

### 4.4 原历史不修改

- 原Write Attempt保持`RESULT_UNKNOWN`；
- Reconcile Attempt历史不改；
- 人工决策作为新的Activity、Audit和HumanTask事实保存。

---

## 5. 权限

最低规则：

| 角色 | 权限 |
|---|---|
| 任务Assignee | 可读取并提交自己的任务 |
| KOC_MANAGER | 可处理本租户KOC HumanTask |
| TENANT_ADMIN/OWNER | 可重新分派、取消 |
| AUDITOR | 只读 |
| EDGE_DEVICE | 禁止调用人类任务API |
| 其他租户 | 404或403 |

Day 3A不做复杂部门和行级ABAC。

---

## 6. 幂等与并发

### 6.1 提交幂等

提交哈希包含：

```text
task_id
decision
payload
```

相同提交：

```text
返回原结果，duplicate=true
```

不同提交：

```text
409 HUMAN_TASK_SUBMISSION_CONFLICT
```

### 6.2 乐观锁

HumanTask使用`version`：

```text
UPDATE ... WHERE id=? AND version=?
```

防止两个管理者同时作出不同结论。

### 6.3 ResumeSignal

一个Signal只能消费一次。

重复Worker处理必须幂等，不得重复推进Run或创建新Generation。

---

## 7. API

```text
GET  /api/v1/human-tasks
GET  /api/v1/human-tasks/{task_id}
POST /api/v1/human-tasks/{task_id}/submit
POST /api/v1/human-tasks/{task_id}/cancel
POST /api/v1/human-tasks/{task_id}/reassign
```

列表筛选：

```text
status
task_type
assignee_id
run_id
due_before
```

提交示例：

```json
{
  "decision": "EXTERNAL_ACTION_COMPLETED",
  "payload": {
    "external_reference": "manual-confirmed"
  },
  "expected_version": 1
}
```

---

## 8. Activity与Audit

Activity至少包括：

```text
HUMAN_TASK_CREATED
HUMAN_TASK_SUBMITTED
HUMAN_TASK_RESOLVED
HUMAN_DECISION_APPLIED
RUN_RESUMED_BY_HUMAN
```

Audit至少包括：

```text
SUBMIT_HUMAN_TASK
CANCEL_HUMAN_TASK
REASSIGN_HUMAN_TASK
APPLY_HUMAN_DECISION
```

不得把敏感自由文本无过滤写入普通日志。

---

## 9. 测试

必须覆盖：

1. UNKNOWN自动创建一个HumanTask；
2. 重复处理同一UNKNOWN不重复创建任务；
3. 人确认已完成，Run完成；
4. 原Write Attempt仍为RESULT_UNKNOWN；
5. 人确认未执行，创建下一Generation；
6. KEEP_PAUSED不推进；
7. 相同提交幂等；
8. 不同提交409；
9. 两个并发提交只成功一个；
10. ResumeSignal只能消费一次；
11. Worker重复处理不重复推进；
12. 跨租户隔离；
13. Edge身份被拒绝；
14. AUDITOR只读；
15. API/Worker重启后任务仍存在；
16. Day 1和Day 2全部回归通过；
17. Alembic空库升级至0004；
18. `alembic check`无漂移。

---

## 10. Demo

创建：

```text
scripts/demo_day3a_human_task.ps1
```

展示三条路径：

```text
A. UNKNOWN → 人确认已完成 → Run完成
B. UNKNOWN → 人确认未执行 → 新Generation成功
C. UNKNOWN → KEEP_PAUSED → 继续等待
```

输出：

- HumanTask状态；
- Exception状态；
- ResumeSignal；
- Run/Step/Command；
- 原Attempt历史；
- Activity时间线；
- 提交幂等。

---

## 11. Definition of Done

- HumanTask和ResumeSignal持久化；
- WAITING_HUMAN有正式任务承接；
- HTTP提交不直接推进业务；
- Worker可应用人工结论；
- 人工决策有责任与审计；
- 原Attempt历史不被篡改；
- 权限、租户、幂等和并发测试通过；
- 全部回归通过；
- Demo可重复；
- 完成后停止并等待验收。

---

## 12. 禁止事项

- 不做二维码；
- 不做OTP；
- 不做NotificationDelivery；
- 不做SecureEphemeralArtifact；
- 不接飞书、微信、短信；
- 不接Browser Service；
- 不接真实Connector；
- 不拆分全部services.py；
- 不做Agent Runtime；
- 不做正式前端。
