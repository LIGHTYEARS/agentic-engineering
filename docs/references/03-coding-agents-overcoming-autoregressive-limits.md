# 2026年 Coding Agent 如何突破自回归局限

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/14 16:16:32

---

## Prompt

现在（2026年7月）最新业界的 coding agent 在工程上具体是如何解决这个问题呢？你需要做 deep research 后再回答我。

---

## Response

我会从公开的一手资料入手，重点拆解 2026 年主流 coding agent 的工程架构：规划、并行候选、工具闭环、评测与回滚，以及它们如何缓解自回归的单路径问题。

初步资料显示，业界并没有“修复自回归本身”，而是把模型包进一个复合系统：用计划、隔离分支、测试反馈、审查代理和并行工作树，把单次生成改造成反复搜索与验证。接下来我会区分哪些能力是真正已落地，哪些仍是营销表述。

搜索与编码代理相关的官方文档

搜索与代理相关的多个官方网站及资源

搜索官方文档和基准测试

搜索多个文档和网站资源

思考了 39s

截至 2026 年 7 月，主流 coding agent 并没有从根本上把 LLM 从“单向自回归模型”改造成 NSGA-II 那样的多目标优化器。

业界真正采用的办法是：

保留自回归模型作为方案生成器，再在模型外部搭建一个可执行、可回滚、可并行、可验证的搜索系统。

换句话说，他们不是要求一个 LLM 在“脑海里”同时权衡十个目标，而是把这些目标外化为：

计划和任务状态；

独立分支或 worktree；

测试、lint、benchmark 和安全扫描；

多个并行 agent；

reviewer、verifier 或 critic；

PR、CI 和人工审批。

这类外部系统现在常被称为 agent harness、scaffolding、agent runtime 或 compound AI system。2026 年的一篇终端 coding agent 论文也明确把可靠系统描述为规划、执行、上下文管理、工具调用和记忆等组件组成的复合架构，而非单次 LLM 调用。(arXiv)

一、核心变化：从“生成一份答案”变成“运行一个闭环”

早期代码生成接近：

需求
  ↓
LLM 一次生成代码
  ↓
用户检查

现在的 coding agent 更接近：

读取仓库
  ↓
构建任务模型和实施计划
  ↓
修改代码
  ↓
运行编译、测试、lint、类型检查
  ↓
读取错误和性能指标
  ↓
修正代码
  ↓
再次验证
  ↓
提交独立分支或 PR
  ↓
其他 agent / 人类审核

可以把它写成一个外部优化循环：

$$
x_{t+1}

\operatorname{Agent}
\left(
x_t,
\operatorname{Feedback}(x_t),
\operatorname{Constraints}
\right)
$$

其中：

$x_t$ 是第 $t$ 版代码；

Feedback 是测试失败、编译错误、benchmark、review comment 等；

Constraints 是架构规则、性能预算、API 兼容要求和安全策略。

这仍然是自回归生成，但每轮生成之间插入了真实世界反馈。模型不必凭语言直觉估计所有目标，而是可以读取工具测量出的结果。

二、业界主要用了七层工程机制

1. 先规划，再写代码：把隐藏目标外化成显式状态

主流产品现在普遍加入了独立的 research / plan 阶段。

GitHub Copilot cloud agent 会先研究仓库、创建实施计划，然后才在独立分支中修改代码。(GitHub Docs)

Devin 的 Interactive Planning 会先搜索代码库，定位相关文件和代码，再提出可由用户调整的详细计划；其 Ask Devin 也明确将“探索架构、规划任务、生成高上下文实施提示”作为进入执行 agent 前的一步。(Devin Docs)

这解决的不是所有优化问题，但它会把原本散落在提示词里的目标变成类似：

acceptance_criteria:
  - eliminate race condition
  - preserve public API
  - no QPS regression > 3%
  - additional RSS < 512 MB
  - all integration tests pass

implementation_steps:
  - reproduce race
  - identify shared mutable state
  - compare lock / CAS / sharding alternatives
  - implement selected option
  - benchmark
  - run compatibility tests

这种计划是 agent 的外部工作记忆。生成到第 100 行时，系统仍然可以重新把这些目标注入上下文，而不是只依赖模型是否还“注意到”最初的要求。

但需要强调：

