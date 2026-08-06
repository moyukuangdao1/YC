---
document_id: yc-codex-day3b-auth-qr-notification
title: YC Codex Day 3B｜认证二维码与通知Fake闭环实施任务书
version: 2.0
status: EXECUTION_BASELINE
updated_at: 2026-08-06
depends_on:
  - day3a-human-task-core
incorporates:
  - YC_Codex_Day3B实施前统一决策_v1.0.md
supersedes:
  - YC_Codex_Day3B_认证二维码与通知Fake闭环_实施任务书_v1.0.md
next_stage:
  - tenant-configuration
---

# YC Codex Day 3B｜认证二维码与通知Fake闭环实施任务书 v2.0

## 0. 唯一目标

在不接真实Browser Service和消息平台的前提下，完成：

```text
认证前置健康检查
→ QR_REQUIRED
→ HumanTask
→ SecureEphemeralArtifact
→ In-App通知
→ 人提交QR_SCANNED
→ 系统二次验证
→ 失败继续等待
→ 成功后恢复原流程
```

并覆盖：

- `INVALID_ACCOUNT`转人工；
- `UNKNOWN`有界重试后转人工；
- Artifact过期、刷新、版本与删除；
- 通知重试和失败不阻塞业务。

---

# 1. 基线与部署

- 基于`e55c999 / day3a-human-task-core`；
- Day 3A正式验收通过；
- 新增Day 3B专用Cell Definition；
- 创建两个Day 3B种子Deployment；
- Day 1—3A原Fake Sample Deployment与测试不变；
- 新增迁移`0005_day3b_auth_qr_notification`；
- 不修改0001—0004。

---

# 2. 模块边界

新增小型模块：

```text
app/auth_runtime/
app/artifacts/
app/notifications/
```

`services.py`只保留分派或兼容入口。

禁止：

- 全面重构Runtime；
- 将全部Day 3B业务继续堆入`services.py`；
- 在数据库事务内执行文件或Provider I/O。

---

# 3. 认证检查

## 3.1 业务结果

Attempt状态仍是：

```text
SUCCEEDED / FAILED / RESULT_UNKNOWN
```

认证事实：

```text
AUTHENTICATED
QR_REQUIRED
INVALID_ACCOUNT
UNKNOWN
```

## 3.2 AuthVerification

新增`auth_verifications`，关联：

- Run；
- Step；
- Healthcheck Command/Attempt；
- HumanTask；
- Submission；
- `PREFLIGHT / POST_QR`；
- `auth_state`；
- 检查时间和结果快照。

## 3.3 路径

### AUTHENTICATED

```text
继续后续Fake写入流程
不创建HumanTask
```

### QR_REQUIRED

```text
LOGIN_REQUIRED Exception
AUTH_QR_SCAN HumanTask
SecureEphemeralArtifact
NotificationDelivery
Run/Step WAITING_HUMAN
```

### INVALID_ACCOUNT

```text
AUTH_MANUAL_REQUIRED Exception
MANUAL_EXCEPTION_RESOLUTION HumanTask
不生成QR
Run/Step WAITING_HUMAN
```

### UNKNOWN

认证检查总尝试数最多3次，通过`available_at`退避。

耗尽后：

```text
AUTH_MANUAL_REQUIRED
MANUAL_EXCEPTION_RESOLUTION
WAITING_HUMAN
```

---

# 4. QR Submission

请求：

```json
{
  "decision": "QR_SCANNED",
  "payload": {},
  "expected_version": 1
}
```

规则：

- payload必须为空对象；
- 未知字段422；
- 扫码时间使用服务端时间；
- HTTP只创建Submission和验证Job；
- 不创建ResumeSignal；
- 不推进Run。

验证失败：

```text
Submission = VERIFICATION_FAILED
HumanTask = WAITING_USER
Run/Step = WAITING_HUMAN
无ResumeSignal
允许新Submission
```

验证成功：

```text
Submission = APPLIED
HumanTask = RESOLVED
Exception = RESOLVED
创建并消费一个ResumeSignal
删除QR原始字节
恢复原流程
```

---

# 5. SecureEphemeralArtifact

## 5.1 模型

```text
id
tenant_id
human_task_id
artifact_type
status
storage_ref
checksum_sha256
version_no
expires_at
deleted_at
created_at
```

状态：

```text
PENDING
AVAILABLE
EXPIRED
DELETED
FAILED
```

## 5.2 ArtifactStore

接口：

```text
put
get
delete
exists
```

Day 3B使用可配置本地目录。

要求：

- 原始字节不进PostgreSQL；
- `storage_ref`不透明；
- 文件名由系统生成；
- 使用临时文件+原子rename；
- 数据库事务外进行文件I/O。

## 5.3 TTL

- Fake默认5分钟；
- 策略可配置；
- Connector未来提供过期时间时取较早值；
- 测试注入时钟。

## 5.4 Fake QR

使用固定版本`qrcode`/Pillow，通过`QrRenderer`接口生成可扫描Fake二维码。

内容：

```text
yc-fake-auth://...
```

不得包含真实敏感信息。

## 5.5 刷新

新增：

```text
POST /api/v1/human-tasks/{task_id}/auth-material/refresh
```

HTTP只创建幂等生成Job。

新Artifact AVAILABLE后：

- 旧Artifact EXPIRED；
- 新Delivery创建；
- 旧字节异步删除。

不做定时无限刷新。

## 5.6 读取

```text
GET /api/v1/human-tasks/{task_id}/artifacts
GET /api/v1/human-tasks/{task_id}/artifacts/{artifact_id}/content
```

