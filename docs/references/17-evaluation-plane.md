# 评估层 Evaluation Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 18:42:51

---

## Prompt

继续，给出下一个 plane 的方案

---

## Response

Searching for evaluation frameworks in agent development

Demystifying evals for AI agents
Research
Codecademy: Learn to Code - for Free
SWE-EVO: Benchmarking Coding Agents in Long-Horizon ...
Codédex | Start Your Coding Adventure ⋆˙
Building and evaluating alignment auditing agents
What Is Coding and What Is It Used For
Claude Opus 4.6 System Card
OpenAI Evals
Learn to Code — For Free — Coding Courses for Busy People
The Coding Train
5 tips to help you get started with coding
Dumb question but what is the difference between coding ...

搜索多个网站以查找评估框架

思考了 9s

下一个是 Evaluation Plane（评测与验证平面）。

对单团队，它的定位应是：

把 Spec 中的需求、不变量和质量目标，转化为可执行的验证体系，对 Search Plane 产生的候选进行一致、分层、可复现的检查，并输出带证据、适用范围和残余风险的评测结论。

它不是单纯的“跑 CI”，也不是让另一个 LLM 看一遍代码后打分。它需要回答：

候选是否满足全部硬约束？

新功能是否正确，旧行为是否被破坏？

测试结果是否能支持对应的需求主张？

多个候选是否在相同环境和标准下被公平比较？

哪些结论是确定性的，哪些仍依赖开放判断？

当前证据足以支持合并、灰度还是仅支持继续实验？

Anthropic 的 Agent eval 实践建议组合使用代码型、模型型和人工 grader，因为不同类型的要求无法由单一评测方式可靠覆盖；同时强调同时评估最终结果和执行轨迹，并对 grader 本身进行校准。(Anthropic)
近期软件工程 Agent benchmark 也逐渐从孤立 Bug 修复转向长期演进、生产工作负载、性能优化和未来变更可维护性，说明“当前测试通过”不足以代表真实工程质量。(arXiv)

一、Evaluation Plane 与相邻 Plane 的边界

Plane
回答的问题
主要产物
Spec Plane
什么叫需求完成？
验收标准、不变量、质量目标
Reality Plane
当前系统和环境实际上是什么？
Baseline、证据、现实限制
Search Plane
有哪些候选方案？
Candidate、假设、实现和实验
Evaluation Plane
每个候选是否满足约束，证据有多强？
Evaluation Pack、失败证据、比较矩阵
Decision Plane
在可行候选中选择哪个？
方案选择、风险接受、ADR
Delivery Plane
上线后是否仍满足要求？
Canary、SLO、回滚与运行证据

Evaluation Plane 判定候选是否达到某些标准，但不负责：

改写业务需求；

为了让候选通过而降低门槛；

决定软目标之间的最终价值权衡；

直接批准生产风险。

二、建议划分为 10 个职责域

职责域
核心职责
主要产物
1. Evaluation Contract
将需求映射为具体检查项
Evaluation Plan、Requirement Map
2. Baseline Management
固化修改前行为和运行基线
Behavioral Baseline、Performance Baseline
3. Deterministic Verification
执行编译、测试、静态分析和规则检查
Test Results、API Diff、Policy Results
4. Scenario & Invariant Testing
验证关键业务场景和不可破坏性质
Scenario Suite、Property Tests
5. Non-functional Evaluation
验证性能、安全、可靠性和资源使用
Benchmark、Security Report、Load Report
6. Regression & Compatibility
检查存量行为、接口和数据兼容性
Regression Matrix、Contract Report
7. Open-ended Quality Review
评估架构、可维护性和可理解性
Rubric Review、Pairwise Review
8. Trajectory Evaluation
检查 Agent 是否越界、遗漏证据或投机通过测试
Trace Review、Process Violations
9. Evidence Integrity
确保结果可复现、来源完整且未被候选操控
Evidence Manifest、Reproduction Record
10. Verdict & Uncertainty
输出分层结论、限制和残余风险
Evaluation Summary、Verdict

