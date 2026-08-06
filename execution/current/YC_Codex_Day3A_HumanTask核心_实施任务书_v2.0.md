---
document_id: yc-codex-day3a-human-task-core
title: YC Codex Day 3A｜HumanTask核心与人工恢复实施任务书
version: 2.0
status: EXECUTION_BASELINE
updated_at: 2026-08-06
depends_on:
  - day2-reliability
supersedes:
  - YC_Codex_Day3A_HumanTask核心_实施任务书_v1.0.md
incorporates_review:
  - YC_Codex_Day3A实施计划_审核意见_v1.0.md
next_stage:
  - day3b-login-notification
---

# YC Codex Day 3A｜HumanTask核心与人工恢复实施任务书 v2.0

## 0. 唯一目标

把当前：

```text
WAITING_HUMAN + MANUAL_REQUIRED Exception
```

升级为：

```text
HumanTask
→ HumanTaskSubmission
→ Worker应用人工结论
→ 必要时创建ResumeSignal
→ 原流程继续、重试或保持暂停
```

Day 3A只实现人工任务核心，不做二维码、OTP、Notification、SecureEphemeralArtifact、真实Connector、Browser Service、Agent Runtime或正式前端。

---

## 1. 开工前检查

必须满足：

- 基于`74f2192 / day2-reliability`；
- Git工作区干净；
- Alembic为`0003_day2_1_semantic_cleanup`；
- 全量16项测试通过；
- 不修改0001—0003；
- 新增迁移`0004_day3a_human_task_core`；
- 先核对现有RBAC模型，不创建第二权限事实源。

---

## 2. 权限基线

### 2.1 禁止单角色Membership

不得新增：

```text
tenant_memberships.role
```

如果当前已经存在Role、Permission和MembershipRole，直接复用。

如果多角色关系尚未落地，建立或补齐：

```text
roles
permissions
membership_roles
role_permissions
```

最低权限键：

```text
human_task.read
human_task.submit
human_task.reassign
human_task.audit
```

角色建议：

```text
TENANT_OWNER / TENANT_ADMIN
→ read / submit / reassign / audit

KOC_MANAGER
→ read / submit KOC任务

KOC_OPERATOR
→ read / submit分派给自己的任务

AUDITOR
→ read / audit

EDGE_DEVICE
→ 无人类任务权限
```

具体角色名应与当前仓库已有定义一致，不得另建同义角色。

---

## 3. 数据模型

### 3.1 HumanTask：稳定工作项

```text
human_tasks
- id
- tenant_id
- run_id
- step_run_id
- exception_id
- task_type
- status
- assignee_membership_id NULL
- assignee_role_key NULL
- dedupe_key
- title
- instruction
- input_schema
- due_at
- expires_at
- version
- created_at
- resolved_at
```

首批任务类型：

```text
EXTERNAL_RESULT_CONFIRMATION
MANUAL_EXCEPTION_RESOLUTION
```

状态：

```text
WAITING_USER
SUBMITTED
VERIFYING
RESOLVED
EXPIRED
FAILED
```

Day 3A不开放会改变业务结论的`cancel`操作。

Assignee约束：

```text
assignee_membership_id
与
assignee_role_key
必须且只能有一个非空
```

`assignee_membership_id`必须属于同一Tenant的有效Membership。

### 3.2 HumanTaskSubmission：人的每次回答

```text
human_task_submissions
- id
- tenant_id
- human_task_id
- submitted_by_membership_id
- decision
- payload
- submission_hash
- status
- followup_job_id
- resume_signal_id NULL
- created_at
- applied_at
```

Submission状态：

```text
SUBMITTED
APPLIED
REJECTED
```

决策：

```text
EXTERNAL_ACTION_COMPLETED
EXTERNAL_ACTION_NOT_EXECUTED
KEEP_PAUSED
```

一个HumanTask可以有多条Submission，适配：

- KEEP_PAUSED后再次判断；
- 未来登录验证失败后的再次提交；
- 完整保留人工操作历史。

### 3.3 ResumeSignal

```text
resume_signals
- id
- tenant_id
- run_id
- step_run_id
- human_task_id
- submission_id
- signal_type
- payload
- status
- created_at
- consumed_at
```

状态：

```text
PENDING
CONSUMED
CANCELLED
```

只有流程真正继续或重试时才创建：

```text
EXTERNAL_ACTION_COMPLETED
EXTERNAL_ACTION_NOT_EXECUTED
```

`KEEP_PAUSED`不创建ResumeSignal。

### 3.4 Active任务去重

使用：

```text
dedupe_key =
tenant_id
+ exception_id
+ task_type
+ responsibility_scope
```

部分唯一索引：

```text
UNIQUE(dedupe_key)
WHERE status IN ('WAITING_USER', 'SUBMITTED', 'VERIFYING')
```

不要永久限制一个Exception只能存在一个任何类型的HumanTask。

---

## 4. 0004迁移

### 4.1 新增

