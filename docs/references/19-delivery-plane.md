# 交付层 Delivery Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 18:50:43

---

## Prompt

继续给下一个 plane 的方案

---

## Response

浏览官方技术资料和部署策略

Software Engineer, Delivery / CD
What is Canary Deployment? When and How To Use It
Deployments and environments
Canary Release: Deployment Safety and Efficiency
3. Notional Reference Model for DevSecOps for ... - NIST Pages
Change starts here · Change.org
How to Implement Canary Model Deployment
CHANGE Definition & Meaning
Canary release vs rolling deployment: Rollback speed, risk ...
Guide for Security-Focused Configuration Management of ...
CHANGE | definition in the Cambridge English Dictionary
What is canary deployment?
Chnge
Why did it take OpenAI 24 hours to roll back a faulty model?
Rev4 Rev5 Compared
CHANGE Definition & Meaning
Canary Deployments for Securing Large Language Models
Canary Deployments, A Safer Way to Release Software
Data Integrity — NIST SP 1800-11 0 documentation - NCCoE
NCP - Glossary - National Institute of Standards and Technology
A/B Testing & Canary Deployments for ML Models: Safe ...

浏览并搜索多个技术资源网站

思考了 9s

下一个是 Delivery & Runtime Plane（交付与运行平面）。

对单团队，它的定位应是：

把 Decision Plane 选定的候选，在受控暴露、持续观测和可快速恢复的条件下交付到真实环境，并用生产证据验证此前基于仓库、测试和 Staging 得出的结论是否仍然成立。

它不是简单的 CI/CD，也不是“PR 合并后自动发布”。它要回答：

被评测通过的代码是否就是实际发布的代码？

哪些变更可以直接发布，哪些必须逐步放量？

如何限制一次发布的用户、流量、数据和地域影响范围？

用哪些指标判断继续、暂停或回滚？

技术指标正常时，业务行为是否仍符合需求？

生产结果与预发布结论不一致时，如何回到前面的 Plane 修正？

Google SRE 将非紧急变更的渐进式发布视为降低风险的基本做法：先将变化应用到少量流量或容量，经过观察后再逐步扩大；GitHub environments 也支持人工审批、分支限制、部署保护规则和环境级 secrets，用于在发布边界实施门禁。(Google SRE)

一、与相邻 Plane 的边界

Plane
回答的问题
主要产物
Evaluation Plane
候选在受控环境中是否满足要求？
Evaluation Pack、Baseline、Verdict
Decision Plane
选择哪个候选，并接受哪些风险？
Decision Manifest、发布条件
Delivery & Runtime Plane
如何接触真实环境，并确认实际效果？
Release Manifest、Rollout Evidence、Runtime Verdict
Learning Plane
生产结果说明前面哪些假设需要修正？
回灌规则、事故样本、评测更新

关键区别是：

Evaluation Plane：
在受控环境中，有证据支持它正确。

Delivery & Runtime Plane：
在真实环境中，逐步确认它仍然正确。

生产环境包含预发布环境难以完整复制的：

真实流量分布；

长尾数据；

用户行为；

下游竞争；

网络抖动；

配置漂移；

热点租户；

持续运行时间；

组织和运维行为。

因此：

$$
\text{Evaluation Pass}
\not\Rightarrow
\text{Production Proven}
$$

二、建议划分为 10 个职责域

职责域
核心职责
主要产物
1. Release Identity
固定将要发布的代码、配置和依赖身份
Release Manifest、Artifact Digest
2. Deployment Readiness
检查发布条件、负责人和环境是否就绪
Readiness Report
3. Change Classification
根据风险选择发布策略
Release Risk Profile
4. Environment Promotion
确保同一制品按顺序进入各环境
Promotion Record
5. Progressive Exposure
控制流量、用户、租户、地域和功能暴露
Rollout Plan
6. Runtime Verification
用真实环境指标检验关键主张
Runtime Evidence
7. Automated Guardrails
根据明确阈值自动暂停或回滚
Guardrail Policy
8. Human Release Control
处理风险接受、异常判断和继续放量
Approval Record
9. Recovery & Mitigation
快速停止影响、回滚代码或关闭功能
Recovery Record
10. Post-release Closure
确认稳定、记录结果并回灌上游
Release Summary、Follow-up Tasks

三、核心产物：Release Contract

