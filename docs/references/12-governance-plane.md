# 治理层 Governance Plane

> Source: 多目标非凸优化解析讨论记录
> Date: 2026/7/15 20:17:08

---

## Prompt

接下来我们讨论 Governance plane 具体方案，你先做深入调研搜索，接着给出回复附上依据。注意我并不是要做一个给多个团队使用的平台产品。

---

## Response

我会聚焦“单团队落地”的治理方案，不讨论多租户平台、复杂计费或跨组织产品化能力。重点调研权限边界、审批分级、沙箱与网络、凭证、审计、风险分层、成本控制和责任机制。

搜索OpenAI、Anthropic和GitHub的治理与安全信息

Running Codex safely at OpenAI
Security - Claude Code Docs
Customizing or disabling the firewall for GitHub Copilot ...
Microsoft Copilot: Your AI companion
AI Risk Management Framework | NIST
AI Agent Security - OWASP Cheat Sheet Series
Agentic.ai: Agentic AI Tools Directory — Find AI That Actually ...
openai/codex: Lightweight coding agent that runs in your ...
Copilot | AI chat for work
Artificial Intelligence Risk Management Framework
OWASP Top 10 for Agentic Applications for 2026
What is Agentic AI?
Agent approvals & security | ChatGPT Learn
Microsoft Copilot - App Store - Apple
Agentic Skills Top 10
Agentic AI, explained
Auto-review of agent actions without synchronous human ...
Codex Pricing
AI RMF - AIRC - NIST AI Resource Center
AIVSS Scoring System For OWASP Agentic AI Core Security ...
Microsoft Copilot | AI Tools for Organizations
AGENTIC Slang Meaning
Companion Edition - Agentic Ai (AAI2) - OWASP Cornucopia
Agent (full access) always asks for approval - Page 2 - Codex
SP 800-53 Control Overlays for Securing AI Systems ...
Agentic AI là gì, ứng dụng Agentic AI trong doanh nghiệp
Codex for Admins and IT - Event
CODEX

搜索治理模型与政策矩阵的细节

Running Codex safely at OpenAI | OpenAI
Security - Claude Code Docs
Application card: GitHub Copilot Agents - GitHub Docs
AI Agent Security - OWASP Cheat Sheet Series
AI Risk Management Framework | NIST
OWASP Top 10 for Agentic Applications for 2026 - OWASP Gen AI Security Project
Agent approvals & security | ChatGPT Learn
Auto-review of agent actions without synchronous human oversight
MCP02:2025 - Privilege Escalation via Scope Creep | OWASP Foundation

已思考 31s

对单个团队而言，Governance Plane 不应该被设计成一个“大而全的 AI 管理平台”。更合适的定位是：

一组嵌入现有代码仓库、Git 工作流、CI/CD 和开发环境中的轻量治理规则，用来限制 Agent 的行动空间、明确审批边界、保存审计证据，并控制事故的最大影响范围。

核心目标不是集中管理所有 Agent，而是回答四个问题：

Agent 可以看到什么？

Agent 可以做什么？

哪些动作必须由人批准？

出现问题时，能否定位、停止和回滚？

OpenAI、Anthropic 和 GitHub 当前公开的 coding agent 安全设计基本都围绕“沙箱边界 + 最小权限 + 风险动作审批 + 可追溯日志”展开。Codex 默认关闭网络并限制在工作区内；Claude Code 默认只读并对写文件和执行命令申请权限；GitHub Copilot cloud agent 使用临时隔离环境、默认防火墙、受限分支和人工合并。(OpenAI)

一、Governance Plane 的范围

建议将 Governance Plane 划分为 8 个治理域。

治理域
核心职责
团队内具体实现
不需要做的平台化能力
1. Risk Classification
对任务和动作分级，决定自动化程度
在 Issue/任务模板中设置风险等级；用 YAML 或标签记录
不需要建设统一风险管理后台
2. Identity & Accountability
明确谁发起任务、哪个 Agent 执行、谁审批
Git 提交身份、任务发起人、PR reviewer、session ID
不需要自研完整 IAM
3. Permission & Tool Policy
限制文件、命令、工具和服务访问
Agent 配置、shell wrapper、MCP allowlist、目录权限
不需要通用策略编排平台
4. Sandbox & Environment
限制写入范围、网络和运行资源
容器、Dev Container、worktree、临时环境
不需要建设云端沙箱产品
5. Approval & Separation of Duties
定义哪些动作可自动执行，哪些必须审批
CLI approval、PR required review、CODEOWNERS、CI gate
不需要开发审批工作流系统
6. Data & Secret Governance
控制代码、日志、凭证和外部数据暴露
.env 隔离、短期 token、secret manager、日志脱敏
不需要自研数据分类平台
7. Audit & Observability
保存 Agent 做了什么、为什么做、结果是什么
session trace、命令日志、PR 证据、CI artifact
不需要建设全功能 APM
8. Budget & Kill Switch
防止无限循环、成本失控和异常行为
token/time/tool-call 上限、任务超时、人工停止、撤销凭证
不需要复杂计费系统

