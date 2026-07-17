# 现实层 Reality Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/16 15:43:15

---

## Prompt

继续给出 Reality Plane 的方案

---

## Response

Searching web for coding agents and provenance evidence

Harness engineering: leveraging Codex in an agent-first ...
Engineering | Journal | ScienceDirect.com by Elsevier
Adding repository custom instructions for GitHub Copilot
Artificial Intelligence Risk Management Framework (AI RMF 1.0)
When vibe coding turns into an unfixable mess - Codex
Is engineering a dying industry in Singapore?
Artificial intelligence
AI Risk Management Framework | NIST
Engineering Courses In Singapore | MDIS
NIST AI RMF to ISO/IEC FDIS 42001 AI Management ...
What Is Artificial Intelligence (AI)?
Undergraduate | College of Engineering | NTU Singapore
Google Gemini
Engineering Jobs in Singapore - Jul 2026 | Jobstreet
Diploma in Engineering with Business | SP
What is Artificial Intelligence (AI)? | Glossary
Getting started with Codex
Engineering industry - Singapore

搜索多个网站以获取编码代理和来源证据

思考了 33s

对于单团队，Reality Plane 的定位应是：

从代码仓库、运行环境和团队经验中采集与当前需求相关的证据，把“系统实际上是什么”整理成可追溯、可更新、带置信边界的任务现实模型。

它不是企业知识平台，也不是把整个仓库做成知识图谱。它只需要围绕当前需求，回答：

当前系统实际如何运行？

哪些结论有直接证据？

哪些只是推断、文档主张或人的记忆？

哪些未知项会影响设计与验收？

证据在什么时间、环境和版本下有效？

大型代码仓库中的事实通常分散在代码、测试、配置、运行指标和历史记录中。GitHub 的文档也强调，coding agent 的仓库理解依赖及时更新的索引、仓库指令以及相关上下文；OpenAI 的 agent-first 工程实践则强调，需要把架构、运行方式和验证方法直接编码进仓库，使 Agent 能够发现并验证系统状态，而不是仅靠提示词猜测。(OpenAI)

一、Reality Plane 与 Spec Plane 的区别

两者容易混淆。

Plane
回答的问题
内容性质
Spec Plane
系统应该变成什么样？
意图、约束、验收标准和决策
Reality Plane
系统现在实际上是什么样？
观测、证据、现状、历史和未知
Search Plane
可以如何从现状走向目标？
候选设计和实现
Evaluation Plane
候选是否真的满足要求？
测试、测量和验证结果

例如：

Spec：
批量取消必须避免重复退款。

Reality：
当前退款接口是否幂等，尚未确认。
现有单笔取消服务在两个路径中调用退款。
生产中曾出现过一次重复回调事故。

Reality Plane 不负责决定“必须怎样实现”，而负责告诉下游 Plane：

你现在面对的真实系统边界和证据是什么。

二、建议划分为 8 个职责域

职责域
核心职责
主要产物
1. Evidence Scope
根据需求确定需要调查哪些事实，避免无目的扫描整个仓库
Investigation Scope、关键问题清单
2. Repository Reconnaissance
探索代码、测试、配置、schema、依赖和构建方式
Repository Map、模块边界、调用链
3. Runtime Observation
收集日志、指标、trace、流量特征和真实故障模式
Runtime Baseline、运行证据
4. Historical Context
调查历史 PR、ADR、事故和废弃方案
Decision History、Incident Context
5. Human Knowledge Capture
获取代码中不存在的业务规则、运维习惯和组织约束
Expert Notes、隐性规则
6. Epistemic Classification
区分观测事实、来源主张、推断、假设、冲突和未知
Evidence Ledger
7. Reality Modeling
将相关证据组织为状态机、依赖图、不变量候选和风险图
Task Reality Model
8. Freshness & Conflict Management
管理证据时效、版本适用范围和来源冲突
Freshness Status、Conflict Log

三、Reality Plane 的核心产物：Task Reality Pack

每个中等以上需求，建议形成一份轻量的 Task Reality Pack。

specs/
└── ORDER-184-batch-cancel/
    ├── spec.md
    ├── acceptance.yaml
    ├── reality/
    │   ├── reality-summary.md
    │   ├── evidence.yaml
    │   ├── repository-map.md
    │   ├── runtime-baseline.yaml
    │   ├── conflicts.md
    │   └── evidence/
    └── decisions.md

