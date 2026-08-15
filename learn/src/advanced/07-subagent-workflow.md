# 第 14 章 子代理与工作流

## 本章目标

- 理解 agent scope：注册怎么做到「只对这一个 agent 生效」。
- 走通子代理接缝：Definition / Provider / delegation Consumer 三角色与 spawn / fork 语义。
- 理解 workflow 能力：模型写的编排脚本如何扇出子代理。
- 掌握 Agent handle 的五种交互原语。

## Agent scope：注册的作用域

第 8 章的扩展点地图最后一行是「把注册限定到单个 agent：用它的 `agent.ctx`」。`docs/glossary.md` 的 agent-scope 条目（:11-21）定义了这套机制：

- **scope** 是 per-agent 注册的单位：一个贡献（工具、提示词节、变量、限制、监听器）要么全局（所有 agent 可见），要么 scoped（恰好属于一个 scope key）。只有两层，扁平——**scoped 注册不会继承给子代理**。
- **agent.ctx** 是 agent 的作用域 context：经它注册的东西既只对该 scope 可见，又随该 scope 销毁——一个事实同时驱动可见性与生命周期。
- **shadowing（遮蔽）** 是就近优先的名字解析：scoped 工具/节/变量只在这个 scope 内替换同名全局项。这是 per-agent 人格、per-agent 工具变体的机制。
- **restriction**（`tools.restrict`）从反方向工作：对全局工具集做交集过滤，被过滤掉的工具在提示词里消失、执行时被拒，与「不存在」不可区分。
- **setup window** 是创建窗口（`CreateAgentOptions.setup`）：scope 和 agent 对象已存在、但 agent 尚未发布、`agent/session-start` 未发、首次提示词未装配——组合子代理的一切都在这里发生。setup 只注册，不驱动。
- **lineage**（父子关系）是数据，不是结构：`parentSession`、持久的 `delegationDepth`、运行时的 `subagentDepth` 都只是记录，**不影响可见性**。

## 子代理接缝

子代理是第 11 章接缝模式的教科书案例，三角色：

- **Definition**：`packages/subagent/subagent`，`SubagentRuntime`（`packages/subagent/subagent/src/index.ts:171`），占有 `ctx.subagents`。核心操作是 `registerProvider(provider)`（:369，重名即抛错）、`start(name, request)`（one-shot 委派）、`startContinuable(spec)`（可持续对话的子代理）等。
- **Providers**：`dsh-subagent-spawn-in-process`（全新子 agent）、`dsh-subagent-fork-in-process`（继承父会话已完成前缀的分叉）、以及跨进程/跨产品的 `dsh-subagent-acp`、`dsh-subagent-claude-code`、`dsh-subagent-codex`、`dsh-subagent-dsh-sdk`。
- **Consumers**：`dsh-tool-subagent`（模型可见的委派工具）、`dsh-tool-subagent-control`（`send_message`）、`dsh-tool-subagent-report`（子代理的 `report`）。

**spawn 与 fork 的区别只在一件事上**：子代理看不看得到父会话的已完成历史——fork 看，spawn 不看（README 里 `inheritsParentContext` 条目；它是描述性的，不代表继承工具或权限）。深度控制由接缝统一拥有：`delegationDepth` 持久在 session header 里且单调递增，恢复的子代理不可能被重新数成顶层（README「Delegation depth」）。

真实组合实例（`examples/headless-agent/cordis.yml:91-137`）：

```yaml
- id: subagent
  name: '@deepseek-ai/dsh-subagent'
- id: subagent-spawn-in-process
  name: '@deepseek-ai/dsh-subagent-spawn-in-process'
  config: { providerName: spawn }
- id: subagent-fork-in-process
  name: '@deepseek-ai/dsh-subagent-fork-in-process'
  config: { providerName: fork }
- id: tool-subagent
  name: '@deepseek-ai/dsh-tool-subagent'
  config:
    provider: spawn
    toolName: subagent
    backgroundMode: continuable
    maxDepth: 1
```

注意 `tool-subagent` 这个 Consumer 被配置了两次（`:111` 与 `:126` 附近），分别以 `spawn` 和 `fork` provider 暴露成 `subagent` 和 `subagent_fork` 两个工具——**同一个 Consumer 包，接不同 provider，产出不同模型工具**。这是接缝「一次替换、全局生效」的正面证据。

配置文件里的注释还记录了一个真实的组合决策（`cordis.yml` fork 工具段上方）：fork 保持 one-shot，因为 continuable 子代理的 `report` 工具与提示词节先于它要复用的历史存在；`run_in_background` 关闭，因为这个例子没挂任务服务。读 cordis.yml 时留意这类注释——它们是组合层面的架构决策记录。

## one-shot 与 continuable：两种子代理生命周期

`SubagentRuntime` 把委派分成两种模式（服务 API 表见 `packages/subagent/subagent/README.md`）：

