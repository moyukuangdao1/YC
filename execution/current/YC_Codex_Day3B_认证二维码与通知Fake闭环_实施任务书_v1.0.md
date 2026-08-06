---
document_id: yc-codex-day3b-auth-qr-notification
title: YC Codex Day 3B｜认证二维码与通知Fake闭环实施任务书
version: 1.0
status: READY_AFTER_DAY3A_ACCEPTANCE
updated_at: 2026-08-06
depends_on:
  - day3a-human-task-core
next_stage:
  - tenant-configuration
---

# YC Codex Day 3B｜认证二维码与通知Fake闭环实施任务书 v1.0

## 0. 唯一目标

在不接真实Browser Service和消息平台的前提下，验证：

```text
执行前认证检查
→ 发现需要二维码登录
→ 生成短时安全材料
→ 通知正确的人
→ 人提交“已扫码”
→ 系统重新验证
→ 失败继续等待
→ 成功后恢复原流程
```

Day 3B只做Fake/In-App闭环。

---

## 1. 重要架构决策

### 1.1 先认证检查，再执行写动作

不模拟“写入命令执行到一半才发现登录失效”。

采用：

```text
FAKE_BROWSER_PROFILE_HEALTHCHECK
→ AUTHENTICATED
或
→ QR_REQUIRED
```

只有认证健康后，Runtime才允许创建后续Fake写入Command。

这避免在登录不明时触碰真实外部副作用。

### 1.2 Attempt状态保持通用

认证检查Attempt仍使用：

```text
SUCCEEDED / FAILED / RESULT_UNKNOWN
```

认证业务事实放在结构化结果中：

```text
auth_state:
- AUTHENTICATED
- QR_REQUIRED
- INVALID_ACCOUNT
- UNKNOWN
```

不得把`QR_REQUIRED`加入通用Attempt执行状态。

---

## 2. 数据模型

### 2.1 SecureEphemeralArtifact

```text
secure_ephemeral_artifacts
- id
- tenant_id
- human_task_id
- artifact_type
- storage_ref
- checksum_sha256
- expires_at
- deleted_at
- created_at
```

首批：

```text
AUTH_QR_IMAGE
```

要求：

- 原始字节不存PostgreSQL；
- 原始字节不进入Evidence；
- 不写Activity/Audit/普通日志；
- 任务解决或过期后删除；
- 只允许任务Assignee、对应角色和管理员读取；
- Day 3B使用需要JWT与Tenant Context的受控下载接口；
- 暂不实现公开一次性链接，避免链接预览或爬虫提前消费。

### 2.2 NotificationDelivery

```text
notification_deliveries
- id
- tenant_id
- human_task_id
- provider_key
- recipient_membership_id
- template_key
- payload_snapshot
- idempotency_key
- status
- attempt_count
- last_error_code
- sent_at
- created_at
```

Provider：

```text
FAKE
IN_APP
```

状态：

```text
PENDING
SENDING
SENT
FAILED
```

通知不是业务事实来源；HumanTask才是。

### 2.3 HumanTask扩展

新增任务类型：

```text
AUTH_QR_SCAN
```

Submission决策：

```text
QR_SCANNED
```

系统验证失败时：

- 当前Submission标记为`REJECTED`或`VERIFICATION_FAILED`；
- HumanTask回到`WAITING_USER`；
- 允许新的Submission；
- 不创建ResumeSignal。

---

## 3. Fake认证流程

### 3.1 已登录

```text
Healthcheck Command
→ Attempt SUCCEEDED
→ auth_state=AUTHENTICATED
→ 不创建HumanTask
→ 原流程继续
```

### 3.2 需要二维码

```text
Healthcheck Command
→ Attempt SUCCEEDED
→ auth_state=QR_REQUIRED
→ 创建/复用LOGIN_REQUIRED Exception
→ 创建/复用AUTH_QR_SCAN HumanTask
→ 创建SecureEphemeralArtifact
→ 创建NotificationDelivery
→ Run/Step进入WAITING_HUMAN
```

### 3.3 人提交已扫码，验证失败

```text
HumanTaskSubmission(QR_SCANNED)
→ 创建VERIFY_AUTH Job
→ Fake Healthcheck返回QR_REQUIRED
→ Submission标记VERIFICATION_FAILED
→ HumanTask回到WAITING_USER
→ 不创建ResumeSignal
→ 可更新二维码Artifact
→ 可重新发送幂等通知
```

### 3.4 人提交已扫码，验证成功

```text
HumanTaskSubmission(QR_SCANNED)
→ VERIFY_AUTH Job
→ auth_state=AUTHENTICATED
→ HumanTask RESOLVED
→ Exception RESOLVED
→ 创建并消费ResumeSignal
→ 删除二维码Artifact
→ 原Step恢复
→ 原流程继续
```

