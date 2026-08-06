\---

document\_id: yc-doc-center
title: YC文档中心
version: 4.0
status: ACCEPTED
updated\_at: 2026-08-06
owner: YC Founder
---

# YC文档中心 README v4.0

## 1\. 当前一句话状态

> YC已完成Day 1 Fake纵向闭环和Day 2 Fake可靠性闭环；当前唯一任务是完成Day 2.1状态语义清理与`day2-reliability` Release冻结，完成后才进入Day 3 HumanTask。

Day 1已验证的闭环为：KOC案例及7条轨道，经Job、Command、CommandAttempt、Fake Edge、Evidence，最终推进寄样履约轨道完成。

\---

## 2\. 当前执行顺序

```text
当前：
Day 2.1 语义清理 + Release冻结

随后：
Day 3 HumanTask与人工恢复

再后：
企业差异配置
→ Core/Cell有限拆分
→ 第一个真实云仓Connector
→ Browser Service正式集成
→ 抖音合作单Cell
```

禁止当前阶段提前建设：

* Agent Runtime；
* Designer Agent；
* 临时容器集群；
* 通用DAG；
* 大型Wiki/图数据库；
* 投流自动化；
* 自动个人微信。

\---

## 3\. 当前权威文件

### 治理

* `governance/YC\_持续文档治理与Agent交接协议\_v2.0.md`

### 当前状态与上下文

* `context/YC\_当前状态快照\_2026-08-06.md`
* `context/YC\_新战略顾问接管上下文\_v1.0.md`

### 战略

* `strategy/YC\_战略规划\_商业计划书\_v3.0.md`
* `strategy/YC\_战略与架构增量决策\_稳定内核与业务扩展\_v1.0.md`

### 产品与架构

* `product/YC\_KOC合作经营中枢\_产品技术方案\_v2.1.md`
* `architecture/YC\_Browser\_Service集成规范\_v1.0.md`
* `open-questions/YC\_关键开放问题与决策门\_v1.0.md`

### 当前执行

* `execution/current/YC\_Codex\_Day2\_1语义清理与Release冻结\_执行方案\_v1.0.md`
* `execution/current/YC\_Codex\_Day2可靠性闭环\_验收意见\_v1.0.md`
* `execution/current/YC\_Codex\_Day3\_HumanTask实施任务书\_v2.0.md`

其中只有Day 2.1是当前可执行任务；Day 3是已批准的下一阶段任务。

### 销售

* `sales/YC\_产品一页纸\_KOC合作经营中枢\_v1.0.md`
* `sales/YC\_5分钟产品演示脚本\_v1.0.md`
* `sales/YC\_企业采购者展示话术\_v2.0.md`
* `sales/YC\_30天付费试点方案\_v1.0.md`
* `sales/YC企业采购者展示PPT\_v2.0.pptx`

销售材料当前仍属于“真实Connector前版本”；第一个真实云仓Cell完成后统一升级。

\---

## 4\. 已冻结工程基线

### Day 1

* Git Tag：`day1-p0`
* Commit：`b78193c`
* 状态：`VALIDATED\_FOR\_FAKE`

工程Release以`yc/docs/releases/day1-p0/`为准。

### Day 2

* 代码已完成；
* Day 2.1未完成前暂不创建最终Tag；
* 目标Tag：`day2-reliability`
* 状态：`IN\_IMPLEMENTATION`

\---

## 5\. 应归档的旧执行文件

以下文件移入`execution/history/`并标记`SUPERSEDED`：

* `YC\_Codex\_Day1\_P0落地方案\_v3.0.md`
* `YC\_Codex\_明日启动任务书\_Day1\_v2.0.md`
* `YC\_Codex\_Week1实施计划\_v1.0.md`
* `YC\_Codex\_下一阶段行动书\_Day2至Week2\_v1.0.md`

`YC\_Codex\_Day2可靠性闭环\_实施记录\_v1.0.md`保留为历史实施记录，不作为当前行动书。

\---

## 6\. 应归档或合并的治理文件

以下两个v1.0文件已由v2.0合并替代：

* `governance/YC\_持续文档治理与Agent交接协议\_v1.0.md`
* `governance/YC\_文档持续沉淀与上下文治理规则\_v1.0.md`

移动到：

```text
archive/governance/
```

并在Front Matter加入：

```yaml
status: SUPERSEDED
superseded\_by: YC\_持续文档治理与Agent交接协议\_v2.0.md
```

\---

## 7\. 阅读顺序

新战略顾问：

1. 本文件；
2. 当前状态快照；
3. 新战略顾问接管上下文；
4. 战略规划；
5. 架构增量决策；
6. 产品技术方案；
7. 当前执行文件；
8. 开放问题。

Codex：

1. 本文件；
2. 当前状态快照；
3. 产品技术方案相关章节；
4. 当前唯一行动书；
5. OPEN\_QUESTIONS；
6. 工程Release与README。

\---

## 8\. 文档目录建议

```text
docs-gpt/
├─ README.md
├─ governance/
├─ context/
├─ strategy/
├─ product/
├─ architecture/
├─ adr/
├─ execution/
│  ├─ current/
│  └─ history/
├─ open-questions/
├─ sales/
└─ archive/
```

\---

## 9\. 更新触发

完成以下任一事件，必须更新本文件：

* `day2-reliability`冻结；
* Day 3验收；
* 第一个真实云仓写入；
* Browser Service正式接入；
* 第一家外部付费试点；
* 产品定位或技术基线变化。

