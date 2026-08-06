---
document_id: yc-codex-day3b-unified-decisions
title: YC Codex Day 3B实施前统一决策
version: 1.0
status: ACCEPTED
updated_at: 2026-08-06
depends_on:
  - day3a-human-task-core
applies_to:
  - YC Codex Day 3B认证二维码与通知Fake闭环
---

# YC Codex Day 3B实施前统一决策 v1.0

## 0. 总体结论

Day 3A正式验收通过，直接以`e55c999 / day3a-human-task-core`进入Day 3B，不创建Day 3A.1 Patch。

Day 3B采用独立Fake Deployment，不修改Day 1—3A冻结流程。

---

# 1. 三十项决策表

| # | 决策 | 选择与补充 |
|---:|---|---|
| 1 | Day 3A取消不变量 | **A**。现有`409 HUMAN_TASK_REQUIRED`语义验收通过；不为错误码名称创建Patch。 |
| 2 | Day 3B Deployment | **A**。新增Day 3B专用Cell Definition和两个种子Deployment；旧Fake Sample Deployment不变。 |
| 3 | `auth_verifications` | **A**。新增显式核心表，稳定关联Run、Step、Command/Attempt、HumanTask、Submission、purpose、auth_state与时间。 |
| 4 | `INVALID_ACCOUNT` | **A**。进入`WAITING_HUMAN`，创建`AUTH_MANUAL_REQUIRED`与`MANUAL_EXCEPTION_RESOLUTION`；不生成QR。 |
| 5 | `UNKNOWN` | **A（修订）**。认证检查总尝试数默认最多3次，使用`available_at`和可配置退避，不真实sleep；耗尽后转人工。 |
| 6 | `QR_SCANNED` payload | **A**。只允许`{}`；扫码提交时间使用服务端时间；未知字段422。 |
| 7 | 二次验证失败的Submission状态 | **B**。新增`VERIFICATION_FAILED`，避免把有效的人类提交误写成普通`REJECTED`；验证事实同时记录于`auth_verifications`。 |
| 8 | ResumeSignal时机 | **A**。HTTP提交不创建；只有系统验证为`AUTHENTICATED`后创建并消费。 |
| 9 | Artifact存储 | **A**。定义`ArtifactStore`接口；Day 3B使用可配置本地目录；数据库只保存不透明`storage_ref`、哈希与生命周期。 |
| 10 | Artifact状态 | **A**。`PENDING / AVAILABLE / EXPIRED / DELETED / FAILED`。 |
| 11 | QR TTL | **A（修订）**。策略可配置，Fake默认5分钟；若Connector提供真实过期时间，取两者较早值。测试注入时钟。 |
| 12 | Fake QR | **B**。使用固定版本的`qrcode`/Pillow生成可扫描但只包含Fake URI/Token的二维码；通过`QrRenderer`接口隔离依赖。 |
| 13 | 验证仍为QR_REQUIRED | **A**。生成新Artifact版本；新版本AVAILABLE后旧版本立即失效并删除字节；创建新版本幂等通知。 |
| 14 | 自然过期后的刷新 | **C**。提供显式、受权限控制、幂等的刷新API；不自动周期刷新，也不让任务因二维码过期进入无材料死状态。 |
| 15 | Artifact元数据 | **A**。原始字节删除，元数据永久保留用于审计。 |
| 16 | QR Artifact访问 | **A（加权限键）**。Owner/Admin、明确Membership Assignee、任务角色的有效成员且拥有`secure_artifact.read`；AUDITOR无内容读取权。 |
| 17 | Artifact列表字段 | **A**。仅返回ID、类型、状态和生命周期时间；不返回`storage_ref`、哈希、文件路径。 |
| 18 | 角色队列通知对象 | **A**。向所有有效且具备任务处理和Artifact读取权限的Membership逐人创建幂等Delivery。 |
| 19 | Provider | **A**。正常Demo使用`IN_APP`；`FAKE`只用于成功、失败和重试测试，不重复双发。 |
| 20 | 通知重试 | **A**。总尝试数默认最多3次，`available_at`退避，可注入时间。 |
| 21 | 通知最终失败 | **A**。Delivery失败，HumanTask继续`WAITING_USER`；记录技术事件/指标，不新增第二个业务Exception。 |
| 22 | Exception拆分 | **A**。`LOGIN_REQUIRED`用于QR；`AUTH_MANUAL_REQUIRED`用于无效账号或UNKNOWN耗尽。 |
| 23 | HumanTask类型 | **A**。QR使用`AUTH_QR_SCAN`；无效账号/UNKNOWN耗尽使用`MANUAL_EXCEPTION_RESOLUTION`。 |
| 24 | Provider事务边界 | **A**。严格两阶段；文件和Provider调用均在数据库事务外，回写时重新校验Job Lease Generation/Token。 |
| 25 | 代码组织 | **A**。新增小型`auth_runtime`、`artifacts`、`notifications`模块；`services.py`只保留分派/兼容入口，不全面重构。 |
| 26 | Artifact生成失败 | **A**。标记FAILED，生成Job最多3次；HumanTask/Run保持等待，不创建假通知。 |
| 27 | Artifact删除失败 | **A**。未实际删除前不写`deleted_at`；删除Job幂等重试；文件不存在视为成功。 |
| 28 | 任务解决后的Delivery | **A**。永久保留状态、时间和非敏感摘要；Artifact仅删除原始字节。 |
| 29 | Release命名 | **A**。迁移`0005_day3b_auth_qr_notification`；Commit与Tag按提案固定。 |
| 30 | 完成后是否继续 | **A**。完成Release后停止，等待人工验收；不进入企业差异配置。 |

