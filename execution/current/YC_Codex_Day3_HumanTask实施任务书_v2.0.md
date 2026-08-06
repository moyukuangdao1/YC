---
document_id: yc-codex-day3-human-task
title: YC Codex Day 3｜HumanTask与人工恢复实施任务书
version: 2.0
status: READY_AFTER_DAY2_1
updated_at: 2026-08-06
depends_on:
  - day2-reliability
  - day2.1-semantic-cleanup
---

# YC Codex Day 3｜HumanTask与人工恢复实施任务书 v2.0

## 0. 今日唯一目标

让YC正式拥有一种新的执行者：

> **Human Worker**

系统必须能够持久等待人完成扫码、验证码、外部结果确认或信息补全；进程重启后任务仍存在，且人的提交不能直接被当成业务成功，必须经过系统验证后恢复原Step。

今天不接真实飞书、微信、短信和Browser Service。

---

## 1. 开工前检查

必须满足：

- Day 2.1状态语义修正完成；
- `day2-reliability` Tag存在；
- Day 1和Day 2测试全部通过；
- 工作区干净；
- 当前Alembic Head可从空库升级。

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
verification_status
due_at
expires_at
created_at
notified_at
submitted_at
resolved_at
version
```

任务类型：

```text
AUTH_QR_SCAN
SMS_OTP_REQUIRED
EXTERNAL_RESULT_CONFIRMATION
APPROVAL_REQUIRED
MISSING_INFORMATION
MANUAL_EXCEPTION_RESOLUTION
```

状态：

```text
OPEN
NOTIFIED
WAITING_USER
SUBMITTED
VERIFYING
RESOLVED
EXPIRED
CANCELLED
FAILED
```

### 2.2 NotificationDelivery

字段：

```text
id
tenant_id
human_task_id
provider
recipient_ref
status
attempt_count
last_error_code
sent_at
created_at
```

Day 3只实现：

```text
FAKE
IN_APP
```

不接真实消息渠道。

### 2.3 SecureEphemeralArtifact

用于二维码等临时敏感材料：

```text
id
tenant_id
human_task_id
artifact_type
storage_ref
access_token_hash
expires_at
consumed_at
deleted_at
created_at
```

要求：

- 短TTL；
- 一次性访问Token；
- 不能通过普通Evidence API永久读取；
- 过期或消费后失效；
- 原始内容不进入Activity、Audit和Wiki。

### 2.4 ResumeSignal

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

用于把人的结果交回Runtime。

---

## 3. Fake业务路径

### 3.1 Fake QR登录

新增Fake命令或Fake Connector结果：

```text
AUTH_QR_REQUIRED
```

流程：

```text
Fake执行步骤发现登录失效
→ 创建Exception(LOGIN_REQUIRED)
→ 创建HumanTask(AUTH_QR_SCAN)
→ 创建SecureEphemeralArtifact(二维码)
→ Run/Step进入WAITING_HUMAN
→ 创建Fake NotificationDelivery
→ 用户提交“已扫码”
→ HumanTask进入SUBMITTED
→ 创建VERIFY_LOGIN Job
→ Fake Edge/Program返回LOGIN_VALID
→ HumanTask RESOLVED
→ Exception RESOLVED
→ ResumeSignal
→ Step从等待点继续
→ Run完成
```

### 3.2 外部结果人工确认

流程：

```text
Reconcile仍UNKNOWN
→ HumanTask(EXTERNAL_RESULT_CONFIRMATION)
→ 用户选择：
   - EXTERNAL_ACTION_COMPLETED
   - EXTERNAL_ACTION_NOT_EXECUTED
   - KEEP_PAUSED
