# 未解决的工程问题

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/15 15:03:52

---

## Prompt

除此之外，还有哪些实际的工程问题亟待解决？

---

## Response

我会按“尚未被主流 coding agent 体系充分解决”的角度补充，重点放在可观测性、权限、安全、状态一致性、成本、评测污染和长期维护等工程问题。

搜索2026年编程代理工程挑战的最新研究

搜索2026年编程代理工程挑战的最新研究

思考了 22s

除去“多分支如何取舍”和“开放式需求如何评测”，当前 coding agent 真正亟待解决的问题，主要集中在以下 12 类。

优先级
工程问题
为什么难
典型解决方向
P0
早期错误检测
Agent 往往在最初几步就形成错误假设，但直到大量修改后才暴露，届时回滚成本很高。最新轨迹研究发现，失败常由早期认知错误引发，并长期隐藏。(arXiv)
先复现再修改；阶段性 checkpoint；每个关键假设都要求证据；限制无验证连续操作步数。
P0
权限与爆炸半径控制
Agent 能运行 shell、访问网络、修改依赖、操作云资源，能力越强，误操作后果越大。Anthropic 将 containment 和 blast radius 视为核心工程问题。(Anthropic)
最小权限、sandbox、网络白名单、临时凭证、工具分级审批、生产环境只读、自动撤销权限。
P0
规格不完整与隐性需求
多 Agent 即使共享代码，也可能因接口语义、状态模型和错误协议理解不同而生成无法整合的实现；仅报告冲突不足以恢复，完整规格更关键。(arXiv)
可执行 spec、schema、ADR、不变量、错误语义、ownership、兼容矩阵；冲突时回到规格而不是直接让 merger 猜。
P0
Verifier 可信度
Agent 的能力上限受评测器限制。错误测试、弱 benchmark 或可被投机的规则，会把系统引向错误目标。(Anthropic)
隐藏测试、独立测试设计、反例生成、多重 verifier、评测完整性校验、线上指标校准。
P1
多 Agent 状态一致性
并发修改会互相干扰，任务依赖难同步，局部正确结果不一定能合成全局正确系统。(arXiv)
中央任务图、隔离 worktree、版本化 artifact、结构化 handoff、集成队列、依赖感知调度。
P1
长期任务的记忆与上下文衰减
长任务跨多个 session，旧决策、失败路径和关键约束容易丢失；原始上下文又不能无限保留。(Anthropic)
分层记忆、决策日志、状态摘要、上下文压缩、可检索项目知识、关键约束持续重注入。
P1
可观测性与故障诊断
只知道“Agent 失败了”没有用，需要知道在哪一步、基于什么上下文、调用了什么工具以及为何没有恢复。OpenAI 的 orchestration 规范和相关工程岗位都把 observability、profiling、diagnostics 列为重点。(OpenAI)
完整 trace、事件时间线、工具 I/O、上下文快照、成本归因、失败分类、可回放执行。
P1
候选搜索的成本控制
多分支、完整测试、benchmark 和 reviewer 会迅速放大 token、计算与 CI 成本。
分层评测漏斗、successive halving、受影响测试选择、缓存构建产物、预算感知搜索、动态停止。
P1
长期可维护性而非一次通过
静态修 Bug benchmark 很难反映数月演进中的架构侵蚀、技术债和后续修改成本。SWE-CI 开始用连续提交历史评估长期维护能力。(arXiv)
多轮演进评测、后续需求测试、架构熵指标、变更成本、回归率、技术债预算。
P1
人机责任与审批边界
哪些决定能自动完成，哪些必须由人批准，尚无统一答案。不同组织对安全、速度和合规的容忍度不同。(OpenAI)
风险分级授权；低风险自动合并，高风险人工审批；Agent 不能批准自己的 PR；明确责任人。
P2
评测与生产环境偏差
离线测试通过不代表真实流量、长尾数据、异构设备和复杂依赖下仍然可靠。
shadow traffic、canary、feature flag、SLO guardrail、自动回滚、生产失败样本回灌。
P2
Agent 资产自身的版本治理
Prompt、skills、规则、模型、工具协议和评测集都会变化，一次升级可能改变整个系统行为。
为 model、prompt、skill、tool、eval、spec 分别版本化；建立兼容矩阵、回归测试与逐步发布。

1. 最核心的未解问题：如何尽早发现“方向错了”

目前大量系统仍采用：

理解需求
→ 连续修改几十步
→ 最后运行测试

真正可靠的流程应改成：

提出假设
→ 最小验证
→ 继续一小步
→ 再验证

例如 Agent 声称：

Bug 是由缓存更新缺少锁造成的。

系统不应马上允许它修改十个文件，而应要求：

hypothesis:
  shared cache mutation is unordered

required_evidence:
  - reproduce race
  - identify two conflicting writers
  - produce trace or failing test

next_action_budget:
  max_files_changed: 2
  max_tool_calls_before_validation: 5

