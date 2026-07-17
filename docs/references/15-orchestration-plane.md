# 编排层 Orchestration Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 15:52:54

---

## Prompt

继续，给出 Orchestration Plane 的方案

---

## Response

搜索技术文献关于编码代理和任务管理

Harness engineering: leveraging Codex in an agent-first ...
Harness: AI for DevOps, Testing, AppSec, and Cost Optimization
Effective harnesses for long-running agents
Adding repository custom instructions for GitHub Copilot
Microsoft Copilot: Your AI companion
KISS Sorcar: A Stupidly-Simple General-Purpose and ...
Harness Open Source is an end-to-end developer platform ...
Agent Harness Engineering
Best practices – Codex
Create custom subagents - Claude Code Docs
Microsoft Copilot
AI Coding Tools Evolve: Claude Code's Multi-Agent ...
What Is a Copilot and How Does It Work?
Multi-agent Orchestration for Enterprise Coding Agents - Cosine
HARNESS Definition & Meaning
LLM-Based Multi-Agent Orchestration: A Survey of ...

搜索技术文献和多个网站资源

思考了 10s

对于单团队，Orchestration Plane 的定位应是：

把已经确认的 Spec 与 Reality Pack，转换为一组有依赖、有边界、有预算、有验证点、可暂停和可恢复的执行任务，并协调人、Agent、工具和工作区完成交付。

它不是多 Agent 平台，也不是做一个通用工作流引擎。更适合的实现方式是：

用仓库内任务文件描述计划；

用 Git branch/worktree 隔离并行任务；

用现有 Issue、PR 和 CI 表示任务状态；

用一个主协调者管理依赖、上下文和集成；

只在确实可并行、职责独立时启用子 Agent。

当前公开实践也倾向于把可靠性归因于整个 harness，而不是单纯依赖模型能力：需要明确任务边界、持久化状态、隔离工作区、自动测试、阶段检查点和恢复机制。OpenAI 的 harness engineering 强调把仓库结构、规则和验证机制编码进环境；Anthropic 的长任务实践强调增量进展、持久化进度和清晰交接；异步软件工程 Agent 的研究则发现，中央任务分派、隔离工作区和基于测试的集成比无结构的 Agent 协作更可靠。(OpenAI)

一、Orchestration Plane 与其他 Plane 的边界

Plane
回答的问题
不负责什么
Spec Plane
应该交付什么？
不决定具体执行顺序
Reality Plane
当前系统实际上是什么？
不选择实现方案
Orchestration Plane
谁在何时、基于什么上下文、执行哪项任务？
不自行改变业务目标
Search Plane
有哪些可行方案？
不拥有最终任务治理权
Evaluation Plane
每项结果是否满足约束？
不替代执行调度
Decision Plane
选择哪个结果或是否继续探索？
不直接修改任务证据

Orchestration Plane 的核心不是“指挥多个 Agent”，而是管理以下变量：

$$
\boxed{
任务依赖
+
执行状态
+
上下文边界
+
工作区隔离
+
资源预算
+
验证门禁
+
人工介入
}
$$

二、建议划分为 9 个职责域

职责域
核心职责
主要产物
1. Work Decomposition
将需求拆成可执行、可验证的工作单元
Task DAG、Work Packages
2. Topology Selection
选择单 Agent、串行、并行、主从或混合模式
Execution Topology
3. Role Assignment
明确每个任务由谁执行、谁验证、谁批准
Role Matrix、Task Owner
4. Context Packaging
为每个执行单元提供最小充分上下文
Context Pack、Handoff Contract
5. Workspace Isolation
隔离候选、子任务和实验，防止相互污染
Worktree、Branch、Sandbox
6. State & Checkpoint Management
维护任务状态、进度、产物和恢复点
Run State、Checkpoint、Progress Log
7. Dependency & Integration Control
管理先后依赖、接口交接和合并顺序
Dependency Gate、Integration Queue
8. Budget & Exception Control
管理时间、token、工具调用和失败重试
Execution Budget、Escalation Record
9. Human Coordination
在关键决策、冲突和越界位置请求人工介入
Human Gate、Approval Record