Decision Plane 选中候选后，应生成一份 Release Contract。

它不是一份传统的发布 checklist，而是：

$$
\text{发布对象}
+
\text{放量阶段}
+
\text{可观察信号}
+
\text{继续条件}
+
\text{停止条件}
+
\text{恢复方案}
$$

release:
  id: REL-ORDER-184-01

  source:
    candidate: CAND-B
    commit: f19c283
    decision: DEC-ORDER-184-01
    spec_version: 3
    reality_version: 4
    evaluation_version: 2

  artifact:
    image: order-service@sha256:8a31...
    config_version: batch-cancel-v1
    database_schema: 143

  exposure:
    mechanism: feature_flag
    initial_scope:
      internal_users: true
      production_traffic_percent: 0

  rollout:
    - stage: internal
    - stage: tenant_allowlist
    - stage: 1_percent
    - stage: 10_percent
    - stage: 50_percent
    - stage: full

  rollback:
    feature_flag_disable: true
    binary_rollback: supported
    data_rollback: not_required

Release Contract 应成为运行流程的唯一权威版本，避免发布人员从 PR 描述、聊天和旧文档中拼接条件。

四、第一阶段：固定 Release Identity

必须先证明：

被发布的制品，就是 Evaluation Plane 实际评测过的制品。

常见错误包括：

评测后又补了一个“小修改”；

发布时重新构建，依赖解析结果不同；

Staging 与生产使用不同镜像；

配置和 feature flag 未纳入版本；

migration 与应用制品不是同一批次；

手工操作改变了运行参数。

Release Manifest

release_identity:
  source_commit: f19c283
  build_run: build-18422

  artifacts:
    application_image:
      digest: sha256:8a31...
    frontend_bundle:
      digest: sha256:19fe...
    migration_package:
      digest: sha256:71bd...

  dependencies:
    lockfile_hash: 39ad2c
    runtime_version: node-24.3

  configuration:
    config_commit: 821ca3
    feature_flag_definition: batch-cancel-v1

  evaluation:
    evaluated_artifact_digest: sha256:8a31...
    matches_release_artifact: true

推荐采用：

构建一次
→ 保存不可变制品
→ 在不同环境间提升同一制品

而不是：

每个环境重新构建一次

五、第二阶段：Deployment Readiness Gate

发布前应检查的不只是“CI 是否绿色”。

deployment_readiness:
  decision_approved: true
  evaluated_artifact_matches: true
  required_checks_passed: true

  owners:
    release_owner: assigned
    module_owner: assigned
    incident_owner: assigned

  observability:
    success_metrics_ready: true
    guardrail_metrics_ready: true
    dashboards_validated: true
    alerts_tested: true

  recovery:
    feature_flag_ready: true
    binary_rollback_tested: true
    data_recovery_documented: true

  operations:
    change_window_valid: true
    dependencies_available: true
    support_team_notified: true

  blocking_unknowns: []

状态建议

状态
含义
NOT_READY
发布对象、证据或恢复能力不足
READY_FOR_STAGING
可以进入共享测试环境
READY_FOR_CANARY
可以接触少量真实流量
READY_FOR_PROGRESSIVE_ROLLOUT
可以逐步扩大暴露
READY_FOR_FULL_RELEASE
当前证据支持全量发布

GitHub deployment environments 可用于要求人工审批、限制可部署分支、应用自定义保护规则，并在保护规则通过后才允许访问对应环境的 secrets。(GitHub Docs)

六、第三阶段：发布风险分类

不应所有变更使用同一种发布流程。

等级
典型变更
推荐策略
R0
文档、无运行影响的开发工具
合并即可
R1
局部、可快速回滚、无数据变化
自动 Staging，低比例 Canary
R2
用户功能、公共 API、关键业务逻辑
Feature flag + 分阶段放量
R3
支付、鉴权、数据模型、基础设施
小范围 Canary + 人工 Gate + 长观察期
R4
不可逆迁移、高敏数据、全局配置
分离迁移、双写/兼容窗口、专门演练

风险提升信号：

release_risk_escalation:
  - changes_database_schema
  - modifies_auth_or_permissions
  - introduces_new_runtime_component
  - affects_cross_tenant_data
  - changes_event_semantics
  - requires_irreversible_operation
  - has_no_fast_feature_disable
  - depends_on_unvalidated_production_behavior

