---
document_id: yc-doc-center
title: YC文档中心
version: 5.0
status: ACCEPTED
updated_at: 2026-08-06
owner: YC Founder
---

# YC文档中心 README v5.0

## 1. 当前一句话状态

> YC已经冻结`day1-p0`和`day2-reliability`两个Fake工程基线；当前唯一主任务是按审核意见完成Day 3A HumanTask核心。Codex必须先使用修订后的Day 3A v2.0执行基线，不得直接依据旧v1.0或完整Day 3方案编码。

当前阶段仍未接入真实飞书、云仓、抖音或Browser Service。

---

## 2. 当前执行顺序

```text
当前：
Day 3A HumanTask核心
→ HumanTask / Submission / ResumeSignal
→ 人工确认已完成、未执行、继续暂停
→ 权限、幂等、并发、审计和持久恢复

随后：
Day 3B Fake二维码、临时安全Artifact与通知
→ 企业差异配置
→ Core/Cell有限拆分
→ 第一个真实云仓Connector
→ Browser Service正式集成
→ 抖音合作单Cell
```

当前禁止提前建设：

- 真实平台Connector；
- Browser Service真实写入；
- Agent Runtime；
- Designer Agent；
- 临时容器集群；
- 通用DAG；
- 大型Wiki或图数据库；
- 自动个人微信；
- 投流自动化。

---

## 3. 当前权威文件

### 治理

- `governance/YC_持续文档治理与Agent交接协议_v2.0.md`

### 当前状态与上下文

- `context/YC_当前状态快照_2026-08-06_v4.0.md`
- `context/YC_新战略顾问接管上下文_v1.0.md`

### 战略

- `strategy/YC_战略规划_商业计划书_v3.0.md`
- `strategy/YC_战略与架构增量决策_稳定内核与业务扩展_v1.0.md`

### 产品与架构

- `product/YC_KOC合作经营中枢_产品技术方案_v2.1.md`
- `architecture/YC_Browser_Service集成规范_v1.0.md`
- `open-questions/YC_关键开放问题与决策门_v1.0.md`

### 当前执行

当前唯一执行基线：

- `execution/current/YC_Codex_Day3A_HumanTask核心_实施任务书_v2.0.md`

当前审核依据：

- `execution/current/YC_Codex_Day3A实施计划_审核意见_v1.0.md`

以下文件只作为上层参考，不是当前直接执行基线：

- `execution/current/YC_Codex_Day3_HumanTask实施任务书_v2.0.md`

旧的Day 3A v1.0在v2.0落库后移入`execution/history/`并标记`SUPERSEDED`。

### 销售

- `sales/YC_产品一页纸_KOC合作经营中枢_v1.0.md`
- `sales/YC_5分钟产品演示脚本_v1.0.md`
- `sales/YC_企业采购者展示话术_v2.0.md`
- `sales/YC_30天付费试点方案_v1.0.md`

若继续保留采购PPT，应将`YC企业采购者展示PPT_v2.0.pptx`提交到`sales/`；当前仓库快照中未包含该文件。

---

## 4. 已冻结工程基线

### Day 1

- Commit：`b78193c`
- Tag：`day1-p0`
- 状态：`VALIDATED_FOR_FAKE`

### Day 2.1

- Commit：`74f2192`
- Tag：`day2-reliability`
- Alembic：`0003_day2_1_semantic_cleanup`
- 全量测试：`16 passed`
- Demo：`11 passed`
- 状态：`VALIDATED_FOR_FAKE`

工程事实以`yc/docs/releases/`为准，`docs-gpt`不重复存放OpenAPI、测试输出和Demo transcript。

---

## 5. 当前目录治理动作

### 应保留

- `context/YC_当前状态快照_2026-08-06_v4.0.md`

### 应归档

- `context/YC_当前状态快照_2026-08-06.md`
- `context/YC_当前状态快照_2026-08-06_v3.0.md`

归档时标记：

```yaml
status: SUPERSEDED
superseded_by: YC_当前状态快照_2026-08-06_v4.0.md
```

### Day 3A文件

v2.0落库后：

- v2.0留在`execution/current/`
- v1.0移入`execution/history/`
- 审核意见可在Day 3A验收后移入`execution/history/`

### 空目录

使用`.gitkeep`或目录README，不再使用名为“空”“空白”的占位文件。

---

## 6. 阅读顺序

### 新战略顾问

1. 本文件；
2. 当前状态快照v4.0；
3. 新战略顾问接管上下文；
4. 战略规划；
5. 架构增量决策；
6. 产品技术方案；
7. 当前执行基线；
8. 开放问题。

### Codex

1. 本文件；
2. 当前状态快照v4.0；
3. 当前Day 3A v2.0执行基线；
4. Day 3A审核意见；
5. 产品技术方案相关章节；
6. OPEN_QUESTIONS；
7. 工程Release与代码。

---

## 7. 更新触发

完成以下任一事件后必须更新本文件和当前状态快照：

- Day 3A冻结；
- Day 3B冻结；
- 第一个真实云仓写入；
- Browser Service正式接入；
- 第一家外部付费试点；
- 产品定位、技术基线或当前唯一任务变化。
