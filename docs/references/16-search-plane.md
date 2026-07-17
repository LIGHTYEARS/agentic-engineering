# 搜索层 Search Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 16:05:43

---

## Prompt

继续，给出 search plane 方案

---

## Response

搜索OpenAI、Anthropic和GitHub上的最新研究

Research
Engineering | Journal | ScienceDirect.com by Elsevier
How we built our multi-agent research system
Engineering
Codecademy: Learn to Code - for Free
Effective Strategies for Asynchronous Software Engineering Agents
Engineering.com
Codédex | Start Your Coding Adventure ⋆˙
College of Engineering | University of Illinois Chicago
The Coding Train
Types of engineering explained with examples and career ...
What Is Coding and What Is It Used For
What is Engineering?
Learn to Code — For Free — Coding Courses for Busy People
5 tips to help you get started with coding
MIT School of Engineering
SLEIGHT-Bench: Finding Blind Spots in AI Monitors
Dumb question but what is the difference between coding ...

搜索OpenAI、Anthropic、GitHub等网站的最新研究

思考了 1m 47s

对于单团队，Search Plane 的定位应是：

在 Spec、Reality 和 Orchestration 给定的目标、事实与执行边界内，受控地探索多个可行方案，并把开放式设计问题转换为可比较、可验证的候选实现。

它不是代码搜索服务，也不是让多个 Agent 随机生成若干 PR。它负责回答：

哪些决策存在真实的方案空间？

应探索哪些具有实质差异的候选？

如何避免候选只是同一方案的表面改写？

何时继续探索、淘汰、修复或组合候选？

如何将搜索成本控制在团队可评审范围内？

多 Agent 并行本身不会自动带来更好的方案。公开实践和研究表明，隔离工作区、明确任务目标、集中协调及基于执行结果的集成很重要；无结构并发会产生编辑干扰、依赖失配和难以整合的局部结果。(OpenAI)

一、Search Plane 与 Orchestration Plane 的区别

Plane
核心问题
主要对象
Orchestration Plane
谁在何时执行什么任务？
Task、依赖、工作区、状态、预算
Search Plane
同一个目标有哪些不同可行解？
Candidate、策略、实验、搜索空间
Evaluation Plane
每个候选表现如何、是否满足约束？
测试、指标、证据
Decision Plane
应选择、组合还是放弃哪些候选？
取舍、风险接受、最终决策

Orchestration 管理执行结构；Search 管理解空间。

例如，对“批量取消订单”：

Orchestration：
先冻结接口，再并行实现后端、前端和测试。

Search：
后端到底采用：
A. 同步串行复用单笔接口
B. 有界并发领域编排
C. 异步任务
D. 按批量大小切换同步与异步

二、建议划分为 9 个职责域

职责域
核心职责
主要产物
1. Search Trigger
判断哪些问题值得产生多个候选
Search Brief
2. Search-Space Definition
明确允许变化和禁止变化的维度
Search Space、Frozen Contracts
3. Candidate Strategy Design
规划有实质差异的探索方向
Candidate Portfolio
4. Candidate Generation
在隔离环境中形成设计、原型或完整实现
Candidate Branch、Design Note
5. Diversity Control
防止候选收敛为同一种实现
Diversity Matrix
6. Progressive Screening
用低成本检查逐步淘汰明显无效方案
Screening Results
7. Repair & Refinement
根据验证反馈修复或细化候选
Candidate Revision
8. Composition Search
判断不同候选优势能否安全组合
Fusion Candidate、Compatibility Report
9. Search Termination
根据证据、预算和收益决定何时停止探索
Search Summary、Stop Record

三、并非每个任务都需要 Search Plane

多数小改动只需要一个实现路径。

应启动搜索的情况

信号
示例
存在多个合理架构路径
同步处理还是异步任务
关键目标相互冲突
延迟、复杂度、可回滚性无法同时最优
Reality 中存在高影响不确定性
下游限流边界不明确
方案不可逆或代价较高
数据迁移、公共 API、领域模型变化
团队对方案存在明显分歧
新建服务还是扩展现有模块
局部最优容易误导
benchmark 更快但运维复杂度显著增加
需求开放且没有唯一正确实现
大型重构、性能优化、交互方案

