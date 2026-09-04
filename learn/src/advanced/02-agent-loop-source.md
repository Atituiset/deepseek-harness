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

## 深读一：并发唤醒的闩锁协议

这个类最难写对的不是 turn/step 本身，而是「任意时刻来一条唤醒消息，驱动不能丢、不能重、不能和取消打架」。它用一套**闩锁-重放**协议收束了这个问题，值得逐行拆：

**入口分类（`send`，:122-129）。** 每条消息先回答一个问题：我现在能不能搭上当前活动？关键判断是 `wakingAfterAbort = wakeup && this.phase.kind !== 'idle' && this.phase.abort.signal.aborted`——唤醒输入不能加入一个**已被 abort 的活动**，所以它被重分类到 `next-turn`，开启新 turn。注意这个值在 `inbox.splice` **之前**捕获：splice 观察者可能重入 `cancel`，若之后再读相位就会分类错位。竞态正确性藏在「先快照、再行动」的纪律里。

**三类相位，三种唤醒归宿（`wakeDriver`，:181-202）。**

- `idle`：起驱动。`phase` 换成 `running`，`kick()` 包进 `ctx.agents.withInitiator(this, ...)`——initiator 边界让驱动内部所有代码都能用 `requireInitiator()` 找回「我是哪个 agent」（工具调度器就这么拿 session 的，见下）。
- `running`（活的）：什么都不做，**不闩**。活驱动自己会认领队列——闩了反而要等它收尾才重放，多绕一圈。
- `maintenance` 或 **已 abort 的驱动**：`wakeRequested = true`，闩住。等维护结束 / 驱动收尾的 `finally` 检查 `wakeRequested && this.inbox.hasPending` 再重放。注意例外：`reason?.kind === 'disposed'` 的 abort 不闩——**拆卸永远不等模型回合**，否则关进程会挂在一个正在生成的 turn 上。

这套协议消灭了一个经典 bug：维护窗口期间来了新消息，旧写法要么丢消息、要么在维护中启动驱动造成两个驱动并发写 session。闩锁把「启动新驱动」这个决定推迟到唯一的收敛点。

**`whenIdle()` 的循环等待（:204-209）。** 不是 `await this.activityDone` 一次完事，而是 `do { await (activity = this.activityDone) } while (activity !== this.activityDone)`——先取引用再 await，醒来看活动是否又被新驱动替换了。这是「等它真正闲下来」而不是「等某一次活动结束」，替换竞态下只有循环版本是对的。

**每个 turn 换新的 AbortController（:334-337）。** turn 收尾时 `phase.abort = new AbortController()`，同时清掉 `wakeRequested`——旧 controller 上的闩随之作废，因为活驱动不需要闩（自己认领队列）。这是「旧闩不跨 turn」的不变量，漏掉它就会出现「明明有新工作却永远不启动」的死驱动。

## 深读二：失败分类学与收容边界

一个生产 agent loop 的失败处理是一个**分类学**问题。这个类把每类失败安排了明确的归宿：

| 失败 | 捕获点 | 归宿 | 模型可见 |
|---|---|---|---|
| 流中断（abort 时有部分内容） | `step` 的 catch（:372-389） | `assembler.interruptedBlocks()` 拼出半截消息，append 带 `interrupted: true` 的 `assistant/message` | 是——半截思考保留在历史里 |
| provider 带内失败（`finish: error`） | `step`（:391-408） | 先问 `agent/request-error` waterfall，没人接管就抛 `LlmError` | 否——变成 turn 结局 |
| 其它一切异常 | `turn` 的 catch（:311-324） | `LlmError` 保留结构化 `failure`，别的异常扁平化为 `errorChain(error)` 文本 + `UNKNOWN` code，append `turn/end { kind: 'error' }` 后 rethrow | 否 |
| 驱动边界 | `kick` 的 catch（:222-223） | **空 catch 块** | 否 |

空 catch 为什么是对的：错误在抛到这里**之前**已经通过 `agent/error` 事件报告过、已经作为 `turn/end` 落盘过。驱动边界只做**收容**（不让 unhandled rejection 杀进程），不做处理。这就是「每个失败恰好有一个所有者」——传两遍会双记，传零遍会丢。

两个容易忽略的细节：

- **abort 分支不吞错误**（:312-315）：`signal.aborted` 时记 `{ kind: 'aborted' }` 后**继续 rethrow**，让上游的 await 链感知取消。取消不是错误，但它必须沿着异步链传播。
- **max-tokens 粘性**（:294-299）：一旦某个 step 撞了输出上限，后续正常完成的 step **不能降级** turn 结局。`if (turnEnds === null || turnEnds.kind !== 'max-tokens') turnEnds = stepEnd`——没有这行，一个「先撞墙再正常收尾」的 turn 会被记成 completed，依赖 turn 结局做重试的策略就会误判。

**给你的 agent 项目偷走的模式**：失败处理不是 try/catch 的层数问题，而是给每类失败指定「谁报告、谁落盘、谁收容」三个所有者。这个类 545 行里真正难写的就这两段，其余是流水线。

## 深读三：请求对象的一生（防篡改链）

`buildRequest` 构造的请求对象走了一条**全程防篡改**的路径，每一步都有明确动机：