三、核心产物：Delivery Execution Plan

每个中等以上需求，在进入正式实施前，建议形成一个轻量的 Delivery Execution Plan。

specs/
└── ORDER-184-batch-cancel/
    ├── spec.md
    ├── acceptance.yaml
    ├── reality/
    │   └── ...
    └── orchestration/
        ├── plan.yaml
        ├── tasks/
        │   ├── T01-investigate-refund.yaml
        │   ├── T02-design-api.yaml
        │   ├── T03-implement-domain.yaml
        │   └── T04-integrate-ui.yaml
        ├── state.yaml
        └── handoffs/

plan.yaml 不应成为复杂 DSL。初期只需描述：

feature_id: ORDER-184
spec_version: 3
reality_version: 4
risk_level: L2

execution_mode: hybrid

objective:
  deliver batch order cancellation with partial success and idempotency

tasks:
  - id: T01
    title: confirm refund idempotency
    type: investigation
    depends_on: []
    executor: repository-analyst
    verifier: module-owner
    output:
      - evidence_item
    blocking: true

  - id: T02
    title: design batch API contract
    type: design
    depends_on: [T01]
    executor: design-agent
    verifier: api-owner
    output:
      - api-contract
      - compatibility-analysis

  - id: T03
    title: implement domain batch orchestration
    type: implementation
    depends_on: [T01, T02]
    executor: implementation-agent
    verifier: domain-test-suite
    workspace: worktree/T03

  - id: T04
    title: implement UI interaction
    type: implementation
    depends_on: [T02]
    executor: implementation-agent
    verifier: ui-test-suite
    workspace: worktree/T04

  - id: T05
    title: integrate candidates
    type: integration
    depends_on: [T03, T04]
    executor: integration-agent
    verifier: full-ci

human_gates:
  - after: T02
    approver: module-owner
  - before: merge
    approver: tech-lead

四、第一阶段：把需求拆成 Work Package

任务拆解的目标不是把工作切得越细越好，而是让每个任务满足四个条件：

输入相对明确；

输出可以保存；

完成状态可以验证；

与其他任务的写入边界尽可能清晰。

一个合格的 Work Package

