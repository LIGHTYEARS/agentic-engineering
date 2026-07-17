# 信息图系列内容规划

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/14 16:33:16

---

## Prompt

这份 deep research 的结果放在一张图中显然无法完成承载，按照内容的层次，你认为需要生成多少张图，每张图分别展示什么主题。用表格简单列出。

---

## Response

建议拆成 6 张图。这样既能保持 Swiss Style 的信息密度，也不会把关键工程机制压缩成难以阅读的小字。

图序
主题
主要内容
1
总览：Coding Agent 如何突破单次自回归
从“单次生成”到“生成—执行—验证—修正”的闭环；给出整体架构总图和核心结论。
2
计划与上下文管理
Plan / Spec / Task State / Project Memory / Skills / AGENTS.md；说明如何把隐性目标外化，减少目标遗忘和任务漂移。
3
工具反馈与硬约束
编译、测试、lint、类型检查、benchmark、profiler、安全扫描、hooks、CI；说明如何把自然语言目标变成可执行 verifier。
4
并行候选与多 Agent 搜索
Worktree、branch、sandbox、subagent、候选 A/B/C；解释如何缓解单路径依赖，以及它与真正 NSGA-II 的区别。
5
Reviewer、Verifier 与候选选择
Writer / Reviewer / Security / Performance agent 分工；硬约束淘汰、best-of-N、非支配比较、人工审批。
6
大型工程落地案例
以“修复核心结算模块并发 Bug”为完整案例：复现、建立 baseline、生成候选、统一评测、淘汰、选型、PR、灰度与回滚。

其中第 1 张负责建立全局认知，第 2～5 张分别解释关键机制，第 6 张把前面的机制串成一次完整工程流程。