它不是永久维护的百科全书，而是当前需求的证据快照。

建议至少包含：

内容
说明
Current behavior
当前系统如何工作
Relevant components
涉及的模块、服务和数据
Dependency map
上下游调用与副作用
Runtime baseline
当前性能、错误率和容量
Confirmed evidence
有直接依据的事实
Inferences
从证据推导但未直接确认的结论
Contradictions
不同来源之间的冲突
Unknowns
尚未确认且可能影响实现的问题
Freshness
证据对应的 commit、环境和采集时间
Limitations
当前模型没有覆盖的范围

四、第一阶段：从 Spec 生成调查问题

Reality Plane 不应一开始就让 Agent“全面研究仓库”。

它应从 Spec 中提取需要现实确认的命题。

假设需求是：

在订单模块增加批量取消功能。

Spec 中可能包含：

invariants:
  - each order is refunded at most once
  - existing single-order behavior remains unchanged

hard_constraints:
  - inventory QPS must not exceed 180
  - existing event consumers must remain compatible

Reality Plane 应生成调查问题：

investigation_questions:
  - id: RQ1
    question: 当前单笔取消完整调用链是什么？
    reason: 批量实现必须复用或保持其行为

  - id: RQ2
    question: 退款调用是否具备幂等能力？
    reason: 决定批量请求和重试策略

  - id: RQ3
    question: 库存服务当前限流和生产峰值是多少？
    reason: 决定并发上限

  - id: RQ4
    question: OrderCancelled 事件有哪些消费者？
    reason: 判断 schema 变化的兼容风险

这样 Agent 调查的是与决策有关的事实，而不是无边界地阅读代码。

五、第二阶段：仓库现实侦察

仓库侦察不能只搜索需求关键词。

Agent 应从多个入口交叉建立现状。

1. 代码入口

寻找：

API 入口；

应用服务；

领域逻辑；

数据访问；

事务边界；

外部服务调用；

事件发送；

权限与审计；

重试和补偿；

feature flag。

2. 测试入口

测试往往比文档更接近当前可执行行为。

需要调查：

单元测试；

集成测试；

contract test；

snapshot；

fixture；

property test；

失败场景；

被跳过或被隔离的测试。

3. 配置和基础设施入口

包括：

环境变量；

限流配置；

超时；

重试次数；

队列配置；

数据库 schema；

部署资源；

CI 流程；

运行时 feature flag。

4. 依赖入口

不仅要找“谁被当前模块调用”，还要找：

谁调用当前模块
当前模块调用谁
谁订阅当前模块的事件
谁读取当前模块写入的数据

GitHub 的仓库探索指南同样建议从入口文件、模块关系、测试和历史修改等多个角度理解陌生代码库，而不仅是查看单个文件。(GitHub Docs)

六、不要直接把搜索结果当成事实

Agent 找到一段代码，不代表它就是生产真实路径。

例如：

async function cancelOrder(orderId) {
  await refund(orderId);
}

可能存在：

该函数已废弃；

只有旧 API 调用它；

实际生产使用另一个 feature flag 路径；

某些租户使用独立实现；

退款在 interceptor 中还有一次调用；

当前部署版本尚未包含这段代码。

因此每条事实都应包含适用上下文：

evidence:
  id: EV-012
  claim: single cancellation calls refund service once
  source:
    type: code
    file: src/order/cancel-service.ts
    lines: 173-192
    commit: a83d91c
  applicability:
    environment: source-main
    feature_flag: new_cancel_flow=true
  confidence: medium
  limitations:
    - production flag distribution not yet verified

代码证据证明的是：

在这个版本和这个路径中，代码表达了什么。

它不一定直接证明生产中真正发生了什么。

七、第三阶段：运行时现实采集

对涉及性能、可靠性、流量或兼容性的需求，仅靠仓库信息不够。

需要收集当前运行 baseline。

建议调查的运行事实

类别
示例
流量
QPS、请求大小、批次分布、热点租户
延迟
P50、P95、P99、超时分布
错误
错误率、重试率、失败类型
资源
CPU、内存、连接池、队列积压
下游
调用量、限流、超时和熔断
数据
订单状态分布、异常状态和长尾数据
行为
用户操作顺序、重复提交和取消行为
事故
真实故障模式和恢复时间

例如：