task:
  id: T03
  objective:
    implement domain-level batch cancellation orchestration

  inputs:
    - spec_version: 3
    - evidence_ids: [EV-001, EV-004, EV-011]
    - api_contract: artifacts/T02/openapi.yaml

  allowed_scope:
    - src/order/cancellation/**
    - tests/order/cancellation/**

  prohibited_scope:
    - src/payment/**
    - migrations/**
    - public-api/**

  deliverables:
    - implementation commit
    - unit tests
    - integration test evidence
    - unresolved issue list

  completion_conditions:
    - existing single-cancel tests pass
    - batch partial-failure scenarios pass
    - no duplicate refund property test passes

  budget:
    max_wall_time: 60m
    max_files_changed: 15
    max_diff_lines: 1000

  escalation:
    - public API change required
    - refund semantics inconsistent with spec
    - migration appears necessary

不合格的任务

完成批量取消功能。

它没有明确：

修改范围；

前置输入；

完成标准；

风险边界；

输出格式；

阻塞条件。

五、任务拆解原则

1. 按可验证产物拆，而不是按“角色想象”拆

不建议机械地建立：

产品 Agent
→ 架构 Agent
→ 编码 Agent
→ 测试 Agent

更应该按产物拆：

确认退款幂等事实
→ 冻结 API contract
→ 实现领域逻辑
→ 构建幂等测试
→ 完成 UI 接入
→ 集成验证

角色只是执行者，产物才是协作接口。

2. 先拆阻塞未知，再拆实现

如果 Reality Plane 仍有一个会改变架构的未知：

退款服务是否原生幂等？

那么它应成为前置阻塞任务，而不是让实现 Agent 自行假设。

3. 按写入边界拆分并行任务

适合并行：

后端领域实现
前端交互实现
性能基线脚本
独立验收测试

不适合并行：

两个 Agent 同时修改同一个核心 service
两个 Agent 同时改变同一个 API schema
一个 Agent 改数据库，另一个按旧 schema 写代码

4. 每个任务只解决一个主要不确定性

任务同时包含太多开放问题时，Agent 很容易在执行过程中自行做未经批准的设计决策。

六、第二阶段：选择执行拓扑

并非所有需求都应该使用多 Agent。

Anthropic 的 agent 设计建议强调，先采用最简单可行模式，只有当任务确实需要并行搜索或独立专业能力时，才增加多 Agent 复杂度；其多 Agent 研究实践也指出，每个子 Agent 必须拥有明确目标、工具、输出格式和任务边界。(Anthropic)

建议支持五种拓扑

拓扑
适用情况
风险
Single Agent
小范围、单模块、低歧义任务
长任务可能上下文衰减
Sequential
任务有严格前后依赖
延迟较高，早期错误会传播
Parallel Independent
子任务写入边界清晰、依赖少
合并冲突、语义不一致
Manager-Worker
任务较大，需要动态拆分和汇总
Manager 成为单点认知瓶颈
Hybrid DAG
部分串行、部分并行、阶段性汇合
编排复杂度最高

选择规则

topology_rules:
  single_agent:
    when:
      - affected_modules <= 1
      - estimated_files <= 8
      - blocking_unknowns == 0
      - candidate_search_not_required

  sequential:
    when:
      - contract_must_be_frozen_before_implementation
      - schema_dependency_is_strict

  parallel:
    when:
      - write_sets_are_disjoint
      - outputs_have_explicit_contracts
      - integration_test_exists

  manager_worker:
    when:
      - investigation_scope_is_large
      - dynamic_follow_up_questions_expected

  hybrid:
    when:
      - multiple_modules
      - partial_parallelism_possible
      - integration_risk_is_high

任务自适应编排研究也表明，拓扑选择本身可能比简单增加 Agent 数量更重要；串行、并行、层级和混合拓扑适用于不同的依赖结构。(arXiv)

七、单团队推荐默认拓扑

对于“存量模块增加新功能”，建议默认使用：

主协调者
   ↓
前置事实任务
   ↓
接口/设计冻结
   ↓
┌──────────────┬──────────────┬──────────────┐
│ 领域实现      │ UI / API 接入 │ 独立测试设计  │
└──────────────┴──────────────┴──────────────┘
   ↓
集成任务
   ↓
统一验证
   ↓
人工评审

即：

中央协调 + 局部并行 + 隔离工作区 + 单一集成入口。

异步软件工程 Agent 的实验发现，无结构的并发容易出现编辑干扰、依赖不同步和集成失败；集中分派、异步执行、隔离工作区和结构化集成能够显著改善结果。(arXiv)

八、第三阶段：角色设计

单团队不需要定义十几个长期 Agent persona。

建议只保留六类逻辑角色，而且角色可以由同一个模型在不同 session 中承担。

角色
主要职责
不能做什么
Coordinator
生成任务图、分配工作、追踪状态、处理阻塞
不直接宣布业务需求已满足
Investigator
解决事实未知、补充 Reality Pack
不在证据不足时自行做业务决策
Designer
生成接口和候选技术方案
不直接修改正式实现分支
Implementer
在限定范围内修改代码
不修改自己的完成标准
Integrator
合并子任务、解决接口冲突
不隐藏不兼容或删除失败测试
Verifier
执行独立检查、生成证据
不通过改实现来“修复”验证结果

人类角色

人类角色
介入位置
Task Owner
需求确认、阻塞问题回答
Module Owner
模块行为和兼容性确认
Tech Lead
拓扑、架构和高风险分支确认
Reviewer
PR 审核和风险接受
Release Owner
上线和回滚决策

Coordinator 不是“更聪明的总 Agent”，而是一个受限状态机：

读取计划
→ 判断可运行任务
→ 分配执行者
→ 收集结构化结果
→ 运行 Gate
→ 更新状态
→ 请求人工或启动下一任务

九、第四阶段：上下文打包

不能把完整仓库、完整 Spec、所有历史日志和所有其他 Agent 输出都交给每个 Agent。

Anthropic 的 context engineering 指出，长时间任务需要主动管理有限上下文，保留高价值信息并压缩或外置历史状态；子 Agent 也适合用于隔离上下文和返回结构化结果。(Anthropic)

每个任务的 Context Pack

context_pack:
  task_id: T03

  objective:
    implement domain batch cancellation

  authoritative_inputs:
    - spec: specs/ORDER-184/spec.md#behavior
    - acceptance: AC1, AC2, AC4
    - invariants: INV1, INV2
    - evidence: EV-001, EV-004
    - api_contract: artifacts/T02/openapi.yaml

  repository_scope:
    primary:
      - src/order/cancellation/**
    reference_only:
      - src/payment/client.ts
      - src/inventory/events.ts

  commands:
    - pnpm test order-cancellation
    - pnpm typecheck

  known_risks:
    - refund is non-transactional
    - inventory rollback is asynchronous

  prohibited_decisions:
    - changing partial-success semantics
    - changing public event schema

  expected_output_schema:
    - commit
    - changed_files
    - tests_run
    - evidence
    - unresolved_questions

上下文分级

级别
内容
Mandatory
当前任务目标、硬约束、相关证据、完成标准
Supporting
相关模块说明、历史决策和代码入口
On-demand
其他服务、历史 PR、详细日志
Excluded
无关模块、敏感信息、其他候选的内部过程

核心原则：

给 Agent 的不是“尽可能多的上下文”，而是“完成当前任务所需的最小充分上下文”。

十、Handoff Contract

Agent 间不能只传递自然语言总结。

每个任务完成后，应输出结构化 handoff：

handoff:
  task_id: T03
  status: completed
  base_commit: a83d91c
  result_commit: c9f120a

  deliverables:
    - src/order/cancellation/batch-service.ts
    - tests/order/cancellation/batch-service.spec.ts

  guarantees:
    - uses existing single-order validation
    - each item returns independent result
    - bounded concurrency is configurable

  assumptions_used:
    - A1: batch size <= 100

  evidence:
    - unit-tests: pass
    - idempotency-test: pass

  interface_changes:
    - BatchCancelResult introduced internally

  unresolved:
    - event emission under partial downstream failure

  integration_instructions:
    - merge after T02 contract commit
    - run full order integration suite

一个可靠 handoff 至少要包含：

$$
结果 + 依据 + 假设 + 接口变化 + 未解决问题 + 集成要求
$$

十一、第五阶段：工作区隔离

每个并行实现任务应使用独立环境：

main/base commit
├── worktree/T03-domain
├── worktree/T04-ui
├── worktree/T05-tests
└── worktree/candidate-B

OpenAI Codex 的并行任务使用独立云沙箱，Codex 应用也通过独立线程和 worktree 支持并行工作；GitHub cloud agent 在独立分支中研究、计划并提交修改。(OpenAI)

隔离单位

场景
隔离方式
独立实现任务
Git worktree + branch
多个候选方案
每个候选独立 worktree
危险命令或依赖实验
容器或临时沙箱
数据库变更
独立临时数据库
性能实验
固定、可重置环境
外部服务测试
mock、stub 或临时测试租户

不允许的模式

多个 Agent 直接共享同一个工作目录
多个 Agent 轮流修改未提交文件
Agent 通过聊天描述未保存的中间状态
集成 Agent 从不明确的本地状态开始合并

十二、第六阶段：任务状态机

任务状态不能只用“进行中”和“完成”。

建议使用：

PLANNED
  ↓
READY
  ↓
RUNNING
  ├─→ BLOCKED
  ├─→ NEEDS_HUMAN
  ├─→ FAILED_RETRYABLE
  ├─→ FAILED_TERMINAL
  ↓
DELIVERED
  ↓
VERIFIED
  ↓
INTEGRATED

状态定义

状态
含义
PLANNED
已创建但前置条件未满足
READY
所有依赖和输入已就绪
RUNNING
正在执行
BLOCKED
因事实、依赖或环境缺失暂停
NEEDS_HUMAN
需要业务、架构或权限决策
FAILED_RETRYABLE
可在调整上下文或环境后重试
FAILED_TERMINAL
当前方案不可继续
DELIVERED
已输出产物，但尚未独立验证
VERIFIED
任务级验证已通过
INTEGRATED
已合并进统一候选

重要区别：

Agent 声称完成
≠ DELIVERED

产物存在
≠ VERIFIED

子任务通过
≠ 系统已集成

十三、Checkpoint 与长任务恢复

长任务不能依赖一个持续对话保存全部状态。

Anthropic 的长任务实践建议保存明确的进度文件、Git 提交和测试状态，使后续 session 能从可验证状态继续，而不是猜测前一个 Agent 做了什么。(Anthropic)

何时建立 checkpoint

完成调查结论；

接口 contract 冻结；

一个子任务首次通过测试；

上下文即将压缩；

达到预算的一半；

准备请求人工确认；

失败但已有可复用进展；

准备跨 Agent handoff。

Checkpoint 内容

checkpoint:
  run_id: ORCH-184
  sequence: 7
  timestamp: 2026-07-16T10:42:00Z

  completed_tasks:
    - T01
    - T02

  running_tasks:
    - T03
    - T04

  blocked_tasks:
    - T05

  accepted_artifacts:
    - artifacts/T02/openapi.yaml

  active_commits:
    T03: c9f120a
    T04: b183dea

  decisions:
    - partial_success=true

  remaining_risks:
    - event compatibility under retry

  next_actions:
    - complete T03 property tests
    - validate UI against T02 contract

十四、第七阶段：依赖管理

任务依赖不仅是“先完成 A，再开始 B”。

建议区分五种依赖。

依赖类型
示例
Completion Dependency
T02 完成后才能开始 T03
Artifact Dependency
T03 需要 T02 的 API schema
Evidence Dependency
需要确认退款幂等性
Approval Dependency
需模块 Owner 批准 contract
Environment Dependency
临时数据库或测试环境必须就绪

例如：

dependencies:
  - task: T03
    requires:
      - type: artifact
        source: T02
        item: api-contract
      - type: evidence
        source: T01
        item: refund-idempotency
      - type: approval
        owner: module-owner
        subject: domain-boundary

Coordinator 只能根据确定性状态启动任务，不能凭一句自然语言“看起来差不多了”判断依赖已经满足。

十五、并行执行条件

任务只有在满足以下条件时才适合并行：

parallel_safety:
  write_sets_disjoint: true
  contracts_frozen: true
  shared_schema_read_only: true
  integration_order_defined: true
  individual_verifiers_available: true
  rollback_independent: true

允许并行的例子

后端实现
+
前端基于冻结 contract 开发
+
独立测试 Agent 编写黑盒测试

不允许并行的例子

Agent A 设计 API
+
Agent B 同时按猜测实现前端

或者：

Agent A 修改订单状态模型
+
Agent B 同时为旧状态模型写迁移

十六、第八阶段：集成控制

子任务不能完成后自由互相 merge。

建议有一个单一 Integration Queue：

T02 API contract
    ↓
T03 domain implementation
    ↓
T04 UI implementation
    ↓
T05 independent tests
    ↓
integration candidate

每次集成必须执行

验证 base commit；

检查前置 contract 版本；

以预定顺序应用 commit；

解决文本冲突；

解决语义冲突；

运行受影响测试；

运行跨模块测试；

保存集成 checkpoint。

文本冲突与语义冲突

文本无冲突不代表可安全合并。

例如：

后端把字段命名为 failedItems
前端按 failures 读取

可能没有 Git 冲突，但存在 contract 冲突。

所以 Integration Gate 应包括：

检查
工具
Git conflict
Git
API contract
schema diff / contract test
类型一致性
typecheck
数据库兼容
migration check
事件兼容
schema registry / consumer test
行为一致性
integration test
不变量
property / scenario test

十七、不要让 Integration Agent 随意“修好一切”

Integration Agent 应受严格限制：

integration_policy:
  allowed:
    - resolve mechanical conflicts
    - update imports
    - align generated types
    - apply approved interface contract

  requires_approval:
    - change business behavior
    - modify frozen API contract
    - remove failing test
    - weaken invariant
    - introduce new dependency
    - discard a subtask result

如果集成需要改变业务语义，应返回：

NEEDS_DECISION

而不是让 Integrator 自行决定。

十八、第九阶段：预算管理

Agent 的执行预算应属于任务契约，而不是模型自由使用。

run_budget:
  total:
    max_wall_time: 4h
    max_model_tokens: 2_000_000
    max_ci_minutes: 120
    max_parallel_agents: 3

  per_task:
    max_turns: 30
    max_tool_calls: 80
    max_retries: 2
    max_files_changed: 20

  stop_conditions:
    - repeated_same_failure >= 3
    - no_new_evidence_after_turns >= 8
    - diff_exceeds_scope
    - spec_conflict_detected

预算不是只控制成本

它还用于防止：

无边界探索；

无限 debug；

在错误方向上持续修改；

生成过多难以评审的代码；

多 Agent 互相放大工作量；

CI 被重复触发。

预算追加

Agent 达到预算时，不应自动继续，而应输出：

budget_extension_request:
  task: T03
  progress: 80%
  reason: integration test exposes undocumented event behavior
  additional_budget:
    wall_time: 30m
    tool_calls: 20
  expected_result:
    confirm event retry semantics
  alternative:
    stop and escalate to event owner

十九、失败与重试策略

不能简单地对失败任务“换一个 Agent 再跑一遍”。

先判断失败类型：

失败类型
处理方式
Tool Failure
修复工具或环境后重试
Context Failure
补充或缩减 Context Pack
Spec Ambiguity
回到 Spec Plane
Reality Gap
回到 Reality Plane
Design Failure
启动新候选
Implementation Failure
在现有方案内修复
Integration Failure
重新定义 contract 或合并顺序
Budget Exhaustion
人工决定追加或终止
Policy Violation
终止并进入 Governance 复核

重试必须改变条件

相同模型
+
相同上下文
+
相同工具
+
相同目标
=
大概率重复相同失败

一个合格重试至少改变一项：

输入证据；

任务边界；

实现候选；

执行者；

工具；

验证反馈；

工作区状态。

二十、人工 Gate

单团队不应在每个小动作上人工审批，但以下位置建议保留人工 Gate。

Gate
谁审批
目的
Plan Gate
Tech Lead
确认任务拆解和高风险路径
Contract Gate
Module/API Owner
冻结跨模块接口
Risk Escalation Gate
对应 Owner
处理新发现的风险
Candidate Gate
Tech Lead
决定继续哪个方案
Integration Gate
Module Owner
确认语义集成
Merge Gate
Reviewer
最终代码接受
Release Gate
Release Owner
承担生产风险

不需要人工 Gate 的情况

读取仓库；

运行已批准测试；

独立 worktree 内修改；

保存 checkpoint；

生成调查报告；

创建 Draft PR；

失败后停止并报告。

核心原则：

人应批准不可逆选择和价值判断，不应审批每一个低风险工具调用。

二十一、Orchestration Readiness Gate

在正式启动执行前，建议检查：

orchestration_readiness:
  spec_ready: true
  reality_sufficient: true
  risk_profile_selected: true
  blocking_unknowns_resolved: true

  task_dag_valid: true
  every_task_has_owner: true
  every_task_has_deliverables: true
  every_task_has_completion_conditions: true
  every_task_has_scope_boundary: true
  every_task_has_budget: true

  parallel_tasks_have_disjoint_write_sets: true
  shared_contracts_are_frozen: true
  integration_order_defined: true
  human_gates_defined: true
  recovery_checkpoint_configured: true

三种状态：

状态
含义
Not Ready
任务边界或关键输入不足
Ready for Exploration
可调研、原型和候选搜索
Ready for Execution
可正式修改和集成

二十二、Orchestration Plane 的完整执行循环

读取 Spec 和 Reality Pack
        ↓
识别阻塞未知与交付产物
        ↓
构建 Task DAG
        ↓
选择执行拓扑
        ↓
为任务生成 Context Pack
        ↓
分配工作区、执行者和预算
        ↓
执行 Ready 任务
        ↓
保存产物与 Handoff
        ↓
运行任务级 Verifier
        ↓
更新状态与 Checkpoint
        ↓
启动下游任务或请求人工
        ↓
统一集成
        ↓
进入 Evaluation Plane

Coordinator 的调度循环可以简化为：

while unfinished_tasks:
    refresh_task_states()

    for task in ready_tasks():
        if within_budget(task) and policy_allows(task):
            dispatch(task)

    collect_results()
    run_task_verifiers()
    create_checkpoint()

    if decision_required():
        pause_and_request_human()

二十三、存量模块新增功能的示例编排

以批量取消订单为例：

顺序
Task
执行模式
产物
Gate
1
确认退款和库存副作用
串行调查
Evidence update
Reality Gate
2
冻结批量 API 与部分成功语义
串行设计
API contract
Owner approval
3A
实现领域批处理逻辑
并行
Domain commit
Unit verifier
3B
编写独立黑盒和幂等测试
并行
Test commit
Test review
3C
实现前端交互
并行
UI commit
UI tests
4
合并后端与测试
串行集成
Backend candidate
Integration tests
5
合并 UI
串行集成
Full candidate
Contract tests
6
性能与下游负载验证
独立执行
Benchmark evidence
Performance Gate
7
完整候选审查
人机协作
Draft PR
Merge Gate

依赖图：

T01 现实确认
   ↓
T02 Contract 冻结
   ├─────────┬─────────┐
   ↓         ↓         ↓
T03 领域实现 T04 测试  T05 UI
   └────┬────┴────┬────┘
        ↓         ↓
      T06 集成
        ↓
      T07 性能验证
        ↓
      T08 PR 审查

二十四、仓库内最小实现

不需要先开发 orchestrator 服务。

可以使用：

repo/
├── AGENTS.md
├── specs/
│   └── ORDER-184/
│       └── orchestration/
│           ├── plan.yaml
│           ├── state.yaml
│           ├── tasks/
│           ├── handoffs/
│           └── checkpoints/
├── scripts/
│   ├── task-ready.py
│   ├── create-worktree.sh
│   ├── validate-handoff.py
│   ├── update-task-state.py
│   └── integrate-task.sh
└── .github/
    └── workflows/
        ├── task-verify.yml
        ├── integration-gate.yml
        └── orchestration-check.yml

GitHub 和 Codex 都支持通过仓库指令编码项目结构、构建命令和工作规范；Codex 也支持生成专用 subagent 并汇总其结果。对单团队而言，这些现成功能加少量脚本通常足以构成初始 Orchestration Plane。(GitHub Docs)

二十五、自动检查项

orchestration-check.yml 可以检查：

checks:
  - all task dependencies exist
  - task graph has no cycle
  - each implementation task has a verifier
  - each task has allowed and prohibited scope
  - parallel tasks do not declare overlapping write sets
  - task spec version matches current spec
  - task reality version matches current reality pack
  - every delivered task has a commit
  - every completed task has a handoff
  - every integrated task was previously verified
  - unresolved blocking issues prevent downstream execution
  - budgets are not exceeded

机器可以验证的是计划结构和状态一致性，不是“拆解本身一定正确”。

二十六、不要让 Orchestration Plane 退化成什么

1. 不要把“更多 Agent”当成更强编排

Agent 数量增加会带来上下文、合并、成本和协调负担。

2. 不要把任务拆成大量微任务

过细拆解会导致 handoff 成本高于执行收益。

3. 不要让 Coordinator 同时承担所有实现

否则所谓多 Agent 只是一个 Agent 的长上下文分支。

4. 不要让 Agent 通过自由文本同步状态

状态必须由 artifact、commit 和 verifier 表示。

5. 不要共享未提交工作区

每个可并行任务都应有清晰 base 和独立 branch。

6. 不要自动合并所有成功子任务

局部通过不等于全局兼容。

7. 不要把集成冲突只当 Git 冲突

更危险的是接口、数据和行为语义冲突。

8. 不要长期保存固定 Agent 拓扑

模型和任务能力会变化，编排假设也会过期。Anthropic 在 managed agents 的实践中明确提醒，harness 中关于模型局限的假设会随模型改进而失效，需要持续重新评估。(Anthropic)

二十七、分阶段落地

Phase 1：单 Agent + 显式计划

先建立：

plan.yaml；

Work Package；

Context Pack；

Task State；

Checkpoint；

人工 Gate。

此时不启用多 Agent。

Phase 2：受控并行

增加：

Git worktree；

独立任务分支；

2-3 个并行任务；

Handoff Contract；

Integration Queue。

Phase 3：动态调度

增加：

自动判断 READY 任务；

失败分类；

预算控制；

自动重试部分工具故障；

阻塞自动上报。

Phase 4：自适应编排

在积累足够历史数据后，再允许 Coordinator 根据：

任务规模；

历史成功率；

冲突概率；

评审负担；

Agent 成本；

选择单 Agent、串行或并行模式。

二十八、衡量 Orchestration Plane 是否有效

指标
含义
Plan completion rate
计划任务完整交付比例
Task first-pass success
子任务首次通过 verifier 的比例
Dependency violation rate
未满足前置依赖即执行的次数
Handoff defect rate
因交接不完整导致的返工
Parallel efficiency
并行执行相对串行节省的时间
Merge conflict rate
文本冲突比例
Semantic integration failure
接口或行为冲突比例
Context rework rate
因上下文缺失导致的重跑
Checkpoint recovery success
中断后成功恢复的比例
Human intervention rate
每个任务所需人工介入次数
Budget overrun rate
超出预算任务比例
Abandoned work ratio
最终未被使用的 Agent 产物比例
Time to verified integration
从开始到形成可信集成候选的时间
Cost per integrated change
每个被集成变更的总成本

重点不是最大化并行度，而是优化：

$$
\text{可信集成吞吐量}

\frac{\text{通过验证并成功集成的变更}}
{\text{时间 + Agent 成本 + 人工协调成本}}
$$

二十九、Orchestration Plane 向后续 Plane 输出什么

下游 Plane
获取内容
Search Plane
候选任务边界、并行预算、隔离环境
Evaluation Plane
每项任务的完成条件、验证顺序和候选 commit
Decision Plane
候选状态、失败原因、成本和残余风险
Delivery Plane
集成 commit、发布依赖、执行证据
Learning Plane
拆解质量、失败拓扑、冲突模式和预算数据
Governance Plane
越界动作、异常执行和审批记录

最终建议

对单团队，Orchestration Plane 最适合采用：

$$
\boxed{
\text{Task DAG}
+
\text{最小 Context Pack}
+
\text{中央协调}
+
\text{隔离 Worktree}
+
\text{结构化 Handoff}
+
\text{任务级 Verifier}
+
\text{Checkpoint}
+
\text{预算与人工 Gate}
}
$$

默认策略应是：

先单 Agent、再局部并行；先冻结共享 contract、再拆任务；先独立验证、再进入统一集成。

Orchestration Plane 的最终输出不应是：

“多个 Agent 已经完成了任务。”

而应是：

哪些任务基于哪个 Spec 和 Reality 版本被执行；
每个任务使用了哪些上下文、权限和预算；
产物保存在哪个 commit；
哪些检查已经通过；
哪些假设仍然存在；
各产物如何被集成为一个统一、可验证的候选。

它的核心价值是：

把一次不透明的长时间 Agent 行为，转换成一组边界清晰、状态持久、证据可查、失败可恢复的工程任务。

Sources:

Harness engineering: leveraging Codex in an agent-first ...

Building Effective AI Agents

AdaptOrch: Task-Adaptive Multi-Agent Orchestration in the Era of LLM Performance Convergence

Adding repository custom instructions for GitHub Copilot
