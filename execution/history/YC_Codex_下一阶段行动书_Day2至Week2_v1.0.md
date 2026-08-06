---
document_id: yc-codex-next-actions-day2-week2
version: 1.0
status: ready_for_execution
updated_at: 2026-08-06
parent_decision: YC_战略与架构增量决策_稳定内核与业务扩展_v1.0.md
baseline: Day 1 P0 Fake Vertical Slice
---

# YC Codex下一阶段行动书：Day 2至Week 2 v1.0

## 0. 总目标

在接触真实云仓和抖音写入动作前，先把YC从“成功/失败二元Fake Runtime”升级为：

> **能够面对结果未知、人工介入、企业配置差异，并允许真实Connector安全挂载的最小稳定框架。**

禁止同时启动Agent Runtime、Designer Agent、通用DAG、完整前端和知识图谱。

---

# 1. 开始前：冻结Day 1基线

必须先完成：

```text
git status干净
git commit: feat: complete YC Day 1 P0 fake vertical slice
git tag: day1-p0
```

保存：

```text
docs/releases/day1-p0/
├─ README.md
├─ openapi.json
├─ pytest-output.txt
├─ alembic-check.txt
├─ demo-transcript.txt
└─ KNOWN_GAPS.md
```

README记录Git Commit、Alembic Revision、测试结果和验收日期。

---

# 2. Day 2：可靠性闭环

## 2.1 唯一目标

实现：

```text
RESULT_UNKNOWN
→ RECONCILING
→ 查询外部结果
→ 已成功 / 确认未执行 / 仍不确定转人工
```

## 2.2 数据与状态

新增或扩展：

```text
JobLease generation/token
CommandAttempt.result_status enum
Exception
ReconciliationLink（可先使用字段关联）
```

状态至少支持：

```text
Attempt: SUCCEEDED / FAILED / RESULT_UNKNOWN
Command: DISPATCHED / SUCCEEDED / FAILED / RESULT_UNKNOWN
Step: WAITING_EXTERNAL / RECONCILING / WAITING_HUMAN / SUCCEEDED / FAILED
Run: RUNNING / RECONCILING / WAITING_HUMAN / SUCCEEDED / FAILED
```

## 2.3 Job租约加固

当前Worker不能只依赖`worker_id`。

领取Job时返回：

```text
job_id
lease_generation
lease_token
lease_expires_at
```

处理Job时必须验证Generation与Token；旧Worker迟到执行必须拒绝。

## 2.4 Fake Reconcile命令

增加：

```text
FAKE_QUERY_SAMPLE_SHIPMENT
```

测试路径A：

```text
创建命令回报RESULT_UNKNOWN
→ Run/Step进入RECONCILING
→ 创建查询Command
→ Fake Edge返回EXISTS
→ 原命令视为成功
→ Run完成
```

测试路径B：

```text
查询返回NOT_FOUND
→ 原动作确认未执行
→ 允许生成新的执行Attempt/Command
```

测试路径C：

```text
查询仍UNKNOWN
→ 创建MANUAL_REQUIRED Exception
→ Run进入WAITING_HUMAN
```

## 2.5 语义加固

- `subject_type`暂只允许`COOPERATION_CASE`；
- 内部子表查询统一附带`tenant_id`；
- Edge结果/Evidence JSON设置大小限制；
- `result_status`和主要状态使用Enum；
- HTTP Edge报告只保存结果、创建Job并返回，不直接推进业务。

## 2.6 Day 2测试

必须覆盖：

- 正常成功；
- 明确失败；
- 结果未知后查询为已存在；
- 结果未知后查询为不存在并重试；
- 仍不确定进入人工；
- Job旧Generation/Token被拒绝；
- Edge重复报告幂等；
- 冲突报告409；
- 跨租户异常读取拒绝。

## 2.7 Day 2 DoD

- 全部Day 1测试继续通过；
- 新状态在时间线可见；
- Fake外部副作用不会因为超时被盲目重复执行；
- 一键Demo可以展示成功、失败和未知三条路径。

---

# 3. Day 3：HumanTask与恢复信号

## 3.1 唯一目标

让Runtime能够正式等待人，而不是仅仅把Run标记失败。

## 3.2 最小模型

```text
human_tasks
notification_deliveries
secure_ephemeral_artifacts
resume_signals
```

HumanTask字段至少包括：

```text
id / tenant_id / run_id / step_run_id
task_type / status / assignee_type / assignee_id
prompt / input_schema / submitted_payload
due_at / expires_at / verification_status
created_at / resolved_at
```

