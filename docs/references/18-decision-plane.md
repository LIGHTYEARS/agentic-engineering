# 决策层 Decision Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 18:47:26

---

## Prompt

继续给出下一个 plane 的方案

---

## Response

搜索决策平面和多智能体评估文献

Demystifying evals for AI agents
Engineering | Journal | ScienceDirect.com by Elsevier
EVALS: Mobile Skills Assessment for Fire Departments and ...
Harness engineering: leveraging Codex in an agent-first ...
Engineering
ORAgentBench: Can LLM Agents Solve Challenging ...
AI Risk Management Framework | NIST
Define success criteria and build evaluations - Claude Console
ENGINEERING | English meaning - Cambridge Dictionary
Engineering.com
Artificial Intelligence Risk Management Framework
What are you visualising for 2026 in AI engineering? How ...
MIT School of Engineering
Petri: An open-source auditing tool to accelerate AI safety ...
Engineering Courses In Singapore | MDIS
Playbook - AIRC - NIST AI Resource Center

搜索多个网站和文献资源

思考了 5s

下一个是 Decision Plane（决策平面）。

对单团队，它的定位应是：

基于 Evaluation Plane 提供的统一证据，在满足硬约束的候选中完成价值取舍、风险接受、方案选择、融合或退回，并把决策固化为可追溯、可执行、可复盘的工程承诺。

它不是让 LLM 给候选打一个总分，也不是由 Tech Lead 凭经验“拍板”。它要回答：

哪些候选已经具备决策资格？

哪些候选因硬约束失败必须淘汰？

多个候选各有优势时，如何避免伪精确总分？

哪些判断可由规则完成，哪些必须由人承担？

当前证据足以支持合并、灰度、继续实验还是返回上游？

选择方案后，团队实际接受了哪些风险与未知？

NIST AI RMF 将风险治理、角色责任、人类监督和风险接受视为组织治理的一部分，而不是模型自行完成的判断；Anthropic 的 Agent eval 实践也强调，代码、模型和人工 grader 各自只能覆盖部分判断，最终决策不能被单一 grader 替代。(NIST 文献出版)

一、Decision Plane 与相邻 Plane 的边界

Plane
回答的问题
不负责什么
Search Plane
有哪些实质不同的可行候选？
不决定最终采用哪个
Evaluation Plane
每个候选通过了什么、失败了什么？
不进行最终价值取舍
Decision Plane
接受哪个候选、哪些风险和取舍？
不篡改评测结果
Delivery Plane
如何安全发布并验证生产效果？
不重新定义候选价值
Learning Plane
决策后来是否正确，规则如何改进？
不追溯性地改写当时依据

Decision Plane 处理的是：

$$
\boxed{
证据
+
偏好
+
风险容忍度
+
时机
+
责任
}
$$

Evaluation 输出“事实和测量”；Decision 输出“组织承诺”。

二、建议划分为 9 个职责域

职责域
核心职责
主要产物
1. Decision Readiness
判断证据是否足以进行决策
Readiness Report
2. Feasibility Filtering
按硬约束淘汰不可接受候选
Feasible Set、Rejection Record
3. Trade-off Modeling
展示多个目标之间的取舍
Trade-off Matrix、Pareto Set
4. Preference Elicitation
明确团队当前真正重视什么
Preference Profile
5. Risk Assessment
评估残余风险、可逆性和暴露范围
Risk Register
6. Decision Synthesis
形成选择、融合、延期或返回建议
Decision Proposal
7. Human Authority
明确谁有权接受哪类风险
Approval Matrix
8. Decision Recording
保存依据、反对意见、限制和触发条件
ADR、Decision Manifest
9. Revisit & Expiry
定义何时必须重新评估
Review Trigger、Expiry Policy

三、核心产物：Decision Package

建议每个中高风险需求维护一份轻量 Decision Package。

specs/
└── ORDER-184/
    ├── decision/
    │   ├── readiness.yaml
    │   ├── candidates.yaml
    │   ├── tradeoffs.md
    │   ├── risks.yaml
    │   ├── decision.yaml
    │   └── ADR-ORDER-184.md

完整 Decision Package 至少包含：

内容
作用
Decision question
本次真正需要决定什么
Eligible candidates
哪些候选具备决策资格
Rejected candidates
为什么淘汰
Hard-constraint evidence
防止软目标掩盖失败
Trade-off matrix
展示不可兼得关系
Evidence strength
区分测量和弱推断
Residual uncertainty
明确尚未确认什么
Risk acceptance
谁接受了什么风险
Decision
选择、融合、延迟或返回
Revisit trigger
什么情况发生时重开决策

