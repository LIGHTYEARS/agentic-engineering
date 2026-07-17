# 学习层 Learning Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 18:54:36

---

## Prompt

继续给出下一个 plane 的方案

---

## Response

Searching web for current AI learning feedback research

Harness engineering: leveraging Codex in an agent-first ...
Harness: AI for DevOps, Testing, AppSec, and Cost Optimization
2026
Demystifying evals for AI agents
Better tools made Copilot code review worse. Here's how ...
AI Risk Management Framework | NIST
Build an Agent Improvement Loop with Traces, Evals, and ...
Public holidays
Artificial Intelligence Risk Management Framework (AI RMF 1.0)
Singapore public holidays 2026
AI RMF Core - AIRC - NIST AI Resource Center
AI Singapore: Home
About NDP 2026
June 2026 Calendar – Singapore
2026 Day of Year Calendar

搜索AI学习反馈研究相关网站

思考了 40s

下一个是 Learning Plane（学习与资产平面）。

对单团队，它的定位应是：

把一次需求交付中产生的事实、失败、人工修正、运行结果和决策反馈，转化为可验证、可复用、可回滚的团队工程资产，使下一次交付更快、更可靠，而不是让同类问题反复依赖人的记忆。

它不是训练基础模型，也不是建设企业知识平台，更不是让 Agent 自动修改自己的规则后直接上线。它要回答：

这次交付暴露了哪些可复用知识？

哪些失败是偶发问题，哪些代表流程或系统性缺陷？

哪些经验应该进入 Spec、测试、仓库说明或治理规则？

如何验证一次“改进”没有造成新的退化？

哪些知识已过期、冲突或不再适用？

如何衡量团队是否真的因为 AI 协作而持续进步？

OpenAI 将可靠 Agent 工程描述为环境、规范、验证和反馈循环共同作用的结果，而不是一次提示词优化；其 Agent improvement loop 也以真实执行轨迹、人类或模型反馈、评测集和回归验证构成持续改进闭环。(OpenAI)
Anthropic 同样强调，eval 的价值会随生命周期积累：真实失败应转化为评测案例，避免团队只在生产中被动修复问题；长任务则需要持久化进度和明确 artifact，不能依赖对话记忆。(Anthropic)

一、Learning Plane 与其他 Plane 的边界

Plane
回答的问题
Learning Plane 接收什么
Governance
Agent 可以做什么？
越权事件、审批摩擦、事故记录
Spec
应该交付什么？
需求歧义、遗漏约束、验收争议
Reality
系统实际上是什么？
新发现事实、过期文档、隐藏依赖
Orchestration
如何组织执行？
拆解失败、上下文缺失、交接问题
Search
如何探索方案？
失败假设、有效原型、搜索成本
Evaluation
候选是否满足要求？
漏测场景、grader 误差、flaky tests
Decision
为什么选择这个方案？
风险预测、偏好、反对意见和后悔
Delivery & Runtime
真实环境结果如何？
生产逃逸缺陷、Canary 结果、回滚轨迹
Learning
哪些经验应固化，如何验证并传播？
以上各 Plane 的反馈

Learning Plane 不负责：

直接修改业务需求；

用生产结果倒推当时决策一定错误；

自动接受未经验证的团队经验；

直接更新所有 Agent 指令；

把一次成功模式推广到所有任务；

用知识沉淀替代正式证据。

二、建议划分为 10 个职责域

职责域
核心职责
主要产物
1. Signal Collection
汇总各 Plane 的反馈、失败和人工干预
Learning Inbox
2. Failure Taxonomy
区分模型、上下文、工具、规格、流程和环境问题
Failure Record
3. Root-cause Analysis
找到可干预的直接原因和系统原因
RCA、Causal Map
4. Knowledge Extraction
将具体事件抽象为可复用规则或模式
Learning Candidate
5. Asset Routing
决定知识应进入哪个资产，而非都写进提示词
Asset Change Proposal
6. Eval Expansion
将真实失败转化为回归测试和评测案例
Eval Case、Regression Suite
7. Improvement Validation
验证规则、Skill、流程或工具变更是否有效
Before/After Evaluation
8. Version & Freshness
管理资产版本、适用范围、冲突和过期
Asset Registry
9. Adoption & Propagation
将已批准改进嵌入仓库和工作流程
Updated Asset、Migration Task
10. Impact Measurement
判断学习是否减少了成本和错误
Learning Metrics、Review Report