计划并不等于多目标优化。

计划主要解决目标遗忘和任务漂移，不能保证权衡正确。真正的权衡仍要依靠后面的测量和候选比较。

2. 用真实执行替代“凭感觉评分”

这是目前最重要的一层。

Coding agent 会直接调用：

编译器；

单元测试；

集成测试；

类型检查；

lint；

安全扫描；

浏览器；

benchmark；

profiler；

Git 和 CI。

Claude Code 的 Agent SDK公开描述了完整的 agent loop：读取文件、执行命令、搜索、编辑代码，并把环境输出重新交给模型。(Claude Platform Docs)

GitHub Copilot cloud agent 在解决冲突后会重新验证 build、tests 和 linter 是否通过，再请求人工审核。(GitHub Docs)

Devin 不仅能运行代码和测试，还支持启动应用，通过浏览器或桌面交互做端到端测试，并提供测试录像作为可审查证据。(Devin Docs)

于是，多目标中的一些目标会从自然语言变成可执行判断：

工程目标
外部 verifier
正确性
单元测试、属性测试、回归测试
兼容性
API contract test、集成测试
性能
benchmark、QPS、P95/P99
内存
RSS、heap profile、allocation count
代码质量
lint、复杂度规则、静态分析
安全
SAST、依赖扫描、权限策略
UI 正确性
浏览器 E2E、截图、视觉 diff

这一步非常关键，因为 LLM 很不擅长仅通过阅读代码准确判断“这个改动会让 QPS 降多少”。但它能理解一个真实输出：

baseline QPS: 84,230
candidate QPS: 52,411
regression: -37.8%
FAIL: allowed regression <= 3%

然后推翻当前方案。

从优化角度看，这相当于把模糊的目标函数：

$$
f_{\text{performance}}(x)
$$

变成可观测数值，而不是让模型猜测它。

3. 用 hooks、CI 和策略代码保证“某些检查绝不被忘记”

只靠提示模型“记得运行测试”仍然不可靠，因为模型可能：

忘记；

认为没有必要；

为了节省时间跳过；

错误地判断测试已足够。

因此业界开始把关键规则从提示词迁移到确定性代码。

Claude Code 的 hooks 可以在生命周期特定节点自动运行 shell、HTTP 或 LLM hook。Anthropic 明确强调：hooks 适合那些必须每次执行、不能依赖模型自行决定的规则，例如改动后自动运行测试、提交前执行 lint，或拦截危险命令。(Claude Platform Docs)

典型配置思想如下：

AfterFileEdit:
  - run formatter
  - run affected unit tests

BeforeCommit:
  - run typecheck
  - run lint
  - verify generated files are current

BeforeToolUse:
  - deny production database writes
  - deny destructive shell commands

OnStop:
  - require all acceptance checks

这相当于把部分目标变成硬约束：

$$
g_i(x) \leq 0
$$

例如：

tests_pass == true
lint_errors == 0
public_api_diff == empty
memory_growth <= 512MB

如果约束不满足，agent 不能宣称任务完成。

OpenAI 在 2026 年有关 harness engineering 的实践文章中也强调，需要尽早把架构边界和约束编码进仓库，否则 agent 的速度会带来架构漂移和质量衰减。(OpenAI)

4. 用 worktree、branch 和 sandbox 保存多个独立轨迹

普通自回归生成通常只沿一条轨迹前进：

方案 A → 修补 A → 再修补 A

2026 年的 agent 工具开始普遍支持并行的隔离工作区。

OpenAI 的 Codex 产品明确将 worktrees、cloud environments 和 parallel agents 作为多 agent 工作流的基础；Codex app 也强调可以同时管理多个 agent，让它们并行执行长任务。(OpenAI)

GitHub Copilot cloud agent 会在独立分支或 GitHub Actions 驱动的隔离环境中工作，再通过 draft PR 提交结果。(GitHub Docs)

GitHub 还为 cloud agent 提供 ephemeral、firewalled environment 和自动安全扫描，避免候选实现直接污染开发者主环境。(GitHub Docs)

这使系统能够真正维护多个候选：

worktree-lock/
  └─ 粗粒度锁方案

worktree-sharded-lock/
  └─ 分段锁方案

worktree-cas/
  └─ 原子状态机方案