四、第一阶段：Decision Readiness

不能因为 Evaluation 已经产出报告，就自动认为可以决策。

Readiness 检查

decision_readiness:
  candidate_set_stable: true
  evaluation_versions_consistent: true
  hard_constraints_evaluated: true
  evidence_reproducible: true
  critical_limitations_declared: true
  decision_owner_assigned: true
  risk_acceptance_authority_known: true
  unresolved_blockers: []

建议状态

状态
含义
NOT_READY
关键证据、负责人或候选缺失
READY_FOR_TECHNICAL_DECISION
可以做技术方案取舍
READY_FOR_RELEASE_DECISION
证据足以决定是否进入交付
DECISION_BLOCKED
需要业务、架构或风险负责人裁定

以下情况不应继续决策：

某个关键硬约束仍未评测；

候选使用了不同 baseline；

性能测试环境不可比；

关键业务语义尚未确认；

候选依赖有争议的 Reality 事实；

负责接受风险的人未参与；

所有候选都存在同一个未解决的致命问题。

此时应明确返回：

Spec Plane
Reality Plane
Search Plane
或 Evaluation Plane

而不是让 LLM填补证据缺口。

五、第二阶段：先建立 Feasible Set

Decision Plane 的第一项实质动作不是排序，而是过滤。

假设有三个候选：

候选
正确性
兼容性
下游 QPS
延迟
A 串行
Pass
Pass
32
8.4s
B 有界并发
Pass
Pass
168
1.48s
C 异步任务
Pass
Pass
140
提交 90ms

若 Spec 定义：

hard_constraints:
  batch_100_p95: <= 2s
  inventory_qps: <= 180

则 A 必须先淘汰。

所有候选
    ↓ 硬约束过滤
可行候选 B、C
    ↓ 软目标取舍
Decision

不能采用：

A 的实现简单 +30
A 的延迟失败 -20
总分仍然最高

硬约束失败不应被其他优点补偿。

六、拒绝记录必须保留

被淘汰候选不应只标记为“未选”。

rejection:
  candidate: CAND-A
  reason_type: hard_constraint_failure
  violated_requirement: NFR-02
  evidence:
    benchmark_run: batch100-A-07
    measured_p95_ms: 8400
    threshold_ms: 2000
  repair_considered:
    bounded_parallelism_would_change_core_candidate_hypothesis
  disposition:
    reject

这有三个作用：

避免后续成员重复提出同一失败方案；

为 Learning Plane 提供失败资产；

防止决策被事后叙事改写成“当时只是偏好 B”。

七、第三阶段：不要立即使用加权总分

大型工程决策通常是多目标问题：

正确性；

性能；

实现复杂度；

运维负担；

可逆性；

长期演进；

交付时间；

风险；

人力。

简单加权总分有明显风险：

总分 =
性能 × 0.3
+ 复杂度 × 0.2
+ 可维护性 × 0.2
+ 风险 × 0.3

问题包括：

分数尺度往往不可比；

权重看似精确但未必真实；

极差指标可能被其他指标抵消；

轻微改变权重就改变结果；

主观评分容易伪装成数学结论。

多目标软件工程研究通常使用 Pareto 非支配集合展示不可直接比较的方案，并指出在决策偏好尚不充分时，过早压缩成单一加权目标可能误导搜索和选择。(arXiv)

八、使用 Pareto Set 表示不可直接比较的候选

定义：

如果候选 B 在所有重要指标上不差于 A，且至少一项明显更好，则 B 支配 A。

例如：

候选
P95
运维复杂度
可恢复性
回滚
B 有界并发
1.48s
中
中
简单
C 异步任务
90ms 提交
高
高
中等

B 和 C 互不支配：

B 运维更简单；

C 恢复能力和大批量能力更强。

因此它们共同构成 Pareto Set：

$$
P = {B,C}
$$

Decision Plane 的任务不是虚构一个客观唯一最优，而是把真实选择表述为：

是否愿意用更高的运行复杂度，换取更强的恢复和扩展能力？

九、第四阶段：明确 Preference Profile

偏好不应该由 Agent 从上下文中猜。

建议决策前明确当前阶段的偏好：