- **one-shot**：`start(name, request)` 发起一次前台委派，拿回一个 `SubagentRun`，一个结果，没有冷恢复。调用方的 `signal` 是规范取消通道：发布前 abort 则 `start()` 在回滚后 reject；发布后 abort 取消剩余 turn 工作但不隐藏 run id。
- **continuable**：`startContinuable(spec)` 建立一个持久的、可多轮对话的子代理——它的 `report` 工具和提示词节在创建时由 `registerContinuableSetup` 组合进未发布的 scope。父代理经 `followup(parent, childId, content, ...)` 投递后续消息（子代理不在场时从持久化的 Session 冷恢复），子代理经 `reportFrom` 向恰好活着的直系父代理汇报。权限是严格的：followup 只能来自记录在子代理持久 header 里的那个父代理，`interrupt` 用错了父地址直接以 `UNAUTHORIZED` 拒绝。

两种模式的**身份**都是持久的：每次本地启动 append 一条 `subagent/descriptor` 事件（`src/descriptor.ts`），记录 provider 名与生命周期 mode——这是子代理目录（`listChildren`/`listDescendants`）跨进程重建事实的依据。描述符事件是 log-only 的：不进 surface、不出现在模型历史、compaction 也删不掉它。

provider 的启动期能力用 `provider.capabilities` 广告（`outputSchema`、`depthLimit`、`toolFilter`、`persona`），服务在创建子代理**之前**就能拒绝不被支持的请求——能力检查前置到 start，而不是让子代理跑到一半才失败。

## 子代理的组合：preset 的加入

子代理不是裸 agent：每个进程内子代理由一次 `applyChildComposition(childCtx, parent, composition)` 调用组合而成（README「Capabilities」一节）——它先把父代理的 agent-preset 组合 join 进来，再施加子代理自己的人设与工具过滤。join 是子代理能力的来源：模型可见行都在 agent 平面上，一个什么都没 join 的子代理面对模型时工具注册表是空的。把 `parent` 做成必填参数是故意的——让「不 join 就组合子代理」在调用点无法表达。

`childSessionMeta()` 把 join 的 preset id 记进子代理的持久 header，理由和顶层会话记录自己 preset 的理由相同：preset 决定模型看到的工具 schema 和提示词节，冷读子代理历史必须重建那个组合而不是部署默认。它从父代理的**活体 scope 链**读，而不是父 header——一个切了 preset 还没说话的父代理跑在新组合上，但它的 header 还写着旧的。

这条链路把第 8 章的几个概念串了起来：scope（子代理的世界）、setup 窗口（组合发生的时机）、session header（持久身份）、model-visible ⟺ logged（preset 必须可重建）。

## workflow：模型写的编排脚本

workflow 接缝（`packages/workflow/`）的结构同构：

- **Definition**：`dsh-workflow`，占有 `ctx.workflowEngine`，定义脚本、run、结果、错误、事件契约。
- **Provider**：`dsh-workflow-worker-thread`——在 worker 线程里隔离执行脚本的引擎。
- **Consumer**：`dsh-tool-workflow`——模型可见工具，模型写一段 JavaScript 编排脚本，脚本里的 `agent()` 调用经配置选定的子代理 provider 扇出子代理。

契约里两条值得记住：`WorkflowRun.result` 一旦返回就**永不 reject**——执行失败以 `stopReason: 'error'` resolve，取消以 `'cancelled'` resolve；run 是持有者拥有的，引擎插件卸载阻止新 start 但不回收已接受的 run。事件（`workflow/start`、`workflow/end`、`workflow/phase`、`workflow/log`、`workflow/agent-start`/`agent-end`）是只读的：payload 携带 `WorkflowRunInfo` 而不是活 run，监听者拿不到取消/处置权（`packages/workflow/workflow/README.md`）。`WorkflowStartRequest.parent` 把每个子代理归因到发起它的 agent；`subagentProvider` 可以整run 指定子代理路由而不暴露给脚本。

同组的 `dsh-tool-ralph`（`packages/workflow/tool-ralph`）是建在这两个原语之上的策略：一个 Ralph loop 是朝着不变目标运行的一串 fresh-agent 轮次，每轮是一个没有父对话种子的子会话，跨轮状态由共享工作区和一份有界结构化 handoff 承载（定义见 `docs/glossary.md` 的 Ralph 条目）。它的位置值得玩味：**不是** loop 模式、不是调度器、不是 goal——只是一个组合了 workflow 与 subagent 原语的模型可见工具。这又是「一切皆插件」：连「跑一个自治循环」这种事都没进 loop。

headless 例子里 workflow 引擎经 spawn provider 扇出（`examples/headless-agent/cordis.yml:131` 附近）：