三、核心产物：Evaluation Pack

每个进入正式评测的候选，应生成独立的 Evaluation Pack。

specs/
└── ORDER-184/
    ├── evaluation/
    │   ├── plan.yaml
    │   ├── baselines/
    │   ├── suites/
    │   ├── candidates/
    │   │   ├── CAND-B/
    │   │   │   ├── results.yaml
    │   │   │   ├── evidence/
    │   │   │   ├── failures.md
    │   │   │   └── limitations.md
    │   │   └── CAND-C/
    │   ├── comparison.yaml
    │   └── summary.md

一个完整 Evaluation Pack 应包含：

内容
作用
Candidate identity
确定评测的是哪个 commit
Spec / Reality version
防止使用过期标准
Environment manifest
确保环境可复现
Baseline
支持差分判断
Hard constraint results
决定候选是否可行
Soft objective measurements
支持后续比较
Scenario coverage
说明测了哪些行为
Failure evidence
保存失败输入和日志
Trace findings
记录执行过程风险
Limitations
明确当前没有证明什么
Verdict
给出机器可消费的结论

四、第一阶段：建立 Evaluation Contract

Evaluation Plane 不应在代码写完后才临时决定“测什么”。

它应该从 Spec Plane 的 Requirement-to-Verifier 映射开始。

例如：

evaluation_contract:
  feature_id: ORDER-184
  spec_version: 3
  reality_version: 4

  requirements:
    - id: INV-01
      statement: each order is refunded at most once
      type: hard_invariant
      verifier:
        - property_test
        - fault_injection_test

    - id: AC-03
      statement: partial failure returns per-order result
      type: behavioral
      verifier:
        - integration_scenario

    - id: NFR-02
      statement: batch of 100 has p95 <= 2s
      type: performance
      verifier:
        - controlled_benchmark

    - id: COMP-01
      statement: existing single-order API behavior remains unchanged
      type: compatibility
      verifier:
        - differential_test
        - contract_test

每个关键要求都需要明确：

$$
\text{Requirement}
\rightarrow
\text{Verifier}
\rightarrow
\text{Evidence}
\rightarrow
\text{Verdict}
$$

如果不存在可信 verifier，不应默认为“人工看起来没问题”，而应明确标记：

UNVERIFIED REQUIREMENT

五、评测对象需要分成四层

很多团队只评测最终代码，但 Agent 交付至少有四个对象。

层级
评测对象
示例
Artifact
代码和配置本身
编译、类型、依赖、diff
Behavior
运行后的系统行为
功能、状态转换、副作用
Outcome
是否实现用户和业务目标
用户能否完成批量取消
Trajectory
Agent 如何得到这个结果
是否越界、删测试、绕规则

Anthropic 的 eval 方法明确区分 outcome grading 和 transcript/trajectory grading：结果可能正确，但过程包含危险操作；过程合理，也不代表结果一定正确。(Anthropic)

例如：

候选最终测试通过

但 Agent 可能：

删除了失败测试；

修改了 fixture 以避开边界输入；

降低了性能阈值；

mock 掉了关键下游；

没有运行它声称运行过的测试。

因此：

$$
\text{Outcome Pass}
\not\Rightarrow
\text{Evaluation Pass}
$$

六、第二阶段：Baseline 管理

没有 Baseline，就很难证明“无回归”“提升”或“兼容”。

建议维护四类 Baseline

Baseline
说明
Behavioral Baseline
当前系统在关键输入下的输出和副作用
Contract Baseline
API、事件、schema 和公共类型
Performance Baseline
延迟、吞吐、内存、CPU 和下游调用
Operational Baseline
错误率、重试、告警、队列和资源情况

例如：