runtime_baseline:
  collected_at: 2026-07-16T03:00:00Z
  environment: production
  window: 14d

  single_cancel:
    request_count: 184220
    p50_ms: 71
    p95_ms: 126
    p99_ms: 382
    error_rate: 0.0008

  inventory_service:
    current_peak_qps: 142
    configured_limit_qps: 200
    observed_throttling_start_qps: 185

  refund:
    retry_rate: 0.003
    duplicate_refund_incidents: 0

这里仍然要区分：

配置上限；

压测观察值；

生产实际值；

团队口头认为的安全值。

它们不能混为同一个“限制”。

OpenAI 在安全分析 Agent 的公开实践中也强调，应在与项目真实环境匹配的沙箱中验证潜在问题，以区分静态推测和真实系统影响。(OpenAI)

八、第四阶段：历史现实采集

大型存量系统中的重要事实往往不是“现在代码怎么写”，而是：

为什么现在会写成这样？

需要查找：

历史 PR；

commit message；

ADR；

issue；

RFC；

incident report；

rollback 记录；

废弃实现；

TODO 和 FIXME；

feature flag 历史。

例如 Agent 看到：

单笔取消逻辑里存在看起来多余的二次状态检查。

如果只看当前代码，它可能认为可以简化。

但历史事故可能表明：

该检查用于防止支付回调与取消请求并发时重复退款。

历史证据的产物应是简短的因果摘要：

historical_context:
  - decision:
      statement: cancellation performs status validation twice
      source: ADR-021
      reason: prevent race with payment callback
      introduced_by: PR-1842
      related_incident: INC-2024-091
      current_relevance: likely_still_valid

历史记录可能过期，因此只能证明“当时为何这样决定”，不能自动证明它现在仍合理。

九、第五阶段：采集团队的隐性知识

并非所有现实知识都存在于仓库。

常见隐性知识包括：

某个服务理论上归 A 团队，实际由 B 维护；

某个接口写着支持重试，但运营从不重试；

某项配置不能调整，因为依赖方有未公开限制；

某段代码看似重复，实际是为了特定客户兼容；

某个测试环境与生产拓扑差异很大；

某个指标长期不可靠；

某个下游团队近期正在迁移。

建议 Agent 在完成初步探索后生成定向问题，而不是让专家“介绍一下系统”。

不够好的问题：

订单取消模块还有什么需要注意的吗？

更好的问题：

代码显示库存回滚通过异步事件完成，但测试只验证了事件成功发送。
生产环境是否存在事件延迟或丢失后的人工补偿流程？

人的回答也不能直接标记为客观事实，应标记来源：

expert_claim:
  statement: production users never submit more than 50 cancellations
  source:
    person: operations-lead
    captured_at: 2026-07-16
  evidence_status: unverified_human_report
  follow_up:
    - query audit logs for batch-size distribution

这可以防止“资深工程师记得如此”自动升级为系统事实。

十、建立统一的证据分类体系

建议 Reality Plane 使用以下七种状态。

类型
定义
示例
Observation
通过代码执行、查询或测量直接获得
过去 14 天 P95 为 126 ms
Artifact Fact
从版本化 artifact 中直接读取
schema 中字段为 non-null
Source Claim
某个文档或人声称如此
ADR 说当时不采用队列
Inference
基于多条证据推出
当前批量实现可能压垮库存服务
Assumption
为推进工作暂时接受
首版最大批量设为 100
Conflict
两个来源给出不一致结论
配置写 200 QPS，运行手册写 150
Unknown
当前没有足够证据
退款接口是否跨区域幂等

关键规则：

Source Claim ≠ Observation
Inference ≠ Fact
Assumption ≠ Decision
Absence of evidence ≠ Evidence of absence

十一、Evidence Ledger

建议使用结构化 evidence ledger，而不是只写一篇调查总结。

evidence_version: 4
feature_id: ORDER-184
base_commit: a83d91c

items:
  - id: EV-001
    statement: existing single-order API is idempotent at request level
    classification: artifact_fact
    source:
      type: code_and_test
      references:
        - src/order/cancel-controller.ts:88
        - tests/cancel-idempotency.spec.ts
    scope:
      api: POST /orders/{id}/cancel
      environment: repository
    confidence: high
    freshness:
      commit: a83d91c
    verified_by:
      - repository-analyst-agent
      - integration-test-run

  - id: EV-002
    statement: refund service itself guarantees idempotency
    classification: unknown
    conflicting_sources:
      - docs/refund-api.md says true
      - client retry code assumes false
    required_action:
      - inspect refund service contract
      - ask payment owner

  - id: EV-003
    statement: inventory service can safely accept 180 QPS
    classification: observation
    source:
      type: load_test
      run_id: inventory-load-842
    confidence: medium
    limitations:
      - staging topology differs from production