---

## 4. 通知Provider接口

定义最小接口：

```python
class NotificationProvider(Protocol):
    provider_key: str

    def send(
        self,
        *,
        delivery_id: UUID,
        recipient_membership_id: UUID,
        template_key: str,
        payload: dict,
        artifact_refs: list[UUID],
    ) -> ProviderResult: ...
```

Day 3B实现：

- `FakeNotificationProvider`
- `InAppNotificationProvider`

不得接飞书、企业微信、微信、短信或邮件。

发送使用持久化Job与幂等键，不在创建HumanTask的事务中执行外部发送。

---

## 5. API

### HumanTask

继续使用现有：

```text
GET /api/v1/human-tasks
GET /api/v1/human-tasks/{id}
POST /api/v1/human-tasks/{id}/submit
```

### Artifact

```text
GET /api/v1/human-tasks/{task_id}/artifacts
GET /api/v1/human-tasks/{task_id}/artifacts/{artifact_id}/content
```

要求：

- JWT；
- Tenant Context；
- Assignee/角色/管理员权限；
- 过期或已删除返回410；
- 跨租户返回404；
- 响应禁止缓存。

### In-App Delivery

```text
GET /api/v1/notification-deliveries
GET /api/v1/notification-deliveries/{id}
```

---

## 6. 安全红线

- QR图片不进入永久Evidence；
- QR原始内容不进入日志、Activity、Audit和数据库JSON快照；
- Artifact响应设置`Cache-Control: no-store`；
- 过期和任务解决后删除字节；
- Notification payload不包含图片Base64；
- Provider只获得授权Artifact引用；
- Day 3B不实现真实OTP；
- 预留`SMS_OTP_REQUIRED`类型，但不创建真实流程；
- 任何OTP不得写入普通数据库、Evidence和日志。

---

## 7. 幂等与恢复

- 同一HumanTask同一Artifact版本只生成一个有效Delivery；
- 重复发送返回原Delivery；
- Provider失败可由Job重试；
- API/Worker重启后Artifact、Task和Delivery状态仍存在；
- 删除Artifact Job幂等；
- 验证Job重复处理不重复创建ResumeSignal；
- 验证失败不创建新HumanTask；
- QR刷新使用新Artifact，旧Artifact失效。

---

## 8. 测试

必须覆盖：

1. 已认证直接继续；
2. QR_REQUIRED创建Task、Artifact和Delivery；
3. 重复QR_REQUIRED不重复创建有效Task；
4. Artifact只有授权成员可读；
5. 跨租户Artifact拒绝；
6. Artifact过期返回410；
7. Artifact删除后不可读；
8. QR内容不出现在日志/Evidence/Activity/Audit；
9. Fake通知发送成功；
10. 通知幂等；
11. 通知失败后Job重试；
12. 第一次扫码验证失败，Task继续WAITING_USER；
13. 失败后可再次提交；
14. 第二次验证成功并恢复；
15. 验证失败不创建ResumeSignal；
16. 验证成功只创建一个ResumeSignal；
17. 任务解决后Artifact删除；
18. Worker重复处理幂等；
19. 进程重启后恢复；
20. Day 1—3A全部回归；
21. Alembic空库升级；
22. 上一Revision带数据升级；
23. downgrade/upgrade往返；
24. `alembic check`无漂移。

---

## 9. Demo

创建：

```text
scripts/demo_day3b_auth_qr.ps1
```

展示：

```text
1. Fake认证检查发现QR_REQUIRED
2. 创建HumanTask
3. 查看In-App通知
4. 读取二维码Artifact
5. 人提交QR_SCANNED
6. 第一次系统验证失败
7. Task继续等待
8. 再次提交
9. 第二次验证成功
10. ResumeSignal恢复流程
11. Artifact已删除
```

输出：

- Run/Step；
- HumanTask；
- Submission历史；
- Verification结果；
- NotificationDelivery；
- Artifact生命周期；
- ResumeSignal；
- Activity与Audit；
- 关键敏感字段扫描结果。

---

## 10. Release

通过后：

```text
commit:
feat: complete YC Day 3B auth QR and notification fake loop

annotated tag:
day3b-auth-human-loop
```

Release目录：

```text
yc/docs/releases/day3b-auth-human-loop/
```

完成后停止，不进入企业差异配置。

---

## 11. 禁止事项

- 不接真实飞书；
- 不接微信/企业微信；
- 不接真实短信；
- 不接Browser Service；
- 不接真实云仓和抖音；
- 不建设公开二维码链接；
- 不做真实OTP；
- 不建设正式前端；
- 不做Agent Runtime；
- 不大规模重构`services.py`。