baseline:
  commit: a83d91c
  environment: staging-fixed-v2

  behavior:
    single_cancel_scenarios: 28
    snapshots_hash: 842ad1

  performance:
    single_cancel_p95_ms: 126
    batch_simulated_100_serial_p95_ms: 8400
    inventory_peak_qps: 142

  contracts:
    openapi_hash: d31c1f
    event_schema_version: 7

Baseline 必须和候选使用相同：

环境；

数据集；

硬件；

warm-up；

负载；

统计方法；

依赖版本。

否则：

候选快 30%

可能只是因为环境不同。

性能优化评测研究也指出，原始 speedup 可能来自 benchmark-specific shortcut，而不是真实优化，因此必须同时验证正确性、测量方法和轨迹。(arXiv)

七、第三阶段：建立分层验证金字塔

建议使用六层评测。

L0 结构和策略检查
L1 单元与局部性质
L2 模块和接口
L3 系统与跨服务
L4 非功能和故障
L5 人类与运行验证

L0：静态和结构检查

包括：

格式化；

编译；

类型检查；

lint；

secret scan；

依赖检查；

license；

protected path；

diff 范围；

migration 检查；

policy 检查。

特点：

成本低；

反馈快；

确定性高；

应最先执行。

L1：单元测试和属性测试

验证：

核心计算；

状态规则；

边界输入；

幂等性；

不变量；

错误处理。

L2：模块和 Contract 测试

验证：

API schema；

事件 schema；

数据库行为；

模块调用；

兼容性；

feature flag 行为。

L3：系统集成测试

验证：

多模块调用链；

下游真实或可信替身；

事务边界；

部分失败；

重试；

跨服务状态。

L4：非功能与故障评测

验证：

性能；

并发；

资源；

安全；

故障恢复；

下游限流；

长时间稳定性。

L5：人工和运行环境验证

包括：

架构 review；

UX review；

staging；

shadow；

canary；

真实流量观察。

Evaluation Plane 内主要负责 L0-L4，并为 Delivery Plane 准备 L5 的验证契约。

八、硬约束与软目标必须分开

评测不应该把所有结果压成一个总分。

硬约束

任一失败通常直接淘汰：

hard_constraints:
  - no duplicate refund
  - no authorization bypass
  - existing public API compatible
  - all critical regression tests pass
  - inventory QPS <= 180

软目标

用于可行候选比较：

soft_objectives:
  - lower p95 latency
  - fewer changed modules
  - lower operational complexity
  - simpler rollback
  - lower maintenance burden

正确流程是：

硬约束
   ↓
可行候选集合
   ↓
软目标比较
   ↓
Decision Plane

错误流程是：

安全失败 -20 分
性能优秀 +30 分
最终仍然通过

硬约束不能被平均分掩盖。

九、场景测试与不变量测试

传统 example-based tests 只验证有限例子。

对于 Agent 生成的新功能，还应优先增加：

property-based tests；

metamorphic tests；

state-machine tests；

differential tests；

fault injection；

concurrency schedule tests。

示例：批量取消不变量

properties:
  - for any order, refund_count <= 1
  - for any order, inventory_rollback_count <= 1
  - failed item does not change successful item's result
  - repeating the same idempotency key creates no new side effect
  - single-order behavior is unchanged

状态机测试

PAID
  ↓ cancel
CANCELLING
  ├─ refund succeeds
  ├─ inventory timeout
  ├─ request retries
  └─ process crashes

系统应验证不同事件顺序下，不变量仍成立。

这比只测一个 happy path 更接近真实系统风险。

十、差分验证是存量模块的关键

“新增功能”最难的往往不是新行为，而是旧行为是否改变。

建议对旧系统和候选做差分：

baseline(commit A, input X)
vs
candidate(commit B, input X)

允许差异

由 Spec 明确的新行为：

allowed_differences:
  - new batch endpoint exists
  - audit log includes batch identifier

禁止差异

forbidden_differences:
  - single cancellation response changes
  - existing event field changes
  - unauthorized error semantics change

差分对象可以包括：

返回值；

数据库变化；

事件；

日志；