三、核心产物：Learning Record

每个值得沉淀的问题，应形成一条结构化 Learning Record。

learning:
  id: LR-ORDER-184-07
  source:
    feature: ORDER-184
    release: REL-ORDER-184-01
    incident: INC-2026-041

  observation:
    canary exposed duplicate inventory events during timeout retries

  classification:
    primary: evaluation_gap
    secondary:
      - reality_gap
      - orchestration_context_gap

  root_cause:
    - existing tests modeled response timeout before side effect
    - production failure occurred after side effect but before acknowledgement
    - event delivery semantics were not included in the task context

  generalizable_lesson:
    workflows with remote side effects must test ambiguous completion states

  proposed_assets:
    - add fault-injection scenario to invariant suite
    - update domain rule documentation
    - add side-effect ambiguity question to Reality template
    - update Work Package checklist

  scope:
    applies_to:
      - order cancellation
      - payment and inventory workflows
    does_not_apply_to:
      - pure read operations

  validation:
    status: pending

这条记录的重点不是“发生了什么”，而是：

$$
事件
\rightarrow
原因
\rightarrow
可复用原则
\rightarrow
具体资产变更
\rightarrow
验证结果
$$

四、第一阶段：建立 Learning Inbox

不应要求每个 Agent 或开发者自行判断什么值得沉淀。

各 Plane 只需把候选信号送入统一 Inbox。

信号来源

来源
典型信号
Governance
越权请求、频繁审批、策略绕过
Spec
实施后才发现的需求歧义
Reality
过期文档、隐藏调用链、错误 baseline
Orchestration
任务拆解错误、上下文遗漏、恢复失败
Search
多个候选重复、关键假设未先验证
Evaluation
生产逃逸、测试污染、grader 误判
Decision
被低估风险、偏好未显式表达
Delivery
Canary 回滚、指标与预期不一致
Human Review
大量人工修订、重复 review 意见
Agent Trace
循环失败、无效工具调用、错误假设

learning_inbox_item:
  source_plane: orchestration
  type: handoff_failure
  task: T04
  symptom:
    UI implementation assumed field `failures`,
    while frozen contract defined `failedItems`
  impact:
    integration_rework_hours: 3.5
  evidence:
    - handoff-T02.yaml
    - commit-b183dea
    - integration-log-42

Inbox 只是待分析队列，不是正式知识库。

五、不是所有反馈都值得固化

建议满足以下任一条件时才进入正式学习流程：

条件
说明
重复发生
同类问题出现两次以上
高影响
一次即造成生产事故或安全风险
可泛化
不只适用于一个具体文件
可干预
能通过规则、测试、工具或文档改善
高人工成本
反复消耗 reviewer 或专家时间
高检测价值
可在更早阶段发现
决策信息价值高
改变了候选选择或发布策略

不建议沉淀：

一次性拼写错误；

仅与某个临时分支有关的问题；

无法说明适用边界的个人偏好；

没有证据支持的经验判断；

已被现有规则覆盖的重复文档；

依赖当前模型短期缺陷的过细 workaround。

可以采用简单 Gate：

learning_trigger:
  accept_if_any:
    recurrence_count: ">= 2"
    severity: high
    production_escape: true
    human_rework_hours: ">= 2"
    cross_module_relevance: true

六、第二阶段：建立失败分类体系

如果所有问题都被归为“Agent 做错了”，就无法改进。

建议至少区分以下 10 类。

