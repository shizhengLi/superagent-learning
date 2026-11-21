# DSH Superagent 源码精读与面试长文计划

本系列面向 DeepSeek、腾讯、字节跳动、阿里云等公司的 Agent、LLM 应用与平台工程岗位面试。读完后应能从一次外部消息讲到任务交付，画出运行时的关键状态与信任边界，并用源码和测试回答「为什么这么设计」「哪个失败窗口还存在」「换成生产系统，你会先改哪里」。

源码基线为本工作区提交 `7bd4741` 中的 `dsh-superagent-prototype`，包版本为 `0.1.0`。这是一个面向单个 GitHub 仓库的主动式编码 Agent 研究原型，作为 DeepSeek Harness 的 bundle 运行，GitHub 是主要接入源，飞书可选。面试以本地原型为案例，公司名仅用于定位备考场景。库外的模型推理循环、Session 持久化、权限宿主与子进程能力沿依赖接口追溯。 [项目说明](../dsh-superagent-prototype/README.md) · [包清单](../dsh-superagent-prototype/package.json) · [架构说明](../dsh-superagent-prototype/docs/007-architecture.md)

## 连载方式

- 共 **32 篇中文长文**，按编号逐篇交付。目录中的文件名是后续写作路径，进度复选框随正文交付更新。
- 常规篇目标 **6,000–10,000 中文字**，涉及准入、状态一致性、并发与恢复的重点篇目标 **10,000–15,000 中文字或以上**。字数按正文估算，代码、命令和参考链接单列。需要完整讲清的调用链、状态转移、失败窗口和方案取舍均可继续展开。
- 每篇从一个面试问题开始。正文先解释机制的输入、输出、约束与代价，再沿函数调用说明实现。用状态图、时序图、决策表或算例展开关键分支，收尾给出 60–90 秒口述版、3–5 道递进追问及参考回答。
- 每篇引用可定位的源码函数、类型或配置键，并指出相关测试中实际断言的行为。能够离线复现时给出命令、输入和预期观察点，按执行情况记录结果。生产演进方案写明假设、取舍、验证指标和原型现有能力。
- 全系列复用同一个教学案例。一位有权限的维护者在 Issue 中提及 Agent，请它修正文案错字。任务进入 Ticket，Worker 准备工作目录、提交 PR 并报告结果，主 Session 把可交付结果回复到相应会话。维护者在 PR 中补充要求，处理完成后由人合并，GitHub 显示最终合并状态。后续篇章加入飞书合并通知、CI、重复事件、策略修改、暂停、崩溃与回复失败。示例数据使用虚构的仓库、账号和 ID。

## 阅读顺序

| 阶段 | 篇号 | 读完能说明什么 |
| --- | --- | --- |
| 一、架构与运行骨架 | 01–04 | 从产品约束推导模块边界，画出启动顺序和主链路 |
| 二、外部事件与上下文 | 05–09 | 解释 GitHub、飞书如何把不同事件稳定转换成统一输入 |
| 三、授权与任务路由 | 10–15 | 从事件判断权限、选择路径，并解释任务如何随对话延续 |
| 四、执行、协作与交付 | 16–21 | 阐明 Ticket 如何落到 Session、目录、后台任务和 PR |
| 五、持久化、治理与恢复 | 22–28 | 证明事务、队列、幂等、配置、权限和恢复分别解决什么问题 |
| 六、验证与系统设计面试 | 29–32 | 用测试建立证据，将原型演进为可度量的生产设计 |

时间紧时可先读 **01 → 03 → 04 → 10 → 11 → 13 → 16 → 18 → 22 → 23 → 24 → 27 → 30 → 32**，再按面试追问补齐其余篇目。

## 一、架构与运行骨架

### 01 · 什么是主动式 Superagent

- [x] [01-what-is-a-superagent.md](01-what-is-a-superagent.md) · 已交付
- **原理与内容**　从人的注意力成本、持续感知、显式委托和结果回流定义本项目的「主动」。用同一条 Issue 对比交互式编码助手、固定工作流和主动 Agent 的控制权、触发条件、状态跨度与反馈方式，再画出原型和 DSH 宿主的职责边界。
- **面试追问**　主动性可以分成哪些层次？为什么主动监听、主动建单、主动执行需要三种不同的约束？
- **精读依据**　[README](../dsh-superagent-prototype/README.md) · [设计原则](../dsh-superagent-prototype/docs/000-philosophy.md) · [插件入口](../dsh-superagent-prototype/src/index.ts) · [Runtime](../dsh-superagent-prototype/src/app/runtime.ts)

### 02 · Cordis 插件、DSH bundle 与运行时装配