不应启动多候选搜索的情况

文档修正；

明确的局部 Bug；

格式化或机械迁移；

已有唯一标准实现；

简单依赖更新；

搜索成本明显高于错误方案的修复成本。

可以使用简单触发规则：

search_trigger:
  start_search_if:
    architecture_choices: ">= 2"
    or_irreversible_change: true
    or_cross_module_contract_change: true
    or_expected_review_disagreement: high

  default_candidate_count:
    low_risk: 1
    medium_risk: 2
    high_uncertainty: 3

  hard_max_candidates: 4

对单团队而言，通常 2-3 个候选已经足够。候选数量超过四个后，评测、集成和人工评审成本往往迅速上升。

四、核心产物：Search Brief

Search Plane 启动前，应形成一份简短的 Search Brief。

feature_id: ORDER-184
spec_version: 3
reality_version: 4

search_question:
  how should batch cancellation be executed safely?

why_search_is_needed:
  - latency and operational simplicity conflict
  - downstream inventory QPS is constrained
  - batch size distribution has a long tail

frozen:
  - partial success semantics
  - existing single-cancel behavior
  - public response contract
  - no duplicate refund invariant

variable_dimensions:
  execution_model:
    - synchronous
    - bounded_parallel
    - asynchronous
    - hybrid
  transaction_model:
    - per_item
    - batch_coordinator
  retry_model:
    - client_retry
    - server_job_retry

objectives:
  hard:
    - no duplicate side effects
    - existing contract tests pass
    - inventory QPS <= 180
  soft:
    - minimize p95 latency
    - minimize operational complexity
    - maximize rollback simplicity

budget:
  max_candidates: 3
  max_prototype_time_each: 60m
  max_full_implementations: 2

这份 Brief 的作用是防止 Agent 把搜索变成无边界的架构重写。

五、定义搜索空间

搜索空间必须明确区分三类内容。

1. 冻结维度

候选不可自行改变：

frozen_dimensions:
  business_semantics:
    - partial success allowed
    - per-order result required

  compatibility:
    - existing single-cancel API unchanged
    - current event consumers remain compatible

  governance:
    - no production access
    - no new runtime dependency without approval

2. 可探索维度

search_dimensions:
  concurrency:
    - serial
    - bounded_parallel

  processing_mode:
    - synchronous
    - asynchronous
    - adaptive

  internal_abstraction:
    - reuse_application_service
    - introduce_batch_coordinator

  result_storage:
    - request_response_only
    - persisted_job_state

3. 禁止区域

prohibited:
  - rewrite payment service
  - change refund policy
  - introduce distributed transaction
  - replace current event infrastructure

这里的关键是：

搜索自由度不应该体现在“可以改任何东西”，而应该体现在若干被明确授权的设计维度中。

六、候选应围绕“假设”产生，而不是随机产生

每个候选都应有一个明确的核心假设。

例如：

候选
核心假设
预期优势
主要风险
A：同步串行
批量规模小，简单性优先
易实现、易回滚、下游压力低
大批量延迟高
B：有界并发
下游存在可利用的并发余量
延迟较低、架构变化小
限流和并发竞态
C：异步任务
用户可接受最终完成
前端响应快、可处理大批量
运维和状态管理复杂
D：混合策略
批量大小存在明显分层
小批量低延迟，大批量稳定
规则与测试复杂

候选的价值来自它们检验不同假设，而不是来自不同 Agent 写出了不同代码风格。

Candidate Contract

candidate:
  id: CAND-B
  name: bounded-parallel-orchestration

  hypothesis:
    bounded concurrency can satisfy latency without exceeding downstream QPS

  differentiator:
    execution_concurrency

  expected_strengths:
    - lower latency than serial
    - no persistent job infrastructure

  expected_weaknesses:
    - complex retry behavior
    - sensitive to downstream limits

  required_experiments:
    - concurrency sweep
    - duplicate side-effect test
    - downstream throttling simulation

  falsification_condition:
    safe concurrency cannot meet p95 target

falsification_condition 很重要：每个候选都应说明什么证据会证明它不值得继续。

七、候选多样性必须被显式设计