这相当于为 Agent 引入“认知单元测试”。

2. 如何判断 Agent 是否真正理解了需求

当前 Agent 很容易把“生成了合理计划”误当成“理解正确”。

工程上可加入三类检查：

反向解释

让 Agent 在修改前回答：

哪些行为必须保持不变？
哪些行为必须改变？
最可能被破坏的三个下游是什么？

反例测试

让独立 Agent 寻找：

什么输入能证明当前理解是错的？

规格一致性检查

把计划与需求做结构化比对：

每个 requirement 是否对应：
- 实施步骤
- verifier
- 发布证据

没有 verifier 的需求应被标记为未闭环。

3. 多 Agent 的通信协议仍然过于自然语言化

很多系统让 Agent 互相传递一段总结：

“我已经完成后端修改，请前端适配。”

这非常脆弱。更可靠的交付应是结构化 artifact：

handoff:
  producer: backend-agent
  commit: a31fd2
  contract_version: users-api-v4

  changed_interfaces:
    - GET /users
    - UserSearchResponse

  guarantees:
    - cursor is stable for 10 minutes
    - results are sorted by created_at desc

  unresolved:
    - retry semantics for 429

  required_consumer_actions:
    - update generated types
    - handle next_cursor=null

Agent 间协作应逐渐接近编译器和构建系统，而不是群聊。

4. 谁来监督监督者

引入 reviewer agent 后会出现递归问题：

谁验证 reviewer？
谁验证 verifier？
谁验证测试本身？

不能无限堆 Agent。实际应采用不同可信层级：

确定性规则
    高可信：编译器、类型系统、形式约束

执行测试
    中高可信：单测、集成、benchmark

独立 LLM reviewer
    中等可信：架构、可维护性、开放式质量

人类
    处理高风险、模糊价值判断和最终责任

原则是：

越关键的约束，越应该下沉到确定性机制，而不是交给另一个 LLM 判断。

5. 如何避免 Agent 制造“可维护但没人理解”的系统

Agent 可以快速生成大量抽象层、配置、测试和文档，但这些资产未必符合团队真实心智模型。

需要评估：

是否沿用现有架构；

是否引入新概念；

新概念是否必要；

团队是否具备维护能力；

发生故障时人类能否快速理解；

是否形成 Agent 才能维护的复杂系统。

建议增加一项 human comprehensibility budget：

maintainability_constraints:
  new_architectural_concepts: <= 1
  new_runtime_dependencies: 0
  new_configuration_layers: <= 1
  critical_path_documented: true
  rollback_steps_human_executable: true

6. Agent 生成速度与组织吞吐不匹配

Agent 可能一天产生几十个 PR，但：

reviewer 数量没增加；

CI 资源有限；

产品验收能力有限；

发布窗口有限；

下游团队无法同步适配。

结果是从“开发瓶颈”转化为：

评审瓶颈
CI 瓶颈
集成瓶颈
发布瓶颈
决策瓶颈

GitHub 对异步 agent 工作流的讨论也强调，收益需要覆盖 review、安全和治理层，而不只是代码生成。(The GitHub Blog)

因此需要建立 Agent 工作的 WIP 限制：

agent_capacity:
  max_open_agent_prs_per_team: 5
  max_high_risk_tasks_in_flight: 1
  review_budget_per_day: 3
  auto_close_stale_candidates_after: 48h

7. 如何衡量 Agent 的真实经济价值

不能只看：

生成代码行数
完成任务数量
SWE-bench 分数

更有价值的指标是：

$$
\text{净收益}

\text{节省的工程时间}

\text{评审成本}

\text{返工成本}

\text{运行成本}

\text{事故预期损失}
$$

建议记录：

指标
含义
Accepted work rate
真正被采用的工作比例
Human correction time
人类修正耗时
Review burden
每个 PR 的评审时间
Rework rate
合并前后返工比例
Rollback rate
生产回滚比例
Time to trusted result
从任务开始到结果可信的时间
Cost per accepted change
每个有效变更的总成本
Defect escape rate
未被评测发现而进入生产的问题

Agent 的目标不应是“产生更多代码”，而应是“降低可信变更的单位成本”。

建议优先解决顺序

对准备在真实团队大规模引入 coding agent 的组织，我会建议按这个顺序建设：

1. 权限与隔离
2. 可执行规格与硬约束
3. 独立 verifier
4. 完整 trace 与可回放
5. 早期假设验证
6. worktree 与结构化 handoff
7. 分层评测和成本预算
8. canary、监控和自动回滚
9. 长期维护评测
10. Agent 资产版本治理

最终可以概括为：

当前最缺的已经不是更强的代码生成能力，而是一个能约束、观察、验证、协调、回滚并持续改进 Agent 的工程控制系统。

Sources:

Failure as a Process: An Anatomy of CLI Coding Agent Trajectories

How we contain Claude across products

An open-source spec for Codex orchestration: Symphony.

GitHub recognized as a Leader in the Gartner® Magic ...