- [x] [02-bundle-and-runtime-bootstrap.md](02-bundle-and-runtime-bootstrap.md) · 已交付
- **原理与内容**　解释依赖注入、插件生命周期与组装入口。沿 `package.json → cordis.patch.yml → apply → Runtime.initialize` 追踪主插件、GitHub/飞书 watcher 和 Web client 如何加载，梳理状态目录锁、管理 API、账号探测、主 Session 创建和后台循环的启动依赖。
- **面试追问**　为什么管理入口可以先于 setup 完成？插件重启时，哪些状态来自 SQLite，哪些来自 DSH Session？
- **精读依据**　[包清单](../dsh-superagent-prototype/package.json) · [bundle 配置](../dsh-superagent-prototype/cordis.patch.yml) · [构建入口](../dsh-superagent-prototype/tsdown.config.ts) · [Runtime](../dsh-superagent-prototype/src/app/runtime.ts) · [bundle 测试](../dsh-superagent-prototype/tests/bundle-patch.spec.ts)

### 03 · 领域模型与 Engine 的事务边界

- [x] [03-domain-model-and-engine.md](03-domain-model-and-engine.md) · 已交付
- **原理与内容**　从事件、状态、授权依据与外部副作用四种职责推导数据模型。画出 Trigger、Thread、Eval、Decision、Mandate、Ticket、Assignment、Input、Channel 的引用关系。沿 `Engine` 的角色接口梳理 Source、主 Session、Worker 和人工管理方能提交的命令，对照内存仓储与 SQLite 仓储。
- **面试追问**　为什么 Thread、Ticket 与 Session 各有身份？为什么一次重试可以换 Assignment，同时沿用 Ticket？
- **精读依据**　[领域类型](../dsh-superagent-prototype/src/core/model.ts) · [Engine](../dsh-superagent-prototype/src/core/engine.ts) · [端口](../dsh-superagent-prototype/src/core/ports.ts) · [内存仓储](../dsh-superagent-prototype/src/core/memory.ts) · [核心测试](../dsh-superagent-prototype/tests/core-engine.spec.ts)

### 04 · 一条 Issue 请求的端到端执行轨迹

- [x] [04-end-to-end-issue-to-pr.md](04-end-to-end-issue-to-pr.md) · 已交付
- **原理与内容**　用一张泳道图串起 Source、Engine、SQLite、Effects、主 Session、Worker、GitHub。沿 `Intake.receive`、`Evals.start`、`orchestrator_route`、Assignment 启动、`orchestrator_report`、Outbox 追踪每一步产生的记录和异步交接点。区分接受请求、回复送达、Worker 就绪、PR 合并等可观察事实。
- **面试追问**　如果建单已提交而 Worker 尚未启动，下一步由谁推动？怎样从一次外部消息找到对应的 Eval、Ticket 和 Worker Session？
- **精读依据**　[Intake](../dsh-superagent-prototype/src/core/intake.ts) · [Evals](../dsh-superagent-prototype/src/core/evals.ts) · [主工具](../dsh-superagent-prototype/src/app/tools/main.ts) · [Executor](../dsh-superagent-prototype/src/executor/dsh.ts) · [运行时测试](../dsh-superagent-prototype/tests/runtime.spec.ts)

## 二、外部事件与上下文

### 05 · Source 插件协议与统一事件模型

- [x] [05-source-contract-and-trigger.md](05-source-contract-and-trigger.md) · 已交付
- **原理与内容**　解释适配器、稳定事件标识、路由实体与回复地址。拆解 `SourceKind`、`SourceContext`、`SourceRegistry`，按注册、配置解析、Scope 过滤、上下文读、Channel 写与执行变量注册梳理接口。对照 Trigger 的 actor、human、addressed、threads、meta 字段。
- **面试追问**　接一个工单系统最少要实现什么？消息、线程与回复地址分别承担什么职责？
- **精读依据**　[SourceKind](../dsh-superagent-prototype/src/sources/kind.ts) · [SourceHost](../dsh-superagent-prototype/src/sources/host.ts) · [SourceRegistry](../dsh-superagent-prototype/src/sources/registry.ts) · [注册测试](../dsh-superagent-prototype/tests/registry.spec.ts)

### 06 · GitHub 轮询、游标与重复事件

- [x] [06-github-polling-and-checkpoints.md](06-github-polling-and-checkpoints.md) · 已交付
- **原理与内容**　逐路梳理通知、仓库级事件流、订阅 Thread、关联 PR 与 CI 检查如何进入系统。讲清 ETag 与游标的配合、游标回退留重叠窗口、失败项重读和 Trigger ID 去重。用同一条评论从多个入口抵达、某一页中断和事件晚到三个案例演算。
- **面试追问**　游标推进过快与过慢各有什么代价？迁移到 webhook 后，如何保留补拉和去重机制？
- **精读依据**　[GitHub Source](../dsh-superagent-prototype/src/sources/github.ts) · [Trigger 转换](../dsh-superagent-prototype/src/sources/github-trigger.ts) · [轮询公用逻辑](../dsh-superagent-prototype/src/sources/poll.ts) · [Source 测试](../dsh-superagent-prototype/tests/sources.spec.ts)