- RBAC缺失的最小关系表或Seed；
- `human_tasks`；
- `human_task_submissions`；
- `resume_signals`；
- 值域、互斥和部分唯一索引；
- 同租户关联所需约束。

### 4.2 旧数据回填

对已有：

```text
Run/Step = WAITING_HUMAN
Exception = OPEN + MANUAL_REQUIRED
```

的数据：

- 创建或复用`EXTERNAL_RESULT_CONFIRMATION` HumanTask；
- 默认`assignee_role_key`使用当前系统实际存在且拥有提交权限的KOC管理角色；
- 生成稳定`dedupe_key`；
- 不创建Submission和ResumeSignal。

### 4.3 Downgrade

按依赖顺序删除：

```text
resume_signals
→ human_task_submissions
→ human_tasks
→ 本迁移新增的RBAC关系/Seed
```

不得破坏0001—0003已有身份数据。

---

## 5. Payload契约

### EXTERNAL_ACTION_COMPLETED

只允许且必须提供：

```json
{
  "external_reference": "1—200字符"
}
```

### EXTERNAL_ACTION_NOT_EXECUTED

只允许且必须提供：

```json
{
  "reason": "1—500字符"
}
```

### KEEP_PAUSED

只允许且必须提供：

```json
{
  "reason": "1—500字符"
}
```

拒绝未知字段。

错误语义：

- 未知枚举、字段类型或结构错误：422；
- 合法字段但与Task/状态不匹配：409；
- 版本冲突：409 `HUMAN_TASK_VERSION_CONFLICT`；
- 不同提交冲突：409 `HUMAN_TASK_SUBMISSION_CONFLICT`。

---

## 6. HumanTask创建

所有进入`WAITING_HUMAN`的Reconcile分支调用统一服务：

```text
create_or_reuse_human_task(...)
```

它必须：

- 同一事务创建或复用Exception和HumanTask；
- 使用`dedupe_key`防重复；
- 写Activity和Audit；
- 不创建Submission；
- 不创建ResumeSignal；
- 不修改原Attempt历史。

不允许不同Handler各自复制HumanTask创建逻辑。

---

## 7. API

```text
GET  /api/v1/human-tasks
GET  /api/v1/human-tasks/{task_id}
POST /api/v1/human-tasks/{task_id}/submit
POST /api/v1/human-tasks/{task_id}/reassign
```

Day 3A不实现会隐式终止业务的：

```text
POST /human-tasks/{id}/cancel
```

### 7.1 列表筛选

```text
status
task_type
assignee_membership_id
assignee_role_key
run_id
due_before
```

### 7.2 提交请求

```json
{
  "decision": "EXTERNAL_ACTION_COMPLETED",
  "payload": {
    "external_reference": "manual-confirmed"
  },
  "expected_version": 1
}
```

### 7.3 提交响应

```text
task_id
submission_id
task_status
task_version
duplicate
followup_job_id
```

提交时ResumeSignal尚未由Worker创建，因此响应不强制包含`resume_signal_id`。

---

## 8. 提交、幂等与并发

### 8.1 提交哈希

```text
task_id
+ decision
+ canonical(payload)
```

### 8.2 HTTP事务

`submit`只负责：

1. Tenant和权限校验；
2. 锁定HumanTask；
3. 检查Task状态和expected_version；
4. 检查已有Submission；
5. 创建`HumanTaskSubmission(SUBMITTED)`；
6. 将Task置为`SUBMITTED`；
7. 创建唯一`APPLY_HUMAN_DECISION` Job；
8. 写Activity/Audit；
9. 提交并返回。

不得在HTTP请求中推进Run、Step、Command或Track。

### 8.3 幂等

- 相同Hash的已存在Submission：返回原Submission和Job，`duplicate=true`；
- Task存在尚未处理的不同Submission：409；
- Task已RESOLVED：409；
- KEEP_PAUSED已被Worker应用后，Task回到WAITING_USER，可再次提交新的最终判断。

### 8.4 并发

HumanTask使用`version`乐观锁。

真实并发测试必须证明：

- 两个不同提交只能有一个成功；
- 不产生两个有效Job；
- 不产生两个ResumeSignal；
- 不重复推进Run。

---

## 9. Worker处理

### 9.1 Job

新增：

```text
APPLY_HUMAN_DECISION
```

Job引用：

```text
human_task_id
submission_id
```

### 9.2 固定锁顺序

统一使用：

```text
Run
→ StepRun
→ Command
→ Exception
→ HumanTask
→ HumanTaskSubmission
→ ResumeSignal
→ CaseTrack
```

事务中禁止调用任何外部系统。

### 9.3 EXTERNAL_ACTION_COMPLETED

```text
Submission → APPLIED
HumanTask → VERIFYING → RESOLVED
原Write Attempt保持RESULT_UNKNOWN
原Write Command → SUCCEEDED
Exception → RESOLVED
创建并消费ResumeSignal
Run / Step / Track按成功路径完成
创建MANUAL_CONFIRMATION Evidence或Decision记录
```

