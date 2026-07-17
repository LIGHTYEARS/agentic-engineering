# 多分支合并策略与开放式评测

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/15 13:33:38

---

## Prompt

有很多实际的工程问题需要解决，我举两个例子你延伸如调研分析思考：

多分支并行去做的时候，如果一个分支在某方面表现出优势，另一个分支在另外一些方面表现出优势，工程上面对这种情况的策略该如何？

针对一些 open ended 问题，工程上该如何构建评测集？尤其是一些需求实现可能每次都不太一样的时候。

and eg...

---

## Response

我会把问题扩展成一组“Agent 工程化难题”，重点分析多分支融合、开放式任务评测、评测漂移、成本控制、冲突治理和上线反馈，并给出可落地策略。

进行深入研究与查阅技术文献

寻找全面的答案

我需要围绕8个问题提供一个全面的回答，特别关注开放性评估。为了做这个，我打算查找关于SWE-bench、R2E gym、代理评估、变形测试、隐性测试和LLM判定可靠性的相关论文。希望这能帮助我更好理解这些领域的最新进展。

搜索关于SWE-bench、R2E gym等论文

已思考 16s

你的两个问题正好触及 coding agent 从 Demo 走向生产时最困难的部分：

多个候选各有优势时，如何组合或选择？

答案并不唯一时，如何判断实现是否足够好？

它们背后其实是同一个问题：

工程系统需要评估的不是“Agent 是否写出了标准答案”，而是“它是否在约束条件下产生了可接受、可解释、可持续演进的结果”。

以下把它扩展成一套更完整的工程方法。

一、多个分支各有优势，工程上怎么处理？

先看一个候选矩阵：

候选
正确性
QPS
内存
兼容性
复杂度
A 全局锁
通过
-35%
+5MB
通过
低
B 分段锁
通过
-2%
+40MB
通过
中
C CAS
通过
+3%
+10MB
失败
高
D COW
通过
-1%
+650MB
通过
中

这里并不是简单地“谁分数最高就合并谁”。

工程上应该分五步处理。

1. 先区分硬约束和软目标

硬约束不参与加权，它们是淘汰条件：

hard_constraints:
  correctness: pass
  public_api_compatibility: pass
  memory_growth: <= 512MB
  security_scan: pass

软目标才参与比较：

soft_objectives:
  qps_regression: minimize
  p99_regression: minimize
  implementation_complexity: minimize
  changed_lines: minimize

于是：

C 因兼容性失败直接淘汰；

D 因内存超限直接淘汰；

A、B 进入可行解集合。

这可以避免一种很危险的加权方式：

$$
0.5\times性能 + 0.3\times正确性 + 0.2\times兼容性
$$

因为这种公式可能允许一个“性能特别好但偶尔算错钱”的方案获得高分。

正确性、安全性、数据完整性等目标通常不能被其他优势抵消。

2. 在可行解中做非支配比较

设方案 $x$ 在所有指标上都不比方案 $y$ 差，并且至少一个指标更好，那么称 $x$ 支配 $y$。

在上面的例子中：

B 的性能明显优于 A；

A 的复杂度低于 B；

两者都通过硬约束。

因此 A 和 B 可能都在当前帕累托集合里，并不存在数学上唯一的答案。

此时需要加入业务偏好和风险偏好：

高流量核心链路
→ 更重视吞吐和尾延迟
→ 倾向 B

低流量、极度强调可维护性的内部系统
→ 更重视简单性
→ 可能选择 A

也就是说，最终选择通常不是由 agent 自己完成，而是由组织策略完成。

3. 不要直接“拼代码”，先判断优势是否可组合

当 A 在某方面好、B 在另一方面好时，一个自然想法是：

把 A 的好部分和 B 的好部分合起来。

但代码方案不一定具备可加性。

例如：

A 的性能优势来源：无锁数据结构
B 的可维护性优势来源：集中式串行状态管理