## 3.3 首批任务类型

```text
AUTH_QR_SCAN
SMS_OTP_REQUIRED
EXTERNAL_RESULT_CONFIRMATION
APPROVAL_REQUIRED
MISSING_INFORMATION
MANUAL_EXCEPTION_RESOLUTION
```

## 3.4 Fake Human流程

先不接飞书和微信，使用API/Fake Notification：

```text
Fake Connector返回AUTH_QR_REQUIRED
→ 创建HumanTask
→ 保存短时二维码Fake Artifact
→ Run进入WAITING_HUMAN
→ 用户提交“已扫码”
→ 创建验证Job
→ Fake Edge Healthcheck确认登录恢复
→ ResumeSignal
→ 原Step继续
```

关键：用户提交不等于完成，必须由系统再次验证。

## 3.5 安全规则

- OTP不写Activity详情、Evidence和普通日志；
- 临时Artifact有TTL和一次性访问Token；
- HumanTask完成后删除二维码原图；
- 永久审计只记录谁在何时完成何种任务；
- 通知发送失败不丢HumanTask。

## 3.6 API

```text
GET  /api/v1/human-tasks
GET  /api/v1/human-tasks/{id}
POST /api/v1/human-tasks/{id}/submit
POST /api/v1/human-tasks/{id}/cancel
GET  /api/v1/secure-artifacts/{token}
```

## 3.7 Day 3 DoD

- Run可进入`WAITING_HUMAN`并跨进程重启保留；
- 人工提交后不直接完成，而是验证后恢复；
- 超时任务进入Exception；
- Tenant A不能读取Tenant B任务和Artifact；
- Demo可展示“登录过期→扫码→恢复”。

---

# 4. Day 4：企业差异配置

## 4.1 唯一目标

证明第二家企业/第二位员工的飞书结构差异可以通过配置承载，而不修改Cell代码。

## 4.2 最小模型

```text
source_bindings
mapping_profiles
mapping_profile_revisions
rule_sets
rule_set_revisions
notification_profiles
schema_snapshots
schema_drift_events
```

## 4.3 Canonical字段

先定义KOC最小字段：

```text
creator.display_name
creator.douyin_id
cooperation.status
commercial.fixed_fee
commercial.commission_rate
sample.route
sample.recipient_name
sample.recipient_phone
sample.recipient_address
platform.cooperation_code
owner.user_id
```

## 4.4 两套Fake Source

创建：

```text
员工A表：达人UID / 坑位费 / 寄样形式
员工B表：抖音ID / 拍摄费 / 发货方式
```

两套Mapping最终生成相同Canonical对象和CooperationCase。

## 4.5 Schema Drift

实现：

```text
字段删除/类型变化/映射失效
→ 暂停对应Deployment新Run
→ 创建SCHEMA_DRIFT Exception
→ 候选映射
→ Dry Run
→ 人工发布新Revision
```

## 4.6 配置冻结

每个Run必须保存：

- Cell版本；
- Mapping版本；
- RuleSet版本；
- Connector版本/模拟版本；
- SourceBinding。

旧Run不得静默读取新配置。

## 4.7 Day 4 DoD

- 两套不同源结构运行同一Fake Cell；
- 修改列名不改Cell代码；
- 删除必填字段时自动暂停；
- Dry Run不产生外部副作用；
- 发布新Mapping后新Run使用新版本，旧Run版本不变。

---

# 5. Day 5：Core/Cell边界与有限重构

## 5.1 唯一目标

停止`services.py`继续膨胀，为真实Cell建立扩展边界。

## 5.2 目标目录

```text
app/
├─ domains/koc/
├─ runtime/
│  ├─ runs.py
│  ├─ jobs.py
│  ├─ commands.py
│  ├─ attempts.py
│  ├─ results.py
│  ├─ human_tasks.py
│  ├─ reconciliation.py
│  └─ handler_registry.py
├─ cells/
│  └─ fake_sample/
└─ connectors/
   └─ fake/
```

## 5.3 Handler Registry

替换总入口中的业务`if/elif`：

```text
job_type → JobHandler
command_type → ConnectorHandler
cell_key/version → CellDefinition
```

禁止动态导入任意代码；只注册受信任内部Handler。

## 5.4 Step Outcome

定义内部统一结果：

```text
Complete
DispatchCommand
WaitHuman
WaitTimer
ResultUnknown
Fail
```

`RunAgent`先保留接口占位，不实现Agent Runtime。

## 5.5 重构纪律