多个 Agent 在相同上下文和相同提示下，常常会产生高度相似的方案。

因此不能只说：

请分别生成三个不同方案。

应主动控制差异维度。

方法一：按架构假设分配

候选 A：优先最小修改
候选 B：优先性能
候选 C：优先可靠性和可恢复性

方法二：按约束偏好分配

候选 A：禁止新增持久化状态
候选 B：允许新增任务表，但禁止新服务
候选 C：允许异步基础设施

方法三：按失败模式分配

候选 A：针对下游限流
候选 B：针对重复请求
候选 C：针对大批量长时间执行

方法四：反常规候选

要求一个 Agent专门挑战当前主流假设：

在不接受“必须使用批处理接口”这一前提的情况下，
寻找能实现用户结果的最小方案。

Anthropic 的多 Agent 研究系统会为子 Agent 提供相互区分的目标和独立上下文，再由主 Agent 汇总；其公开经验同时指出，多 Agent 最适合可并行、可独立调查且拥有明确边界的工作。(Anthropic)

八、建立 Diversity Matrix

候选生成后，先检查它们是否真的不同。

维度
A 串行
B 有界并发
C 异步任务
执行模式
同步
同步
异步
状态持久化
无
无
有
并发控制
1
N
Worker
用户等待
完整等待
完整等待
提交后查询
重试责任
客户端
客户端/服务端
任务系统
回滚复杂度
低
中
中
运维复杂度
低
中
高
最大批量能力
低
中
高

如果两个候选只在类名或局部实现上不同，应合并为一个候选，而不是消耗两套评测预算。

可以设置：

diversity_gate:
  minimum_distinct_dimensions: 2
  reject_if:
    - same_architecture_and_same_failure_model
    - only_naming_or_style_difference
    - one_candidate_is_strictly_incomplete_version_of_another

九、候选生命周期

建议采用以下状态：

PROPOSED
   ↓
PROTOTYPING
   ├─→ REJECTED_EARLY
   ↓
SCREENED
   ↓
IMPLEMENTING
   ├─→ NEEDS_REPAIR
   ↓
EVALUATED
   ├─→ DOMINATED
   ├─→ FUSION_SOURCE
   └─→ FINALIST

状态
含义
PROPOSED
只有假设和方案草图
PROTOTYPING
正在验证最关键的不确定性
REJECTED_EARLY
低成本证据已证明不可行
SCREENED
通过初步门槛
IMPLEMENTING
正在形成可完整评测的实现
NEEDS_REPAIR
基本方向有效，但存在可修复缺陷
EVALUATED
已完成统一评测
DOMINATED
在主要指标上被其他候选完全压制
FUSION_SOURCE
不独立入选，但有可复用优势
FINALIST
进入 Decision Plane

这里不建议直接使用“WINNER”状态，因为最终选择属于 Decision Plane。

十、采用分层搜索漏斗

不要一开始就把所有候选实现到生产级别。

推荐四层漏斗：

方案草图
→ 风险原型
→ 最小完整候选
→ 生产级候选

Level 1：设计筛选

低成本检查：

是否违反硬约束；

是否需要未授权基础设施；

是否依赖错误事实；

是否存在明显不可逆风险；

与已有失败方案是否重复。

产物通常只需要设计说明和调用图。

Level 2：关键假设原型

只实现最不确定的部分：

候选
关键原型
串行
100 单真实延迟
有界并发
并发数与下游 QPS 曲线
异步任务
幂等任务恢复与状态模型
混合
模式切换边界是否稳定

Level 3：最小完整实现

候选必须：

编译；

通过核心行为测试；

支持统一 benchmark；

包含回滚方式；

输出结构化 evidence。

Level 4：生产候选

只对 1-2 个 finalist 完成：

完整测试；

观测埋点；

feature flag；

文档；

灰度与回滚；

运维配置。

这种 successive halving 思路可以避免在明显不合适的方案上投入完整实现成本。

十一、早期筛选由硬约束驱动

候选首先应接受确定性筛选。

early_screen:
  compile: required
  existing_tests: required
  invariant_smoke_tests: required
  public_api_breaking_change: forbidden
  new_runtime_dependency: approval_required
  max_changed_modules: 3
  downstream_qps_estimate: "<= 180"