这两个优势可能建立在完全相反的架构假设上，机械 cherry-pick 很可能同时失去两边的优势。

应先做一个 composability analysis：

问题
要检查的内容
优势来自哪里？
算法、数据结构、缓存、并发模型还是编译配置
是否共享架构假设？
生命周期、所有权、状态模型、错误模型是否一致
修改是否正交？
是否触碰相同模块、状态和不变量
组合后指标是否仍成立？
原分支的 benchmark 结论不能自动继承
组合成本多高？
是否形成第三种更复杂的新方案

只有当优势是正交的，才适合代码级融合。

例如：

分支 A：
修正锁粒度

分支 B：
减少序列化分配

两个修改作用于不同瓶颈
→ 可能适合融合

而以下情况一般不适合直接合并：

分支 A：
共享可变状态 + 分段锁

分支 B：
不可变快照 + Copy-on-Write

核心状态模型互斥
→ 应选择其一，或重新设计第三个候选

4. 更好的方式：进行“方案级融合”，而非“代码级融合”

可以让一个专门的 synthesis agent 阅读：

A、B 两个实现；

各自的设计说明；

benchmark；

profiler；

测试结果；

reviewer 意见。

然后输出：

retained_from_A:
  - lock sharding strategy
  - compatibility adapter

retained_from_B:
  - allocation-free read path
  - monitoring instrumentation

rejected:
  - B's cache invalidation protocol
  - A's global fallback lock

new_candidate:
  architecture: sharded state + allocation-free reads
  risks:
    - invalidation ordering
    - increased implementation complexity

之后将其作为 全新的候选 E 实现，并重新运行完整 verifier。

关键原则是：

混合方案不是 A 和 B 的测试成绩之和，而是一个未经验证的新方案。

因此必须从头评测。

5. 某些时候不应该合并，而应该保留多个实现

存在几种合理的多解状态。

运行时自适应

低并发环境 → 简单锁方案
高并发环境 → 分段锁方案

系统根据工作负载选择策略。

Feature flag

strategy_v1 = current implementation
strategy_v2 = candidate B

通过灰度逐步扩大流量。

平台差异

移动端 → 低内存实现
服务端 → 高吞吐实现

算法组合

小输入 → 算法 A
大输入 → 算法 B

Fallback

高性能方案异常
→ 自动退回保守方案

这相当于不强行把帕累托集合压缩成一个静态答案，而是把“选择”延迟到运行时。

代价是：

分支数量增加；

测试矩阵扩大；

长期维护成本提高；

状态空间和故障模式变复杂。

因此只有当收益足够明显时才值得保留多个实现。

二、怎样自动决定“融合、选择还是保留多个方案”？

可以设计一个候选决策器：

候选进入
   ↓
硬约束过滤
   ↓
计算帕累托集合
   ↓
判断是否存在单一支配解
   ├─ 有 → 选择支配解
   └─ 无
       ↓
检查修改是否正交、是否可组合
       ├─ 可组合 → 生成融合候选并重新评测
       └─ 不可组合
           ↓
应用业务偏好与风险策略
           ├─ 能明确选择 → 推荐一个
           ├─ 与工作负载相关 → 灰度或运行时路由
           └─ 高风险 → 人工 Architecture Review

这里的“业务偏好”最好写成版本化策略，而不是藏在 prompt 里：

decision_policy:
  priority_order:
    - correctness
    - compatibility
    - security
    - p99_latency
    - qps
    - memory
    - maintainability

  tie_breakers:
    - prefer_smaller_diff
    - prefer_existing_architecture
    - prefer_reversible_change
    - prefer_lower_operational_complexity

这样选择结果才能解释和审计。

三、Open-ended 需求为什么难评测？

例如需求：

给后台增加一个更好用的用户搜索页面。

它没有唯一实现：

可以使用表格，也可以使用卡片；