外部调用；

资源使用；

错误类型；

顺序和时序。

十一、兼容性验证

大型仓库中至少要检查五类兼容性。

类型
典型 verifier
Source compatibility
编译、类型检查
API compatibility
OpenAPI / GraphQL diff
Event compatibility
Schema registry、consumer contract
Data compatibility
Migration test、旧数据回放
Behavioral compatibility
Differential scenario tests

仅仅“代码能编译”不足以证明行为兼容。

例如：

字段仍然存在

但其：

默认值变了；

null 语义变了；

顺序变了；

错误码变了；

触发条件变了。

这类变化可能只能通过 contract test 和历史流量回放发现。

十二、非功能评测

1. 性能

性能评测必须记录：

benchmark:
  environment:
    cpu: 8
    memory_gb: 16
    runtime: node-24
  dataset: batch-cancel-v4
  warmup: 2m
  runs: 20
  aggregation:
    - median
    - p95
    - confidence_interval

不要只记录：

执行用了 1.4 秒。

还需要：

与 baseline 比较；

多次运行；

方差；

异常值；

正确性同时通过；

是否改变 workload。

2. 可靠性

包括：

retry；

crash recovery；

timeout；

partial failure；

duplicate request；

downstream unavailable；

message redelivery；

database deadlock。

3. 安全

包括：

权限绕过；

输入验证；

注入；

secret；

依赖风险；

敏感日志；

tenant isolation；

rate limit。

4. 资源

包括：

CPU；

内存；

连接池；

队列；

文件描述符；

外部调用次数；

CI 时长。

十三、开放式质量应使用 Rubric，而不是自由评价

可维护性、架构一致性和可理解性无法完全由测试决定。

建议使用结构化 rubric。

architecture_rubric:
  domain_consistency:
    5: fully reuses current domain boundaries
    3: introduces one justified abstraction
    1: duplicates business logic across modules

  rollback:
    5: feature flag only
    3: application rollback required
    1: irreversible migration

  cognitive_load:
    5: no new operational concept
    3: one documented new concept
    1: multiple coupled runtime components

评测时优先使用：

有明确锚点的 rubric；

候选间 pairwise comparison；

多 reviewer；

人工抽检；

reviewer 与事实证据分离。

Anthropic 建议根据任务组合代码 grader、模型 grader 和人类 grader，并通过代表性样本校准 grader，避免模型 judge 的偏好成为隐形标准。(Anthropic)

十四、LLM Judge 的适用边界

LLM Judge 适合：

检查说明是否完整；

对照 rubric 分析架构；

识别潜在遗漏；

比较两个方案的可理解性；

总结证据；

生成 review 问题。

不适合单独负责：

数值计算；

测试是否通过；

性能是否达标；

安全性质是否成立；

API 是否兼容；

数据是否一致；

最终生产风险接受。

推荐架构：

确定性结果
+
结构化 evidence
+
LLM rubric review
+
人类抽检

而不是：

代码 diff
→ LLM 看一遍
→ 通过

十五、独立 Verifier 原则

实现者不能完全控制自己的 verifier。

最低限度应做到：

对象
控制者
实现代码
Implementer
核心验收标准
Spec Owner
测试框架
Evaluation Plane
隐藏/独立测试
Verifier 或 Module Owner
最终门禁
CI + Human Gate

防止以下行为：

删除失败测试；

修改阈值；

把真实依赖换成过度理想化 mock；

只运行局部 suite；

使用与 baseline 不同的数据集；

把失败标为 flaky；

更改 evaluation script。

十六、测试污染与 Goodhart 问题

候选知道全部测试后，可能形成“通过测试但不满足真实需求”的实现。

工程上可采用：

部分隐藏测试；

独立场景设计；

生产流量回放；

属性测试；

变形测试；

Challenger Agent；

人工随机抽样；

多环境验证。

例如：

测试只检查返回 100 个结果

候选可能为每项都返回 success，却没有真实执行取消。