早期筛选不应使用模糊的总分：

方案 A：82 分
方案 B：79 分

因为一个硬约束失败不应该被其他软指标抵消。

正确顺序是：

硬约束筛选
→ 可行性判断
→ 软目标比较

十二、Search Plane 可以使用快速代理指标，但不能把它们当最终事实

在早期阶段，可以使用较便宜的 proxy：

最终目标
早期 Proxy
生产性能
本地或 staging microbenchmark
运维复杂度
新增组件、配置和运行状态数量
可维护性
新增概念数、修改模块数、重复逻辑
可回滚性
是否有 feature flag、是否涉及迁移
兼容性
schema diff、受影响消费者列表
下游压力
静态调用次数估算、负载模拟

但必须标记限制：

proxy_result:
  claim: candidate B likely meets latency target
  evidence: local benchmark
  confidence: medium
  limitations:
    - no production traffic distribution
    - downstream service mocked

Search Plane 的 proxy 用于决定“是否继续投入”，而不是证明“可以上线”。

十三、修复与重新生成必须区分

候选失败后，应判断是方向失败还是实现缺陷。

情况
处理
违反核心架构约束
淘汰候选
核心假设被实验否定
淘汰候选
局部 Bug
修复原候选
测试遗漏
补测试后重跑
上下文误解
重新生成 Context Pack
实现偏离候选假设
回退到原设计重新实现
性能差但有明显优化路径
给一次有限修复预算
多次修复仍失败
停止继续投入

Repair Budget

repair_policy:
  max_repair_rounds: 2
  continue_if:
    - failure_is_local
    - core_hypothesis_still_supported
    - expected_fix_cost_is_low
  stop_if:
    - same_failure_repeats
    - new_hard_constraint_violation
    - candidate_requires_spec_change

不能让 Agent无限修补一个已经被事实否定的方向。

十四、反例搜索应独立于候选实现

每个候选都应有一个独立的 Challenger，专门寻找：

能破坏不变量的输入；

未覆盖的并发顺序；

边界批量；

重试组合；

下游超时；

非法状态；

环境差异；

兼容性断裂；

候选依赖的未证实假设。

challenge:
  candidate: CAND-B
  target_hypothesis:
    bounded concurrency preserves at-most-once side effects

  adversarial_scenarios:
    - timeout after refund succeeds but before response
    - duplicate batch request with same item set
    - two overlapping batches
    - inventory throttling midway
    - worker process termination

实现 Agent 不应拥有排除这些反例的权力。

十五、组合候选：不是拼接最佳代码片段

有时 A、B、C 都不是最终方案，但各自包含有效优势。

例如：

A 的单笔领域逻辑复用最好；

B 的并发控制最好；

C 的任务状态和恢复模型最好。

不能直接把三者代码合并。应先做可组合性分析：

composition:
  source_candidates:
    - CAND-A
    - CAND-B

  selected_properties:
    - A.reuse_existing_validation
    - B.bounded_concurrency

  compatibility:
    shared_state_model: compatible
    retry_semantics: conflict
    public_contract: compatible

  required_decision:
    choose client-level or server-level idempotency

  outcome:
    create_new_candidate: CAND-D

融合后得到的是一个新候选，必须重新接受完整验证，不能继承源候选的通过状态。

$$
A\ \text{通过} + B\ \text{通过}
\not\Rightarrow
A+B\ \text{通过}
$$

十六、候选隔离与分支结构

每个候选应基于同一个已确认 base commit：

base/spec-v3-reality-v4
├── candidate/A-serial
├── candidate/B-bounded-parallel
├── candidate/C-async-job
└── candidate/D-hybrid

每个候选独立拥有：

worktree；

branch；

sandbox；

测试结果；

benchmark 结果；

Candidate Contract；

修改说明；

未解决问题。

OpenAI 的 Codex 工作流通过独立线程、worktree 或云沙箱支持并行任务；相关异步 SWE 研究也将 branch-and-merge、隔离工作区和基于测试的整合视为可靠协作的核心机制。(OpenAI)

禁止候选互相污染

候选在初始探索阶段不应看到其他候选的实现，以避免过早收敛。