```yaml
- id: workflow-worker-thread
  name: '@deepseek-ai/dsh-workflow-worker-thread'
  config: { provider: spawn }
```

换 provider 一行配置，模型的编排脚本就能把子代理派到别的产品里去。

## 原语选择速查

面对一个「让 agent 干活」的需求，按这张表选机制：

- 委派一个独立任务、只要结果 → `tool-subagent` 配 spawn provider（one-shot）。
- 委派的任务需要看到目前对话已完成的部分 → fork provider。
- 需要与子代理多轮来回、子代理主动汇报 → continuable 子代理（`startContinuable` + `report`/`send_message`）。
- 需要模型自己写编排逻辑、扇出多个子代理并汇总 → `tool-workflow`。
- 需要一个朝固定目标自治迭代的循环 → `dsh-tool-ralph`（workflow + subagent 的策略组合，不是 loop 改动）。
- 只是给当前 agent 补一条上下文 → `agent.inject()`，根本不用子代理。

进程外 provider（`dsh-subagent-acp`、`dsh-subagent-claude-code`、`dsh-subagent-codex`、`dsh-subagent-dsh-sdk`）把「子代理」推广到其它产品：一次委派可以是另一个编码 Agent 产品里的一轮对话。接口不变，距离变了——这是接缝抽象价值的极限测试。

## Agent handle：五种交互原语

回到单 agent 层面，`Agent` 接口（`packages/core/agent/src/runtime-types.ts`）暴露了人与程序驱动 agent 的全部原语，实现在第 9 章的 `ReactLoopAgent` 里：

| 方法 | 语义 | 实现 |
|---|---|---|
| `followup(msg)` | 排队为新 turn，立即唤醒 | `runtime-types.ts:124` / `agent.ts:122` |
| `steer(msg)` | 插到下一个 step 边界，立即生效 | `runtime-types.ts:133` / `agent.ts:126` |
| `inject(msg)` | 注入上下文，不唤醒，等下一次认领 | `runtime-types.ts:143` / `agent.ts:130` |
| `cancel(cause, options?)` | abort 当前活动，可保留 inbox | `agent.ts:134` |
| `whenIdle()` | 等当前活动及闩住的唤醒全部排空 | `runtime-types.ts:93` / `agent.ts:195` |

子代理的控制面就建在这五个原语上：`SubagentRuntime.followup` 就是给子 agent 的 inbox 投一条 followup，`interrupt` 是带权限校验的 `cancel({ keepInbox: true })`（`packages/subagent/subagent/README.md` 的服务 API 表）。

读码入口建议：

- 接口契约：`packages/core/agent/src/runtime-types.ts`（每个方法的 JSDoc 就是行为规范）。
- 默认实现：`packages/core/agent-loop/src/agent.ts`（第 9 章已读）。
- 服务面：`packages/subagent/subagent/src/index.ts` 的 `SubagentRuntime`（:171）。
- 委派 Consumer：`packages/subagent/tool-subagent/src/`——看模型参数如何变成 `SubagentStartRequest`。
- workflow 引擎：`packages/workflow/workflow-worker-thread/src/`——看脚本隔离与子代理扇出的接线。

## 动手练习

以下练习都不需要 API key（1、2 除外，它们用真实模型是为了观察事件；换成第 13 章的 echo adapter 也能做）。

**练习 1：观察一次委派。** 用 headless profile 跑一个明显需要委派的任务，例如「spawn 一个子代理统计 learn/ 目录的行数，把结果报回来」。然后对比 `.sessions/` 下父会话与子会话的 JSONL：

- 子会话 header 里的 `parentSession` 与 `delegationDepth` 是什么？
- 子会话第一条 `user/message` 与父会话里的 `tool/call`（`subagent` 工具）参数有什么关系？
- 父会话里 `subagent` 工具的 `tool/result` 内容是什么？

**练习 2：spawn vs fork 对比。** 同一任务分别用 `subagent`（spawn）和 `subagent_fork`（fork）工具委派一次，对比两个子会话的日志长度与首条消息——fork 子代理继承了父会话的已完成前缀，spawn 没有。日志说了算。

**练习 3：用 setup 窗口组合一个受限子代理。** 在组合里给 `tool-subagent` 的请求加上 `toolFilter`（或写一个 `agent/pre-step` 监听器观察子代理的 scope），验证被过滤的工具在子代理的提示词里不存在、直接调用也被拒——且与「不存在」无法区分。

## 延伸阅读

- `packages/subagent/README.md` — 子代理家族总览（实现与 consumer 地图）。
- `docs/subsystems/subagent.md` — 子代理子系统参考。
- `docs/glossary.md` 的 agent-scope 条目 — scope/shadowing/restriction 的精确定义。
- 入门篇 [第 7 章 组合与覆盖](../beginner/07-compose-overlay.md)；下一章 [第 15 章 测试体系](./08-testing.md)。