worktree-copy-on-write/
  └─ 不可变快照方案

然后分别执行：

correctness tests
race detector
QPS benchmark
memory benchmark
compatibility suite

这已经开始接近多目标搜索，但仍与 NSGA-II 有明显区别：

通常候选数只有 2～10 个，而不是大种群；

候选生成主要靠 LLM 和人工任务分解；

没有统一的遗传交叉和变异机制；

选择通常仍由测试门槛、reviewer 或人类完成；

很少自动维护完整的帕累托前沿。

因此更准确的称呼是 small-N parallel search，而不是成熟的进化优化。

5. 用 subagent 分离职责，避免一个上下文同时承担所有目标

Claude Code 支持自定义 subagents，每个子代理有独立上下文、工具权限、提示、hooks 和 skills。(Claude Platform Docs)

Anthropic 还明确给出并行开发的例子：一个 subagent 构建后端 API，主 agent 构建前端；hooks 则自动运行测试和 lint。(Anthropic)

这种架构通常会把任务角色分开：

Planner agent
  └─ 分解任务和验收标准

Implementation agent
  └─ 编写代码

Test agent
  └─ 创建和运行测试

Security reviewer
  └─ 检查安全边界

Performance reviewer
  └─ 检查复杂度、锁竞争、内存

Integrator agent
  └─ 合并结果、解决冲突

这样做缓解了“一个上下文里要同时记住所有事情”的压力。

例如一个安全 reviewer 的上下文可以只包含：

任务目标
diff
威胁模型
权限边界
安全规则

而不是整个实施过程的上百条终端输出。

这里的核心不是“多 agent 天生更聪明”，而是：

职责拆分降低了每个决策点的目标维度，并让不同目标拥有独立的检查轨迹。

不过多 agent 也引入了新的问题：

两个 agent 同时修改同一个文件；

对共享接口作出不兼容假设；

重复工作；

合并冲突；

reviewer 被错误实现误导；

多个 agent 共享了相同盲点。

2026 年业界已经把 agent collision 和多 agent 管理视为新的工程问题，而不是免费的能力增益。(Business Insider)

6. 用 reviewer / verifier agent 做第二遍独立判断

当前较成熟的工作流越来越少让“写代码的 agent”自己给自己打满分。

常见做法是：

Writer agent
  ↓
Deterministic tests
  ↓
Review agent
  ↓
Security agent
  ↓
Human review

原因和人类工程一样：实现者容易锚定自己的方案。

OpenAI Codex 已覆盖代码审查、PR 和多 agent 工作流；GitHub Copilot cloud agent 通过 draft PR 和迭代评论将实现与审核分离。(OpenAI)

研究领域则更进一步发展出 verifier-driven 方法：生成多个候选，用执行结果或独立 verifier 比较候选，而不是让原始生成模型自行判断。2026 年的 Agentic Verifier 工作通过主动寻找能够区分候选程序的反例输入，在多个编程基准上报告了 Best@K 的明显提升。(arXiv)

另一项 2026 年研究提出联合查看多个候选的 multi-sequence verifier，而不是独立给每个候选打分；其目标正是提高 best-of-N 选择质量并减少并行采样的延迟。(arXiv)

在真实 coding agent 中，这通常表现为：

候选 A：测试全部通过，QPS -18%
候选 B：测试全部通过，QPS -1%，内存 +410MB
候选 C：测试失败 2 个，但 QPS +4%

verifier 不只是问“代码看起来好吗”，而是结合执行证据做选择。

7. 用上下文压缩、项目记忆和持续重注入对抗“目标衰减”

长任务还有一个问题：随着工具输出、日志和代码 diff 增多，原始需求会被淹没。

2026 年的 agent harness 通常会使用：

repository map；

语义代码搜索；

摘要；

context compaction；

持久项目说明；

AGENTS.md、CLAUDE.md、skills；

事件触发式提醒；

结构化任务列表。

OPENDEV 的 2026 年论文专门提到 adaptive context compaction、项目记忆和 event-driven system reminders，用来减少上下文膨胀和 instruction fade-out。(arXiv)

Claude Code 的 subagents 在独立上下文中执行后返回摘要，skills 还能动态注入项目规则和专门知识。(Anthropic)

