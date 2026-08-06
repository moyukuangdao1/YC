---
document_id: yc-codex-day3a-plan-review
title: YC Codex Day 3A HumanTask实施计划｜审核意见
version: 1.0
status: APPROVED_WITH_REQUIRED_CHANGES
updated_at: 2026-08-06
reviewer_role:
  - 战略总顾问
  - CEO导师
  - 技术总指导
applies_to:
  - YC Codex Day 3A HumanTask核心与人工恢复实施计划
---

# YC Codex Day 3A HumanTask实施计划｜审核意见 v1.0

## 1. 结论

方案整体方向正确，约80%可以直接保留：

- 严格停留在Day 3A；
- HumanTask与ResumeSignal持久化；
- HTTP只保存人工提交并创建后续Job；
- Worker异步应用人工结论；
- 原Write Attempt的`RESULT_UNKNOWN`历史不被篡改；
- 多租户、权限、幂等、乐观锁和并发测试；
- Release、Tag和停止边界。

但不能按原方案直接实施。以下四项属于必须修改的架构问题：

1. 不得在`tenant_memberships`上新增单一非空`role`作为新的权限事实源；
2. HumanTask必须与HumanTaskSubmission/Attempt分离；
3. Assignee不能使用一个混合语义的`assignee_id`；
4. `KEEP_PAUSED`与`cancel`语义必须修正。

修正后可以执行。

---

# 2. 必须修正一：不得退化为单角色Membership

原计划：

```text
tenant_memberships新增非空role：
OWNER / TENANT_ADMIN / KOC_MANAGER / AUDITOR
```

该设计与YC已确认的多角色、未来细粒度权限方向冲突，也可能与现有Role/Permission表形成两个事实来源。

## 2.1 正确原则

先核对当前真实模型：

### 如果已有Role/Permission/MembershipRole

直接复用，不新增`tenant_memberships.role`。

### 如果当前只有TenantMembership，角色表尚未真正使用

也应增加：

```text
roles
permissions
membership_roles
role_permissions
```

或先实现最小的`membership_roles`，而不是单一`role`列。

## 2.2 原因

单角色列会造成：

- 一个用户不能同时拥有KOC管理和财务查看权限；
- 未来增加品牌、部门、数据范围时必须再次迁移；
- 角色和权限表成为空壳；
- 角色判断散落在业务代码中；
- 当前遗漏`KOC_OPERATOR / FINANCE_VIEWER / VIEWER`等既有角色。

## 2.3 Day 3A最低权限键

建议使用权限，而不是在服务中大量写角色名判断：

```text
human_task.read
human_task.submit
human_task.reassign
human_task.manage
human_task.audit
```

角色映射：

```text
TENANT_OWNER / TENANT_ADMIN
→ 全部权限

KOC_MANAGER
→ read / submit KOC范围任务

KOC_OPERATOR
→ read / submit分派给自己的任务

AUDITOR
→ read / audit

EDGE_DEVICE
→ 无人类任务权限
```

---

# 3. 必须修正二：HumanTask与Submission分离

原计划把以下内容放在`human_tasks`中：

```text
submitted_payload
submission_hash
submitted_at
```

并通过`predecessor_task_id`处理`KEEP_PAUSED`。

不建议采用。

## 3.1 正确模型

### HumanTask：稳定工作项

```text
human_tasks
- id
- tenant_id
- run_id
- step_run_id
- exception_id
- task_type
- status
- assignee_membership_id
- assignee_role_key
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

### HumanTaskSubmission：人的每一次回答

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
- resume_signal_id
- created_at
- applied_at
```

Submission状态可先使用：

```text
SUBMITTED
APPLIED
REJECTED
DUPLICATE
```

## 3.2 为什么现在就要拆

Day 3B二维码登录必然出现：

```text
人提交“已扫码”
→ 系统验证失败
→ 人再次提交
→ 系统再次验证
```

一个HumanTask可能存在多次人工提交。

如果把提交数据直接放在HumanTask上，只能：

- 覆盖历史；
- 不断创建后继任务；
- 或引入复杂的predecessor链。

这与YC已采用的：

```text
Command
→ CommandAttempt
```

同一原则一致：

```text
HumanTask
→ HumanTaskSubmission
```

## 3.3 不采用predecessor_task_id作为正常暂停机制

前驱任务链可留作未来“任务被替换”的特殊能力，但不应用于每一次`KEEP_PAUSED`。

否则多次暂停会生成大量虚假任务，管理者也很难区分：

