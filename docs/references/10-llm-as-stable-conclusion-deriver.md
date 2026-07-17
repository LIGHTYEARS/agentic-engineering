# LLM 作为稳定结论推导器

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/15 19:39:33

---

## Prompt

但是这里就有一个前提对吗：LLM 必须是证据链的语义化表示的稳定结论推导器？也就是给它一个证据链的语义化表示，它从中推导出的结论是正确的？

---

## Response

考虑使用网络搜索获取概念性答案

用户请求的是关于LLM可靠性的概念性问题，可能需要官方来源或技术性文章。考虑到问题涉及因素和推理，我可能需要引用相关文献或研究来进一步阐明。我将搜索权威资料讨论推理的准确性、校准与工具的正确使用。

搜索关于推理一致性和工具使用的文献

Lie to Me: How Faithful Is Chain-of-Thought Reasoning in ...
A Closer Look at Bias and Chain-of-Thought Faithfulness ...
FAITHFULNESS in Traditional Chinese - Cambridge Dictionary
Introducing Model Spec Evals
Reasoning models struggle to control their chains of ...
Reasoning models don't always say what they think
Measuring short-form factuality in large language models
Research
OpenAI Evals
GPT-5.6 Preview System Card - OpenAI Deployment Safety Hub
Better Evaluation
Model Spec (2025/12/18) - OpenAI
Models | OpenAI API
What is evaluation?

已思考 1m 35s

对，这确实是前面那套工程流程里的一个隐含前提。但这个前提如果表述为：

“只要把证据链语义化地交给 LLM，它就能稳定推出正确结论”

那么它过强，而且目前不成立。

更准确的表述应该是：

LLM 可以作为证据链上的一个推理组件，但不能被假设为一个稳定、完备、始终正确的结论推导器。

1. 这里其实有三种不同能力

给定一条证据链：

原始日志
→ 指标汇总
→ benchmark 结果
→ 约束条件
→ 结论

LLM 至少需要同时具备三种能力。

A. 证据忠实性

它有没有正确读取证据？

例如表格里是：

baseline = 10,000
candidate = 13,000

模型不能看成 15,000，也不能漏掉单位或实验环境。

B. 推理有效性

结论是否真的能从证据推出？

$$
\frac{13000-10000}{10000}=30%
$$

这里“提升 30%”可以推出，但“生产环境提升 30%”未必可以，因为证据可能只来自 staging。

C. 结论校准

它是否正确表达了结论的范围和不确定性？

可靠结论应该是：

在 staging-8c、workload-v4、10 次运行条件下，
吞吐量中位数相对基线提升约 30%。
尚未验证生产流量。

而不是：

性能提升了 30%。

这三层任何一层出错，最终结论都会失真。

2. 为什么“证据齐全”仍然不能保证推导正确

证据可能被错误组织

即使所有事实都在上下文里，模型也可能：

忽略某条反例；

混淆基线和候选；

错读单位；

使用错误公式；

把相关性当因果；

把局部结果外推到整体；

忽略时间范围；

选择性引用支持当前结论的证据。

也就是说：

$$
E \models C
$$

表示证据 $E$ 逻辑蕴含结论 $C$。

LLM 实际做的是近似判断：

$$
\operatorname{LLM}(E) \rightarrow \hat C
$$

而没有机制保证：

$$
E \models \hat C
$$

近期研究一直在发现 reasoning trace 并不总是忠实，模型可能给出流畅、合理、却并非真正由给定证据推出的解释。即使是推理模型，也并非完全可靠。(arXiv)

3. 还有一个更隐蔽的问题：结论正确，理由也可能不对

考虑：

证据：
baseline = 100
candidate = 130

模型结论：
提升 30%

结论是正确的。

但模型内部可能不是通过正确公式算出来，而只是因为：

100 → 130

看起来像常见的“提升 30%”模式。

如果把数据稍微换成：

baseline = 80
candidate = 104

它也许仍能答对，也许就会错。

所以需要区分：

类型
含义
结果正确
最终答案碰巧正确
推导有效
每一步都由已验证前提推出
推导忠实
展示的理由确实影响了模型结论
推导稳定
输入轻微变化时仍使用正确规则
推导完备
没有遗漏影响结论的重要证据

研究表明，模型生成的 chain-of-thought 有时是事后合理化，而不一定忠实反映产生答案的机制。(Anthropic)

因此，不能把“它解释得很合理”当成“它进行了正确推导”。

4. 这意味着前面的工程架构还要再加一层

不能设计为：

现实
→ 工具采集证据
→ LLM 阅读证据
→ 得出最终结论

更可靠的是：

现实
→ 工具采集证据
→ 结构化事实
→ 确定性计算 / 规则验证
→ LLM 解释与综合
→ 独立校验
→ 人类或策略决策

核心变化是：

能确定性计算的部分，不交给 LLM。

例如性能提升百分比：

improvement = (candidate - baseline) / baseline

应由代码计算。

内存是否超限：

assert memory_growth_mb <= 512

应由规则判断。

API 是否破坏兼容：

API diff tool
→ breaking / non-breaking

应由专门工具判断。

LLM 负责的是：

整合多个指标；

识别冲突；

总结证据；