可以分为两个阶段：

独立探索阶段
→ 冻结初始候选
→ 交叉评审阶段
→ 修复或组合

十七、统一候选输出格式

每个候选必须用相同结构提交：

candidate_result:
  id: CAND-B
  base_commit: a83d91c
  result_commit: f19c283

  hypothesis:
    bounded concurrency satisfies p95 and QPS constraints

  architecture:
    execution_mode: synchronous
    concurrency_limit: configurable
    persistent_state: false

  changed_scope:
    modules:
      - order-cancellation
    files: 11
    diff_lines: 624

  evidence:
    compile: pass
    unit_tests: pass
    invariant_tests: pass
    prototype_benchmark:
      p95_ms: 1480
      inventory_peak_qps: 168

  assumptions:
    - batch size <= 100
    - refund API request key is stable

  risks:
    - process crash requires client retry
    - concurrency tuning is environment-dependent

  rollback:
    feature_flag: true
    migration_required: false

  unresolved:
    - behavior when downstream latency exceeds request timeout

统一结构是 Evaluation Plane 公平比较候选的前提。

十八、搜索终止条件

Search Plane 需要明确何时停止，否则候选可以无限增加。

成功终止

满足以下条件之一：

只有一个候选满足全部硬约束；

已有 2-3 个非支配候选，可交给 Decision Plane；

新增候选的边际信息价值很低；

关键设计维度已被覆盖；

候选结果已经稳定复现。

失败终止

所有候选均违反硬约束；

关键 Reality 未知导致所有候选依赖猜测；

Spec 本身存在冲突；

预算耗尽；

无法建立可信 verifier；

实现需要越出治理范围。

Stop Record

search_stop:
  reason: diminishing_returns

  candidates_generated: 3
  candidates_screened_out:
    - CAND-A: latency target failed
    - CAND-C: operational complexity exceeds accepted scope

  finalists:
    - CAND-B

  uncovered_dimensions:
    - batches above 1000 items

  remaining_uncertainty:
    - production downstream latency distribution

  recommendation:
    proceed_to_full_evaluation

十九、Search Plane 不负责最终选择

Search Plane 可以报告：

B 是当前唯一满足硬约束的候选。

但不应在仍存在多个可行候选时直接说：

B 是最佳方案。

它应输出：

候选
硬约束
优势
劣势
证据强度
B 有界并发
全部通过
延迟低、改动适中
并发调优复杂
中高
C 异步任务
全部通过
可恢复、大批量能力强
运维复杂
中
D 混合策略
全部通过
长短批量兼顾
规则与测试复杂
中

最终价值权衡属于 Decision Plane。

二十、人与 AI 的职责边界

活动
AI 负责
人负责
判断是否需要搜索
根据风险和不确定性建议
确认是否值得投入
定义搜索维度
提取可变维度和候选假设
冻结业务语义和架构禁区
生成候选
设计、原型、实现和记录
对关键方向提供约束
控制多样性
检查候选差异和重复
确认差异是否有业务价值
早期筛选
执行确定性检查和 proxy 实验
审批重要淘汰理由
候选修复
根据失败证据进行有限修复
决定是否追加预算
反例搜索
生成边界和攻击场景
确认高风险业务场景
候选组合
分析兼容关系并生成融合候选
批准语义和架构变化
停止搜索
根据预算和边际收益提出建议
决定进入 Evaluation 或返回上游

核心原则：

AI 可以扩展并探索方案空间，但人需要控制什么值得探索、哪些边界不可改变，以及何时搜索已经足够。

二十一、Search Readiness Gate

开始多候选实现前检查：

search_readiness:
  spec_ready: true
  reality_sufficient: true
  orchestration_ready: true

  search_question_explicit: true
  frozen_dimensions_defined: true
  variable_dimensions_defined: true
  prohibited_regions_defined: true

  candidate_hypotheses_distinct: true
  common_baseline_defined: true
  common_verifiers_available: true
  candidate_workspaces_isolated: true

  search_budget_defined: true
  early_stop_conditions_defined: true
  human_owner_confirmed: true

状态建议：