- 真正新增的任务；
- 同一任务的一次暂停回答；
- 验证失败后的再次提交。

---

# 4. 必须修正三：Assignee字段必须类型安全、租户安全

原计划：

```text
assignee_type
assignee_id
```

其中：

- ROLE时存`KOC_MANAGER`；
- USER时存用户ID。

这会让一个字段同时保存字符串角色和UUID用户，无法形成可靠外键。

## 4.1 推荐字段

```text
assignee_membership_id UUID NULL
assignee_role_key TEXT NULL
```

数据库CHECK：

```text
两者必须且只能有一个非空
```

为什么使用Membership而不是User：

> User是全局身份，Membership才表示“这个人在这家企业中的身份”。

这样可直接保证：

- 用户属于当前Tenant；
- 被分派人是有效成员；
- 同一User在不同Tenant的身份不会混淆。

## 4.2 ROLE任务

`assignee_role_key=KOC_MANAGER`表示：

> 本租户任何具备对应权限的有效成员都可以领取或提交。

不要把Role字符串放入UUID字段。

---

# 5. 必须修正四：KEEP_PAUSED和Cancel语义

## 5.1 KEEP_PAUSED

原计划：

```text
当前Task→RESOLVED
创建一个后继WAITING_USER任务
消费ResumeSignal
```

不建议。

正确行为：

```text
记录一条HumanTaskSubmission：
decision = KEEP_PAUSED

HumanTask保持WAITING_USER
Run/Step保持WAITING_HUMAN
Exception保持OPEN
不创建ResumeSignal
不创建新写Attempt
不创建后继HumanTask
```

可以：

- 更新due_at；
- 写Activity/Audit；
- 增加Task version；
- 记录原因。

因为业务实际上没有恢复，不能制造“Resume”事实。

## 5.2 Cancel

原计划：

```text
取消Task
→ Exception RESOLVED
→ Run/Step FAILED
→ Track BLOCKED
```

语义不正确。

取消一个人工任务并不等于：

- 外部事实已经确认；
- 原异常已经解决；
- 企业决定本次业务失败。

建议Day 3A：

> **暂不实现`/cancel`业务终止语义。**

保留：

```text
reassign
submit
```

未来如需终止整条业务，应设计独立动作：

```text
ABORT_RUN
```

并明确：

```text
Run / Step = CANCELLED
Exception = CLOSED_AS_ABORTED
Track = BLOCKED
```

不能用“取消HumanTask”隐式代替“终止业务”。

如果Codex坚持保留`/cancel`，则它只能取消重复/错误创建的任务，并要求同一等待状态下仍有另一条有效HumanTask；不得自动解决Exception或失败Run。

---

# 6. ResumeSignal正确语义

ResumeSignal只在流程确实要继续时创建：

## EXTERNAL_ACTION_COMPLETED

```text
HumanTaskSubmission
→ APPLY_HUMAN_DECISION Job
→ Worker确认人工结论
→ ResumeSignal
→ 原Command确认为成功
→ Run继续
```

## EXTERNAL_ACTION_NOT_EXECUTED

```text
HumanTaskSubmission
→ Worker应用
→ ResumeSignal
→ 原Command恢复PENDING
→ 新Generation
```

## KEEP_PAUSED

```text
只记录Submission
不创建ResumeSignal
不消费Signal
```

一个HumanTask最多一个最终有效ResumeSignal是合理的。

---

# 7. Active HumanTask唯一性

原计划：

> 同一Exception最多一个有效任务。