类型
定义
例子
Spec Failure
需求或验收标准不完整
未定义部分成功
Reality Failure
对现状或运行环境理解错误
文档与生产 flag 不一致
Context Failure
正确信息存在，但没有提供给执行者
漏掉事件重试语义
Decomposition Failure
任务边界或依赖设计错误
接口未冻结就并行前后端
Search Failure
候选空间或假设设计不佳
三个候选实质相同
Implementation Failure
方案正确但代码有局部缺陷
错误处理遗漏
Verification Failure
verifier 不完整或不可信
只测返回值，未测副作用
Decision Failure
权衡、偏好或风险接受不合理
忽略运维容量
Delivery Failure
发布、配置或运行控制问题
发布物与评测物不一致
Governance Failure
权限、审计或审批机制问题
Agent 读取了不必要凭证
Model Capability Gap
当前模型在特定任务上表现不稳定
长上下文跨文件追踪失败
Tool / Infrastructure Failure
沙箱、CI、索引或工具错误
测试环境网络间歇失败

OpenAI 的 Codex Agent 工程岗位描述也明确区分 harness 问题、模型行为、推理或运行时问题以及产品失败；如果不区分故障层，团队很容易针对错误组件做优化。(OpenAI)

七、症状、直接原因和系统原因需要分开

例如：

症状：
Agent 修改了错误的事件字段。

直接原因：
它引用了一份旧 schema 文档。

系统原因：
Reality Pack 没有记录 schema 版本，
Context Pack 允许未版本化文档作为权威来源，
CI 没有 consumer contract test。

如果只修直接原因：

提醒 Agent 下次小心阅读文档。

问题很可能再次发生。

更可靠的 RCA：

root_cause_analysis:
  symptom:
    wrong event field emitted

  immediate_cause:
    agent followed stale documentation

  contributing_causes:
    - document had no version or freshness marker
    - context pack did not mark authoritative schema
    - candidate tests mocked consumers
    - no schema compatibility gate

  systemic_causes:
    - artifact authority rules undefined
    - Reality Plane freshness checks incomplete

  corrective_actions:
    immediate:
      - fix event field
    preventive:
      - add schema diff check
      - mark schema registry as authoritative
      - expire unversioned event docs

八、避免让 LLM 单独完成 Root Cause Analysis

LLM 可以帮助：

汇总轨迹；

提出多个原因假设；

对比相似事件；

识别缺失证据；

生成因果图草稿。

但 RCA 不能只靠它生成一段流畅解释。

应结合：

Git diff；

测试结果；

Agent trace；

运行日志；

时间线；

权限记录；

人工 review；

重现实验。

推荐流程：

事实时间线
→ 候选原因
→ 反事实检查
→ 重现实验
→ 根因确认
→ 资产改进

反事实问题示例：

如果 Context Pack 包含了正确 schema，
该错误是否仍会发生？

如果存在 consumer contract test，
该错误最晚会在哪个阶段被发现？

九、第三阶段：把经验路由到正确资产

学习不等于“更新 AGENTS.md”。

不同问题应进入不同资产。

学习类型
应进入的资产
业务语义
Spec 模板、领域规则
系统现状
Reality Pack、架构文档、runbook
任务拆解模式
Orchestration 模板
候选策略
Search playbook
真实失败场景
Evaluation suite
权限事故
Governance policy
发布异常
Guardrail、rollback runbook
架构选择原因
ADR
Agent 操作技巧
Skill 或工具 wrapper
模型短板
Model routing 或人工 Gate
工具缺陷
Script、CI 或环境配置

例如：

发现“退款成功但响应超时”导致重复副作用

不应只更新提示词：

请注意退款可能成功但响应超时。

更应该：

Spec：
增加 at-most-once 不变量

Reality：
记录退款接口确认语义

Evaluation：
增加 ambiguous completion 故障注入

Orchestration：
将远程副作用语义纳入 Context Pack

Delivery：
增加重复退款运行时告警

一条高价值学习经常需要修改多个 Plane。

十、建立 Asset Change Proposal

Learning Plane 不应直接修改正式资产。