### 07 · 飞书的实时流、历史补拉与话题线程

- [x] [07-feishu-live-backfill-and-threads.md](07-feishu-live-backfill-and-threads.md) · 已交付
- **原理与内容**　分别追踪事件消费、群聊历史补拉、群名刷新和已订阅话题轮询。对照私聊、群聊提及、根消息、话题内回复和历史事件的映射。推演实时流断开后如何补数据，以及新加入群聊、消息撤回、重复到达和权限变化怎样影响可观察结果。
- **面试追问**　为什么实时消费和历史补拉都需要？如何解释「机器人能收到」与「系统授权执行」之间的距离？
- **精读依据**　[飞书 Source](../dsh-superagent-prototype/src/sources/feishu.ts) · [循环与补拉](../dsh-superagent-prototype/src/sources/feishu-loops.ts) · [Trigger 转换](../dsh-superagent-prototype/src/sources/feishu-trigger.ts) · [Source 测试](../dsh-superagent-prototype/tests/sources.spec.ts)

### 08 · Trigger 落库、Thread 关联与订阅扇出

- [x] [08-intake-thread-correlation-and-fanout.md](08-intake-thread-correlation-and-fanout.md) · 已交付
- **原理与内容**　沿 `Intake.receive` 推演时间标准化、ID 幂等、过滤留痕、Thread 关联、按 Ticket 合并扇出和主 Thread 的 Eval 排队。用同一事件关联两个 Thread 的例子，展示两个订阅命中同一 Ticket 时如何合并，以及 context-only 命中对输入的影响。
- **面试追问**　为什么既要保留被过滤的 Trigger，也要保存它的处置记录？为什么订阅命中与任务归属需要分开分析？
- **精读依据**　[Intake](../dsh-superagent-prototype/src/core/intake.ts) · [领域模型](../dsh-superagent-prototype/src/core/model.ts) · [Thread 索引](../dsh-superagent-prototype/src/app/store.ts) · [后续消息测试](../dsh-superagent-prototype/tests/followups.spec.ts)

### 09 · 按需读取上下文与模型可见信息

- [x] [09-bounded-context-and-resource-reads.md](09-bounded-context-and-resource-reads.md) · 已交付
- **原理与内容**　分别分析 Eval 的线程窗口、消息分页、图片引用与评估缓存，以及主 Session/Worker 的 `orchestrator_read`、资源语法和可见性。用飞书长线程和 GitHub 大文件演示引用、游标、offset 与快照怎样限制一次模型输入的体积。
- **面试追问**　为什么先给候选摘要，再让模型读取原文？上下文缺页、过期或引用超出权限时，如何维持判断的可解释性？
- **精读依据**　[MasterAssessor](../dsh-superagent-prototype/src/app/assessment.ts) · [线程窗口](../dsh-superagent-prototype/src/sources/admission-context.ts) · [模型资源](../dsh-superagent-prototype/src/app/resources.ts) · [评估测试](../dsh-superagent-prototype/tests/assessment.spec.ts) · [飞书图片测试](../dsh-superagent-prototype/tests/feishu-image.spec.ts)

## 三、授权与任务路由

### 10 · Policy 准入矩阵与授权强度

- [x] [10-policy-admission-matrix.md](10-policy-admission-matrix.md) · 已交付
- **原理与内容**　用决策表串起 Source 的 listened/observed、成员允许列表、bot 列表、是否提及、私聊、既有 Ticket 和 admission 单元格。逐条说明如何得到 `explicit`、`inferred` 或拒绝，并用维护者、陌生用户、登记的 bot、无发送者的 CI 四类输入走读。
- **面试追问**　为什么「提到机器人」不足以证明委托？为什么系统事件可以进入某些评估路径，但授权强度不同？
- **精读依据**　[Admission 默认值](../dsh-superagent-prototype/src/admission.ts) · [Policy gate](../dsh-superagent-prototype/src/policy.ts) · [授权文档](../dsh-superagent-prototype/docs/002-authorization.md) · [策略测试](../dsh-superagent-prototype/tests/policy.spec.ts)

### 11 · 显式路由与推断评估的双路径