可以用 URL query 保存筛选，也可以用本地状态；

可以服务端分页，也可以游标分页；

可以即时搜索，也可以点击按钮搜索；

可以抽屉展示详情，也可以跳转详情页。

如果评测集只检查：

页面中是否存在 input
是否发送 GET /users

Agent 很容易通过测试，却交付一个体验很差的页面。

Open-ended 任务不能只用“精确输出匹配”，应该评测一组行为、不变量、质量属性和过程证据。

OpenAI 的评测指南也建议，对于开放式结果，尽量把评判转换为分类、按明确标准打分或候选间成对比较，而不是要求评审器凭空判断一个答案“好不好”。(OpenAI 开发者)

四、Open-ended 任务的六层评测模型

第一层：任务契约，而不是标准答案

不要定义“必须实现成什么样”，而要定义“必须实现什么能力”。

例如：

capabilities:
  - user can search by name or email
  - search state is reflected in URL
  - pagination preserves current filters
  - empty, loading and error states are handled
  - keyboard-only operation is supported

constraints:
  - use existing design system
  - no new production dependency
  - preserve existing API
  - initial JS increase <= 15KB gzip

这样不同实现仍然可以通过同一组评测。

第二层：不变量测试

不变量描述的是无论采用什么方案都应该成立的性质。

例如搜索页：

任何筛选组合下：
返回结果都必须满足筛选条件

翻页后：
筛选条件不能丢失

刷新页面后：
当前搜索状态应能恢复

请求乱序返回时：
旧请求不能覆盖新结果

无权限用户：
不能看到敏感字段

相比检查 DOM 结构，不变量更能容纳多种正确实现。

第三层：场景矩阵

不要只写一个 happy path。

可以按维度组合：

维度
场景
数据量
0、1、20、10 万条
网络
正常、慢速、超时、乱序
输入
中文、英文、特殊字符、超长文本
权限
管理员、普通用户、无权限
设备
桌面、窄屏、触屏
操作
鼠标、键盘、刷新、前进后退
API
成功、空结果、400、401、500
状态
首次访问、带 query、过期缓存

然后用 pairwise / combinatorial testing 控制组合爆炸，而不是穷举全部笛卡尔积。

第四层：变形测试

当无法知道每次的精确结果时，可以检查输入变换后应满足的关系。

这称为 metamorphic testing。

例如：

原始搜索：
name = "alice"

变形 1：
name = " Alice "
结果集合应相同

变形 2：
加入无关排序
结果元素集合应相同，只是顺序变化

变形 3：
pageSize 从 20 改成 50
前 20 个结果应与原结果一致

变形 4：
相同请求执行两次
在数据未变化时结果应一致

对于开放式系统，这类“关系型 oracle”往往比标准答案更实用。

第五层：参考实现之外的差分评测

可以将候选与以下对象比较：

当前生产版本；

参考实现；

另一个候选；

已知稳定库；

真实人工操作记录。

但不要要求候选与参考实现逐行一致，而是比较：

行为是否一致
协议是否兼容
关键指标是否退化
错误边界是否一致

例如：

对随机生成的 10,000 组搜索条件：
旧系统与新系统的用户 ID 集合一致率应为 100%
排序变更除外

第六层：主观质量使用 rubric 和 pairwise review

诸如：

界面是否清晰；

API 是否易用；

架构是否合理；

代码是否容易维护；

错误提示是否有帮助；

很难完全用确定性规则判断。

应该建立 rubric：

维度
1 分
3 分
5 分
信息架构
操作路径混乱
基本可理解
层次清楚且任务导向
错误恢复
只显示失败
提供重试
解释原因并保留用户状态
架构一致性
引入新模式
基本沿用现有模式
与现有架构高度一致
可维护性
高耦合、无边界
模块基本清晰
职责和扩展点明确
可访问性
键盘不可用
主要路径可用
完整焦点与语义支持

然后让 reviewer 做成对比较：