所以 verifier 应检查：

输入
→ 状态变化
→ 副作用
→ 事件
→ 返回值

而不只检查最终 JSON。

SWE-bench Verified 的建立本身就是因为原始任务中存在不可解或测试不充分问题，说明评测集质量直接决定 benchmark 结论可信度。(OpenAI)

十七、Trajectory Evaluation

除了结果，还应检查 Agent 的执行轨迹。

需要关注的轨迹事件

类型
示例
Scope violation
修改了禁止目录
Test manipulation
删除或弱化测试
Unsupported claim
声称测试通过但没有记录
Repeated failure
多次重复相同错误操作
Hidden assumption
未记录地改变业务语义
Unsafe command
执行越权或危险命令
Evidence omission
忽略关键失败日志
Shortcut exploitation
修改 benchmark 而非实现

trajectory_findings:
  - type: test_threshold_change
    severity: high
    file: benchmark/config.yaml
    evidence: diff-183
    verdict: candidate_invalid

最终功能即使通过，也可能因为轨迹违反治理策略而被拒绝。

十八、证据完整性

评测结果必须可以重放。

每个结果至少记录

evidence:
  candidate_commit: f19c283
  baseline_commit: a83d91c
  spec_version: 3
  reality_version: 4

  environment:
    image: eval-order-v7
    dependency_lock_hash: 39ad2c
    hardware_profile: perf-standard-8c

  command:
    - pnpm test batch-cancel
    - ./bench/run.sh --scenario batch100

  artifacts:
    - junit.xml
    - benchmark.json
    - api-diff.json
    - trace.log

  started_at: ...
  completed_at: ...
  exit_code: 0

没有这些信息的“测试通过”只能作为弱证据。

十九、评测结果应使用分层 Verdict

不建议只输出 Pass / Fail。

建议使用：

Verdict
含义
INVALID
候选或证据无效，例如评测脚本被修改
FAIL
明确违反硬约束
INCONCLUSIVE
证据不足或环境不可信
PASS_WITH_LIMITATIONS
当前范围通过，但存在明确限制
PASS
全部规定检查通过
NOT_EVALUATED
尚未执行某项检查

例如：

verdict:
  overall: PASS_WITH_LIMITATIONS

  hard_constraints:
    correctness: PASS
    compatibility: PASS
    security: PASS
    performance: PASS

  limitations:
    - not tested with production traffic distribution
    - cross-region refund behavior not evaluated

  recommended_next_stage:
    canary_1_percent

“通过”必须说明：

在哪个环境；

针对哪个版本；

覆盖哪些范围；

还有哪些未验证项。

二十、多个候选的比较方式

Evaluation Plane 输出统一矩阵，但不直接决定最终选择。

指标
A 串行
B 有界并发
C 异步
正确性
Pass
Pass
Pass
兼容性
Pass
Pass
Pass
100 单 P95
8.4s
1.48s
提交 90ms
下游峰值 QPS
32
168
140
故障恢复
低
中
高
新运行组件
0
0
2
回滚复杂度
低
低
中
维护 rubric
5
4
3
证据强度
高
高
中

Evaluation Plane 可以指出：

A 违反延迟硬约束，淘汰。
B 和 C 均可行。

但“选 B 还是 C”属于 Decision Plane，因为这涉及：

运维能力；

未来批量规模；

团队偏好；

发布时机；

架构方向。

二十一、Evaluation Readiness Gate

正式评测前检查：

evaluation_readiness:
  spec_version_locked: true
  reality_baseline_available: true
  candidate_commit_immutable: true

  hard_constraints_mapped: true
  critical_requirements_have_verifiers: true
  baseline_environment_reproducible: true
  shared_candidate_environment_defined: true

  hidden_or_independent_tests_available: true
  benchmark_method_defined: true
  failure_artifacts_retained: true
  evaluator_independent_from_implementer: true

  limitations_template_ready: true
  human_reviewers_assigned: true