提出候选解释；

标记缺失信息；

生成决策说明。

而不应负责单独裁决所有事实。

5. 最理想的模式是“证明携带型结论”

Agent 不应只输出：

推荐方案 B。

而应输出：

claim:
  recommended_candidate: B

hard_constraints:
  correctness:
    result: pass
    evidence: race-test-1842
  compatibility:
    result: pass
    evidence: api-diff-77
  memory:
    value: 37MB
    threshold: 512MB
    result: pass

soft_objectives:
  qps_delta: -2.1%
  p99_delta: +2.7%
  complexity: medium

comparison:
  dominated_candidates:
    - A
  rejected_candidates:
    - C: API compatibility failure
    - D: memory limit exceeded

uncertainty:
  - not tested on production traffic

这里结论不是一段自由文本，而是一个：

$$
\text{结论}+\text{前提}+\text{证据}+\text{规则}+\text{范围}
$$

这接近 proof-carrying claim。

LLM 可以生成这份结构，但系统还需要验证：

引用的 evidence 是否存在；

数值是否一致；

比较规则是否正确；

是否遗漏失败项；

结论是否被证据支持。

6. 可以把推理任务分成三类

第一类：确定性可计算

例如：

百分比；

测试是否通过；

内存是否超限；

API diff；

类型是否兼容；

依赖是否合法。

这类应由程序完成。

第二类：规则可判定

例如：

正确性失败 → 淘汰
内存 > 512MB → 淘汰

这类应由 policy engine 或规则系统完成。

第三类：开放式综合判断

例如：

哪个架构长期更易维护；

是否值得引入新抽象；

两个非支配方案如何取舍；

当前发布窗口是否适合上线。

这类才适合 LLM 辅助，但必须：

使用 rubric；

给出证据；

多 reviewer；

显式不确定性；

保留人工决策。

7. LLM 最适合扮演什么角色

不是：

证据输入后必然输出真结论的定理证明器。

而更像：

一个高能力但不完全可靠的证据解释器、假设生成器和综合分析器。

它的优点是：

能跨多种非结构化证据；

能识别潜在关系；

能总结复杂上下文；

能提出缺失信息；

能把技术证据翻译为决策语言。

它的弱点是：

推导没有普遍 soundness 保证；

容易被流畅但错误的证据链影响；

可能遗漏反例；

可能过度概括；

自己的解释未必忠实；

不稳定且受提示形式影响。

甚至用 LLM 作为 judge 时，弱 judge 也可能被一条流畅但错误的 reasoning chain 说服。(arXiv)

8. 那么工程上需要什么前提？

不是要求：

$$
\operatorname{LLM}(E)=C_{\text{true}}
$$

而是建立一个弱一些、可实现的前提：

$$
P(\text{严重错误通过系统}) \leq \epsilon
$$

这个保证来自整个系统，而不是单个模型。

可以通过以下组合降低风险：

结构化证据
+
确定性计算
+
逐步前提验证
+
独立 verifier
+
反例搜索
+
多次采样
+
规则约束
+
人工审批

有些研究正尝试逐步验证每一个 reasoning step，只允许基于此前已验证前提继续推导，以减少早期错误向后传播。(arXiv)

但这仍是提高可靠性，不是提供一般性的绝对正确保证。

9. 对大型代码仓库工作流的直接影响

存量模块新增功能时，不能让 Agent 最后自由总结：

所有需求已经满足。

应要求每个需求都对应一个机器可校验状态：

需求主张
验证者
原接口保持兼容
API diff / contract test
性能无明显下降
benchmark runner
不重复产生副作用
幂等性测试
下游全部兼容
集成测试矩阵
新功能满足业务规则
黑盒场景测试
架构可维护
reviewer rubric + 人审
可安全上线
canary metrics + rollback policy

然后 LLM 只能在这些结果之上做综合：

硬约束全部通过。
B 在 QPS 和 P99 上优于 A。
B 的复杂度稍高，但没有引入新运行时依赖。
基于当前策略推荐 B。

其中：

“硬约束全部通过”由系统计算；

“B 优于 A”由指标比较得到；

“复杂度稍高”由静态指标与 review 支持；

“推荐 B”才是策略性判断。

10. 最终结论

你的判断抓住了一个非常关键的薄弱点：

如果整个 Agent 架构最终仍然依赖 LLM 从语义化证据中一次性推出正确结论，那么系统只是把“无依据生成”升级成了“有依据但仍可能误推导”。

因此更可靠的架构不是：

$$
\text{证据} \rightarrow \text{LLM} \rightarrow \text{真结论}
$$

而是：

$$
\boxed{
\text{证据}
\rightarrow
\text{确定性验证}
\rightarrow
\text{局部可校验推导}
\rightarrow
\text{LLM 综合}
\rightarrow
\text{独立复核}
}
$$

LLM 不需要成为绝对稳定的结论推导器；工程系统需要做的是：

把必须正确的推导尽量外移到可计算、可执行和可验证的机制中，只把无法完全形式化的综合判断留给 LLM，并对这些判断保持不确定性和监督。

Sources:

Language Models Don't Always Say What They Think

Measuring Faithfulness in Chain-of-Thought Reasoning