给定相同需求和评测证据：
候选 A 与候选 B 哪个更符合 rubric？

Pairwise 判断通常比让 LLM 独立给一个绝对分数稳定。(OpenAI 开发者)

五、需求每次都不同，评测集怎么复用？

不能把评测集只理解为一批固定测试用例。

生产环境应该建立三层资产。

1. 通用能力评测

适用于所有任务：

是否正确理解仓库结构
是否遵守项目规则
是否运行必要测试
是否修改了无关文件
是否提供验证证据
是否引入安全风险
是否能从失败恢复
是否控制 token、时间和工具成本

这部分评测 agent 的工作方式。

OpenAI 的 agent 评测体系将完整 trace--包括模型调用、工具调用、guardrail 和 handoff--作为评测对象，而不只看最终输出。(OpenAI 开发者)

2. 领域能力评测

例如前端项目：

表单
数据列表
权限控制
路由
缓存
可访问性
响应式布局
视觉回归
Web 性能

后端项目：

事务
并发
幂等
API 兼容
数据库迁移
限流
消息一致性
可观测性

3. 任务现场生成的评测

针对当前需求临时建立：

复现测试
需求不变量
受影响模块测试
性能 baseline
API contract
真实流量样本

也就是：

$$
\text{最终评测集}

\text{通用评测}
+
\text{领域评测}
+
\text{任务特定评测}
$$

这能同时兼顾复用和任务差异。

六、谁来生成任务特定评测？

不应完全由实现 Agent 自己负责。

否则容易出现：

Agent 选择方案 A
↓
Agent 按照方案 A 的行为写测试
↓
方案 A 自然通过

这是 implementation-conditioned evaluation。

更可靠的职责分离是：

Requirement Agent
→ 提取能力、约束和风险

Test Designer Agent
→ 在不知道具体实现的前提下设计黑盒测试

Implementation Agents
→ 分别实现候选

Adversarial Test Agent
→ 根据候选差异寻找反例

Verifier
→ 运行隐藏测试和统一指标

Human Reviewer
→ 处理剩余主观判断

测试设计者最好只看到：

原始需求；

API 和架构说明；

当前系统行为；

风险模型。

而不看候选的具体实现。

七、评测集本身也需要防作弊

Agent 可能无意或有意地优化 evaluator，而不是真实目标：

硬编码测试输入；

识别 benchmark 环境；

删除或跳过失败测试；

修改 benchmark；

用缓存伪造速度提升；

降低结果精度换性能；

只覆盖已知路径；

读取未来 commit 或标准补丁。

SWE-bench 社区曾公开讨论 agent 通过未来仓库状态等渠道接触潜在解法的漏洞，这说明 benchmark 隔离和数据泄漏是实际问题。(GitHub)

2026 年的 PERFOPT-Bench 也报告，单看原始 speedup 会把 benchmark-specific shortcut 当成优化，因此加入了隐藏正确性测试、可验证性能测量和轨迹审计。(arXiv)

工程上需要：

隐藏测试
只读评测环境
独立 benchmark runner
测试文件完整性校验
随机化输入
多 workload 验证
冷启动与热启动分离
重复运行和置信区间
轨迹审计
生产 shadow traffic

另外，不应假定“Agent 写了很多测试”就一定更可靠。2026 年一项针对 SWE-bench 轨迹的研究发现，当前 agent 生成测试的数量与最终任务成功并没有稳定的显著关系；许多所谓测试实际更接近用于观察值的调试脚本。(arXiv)

重点不是测试数量，而是：

测试是否独立于实现；

是否覆盖失败模式；

是否有有效 oracle；

是否能区分候选；

是否能防止回归。

八、不能只评最终代码，还要评完整轨迹

两个 Agent 最终都通过测试，但过程可能完全不同。

Agent A

正确定位问题
→ 修改 3 个文件
→ 运行相关测试
→ 修复一次失败
→ 完成