状态建议：

状态
含义
Not Ready
缺少 baseline、verifier 或稳定候选
Ready for Screening
可以执行低成本硬约束检查
Ready for Full Evaluation
可以执行完整功能和非功能验证
Ready for Decision
候选证据已经足够进行取舍

二十二、完整评测流程

读取 Spec、Reality 和 Candidate Contract
                ↓
验证版本、commit 和评测环境
                ↓
加载 Baseline
                ↓
执行 L0 静态与政策检查
                ↓
执行 L1 单元、不变量和属性测试
                ↓
执行 L2 contract 与兼容性检查
                ↓
执行 L3 系统场景和故障测试
                ↓
执行 L4 性能、安全和资源评测
                ↓
执行轨迹与证据完整性检查
                ↓
开放式 rubric 与人工 review
                ↓
生成候选 Verdict
                ↓
统一比较矩阵
                ↓
进入 Decision Plane

二十三、存量模块新增功能示例

以批量取消订单为例。

Hard Gate

检查
方法
不重复退款
幂等属性测试 + 故障注入
不重复回滚库存
事件重放测试
权限不被绕过
权限矩阵测试
单笔接口无回归
Baseline 差分测试
事件兼容
Consumer contract test
下游 QPS ≤ 180
可控负载测试

Scenario Suite

全部成功
部分状态非法
权限不足
退款成功后响应超时
库存服务限流
重复批量请求
两个批量请求重叠
进程中途退出
事件重复投递
Feature flag 回退

软目标

P95；

代码改动范围；

新增运行概念；

回滚复杂度；

运维可见性；

长期扩展能力。

二十四、仓库内最小实现

不需要建立独立评测平台。

repo/
├── specs/
│   └── ORDER-184/
│       └── evaluation/
│           ├── plan.yaml
│           ├── comparison.yaml
│           └── candidates/
├── tests/
│   ├── acceptance/
│   ├── invariants/
│   ├── contracts/
│   └── fault-injection/
├── benchmarks/
│   └── batch-cancel/
├── scripts/
│   ├── collect-baseline.sh
│   ├── run-evaluation.sh
│   ├── verify-evidence.py
│   └── compare-candidates.py
└── .github/
    └── workflows/
        ├── evaluation-screen.yml
        ├── candidate-full-eval.yml
        └── evidence-integrity.yml

OpenAI 的 harness engineering 实践也强调将验证工具、仓库规则、测试和可观察反馈编码进工程环境，让 Agent 能自己运行并纠正，但最终结论仍由可复现的系统证据支撑。(OpenAI)

二十五、自动检查项

evaluation_checks:
  - candidate commit is immutable
  - spec and reality versions match
  - evaluation scripts are unchanged by candidate
  - all hard requirements have results
  - test command and exit code are recorded
  - baseline and candidate use same environment
  - benchmark includes multiple runs
  - performance gains preserve correctness
  - failed tests are not removed or skipped
  - public contract changes are explicitly approved
  - limitations are non-empty when coverage is partial
  - every PASS claim links to evidence

二十六、不要让 Evaluation Plane 退化成什么

1. 不要等到实现完成才设计测试

否则 verifier 很容易被实现结构影响。

2. 不要把 CI 绿灯等同于需求完成

CI 只代表已配置检查通过。

3. 不要用一个总分覆盖硬约束失败

安全和正确性不能被性能分数抵消。

4. 不要让实现 Agent 修改评测规则

评测资产必须独立保护。

5. 不要只验证新增行为

存量模块必须重点验证旧行为回归。

6. 不要只测 happy path

需要重试、并发、故障和恢复场景。

7. 不要只看最终结果

还要检查 Agent 是否投机或越界。

8. 不要依赖单个 LLM Judge

开放式 review 必须建立在确定性证据之上。

9. 不要把 staging 结果外推为生产事实

必须标记环境范围。

10. 不要只保留通过报告

失败输入和淘汰证据对 Learning Plane 更有价值。