- 所有测试先绿再移动；
- 每次只移动一类逻辑；
- 不改API契约，除非有明确迁移；
- 不抽象未被两个场景使用的复杂基类；
- 保留兼容层后再删除旧函数。

## 5.6 Day 5 DoD

- `services.py`不再拥有业务Cell分支；
- Fake Cell通过Registry运行；
- Day 1—4测试全部通过；
- 新增一个Fake Cell只需新包+注册，不修改Runtime核心分支。

---

# 6. Week 2：第一个真实Connector——云仓寄样履约

## 6.1 选择理由

优先云仓而非抖音合作单：

- 外部唯一ID通常更清晰；
- 查询是否创建更容易；
- 幂等与对账更容易验证；
- 与现有脚本能力匹配；
- 登录/二维码复杂度通常低于抖音。

如果实际云仓缺乏可查询外部ID或比抖音更不稳定，再通过Spike调整顺序。

## 6.2 Connector契约

```text
prepare(input, config)
execute(command, edge_context)
reconcile(command, external_reference)
healthcheck(profile)
capabilities()
```

Connector不得直接更新CellRun和CaseTrack，只返回结构化结果。

## 6.3 接入顺序

### 阶段A：现有脚本审计

记录：

- 输入；
- 输出；
- 鉴权；
- 外部唯一ID；
- 重复查询方式；
- 失败类型；
- 登录恢复；
- 可否官方API；
- Browser Service调用；
- 敏感字段。

### 阶段B：Read-only/Dry Run

- 校验输入；
- 解析商品映射；
- 查询登录态；
- 查询疑似重复发货单；
- 不创建订单。

### 阶段C：受控写入

限制：

- 单Tenant；
- 单账号/Profile；
- 单次1条；
- 必须人工审批；
- 全部Evidence；
- 失败不自动重试写入；
- 先Reconcile。

### 阶段D：物流查询与飞书写回

创建成功后：

- 持久化外部ID；
- 查询发货状态；
- 回传快递单号；
- 写回失败独立重试，不重复创建发货单。

## 6.4 Browser Service集成

Edge与Browser Service同机：

```text
Browser Service: 127.0.0.1:9527 + Token
```

先做能力握手：

```text
service_version
api_contract_version
capabilities
browser_version
profile_health
```

Cloud不获得任意低级浏览器能力。

## 6.5 Week 2真实验收

至少完成：

1. 真实Dry Run；
2. 人工审批一条真实寄样；
3. 创建成功并获得外部唯一ID；
4. 模拟飞书写回失败，再恢复写回；
5. 重复触发不创建第二单；
6. Browser/Edge重启后可继续；
7. 登录过期进入HumanTask并恢复；
8. 完整Evidence和时间线；
9. 统计人工基线与实际节省时间。

---

# 7. 本阶段禁止事项

- 不建设Designer Agent；
- 不建设完整Agent Runtime；
- 不建设临时容器集群；
- 不做通用DAG/拖拽编辑器；
- 不做大型Wiki或知识图谱；
- 不同时接云仓、抖音、飞书真实写入；
- 不自动操作个人微信；
- 不自动投流、转账、改价；
- 不因为目录重构而暂停真实纵向价值；
- 不把企业差异做成代码分支。

---

# 8. Codex执行方式

每个Day/任务包：

```text
1. 先复述目标和不可变边界
2. 列出修改文件
3. 只完成当前任务包
4. 执行迁移、测试和Demo
5. 汇报结果和风险
6. 等待人工验收后进入下一包
```

汇报格式：

```text
任务包：
完成状态：
修改文件：
迁移：
API/契约：
新增状态：
测试结果：
Demo记录：
已知风险：
开放问题：
是否建议进入下一步：
```

遇到业务语义不清，不猜测，写入`docs/OPEN_QUESTIONS.md`。

---

# 9. 下一阶段四道门

## 技术门

- 未知结果不盲目重试；
- 人工等待可持久恢复；
- 企业差异主要靠配置；
- Runtime与Cell已分层；
- 第一个真实副作用可对账。

## 产品门

- 业务人员看得懂异常和下一步；
- 管理者能看到运行状态；
- 登录过期不是“找技术人员”；
- 第二张不同结构表无需改代码。

## 业务门

- 真实寄样连续运行；
- 漏单/重复/回写延迟可统计；
- 员工重复操作减少；
- 创始人人工维护时间记录并下降。

## 商业门

- 可稳定展示成功与异常；
- 能解释企业差异如何适配；
- 能说明部署、安全与人工交互；
- 形成30天付费试点的可信样板。