方向正确，但建议使用显式`dedupe_key`：

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
WHERE status IN ('OPEN', 'WAITING_USER', 'SUBMITTED', 'VERIFYING')
```

这比单纯按Exception唯一更有扩展性。

未来同一Exception可能同时需要：

- 管理者确认外部结果；
- 财务确认费用；
- 账号管理员恢复登录。

Day 3A仍然只创建一种任务，但数据结构不应永久锁死“一异常只能有一个任何类型任务”。

---

# 8. Worker事务与锁顺序

Worker应用人工结论时可以锁定相关对象，但必须制定统一锁顺序，避免未来并发死锁。

建议顺序：

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

或由团队选择另一套固定顺序，但所有Handler必须一致。

要求：

- 锁内不调用任何外部系统；
- 锁内只执行短事务；
- Worker崩溃时事务整体回滚；
- Job重试不得重复创建Signal、Task或Generation。

---

# 9. 人工确认的证据语义

`EXTERNAL_ACTION_COMPLETED`当前要求`external_reference`，Fake阶段可以接受。

但必须明确：

```text
人工确认
≠
平台机器证据
```

保存为：

- HumanTaskSubmission；
- Activity；
- Audit；
- 一个`MANUAL_CONFIRMATION`类型Evidence或Decision记录。

不能伪装成Browser或平台返回的机器Evidence。

后续真实试点可要求：

```text
external_reference
或
screenshot/evidence_id
```

---

# 10. 保留原计划的内容

以下内容批准：

- 新增0004，不修改0001—0003；
- 迁移已有WAITING_HUMAN数据；
- HTTP提交不推进业务；
- Worker异步应用人工结论；
- 严格Pydantic payload与422/409；
- 原Write/Reconcile Attempt历史不变；
- `NOT_EXECUTED`生成更高Generation；
- 人工任务多租户隔离；
- Edge身份不能访问人类API；
- 审计不保存自由文本原文；
- 测试真实并发；
- Demo三条路径；
- Release、Commit、Tag；
- 完成后停止，不进入Day 3B。

---

# 11. 修订后的Day 3A核心表

```text
human_tasks
human_task_submissions
resume_signals
```

不新增：

```text
tenant_memberships.role
predecessor_task_id（非P0需要）
```

可选保留：

```text
replacement_task_id
```

但不用于KEEP_PAUSED正常流程。

---

# 12. 修订后的三条路径

## A. 人确认已完成

```text
HumanTaskSubmission(APPLIED)
→ Worker
→ 原Attempt仍RESULT_UNKNOWN
→ 原Command SUCCEEDED
→ Exception RESOLVED
→ ResumeSignal CONSUMED
→ HumanTask RESOLVED
→ Run/Step/Track完成
```

## B. 人确认未执行

```text
HumanTaskSubmission(APPLIED)
→ 原Command PENDING
→ Exception RESOLVED
→ ResumeSignal CONSUMED
→ HumanTask RESOLVED
→ Run RUNNING
→ Step WAITING_EXTERNAL
→ 下一次领取生成更高Generation
```

## C. 保持暂停

```text
HumanTaskSubmission(APPLIED)
→ HumanTask仍WAITING_USER
→ Exception仍OPEN
→ Run/Step仍WAITING_HUMAN
→ 无ResumeSignal
→ 无新Generation
```

---

# 13. 修订后的新增测试

除原计划外必须增加：

1. 一个HumanTask可保存多条Submission；
2. KEEP_PAUSED不创建后继任务；
3. KEEP_PAUSED不创建ResumeSignal；
4. KEEP_PAUSED后仍可再次提交最终决定；
5. 角色与Membership不产生第二权限事实源；
6. USER Assignee使用同租户Membership；
7. ROLE与USER Assignee互斥约束；
8. HumanTask取消不会隐式解决Exception或失败Run；
9. 人工确认Evidence明确标记为MANUAL；
10. 固定锁顺序下的并发提交测试。

---

# 14. 给Codex的直接审批意见

> 方案整体批准，但必须按审核意见修改后执行：
>
> 1. 禁止在`tenant_memberships`新增单一非空`role`。复用既有RBAC；如现有多角色关系未落地，实现`membership_roles`，不要制造第二权限事实源。
> 2. 将HumanTask与HumanTaskSubmission分离。HumanTask是稳定工作项，人的每次提交单独持久化；不要用`predecessor_task_id`承载正常KEEP_PAUSED和重复提交。
> 3. Assignee改为`assignee_membership_id`或`assignee_role_key`二选一，并增加数据库互斥CHECK；USER分派必须绑定本租户Membership。
> 4. KEEP_PAUSED只记录Submission，HumanTask保持WAITING_USER，Run/Step继续WAITING_HUMAN；不创建ResumeSignal、不创建后继任务。
> 5. Day 3A暂不实现会改变业务结论的`cancel`。取消HumanTask不得自动解决Exception或使Run失败；未来业务终止使用独立`ABORT_RUN`语义。
> 6. ResumeSignal只用于真正恢复或重试流程的最终决定。
> 7. Active任务使用`dedupe_key + partial unique index`，不要永久限制同一Exception只能有一个任何类型任务。
> 8. Worker制定并遵守统一锁顺序，事务内不得调用外部系统。
> 9. 人工确认作为MANUAL决策/Evidence保存，不冒充平台机器证据。
>
> 其余迁移、API、幂等、权限、Worker异步处理、历史保护、测试、Demo与Release计划按原方案执行。完成后停止，不进入Day 3B。