NIST AI RMF 强调治理应贯穿 AI 系统的设计、开发、使用和评估，而不是只在上线前做一次审批；OWASP 的 Agent 安全指南则将最小权限、外部输入验证、高风险动作人工介入、结构化日志和成本限制列为核心控制。(OWASP Cheat Sheet Series)

二、第一步：建立任务风险分级

不要按“使用哪个模型”分风险，而应按任务可能产生的后果分级。

建议采用四级模型

等级
任务示例
Agent 自主权
人工要求
L0：只读分析
解释代码、查依赖、生成设计建议、定位问题
可读取授权仓库，不得写文件、不得联网
通常无需逐步审批
L1：低风险修改
文档、测试补充、局部重构、非生产脚本
可修改 worktree，可运行本地测试
PR 合并前人工 review
L2：业务功能修改
存量模块新增功能、数据库查询变化、接口扩展
可修改隔离分支、运行 CI；网络受限
至少一名模块 owner 审批
L3：高风险修改
支付、鉴权、数据迁移、生产配置、依赖升级、CI workflow
严格工具白名单；关键操作人工批准
双人审批，必须灰度与回滚方案
L4：禁止自动执行
生产数据删除、直接发布、修改权限体系、读取高敏感数据
Agent 只能生成方案和操作清单
必须由人手动执行

风险等级不应完全由 Agent 自己决定。Agent 可以提出建议，但最终等级由任务模板规则或负责人确认。

可以通过简单规则自动提高等级：