发布策略由变更后果决定，而不是由改动行数决定。

七、第四阶段：选择发布机制

常见机制解决的是不同问题。

机制
主要作用
不能替代什么
Feature Flag
控制功能是否启用
不能回滚 schema 或副作用
Canary Deployment
控制新版本接收的流量
不能自动证明业务正确
Blue/Green
快速切换整套版本
数据变化仍需单独处理
Shadow Traffic
用真实输入观察候选，但不影响用户
无法完整验证写副作用
Tenant Allowlist
选择特定用户或租户
样本可能不具代表性
Dark Launch
先部署能力但不开放入口
不能验证真实用户体验
Rolling Deployment
分批替换实例
版本混合期需保证兼容
Kill Switch
紧急关闭功能或路径
需要预先实现并验证

Google SRE 将 Canary 定义为先在一小部分真实流量或实例上验证变化，在观察期未发现异常后再逐步推进；如出现问题，应快速恢复到已知良好版本。(Google SRE)

八、推荐的默认发布阶梯

对“存量模块增加新功能”，建议默认采用：

合并候选
  ↓
Staging 验证
  ↓
Dark Launch
  ↓
内部用户
  ↓
指定租户 / 用户
  ↓
1% Canary
  ↓
10%
  ↓
50%
  ↓
100%
  ↓
稳定观察

示例：

阶段
暴露范围
主要目标
Gate
Staging
测试数据
部署和基础集成正确
自动
Dark
生产环境但无用户入口
配置、依赖和资源正常
自动
Internal
员工或测试账号
行为和体验检查
人工
Allowlist
少数可控租户
验证真实业务行为
人工
1%
极小流量
发现严重异常
自动 + 人工
10%
有代表性样本
判断趋势和长尾
人工
50%
大规模验证
验证容量和下游竞争
人工
100%
全量
完成交付
自动
Bake
全量后持续观察
发现延迟性问题
自动

比例只是示例。真正的阶段应由：

业务风险；

总流量；

最小统计样本；

故障可检测时间；

用户分布；

数据不可逆性；

共同决定。

九、每个阶段都必须定义 Entry、Observe、Exit

不能只写：

先灰度 1%，没问题再全量。

应明确：

rollout_stage:
  name: canary_1_percent

  entry:
    - previous_stage_passed
    - no_active_incident
    - dashboards_healthy
    - release_owner_available

  exposure:
    traffic_percent: 1
    tenant_strategy: representative_sample

  minimum_observation:
    duration: 30m
    successful_requests: 5000

  continue_if:
    - error_rate_delta <= 0.1_percent
    - p95_latency_delta <= 5_percent
    - duplicate_refund_count == 0
    - support_incidents == 0

  pause_if:
    - evidence_inconclusive
    - sample_size_insufficient

  rollback_if:
    - duplicate_refund_count > 0
    - authorization_violation_count > 0
    - inventory_qps > 180

“观察 30 分钟”和“至少 5000 次请求”最好同时存在。只用时间可能样本不足，只用样本可能漏掉周期性问题。

十、第五阶段：Runtime Verification

生产验证不能只监控 CPU 和错误率。

建议分成五类信号。

信号
回答的问题
示例
Technical Health
系统是否正常运行？
错误率、延迟、CPU
Functional Correctness
新功能是否按规格工作？
成功率、状态一致性
Invariant Signals
是否发生不可接受行为？
重复退款、权限绕过
Dependency Impact
是否伤害上下游？
下游 QPS、超时、积压
User / Business Outcome
需求是否真正产生价值？
完成时间、使用率、失败恢复

技术健康指标

请求错误率；

P50/P95/P99；

CPU、内存；

连接池；

队列积压；

timeout；

restart；

saturation。

功能正确性指标

每批成功和失败数量；

非法状态拒绝率；

部分成功处理准确率；

重试成功率；

API 与状态结果一致性。

不变量信号

runtime_invariants:
  duplicate_refund_count: 0
  duplicate_inventory_rollback_count: 0
  unauthorized_cancel_count: 0
  orphan_batch_count: 0
  inconsistent_order_state_count: 0

不可接受事件通常应使用绝对值为零，而不是平均率。

业务结果

假设需求目的是减少人工处理时间，则还应监控：

平均处理订单数；

操作完成时间；

重复操作率；

人工恢复次数；