asset_change_proposal:
  id: ACP-184-07
  learning_id: LR-ORDER-184-07

  changes:
    - asset: tests/invariants/remote-side-effects.spec.ts
      type: add_eval_case
      expected_effect:
        detect ambiguous completion retries

    - asset: specs/_template/reality.yaml
      type: add_question
      content:
        "Does the remote side effect expose a stable idempotency contract?"

    - asset: .agent/context-pack-template.yaml
      type: add_required_field
      field: remote_side_effect_semantics

  risk:
    evaluation_runtime_delta: +2m
    agent_context_delta_tokens: +120

  owner: module-owner
  status: proposed

这样可以评估改进本身的成本，而不是无条件累积规则。

十一、第四阶段：将真实失败转为 Eval

这是 Learning Plane 最重要的闭环。

流程应是：

生产或人工发现失败
→ 最小可重现案例
→ 隐藏或独立 Eval Case
→ 验证旧系统能稳定复现失败
→ 实施改进
→ 验证新系统通过
→ 加入持续回归集

Anthropic 将真实失败转为评测案例视为避免“生产中修一个又坏一个”的核心方法；OpenAI 的改进循环也从真实 trace 和反馈出发，将它们转成 eval，再验证改动是否改善整体表现。(Anthropic)

Eval Case 示例

eval_case:
  id: EVAL-REMOTE-SIDE-EFFECT-01

  origin:
    incident: INC-2026-041

  scenario:
    - refund service commits side effect
    - response is lost
    - caller retries same operation

  expected:
    refund_count: 1
    final_order_state: cancelled
    retry_result: compatible

  applicable_modules:
    - order
    - payment

  hidden_from_implementer: true

十二、保留失败样本，而不只保留成功修复

需要保存：

触发输入；

环境；

失败 commit；

失败 trace；

错误输出；

人工修订；

最终修复；

验证证据。

learning/cases/
└── INC-2026-041/
    ├── input.json
    ├── environment.yaml
    ├── failing-commit.txt
    ├── trace.jsonl
    ├── expected.yaml
    ├── fix-commit.txt
    └── notes.md

这样后续可以判断：

新模型是否仍会失败；

新工具是否改善；

新 prompt 是否只是偶然通过；

某条规则是否仍有必要。

十三、第五阶段：改进必须通过 A/B 或回归验证

不能因为某条新 instruction 看起来更合理，就认为系统已改善。

例如新增规则：

修改 API 前必须检查所有消费者。

需要评估：

指标
变更前
变更后
Consumer 漏检率
18%
4%
平均工具调用
42
68
平均任务耗时
31m
45m
无关仓库扫描率
5%
37%

可能发现准确率提高，但成本和噪声也显著上升。

Improvement Evaluation

improvement_evaluation:
  proposal: ACP-184-07

  eval_set:
    positive_cases: 20
    negative_controls: 20
    unrelated_tasks: 30

  metrics:
    target_failure_rate:
      before: 0.22
      after: 0.04

    unrelated_task_regression:
      before: 0.03
      after: 0.08

    median_cost:
      before: 1.0
      after: 1.18

  verdict:
    accept_with_scope_limit

  scope:
    apply_only_to:
      - workflows_with_remote_side_effects

关键原则：

修复目标问题的同时，检查是否损害了无关任务。

十四、避免 Prompt 与规则无限膨胀

每次失败都增加一句 instruction，会导致：

上下文增长；

规则冲突；

Agent 忽略重点；

无关任务受干扰；

维护者不理解规则来源；

旧 workaround 长期残留。

建议建立规则预算：

instruction_budget:
  repository_global:
    max_lines: 200
  path_specific:
    max_lines: 120
  task_context:
    max_tokens: 4000

新增规则前问：

能否用测试代替？

能否用工具强制？

能否只放在相关路径？

能否加入模板而非全局规则？

是否已有相同规则？

当前模型是否已经不再需要它？

OpenAI 的 harness engineering 将仓库可读性、规则和反馈机制视为整体系统，但并不意味着把所有知识堆入单个提示；GitHub 近期关于代码审查的实践也指出，更通用、更强的工具和指令可能反而降低特定任务效果，说明资产应围绕明确任务边界设计。(OpenAI)

