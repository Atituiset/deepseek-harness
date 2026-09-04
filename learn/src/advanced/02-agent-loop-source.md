# 第 9 章 Agent Loop 源码精读

## 本章目标

- 逐行读懂默认 Agent 驱动 `ReactLoopAgent`：一个不到 500 行的类，是整个产品的发动机。
- 理解 turn / step 两级状态机如何与持久事件一一对应。
- 理解错误、取消、重试在循环里的归宿。
- 读完本章，上一章的流水线图对你应该不再是文档，而是代码。

源文件：`packages/core/agent-loop/src/agent.ts`（496 行）、`packages/core/agent-loop/src/tool-calls.ts`（289 行）、`packages/core/agent-loop/src/index.ts`（713 行）。建议打开编辑器对照阅读。

## 类的全貌

`ReactLoopAgent` 定义在 `packages/core/agent-loop/src/agent.ts:70`，实现 `Agent` 接口。构造函数（:88-104）里值得注意的三件事：

1. `this.dispatch = agentEvents(loopCtx, this)`（:94）——一个融合的事件分发器，构造一次，热路径上零分配。
2. `this.inbox = new Inbox(session, {...})`（:95-101）——输入队列，三个回调分别发 `agent/inbox/inserted`、`discarded`、`claimed` 活体事件。
3. `this.ctx = this.scope.ctx.extend({ agent: this })`（:104）——每个 agent 有自己的作用域 context，`agent.ctx` 上注册的东西只对这个 agent 可见（第 14 章详述）。

驱动的相位用 discriminated union 表达（`agent.ts:39-48`）：`idle | maintenance | running`。`running` 相位携带 `abort`、`turn`、`step`——**turn 和 step 编号就住在相位对象里**，状态即事实。

## 输入路由：send 与三个糖

`send(message, target, wakeup)` 在 `agent.ts:122-129`：把消息 splice 进 inbox 的 `next-turn` 或 `next-step` 队列，`wakeup` 为真则唤醒驱动。注意 :125 的细节：唤醒输入不能加入一个已被 abort 的活动，所以它被重分类到 `next-turn`，开启新 turn。

三个公开方法是 `send` 的参数化糖：

```ts
followup(input) → send(input, 'next-turn', true)   // agent.ts:131-133
steer(input)    → send(input, 'next-step', true)   // agent.ts:135-137
inject(input)   → send(input, 'next-step', false)  // agent.ts:139-141
```

- `followup`：追加到当前工作之后，开新 turn。
- `steer`：插到下一个 step 边界，立即生效——这就是「打断但不取消」。
- `inject`：塞进队列但不唤醒，等下一条唤醒消息一起被认领。文件变更通知、skill 内容这类上下文都走这条路。

`cancel(cause, options)`（:143-149）清空 inbox 并 abort 当前相位的 controller——整个循环的取消都建立在 `AbortSignal` 上，没有第二套取消机制。

## kick：驱动入口

`kick()` 在 `agent.ts:219-232`，是整个驱动的发动机，核心只有一行：

```ts
while (await this.turn()) {}
```

`turn()` 返回 `true` 表示「inbox 里还有工作，再来一圈」。`try/catch/finally` 里体现了容错的边界：已报告的失败和取消在驱动边界被收容（catch 块什么都不做，因为错误已经通过 `agent/error` 事件报告过）；`finally` 把相位落回 `idle`，如果期间有 wake 被闩住（`wakeRequested`）且 inbox 仍有未决工作，立刻再起一个驱动（:229）。

## turn：一个 turn 的生死

`turn()` 在 `agent.ts:255-339`，是本章的主菜。按顺序看它做的事：

**1. 开 turn（:262-268）。** `session.append('turn/start', { turn })`——注意这个 append 发生在认领任何输入之前。即使后面 pre-step 拒绝了全部输入，这个 turn 边界也已经在日志里：日志记录尝试，不只为成功记账。

**2. step 循环（:272-310）。** `while (true)` 里每圈先 `preStep` 认领输入：

