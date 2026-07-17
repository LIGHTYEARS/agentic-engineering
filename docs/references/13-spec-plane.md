# 规约层 Spec Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/15 20:45:01

---

## Prompt

ok，继续给出 2. Spec Plane 的方案

---

## Response

我会继续按“单团队、非平台产品”的前提设计 Spec Plane，重点放在需求语义拆解、事实与假设分离、可执行验收、不变量、开放问题、变更版本化，以及人与 AI 的确认边界。

搜索开放AI、Anthropic、NIST和工程论文

AI Risk Management Framework | NIST
Codex in ChatGPT | AI Coding Agents for Software ...
2026 Agentic Coding Trends Report
Adding repository custom instructions for GitHub Copilot
[PDF] Artificial Intelligence Risk Management Framework (AI RMF 1.0)
Artificial intelligence
The End of Software Engineering: How AI Agents Are ...
Codex | OpenAI Developers
Effective harnesses for long-running agents
Playbook - AIRC - NIST AI Resource Center
陪你探索、學習與創作的AI 聊天助手
Tips and Tricks for using Codex
什麼是人工智慧(AI)？ | 趨勢科技(TW)
Google Gemini

构建详细方案并引用来源

我需要制定一个详细的方案，确保所有网络上的声明都得到引用。基于官方GitHub最佳实践、OpenAI的应用、Anthropic评估和NIST规范。结构可能包括8个责任、产物、工作流程、模板、角色、关键节点、指标和MVP阶段。每个段落都需要引用相关文献。

制定详细方案并引用网络来源