二十七、分阶段落地

Phase 1：需求-测试映射

先建立：

硬约束清单；

Requirement-to-Verifier；

Baseline；

统一候选测试命令；

证据保存。

Phase 2：存量差分和不变量

增加：

Differential tests；

Property tests；

Contract tests；

失败场景；

protected evaluation assets。

Phase 3：非功能与轨迹

增加：

Benchmark；

Fault injection；

Security checks；

Resource measurements；

Agent trace review。

Phase 4：持续评测资产

增加：

生产事故回灌；

历史回归集；

长期维护测试；

grader 校准；

flaky test 治理；

多次发布后的评测有效性分析。

SWE-CI 和 SWE-EVO 这类研究将未来修改和持续演进纳入评测，说明成熟阶段应关注“这次改动是否让下一次修改更难”，而不只是当前 PR 是否通过。(arXiv)

二十八、衡量 Evaluation Plane 是否有效

指标
含义
Requirement verification coverage
有 verifier 的关键需求比例
Hard-constraint escape rate
未被发现而进入后续阶段的硬约束问题
Production defect escape
评测未发现的生产缺陷
Regression detection rate
合并前发现的旧行为回归比例
False pass rate
评测通过但实际错误的比例
False rejection rate
合格候选被错误淘汰的比例
Reproduction success
其他人能否复现评测结果
Evaluation lead time
完整评测耗时
Evaluation cost
CI、环境和人工成本
Flaky test rate
非确定性失败比例
Evidence completeness
结论能追溯到证据的比例
Benchmark variance
性能结果的稳定程度
Judge agreement
LLM grader 与人工 reviewer 的一致性
Unverified requirement count
尚无可信验证方式的要求数量
Post-release metric mismatch
预发布结果与生产结果差异
Future-change regression
后续需求因本次设计产生的回归率

Evaluation Plane 的目标不是测试越多越好，而是：

$$
\boxed{
\frac{\text{提前发现的高价值问题}}
{\text{评测时间 + 环境成本 + 人工成本}}
}
$$

二十九、向后续 Plane 输出什么

下游 Plane
获取内容
Decision Plane
可行候选、硬约束结果、软目标矩阵和证据强度
Delivery Plane
发布前置条件、canary 指标、回滚触发条件
Learning Plane
失败场景、漏测问题、grader 误差和生产逃逸缺陷
Search Plane
候选失败原因、可修复点和新实验建议
Orchestration Plane
重测、修复和补证据任务
Spec Plane
无法验证、冲突或需要重新定义的需求
Reality Plane
baseline 偏差、环境差异和新发现的现实事实

最终建议

对单团队，Evaluation Plane 最适合采用：

$$
\boxed{
\text{Requirement-to-Verifier 映射}
+
\text{Baseline 差分}
+
\text{硬约束优先}
+
\text{场景与不变量测试}
+
\text{Contract 与回归验证}
+
\text{性能、安全和故障评测}
+
\text{轨迹审计}
+
\text{证据可复现}
+
\text{Rubric 与人工复核}
}
$$

默认原则应是：

确定性的要求用工具证明，开放式的质量用 rubric 评估，真实运行效果留给灰度验证；任何结论都必须同时说明证据、范围和未覆盖部分。

Evaluation Plane 的最终输出不应是：

“测试全部通过。”

而应是：

候选基于哪个 commit 和环境被评测；
哪些需求由哪些 verifier 覆盖；
哪些硬约束已经通过；
哪些指标优于或劣于 baseline；
哪些结论仍然依赖人工判断；
哪些场景尚未验证；
当前证据支持进入哪个交付阶段。

其核心价值是：

把“这个实现看起来正确”，转化为“在明确范围内，有一组可执行、可复现并可质疑的证据支持它正确”。

Sources:

Demystifying evals for AI agents

SWE-EVO: Benchmarking Coding Agents in Long-Horizon ...

Introducing SWE-bench Verified