- `decision.kind === 'reject'` → `turnEnds = { kind: 'blocked' }`，返回 `false`（:276-279）。pre-step 的拒绝是一个结构化的 turn 结局，不是异常。
- 首轮（`phase.step === 0`）enter 决策被改写为空消息 → `turnEnds = { kind: 'completed' }`，不消耗模型调用（:283-286）。注释解释了为什么：被移除的唤醒消息或被清空的 enter 仍然拥有这个 turn 边界，只是不花模型调用。
- 否则 `session.append('step/start', ...)`（:288），把决策里的消息逐条 append 为 `user/message`（:291-293，带 `surfaceOp: 'append'`——它们要进入模型历史），然后执行 `this.step(decision.assembly, decision.startsRequestSeries === true)`——enter 决策可以携带 `startsRequestSeries` 开启一个新的模型消息系列，第 8 章讲过的 `request/header { reason: 'series' }` 事件就是从这里落盘的。

**3. turn-stopping 串行门（:304-308）。** 一个 step 正常结束且 next-step 队列已空时，先过 `agent/turn-stopping` 这个 serial 事件——监听者（比如 goal 策略）可以在这里注入新工作阻止收尾。过了门之后再检查一次队列：门里的监听者可能恰好放了新消息进来。

**4. 结构化收尾（:311-332）。** catch 块把一切都归约为 `TurnEndReason`：abort 时记 `{ kind: 'aborted', reason }`；其它错误记 `{ kind: 'error', error }`——`LlmError` 保留它的结构化事实，别的异常扁平化成 `errorChain` 文本、code 为 `UNKNOWN`（:318-323）。`finally` 里无条件 append `turn/end`（:328）。**每个 turn 都有结局，结局永远是日志里的结构化数据**，没有「悄悄死掉」的 turn。

**5. 接续（:333-338）。** inbox 空了返回 `false` 让 `kick` 收工；否则换一个新的 `AbortController`（旧 controller 上的闩锁随之失效），step 归零，返回 `true` 进入下一个 turn。

## 相位与维护窗口

回头看相位 union（`agent.ts:39-48`）的三个成员，它们回答了「agent 现在在干什么」的全部问题：

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | { kind: 'maintenance'; abort: AbortController; lastTurn: number; wakeRequested: boolean }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

`status` getter（:108-120）把 `maintenance` 对外也报成 `idle`——compaction 这类维护工作发生在 `runMaintenance`（:151-171）里：它占住相位（此时 `kick` 起不来），但不算模型活动；期间到达的 wake 被闩住（`wakeRequested`），维护结束且 inbox 有货时立即重放（:167）。「闩住-重放」是这个类处理并发唤醒的统一手法，`wakeDriver`（:181-202）里同一份逻辑又出现一次：live 驱动不闩（它自己会认领队列），maintenance 和已 abort 的驱动才闩。

`whenIdle()`（:204-217）的实现也值得一读：循环等待 `activityDone`，直到等待期间没有新活动替换它——这是「等它真正闲下来」而不是「等某一次活动结束」。

## preStep：模型看到什么由 waterfall 决定

`preStep()` 在 `agent.ts:234-253`：

1. `this.inbox.claim(target, position.turn)`（:238）——认领排队输入，触发 `agent/inbox/claimed`。
2. `ctx.systemPrompt.assemble(...)`（:239）——装配提示词节与工具 schema。
3. `agent/pre-step` waterfall（:243-249）——默认实现原样放行（`kind: 'enter'`），监听者可以改写消息或返回 `{ kind: 'reject' }`。

这一步把「模型看到什么」完全交给扩展点：持久化的是结果（`user/message` 事件），决策过程是活体 waterfall。第 8 章的「agent/pre-step 决定模型看到什么」就是这 18 行。

## step：一次模型请求的全程

`step()` 在 `agent.ts:341-443`。外层 `while (true)` 是重试循环，每一圈：

**1. 构造请求（:350-355）。** `buildRequest(...)` 拿到冻结的请求对象；`this.session.deriveMessages()` 就在这里——**模型历史在每次请求前从日志现算**，内存里没有第二份历史副本。