这份 ledger 应成为 Search、Evaluation 和 Decision Plane 的输入。

十二、证据需要附带五个关键属性

每条重要证据至少应附带：

属性
问题
Provenance
它来自哪里？
Version
对应哪个 commit、schema 或文档版本？
Time
在什么时间采集？
Scope
在什么环境、租户、流量和配置下成立？
Confidence
证据有多直接、是否可复现？

NIST AI RMF 强调，可信系统需要围绕数据质量、文档、验证、监控和 provenance 建立持续机制，而不是仅依靠一次性结论。(NIST出版物)

不建议使用过于精确的“事实概率”

例如：

confidence: 83.7%

除非这个数字有真实统计模型支持。

团队内更适合使用：

high
medium
low
disputed
unknown

并记录原因。

十三、冲突处理机制

Reality Plane 不应强行把所有来源融合成一个结论。

例如：

README：
库存服务最大支持 200 QPS。

运行手册：
建议不要超过 150 QPS。

最近压测：
180 QPS 开始出现 P99 抖动。

生产监控：
峰值曾达到 172 QPS，但只持续 30 秒。

错误处理方式：

综合判断上限约为 180 QPS。

这会隐藏不同事实的含义。

正确方式是保留冲突：

conflict:
  topic: inventory_safe_qps

  sources:
    - value: 200
      type: configured_limit
    - value: 150
      type: operational_recommendation
    - value: 180
      type: staging_observation
    - value: 172
      type: production_peak

  unresolved_question:
    what sustained production QPS is safe?

  temporary_decision:
    design budget: 150 QPS

  decision_owner:
    inventory-service-owner

Reality Plane 负责暴露冲突。

Spec 或 Decision Plane 负责选择采用哪个约束。

十四、构建 Task Reality Model

Evidence Ledger 是原始证据集合；Reality Model 是面向任务的结构化现状。

对于批量取消需求，可以输出五类模型。

1. 模块边界图

Order UI
   ↓
Order API
   ↓
Cancellation Application Service
   ├─ Permission Service
   ├─ Order Repository
   ├─ Refund Service
   ├─ Inventory Event
   └─ Audit Log

2. 状态机

CREATED
  ↓
PAID
  ├─→ SHIPPED
  └─→ CANCELLING
          ├─→ CANCELLED
          └─→ CANCEL_FAILED

标记：

合法转换；

非法转换；

可重试转换；

不可逆转换；

并发竞争点。

3. 副作用图

取消订单
├─ 修改订单状态
├─ 发起退款
├─ 回滚库存
├─ 发布事件
├─ 记录审计
└─ 发送通知

4. 一致性模型

订单状态：数据库事务
退款：同步远程调用
库存：异步最终一致
通知：best effort

5. 风险热点图

高风险：
- 重复退款
- 状态竞争
- 事件重复消费

中风险：
- 下游限流
- 批量响应过大

低风险：
- UI 排序

这些模型不是新的业务决策，而是当前证据的任务化组织。

十五、Reality Readiness Gate

Search Plane 开始正式方案设计前，建议设置 Reality Gate。

三种状态

状态
含义
Insufficient
关键现实未知，不能可靠设计
Sufficient for Exploration
可以做原型和方案探索
Sufficient for Implementation
关键事实和风险边界已足够明确

Gate 检查项

reality_readiness:
  relevant_modules_identified: true
  primary_execution_path_traced: true
  side_effects_identified: true
  data_and_state_model_understood: true
  runtime_baseline_collected: true
  critical_dependencies_identified: true
  historical_constraints_checked: true
  blocking_conflicts_resolved: true
  evidence_freshness_acceptable: true
  remaining_unknowns_documented: true

不要求全部事实都确定。

要求的是：

剩余未知不会导致当前方案建立在错误架构前提上。

十六、什么情况下必须停止并请求人确认

Agent 在 Reality Plane 中遇到以下情况，应停止自动升级结论：