- [x] [11-explicit-routing-and-inferred-eval.md](11-explicit-routing-and-inferred-eval.md) · 已交付
- **原理与内容**　对照 `orchestrator_route` 和 `orchestrator_eval`、`orchestrator_eval_read`、`orchestrator_eval_decide` 的契约。画出模型识别意图、当前规则覆盖检查、方向与证据校验、Engine 复核的职责分工。用 `off / observe / auto` 演算同一推断请求，包括 `would_accept` 的审计意义。
- **面试追问**　为什么结构化输出仍要做业务校验？怎样灰度扩大主动处理范围，同时量化越权建单和漏接？
- **精读依据**　[主工具](../dsh-superagent-prototype/src/app/tools/main.ts) · [MasterAssessor](../dsh-superagent-prototype/src/app/assessment.ts) · [结果转换](../dsh-superagent-prototype/src/policy.ts) · [准入提示词](../dsh-superagent-prototype/prompts/admission.md) · [工具测试](../dsh-superagent-prototype/tests/tools.spec.ts)

### 12 · Thread 级批处理、超时与评估重排

- [x] [12-eval-batching-timeouts-and-requeue.md](12-eval-batching-timeouts-and-requeue.md) · 已交付
- **原理与内容**　拆解 `pending`、`evaluating`、批内主 Trigger 选择、候选交付、deadline 与重排。演算「候选卡片还在等链接」「同一 Thread 多条消息陆续到达」「暂停跨越原截止时间」「旧 Eval 结果迟到」的四条时序，说明哪些消息进入下一批、哪些结果会失效。
- **面试追问**　为什么超时从候选实际送达开始计？批处理如何权衡信息完整性、尾延迟和重复建单？
- **精读依据**　[Evals](../dsh-superagent-prototype/src/core/evals.ts) · [Engine 时钟入口](../dsh-superagent-prototype/src/core/engine.ts) · [评估交付](../dsh-superagent-prototype/src/app/assessment.ts) · [并发测试](../dsh-superagent-prototype/tests/runtime-concurrency.spec.ts)

### 13 · Decision、Mandate 与策略变更后的再验证

- [x] [13-decisions-mandates-and-revalidation.md](13-decisions-mandates-and-revalidation.md) · 已交付
- **原理与内容**　区分 `accept / ignore / reopen / stop / unavailable` 的记录意义，追踪 Mandate 中的 actor、strength、basis、规则与策略版本如何形成授权证据。用移除成员、禁用规则、撤掉 Source Scope 和切换模式推演进行中的 Eval、已有 Ticket 与后续恢复各自的行为。
- **面试追问**　为什么需要记录授权依据？撤销权限为何要区分在途评估、已运行 Worker 与从结束态恢复的任务？
- **精读依据**　[领域记录](../dsh-superagent-prototype/src/core/model.ts) · [policyVersion 与 revalidate](../dsh-superagent-prototype/src/policy.ts) · [Ticket](../dsh-superagent-prototype/src/core/tickets.ts) · [Evals](../dsh-superagent-prototype/src/core/evals.ts) · [策略测试](../dsh-superagent-prototype/tests/policy.spec.ts)

### 14 · 后续消息的归属、转派与退回

- [x] [14-followup-ownership-and-handback.md](14-followup-ownership-and-handback.md) · 已交付
- **原理与内容**　按 `followupOwners`、`attach`、`handback` 追踪已有工作上的人类追问。比较任务主线程、context-only 订阅、显式停止请求和多个 Worker 同时拥有消息时的处理。用两张状态表说明退回条件、最后一个所有者退出后再评估及防止同一 Ticket 接回已退消息的机制。
- **面试追问**　为什么新消息有时直达 Worker？如何避免主 Session 和多个 Worker 重复接单，又保留人工停止通道？
- **精读依据**　[归属判断](../dsh-superagent-prototype/src/core/model.ts) · [Engine](../dsh-superagent-prototype/src/core/engine.ts) · [Ticket 行为](../dsh-superagent-prototype/src/core/tickets.ts) · [Worker 工具](../dsh-superagent-prototype/src/app/tools/executor.ts) · [后续消息测试](../dsh-superagent-prototype/tests/followups.spec.ts)

### 15 · Ticket 生命周期、人工介入与完成语义

- [x] [15-ticket-lifecycle-and-human-handoff.md](15-ticket-lifecycle-and-human-handoff.md) · 已交付
- **原理与内容**　把 Ticket 是否结束、Assignment 状态、Worker 报告、面板投影四个维度放进同一张状态表。逐步复盘 `ready` 无 PR、有 PR 待审、`blocked` 等人回答、投递失败重试、人工停止，以及已结束 Ticket 因人类消息恢复的时序。
- **面试追问**　Worker 已经 ready 时，为什么人仍可能有事要做？怎样从报告、输入和 PR 事实证明任务可以结束？
- **精读依据**　[状态派生](../dsh-superagent-prototype/src/core/stage.ts) · [Ticket](../dsh-superagent-prototype/src/core/tickets.ts) · [Assignment](../dsh-superagent-prototype/src/core/assignments.ts) · [状态测试](../dsh-superagent-prototype/tests/stage.spec.ts)