Best practices for using GitHub Copilot to work on tasks - GitHub Enterprise Cloud Docs
Harness engineering: leveraging Codex in an agent-first world | OpenAI
Demystifying evals for AI agents \ Anthropic
[

Govern

AIRC

](https://airc.nist.gov/airmf-resources/playbook/govern/)

Intent Formalization: A Grand Challenge for Reliable Coding in the Age of AI Agents

已思考 51s

对于单团队，Spec Plane 的定位应是：

把模糊的需求表达，转化为人、Agent 和工具都能共同理解、执行和验证的交付契约。

它不是需求管理平台，也不是要求产品经理写形式化语言。更现实的做法是：在现有 Issue、设计文档和仓库中，增加一层轻量、版本化、半结构化的 specification。

GitHub 对 coding agent 的实践建议已经明确指出，任务越清晰、范围越明确、验收标准越完整，Agent 的结果越好；OpenAI 的 harness engineering 也将“人负责指定意图，Agent 负责执行”视为 agent-first 工程的核心。(GitHub Docs)

一、Spec Plane 要解决什么问题

Spec Plane 不只是把需求写得更详细，而是处理六类语义风险。

风险
示例
Spec Plane 的处理方式
需求歧义
“支持批量取消”是否允许部分成功？
显式列出待确认问题和决策
隐含约束
不能重复退款，但需求没写
记录不变量和禁止行为
事实与假设混淆
“现有接口支持幂等”可能只是猜测
区分事实、假设、待验证项
验收不可执行
“性能要好”“体验流畅”
转换为指标、场景或 rubric
实现绑死需求
“必须使用消息队列”
区分目标、约束和建议方案
需求漂移
开发中途规则变化
版本化 spec 和变更记录

核心原则是：

需求说了什么
≠
系统实际是什么
≠
最后应该如何实现
≠
如何证明实现正确

二、建议将 Spec Plane 划分为 8 个职责域

职责域
核心问题
主要产物
1. Intent Capture
用户真正想解决什么问题？
Problem Statement、User Outcome
2. Scope Definition
做什么、不做什么、影响哪些范围？
In Scope、Out of Scope、Affected Areas
3. Fact & Assumption Separation
哪些是事实，哪些是推测？
Facts、Assumptions、Unknowns、Evidence
4. Behavioral Contract
系统在不同场景下应表现如何？
Scenarios、State Transitions、Error Semantics
5. Constraints & Invariants
哪些条件绝不能被破坏？
Hard Constraints、Invariants、Compatibility Rules
6. Quality Attributes
性能、安全、可维护性要求是什么？
SLO、Budgets、Rubrics
7. Acceptance & Traceability
每个需求如何被验证？
Acceptance Criteria、Requirement-to-Check Mapping
8. Change Management
需求如何确认、冻结和变更？
Spec Version、Decision Log、Change Record

三、Spec Plane 的核心产物：Feature Contract

建议每个中等以上需求维护一个 Feature Contract。

它可以是 Issue 中的结构化模板，也可以是仓库内 Markdown/YAML 文件，不需要建立独立产品。

例如：

specs/
└── ORDER-184-batch-cancel/
    ├── spec.md
    ├── acceptance.yaml
    ├── decisions.md
    └── evidence/

推荐文档结构

# Feature: 批量取消订单

## 1. Problem
当前用户只能逐个取消订单，处理大量异常订单耗时过长。

## 2. Intended outcome
允许具备权限的用户批量提交取消请求，并清楚看到每个订单的处理结果。

## 3. In scope
- 一次选择多个订单
- 批量提交取消
- 返回逐项结果
- 支持失败项重试

## 4. Out of scope
- 自动选择待取消订单
- 跨租户批量操作
- 修改现有退款政策

## 5. Confirmed facts
- 现有单笔取消接口存在
- 库存回滚为异步操作
- 下游事件 schema 仅支持单订单

## 6. Assumptions
- 批量数量不会超过 100
- 用户接受部分成功

## 7. Open questions
- 是否必须全部成功或全部失败？
- 是否需要后台异步任务？
- 重试窗口多长？

## 8. Behavioral scenarios
...

## 9. Invariants
...

## 10. Acceptance criteria
...

## 11. Evidence mapping
...

四、第一阶段：Intent Capture

Spec Plane 首先要明确“为什么做”，而不是直接描述 UI 或代码。

不够好的需求

给订单页加一个批量取消按钮。

它把需求直接绑定到某个 UI 方案，却没有说明业务目标。

更好的表达

problem:
  current_state:
    operators must cancel orders one by one

  user_impact:
    handling 100 failed orders takes about 20 minutes

  desired_outcome:
    reduce manual operation time while preserving cancellation safety

  success_signal:
    operators can safely process large batches with clear per-order results

这样 Agent 可以探索：

同步批量接口；

异步任务；

有界并发；

队列；

分批处理。

而不是被“加按钮”提前锁死。

五、第二阶段：Scope Definition

大型仓库中最危险的不是“Agent 少做了一点”，而是它把范围理解得过大。

每份 spec 都应明确：

In Scope

in_scope:
  - batch selection in order list
  - batch cancellation API
  - per-order result display
  - audit log

Out of Scope

out_of_scope:
  - redesign single-order cancellation
  - change refund policy
  - modify inventory service protocol
  - support cross-tenant operations

Non-goals

Non-goal 与 out-of-scope 略有区别，它表达“明确不追求”。

non_goals:
  - maximum theoretical throughput
  - replacing existing cancellation architecture
  - real-time progress streaming in first release

GitHub 对 Agent 任务的建议包括清晰描述问题、完整验收标准以及修改范围；即使 Agent 能自行搜索代码，明确任务边界仍然有助于减少错误扩张。(GitHub Docs)

六、第三阶段：事实、假设、未知项分离

这是整个 Spec Plane 中最重要的部分。

建议所有关键描述都打上 epistemic status：

状态
含义
Confirmed Fact
有代码、数据、文档或负责人确认
Assumption
当前用于推进的假设
Inference
从现有证据推导，但未直接确认
Unknown
尚未确定
Decision
已由有权人确认的选择
Constraint
必须满足的条件

示例：

facts:
  - statement: single-order cancellation publishes OrderCancelled event
    evidence:
      - src/order/cancel-service.ts:184
      - cancel-order.integration.spec.ts

assumptions:
  - statement: maximum batch size is 100
    owner: product-owner
    expires_before: implementation

unknowns:
  - partial_success_semantics
  - retry_window
  - downstream_rate_limit

decisions:
  - statement: partial success is allowed
    decided_by: product-owner
    date: 2026-07-15

Agent 可以辅助发现假设，但不应自行把假设升级为事实。

七、第四阶段：Behavioral Contract

自然语言需求必须进一步转换为行为场景。

建议使用轻量 Given-When-Then，而不是强制引入复杂 BDD 工具。

Scenario: 所有订单均可取消
  Given 用户有批量取消权限
  And 选择了 10 个可取消订单
  When 用户提交批量取消
  Then 返回 10 个成功结果
  And 每个订单只产生一次取消事件

Scenario: 部分订单不可取消
  Given 用户选择了 8 个可取消订单和 2 个已发货订单
  When 用户提交批量取消
  Then 8 个订单取消成功
  And 2 个订单返回状态不允许错误
  And 成功订单不因失败订单而回滚

Scenario: 请求重复提交
  Given 相同幂等键的批量请求已执行成功
  When 客户端再次提交
  Then 不产生新的退款或库存回滚
  And 返回与首次执行兼容的结果

重点不是采用哪种格式，而是覆盖：

happy path；

partial failure；

边界输入；

重试；

并发；

权限；

下游故障；

回滚；

兼容行为。

八、第五阶段：定义不变量

Acceptance criteria 描述“应该出现什么”，不变量描述“无论采用什么实现，都不能被破坏什么”。

例如：

invariants:
  - each order is refunded at most once
  - inventory rollback occurs at most once
  - existing single-order API behavior remains unchanged
  - unauthorized users cannot cancel any order
  - one failed order must not corrupt another order's state
  - audit records must be attributable to the initiating user

不变量通常是最适合跨候选复用的规格，因为它不绑定具体实现。

2026 年关于 intent formalization 的研究特别区分了 specification 和 verification：规格描述代码应该做什么，验证则检查代码是否做到；自然语言规格虽然提高可追溯性，但只有部分关键性质被转成可检查形式时，才能真正形成机械验证闭环。(arXiv)

九、第六阶段：硬约束、软目标和偏好分离

不要把所有要求放在一个平面列表中。

硬约束

违反即不可接受：

hard_constraints:
  correctness:
    - no duplicate refund
    - no unauthorized cancellation

  compatibility:
    - single-order API unchanged
    - existing event consumers continue to work

  resources:
    - batch size <= 100
    - downstream inventory QPS <= 180

  governance:
    - no new runtime dependency without approval

软目标

用于候选比较：

soft_objectives:
  - minimize p95 latency
  - minimize code complexity
  - minimize changed modules
  - maximize rollback simplicity

实现偏好

可以被推翻，但需要理由：

preferences:
  - reuse existing single-order domain logic
  - prefer bounded concurrency
  - avoid introducing a new queue

这样 Agent 不会把“倾向使用现有逻辑”误读成“绝对禁止任何新抽象”。

十、第七阶段：质量属性可测量化

“性能好”“易维护”“安全”都不能直接成为验收条件。

性能

performance:
  baseline:
    single_cancel_p95: 120ms

  target:
    batch_100_p95: <= 2s
    single_cancel_regression: <= 5%
    inventory_qps_peak: <= 180

可靠性

reliability:
  duplicate_side_effect_rate: 0
  retry_success_rate: >= 99%
  partial_failure_data_corruption: 0

可维护性

可维护性不适合完全机器化，但可以通过 rubric 表达：

maintainability_rubric:
  architecture_consistency:
    5: reuses existing domain boundaries
    3: adds one justified abstraction
    1: introduces parallel business logic

  rollback_complexity:
    5: feature flag only
    3: application rollback required
    1: irreversible migration required

Anthropic 的 agent eval 实践建议组合使用代码型、模型型和人工 grader：确定性条件交给测试和静态分析，开放式质量则使用 rubric、pairwise comparison 和人工判断。(Anthropic)

十一、第八阶段：Requirement-to-Evidence Mapping

每一个关键需求都必须回答：

谁来验证？用什么证据？在什么阶段验证？

建议维护一个 traceability matrix。

Requirement
类型
Verifier
阶段
Owner
不重复退款
硬约束
幂等集成测试
CI
Test Designer
现有接口兼容
硬约束
API diff + contract test
CI
Module Owner
100 单 P95 ≤ 2s
硬约束
Benchmark
Staging
Performance Owner
下游 QPS ≤ 180
硬约束
Load simulation
Staging
Service Owner
方案易回滚
软目标
Rubric + 人审
PR Review
Tech Lead
用户容易理解部分失败
软目标
UX review
Preview
Product/Design

如果一项需求没有 verifier，应标记：

UNVERIFIED REQUIREMENT

而不是默认 Agent 会“凭理解做到”。

十二、Spec Readiness Gate

在 Agent 正式进入 Search Plane 之前，应有一个轻量 Gate。

建议分成三种状态：

状态
含义
Draft
仍在探索，不允许正式实现
Ready for Exploration
可以做技术调研和原型，不可提交最终实现
Ready for Implementation
核心行为、硬约束和验收方式已确认

Ready for Implementation 检查表

readiness:
  problem_defined: true
  intended_outcome_defined: true
  in_scope_defined: true
  out_of_scope_defined: true
  key_unknowns_resolved: true
  invariants_defined: true
  hard_constraints_defined: true
  acceptance_criteria_defined: true
  verifier_mapping_defined: true
  human_owner_confirmed: true

不需要要求所有细节都完整。关键是那些会造成架构或业务语义分叉的问题必须先确认。

Anthropic 对长时间 Agent 的实验发现，仅给一个高层目标容易导致 Agent 一次尝试做得过多、跨上下文丢失状态，并迫使后续 session 猜测前面发生了什么；明确、可延续的任务 artifact 是长任务可靠性的基础。(Anthropic)

十三、需求不完整时，Agent 应如何工作

真实需求通常不可能在开始时完全明确。

所以 Spec Plane 不应要求“先写完完美规格再开始”，而应采用 progressive specification。

需求草稿
→ 只读探索
→ 发现约束和未知
→ 补充 Spec
→ 小原型
→ 验证关键假设
→ 冻结核心 Contract
→ 正式实现

建议将问题分为三类：

Blocking Question

不回答就不能实现：

是否允许部分成功？

Design-Influencing Question

可以探索，但最终设计前必须回答：

是否需要支持超过 1000 单的大批量？

Deferrable Question

可以先采用默认值：

按钮文案最终使用什么？

十四、Spec 变更管理

Agent 开始实施后，spec 仍可能变化。

要求每次变更记录：

change:
  id: SPEC-CHANGE-07
  date: 2026-07-16
  changed_by: product-owner

  before:
    partial_success: false

  after:
    partial_success: true

  reason:
    operators need successful orders to proceed independently

  impact:
    - API response model
    - transaction boundary
    - tests
    - UI error handling

  affected_candidates:
    - candidate-A
    - candidate-B

实施 Agent 检测到 spec 版本变化后，应停止并执行 impact analysis，而不是继续沿旧需求生成。

NIST 的治理建议强调，应明确预期用途、监控和验证流程，并建立 change management 与人工监督责任；这类原则同样适用于团队内 Agent 工作流。(NIST AI Resource Center)

十五、人与 AI 的职责边界

活动
AI 负责
人负责
需求拆解
提取能力、场景、歧义
确认业务意图
范围分析
找相关模块和潜在影响
决定真正的 scope
事实收集
搜索代码、测试、文档
确认隐性组织知识
假设识别
列出未证实前提
接受、拒绝或要求验证
场景生成
扩展边界和异常场景
确认业务语义
不变量建议
从现有代码和领域模式中推导
确认是否真是业务规则
验收条件
转换为结构化条目
接受最终验收标准
实现建议
提出候选
Spec Plane 中原则上不锁定最终方案
Spec 批准
不可自行批准
Product Owner / Module Owner
Spec 变更
检测影响并建议更新
有权人批准变更

一句话：

AI 可以起草 specification，但不能单方面定义业务真相。

十六、仓库内最小实现

建议使用以下结构：

repo/
├── AGENTS.md
├── specs/
│   ├── _template/
│   │   ├── spec.md
│   │   ├── acceptance.yaml
│   │   └── decisions.md
│   └── ORDER-184-batch-cancel/
│       ├── spec.md
│       ├── acceptance.yaml
│       ├── decisions.md
│       └── evidence-links.md
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   └── feature-agent.yml
│   └── workflows/
│       └── spec-readiness.yml
└── docs/
    └── domain-rules/

GitHub 已支持 repository-wide 与 path-specific custom instructions；对单团队而言，可以用这类仓库内 instruction 将模块规则和领域约束直接放在代码附近，而不是建设额外平台。(GitHub Docs)

十七、acceptance.yaml 示例

spec_version: 3
feature_id: ORDER-184
status: ready_for_implementation

problem:
  actor: operations_user
  outcome: cancel multiple orders safely

scope:
  in:
    - batch API
    - batch UI
    - audit logging
  out:
    - refund policy changes
    - cross-tenant operation

facts:
  - id: F1
    statement: single cancellation emits one OrderCancelled event
    evidence:
      - src/order/cancel-service.ts:184

assumptions:
  - id: A1
    statement: maximum batch size is 100
    owner: product-owner
    status: accepted

invariants:
  - id: INV1
    statement: each order is refunded at most once
  - id: INV2
    statement: existing single cancellation behavior is unchanged

hard_constraints:
  - id: HC1
    statement: all existing contract tests pass
    verifier: api-contract-ci
  - id: HC2
    statement: inventory QPS must not exceed 180
    verifier: batch-load-test

soft_objectives:
  - id: SO1
    statement: minimize batch P95 latency
  - id: SO2
    statement: minimize implementation complexity

acceptance:
  - id: AC1
    requirement: partial success returns per-order result
    verifier:
      type: integration_test
      id: batch-partial-failure
  - id: AC2
    requirement: retry produces no duplicate side effects
    verifier:
      type: property_test
      id: batch-idempotency

open_questions: []

approvals:
  product_owner: approved
  module_owner: approved

十八、不要把 Spec Plane 变成什么

1. 不要变成超长 PRD

Agent 不需要几十页背景材料，需要的是可定位、可验证的 contract。

2. 不要完全依赖自然语言

关键约束应转成结构化字段、测试或 policy。

3. 不要试图形式化所有内容

只优先形式化：

高风险；

高歧义；

可机械验证；

容易回归；

多候选需要统一比较的部分。

Intent formalization 的研究也强调，不必追求把全部自然语言规格自动转换为完整形式逻辑，而应优先处理最易歧义且高价值的性质。(arXiv)

4. 不要让 spec 混入大量实现细节

除非某种技术选择本身是约束，否则放入 candidate design，而不是 feature spec。

5. 不要让实现 Agent 定义自己的成功标准

验收标准至少需要 Product、Module Owner 或独立 Test Designer 确认。

十九、分阶段落地方案

Phase 1：最小 Spec 模板

先要求中等以上需求具备：

Problem；

Intended Outcome；

In/Out Scope；

Acceptance Criteria；

Open Questions；

Human Owner。

Phase 2：引入事实与不变量

增加：

Facts / Assumptions / Unknowns；

Invariants；

Hard / Soft separation；

Requirement-to-Verifier mapping。

Phase 3：半自动检查

CI 检查：

spec 是否存在；

是否仍有 blocking question；

每个硬约束是否有 verifier；

spec version 与 PR 是否匹配；

高风险模块是否有 owner approval。

Phase 4：Agent 辅助维护

Agent 自动：

从需求生成 spec 草稿；

从仓库发现潜在约束；

找出未覆盖 requirement；

分析 spec 变更对候选和测试的影响；

生成 PR 证据矩阵。

二十、衡量 Spec Plane 是否有效

指标
含义
Spec clarification count
实施前发现并解决多少歧义
Requirement coverage
有 verifier 的关键需求比例
Late requirement change rate
实施后期需求变更比例
Rework caused by misunderstanding
因需求误解导致的返工
Agent clarification precision
Agent 提出的问题中真正有价值的比例
Unverified claim count
PR 中无证据支持的结论数量
Scope creep rate
修改超出已确认范围的比例
Spec-to-code traceability
需求能否追踪到代码和测试
Human approval latency
Spec 确认所需时间
Spec maintenance cost
团队维护 spec 的额外成本

目标不是让文档数量增加，而是降低：

误解成本
+
返工成本
+
验收争议
+
隐性回归

最终建议

对单团队而言，Spec Plane 最适合采用：

$$
\boxed{
\text{轻量 Feature Contract}
+
\text{事实/假设/未知分离}
+
\text{不变量}
+
\text{硬约束与软目标}
+
\text{Requirement-to-Verifier 映射}
+
\text{人类确认 Gate}
+
\text{版本化变更记录}
}
$$

其最终输出不是一篇“更好的需求文档”，而是一份能够被后续 Plane 消费的合同：

下游 Plane
从 Spec Plane 获取什么
Reality Plane
待确认事实、证据缺口、未知项
Orchestration Plane
任务边界、依赖、风险和阶段 Gate
Search Plane
目标、约束和允许探索的空间
Evaluation Plane
不变量、验收标准和 verifier 映射
Decision Plane
硬约束、软目标和业务优先级
Delivery Plane
上线条件、监控指标和回滚触发器

Spec Plane 的核心价值是：

把“大家大概知道要做什么”，转化为“人、Agent 和工具对目标、边界、证据和完成条件拥有同一个可追踪版本”。

Sources:

Best practices for using GitHub Copilot to work on tasks - GitHub Enterprise Cloud Docs

Intent Formalization: A Grand Challenge for Reliable Coding in the Age of AI Agents

Demystifying evals for AI agents \ Anthropic

[

Govern

AIRC

](https://airc.nist.gov/airmf-resources/playbook/govern/)