Codex 的 Skills 也被定位为把团队标准、工作流和工程方式传给 agent，并让其跨任务一致执行。(OpenAI)

这相当于不断把目标重新放回“注意力前台”：

当前任务目标
已完成步骤
当前候选方案
未满足约束
最近测试结果
下一步行动

而不是无限保留所有原始终端日志。

三、一个真实工程化的多目标修 Bug 流程

以你前面的例子为准：

修核心结算模块的并发 Bug，同时不能损害性能、内存、兼容性和可维护性。

2026 年较成熟的 agent 工作流会这样设计。

第一步：把目标分成硬约束和软目标

硬约束

correctness:
  race_test: pass
  existing_tests: pass

compatibility:
  public_api_diff: none
  downstream_modules_pass: 15/15

resource:
  extra_memory_mb: <= 512

软目标

performance:
  qps_regression: minimize
  p99_regression: minimize

maintainability:
  complexity_delta: minimize
  changed_lines: minimize

这是一个非常重要的工程技巧。

不是给十个目标随便设权重：

$$
0.4 f_1 + 0.2 f_2 + 0.1 f_3 + \cdots
$$

而是先做 constraint satisfaction：

不满足正确性、兼容性或资源上限
→ 直接淘汰

再在可行解中比较性能和维护成本。

这比让 LLM自己决定“正确性和性能谁更重要”可靠得多。

第二步：先复现问题，建立 verifier

Agent 不立即修改代码，而是先：

写一个可稳定触发竞态的测试；

运行 race detector；

记录 baseline QPS；

记录 baseline 内存；

保存现有 API contract。

理想结果是：

race_reproduction: FAIL, 78/100 runs
baseline_qps: 85,200
baseline_p99: 41ms
baseline_rss: 1.3GB
api_contract_hash: abc123

没有 baseline，所谓“不能降低性能”是无法验证的。

这也是当前 agent 工程中最容易被忽略的一点：真正的瓶颈经常不是模型不会写代码，而是仓库缺少可靠 verifier。

第三步：并行产生少量候选实现

由 planner 或多个 subagent 生成：

A：粗粒度 mutex；

B：分段锁；

C：CAS；

D：请求合并；

E：copy-on-write。

每个候选在独立 worktree 或 branch 中运行。

第四步：分别执行统一评分协议

1. compile
2. unit tests
3. race test 1,000 runs
4. downstream integration tests
5. QPS benchmark
6. P99 benchmark
7. memory profile
8. static analysis
9. API diff

得到：

候选
正确性
QPS 变化
内存变化
兼容性
复杂度
A 全局锁
通过
-42%
+5MB
通过
低
B 分段锁
通过
-2.1%
+37MB
通过
中
C CAS
通过
+1.8%
+12MB
失败
高
D 请求合并
失败
+6%
+80MB
通过
高
E COW
通过
-0.8%
+690MB
通过
中

先按硬约束淘汰：

C：兼容性失败；

D：正确性失败；

E：内存超标。

剩下 A 和 B。

B 在性能上明显支配 A，因此选择 B。

这才是真正意义上的工程化多目标权衡。LLM 的作用主要是：

提出候选；

实施候选；

解读失败；

修正候选；

总结证据。

目标计算和淘汰则由工具系统完成。

四、这些系统真的在做“帕累托优化”吗？

多数情况下，没有严格做到。

商业 coding agent 的默认工作方式通常仍是：

单个 agent
→ 一个主要实现
→ 测试失败就修
→ 直到测试通过

并不是：

持续维护 50 个候选
→ 对 10 个目标计算完整指标
→ 非支配排序
→ 保持解集多样性
→ 交叉和变异
→ 收敛到帕累托前沿

当前业界更接近以下组合：

约束驱动搜索

先保证 tests、lint、安全和 API 兼容。

hill climbing

当前实现不通过，就根据反馈不断局部修正。

best-of-N

并行生成少量候选，然后选一个。

planner-executor-reviewer

规划者、执行者、审核者分工。

verifier-guided search

用可执行检查或独立 verifier 指导下一轮。

human-in-the-loop selection

最终由人类在 PR 中选择权衡。