列表仅返回：

```text
id / artifact_type / status / expires_at / deleted_at / created_at
```

内容读取：

- JWT；
- Tenant Context；
- `secure_artifact.read`；
- Assignee/角色/Owner/Admin；
- AUDITOR拒绝；
- 跨租户404；
- 过期/删除410；
- `Cache-Control: no-store`。

---

# 6. NotificationDelivery

## 6.1 Provider

实现：

- `IN_APP`：正常Demo；
- `FAKE`：测试成功、失败和重试。

同一任务正常流程不重复双发。

## 6.2 角色队列

向所有满足以下条件的Membership逐人创建Delivery：

- Tenant有效；
- User有效；
- 能提交该HumanTask；
- 能读取对应SecureArtifact。

每个收件人独立幂等。

## 6.3 状态与重试

```text
PENDING
SENDING
SENT
FAILED
```

默认总尝试数最多3次，通过Job `available_at`退避。

最终失败：

- Delivery=`FAILED`；
- HumanTask继续`WAITING_USER`；
- 记录技术事件和指标；
- 不创建第二业务Exception；
- 任务列表仍可发现任务。

## 6.4 Payload

只保存非敏感元数据，不保存字节、Base64、`storage_ref`、路径、OTP、Cookie和公开链接。

---

# 7. 两阶段Worker

文件和Provider I/O统一使用：

```text
事务1：
锁Job与业务记录
写PREPARED/SENDING状态
提交

事务外：
ArtifactStore或NotificationProvider调用

事务2：
使用Job lease generation/token重新校验
写成功/失败结果
创建后续Job
提交
```

固定锁顺序，事务内不做外部I/O。

---

# 8. 失败恢复

## Artifact生成失败

- 状态FAILED；
- 生成Job总尝试数最多3次；
- Task/Run继续等待；
- 不创建可用通知。

## Artifact删除失败

- 不提前写`deleted_at`；
- 删除Job幂等重试；
- 文件不存在视为成功。

## 二次验证仍QR_REQUIRED

- 当前Submission=`VERIFICATION_FAILED`；
- HumanTask回到WAITING_USER；
- 生成新Artifact版本；
- 旧版本在新版本AVAILABLE后失效；
- 创建新幂等Delivery；
- 不创建新HumanTask。

---

# 9. API

新增：

```text
POST /api/v1/human-tasks/{task_id}/auth-material/refresh
GET  /api/v1/human-tasks/{task_id}/artifacts
GET  /api/v1/human-tasks/{task_id}/artifacts/{artifact_id}/content
GET  /api/v1/notification-deliveries
GET  /api/v1/notification-deliveries/{id}
```

沿用HumanTask提交API。

---

# 10. 测试

至少覆盖：

1. 旧Day 1—3A全部回归；
2. 独立Day 3B Deployment；
3. AUTHENTICATED直接继续；
4. QR_REQUIRED创建AuthVerification、Task、Artifact和Delivery；
5. INVALID_ACCOUNT转人工且无QR；
6. UNKNOWN总尝试数和退避；
7. UNKNOWN耗尽转人工；
8. QR payload空对象；
9. 二次验证失败为VERIFICATION_FAILED；
10. 失败后同一Task可再次提交；
11. 验证成功只创建一个ResumeSignal；
12. ArtifactStore本地实现；
13. Artifact PENDING到AVAILABLE；
14. 可扫描Fake QR；
15. TTL注入时钟；
16. 过期410；
17. 手工刷新API；
18. 新版本可用后旧版本失效；
19. 任务解决后字节删除；
20. 元数据保留；
21. Artifact权限矩阵；
22. AUDITOR拒绝；
23. 列表不泄露路径与哈希；
24. 角色队列多收件人幂等Delivery；
25. IN_APP正常路径；
26. FAKE失败/重试；
27. Delivery最终失败不阻塞Task；
28. Artifact生成失败重试；
29. Artifact删除失败重试；
30. 两阶段Worker租约重新校验；
31. 敏感内容扫描；
32. 空库升级至0005；
33. 0004带数据升级；
34. downgrade/upgrade；
35. `alembic check`；
36. Demo可重复。

---

# 11. Demo

`scripts/demo_day3b_auth_qr.ps1`展示：

```text
A. AUTHENTICATED直接继续
B. QR_REQUIRED → 通知 → 扫码 → 第一次验证失败
C. 同一Task第二次提交 → 验证成功 → 恢复
D. Artifact自然过期 → 410 → 人工刷新 → 新版本
E. INVALID_ACCOUNT转人工
F. UNKNOWN三次后转人工
```

输出：

- AuthVerification；
- HumanTask和Submission；
- Artifact版本与生命周期；
- NotificationDelivery；
- ResumeSignal；
- Run/Step；
- 敏感字段扫描。

---

# 12. Release

迁移：

```text
0005_day3b_auth_qr_notification
```

Commit：

```text
feat: complete YC Day 3B auth QR and notification fake loop
```

Annotated Tag：

```text
day3b-auth-human-loop
```

Release目录：

```text
yc/docs/releases/day3b-auth-human-loop/
```

完成后停止，等待人工验收，不进入企业差异配置。

---

# 13. 禁止事项

- 不接真实Browser Service；
- 不接真实飞书、微信、企业微信、短信和邮件；
- 不接真实云仓或抖音；
- 不做真实OTP；
- 不建设公开二维码链接；
- 不建设正式前端；
- 不建设Agent Runtime；
- 不大规模重构Runtime；
- 不自动进入下一阶段。