状态
含义
Not Needed
单一路径即可
Ready for Design Search
可以生成设计候选
Ready for Prototype Search
可以进行关键实验
Ready for Implementation Search
可以创建完整实现候选

二十二、完整搜索循环

读取 Spec、Reality 和 Execution Plan
                ↓
判断是否存在有价值的方案空间
                ↓
冻结不可变 contract
                ↓
定义搜索维度和候选假设
                ↓
生成差异化候选组合
                ↓
独立工作区原型
                ↓
低成本硬约束筛选
                ↓
关键假设实验
                ↓
淘汰 / 修复 / 扩展候选
                ↓
形成少量完整候选
                ↓
统一格式输出
                ↓
进入 Evaluation Plane

简化调度逻辑：

candidates = propose_distinct_candidates(search_brief)

for candidate in candidates:
    prototype(candidate)

    if violates_hard_constraint(candidate):
        reject(candidate)
        continue

    test_core_hypothesis(candidate)

    if hypothesis_falsified(candidate):
        reject(candidate)
    elif worth_full_implementation(candidate):
        implement(candidate)

stop_when_budget_exhausted_or_information_gain_low()

二十三、存量模块新增功能示例

以批量取消订单为例：

搜索问题

如何在不破坏单笔取消语义和下游稳定性的情况下，
实现最多 100 个订单的批量取消？

候选组合

候选
方案
需要首先验证的假设
A
串行调用现有单笔逻辑
100 单延迟是否可接受
B
受限并发调用单笔逻辑
并发 8 是否不超过下游上限
C
持久化异步任务
增加运行状态是否值得
D
≤10 同步，>10 异步
切换策略是否增加过多复杂度

搜索漏斗

设计检查：
A、B、C、D

关键原型：
A 测延迟
B 测并发/QPS
C 测任务恢复
D 只建决策模型

筛选结果：
A：100 单 P95 过高 → 淘汰
C：满足功能，但超出首版运维范围 → 暂存
B：满足目标 → 完整实现
D：复杂度收益不足 → 淘汰

进入 Evaluation：
B
C 作为未来扩展参考

二十四、仓库内最小实现

不需要开发通用搜索服务。

repo/
├── specs/
│   └── ORDER-184/
│       ├── search/
│       │   ├── brief.yaml
│       │   ├── portfolio.md
│       │   ├── diversity-matrix.md
│       │   ├── candidates/
│       │   │   ├── A-serial.yaml
│       │   │   ├── B-bounded.yaml
│       │   │   └── C-async.yaml
│       │   ├── experiments/
│       │   └── search-summary.md
│       └── ...
├── scripts/
│   ├── create-candidate-worktree.sh
│   ├── validate-candidate.py
│   ├── compare-candidate-structure.py
│   └── candidate-screen.sh
└── .github/
    └── workflows/
        ├── candidate-screen.yml
        └── candidate-evidence.yml

OpenAI 的 harness engineering 实践强调，应把构建、验证、文档和环境规则编码进仓库，使 Agent 能在可发现、可验证的系统中工作，而不是依赖一次性提示。(OpenAI)

二十五、可以自动检查什么

candidate_checks:
  - candidate references current spec and reality versions
  - candidate has a distinct hypothesis
  - frozen contracts are unchanged
  - prohibited paths are untouched
  - branch is based on approved base commit
  - required prototype experiment exists
  - hard-constraint results are present
  - benchmark environment is recorded
  - assumptions and limitations are declared
  - rollback mechanism is documented
  - rejected candidate includes rejection evidence
  - fusion candidate reruns all required checks

还可以自动检查候选重复度：

修改文件重叠率；

依赖图变化；

接口结构；

状态模型；

执行模式；

核心控制流。

但机器检测“候选是否真正有价值地不同”仍需要技术负责人复核。

二十六、不要让 Search Plane 退化成什么

1. 不要变成随机生成多个 PR

每个候选必须对应清晰假设。

2. 不要用候选数量代替候选质量

三个实质相同的实现不构成搜索。

3. 不要让候选修改验收标准

候选只能接受共同 verifier，不能为自己定义成功。

4. 不要一开始完整实现所有候选

先验证最可能否定方案的关键假设。

5. 不要让候选共享未提交代码

否则无法判断结果来自哪个方案。