研究系统正在探索更强的搜索。例如 SWE-TRACE 会在长轨迹中动态评估和剪枝动作候选，以减少普通并行采样的成本；SWE-Master 报告使用 test-time scaling 和环境反馈后，其 SWE-bench Verified 成绩从 61.4% 提高到 70.8%。(arXiv)

但这些研究机制并不意味着所有商业 agent 默认都在运行树搜索或 best-of-8。实际产品通常受限于：

token 成本；

VM 成本；

benchmark 时间；

合并复杂度；

候选验证质量；

用户等待时间。

五、当前最大的瓶颈已经从“生成”转移到“验证”

截至 2026 年，越来越明显的现实是：

写出五个候选并不难，困难的是判断哪个候选真的好。

例如性能目标可能要求：

稳定压测环境；

合理 warm-up；

多轮统计；

噪声控制；

与 baseline 对比；

真实流量模型。

兼容性目标可能要求：

下游 15 个仓库；

多版本依赖矩阵；

schema migration；

feature flag；

灰度发布。

安全目标可能要求：

威胁模型；

权限边界；

数据流分析；

运行时策略。

所以 agent 的上限越来越取决于仓库是否具有：

高质量测试
明确验收标准
可重复构建环境
稳定 benchmark
机器可读架构规则
快速 CI
可观察性

这就是为什么 2026 年“harness engineering”变得重要：同一个模型，放在差的仓库环境里会反复猜；放在强约束、强测试、强反馈的环境里，会表现得像完全不同的 agent。(OpenAI)

六、几个主流产品的侧重点

OpenAI Codex

目前公开强调的工程方向包括：

多 agent 并行；

worktree 和云环境隔离；

长时间后台任务；

Skills 注入团队规则；

IDE、CLI、云端统一；

PR、refactor、migration 和 code review；

企业级治理、遥测和安全边界。(OpenAI)

它主要通过“并行代理 + 隔离环境 + 工程规则 + 人类控制台”缓解单轨迹问题。

Claude Code

公开机制更偏向可组合 agent runtime：

plan / exploration 工作流；

subagents；

独立上下文；

skills；

hooks；

MCP；

Agent SDK；

context management；

权限控制；

后台任务。(Claude Platform Docs)

它的突出特点是方便团队把自己的 verifier 和确定性规则接入 agent 生命周期。

GitHub Copilot cloud agent

更强调与现有 SDLC 原生结合：

Issue 到 agent；

GitHub Actions 隔离环境；

自动创建 branch；

research 和 plan；

draft PR；

CI、build、test、lint；

review comment 迭代；

firewall、allowlist 和组织级策略。(GitHub Docs)

它不是另建一套优化平台，而是把 agent 嵌入 branch / PR / CI 这一套已有的人类验证流程中。

Devin / Devin Desktop

公开能力侧重：

仓库索引和高上下文搜索；

Interactive Planning；

长时间自主执行；

编写、运行、测试代码；

浏览器和桌面端到端测试；

测试录像；

多 agent command center。(Devin Docs)

它试图把“实现 + QA 证据”作为一个完整交付单元。

七、这种办法还没有完全解决什么

1. verifier 不完整时，agent 会优化错误指标

测试全通过并不等于需求正确。

Agent 可能：

修改测试以适配错误实现；

只通过已有测试；

漏掉未编码的产品需求；

在 benchmark 的特定输入上过拟合；

用缓存、硬编码或捷径欺骗 evaluator。

DeepSWE 等 2026 年新 benchmark 正在强调：历史 PR 自带测试可能只验证特定修法，可能错判替代实现或放过不完整实现。(arXiv)

2. 多 agent 可能是“同质化并行”

如果多个 agent：

使用相同模型；

接收相同提示；

读取相同上下文；

采用相同工具；

它们可能生成非常相似的候选。

真正提高多样性需要：

不同角色；

不同方案约束；

不同模型；

不同随机种子；

不同搜索策略；

对候选相似度进行惩罚。

3. reviewer 也可能犯错

LLM reviewer 不是形式证明器。

它容易：

偏好写得更漂亮的方案；

被长解释说服；

漏掉隐蔽回归；

与 writer 共享相同错误先验。

因此 reviewer agent 最可靠的用途是补充确定性测试，而不是替代测试。

4. 真实多目标通常无法在 PR 阶段完全测量

例如：

生产流量下的锁竞争；