用户取消批次中途放弃率。

否则可能出现：

技术指标全部正常，
但用户因为错误反馈难以理解而不使用功能。

十一、Release Indicators 应来自前面的 Plane

Runtime 指标不能发布前临时发明。

来源
转换为运行信号
Spec 不变量
零容忍告警
Spec 验收目标
业务成功指标
Reality Baseline
差分参照
Evaluation 指标
Canary 对照
Decision 条件
发布 Gate
Decision 残余风险
专项观测指标

例如：

Spec：
每个订单最多退款一次。

Evaluation：
故障注入测试中 refund_count ≤ 1。

Delivery：
生产 duplicate_refund_count 必须始终为 0。

这形成完整链条：

$$
\text{Requirement}
\rightarrow
\text{Test}
\rightarrow
\text{Runtime Signal}
$$

十二、第六阶段：基线与对照组

发布后看到错误率为 0.5%，不能单独判断好坏。

需要比较：

发布前同一服务；

未暴露的 control 组；

同时期历史范围；

同类租户；

同一 workload；

同一地域。

runtime_comparison:
  canary:
    error_rate: 0.42%
    p95_ms: 138

  control:
    error_rate: 0.39%
    p95_ms: 126

  delta:
    error_rate: +0.03%
    p95: +9.5%

  decision:
    latency_guardrail_failed

Google SRE 的 Canary 方法强调，Canary 的意义不仅是查看新版本本身，还在于将其行为与未变化的 control 集合进行比较，从而降低环境波动造成的误判。(Google SRE)

十三、自动 Guardrail 与人类判断分离

适合自动处理

明确硬阈值；

不变量违反；

服务不可用；

安全事件；

指标剧烈恶化；

下游超过容量；

数据一致性异常。

automatic_rollback:
  - duplicate_refund_count > 0
  - authorization_bypass_count > 0
  - error_rate_delta > 2_percent
  - p99_latency_delta > 30_percent
  - inventory_qps > 200

适合暂停并由人判断

轻微性能退化但业务收益明显；

用户反馈存在分歧；

数据样本不足；

新旧版本指标接近；

指标异常可能由外部事件造成；

业务价值尚不明确。

automatic_pause:
  - p95_latency_delta between 5_percent and 15_percent
  - error_rate_delta between 0.1_percent and 0.5_percent
  - business_success_rate inconclusive

核心原则：

明确不可接受的行为由机器立即阻止；需要价值取舍的情况由人判断。

十四、第七阶段：回滚并不只有一种

恢复方式
适用场景
Feature Disable
新功能路径有问题，但基础部署正常
Traffic Shift
Canary 版本异常
Binary Rollback
新版本整体有问题
Config Rollback
参数或路由错误
Forward Fix
回滚风险高，但修复较小
Data Compensation
已产生外部副作用
Migration Roll-forward
schema 无法安全回退
Dependency Isolation
下游异常，需要熔断或降级

Recovery Contract

recovery:
  feature_disable:
    command: disable batch-cancel-v1
    expected_time: 2m

  binary_rollback:
    target_digest: sha256:previous...
    expected_time: 10m

  data_compensation:
    required_if:
      - duplicate_side_effect_detected
    owner: payment-operations
    runbook: RUNBOOK-REFUND-07

  verification_after_recovery:
    - new_requests_use_single_cancel
    - error_rate_returns_to_baseline
    - no_new_orphan_batches

回滚能力必须在发布前测试，而不是事故发生后才检查 runbook 是否可用。

NIST DevSecOps 参考模型也将部署监控、成功验证以及失败情况下的回滚管理视为运行团队的核心职责。(NIST页面)

十五、数据库和状态变更需要独立策略

代码回滚通常比数据回滚容易。

对于 schema 和状态变更，建议采用：

Expand
→ 兼容代码上线
→ 数据迁移 / 双写
→ 验证
→ 切换读取
→ 稳定观察
→ Contract

推荐要求

新旧代码均能处理过渡 schema；

migration 可重复执行；

migration 有速率控制；

大规模迁移与功能上线分开；

数据验证脚本独立于应用；

删除旧字段安排在后续发布；

回滚时不会读取无法理解的新数据。

data_change:
  strategy: expand_migrate_contract

  expand:
    backward_compatible: true

  migration:
    batch_size: 1000
    pause_on_error_rate: 0.1_percent

  validation:
    row_count_match: required
    semantic_checksum: required

  contract:
    earliest_after: 14d