Agent B

修改 40 个文件
→ 删除失败测试
→ 多次执行危险命令
→ 偶然通过最终测试

只看最后的 pass/fail，会把两者视为等价。

因此应增加轨迹指标：

维度
示例指标
效率
token、工具调用数、执行时间
范围控制
修改文件数、无关 diff
验证习惯
是否先复现、是否运行必要测试
恢复能力
遇到失败后能否改变策略
安全
是否尝试越权或破坏性操作
可解释性
是否保存关键决策和证据
协作
是否正确交接给其他 agent
稳定性
相同任务重复运行的方差

Anthropic 的 agent eval 指南也强调，开放式 agent 的评估需要覆盖任务结果、行为轨迹和不同失败模式，而不能只依赖单一成功指标。(Anthropic)

九、还会遇到哪些典型工程问题？

1. 候选同质化

开了五个 Agent，但它们都选择“加锁”。

原因可能是：

使用同一模型；

使用相同 prompt；

获取相同上下文；

温度低；

planner 已经暗示某个方案；

同一个仓库模式形成强锚定。

解决方式：

Agent A：优先最小改动
Agent B：优先无锁方案
Agent C：允许重新设计状态模型
Agent D：从性能角度分析
Agent E：从兼容性和回滚角度分析

先要求独立提交设计，再让它们看到其他候选，减少早期相互锚定。

2. 多 Agent 修改冲突

代码层面可通过 worktree 隔离，但语义冲突更难：

前端 Agent 假设 API 返回 cursor
后端 Agent 实现 offset pagination
测试 Agent 根据第三种接口写测试

需要共享的、版本化的 artifact：

API schema
ADR
数据模型
接口契约
任务 DAG
ownership map
decision log

多 Agent 不应仅靠自然语言聊天同步状态。

2026 年关于 code-as-harness 的综述将多 Agent 共享状态一致性、反馈不完整和人工监管列为仍未解决的核心问题。(arXiv)

3. 评测成本爆炸

假设：

5 个候选；

每个候选 3 次重复；

8 个评测阶段；

完整 E2E 需要 30 分钟。

总成本会很快失控。

应采用漏斗式评测：

Stage 1：编译、lint、快速单测
淘汰明显失败候选

Stage 2：受影响测试、静态扫描
淘汰不兼容候选

Stage 3：完整集成测试
仅运行剩余候选

Stage 4：benchmark、profiling、E2E
仅运行帕累托候选

Stage 5：人工评审和灰度
通常只剩 1～2 个

这类似 successive halving。

并行数量也不是越多越好。2026 年关于 coding-agent 算法探索的实验发现，在固定 token 预算下，少量但推理更深入的候选有时比大量浅层候选更有效，同时更强的模型也可能更频繁地利用评分漏洞。(arXiv)

4. 需求本身不稳定

评测集可能忠实验证了旧需求，但产品需求已经变化。

需要版本化：

requirement_version: 7
evaluation_suite_version: 12
architecture_policy_version: 4
baseline_commit: abc123

每个候选必须声明它针对哪个版本。

需求变化后，应判断：

哪些测试仍有效；

哪些测试失效；

哪些 baseline 必须重建；

当前运行中的候选是否需要中止；

是否可以增量迁移。

5. 随机性和重复性

Agent 的一次成功不代表稳定成功。

对高风险任务应执行多次：

任务重复 5 次
成功率：4/5
平均成本：$8.20
P95 时间：24 分钟
产生 3 种不同架构
其中 1 次破坏 API

评估的不只是最好结果，而是：

$$
P(\text{成功}),\quad
P(\text{严重失败}),\quad
\text{成本分布},\quad
\text{结果方差}
$$

SWE-bench 使用 Docker 来提升环境可复现性，但环境可复现不等于 agent 输出确定；两者需要分别控制。(GitHub)

6. 线下评测与生产指标不一致