→ 系统验证权限和输入
→ 创建后续Job
→ 已完成：按成功继续
→ 未执行：允许安全重试
→ 暂停：保持WAITING_HUMAN
```

---

## 4. 关键纪律

### 4.1 人的提交不等于业务完成

```text
SUBMITTED
→ VERIFYING
→ RESOLVED
```

二维码场景必须由Fake Login Healthcheck确认。

### 4.2 权限

- 普通Operator只能处理分派给自己的HumanTask；
- KOC_MANAGER可处理本租户业务任务；
- Tenant Admin可重新分派；
- 跨租户访问返回404/403；
- Edge设备不能调用人类任务API。

### 4.3 幂等

- 相同提交内容重复提交返回原结果；
- 不同提交内容在已提交后返回409；
- 已解决任务不能再次改变结论；
- ResumeSignal只能消费一次。

### 4.4 安全

OTP场景只实现接口和红线，不保存真实OTP：

- submitted_payload中的OTP字段必须在处理后清除或单独短期存储；
- 日志、Activity、Audit、Evidence中不得出现明文OTP；
- 二维码Artifact不进入永久Evidence；
- 普通诊断导出不能包含二维码或OTP。

### 4.5 Runtime一致性

进入`WAITING_HUMAN`时，必须保证至少存在一个有效Open HumanTask。

HumanTask解决后：

- 更新任务；
- 更新Exception；
- 创建ResumeSignal；
- 创建后续Job；

应在同一事务中完成。

---

## 5. API

```text
GET  /api/v1/human-tasks
GET  /api/v1/human-tasks/{task_id}
POST /api/v1/human-tasks/{task_id}/submit
POST /api/v1/human-tasks/{task_id}/cancel
POST /api/v1/human-tasks/{task_id}/reassign

GET  /api/v1/secure-artifacts/{access_token}
```

列表支持：

```text
status
task_type
assignee
run_id
due_before
```

---

## 6. 状态转换

### HumanTask

```text
OPEN → NOTIFIED → WAITING_USER
WAITING_USER → SUBMITTED
SUBMITTED → VERIFYING
VERIFYING → RESOLVED / FAILED / WAITING_USER
OPEN/NOTIFIED/WAITING_USER → EXPIRED / CANCELLED
```

### Run / Step

```text
RECONCILING → WAITING_HUMAN
WAITING_HUMAN → RUNNING / RECONCILING / FAILED / CANCELLED
```

CaseTrack第一版可以保持`BLOCKED`或`IN_PROGRESS`，但时间线必须明确“等待人类”。

---

## 7. 测试

必须覆盖：

1. Fake QR登录完整恢复；
2. 用户提交后验证失败，任务回到`WAITING_USER`；
3. 二维码Token一次性访问；
4. 二维码过期拒绝；
5. 重复相同提交幂等；
6. 冲突提交409；
7. ResumeSignal只能消费一次；
8. `EXTERNAL_ACTION_COMPLETED`路径；
9. `EXTERNAL_ACTION_NOT_EXECUTED`路径；
10. `KEEP_PAUSED`路径；
11. HumanTask与Tenant隔离；
12. Edge不能调用用户任务API；
13. OTP不出现在日志/Evidence/Activity快照；
14. API/Worker重启后任务仍可恢复；
15. Day 1和Day 2全部回归通过。

---

## 8. Demo

创建：

```text
scripts/demo_day3_human_task.ps1
```

演示：

```text
1. 启动Fake Run
2. 触发AUTH_QR_REQUIRED
3. 查看Run进入WAITING_HUMAN
4. 读取一次性二维码Artifact
5. 提交“已扫码”
6. Fake验证失败一次
7. 重新提交/验证成功
8. Run恢复并完成
9. 查看Activity、Audit和HumanTask时间线
```

---

## 9. Definition of Done

- HumanTask持久化并有明确权限；
- Run可以安全等待人；
- 人提交后由系统验证；
- 二维码和OTP遵守安全红线；
- ResumeSignal可恢复原流程；
- 跨进程重启后不丢；
- 全部回归测试通过；
- Demo可重复执行；
- 不接真实消息渠道和Browser Service。

---

## 10. 禁止事项

- 不自动操作个人微信；
- 不接飞书机器人；
- 不接真实短信；
- 不接真实二维码登录；
- 不开始SourceBinding/MappingProfile；
- 不拆分全部`services.py`；
- 不建设Agent Runtime；
- 不建设正式前端；
- 不把HumanTask简化成Exception状态字段。