十五、知识应分层存放

层级
内容
示例
Global Team Rules
几乎所有任务适用的原则
禁止 Agent 合并自己的 PR
Repository Rules
当前仓库稳定约束
构建、测试、目录边界
Path-specific Rules
模块或目录规则
支付模块幂等约束
Feature Assets
当前需求事实和决策
Spec、Reality、ADR
Eval Assets
可执行知识
测试、基准、故障场景
Runtime Assets
运行与恢复知识
Dashboard、Guardrail、Runbook
Temporary Context
当前任务临时信息
候选假设、实验结果

越具体、越短期的信息，越不应进入全局 instruction。

十六、第六阶段：资产版本和 Provenance

每项学习资产都应回答：

从哪次事件产生？

谁批准？

适用于哪些范围？

何时生效？

如何验证？

什么情况下失效？

asset:
  id: RULE-REMOTE-SIDE-EFFECT-01
  type: path_specific_instruction
  source_learning:
    - LR-ORDER-184-07
  applies_to:
    - src/order/**
    - src/payment/**
  introduced_at: 2026-07-22
  owner: architecture-owner
  validated_by:
    - eval-suite-remote-side-effects-v2
  expiry_review: 2026-10-22

没有来源的“祖传规则”很难判断是否仍必要。

十七、处理知识冲突

可能出现：

旧规则：
所有批处理必须异步。

新证据：
小于 10 项的同步批处理更简单且满足目标。

不要静默覆盖。

knowledge_conflict:
  topic: batch_processing_mode

  existing_asset:
    id: RULE-BATCH-02
    claim: all batch operations must be asynchronous

  new_evidence:
    - DEC-ORDER-184-01
    - runtime-batch-v1

  analysis:
    old rule was created for batches above 10,000 items

  resolution:
    narrow_scope

  updated_rule:
    asynchronous required when:
      - batch_size > 1000
      - operation duration may exceed request timeout

学习的目标不是让规则越来越多，而是让规则越来越准确。

十八、第七阶段：Freshness 与失效管理

不同资产的有效期不同。

资产
重新验证触发器
仓库说明
架构或目录变化
API 规则
schema 版本变化
Agent instruction
模型或工具升级
Benchmark
硬件、数据集或运行时变化
Eval Case
需求语义变化
权限规则
工具或安全边界变化
Runbook
部署方式或基础设施变化
Search Playbook
历史任务分布变化
Decision heuristic
生产规模或团队能力变化

GitHub 也提醒，自定义指令效果会随 Agent 能力演进而变化，应持续验证，而不是假设旧说明永久有效。(The GitHub Blog)

Staleness Check

freshness:
  asset: AGENT-RULE-17
  review_if:
    - model_family_changed
    - toolset_changed
    - no_activation_in_90d
    - related_eval_pass_rate_above_99_percent_without_rule

十九、可以主动删除学习资产

成熟 Learning Plane 必须具备“遗忘”能力。

删除条件：

对应故障已由确定性机制消除；

新工具使 instruction 不再必要；

规则与当前架构冲突；

长期没有触发；

A/B 显示规则无改善；

规则导致明显副作用；

适用模块已被删除；

模型能力已稳定覆盖。

asset_retirement:
  asset: PROMPT-RULE-23
  reason:
    replaced_by_schema_contract_check
  evidence:
    no additional detection benefit across 50 eval cases
  rollback:
    previous_version_preserved

二十、第八阶段：从单次教训升级为团队模式

当多个 Learning Record 指向同一问题时，应聚类。

例如三次问题：

Agent 使用旧 API 文档
Agent 使用旧数据库 schema
Agent 使用旧 feature flag 说明

共同模式可能是：

仓库缺乏权威来源和时效标记。

此时不应新增三条提示，而应建设统一机制：

authoritative source registry；

freshness metadata；

CI 引用校验；

Reality Pack 强制版本字段。

learning_cluster:
  id: LC-STALE-AUTHORITY-01
  members:
    - LR-101
    - LR-184
    - LR-207
  systemic_pattern:
    stale artifacts are treated as authoritative
  systemic_fix:
    introduce source authority and freshness policy

二十一、Agent Skill 的沉淀标准

不是每段成功对话都应该变成 Skill。

适合沉淀为 Skill 的任务通常：

重复发生；

输入输出稳定；

有明确完成标准；

需要固定工具顺序；

有高价值检查清单；

可通过 eval 验证；

不依赖大量临时业务判断。

例如：

生成模块影响分析
收集性能 baseline
检查 API compatibility
创建故障注入场景
验证 release identity

Skill Contract

skill:
  id: collect-runtime-baseline
  version: 2

  inputs:
    - service
    - metric_window
    - environment

  steps:
    - resolve metric sources
    - query control period
    - normalize units
    - detect missing data
    - emit structured baseline

  outputs:
    schema: runtime-baseline-v2

  evals:
    - baseline-correct-source
    - missing-metric-detection
    - unit-normalization

二十二、Learning Plane 不等于让 Agent 自我修改

高风险反模式：

Agent 发现问题
→ Agent 修改自己的系统指令
→ 下一次任务立即使用

这会引入：

无审查策略漂移；

目标投机；

权限扩大；

测试污染；

不可追溯行为变化；

局部修复破坏全局。

正确流程：

发现反馈
→ 提出资产变更
→ 独立评测
→ 人类批准
→ 版本化发布
→ 监控效果
→ 可回滚

NIST AI RMF 要求持续监控、反馈机制和定期复核，同时明确组织责任，而不是让系统未经监督地自我改变。(NIST出版物)

二十三、人与 AI 的职责边界

活动
AI 负责
人负责
收集信号
汇总 trace、review 和运行结果
补充业务背景
问题分类
提出 failure taxonomy
确认高影响分类
RCA
生成原因假设、时间线和反事实问题
确认根因与责任边界
知识抽取
提炼可泛化模式
判断是否值得组织化
资产路由
建议改测试、规则、Skill 或文档
批准正式资产变更
Eval 生成
将失败转成测试案例
审核期望行为
改进验证
执行 A/B 和回归评测
接受成本与副作用
版本治理
检查过期和冲突
决定淘汰或收窄规则
推广
自动生成 PR 和迁移任务
确认适用范围
影响评估
汇总长期指标
调整团队策略

核心原则：

AI 可以提炼和验证学习候选，但不能单方面决定团队以后必须如何工作。

二十四、Learning Readiness Gate

某项学习进入正式资产前，应检查：

learning_readiness:
  source_evidence_exists: true
  failure_reproducible: true
  root_cause_supported: true
  lesson_generalizable: true
  applicability_scope_defined: true
  target_asset_identified: true
  regression_eval_available: true
  negative_controls_available: true
  owner_assigned: true
  rollback_available: true

状态建议：

状态
含义
CAPTURED
已记录，但尚未分析
ANALYZED
已完成分类和 RCA
PROPOSED
已提出资产变更
VALIDATING
正在运行改进评测
ACCEPTED
已批准进入正式资产
DEPLOYED
已在流程中生效
MONITORING
观察真实效果
REJECTED
无充分收益或不可泛化
RETIRED
已过期或被替代

二十五、完整学习闭环

交付过程与生产反馈
        ↓
进入 Learning Inbox
        ↓
分类：一次性还是系统性
        ↓
构建事实时间线和 RCA
        ↓
提炼可泛化学习候选
        ↓
选择正确资产承载方式
        ↓
生成 Eval 与负向对照
        ↓
验证改进效果和副作用
        ↓
人工批准
        ↓
版本化发布资产
        ↓
监控后续任务表现
        ↓
保留、收窄、回滚或淘汰

二十六、批量取消订单的学习示例

生产发现

在库存服务响应丢失时，
批量任务重试导致重复事件。

Learning 分析

classification:
  primary: evaluation_failure
  contributing:
    - reality_gap
    - context_pack_gap

root_cause:
  - tests did not model ambiguous remote completion
  - Reality Pack recorded timeout but not acknowledgement semantics
  - implementation task lacked side-effect contract

资产变更

Plane
变更
Spec
增加“每个库存回滚最多一次”不变量
Reality
增加外部调用确认语义调查项
Orchestration
Context Pack 增加副作用与重试字段
Evaluation
增加响应丢失故障注入
Delivery
增加重复事件运行告警
Governance
无需改变
Learning
新建远程副作用 Skill

验证

旧流程：20 个类似任务中 5 个漏掉该场景
新流程：20 个类似任务中 1 个漏掉
额外平均成本：每任务 +3 分钟
结论：接受，仅应用于远程写副作用流程

二十七、仓库内最小实现

单团队不需要建设独立学习平台。

repo/
├── learning/
│   ├── inbox/
│   ├── records/
│   ├── clusters/
│   ├── proposals/
│   └── retired/
├── evals/
│   ├── regressions/
│   ├── agent-behavior/
│   └── production-escapes/
├── .agent/
│   ├── skills/
│   ├── instructions/
│   └── asset-registry.yaml
├── docs/
│   ├── adr/
│   ├── incidents/
│   └── domain-rules/
├── scripts/
│   ├── create-learning-record.py
│   ├── validate-learning.py
│   ├── run-improvement-eval.py
│   ├── check-asset-freshness.py
│   └── retire-asset.py
└── .github/
    └── workflows/
        ├── learning-proposal.yml
        ├── improvement-eval.yml
        └── asset-freshness.yml

二十八、asset-registry.yaml 示例

assets:
  - id: SKILL-BASELINE-02
    type: agent_skill
    path: .agent/skills/collect-runtime-baseline.md
    owner: platform-engineer
    source_learning:
      - LR-104
      - LR-128
    applies_to:
      - backend-services
    validated_by:
      - EVAL-BASELINE-SUITE-V3
    introduced_at: 2026-06-12
    review_at: 2026-09-12

  - id: RULE-REMOTE-SIDE-EFFECT-01
    type: path_instruction
    path: src/payment/AGENTS.md
    source_learning:
      - LR-ORDER-184-07
    status: active

二十九、可以自动检查什么

learning_checks:
  - every learning record links to evidence
  - root cause is separated from symptom
  - proposed lesson defines applicability scope
  - asset proposal names an owner
  - prompt or instruction changes include evals
  - new eval reproduces the original failure
  - improvement is tested on negative controls
  - asset changes are versioned
  - conflicting rules are not silently overwritten
  - expired assets block automatic reuse
  - retired assets retain provenance
  - deployed improvements have monitoring metrics

不能完全自动判断：

一次经验是否真的可泛化；

某个规则的组织成本是否值得；

哪个团队偏好应成为标准；

某项人工经验是否包含重要隐性背景；

是否应该收窄或扩大适用范围。

三十、不要让 Learning Plane 退化成什么

1. 不要变成文档归档库

保存文件不等于形成学习闭环。

2. 不要把每次失败都写进全局提示词

优先使用测试、工具、模板和局部规则。

3. 不要只记录最终修复

必须保留失败输入、轨迹和原始证据。

4. 不要把人的解释自动标记为根因

需要事实时间线和重现支持。

5. 不要只验证目标案例

必须运行负向对照和无关任务回归。

6. 不要让 Agent 直接修改正式治理规则

资产变更必须评测、审批和版本化。

7. 不要让规则只增不减

需要 freshness、冲突处理和淘汰机制。

8. 不要把模型升级视为自动改善

环境、资源和 harness 变化会影响评测结果；Agent coding eval 的运行基础设施本身也可能带来显著噪声。(Anthropic)

9. 不要把结果成功等同于决策正确

应评估当时证据和过程，而不是只看运气。

10. 不要追求“完全自动自我改进”

团队需要控制知识、权限和工作方式的演化。

三十一、分阶段落地

Phase 1：Learning Record + 生产逃逸回归

先建立：

统一失败分类；

重大问题 Learning Record；

生产逃逸转 Eval；

人工资产变更 PR；

来源与 owner。

Phase 2：跨 Plane 资产路由

增加：

Spec、Reality、Orchestration 等模板变更；

Agent Skill；

path-specific instruction；

asset registry；

freshness review。

Phase 3：改进评测

增加：

before/after eval；

negative controls；

成本与准确率对比；

自动回归；

规则冲突检查。

Phase 4：系统性模式发现

增加：

Learning Record 聚类；

组织级失败趋势；

模型和工具路由优化；

过期资产自动提醒；

基于历史数据调整 Plane Gate。

三十二、衡量 Learning Plane 是否有效

指标
含义
Repeat failure rate
同类已知问题再次发生的比例
Production escape recurrence
同类生产逃逸是否重复
Learning-to-eval conversion
高价值问题转成 Eval 的比例
Learning-to-asset lead time
从发现到正式资产生效的时间
Improvement acceptance rate
提议中真正产生收益的比例
Target failure reduction
改进对目标问题的降低幅度
Unrelated regression rate
改进对其他任务造成的退化
Human correction recurrence
相同人工 review 意见重复出现频率
Asset usage rate
正式资产被实际触发和使用的比例
Stale asset rate
过期但仍被引用的资产比例
Rule conflict rate
指令和规则冲突数量
Instruction growth
上下文规则体积变化
Eval suite growth quality
新增 Eval 中高价值案例比例
Asset retirement rate
被替代或无效资产的淘汰率
Cost per prevented recurrence
防止一次重复问题所需成本
Cross-plane improvement rate
一项学习同时改善多个 Plane 的比例
Decision calibration improvement
风险预测与真实结果的一致性变化
Time to trusted delivery
整体可信交付时间是否下降

Learning Plane 的目标不是积累最多知识，而是：

$$
\boxed{
\frac{\text{被避免的重复错误}
+
\text{减少的人工修正}
+
\text{更早发现的问题}}
{\text{资产维护成本}
+
\text{评测成本}
+
\text{上下文和流程复杂度}}
}
$$

三十三、向各 Plane 回灌什么

Plane
Learning Plane 回灌内容
Governance
新风险动作、权限规则和审计要求
Spec
新不变量、常见歧义和验收模板
Reality
新事实源、时效规则和调查问题
Orchestration
更可靠拆解、Context Pack 和重试策略
Search
有效候选模式、失败假设和搜索触发条件
Evaluation
真实失败场景、grader 校准和回归测试
Decision
风险预测偏差、偏好后果和重开条件
Delivery
新 Guardrail、Canary 指标和恢复流程

最终建议

对单团队，Learning Plane 最适合采用：

$$
\boxed{
\text{统一反馈入口}
+
\text{失败分类与 RCA}
+
\text{经验的适用范围}
+
\text{正确的资产路由}
+
\text{生产失败转 Eval}
+
\text{改进前后对照验证}
+
\text{人工批准}
+
\text{版本与 Provenance}
+
\text{Freshness 和淘汰}
+
\text{跨 Plane 回灌}
}
$$

默认原则应是：

不是把每次经验写进提示词，而是把能机械验证的知识变成测试，把能强制执行的知识变成工具或策略，把稳定领域知识放在代码附近，把仍然依赖判断的内容保留给人。

Learning Plane 的最终输出不应是：

“团队总结了一些经验。”

而应是：

哪个真实事件暴露了什么问题；
问题属于哪个 Plane；
有哪些证据支持根因；
它能泛化到什么范围；
需要修改哪些正式资产；
改进是否经过回归和负向验证；
由谁批准；
何时生效；
如何监控；
什么情况下应收窄、回滚或删除。

它的核心价值是：

把一次性交付中的失败和人工智慧，转化为经过验证、可复用、可淘汰的工程反馈循环，使系统能力沉淀在团队环境中，而不是只存在于某次对话或某个人的记忆里。

Sources:

Harness engineering: leveraging Codex in an agent-first ...

Demystifying evals for AI agents

Unlocking the full power of Copilot code review

Artificial Intelligence Risk Management Framework