## 四、执行、协作与交付

### 16 · Orchestrator 与 Worker 的 Session、模型与工具

- [ ] 计划文件 `16-orchestrator-worker-sessions.md` · 10,000–15,000 字或以上
- **原理与内容**　从「决策要看全局，执行要控制范围」推导常驻主 Session 与逐 Ticket Worker。跟踪 `Sessions.ensure`、角色模型解析、prompt/skill 装饰、工具注册与限制。对照 DSH 提供的能力和本项目定义的路由、评估、报告与资源协议，说明一次任务如何跨多个回合。
- **面试追问**　为什么不让一个模型 Session 永远处理所有工单？主 Session 成为瓶颈时，哪些工作可并发，哪些仍需保持路由一致？
- **精读依据**　[Sessions](../dsh-superagent-prototype/src/app/sessions.ts) · [角色模型](../dsh-superagent-prototype/src/app/role-models.ts) · [主工具](../dsh-superagent-prototype/src/app/tools/main.ts) · [Worker 工具](../dsh-superagent-prototype/src/app/tools/executor.ts) · [模型配置测试](../dsh-superagent-prototype/tests/role-models.spec.ts)

### 17 · Assignment 启动、工作区准备与执行环境

- [ ] 计划文件 `17-executor-workspace-and-environment.md` · 6,000–10,000 字
- **原理与内容**　从 `ticketExecution` 和 `launchQueued` 追到 `prepareDirectory`、`Sessions.ensure`、初次任务消息与 started 回写。讲清执行目录如何命名、可配置准备命令何时运行、典型 Git worktree 的用途，以及命令超时、残留目录、重试、工作进程账号与环境变量的关系。
- **面试追问**　如何让准备失败后的重试可预测？工作目录、CLI 包装器和真正的进程权限隔离分别提供什么保证？
- **精读依据**　[Executor](../dsh-superagent-prototype/src/executor/dsh.ts) · [Ticket 环境](../dsh-superagent-prototype/src/app/execution-environment.ts) · [命令环境](../dsh-superagent-prototype/src/infra/environment.ts) · [环境测试](../dsh-superagent-prototype/tests/environment.spec.ts)

### 18 · Worker 槽位、背压与优先级

- [ ] 计划文件 `18-worker-slots-and-backpressure.md` · 10,000–15,000 字或以上
- **原理与内容**　用 `maxWorkers=2` 演算启动预约、运行回合、后台工作占槽、释放与人类唤醒的额外槽。对照 `WorkerSlots`、Effects handler 上限和串行化 Pass，分析慢 prepare、排队、人工追问插入、配置热改与资源饱和。
- **面试追问**　为何启动预约要在第一个 await 之前完成？Worker 槽位限制与供应商每分钟请求/Token 限额怎样协同？
- **精读依据**　[WorkerSlots](../dsh-superagent-prototype/src/executor/dsh.ts) · [Effects](../dsh-superagent-prototype/src/app/effects.ts) · [Passes](../dsh-superagent-prototype/src/app/passes.ts) · [并发测试](../dsh-superagent-prototype/tests/runtime-concurrency.spec.ts)

### 19 · Session 投递幂等与 Worker 已读回执

- [ ] 计划文件 `19-session-delivery-and-read-receipts.md` · 10,000–15,000 字或以上
- **原理与内容**　沿 Ticket Input、`AgentDelivery`、Session inbox、历史记录与 `inputsRead` 画投递状态图。对照 `queue / steer / inject` 的语义，说明稳定消息 ID 怎样在当前 inbox 和历史记录中用于去重，以及何时重新校验 Assignment。推演收到、被回合领取、进入历史及停止并发发生时的结果。
- **面试追问**　为什么「投递成功」和「Worker 已读」要分别记录？崩溃发生在发消息与回写状态之间时，系统凭什么决定是否重发？
- **精读依据**　[AgentDelivery](../dsh-superagent-prototype/src/app/delivery.ts) · [Executor 投递与事件监听](../dsh-superagent-prototype/src/executor/dsh.ts) · [Assignment 回执](../dsh-superagent-prototype/src/core/assignments.ts) · [投递测试](../dsh-superagent-prototype/tests/delivery.spec.ts)

### 20 · 后台命令、子 Agent 与退出清理