preference_profile:
  horizon: first_release

  priorities:
    1: correctness
    2: compatibility
    3: rollback_simplicity
    4: delivery_time
    5: latency
    6: future_scale

  constraints:
    new_runtime_service: avoid
    migration: avoid
    oncall_capacity: limited

  accepted_tradeoffs:
    - accept manual retry for rare failures
    - accept batch limit of 100

  not_accepted:
    - new persistent job infrastructure
    - irreversible data migration

偏好需要绑定时间和场景。

当前首版偏好可能是：

简单、可逆、快速上线

六个月后的偏好可能变成：

大规模、可恢复、跨区域

因此不存在脱离时间背景的永久“最佳方案”。

十、区分四类决策依据

类型
示例
可信程度
Measured Fact
B 的 P95 为 1.48s
高，需环境有效
Deterministic Result
API diff 无 breaking change
高
Structured Judgment
B 的运维复杂度低于 C
中，需 rubric
Preference
当前不愿引入新任务系统
组织选择，不是真假命题

Decision Plane 必须防止：

“我们不希望增加队列”

被写成：

“队列方案技术上不可行”

前者是偏好，后者是事实判断。

十一、第五阶段：残余风险评估

候选通过 Evaluation 并不代表无风险。

每个 finalist 都需要 Residual Risk Profile。

residual_risk:
  candidate: CAND-B

  items:
    - id: R1
      description: production downstream latency distribution differs from staging
      likelihood: medium
      impact: medium
      detectability: high
      mitigation:
        - concurrency feature flag
        - canary QPS guardrail
      owner: order-module-owner

    - id: R2
      description: process crash requires client retry
      likelihood: low
      impact: medium
      detectability: high
      accepted_by: product-owner

风险至少考虑：

维度
问题
Likelihood
发生概率大致如何？
Impact
发生后影响多大？
Detectability
能否快速发现？
Reversibility
能否回滚或补偿？
Exposure
会影响多少用户和数据？
Mitigation
是否存在保护措施？
Ownership
谁负责监控和处理？

不建议在没有统计依据时给出伪精确概率，例如 17.4%。

使用：

low / medium / high / unknown

并附理由更合适。

十二、风险与不确定性必须分开

两者不是同一概念。

Risk

已知可能发生的事件：

高并发可能触发库存服务限流。

可以评估和缓解。

Uncertainty

尚不知道系统如何表现：

生产热点租户的批量大小分布未知。

不能直接计算风险，只能：

补充调查；

限制发布；

设计 canary；

保守设定阈值；

明确暂时不支持某些范围。

uncertainty:
  statement: batches above 100 were not evaluated
  disposition:
    - enforce hard limit 100
    - collect rejected batch metrics
    - revisit after 30 days

十三、第六阶段：支持五种决策结果

Decision Plane 不应只有“选 A”。

决策类型
适用情况
SELECT
一个候选明显符合当前目标
SELECT_WITH_CONDITIONS
可以采用，但必须增加约束或灰度条件
FUSE
多个候选有可组合优势
DEFER
当前收益不足以覆盖风险或成本
RETURN
证据、需求或方案不足，返回上游
REJECT_ALL
所有候选不可接受

SELECT

decision:
  type: select
  candidate: CAND-B

SELECT WITH CONDITIONS

decision:
  type: select_with_conditions
  candidate: CAND-B
  conditions:
    - batch_size_limit: 100
    - concurrency_default: 6
    - canary_required: true
    - rollback_on_inventory_qps: "> 180"

FUSE

融合不等于直接拼接。

decision:
  type: fuse
  sources:
    - CAND-B
    - CAND-C
  selected_properties:
    - B.bounded_concurrency
    - C.persisted_failure_record
  outcome:
    create_new_candidate: CAND-D
  next_plane:
    Search

新融合候选必须重新进入 Search、Evaluation，不能直接发布。

DEFER

decision:
  type: defer
  reason:
    production volume does not justify async infrastructure
  revisit_when:
    - p95 batch size > 80
    - monthly manual recovery incidents >= 3

十四、决策不应该掩盖反对意见

对于中高风险决策，建议记录 dissent。

dissent:
  reviewer: platform-owner
  position:
    prefer asynchronous job model
  reason:
    expected batch growth may make bounded concurrency temporary
  response:
    first-release volume does not justify new operational component
  revisit_trigger:
    batch_p95_size > 80

反对意见的价值是：

暴露未达成共识的假设；

为未来重开决策保存背景；

防止决策记录只留下胜方叙事；

便于判断后来失败是否早有预警。

十五、第七阶段：人类权限与责任边界

决策权应依据风险类型，而不是依据谁最资深。