不要把：

应用已回滚

误认为：

系统状态已经恢复。

十六、第八阶段：Agent 在发布中的权限边界

Delivery Plane 是风险最高的阶段之一。

活动
Agent 可以做
Agent 不应默认做
生成发布计划
可以
-
检查 Readiness
可以
-
部署 Staging
低风险时可以
-
创建生产部署请求
可以
-
启动已批准 Canary
可按策略执行
不得自行扩大权限
分析运行指标
可以
-
建议继续或暂停
可以
-
自动执行硬阈值回滚
可以，由策略授权
不得自行修改阈值
扩大到全量
需满足 Gate
高风险任务需人批
修改生产数据
默认禁止
除非专门审批和 runbook
更改告警阈值
需人审批
防止为了通过而降标准
忽略异常继续放量
禁止
-

Agent 的角色应是：

发布执行助手、证据收集器和异常分析器。

而不是：

生产风险的自主接受者。

十七、发布 Agent 与实现 Agent 应隔离

不建议让完成实现的 Agent直接拥有生产发布决定权。

角色
职责
Implementation Agent
形成候选和发布说明
Evaluation Agent / CI
输出验证证据
Release Coordinator
执行已批准的发布计划
Runtime Observer
分析指标和异常
Release Owner
决定继续、暂停或回滚
Incident Owner
异常时接管处置

原因是实现 Agent 存在完成目标的倾向，容易：

低估模糊指标；

把异常解释成环境噪声；

倾向继续放量；

偏好修复而不是回滚；

维护自己方案的叙事。

十八、Runtime Verdict

每个阶段结束后，应输出结构化结论。

runtime_verdict:
  release_id: REL-ORDER-184-01
  stage: canary_10_percent

  exposure:
    users: 18420
    requests: 63142
    duration: 2h

  technical:
    error_rate_delta: +0.02%
    p95_latency_delta: +3.1%
    resource_delta:
      cpu: +4.2%

  invariants:
    duplicate_refund: 0
    unauthorized_cancel: 0
    inconsistent_state: 0

  business:
    median_operator_time_delta: -61%
    batch_retry_rate: 1.8%

  limitations:
    - enterprise tenants underrepresented
    - no regional outage occurred during observation

  verdict: PASS_WITH_LIMITATIONS
  next_action: expand_to_50_percent

建议使用：

Verdict
含义
INVALID
指标、制品或环境身份不可信
ROLLBACK
出现不可接受问题
PAUSE
需要更多证据或人工判断
HOLD
指标正常，但等待规定观察期
PROCEED_WITH_LIMITS
可继续，但保持额外限制
PROCEED
当前阶段通过
FULLY_RELEASED
已全量并完成稳定观察

十九、发布异常的路由

生产异常不能统一回到实现 Agent。

异常类型
返回位置
发布脚本或环境故障
Delivery Plane
候选实现 Bug
Search / Orchestration
测试遗漏
Evaluation Plane
生产 baseline 与预期不符
Reality Plane
需求语义与用户行为不一致
Spec Plane
原方案风险判断错误
Decision Plane
权限或越界问题
Governance Plane
重复性组织问题
Learning Plane

例如：

Canary 中发现真实批次 P95 为 450，
而 Reality Pack 假设最大批量为 100。

这不仅是实现问题，还说明 Reality Plane 的流量模型不完整，Spec 中的批量限制可能也需要重新确认。

二十、Post-release Closure

全量发布不是立即结束。

需要一个 Bake Period，用于观察：

周期性任务；

峰谷流量；

资源泄漏；

缓慢队列积压；

用户使用迁移；

长尾异常；

告警质量；

人工恢复负担。

release_closure:
  full_rollout_at: 2026-07-20T08:00:00Z
  bake_until: 2026-07-27T08:00:00Z

  closure_conditions:
    - no_severity_1_or_2_incident
    - no_invariant_violation
    - p95_within_budget
    - business_success_signal_met
    - support_load_within_expected_range
    - no_manual_recovery_backlog

  result: stable

发布关闭时应：

标记 Release Contract 完成；

更新 Reality Baseline；

保存最终运行证据；

关闭临时 flag 或保留原因；

清理临时兼容逻辑的后续任务；