- [ ] 计划文件 `20-background-jobs-and-subagents.md` · 6,000–10,000 字
- **原理与内容**　区分模型回合结束、后台命令结束、子 Agent 完成及 Ticket handoff。走读 `workerIdle`、`untilQuiet`、`handOff`、`stopBackground`，解释哪些工作继续占槽、哪些进入人工交接，以及停止或重启时如何收拢后台执行。
- **面试追问**　Agent 显示 idle 后，为什么还可能占用 Worker 槽位？如何防止 Session 已停止而后台任务仍生成新的回合？
- **精读依据**　[后台任务逻辑](../dsh-superagent-prototype/src/executor/background.ts) · [Executor](../dsh-superagent-prototype/src/executor/dsh.ts) · [角色工具限制](../dsh-superagent-prototype/src/app/sessions.ts) · [后台任务测试](../dsh-superagent-prototype/tests/background.spec.ts)

### 21 · PR 归属、Review/CI 与任务交付

- [ ] 计划文件 `21-pull-request-ownership-and-completion.md` · 6,000–10,000 字
- **原理与内容**　从 Worker 报告、GitHub Source 发现、分支关联和 Thread `own` 标记追踪 PR 归属。解释多个 PR、review、CI、PR 重开或关闭、关闭后尚有新输入未处理时的状态变化。对照交付提示词的自动合并约定、GitHub 分支保护的强制约束与源码实际记录的事实。
- **面试追问**　为什么不能把「PR 已创建」直接映射成「任务已完结」？如何防止把别的工单的 PR 算成当前工单的交付？
- **精读依据**　[PR 关联](../dsh-superagent-prototype/src/sources/github-pull.ts) · [报告工具](../dsh-superagent-prototype/src/app/tools/executor.ts) · [结束逻辑](../dsh-superagent-prototype/src/core/tickets.ts) · [交付要求](../dsh-superagent-prototype/prompts/team/delivery.md) · [Source 测试](../dsh-superagent-prototype/tests/sources.spec.ts)

## 五、持久化、治理与恢复

### 22 · SQLite 持久化、索引与迁移

- [ ] 计划文件 `22-sqlite-transactions-indexes-and-migrations.md` · 10,000–15,000 字或以上
- **原理与内容**　讲解 `node:sqlite`、WAL、JSON 记录与生成列索引、Trigger–Thread 关联、typed KV、嵌套 SAVEPOINT 和 `afterCommit`。从提交失败、内层回滚、提交后回调异常和 schema 10→11 迁移失败四个情境说明原子性、持久性和兼容边界。
- **面试追问**　什么时候 JSON 记录配部分索引够用？为什么状态目录锁、SQLite 事务与 Effect 持久化需要各自存在？
- **精读依据**　[Store](../dsh-superagent-prototype/src/app/store.ts) · [读取投影](../dsh-superagent-prototype/src/app/store-read.ts) · [KV](../dsh-superagent-prototype/src/kv.ts) · [迁移](../dsh-superagent-prototype/src/app/migration.ts) · [迁移测试](../dsh-superagent-prototype/tests/migration.spec.ts)

### 23 · 持久 Effects 队列与异步一致性

- [ ] 计划文件 `23-durable-effects-and-consistency.md` · 10,000–15,000 字或以上
- **原理与内容**　沿 assessment、launch、delivery、relay、notice、reply 六类 Effect 讲清事务内入队、提交后派发、按 kind 的并发限制、`seq`、`generation` 和重试退避。用「正在执行时同 key 新 payload 覆盖」与「旧任务取消后再入队」两个并发案例解释完成时为何要匹配旧行。
- **面试追问**　为什么本地事务加重试仍可能重复触达外部服务？哪些 Effect 适合串行，哪些串行后容易发生队首阻塞？
- **精读依据**　[Effects](../dsh-superagent-prototype/src/app/effects.ts) · [Passes](../dsh-superagent-prototype/src/app/passes.ts) · [队列测试](../dsh-superagent-prototype/tests/effects.spec.ts) · [旧队列恢复测试](../dsh-superagent-prototype/tests/queue.spec.ts)

### 24 · 对外回复、Outbox 与通知节制

- [ ] 计划文件 `24-outbox-replies-and-attention.md` · 10,000–15,000 字或以上
- **原理与内容**　区分确认、答复、需人处理的提醒与结果通知，追到回复目的校验、Channel 选择、持久回执、重试和 `channelGone / channelIgnored`。解释同 Ticket 同会话的顺序约束与失败阈值，进一步分析外部已收消息、回执仍 pending 的崩溃窗口。
- **面试追问**　怎样让可观测的外发失败与用户通知节制同时成立？为什么存在本地幂等键仍需要分析 GitHub/飞书接口的幂等能力？
- **精读依据**　[回复规则](../dsh-superagent-prototype/src/core/replies.ts) · [Outbox](../dsh-superagent-prototype/src/app/outbox.ts) · [GitHub Channel](../dsh-superagent-prototype/src/sources/github-channel.ts) · [飞书 Channel](../dsh-superagent-prototype/src/sources/feishu-channel.ts) · [Outbox 测试](../dsh-superagent-prototype/tests/outbox.spec.ts)