**2. 流式消费（:361-376）。** `preparedCall?.stream(request) ?? ctx.llm.stream(request)`，然后逐 chunk：

```ts
for await (const chunk of stream) {
  signal.throwIfAborted()
  chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
  assembler.push(chunk)
}
```

每个 chunk 先落盘再进 `BlockAssembler`——顺序不能反，落盘的 chunk 是回放保真的依据，`chunkSeqs` 记下它们的 seq 供后面引用。

**3. 错误归宿（:388-404）。** 流以 `finish.kind === 'error' | 'aborted'` 结束时，走 `agent/request-error` waterfall 问策略层怎么办。`llm-retry` 插件就在这里决定是否重试（见第 13 章）：返回 `{ kind: 'retry' }` 就 `continue` 再来一圈；否则抛 `LlmError`，由 `turn()` 的 catch 归约为 `turn/end { kind: 'error' }`。

**4. 落盘助手消息（:408-426）。** `assembler.blocks()` 拼出完整消息，append `assistant/message`，`sourceEventSeqs: chunkSeqs` 把消息和产生它的 chunk 序列绑在一起——这条引用链让 UI 可以从消息回溯到 token 级原始流。

**5. 工具调用（:431-437）。** 消息里没有 `tool-call` block → `{ kind: 'completed' }`；有则交给 `executeToolCalls`，它返回的 `concluded` 标记决定这个 turn 是就此结束还是再转一圈（工具结果会作为下一轮输入）。

## buildRequest：瀑布、绑定与落盘

`buildRequest()` 在 `agent.ts:444-545`，职责是把「这次请求的确切配置」钉死。三个阶段：

**1. 种子配置（:462-480）。** 从持久化的 request header 恢复显式设置（`reasoningEffort` 只在 provider/model 完全匹配且非 adapter 默认值时恢复，:465-469），加上本次的 provider/model 路由，`deepFreeze(structuredClone(...))` 冻结。

**2. `agent/request` waterfall（:481-488）。** 插件在这里改请求配置——换模型、调参数。waterfall 出来如果没有 provider/model 直接抛错：路由必须可解析。

**3. 绑定 adapter 并落盘（:489-540）。** `ctx.llm.prepareCall(proposedConfig, signal)` 把配置绑定到具体 adapter 注册（:494），得到的 `preparedCall` 自带该 adapter 的精确默认值和 `retryPolicy`。随后 `request/header`（:503-515，仅在变化时追加，`reason` 区分 `initial | resume | change`）和 `request/context`（:527-531，路由或上下文窗口变化时）落盘。最后构造的请求对象（:532-540）同样是 `deepFreeze` 的——**发出请求的每一个字节都必须能从事后的日志重建**，header 事件就是重建的原料。

## executeToolCalls：并行池与屏障

`packages/core/agent-loop/src/tool-calls.ts:60` 的 `executeToolCalls` 调度一个 step 的全部工具调用。要点：

- **模式分组（:85-100）**：按 `ctx.tools.executionMode()` 把调用分成并行组和独占调用；独占调用构成屏障——它前面的并行池排空后它才执行，它执行时后面什么都拿不到。
- **有界滚动池（:199-214）**：`fillPool` 维持 `inFlight.size < maxParallelToolCalls`。上限从 `ctx.agentLoop.config` 读取（`tool-calls.ts:132`），默认值 10（`packages/core/agent-loop/src/constants.ts:7`），可通过 agent-loop 的 Config 在 cordis.yml 里改。
- **持久化先于执行**：`appendToolCall`（:263）在开始执行前 append `tool/call`；`appendToolResult`（:269）按模型顺序提交 `tool/result`，`sourceEventSeqs: [callSeq]` 把结果链回调用（:289）。dispatch 可以乱序完成，**提交永远按模型顺序**（`commitReady`，:147-161）。
- **abort 的账也要记**：被取消但已开始的调用照常提交结果；没开始的调用收到一条合成的错误 `tool/result`（`appendSkippedToolCall`，:97），code 为 `TOOL_ABORTED_BEFORE_DISPATCH`——回放时每个 `tool/call` 都有配对的 `tool/result`，日志永远自洽。