将异常场景加入 Evaluation；

将新领域事实回灌 Spec 和 Reality。

二十一、完整流程

读取 Decision Manifest
        ↓
固定 Release Identity
        ↓
Deployment Readiness Gate
        ↓
选择发布策略和暴露阶段
        ↓
部署同一不可变制品
        ↓
Dark / Internal / Canary
        ↓
采集技术、业务和不变量信号
        ↓
自动 Guardrail 判断
        ├── 继续
        ├── 暂停
        └── 回滚
        ↓
逐步扩大暴露
        ↓
全量发布与 Bake
        ↓
Release Closure
        ↓
生产证据回灌 Learning 及上游 Plane

二十二、批量取消订单示例

Decision Plane 输入

selected_candidate: bounded_parallel

conditions:
  max_batch_size: 100
  default_concurrency: 6
  inventory_qps_limit: 180
  feature_flag_required: true

发布阶段

阶段
范围
观察重点
Internal
内部运营账号
行为、错误反馈、审计
Allowlist
2 个低风险租户
真实批量和下游调用
1%
随机流量
严重回归与不变量
10%
代表性租户
P95、热点和重试
50%
大规模流量
容量、资源和依赖竞争
100%
全量
稳定性和业务收益

硬回滚条件

rollback_if:
  duplicate_refund_count: "> 0"
  unauthorized_cancel_count: "> 0"
  inventory_qps: "> 180"
  inconsistent_order_state_count: "> 0"

暂停条件

pause_if:
  p95_latency_regression: "> 5%"
  partial_failure_rate: "> expected_baseline + 0.5%"
  operator_support_tickets: abnormal

业务成功信号

success:
  median_manual_processing_time_reduction: ">= 50%"
  successful_batch_completion_rate: ">= 98%"
  manual_recovery_rate: "<= 1%"

二十三、仓库内最小实现

不需要开发一个发布平台。

repo/
├── specs/
│   └── ORDER-184/
│       └── delivery/
│           ├── release.yaml
│           ├── rollout.yaml
│           ├── guardrails.yaml
│           ├── runtime-results.yaml
│           └── release-summary.md
├── deploy/
│   ├── environments/
│   ├── feature-flags/
│   └── dashboards/
├── runbooks/
│   ├── rollback.md
│   └── batch-cancel-recovery.md
├── scripts/
│   ├── verify-release-identity.py
│   ├── check-deployment-readiness.py
│   ├── evaluate-canary.py
│   └── rollback.sh
└── .github/
    └── workflows/
        ├── deploy-staging.yml
        ├── deploy-canary.yml
        ├── promote-release.yml
        └── rollback.yml

GitHub environments 和 rulesets 已能覆盖不少单团队需求，包括环境审批、分支限制、部署保护、环境 secrets，以及要求特定环境部署成功后才允许合并。(GitHub Docs)

二十四、可以自动检查什么

delivery_checks:
  - release artifact matches evaluated artifact
  - decision and release versions match
  - rollout stages are ordered
  - each stage has entry and exit conditions
  - hard runtime invariants have monitors
  - rollback conditions are machine-readable
  - rollback command has been tested
  - dashboard queries return valid data
  - canary and control populations are comparable
  - required owners are available
  - production secrets are environment-scoped
  - no rollout continues after guardrail failure
  - full release requires completed canary evidence

不能自动决定的包括：

轻微退化是否值得接受；

用户反馈是否影响业务价值；

当前是否适合承担额外风险；

是否应该继续灰度而不是延期；

某个不可逆数据动作是否值得。

二十五、不要让 Delivery & Runtime Plane 退化成什么

1. 不要把 CI 通过直接等同于可全量发布

CI 只证明配置过的预发布检查通过。

2. 不要重新构建不同环境的制品

否则无法证明发布的是已评测候选。

3. 不要只有技术健康指标

功能错误和业务失败可能不会提高 CPU 或 5xx。

4. 不要只写“观察无异常”

必须保存指标、对照组、样本量和时间范围。

5. 不要让实现 Agent 自己决定继续放量

生产风险接受需要独立责任人。

6. 不要把 Feature Flag 当万能回滚

它不能撤销已经发生的数据和外部副作用。

7. 不要在告警触发后临时讨论如何回滚

恢复方式必须在发布前定义并演练。

8. 不要一次同时发布多个无法区分的高风险变化