### 25 · Prompt、Skill 与模型接口的版本治理

- [ ] 计划文件 `25-prompt-skill-and-model-surface.md` · 6,000–10,000 字
- **原理与内容**　拆解内置 prompt、团队 communication/scope/delivery、运行时环境与 Policy 段的装配顺序。跟踪内置片段覆盖、Skill 注册、Prompt revision、乐观并发和 Session 使用的版本，讲清提示词、Zod schema、工具权限与业务校验各自约束什么。
- **面试追问**　可以按 Prompt revision 复盘一次决策到什么程度？如何验证 Prompt 里提到的工具和资源在相应角色中确实可用？
- **精读依据**　[Prompt 组装](../dsh-superagent-prototype/src/app/prompts.ts) · [版本仓储](../dsh-superagent-prototype/src/app/versions.ts) · [Session 绑定](../dsh-superagent-prototype/src/app/sessions.ts) · [提示词测试](../dsh-superagent-prototype/tests/prompts.spec.ts) · [模型接口测试](../dsh-superagent-prototype/tests/model-surface.spec.ts)

### 26 · 设置、权限历史、管理 API 与运营面板

- [ ] 计划文件 `26-settings-api-and-operations-panel.md` · 6,000–10,000 字
- **原理与内容**　从 React 面板轮询、共享 API 类型和后端视图追到 SQLite 中的运行记录。覆盖 Overview、Source、Trigger、Eval、Execution、Settings 的排查价值，以及按配置块保存的 revision、异步检查后的二次冲突检测、Policy 变更记录和账号配置重启。
- **面试追问**　怎样区分「来源没收到」「准入被拒」「Worker 排队」「消息外发失败」？两个操作员改同一块配置或各改一块时会发生什么？
- **精读依据**　[管理路由](../dsh-superagent-prototype/src/app/http.ts) · [Settings](../dsh-superagent-prototype/src/app/settings.ts) · [配置历史](../dsh-superagent-prototype/src/app/settings-history.ts) · [Overview 视图](../dsh-superagent-prototype/src/app/views/overview.ts) · [前端轮询](../dsh-superagent-prototype/src/client/api.ts) · [设置测试](../dsh-superagent-prototype/tests/settings.spec.ts)

### 27 · 信任边界、最小权限与人工控制

- [ ] 计划文件 `27-trust-boundaries-and-permissions.md` · 10,000–15,000 字或以上
- **原理与内容**　画出外部消息、策略判定、模型和工具、进程、文件系统、第三方账号之间的边界。逐层检查 Source 范围、Mandate、角色 preset、工具 allowlist、Ticket 结束 guard、panel-only 设置、管理 API 同源校验及命令环境。用防御性案例说明怎样评审提示词注入、越权读写、密钥暴露和误合并风险。
- **面试追问**　哪些约束由代码或外部权限系统执行？若要给不同团队开放服务，在哪些层补身份、授权、凭据隔离和审计？
- **精读依据**　[Session 工具控制](../dsh-superagent-prototype/src/app/sessions.ts) · [preset 配置](../dsh-superagent-prototype/cordis.patch.yml) · [执行环境](../dsh-superagent-prototype/src/infra/environment.ts) · [HTTP 管理边界](../dsh-superagent-prototype/src/app/http.ts) · [部署权限](../dsh-superagent-prototype/docs/006-deployment.md)

### 28 · 暂停、关闭、崩溃恢复与故障排查

- [ ] 计划文件 `28-pause-shutdown-and-crash-recovery.md` · 10,000–15,000 字或以上
- **原理与内容**　用故障矩阵列出 Source 读取失败、Eval 超时、准备失败、模型临时错误、输入投递失败和外发失败各有何证据、自动重试及人工处理入口。对比 pause、插件重启和进程崩溃，重点推演 running Assignment 对账、pending launch 补排、Session inbox 保留和 Effects drain。
- **面试追问**　暂停时哪些流程继续记录状态？进程在任意两个持久化动作之间终止后，系统在哪些点能恢复，哪些点需要对账？
- **精读依据**　[Runtime 生命周期](../dsh-superagent-prototype/src/app/runtime.ts) · [Executor 恢复](../dsh-superagent-prototype/src/executor/dsh.ts) · [Effects](../dsh-superagent-prototype/src/app/effects.ts) · [模型重试健康](../dsh-superagent-prototype/src/app/retry-health.ts) · [运行时并发测试](../dsh-superagent-prototype/tests/runtime-concurrency.spec.ts)

## 六、验证与系统设计面试

### 29 · 如何用测试证明这个 Agent 可信