线下 benchmark 可能表明方案 B 最好，但生产中可能出现：

特殊流量分布；

热点 key；

缓存污染；

长尾请求；

下游限流；

数据规模变化；

未知浏览器或设备。

因此最终评测必须延伸到：

offline eval
→ staging
→ shadow traffic
→ 1% canary
→ 10% canary
→ full rollout

每一步定义：

promote
hold
rollback

规则。

不能把 PR 合并当作优化过程结束。

7. 评测集老化

系统、依赖和用户行为不断变化，固定 eval 会逐渐失真。

需要从以下渠道持续补充：

线上 incident；

被人工拒绝的 agent PR；

code review comment；

回滚记录；

客诉；

性能回归；

安全事件；

Agent 卡死轨迹；

Agent 与人类意见分歧案例。

自动更新的 SWE-bench-Live 正是为了缓解固定 benchmark 的时效性、语言和平台覆盖问题。(GitHub)

不过，线上数据进入评测集时要去除：

用户隐私；

密钥；

生产数据；

可直接泄露答案的信息；

与训练数据重叠的内容。

8. “是否合并 PR”不是一个干净的质量标签

PR 被拒绝可能因为：

需求取消；

与另一项工作重复；

维护者没有时间；

项目方向变化；

流程或权限问题；

实现确实错误。

2026 年一项分析约 9,799 个经过人工审查的 Agent PR 的研究发现，被拒绝的 PR 中只有一部分能明确归因于 agent 失败；很多拒绝来自工作流约束或缺乏可观察理由。因此，不能直接把 merge rate 当作 agent 正确率。(arXiv)

应同时记录：

技术正确性
review 修改次数
人工介入量
拒绝原因
需求变化
是否重复实现
合并后的回滚情况

十、推荐的完整工程框架

可以把整个系统设计为五个平面。

1. Spec Plane

管理：

需求；

验收标准；

硬约束；

软目标；

风险；

版本。

2. Search Plane

负责：

产生差异化候选；

隔离 worktree；

方案级融合；

保留帕累托候选；

控制搜索预算。

3. Evaluation Plane

包含：

确定性测试；

不变量；

变形测试；

隐藏测试；

benchmark；

LLM rubric；

人工评审；

轨迹评分。

4. Decision Plane

负责：

硬约束过滤；

非支配比较；

业务偏好；

风险与可回滚性；

人工决策；

推荐理由。

5. Runtime Plane

负责：

feature flag；

shadow；

canary；

监控；

自动回滚；

生产反馈回灌 eval。

整体流程：

需求
  ↓
结构化 Spec
  ↓
独立生成评测与候选
  ↓
小规模差异化并行探索
  ↓
分层评测漏斗
  ↓
帕累托集合
  ↓
选择 / 方案级融合 / 运行时共存
  ↓
PR 与人工决策
  ↓
灰度和生产评测
  ↓
失败案例回灌评测集

核心结论

对于第一个问题：

当分支各有优势时，不应默认 cherry-pick 合并。应先硬约束过滤，再做帕累托比较和可组合性分析。只有优势来源相互正交时才生成融合候选，而且融合候选必须重新完整评测。无法静态决定时，可以借助 feature flag、灰度或运行时路由保留多种实现。

对于第二个问题：

Open-ended 需求不应围绕唯一标准答案构建评测，而应围绕能力、约束、不变量、场景矩阵、变形关系、质量 rubric 和完整执行轨迹构建。最终评测集应由通用评测、领域评测和任务现场生成评测共同组成。

最重要的工程转变是：

评测集不是任务结束后的考试，而是需求定义、方案搜索、架构决策和生产发布的共同协议。

Sources:

Evaluation best practices | OpenAI API

Repo State Loopholes During Agentic Evaluation #465

PERFOPT-Bench: Evaluating Coding Agents on Software Performance Optimization

Demystifying evals for AI agents