决策
建议负责人
局部实现方式
Module Owner
跨模块 API
API / Domain Owner
新运行组件
Tech Lead / Platform Owner
安全风险接受
Security Owner
用户行为语义
Product Owner
数据迁移
Data / Database Owner
发布风险
Release Owner
业务优先级
Product / Business Owner

NIST AI RMF 明确要求区分人机配置中的角色与责任，并确保具有管理、受托或法律责任的人员承担治理和监督职责。(NIST 文献出版)

AI 可以做什么

汇总证据；

检查候选是否被支配；

识别冲突；

生成 trade-off matrix；

找出结论与证据不一致；

提出决策选项；

生成反对意见；

草拟 ADR。

AI 不应做什么

自行接受业务或安全风险；

把弱证据写成确定事实；

在缺失偏好时自动选择方案；

修改硬约束使候选通过；

替代有权人签署决策；

隐藏不确定性以形成明确答案。

十六、LLM 在 Decision Plane 中的正确角色

LLM 应当被定位为：

证据综合器、取舍解释器、冲突发现器和决策草拟器。

不是：

最终正确方案的稳定裁决器。

建议架构：

确定性过滤
    ↓
Pareto / 规则分析
    ↓
LLM 综合证据与生成选项
    ↓
独立一致性检查
    ↓
有权人决策

Anthropic 建议在 Agent 评测中组合代码型、模型型与人工 grader，并对 grader 进行校准；这同样意味着 Decision Plane 不应把模型判断当成唯一裁决依据。(Anthropic)

十七、决策一致性检查

在提交人审前，可以由独立规则或 reviewer 检查：

decision_consistency_checks:
  - selected candidate is in feasible set
  - no hard failure is omitted
  - cited evidence exists
  - candidate metrics use comparable baseline
  - preference is not mislabeled as fact
  - all high risks have owner
  - accepted risks identify approver
  - unresolved uncertainty has disposition
  - decision conditions are testable
  - revisit triggers are measurable

例如系统应阻止：

选择 CAND-A，因为实现最简单。

如果 CAND-A 已因硬约束失败被淘汰。

十八、推荐使用分阶段决策

一次大型功能通常不需要直接做“最终生产决策”。

建议分成：

设计决策
→ 实现候选决策
→ 合并决策
→ 灰度决策
→ 全量决策

阶段
决策问题
Design
哪条技术路线值得完整实现？
Implementation
哪个候选值得进入完整评测？
Merge
是否具备进入主分支的证据？
Canary
是否允许接触少量真实流量？
Full rollout
是否满足全量发布条件？

这样可以避免一个早期决策承担过多未知。

十九、Decision Manifest 示例

decision_id: DEC-ORDER-184-01
date: 2026-07-16

question:
  which execution model should be used for batch cancellation v1?

inputs:
  spec_version: 3
  reality_version: 4
  evaluation_version: 2

eligible_candidates:
  - CAND-B
  - CAND-C

rejected:
  - candidate: CAND-A
    reason: batch p95 exceeds hard threshold

preference_profile:
  release_stage: v1
  priorities:
    - correctness
    - rollback_simplicity
    - operational_simplicity
    - delivery_time
  avoid:
    - new persistent infrastructure

decision:
  type: select_with_conditions
  candidate: CAND-B

rationale:
  - satisfies all hard constraints
  - lower operational burden than CAND-C
  - supports rollback by feature flag
  - current batch distribution does not justify async job infrastructure

conditions:
  - max_batch_size: 100
  - default_concurrency: 6
  - canary_percent: 1
  - rollback_if_inventory_qps_above: 180

accepted_risks:
  - risk_id: R2
    accepted_by: module-owner

remaining_uncertainty:
  - production hotspot tenant behavior

revisit_when:
  - batch_p95_size > 80
  - manual_retry_incidents >= 3_per_month
  - inventory_throttling_rate > 0.5_percent

approvals:
  module_owner: approved
  product_owner: approved
  release_owner: pending

二十、ADR 应重点记录什么

传统 ADR 常只写：

我们决定使用有界并发。

更好的 ADR 应包含：

当时的问题；

当时可用的候选；

已确认事实；

组织偏好；

淘汰理由；

接受的风险；

条件与适用范围；

重开触发器。

# Decision

首版采用有界并发。

# Why now

当前批量规模较小，团队不希望在首版引入持久化任务基础设施。

# Evidence

- 100 单 P95：1.48s
- 下游峰值：168 QPS
- 回归与幂等测试通过