- [ ] 计划文件 `29-testing-agent-invariants.md` · 6,000–10,000 字
- **原理与内容**　从仓储单测、领域状态机、虚拟宿主、并发编排、模型接口契约、面板测试和构建检查整理证据层次。精读「已停止 Ticket 不接收迟到输入」「事务回滚后的 Effect 不执行」「迁移中途失败会回滚」等测试，解释如何构造竞态与断言不变量。再设计实际模型评估所需的样例、人工标注和重复运行。
- **面试追问**　mock 通过能证明什么？如何设计授权误接率、重复建单率和有效交付率的离线评估？
- **精读依据**　[测试配置](../dsh-superagent-prototype/vitest.config.ts) · [宿主夹具](../dsh-superagent-prototype/tests/fixtures/host.ts) · [核心测试](../dsh-superagent-prototype/tests/core-engine.spec.ts) · [并发测试](../dsh-superagent-prototype/tests/runtime-concurrency.spec.ts) · [模型接口测试](../dsh-superagent-prototype/tests/model-surface.spec.ts)

### 30 · 单仓库原型如何演进到生产级多租户

- [ ] 计划文件 `30-production-scaling-and-multitenancy.md` · 10,000–15,000 字或以上
- **原理与内容**　以单仓库、状态目录锁、SQLite 和进程内 Worker 上限为起点，做需求与容量估算。比较每仓库独立实例、共享控制面与隔离执行面两条路线，推导租户分区键、可持久化队列、租约与 fencing、工作区生命周期、模型配额、审计及上线迁移。为每项演进列出代价与启用条件。
- **面试追问**　给出明确的峰值请求、到达率和处理时长后，怎样估算槽位与积压？怎样避免一个仓库的重试风暴拖慢其他仓库？
- **精读依据**　[架构边界](../dsh-superagent-prototype/docs/007-architecture.md) · [Runtime](../dsh-superagent-prototype/src/app/runtime.ts) · [Store](../dsh-superagent-prototype/src/app/store.ts) · [Effects](../dsh-superagent-prototype/src/app/effects.ts)

### 31 · 成本、时延、质量与有限主动性的评估

- [ ] 计划文件 `31-evaluation-metrics-cost-and-latency.md` · 6,000–10,000 字
- **原理与内容**　利用 Trigger、Decision、Assignment、Outbox 与健康记录，设计授权错误率、建单率、重复率、人工介入率、交付率、恢复时长和消息噪声指标。将端到端时延拆成摄取、评估、排队、执行、等待人工与外发，结合教学案例计算分母和采样原则，说明如何从 observe 到 auto 逐步上线。
- **面试追问**　判断主动 Agent 价值时，为什么不能只看接了多少任务？哪些指标能由现有记录推算，哪些需要增加埋点或人工判定？
- **精读依据**　[审计记录定义](../dsh-superagent-prototype/src/core/model.ts) · [策略模式与审计](../dsh-superagent-prototype/src/policy.ts) · [Overview](../dsh-superagent-prototype/src/app/views/overview.ts) · [记录视图](../dsh-superagent-prototype/src/app/views/records.ts)

### 32 · 系统设计白板题与项目面试答辩

- [ ] 计划文件 `32-system-design-interview-playbook.md` · 10,000–15,000 字或以上
- **原理与内容**　准备 90 秒项目介绍、5 分钟源码链路、30 分钟白板设计三套话术。围绕 Harness 与模型协作、IM 协同与权限、事件吞吐与 Agent 调度、云上隔离与可靠交付四类岗位视角组织深挖题。每组提供一条源码证据链、一段失败推演、一项替代设计及度量方案。
- **面试追问**　现场串讲「飞书群提出明确请求 → GitHub 开 PR → 人类补充意见 → 合并与回执」，再推演主 Session 不可用、权限中途撤销和远端已回复但本地回执未写入三个情境。
- **精读依据**　[架构全景](../dsh-superagent-prototype/docs/007-architecture.md) · [授权规则](../dsh-superagent-prototype/docs/002-authorization.md) · [执行行为](../dsh-superagent-prototype/docs/003-execution.md) · [接入新 Source](../dsh-superagent-prototype/docs/008-cookbook.md) · [管理视图](../dsh-superagent-prototype/docs/005-panel.md)

## 单篇交付检查

1. 首屏给出该篇要解决的面试问题，并交代它在全链路的位置。
2. 解释机制的工作原理，给出一条可追踪的正常路径与至少一个有代表性的失败或并发路径。
3. 核对源码链接、函数名和测试断言。文档描述与实现有差异时，记录实际调用链和差异所在。
4. 检查原型能力与生产设计的陈述，给后者补上假设和验证方式。回顾案例中的 ID、状态、操作方和时间顺序。
5. 完成口述版与追问答案，按计划路径保存正文并更新本页进度。