6. 不要只保留最终候选

被淘汰方案及其失败证据是重要团队资产。

7. 不要只比较代码质量

还应比较运行状态、运维负担、回滚、兼容和真实风险。

8. 不要把 LLM judge 作为唯一筛选器

硬约束、性能和兼容性应尽量由工具验证。

9. 不要无限修复失败方案

应定义 falsification 和 repair budget。

10. 不要过早融合

候选尚未独立暴露优缺点前，融合会导致快速收敛和证据污染。

二十七、分阶段落地

Phase 1：人工定义两个候选

先只支持：

Tech Lead 明确两个方案方向；

独立 worktree；

Candidate Contract；

统一测试；

人工比较。

Phase 2：Agent 辅助候选生成

增加：

Agent 从 Spec 和 Reality 提出候选；

Diversity Matrix；

关键假设原型；

早期自动筛选。

Phase 3：分层预算搜索

增加：

设计、原型、完整实现三级预算；

自动终止明显失败候选；

有限 repair；

候选 benchmark 汇总。

Phase 4：基于历史的搜索策略

积累足够数据后再引入：

哪类任务适合几个候选；

哪类方案历史成功率高；

哪些 proxy 能预测最终结果；

哪种 Agent 或模型适合哪类探索；

搜索边际收益估算。

二十八、衡量 Search Plane 是否有效

指标
含义
Search trigger precision
启动搜索的任务中，搜索是否真正改变了方案
Candidate diversity
候选在关键设计维度上的差异
Early rejection rate
在低成本阶段淘汰无效方案的比例
Falsification efficiency
否定一个错误方向所需成本
Candidate first-pass rate
候选首次通过核心筛选比例
Repair success rate
有限修复后候选恢复比例
Duplicate candidate rate
实质重复候选比例
Search budget utilization
实际使用的搜索预算
Evidence per candidate
每个候选产生的有效证据数量
Finalist count
进入 Decision 的候选数
Abandoned implementation cost
未采用完整实现的成本
Fusion success rate
融合候选最终通过验证的比例
Decision reversal rate
生产后发现被淘汰候选反而更优的频率
Time to viable candidate
形成首个可行候选所需时间
Cost per selected candidate
得到最终可选方案的总搜索成本

Search Plane 的目标函数不是：

$$
\text{生成尽可能多的候选}
$$

而是：

$$
\boxed{
\frac{\text{有决策价值的信息}}
{\text{实现成本 + 评测成本 + 人工评审成本}}
}
$$

二十九、向后续 Plane 输出什么

下游 Plane
获取内容
Evaluation Plane
候选 commit、假设、共同 verifier、实验结果和限制
Decision Plane
Candidate Portfolio、差异矩阵、淘汰证据、finalist
Delivery Plane
候选的回滚、依赖、运行模式和发布差异
Learning Plane
失败方案、有效 proxy、修复轨迹和搜索成本
Orchestration Plane
新增实验任务、修复任务、融合候选任务
Spec / Reality Plane
搜索中暴露的新歧义、冲突事实和缺失约束

最终建议

对单团队，Search Plane 最适合采用：

$$
\boxed{
\text{明确搜索触发条件}
+
\text{冻结共享 Contract}
+
\text{假设驱动候选}
+
\text{受控多样性}
+
\text{隔离 Worktree}
+
\text{分层筛选漏斗}
+
\text{独立反例搜索}
+
\text{有限修复与组合}
+
\text{明确终止条件}
}
$$

默认策略应是：

简单任务不搜索；真正开放的问题只生成少量实质不同的候选；先用最便宜的实验否定错误方向，再把资源投入少数完整实现。

Search Plane 的最终输出不应是：

“Agent 生成了三个方案。”

而应是：

我们探索了哪些设计维度；
每个候选建立在哪些假设上；
什么证据支持或否定了这些假设；
哪些候选因何被淘汰；
哪些候选仍然可行；
当前方案空间中还剩下哪些未探索和不确定部分。

它的核心价值是：

把模型容易进行的“多写几份代码”，转化为一套有假设、有差异、有实验、有淘汰依据的工程搜索过程。

Sources:

Introducing the Codex app

How we built our multi-agent research system