risk_escalation:
  touches:
    - auth/**
    - payment/**
    - migrations/**
    - .github/workflows/**
    - infra/**
  adds_runtime_dependency: true
  changes_public_api: true
  handles_personal_data: true
  requires_external_network: true

任何一项命中，就至少进入 L2 或 L3。

三、第二步：对“动作”分级，而不只对任务分级

同一个任务内部，不同动作的风险不同。

例如，修复一个前端 Bug 时：

读取代码风险低；

修改源文件风险中等；

执行 npm test 风险低；

执行未知仓库脚本风险较高；

运行 curl | bash 风险很高；

上传日志到外部服务风险很高。

因此治理对象应是：

$$
\text{任务风险} + \text{动作风险} + \text{资源敏感度}
$$

建议动作矩阵

动作
默认策略
说明
读取当前仓库文件
自动允许
排除 secrets 和敏感目录
搜索 Git 历史
自动允许
只读
修改当前 worktree
L1 以上允许
不允许直接修改主分支
运行 lint、typecheck、unit test
自动允许
使用经过确认的固定命令
运行任意 shell 命令
需策略匹配
不应使用通配式全允许
安装已有 lockfile 依赖
通常允许
需使用锁文件、禁止 lifecycle script 或在沙箱中执行
新增生产依赖
必须审批
需检查许可证、安全和维护状态
修改 lockfile
必须 review
防止供应链变化被隐藏
访问公网
默认拒绝
按域名和方法单独放行
访问内部 API
默认拒绝
使用只读、短期、最小权限凭证
修改 CI workflow
L3 审批
workflow 可能读取 secrets 或获得高权限
写数据库
测试环境有限允许
生产环境禁止
创建 PR
自动允许
只能创建 Draft PR
合并 PR
Agent 禁止
必须由人或确定性发布流程完成
发布生产
Agent 禁止或严格审批
不应由实现 Agent直接执行
删除文件或批量重写
根据范围审批
设置文件数量和目录限制

OWASP 明确反对 wildcard 权限以及允许 Agent 在无沙箱环境执行任意代码，并建议将不可逆操作的决策与执行分离。(OWASP Cheat Sheet Series)

四、第三步：默认采用“隔离内自治，边界处审批”

这是 Governance Plane 最重要的设计原则。

安全工作区内部
  ├─ 读取仓库
  ├─ 修改 worktree
  ├─ 运行批准过的测试
  ├─ 生成报告
  └─ 创建本地提交
          ↓
跨越边界
  ├─ 访问公网
  ├─ 读取其他目录
  ├─ 获取凭证
  ├─ 修改 CI
  ├─ 推送远程
  └─ 发布或操作环境
          ↓
审批或阻断

OpenAI 将这种模式概括为：低风险动作在有界环境中应当顺畅运行，高风险或越界动作必须停下来审查；沙箱负责定义技术边界，审批策略负责决定什么时候可以跨越边界。(OpenAI)

对单团队来说，落地方式可以很简单：

agent_environment:
  filesystem:
    read:
      - repo/**
    write:
      - repo/**
    deny:
      - ~/.ssh/**
      - ~/.aws/**
      - ~/.kube/**
      - /etc/**
      - ../other-repos/**

  network:
    default: deny
    allow:
      - registry.npmjs.org
      - internal-package-mirror.company
      - api.github.com

  process:
    allowed:
      - git status
      - git diff
      - pnpm test
      - pnpm lint
      - pnpm typecheck
    denied_patterns:
      - "curl * | sh"
      - "sudo *"
      - "rm -rf /"
      - "git push --force*"

不一定要引入 OPA 或自研 policy engine。初期可以用：

Agent 原生 permission 配置；

shell wrapper；

Dev Container；

CI required checks；

GitHub branch protection；

CODEOWNERS；

仓库内 YAML 规则。

五、第四步：身份与责任归属

Agent 不能成为无法追责的匿名执行者。

每次任务至少要记录：

agent_run:
  run_id: agent-2026-0715-1842
  initiated_by: zhangsan
  task_id: ORDER-184
  repository: checkout-service
  base_commit: abc123
  agent: codex
  model_version: ...
  permission_profile: feature-development-v2
  risk_level: L2
  human_owner: lisi

每个 Agent 提交或 PR 都应能追溯到：

谁发起；

使用了哪个任务；

Agent 使用了什么权限；

修改基于哪个 commit；

运行了哪些测试；

哪个人最终批准。

GitHub Copilot cloud agent 的公开设计中，Agent 提交会标记 Agent 作者和任务发起人，提交关联完整 session log；Agent 只能向受限分支推送，不能批准或合并自己的 PR。(GitHub Docs)

单团队建议

不需要创建复杂的 Agent 用户体系，可以采用：

Git author:
  ai-agent[bot]

Git co-author / metadata:
  human initiator

PR labels:
  agent-generated
  risk:L2
  requires:module-owner

PR body:
  session ID
  permission profile
  verification evidence

六、第五步：职责分离

至少要保持以下三种角色分离：

角色
可以做什么
不可以做什么
Implementation Agent
修改代码、运行测试、提出结论
不能批准自己的越权动作，不能决定最终上线
Verifier / CI
执行确定性检查、生成证据
不修改候选实现以让测试通过
Human Owner
接受风险、批准合并和发布
不应在不了解证据的情况下机械批准

高风险任务再增加：

Security reviewer；

Database owner；

Release owner。

OpenAI 的 auto-review 实践专门将完成任务的主 Agent 与判断越界动作是否安全的审批 Agent 分离，因为主 Agent 会受到“完成任务”目标的压力，容易把安全边界视为障碍。(alignment.openai.com)

不过，对单团队而言，不建议一开始就自动部署“审批 Agent”。先使用确定性规则和人工审批更简单、更容易审计。自动审批可以在收集足够动作日志后再引入。

七、第六步：凭证和数据治理

1. Agent 不应继承开发者全部凭证

禁止直接把以下目录暴露给 Agent：

~/.ssh
~/.aws
~/.kube
~/.config/gcloud
完整浏览器 profile
生产数据库凭证

需要访问外部资源时，使用：

短期凭证；

只读凭证；

单仓库 token；

单环境 token；

任务结束后自动失效的 token。

OWASP 建议使用按 Agent 和 session 绑定的身份、短期权限和 JIT 提权，而不是共享长期 service account。(owasp.org)

2. 将数据分为三类即可

对单团队不必建立复杂数据分类系统，可以使用：

分类
示例
Agent 策略
Public / Internal
普通代码、公开文档、测试数据
可在授权工具中使用
Confidential
私有仓库、内部架构、非公开日志
只能进入批准模型和隔离环境
Restricted
密钥、生产用户数据、支付信息、身份数据
默认不进入模型上下文；必须脱敏或使用代理查询

3. 外部内容视为不可信输入

Issue、PR 评论、网页、依赖文档和代码注释可能包含 prompt injection。Claude Code 对联网命令默认要求审批，GitHub 也将 Issue 和评论中的隐藏指令视为攻击面。(Claude Platform Docs)

因此：

外部文档内容
≠
系统指令
≠
可信事实

Agent 从外部内容提取的信息，应标记来源并限制其直接触发工具调用。

八、第七步：审计日志应该记录什么

不需要保存所有模型内部推理，但应保存可复现和可问责的信息。

最小审计事件

类别
记录内容
Task
需求、风险等级、发起人、负责人
Environment
base commit、分支、沙箱配置、模型版本
Permissions
初始权限、提权申请、批准或拒绝
Tools
工具名、命令、参数、时间、exit code
Files
读取范围摘要、修改文件、diff
Network
访问域名、请求类型、返回状态
Verification
执行的测试、结果、benchmark、扫描报告
Decision
推荐方案、证据引用、人工审批人
Delivery
PR、合并 commit、灰度、回滚
Cost
token、执行时间、CI 时间、外部工具成本

GitHub 的 Agent 提交关联 session logs，OWASP 也建议对高风险动作记录结构化决策元数据。(GitHub Docs)

不建议记录

明文 secrets；

生产用户完整数据；

未脱敏请求体；

无限制保存所有上下文；

不受控的模型隐藏推理。

九、第八步：预算与异常停止机制

治理不仅是安全，也包括防止 Agent 无限运行。

建议每个任务声明：

execution_budget:
  max_wall_time: 45m
  max_model_turns: 40
  max_tool_calls: 100
  max_files_changed: 20
  max_diff_lines: 1200
  max_parallel_candidates: 3
  max_ci_runs: 4
  max_external_requests: 10

达到上限后：

停止继续执行
→ 输出当前状态
→ 列出尚未完成事项
→ 请求人工决定是否追加预算

OWASP 明确建议设置 token、成本、重试和工具链限制，避免无界循环造成资源耗尽。(OWASP Cheat Sheet Series)

还应设置 kill switch：

中止 Agent session；

撤销临时 token；

停止容器；

禁止继续推送；

标记分支不可合并；

保留日志和 worktree 供复盘。

十、仓库内应该有哪些治理文件

建议先保持轻量，全部放在仓库里。

repo/
├── AGENTS.md
├── .agent/
│   ├── governance.yaml
│   ├── permissions.yaml
│   ├── risk-rules.yaml
│   ├── approved-commands.yaml
│   ├── network-allowlist.yaml
│   └── task-templates/
│       ├── low-risk-feature.yaml
│       └── high-risk-change.yaml
├── docs/
│   ├── architecture/
│   ├── security-boundaries.md
│   └── agent-operating-guide.md
└── .github/
    ├── CODEOWNERS
    ├── pull_request_template.md
    └── workflows/
        ├── agent-policy-check.yml
        └── agent-evidence-check.yml

governance.yaml 示例

version: 1

defaults:
  network: deny
  filesystem: workspace-write
  production_access: deny
  merge_permission: deny

risk_levels:
  L1:
    max_files_changed: 10
    reviewers: 1
  L2:
    max_files_changed: 25
    reviewers:
      - module-owner
  L3:
    reviewers:
      - module-owner
      - security-or-architecture-owner
    require:
      - rollback_plan
      - canary_plan
      - security_scan

protected_paths:
  - auth/**
  - payment/**
  - migrations/**
  - infra/**
  - .github/workflows/**

always_require_human_approval:
  - add_runtime_dependency
  - modify_database_schema
  - change_public_api
  - access_external_network
  - modify_ci
  - push_remote

PR 模板中增加 Agent 证据区

## Agent execution

- Run ID:
- Initiated by:
- Risk level:
- Permission profile:
- Base commit:

## Evidence

- [ ] Unit tests
- [ ] Integration tests
- [ ] Typecheck
- [ ] Security scan
- [ ] API compatibility
- [ ] Performance baseline comparison

## Boundary crossings

| Action | Reason | Approved by |
|---|---|---|

## Remaining uncertainty

- 

十一、推荐的人类角色分工

单团队不需要专门成立 AI 治理委员会。

角色
Governance Plane 职责
Tech Lead
制定风险分级、权限模板和关键审批规则
Module Owner
审核涉及本模块的 L2/L3 修改
Security 联系人
审核凭证、网络、鉴权和高敏感数据变化
Developer / Task Owner
发起任务、确认需求、负责最终交付
Agent
在权限内执行、报告越界需求、提供证据
CI / Verifier
独立执行硬约束检查

可以采用一个简单责任原则：

任务发起人对需求负责，模块 Owner 对技术接受负责，发布人对上线风险负责，Agent 不承担最终责任。

十二、一次存量模块新增功能的 Governance 流程

假设需求是：

在订单模块增加批量取消功能。

1. 任务进入
   ↓
2. 自动风险识别
   命中：订单核心逻辑、下游副作用、接口变化
   → 风险 L2
   ↓
3. 选择权限模板
   workspace-write
   network-deny
   no-production-credentials
   max 3 worktrees
   ↓
4. Agent 只读探索
   ↓
5. 人确认 Spec 和风险等级
   ↓
6. Agent 创建候选分支并修改
   ↓
7. 越界动作单独审批
   例如新增依赖、修改事件 schema
   ↓
8. 独立 CI 和 verifier
   ↓
9. Draft PR
   Agent 不能批准、不能合并
   ↓
10. Module Owner review
   ↓
11. 灰度与发布审批
   ↓
12. 日志、证据和事故回灌

十三、最小可行版本

第一阶段不建议立刻做所有能力。

第 1 阶段：先控制爆炸半径

Agent 只能在 worktree 工作；

默认禁止公网；

不提供生产凭证；

Agent 不允许合并；

所有 Agent PR 必须人工 review；

保护高风险目录；

保留 session 和工具日志。

第 2 阶段：增加风险分级与自动检查

L0-L3 任务分类；

CODEOWNERS；

Agent PR 模板；

CI policy checks；

依赖、secret、CodeQL 或同类安全检查；

超时和成本预算。

第 3 阶段：减少人工审批摩擦

常用安全命令 allowlist；

根据任务动态生成最小权限；

低风险动作自动批准；

高风险越界动作仍人工审批；

根据历史日志调整规则。

不建议一开始启用 unrestricted/full access。OpenAI 的内部实践指出，过于频繁的审批会导致用户转向 Full Access 或配置过宽规则，从而反而降低安全性；因此目标应该是让沙箱内的日常操作无摩擦，而不是让所有操作都弹窗。(alignment.openai.com)

十四、衡量 Governance 是否有效

不要只统计“拦截了多少动作”，还要评估治理是否妨碍正常交付。

指标
含义
Boundary-crossing rate
每个任务需要越界的动作比例
Human approval count
每个任务的人工审批次数
Approval precision
被拒绝动作中真正危险动作比例
Full-access usage
是否有人绕过治理使用全权限
Policy exception rate
临时放宽规则的频率
Agent incident rate
Agent 导致的安全或数据事故
Mean time to stop
异常发生后停止 Agent 所需时间
Trace completeness
能否重建关键操作和责任链
Cost per accepted change
每个被接受修改的总成本
Governance-induced delay
治理环节增加的交付时间

Governance Plane 的目标函数不是：

尽可能多地阻止 Agent

而是：

在可接受风险下，
尽可能减少低风险工作的摩擦，
并确保高风险行为可控、可见、可回滚。

最终建议

对你描述的团队内半自动需求交付体系，Governance Plane 最适合采用：

$$
\boxed{
\text{仓库内策略}
+
\text{沙箱隔离}
+
\text{动作级权限}
+
\text{风险分级审批}
+
\text{Git/CI 强制门禁}
+
\text{完整审计与停止能力}
}
$$

不要先做平台。优先利用现有组件：

Agent 自带 permissions；

Dev Container 或容器；

Git worktree；

GitHub/GitLab branch protection；

CODEOWNERS；

CI；

Secret Manager；

仓库内 YAML 策略；

PR 模板和 session trace。

只有当多个仓库之间出现大量重复配置、策略漂移和维护成本后，再考虑把其中一小部分抽成共享模板或内部库。

Sources:

Running Codex safely at OpenAI | OpenAI

AI Agent Security - OWASP Cheat Sheet Series

Application card: GitHub Copilot Agents - GitHub Docs

Auto-review of agent actions without synchronous human oversight

MCP02:2025 - Privilege Escalation via Scope Creep | OWASP Foundation

Security - Claude Code Docs