一周后的缓存命中率；

灰度环境中的尾延迟；

长期内存碎片；

运维复杂度；

团队维护成本。

这些需要部署后观测、canary、feature flag 和自动回滚。

因此完整闭环应继续延伸：

code agent
→ CI
→ PR
→ staging
→ canary
→ production telemetry
→ rollback / follow-up task

5. 长任务可靠性仍不稳定

METR 的 time-horizon 研究显示，前沿 agent 的可完成任务长度持续增长，但其指标代表的是某类任务难度下的成功概率，不等于 agent 能在该时长内稳定自主工作；METR 也明确提醒，超过 16 小时区域的测量目前不够可靠。(METR)

而 METR 在比较 Codex、Claude Code 与较简单 scaffold 时，也没有发现这些产品 scaffold 在其测试上具有统计显著的普遍优势。这说明复杂 harness 很重要，但并非所有任务上都自动胜出。(METR)

八、从 Tech Lead 视角，最有效的落地方式

对大型前端或全栈代码库，不应把任务写成：

修复这个并发 Bug，保持性能和兼容性。

更好的 agent contract 是：

task:
  fix: checkout settlement race condition

reproduction:
  command: pnpm test settlement-race --repeat=100
  current_failure_rate: "> 20%"

hard_constraints:
  - all existing tests pass
  - no public TypeScript API changes
  - downstream contract suite passes
  - additional memory < 512MB
  - no new runtime dependency

performance_budget:
  qps_regression: "<= 3%"
  p99_regression: "<= 5%"

required_evidence:
  - before/after race test
  - before/after benchmark
  - memory profile
  - API extractor diff
  - explanation of rejected alternatives

workflow:
  - reproduce before editing
  - propose at least 3 approaches
  - implement 2 viable candidates in separate worktrees
  - run the same evaluation suite
  - recommend one based on evidence
  - open a draft PR only

配套仓库能力应包括：

AGENTS.md / CLAUDE.md / Skills
architecture tests
API Extractor
affected-test selection
benchmark harness
Playwright E2E
visual regression
bundle-size budget
dependency policy
CI required checks
preview environment
canary and rollback

对于前端项目，多目标 verifier 可以具体落成：

目标
机器检查
功能正确
Vitest/Jest、Playwright
类型兼容
tsc --noEmit、API Extractor
bundle 体积
size-limit、bundle analyzer
性能
Lighthouse、Web Vitals、交互 benchmark
视觉一致
screenshot diff
可访问性
axe、eslint-plugin-jsx-a11y
依赖风险
lockfile diff、license/security scan
架构边界
dependency-cruiser、Nx boundaries
代码规范
ESLint、Prettier
浏览器兼容
多浏览器 E2E

这会比“换一个更聪明的模型”更稳定地提升 agent 交付质量。

九、最终判断

2026 年 7 月业界对自回归局限的工程解法，可以概括为四句话：

把目标写出来：通过 plan、spec、task state 和项目规则，减少目标遗忘。

把目标测出来：通过测试、benchmark、profiler、CI 和生产遥测，减少模型猜测。

把候选保留下来：通过 branch、worktree、sandbox 和并行 agents，减少单路径依赖。

把选择交给 verifier：通过硬约束、独立 reviewer、执行证据和人类审批，减少自我评价偏差。

所以现代 coding agent 不是：

一个特别强的 LLM 独自在脑中解决多目标非凸优化。

而是：

LLM 生成候选，工具测量结果，harness 保持状态，多个 agent 扩展搜索，CI 和人类完成选择。

严格来说，它们还没有普遍实现真正的 NSGA-II 式帕累托搜索；但已经通过 small-N 并行候选 + verifier-driven iteration + constraint gating，把原本的一条自回归轨迹改造成了一个有限但实用的工程搜索过程。

TechRadar

TechRadar

Business Insider

Sources:

Building AI Coding Agents for the Terminal: Scaffolding, Harness, Context Engineering, and Lessons Learned

About GitHub Copilot cloud agent

Interactive Planning

Agent SDK overview - Claude Code Docs

Harness engineering: leveraging Codex in an agent-first ...

Enabling Claude Code to work more autonomously

I went to an AI conference and got a crash course in middle management

Task-Completion Time Horizons of Frontier AI Models