情况
原因
代码、文档和生产指标冲突
不能自行判断哪个代表业务真相
发现未记录的跨团队依赖
需要责任方确认
无法判断某段逻辑是否遗留兼容代码
删除或复用风险高
需要访问敏感生产数据
受 Governance Plane 约束
关键指标缺失或明显不可靠
无法形成有效 baseline
历史决策依据已经失效，但替代规则未知
属于业务或架构决策
用户口述与系统行为明显不一致
需要确认是 Bug 还是需求误解
证据只能支持相关性，需求却要求因果结论
需要实验设计

Agent 应输出具体阻塞：

blocking_unknown:
  question: refund service是否具备跨请求幂等性？
  impact:
    - batch retry design
    - transaction boundary
    - acceptance tests
  evidence_checked:
    - client code
    - API docs
    - integration tests
  conflict:
    docs_claim_true_but_client_adds_deduplication
  required_owner:
    payment-service-owner

十七、Reality Plane 与 LLM 推理可靠性的关系

前面的讨论已经指出，LLM 不能被假设为稳定的事实推导器。

所以 Reality Plane 的架构不能是：

所有材料
→ LLM 阅读
→ 输出“系统事实”

应当是：

代码与配置
→ 确定性解析

测试和命令
→ 可执行结果

日志与指标
→ 查询和统计工具

Git 历史
→ 版本化记录

专家访谈
→ 来源化主张

以上证据
→ LLM 整理、关联和发现缺口
→ 规则校验和人工确认
→ Reality Pack

LLM 更适合：

找相关入口；

生成调查假设；

连接分散证据；

发现矛盾；

总结调用链；

提出缺失问题；

将复杂证据组织为任务模型。

确定性工具更适合：

解析依赖；

查询 schema；

计算指标；

执行测试；

检查 commit；

验证文件引用；

检查证据是否存在；

判断数据是否过期。

十八、人与 AI 的职责边界

活动
AI 负责
人负责
调查范围
根据 Spec 生成问题清单
确认调查优先级
仓库探索
搜索、建图、追踪调用路径
确认特殊或隐性路径
运行数据分析
生成查询、汇总结果、识别异常
批准敏感访问并解释业务背景
历史调查
查找 PR、ADR 和事故
判断历史理由是否仍有效
隐性知识采集
生成定向问题和结构化记录
提供并确认领域知识
证据分类
标记事实、主张、推断和未知
解决高影响争议
冲突发现
自动比较不同来源
决定采用哪个约束或发起验证
Reality Model
生成模块图、状态机和风险图
审核关键系统模型
Readiness
自动检查证据完整度
确认是否可以进入实现阶段

核心原则：

AI 可以发现和组织证据，但不能单方面宣布存在争议的系统现实。

十九、仓库内最小实现

对单团队，不建议建设统一知识平台。

可以使用：

repo/
├── AGENTS.md
├── specs/
│   └── ORDER-184-batch-cancel/
│       ├── spec.md
│       ├── acceptance.yaml
│       └── reality/
│           ├── summary.md
│           ├── evidence.yaml
│           ├── conflicts.md
│           └── baseline.yaml
├── docs/
│   ├── architecture/
│   ├── domain-rules/
│   ├── runbooks/
│   └── incidents/
├── scripts/
│   ├── repo-map.sh
│   ├── collect-baseline.py
│   └── verify-evidence.py
└── .github/
    └── workflows/
        └── reality-check.yml

GitHub 当前支持仓库级和路径级自定义指令，也支持通过仓库索引提高 Agent 的代码库理解质量；对单团队来说，这些仓库内机制通常已经足够作为 Reality Plane 的基础。(GitHub Docs)

二十、可以自动检查什么

reality-check.yml 不需要验证“模型理解是否正确”，而是检查基本证据完整性。

例如：

checks:
  - evidence references point to existing files
  - referenced commit exists
  - runtime evidence includes collection timestamp
  - production claims include environment field
  - high-confidence claims have direct evidence
  - conflicts are not silently marked resolved
  - blocking unknowns are empty before implementation
  - baseline is newer than configured freshness threshold
  - spec version matches reality pack version

还可以禁止：

“代码看起来……”
“应该是……”
“大概……”
“根据常见做法……”

直接以 confirmed_fact 类型出现。

二十一、证据时效策略

不同证据的保质期不同。

证据类型
建议时效
当前 commit 中的代码结构
base commit 变化后重新确认
API schema
schema 或依赖版本变化后重新确认
运行指标
通常 7-30 天，根据流量波动决定
性能 benchmark
硬件、依赖或实现变化后失效
feature flag 状态
每次发布前重新查询
人工口述
尽快用系统证据验证
历史 ADR
原因长期有效，但适用性需重新判断
事故记录
事实长期有效，缓解措施可能已变化