## AgentLoop 服务：工厂与事务

`packages/core/agent-loop/src/index.ts:359` 的 `AgentLoop` 是包级别的 Cordis Service（`ctx.agentLoop`），负责按配置创建和恢复 agent。`static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt']`（:360）一行列出了驱动的全部依赖——这就是 loop 需要的整个世界。

创建路径汇于 `setupAndPublish`（:793-813）：准备 session → 构造 `ReactLoopAgent` → 跑 `CreateAgentOptions.setup` 组合窗口（:806，scope 和 agent 已存在但尚未发布，插件在此注册 agent 专属的东西）→ 发布。任何一步失败都会 dispose 掉半成品——**agent 要么完整发布，要么不存在**，没有中间态泄漏。

Config 面（:317-341）只有两个旋钮：`maxParallelToolCalls` 和预创建/恢复的 `agents` 列表（每项带 `id`、`provider`、`model`、可选 `sessionId`/`resumeSessionId`/`cwd`）。`validateConfiguredAgents`（:341-356）在任何 agent 启动前拒绝自相矛盾的配置（`sessionId` 与 `resumeSessionId` 互斥、精确身份不可重复）——「配置错误在加载期炸响」的仓库惯例。

## 读码路线建议

第一遍按调用顺序读：`index.ts` 的 `AgentLoop`（怎么造出来）→ `agent.ts` 的构造函数与 `send`（怎么喂输入）→ `kick`/`turn`/`preStep`/`step`/`buildRequest`（怎么跑一圈）→ `tool-calls.ts`（工具怎么调度）。第二遍带着问题读：取消怎么传播（追 `signal`）、错误怎么归一（追 `LlmError` 与 `errorChain`）、持久化顺序（追每个 `session.append` 的先后）。同目录还有 `runtime-context.ts`（运行时上下文投影）与 `constants.ts`（默认值），读完主文件再翻。

## 动手练习

**练习 1：从日志还原一个 turn。** 跑一次 `DEEPSEEK_API_KEY=... pnpm dsh --profile headless "用 bash 列出当前目录"`，然后在 harness home 的 sessions 目录下找到本次会话的 JSONL 日志（`packages/bundle/base/cordis.patch.yml` 里 persistence 行的 `root: !!js dshHomePath('sessions')`）。用 `jq -r .type` 或 `grep -o '"type":"[^"]*"'` 列出事件类型序列，对照本章的流水线：找到 `turn/start` → `step/start` → `user/message` → `assistant/chunk*` → `assistant/message` → `tool/call` → `tool/result` → `step/end` → `turn/end` 的完整链条，并回答：`assistant/message` 的 `sourceEventSeqs` 指向哪些事件？

**练习 2：观察 pre-step 拒绝。** 写一个临时插件挂到示例组合上，在 `agent/pre-step` 里对包含特定关键词的消息返回 `{ kind: 'reject' }`（记得其它消息调 `next()`）。用 headless profile 发一条命中关键词的任务，在会话日志里确认出现了 `turn/start` → `turn/end { kind: 'blocked' }`，且没有任何 `step/start`。

**练习 3：调并行度。** 在 cordis.yml 的 agent-loop 配置里把 `maxParallelToolCalls` 设为 1，让模型一次提出多个只读工具调用（比如「同时读这三个文件」），对比改之前日志里 `tool/call`/`tool/result` 的交错方式，验证串行模式下调用不再重叠。

## 延伸阅读

- `docs/agent-lifecycle.md` — 同一过程的时序图视角。
- `docs/subsystems/core.md` — Agent handle 的取消与错误恢复契约。
- `packages/core/agent/src/inbox.ts` — 输入队列的实现，`claimed/discarded` 事件的来源。
- 上一章 [第 8 章 架构总览](./01-architecture.md)；下一章 [第 10 章 Session](./03-session-events.md) 讲这些事件落进去的那本账。