1. **种子冻结**（:468-477）：`deepFreeze(structuredClone(...))`——structuredClone 切断与持久化 header 的引用，deepFreeze 让下游拿到的就是不可变值。
2. **瀑布提案**（:478-481）：`agent/request` 的监听者要改配置，只能返回**新对象**——旧对象冻着，改不动。
3. **`markAgentLoopRequest`**（:535，实现在 `packages/llm/llm/src/call-config.ts:66`）：把**这个对象身份**记进一个进程级 `WeakSet`。为什么用身份标记而不是加字段？因为加字段会进模型请求、要进日志重建；而 `llm/stream` waterfall 的监听者只需要知道「这是 loop 组装的请求，不是别人直接调 `ctx.llm`」——一个进程内、不落盘、零序列化成本的判别。
4. **header 落盘的三个 reason**（:507-518）：`initial | resume | change | series`。不是每次请求都 append——`headerEquals` 相等就不记。这正是「model-visible ⟺ logged」的精妙平衡：**能从日志重建**不等于**每次都记**，省掉的是 token 和日志噪音，保住的是可重建性。

如果把这套东西移植到你自己的 agent：冻结 + 身份标记 + 按变化落盘，三件套缺一不可。冻结防插件误伤，标记给下游策略判别权，变化式落盘控制成本。

## 读码路线建议

第一遍按调用顺序读：`index.ts` 的 `AgentLoop`（怎么造出来）→ `agent.ts` 的构造函数与 `send`（怎么喂输入）→ `kick`/`turn`/`preStep`/`step`/`buildRequest`（怎么跑一圈）→ `tool-calls.ts`（工具怎么调度）。第二遍带着问题读：取消怎么传播（追 `signal`）、错误怎么归一（追 `LlmError` 与 `errorChain`）、持久化顺序（追每个 `session.append` 的先后）。同目录还有 `runtime-context.ts`（运行时上下文投影）与 `constants.ts`（默认值），读完主文件再翻。

## 动手练习

**练习 1：从日志还原一个 turn。** 跑一次 `DEEPSEEK_API_KEY=... pnpm dsh --profile headless "用 bash 列出当前目录"`，然后在 harness home 的 sessions 目录下找到本次会话的 JSONL 日志（`packages/bundle/base/cordis.patch.yml` 里 persistence 行的 `root: !!js dshHomePath('sessions')`）。用 `jq -r .type` 或 `grep -o '"type":"[^"]*"'` 列出事件类型序列，对照本章的流水线：找到 `turn/start` → `step/start` → `user/message` → `assistant/chunk*` → `assistant/message` → `tool/call` → `tool/result` → `step/end` → `turn/end` 的完整链条，并回答：`assistant/message` 的 `sourceEventSeqs` 指向哪些事件？

**练习 2：观察 pre-step 拒绝。** 写一个临时插件挂到示例组合上，在 `agent/pre-step` 里对包含特定关键词的消息返回 `{ kind: 'reject' }`（记得其它消息调 `next()`）。用 headless profile 发一条命中关键词的任务，在会话日志里确认出现了 `turn/start` → `turn/end { kind: 'blocked' }`，且没有任何 `step/start`。

**练习 3：调并行度。** 在 cordis.yml 的 agent-loop 配置里把 `maxParallelToolCalls` 设为 1，让模型一次提出多个只读工具调用（比如「同时读这三个文件」），对比改之前日志里 `tool/call`/`tool/result` 的交错方式，验证串行模式下调用不再重叠——同时观察 `tool/call` 的 seq 依然全部先于任何 `tool/result`，不变量 A 不因配置改变。

**练习 4：在日志里找闩锁。** 用 web profile 启动一个需要长工具调用的任务（比如让 agent 跑一个几秒的 bash 命令），趁它运行时再发一条 followup 消息。然后在日志里回答：第二条消息的 `user/message` 在哪个 seq？它在当前 turn 的 `step/end` 之后、下一个 `turn/start` 之前吗？——这就是「唤醒不闩、活驱动自己认领」在数据上的样子。再在**任务还没开始时**（maintenance 窗口极难撞上，可以改用「agent 空闲后立刻连发两条」替代）观察两条消息被同一个 turn 认领。

**练习 5：给 abort 写断言。** 跑一个会触发多次工具调用的任务，中途 Ctrl-C。打开日志验证不变量 C：数一下 `tool/call` 与 `tool/result` 的数量是否相等；找出被合成结果覆盖的调用（`error.info.code` 为 `TOOL_ABORTED_BEFORE_DISPATCH`）；确认 `turn/end` 的 `reason.kind` 是 `aborted`。如果流中断时有半截输出，找到那条 `interrupted: true` 的 `assistant/message`——它的 `sourceEventSeqs` 指向哪些 chunk？

## 延伸阅读

- `docs/agent-lifecycle.md` — 同一过程的时序图视角。
- `docs/subsystems/core.md` — Agent handle 的取消与错误恢复契约。
- `packages/core/agent/src/inbox.ts` — 输入队列的实现，`claimed/discarded` 事件的来源。
- 上一章 [第 8 章 架构总览](./01-architecture.md)；下一章 [第 10 章 Session](./03-session-events.md) 讲这些事件落进去的那本账。