# Conditions

- 最大批量为 100
- 默认并发为 6
- 必须通过 feature flag 灰度

# Not proven

- 未验证超过 100 单
- 未验证跨区域退款延迟

# Revisit

当批量 P95 超过 80 或人工恢复事件每月超过 3 次时重评异步方案。

二十一、Decision Expiry

决策不是永久事实。

以下变化应使决策过期或要求重新确认：

变化
原因
Spec 核心语义变化
目标不同
Reality 关键事实变化
原决策前提失效
流量规模显著变化
性能与架构取舍变化
下游能力变化
原有限制可能消失
团队运维能力变化
可接受复杂度变化
新安全要求
风险阈值变化
生产事故
原风险估计错误
新候选出现
Pareto 集合变化

decision_expiry:
  invalidated_by:
    - spec_major_change
    - batch_p95_size > 80
    - downstream_qps_limit_change
    - new_regulatory_requirement

二十二、生产反馈不得悄悄改写决策

上线后发现 B 表现不佳，不能直接说：

当时选 B 是错误的。

应区分：

当时证据下的决策是否合理；

当时是否遗漏关键事实；

生产环境是否发生变化；

Evaluation 是否失效；

风险是否被正确记录；

触发器是否及时响应。

Learning Plane 应评价的是：

$$
\text{决策过程质量}
\neq
\text{结果是否幸运}
$$

一个基于充分证据的决策可能遭遇低概率失败；一个证据薄弱的决策也可能偶然成功。

二十三、Decision Plane 的完整流程

读取 Evaluation Pack
        ↓
检查 Decision Readiness
        ↓
硬约束过滤
        ↓
形成 Feasible Set
        ↓
识别支配关系与 Pareto Set
        ↓
明确当前 Preference Profile
        ↓
评估残余风险与不确定性
        ↓
生成决策选项
        ↓
独立一致性检查
        ↓
有权人批准、拒绝或返回
        ↓
记录 ADR、条件和重开触发器
        ↓
进入 Delivery Plane

二十四、批量取消订单示例

Evaluation 输入

候选
硬约束
P95
运维复杂度
恢复能力
回滚
A 串行
延迟失败
8.4s
低
低
简单
B 有界并发
Pass
1.48s
中
中
简单
C 异步任务
Pass
提交 90ms
高
高
中

决策过程

A 因延迟硬约束失败 → 淘汰

B 与 C 互不支配：
B：简单、快速交付、容易回滚
C：恢复能力强、扩展性强

当前偏好：
首版、小批量、运维能力有限

最终：
选择 B，但限制批量和并发，
通过 canary 收集是否需要 C 的真实证据。

这不是证明 B 永远优于 C，而是：

在当前事实、目标和组织偏好下，B 是可逆性更强的阶段性选择。

二十五、仓库内最小实现

不需要开发决策平台。

repo/
├── specs/
│   └── ORDER-184/
│       └── decision/
│           ├── readiness.yaml
│           ├── tradeoffs.md
│           ├── risks.yaml
│           ├── decision.yaml
│           └── ADR-ORDER-184.md
├── docs/
│   └── adr/
├── scripts/
│   ├── check-feasible-set.py
│   ├── validate-decision.py
│   └── decision-expiry.py
└── .github/
    └── workflows/
        ├── decision-readiness.yml
        └── decision-consistency.yml

OpenAI 的 harness engineering 实践强调将规则、验证与项目结构编码进仓库，而不是依赖模型在每次任务中重新猜测流程；对于单团队，仓库内决策文件和 CI 检查通常已足够。(OpenAI)

二十六、可以自动检查什么

decision_checks:
  - all selected candidates passed hard constraints
  - rejected candidates have evidence
  - all cited evaluation artifacts exist
  - preference profile is declared
  - high risks have owners
  - risk acceptance has authorized approver
  - unresolved uncertainties have disposition
  - conditional decision includes measurable conditions
  - fusion creates a new candidate
  - selected candidate commit matches evaluation commit
  - decision references current spec and reality versions
  - revisit triggers are defined

机器不能验证：

团队是否真的愿意承担某项风险；

当前业务价值优先级是否正确；

未来架构方向是否值得；

管理者是否应该接受延期。

这些必须由人负责。

二十七、不要让 Decision Plane 退化成什么

1. 不要让 LLM 输出一个“最佳方案”

没有偏好和风险背景，就没有唯一最佳。

2. 不要使用总分掩盖硬约束失败