---

# 2. 对第14项的特别说明

仅选择“过期返回410、不提供刷新”会产生死状态：

```text
Task仍WAITING_USER
→ QR已过期且不可读取
→ 人无法扫码
→ 又没有刷新入口
```

因此Day 3B必须提供：

```text
POST /api/v1/human-tasks/{task_id}/auth-material/refresh
```

该接口只：

- 校验Tenant、权限和Task状态；
- 创建幂等`GENERATE_AUTH_QR` Job；
- 不在HTTP事务中写文件；
- 不创建新HumanTask；
- 不创建ResumeSignal；
- 不推进Run。

生成新Artifact成功后，再原子切换当前有效版本并使旧版本失效。

---

# 3. `auth_verifications`最小字段

```text
id
tenant_id
run_id
step_run_id
command_id
attempt_id
human_task_id NULL
submission_id NULL
purpose                # PREFLIGHT / POST_QR
sequence_no
auth_state             # AUTHENTICATED / QR_REQUIRED / INVALID_ACCOUNT / UNKNOWN
result_snapshot
checked_at
created_at
```

约束：

- 同租户关联；
- `POST_QR`必须关联Submission；
- 不使用Command JSON或Job Payload作为唯一关联事实；
- Attempt执行状态继续保持通用，不加入认证业务状态。

---

# 4. Auth UNKNOWN重试策略

Day 3B只实现认证专用的有界重试，不建设通用重试平台：

```text
max_attempts_total = 3
```

默认退避可配置，例如：

```text
attempt 1：立即
attempt 2：+5秒
attempt 3：+15秒
```

测试使用注入时钟，不真实等待。

第三次仍为`UNKNOWN`：

```text
AUTH_MANUAL_REQUIRED
+ MANUAL_EXCEPTION_RESOLUTION
+ Run/Step WAITING_HUMAN
```

---

# 5. Scannable Fake QR

Fake QR内容：

```text
yc-fake-auth://<tenant>/<profile>/<random-token>
```

要求：

- 明确标识为Fake；
- 不包含真实Cookie、账号、OTP或平台Token；
- 随机Token仅用于Fake验证状态；
- 依赖固定版本；
- 通过`QrRenderer`接口封装；
- 原始PNG遵守SecureEphemeralArtifact的全部安全规则。

---

# 6. 通知与敏感数据

`NotificationDelivery.payload_snapshot`只能保存：

- HumanTask ID；
- 标题；
- 到期时间；
- Artifact ID；
- 非敏感提示。

禁止保存：

- QR原始字节；
- Base64；
- `storage_ref`；
- 文件路径；
- OTP；
- Browser Cookie；
- 公开可访问链接。

若角色队列没有任何可处理成员：

- HumanTask保持等待；
- Delivery不创建或标记`NO_ELIGIBLE_RECIPIENT`；
- 记录技术告警和指标；
- 不伪造通知成功。

---

# 7. Artifact生命周期

推荐转换：

```text
PENDING
→ AVAILABLE
→ EXPIRED
→ DELETED

PENDING
→ FAILED

AVAILABLE
→ DELETED       # Task解决时
```

新二维码版本生成：

```text
创建新PENDING记录
→ 事务外写入
→ 新记录AVAILABLE
→ 同事务将旧记录EXPIRED
→ 创建新Delivery
→ 异步删除旧字节
```

避免先删除旧二维码、但新二维码生成失败导致无可用材料。

---

# 8. Day 3B结束门

必须同时通过：

- Day 1—3A全部回归；
- 认证已登录直接通过；
- QR_REQUIRED完整闭环；
- 第一次扫码验证失败仍等待；
- 第二次验证成功恢复；
- INVALID_ACCOUNT转人工；
- UNKNOWN有界重试后转人工；
- Artifact权限、TTL、刷新、版本和删除；
- Notification幂等、失败与重试；
- 敏感内容扫描；
- 空库与0004带数据迁移；
- downgrade/upgrade；
- `alembic check`；
- Demo；
- Commit、Tag、Release；
- 工作区干净。

完成后停止。