否则异常发生后无法归因。

9. 不要在 Canary 中只选最安全、最不具代表性的用户

样本需要兼顾爆炸半径和代表性。

10. 不要全量后立即宣布完成

需要稳定观察和后续关闭。

二十六、分阶段落地

Phase 1：Release Identity 与人工灰度

先建立：

不可变制品；

Staging；

Feature Flag；

人工 1% → 10% → 100%；

基础 dashboard；

手动回滚 runbook。

Phase 2：机器可读 Guardrail

增加：

发布契约；

Canary / control 比较；

自动暂停；

硬阈值自动回滚；

环境审批；

Release evidence。

Phase 3：功能与业务运行验证

增加：

不变量监控；

业务成功指标；

数据一致性验证；

长尾和租户分层；

Bake Period。

Phase 4：闭环式交付

增加：

生产异常自动生成上游任务；

Runtime evidence 更新 Reality Pack；

生产逃逸缺陷加入 Evaluation；

决策触发器自动检查；

发布风险与历史表现关联。

二十七、衡量 Delivery & Runtime Plane 是否有效

指标
含义
Deployment success rate
发布无人工恢复完成的比例
Change failure rate
发布导致故障或降级的比例
Mean time to detect
问题出现到发现的时间
Mean time to mitigate
发现到停止影响的时间
Rollback success rate
回滚或关闭功能成功比例
Canary catch rate
Canary 阶段发现的问题比例
Full-rollout escape rate
全量后才发现的问题比例
Artifact identity mismatch
发布物与评测物不一致次数
Guardrail precision
自动暂停中真正异常的比例
Guardrail recall
应暂停但未触发的比例
Runtime invariant violations
生产不变量违反数量
Staging-production mismatch
预发布和生产表现差异
Progressive rollout lead time
从 Canary 到全量耗时
Manual approval latency
等待人工 Gate 的时间
Recovery drill success
回滚演练成功比例
Feature-flag cleanup time
临时 flag 清理时间
Business outcome achievement
需求目标实际达成比例
Production evidence completeness
发布结论可追溯比例

目标不是最快完成发布，而是优化：

$$
\boxed{
\frac{\text{安全交付并实现真实价值的变更}}
{\text{发布时间 + 运行风险 + 恢复成本 + 人工负担}}
}
$$

二十八、向后续及上游 Plane 输出什么

Plane
获得内容
Learning Plane
生产成功、失败、误判和恢复轨迹
Reality Plane
新的生产 baseline、流量和依赖事实
Evaluation Plane
生产逃逸场景和预发布差距
Decision Plane
风险预测是否准确、重开条件是否触发
Spec Plane
用户目标是否真实达成、语义是否需调整
Search Plane
方案在真实环境中的优劣和新候选方向
Governance Plane
越权、异常发布和审批问题
Orchestration Plane
补救、回滚、修复与清理任务

最终建议

对单团队，Delivery & Runtime Plane 最适合采用：

$$
\boxed{
\text{不可变发布制品}
+
\text{Deployment Readiness Gate}
+
\text{风险分级发布}
+
\text{Feature Flag / Canary}
+
\text{Canary-Control 对照}
+
\text{技术、业务与不变量信号}
+
\text{自动 Guardrail}
+
\text{独立人类发布责任}
+
\text{已演练恢复机制}
+
\text{生产证据回灌}
}
$$

默认原则应是：

同一个被评测过的制品逐级提升；每扩大一次暴露，都必须获得新的现实证据；不可接受的行为由机器立即阻止，模糊取舍由有权人决定。

Delivery & Runtime Plane 的最终输出不应是：

“功能已经上线。”

而应是：

哪个不可变制品被发布；
依据哪个 Decision 和 Evaluation；
经过了哪些暴露阶段；
每个阶段观察了多少真实样本；
技术、业务和不变量指标如何；
发生过哪些暂停、修复或回滚；
哪些风险已被生产证据消除；
哪些不确定性仍然存在；
当前证据是否支持稳定关闭本次交付。

它的核心价值是：

把“预发布环境中看起来正确”，转换成“在受控真实暴露中被持续验证、可快速停止并能形成反馈闭环的生产变更”。

Sources:

Google SRE: Production Services Best Practices

Deployments and environments

3. Notional Reference Model for DevSecOps for ... - NIST Pages