先过滤，再取舍。

3. 不要把组织偏好伪装成技术事实

“我们暂时不想维护队列”不是“队列技术不可行”。

4. 不要只保存最终结论

必须保留候选、证据、风险和反对意见。

5. 不要把不确定性自动转成中等风险

未知就是未知，可能需要实验或限制范围。

6. 不要允许实现 Agent 批准自己的方案

至少需要独立 verifier 和有权 owner。

7. 不要为了尽快决定而改变 Evaluation 阈值

阈值变化应返回 Spec Plane。

8. 不要把融合当成免费组合

融合候选必须重新评测。

9. 不要把决策视为永久结论

需要 expiry 和 revisit trigger。

10. 不要以生产结果倒推当时决策理由

要区分过程质量和结果运气。

二十八、分阶段落地

Phase 1：硬约束过滤 + ADR

先建立：

Eligible / Rejected 候选；

简单 trade-off 表；

人工 owner；

ADR；

风险和限制。

Phase 2：风险与条件化决策

增加：

Risk Register；

SELECT_WITH_CONDITIONS；

canary 条件；

回滚阈值；

重开触发器。

Phase 3：Pareto 与一致性检查

增加：

支配关系检查；

Pareto Set；

preference profile；

自动 evidence 引用检查；

dissent 记录。

Phase 4：决策质量反馈

积累数据后分析：

哪类偏好导致长期返工；

哪些风险长期被低估；

哪些证据最能预测生产表现；

哪类决策需要更多人审；

哪些决策条件从未被监控。

二十九、衡量 Decision Plane 是否有效

指标
含义
Decision readiness failure rate
因证据不足被退回的比例
Hard-failure selection rate
错误选中违反硬约束候选的次数
Decision lead time
从 Evaluation 完成到决策完成时间
Evidence traceability
决策依据可追踪比例
Risk ownership coverage
高风险有明确 owner 的比例
Conditional decision compliance
发布是否遵守附加条件
Revisit trigger coverage
关键决策是否定义重开条件
Decision reversal rate
短期内被推翻的比例
Late surprise rate
决策后才发现关键未知的比例
Dissent predictive value
反对意见后来命中真实问题的比例
Selected candidate regret
生产后相对其他候选的后悔程度
Decision documentation cost
形成决策包的人力成本
Human override rate
人类否决 AI 推荐的比例
AI recommendation calibration
AI 推荐与最终证据的一致性
Unowned residual risk
没有责任人的残余风险数量

目标不是追求所有决策永不改变，而是降低：

$$
\text{无依据选择}
+
\text{风险无人承担}
+
\text{限制未被记录}
+
\text{生产后无法解释}
$$

三十、向后续 Plane 输出什么

下游 Plane
获取内容
Delivery Plane
被选候选、发布条件、canary 指标和回滚阈值
Learning Plane
决策依据、反对意见、风险预测和后续结果
Orchestration Plane
附加实现、补验证或融合任务
Search Plane
融合要求、缺失候选或重新探索方向
Evaluation Plane
需要补充的评测和条件验证
Spec Plane
需要改变的目标、约束或偏好
Reality Plane
需要补充验证的未知事实

最终建议

对单团队，Decision Plane 最适合采用：

$$
\boxed{
\text{Readiness Gate}
+
\text{硬约束过滤}
+
\text{Pareto 取舍}
+
\text{显式偏好}
+
\text{残余风险登记}
+
\text{条件化选择}
+
\text{有权人批准}
+
\text{ADR 与重开触发器}
}
$$

默认原则应是：

工具决定哪些候选不合格，证据说明候选之间有什么差异，人决定愿意用什么代价换取什么价值，并对该风险承担责任。

Decision Plane 的最终输出不应是：

“AI 推荐方案 B。”

而应是：

哪些候选通过了硬约束；
哪些候选因何被淘汰；
剩余候选在哪些维度互有取舍；
团队当前采用了什么偏好；
接受了哪些风险和未知；
为什么此时选择该方案；
必须满足哪些发布条件；
什么变化发生时需要重新决策。

它的核心价值是：

把“选一个看起来最好的方案”，转化为一项有证据、有条件、有责任人、允许未来被修正的工程决策。

Sources:

Artificial Intelligence Risk Management Framework

How to Evaluate Solutions in Pareto-based Search-Based Software Engineering? A Critical Review and Methodological Guidance

Demystifying evals for AI agents

Harness engineering: leveraging Codex in an agent-first ...