建议不要简单设置统一的“30 天过期”。

应根据来源类型和用途判断。

二十二、避免 Reality Plane 退化成什么

1. 不要变成全仓库知识图谱项目

只建当前任务需要的模型。

2. 不要把文档当最高权威

文档是一个来源，不是现实本身。

3. 不要把生产日志直接塞进 LLM

先过滤、聚合、脱敏和结构化。

4. 不要让 Agent 无限制探索

调查范围由 Spec 和风险驱动。

5. 不要追求“所有事实都确认后再开发”

只需确认那些会改变方案或验收结果的事实。

6. 不要让 Reality Plane 决定产品需求

它描述现状，不替代 Spec Plane 的业务选择。

7. 不要自动消解来源冲突

冲突本身就是重要信息。

8. 不要只保留最终摘要

应保留来源、版本、时效和适用范围。

二十三、分阶段落地

Phase 1：Repository Reality

先只做：

相关代码路径；

测试；

schema；

依赖；

Git 历史；

Facts / Inferences / Unknowns；

Evidence Ledger。

适用于暂时无法开放生产数据的团队。

Phase 2：Runtime Reality

增加：

当前指标；

日志查询；

trace；

流量分布；

配置状态；

生产 baseline。

Phase 3：Human Knowledge Capture

增加：

模块 owner 定向确认；

历史决策；

运维经验；

组织依赖；

冲突解决记录。

Phase 4：自动新鲜度与证据校验

增加：

commit 漂移检测；

配置变化检测；

evidence reference 校验；

baseline 过期提醒；

Spec 变化后的重新调查任务。

二十四、衡量 Reality Plane 是否有效

指标
含义
Blocking unknown discovery rate
实现前发现多少关键未知
Evidence-backed fact ratio
关键现状结论中有证据的比例
Stale evidence rate
使用过期证据的比例
Reality-model correction rate
人工需要纠正模型现状理解的频率
Hidden dependency discovery
实现前发现的隐性上下游数量
Late architecture surprise rate
实现后期才发现关键现实约束的频率
Production mismatch rate
staging 或仓库结论与生产不一致的比例
Human clarification efficiency
每次专家沟通解决多少关键问题
Investigation lead time
Reality Pack 形成所需时间
Rework caused by incorrect assumptions
因现状理解错误造成的返工
Evidence reuse rate
已确认现实资产被后续需求复用的比例
Conflict resolution time
高影响证据冲突的解决时间

目标不是增加调研时间，而是降低：

$$
\text{错误前提}
+
\text{隐藏依赖}
+
\text{生产意外}
+
\text{后期返工}
$$

二十五、Reality Plane 向后续 Plane 输出什么

下游 Plane
获取内容
Spec Plane
已确认事实、需要修正的假设和新增约束
Orchestration Plane
涉及模块、依赖关系、专家 owner 和风险热点
Search Plane
现有架构、可复用能力、技术边界和历史失败方案
Evaluation Plane
baseline、真实数据分布、事故场景和环境限制
Decision Plane
候选方案所依赖事实、证据强度和残余未知
Delivery Plane
生产指标、灰度范围、告警阈值和回滚条件
Learning Plane
新发现的领域规则、证据资产和失效假设

最终建议

对单团队，Reality Plane 最适合采用：

$$
\boxed{
\text{需求驱动的调查问题}
+
\text{多源证据采集}
+
\text{Facts / Claims / Inferences / Unknowns 分离}
+
\text{来源与时效追踪}
+
\text{任务化现实模型}
+
\text{冲突显式保留}
+
\text{人工确认 Gate}
}
$$

其最终输出不应是一句：

“Agent 已经理解了仓库。”

而应是一份可审计的声明：

我们基于哪个 commit、哪些代码路径、哪些测试、
哪些运行数据和哪些专家确认，认为当前系统如何运行；

哪些结论仍存在冲突；

哪些未知会影响方案；

当前证据足以支持哪些决策，又不足以支持哪些决策。

Reality Plane 的核心价值是：

把 Agent 对系统的“语义理解”，转化为一组有来源、有范围、有时效并允许被反驳的现实证据。

Sources:

Harness engineering: leveraging Codex in an agent-first ...

Using GitHub Copilot to explore a codebase

Artificial Intelligence Risk Management Framework (AI RMF 1.0)