人工Evidence必须标记：

```text
evidence_source = MANUAL_CONFIRMATION
```

不得伪装成Browser或平台证据。

### 9.4 EXTERNAL_ACTION_NOT_EXECUTED

```text
Submission → APPLIED
HumanTask → RESOLVED
原Write Command → PENDING
Exception → RESOLVED
创建并消费ResumeSignal
Run → RUNNING
Step → WAITING_EXTERNAL
下一次Edge领取生成更高Generation
旧Attempt历史不变
```

### 9.5 KEEP_PAUSED

```text
Submission → APPLIED
HumanTask → WAITING_USER
Exception保持OPEN
Run / Step保持WAITING_HUMAN
不创建ResumeSignal
不创建新Generation
不创建后继HumanTask
可更新due_at和version
```

---

## 10. Reassign

只允许具备`human_task.reassign`权限的成员。

### 分派给成员

- 目标必须是同租户有效Membership；
- AUDITOR或没有提交权限的成员不能成为可处理Assignee。

### 分派给角色

- 目标角色必须存在；
- 该角色必须拥有`human_task.submit`；
- `assignee_membership_id`与`assignee_role_key`保持互斥。

Reassign使用`expected_version`，写Activity与Audit，不改变Run状态。

---

## 11. Activity、Audit与历史

Activity至少包括：

```text
HUMAN_TASK_CREATED
HUMAN_TASK_SUBMITTED
HUMAN_TASK_REASSIGNED
HUMAN_TASK_RESOLVED
HUMAN_DECISION_APPLIED
RUN_RESUMED_BY_HUMAN
HUMAN_TASK_KEPT_PAUSED
```

Audit至少包括：

```text
SUBMIT_HUMAN_TASK
REASSIGN_HUMAN_TASK
APPLY_HUMAN_DECISION
```

Audit和普通日志只记录：

- 身份；
- decision；
- 对象ID；
- payload hash；
- 状态摘要。

自由文本原文只保存在受权限保护的Submission payload中，不复制到普通日志。

---

## 12. 测试

必须覆盖：

1. UNKNOWN自动创建一个有效HumanTask；
2. 重复处理UNKNOWN不重复创建任务；
3. 0003已有WAITING_HUMAN数据升级后补建任务；
4. COMPLETED路径完成Run；
5. 原Write Attempt仍为RESULT_UNKNOWN；
6. NOT_EXECUTED路径创建下一Generation；
7. KEEP_PAUSED不推进业务；
8. KEEP_PAUSED不创建ResumeSignal；
9. KEEP_PAUSED不创建后继任务；
10. KEEP_PAUSED后可再次提交最终决定；
11. 一个Task可保存多条Submission；
12. 相同提交幂等；
13. 不同待处理提交409；
14. expected_version冲突409；
15. 两个真实并发提交仅一个成功；
16. Worker重复处理幂等；
17. ResumeSignal最多一个且只消费一次；
18. 跨租户隔离；
19. Edge身份拒绝；
20. AUDITOR只读；
21. USER Assignee必须是同租户Membership；
22. ROLE/USER Assignee互斥；
23. 非法Reassign目标拒绝；
24. 人工Evidence标记为MANUAL；
25. Payload 422/409矩阵；
26. API/Worker重启后任务仍可恢复；
27. Day 1和Day 2全部回归；
28. 空库升级至0004；
29. 0003旧数据升级；
30. downgrade/upgrade往返；
31. `alembic check`无漂移。

---

## 13. Demo

创建：

```text
scripts/demo_day3a_human_task.ps1
```

展示：

```text
A. UNKNOWN → 人确认已完成 → Run完成
B. UNKNOWN → 人确认未执行 → 新Generation成功
C. UNKNOWN → KEEP_PAUSED → 同一任务继续开放
D. KEEP_PAUSED后再次提交最终结论
```

输出：

- HumanTask；
- Submission历史；
- Exception；
- ResumeSignal；
- Run/Step/Command；
- 原Attempt历史；
- Activity；
- 提交幂等；
- Generation变化。

---

## 14. Release

通过全部门禁后：

```text
commit:
feat: complete YC Day 3A human task core

annotated tag:
day3a-human-task-core
```

Release目录：

```text
yc/docs/releases/day3a-human-task-core/
├─ README.md
├─ openapi.json
├─ pytest-output.txt
├─ alembic-check.txt
├─ demo-transcript.txt
├─ migration-summary.md
└─ KNOWN_GAPS.md
```

完成后确认工作区干净并停止，不进入Day 3B。

---

## 15. 禁止事项

- 不做二维码；
- 不做OTP；
- 不做NotificationDelivery；
- 不做SecureEphemeralArtifact；
- 不接飞书、微信、短信；
- 不接Browser Service；
- 不接真实Connector；
- 不做Agent Runtime；
- 不做正式前端；
- 不大规模拆分`services.py`；
- 不新增单一Membership角色列；
- 不用取消HumanTask隐式终止Run。
