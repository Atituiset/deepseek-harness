# 第 12 章 工具执行管线：从 tool/call 到 tool/result

## 本章目标

- 走完一次工具调用的完整管线：五级扩展点各自能做什么、不能做什么。
- 理解 `ToolRuntime` 注册表：注册、守卫、作用域视图、schema 投影。
- 理解 render intent 设计：工具的 UI 呈现为什么是纯函数。
- 知道 PTC 模式是什么、`run_code` 如何复用这条管线。

源文件：`packages/core/tools/src/index.ts`（1946 行）；权威流程图在 `docs/tool-execution-pipeline.md`。

## 管线的五级扩展点

一次工具调用从模型输出的 `tool-call` block 开始，到 `tool/result` 事件落盘结束。中间有五个文档化的扩展点，按序：

**1. `tools/pre-execute`（waterfall，签名在 `packages/core/tools/src/index.ts:144`）。** 决策门。监听器返回三种决策（`PreToolDecision`，:581-584）：

```ts
| { kind: 'allow' }
| { kind: 'deny'; reason: string }
| { kind: 'ask'; reason?: string }
```

`ask` 把决定权交给 `ctx.approval` 的一次性人工提示；审批缺席或无法回答时按 deny 处理。权限、审批、沙箱策略都挂在这里（实际接线见 `packages/interaction/`：`permission-presets`、`user-approval` 等）。

**2. 单调守卫（monotonic guards）。** `guard()` 注册在 :1101，语义在 JSDoc（:1092-1100）：守卫在 pre-execute 之后评估，任何匹配的守卫可以返回拒绝理由，但**没有守卫能 force-allow 一个被别的守卫拒绝的调用**——「单调」即拒绝只能增加不能撤销。注册路径决定作用范围：普通 context 上的守卫是全局的，`agent.ctx` 上的只作用于那个 agent。所有者级的、不允许被重排的硬策略（而非可协商的审批）应该做成守卫而不是 waterfall 监听器。

**3. `tools/execute`（around waterfall，:155）。** 包裹真正的工具体。这是「around」语义：监听器拿到 `next()`，可以在前后包任何东西——超时、重试、指标。`packages/guard/timeout-policy` 的超时就是这里的一个 wrapper。`ToolDispatchExecution`（`index.ts:376` 附近）给了 wrapper 一个可变视图：允许替换并恢复 `exec.signal` 来施加 deadline，但不能移除它。

**4. `tools/post-execute`（waterfall，:167）。** 拿到已执行的结果，决策三种（`PostToolDecision`，:590-593）：`accept`（可替换 content 或 value 之一，不可同时）、`block`（把结果换成对模型的反馈）、以及两者都可携带的 `additionalContexts`（追加进后续请求的上下文）。结果重写、结果拦截、给模型附加上下文都发生在这。

**5. `finalizeContent` + `tools/result`。** 工具自己声明的 `finalizeContent` 是最后一道只作用于 content 的不变量；之后 `tools/result`（:189）是**只读**通知——此时结果已冻结、已落盘定稿，监听器只能观察（UI 更新、遥测）。

把五级连起来（与 `docs/tool-execution-pipeline.md` 的图一致）：

```text
tool/call（agent-loop 落盘，执行前）
  → tools/pre-execute（allow/deny/ask；ask 经 ctx.approval）
  → 单调守卫（deny or abstain）
  → tools/execute（around：超时/重试/指标）→ 工具 execute() 体
  → 注册表外层归一化（快照失败 → isError）
  → tools/post-execute（accept/block/附加上下文）
  → definition.finalizeContent（content 不变量）
  → tools/result（只读，冻结的最终结果）
  → tool/result（agent-loop 落盘，按模型顺序）
```

两个边界职责别记混：**durable 的 `tool/call` 与 `tool/result` 事件是 agent-loop 落的**（`tool-calls.ts:263` 与 :281，见第 9 章），tools 包自己只负责执行与扩展点。管线任何一环抛错都会被归一化为 `isError` 结果而不是炸掉 turn。

选扩展点时的速查表：

- 想**拒绝**一类调用 → 硬策略用 `guard()`（单调、不可被绕过），可协商策略用 `tools/pre-execute` 的 deny/ask。
- 想给调用加**超时/重试/指标** → `tools/execute` 的 around wrapper。
- 想**改写或拦截结果**、给模型**追加上下文** → `tools/post-execute`。
- 只想**观察**最终结果（UI、遥测）→ `tools/result`，它是只读的，改不了任何东西。
- 想改的是**模型看到的工具清单** → 不动管线，动作用域：`register` 的 scoped 注册或 `restrict` 的交集过滤。

选错层的典型后果：把超时写进 `tools/post-execute`（结果已产生，超时无从谈起），或把审批写进 `tools/result`（只读，拒不了）。

## ToolRuntime：注册表的四个面孔

先存一份事件签名速查（都在 `packages/core/tools/src/index.ts` 顶部的事件声明区）：

- `tools/pre-execute`（:144）：`(exec, next) => Promise<PreToolDecision>`
- `tools/execute`（:155）：`(exec, next) => Promise<ToolExecutionResult>`
- `tools/post-execute`（:167）：`(exec, result, next) => Promise<PostToolDecision>`
- `tools/result`（:189）：`(exec, result) => undefined`——返回 `undefined`，从类型上就断了你改结果的念头。

把管线和持久事件对齐到一条时间线上，顺序保证一目了然：

1. 模型输出 `tool-call` block → agent-loop 先落 `tool/call`（执行前）。
2. `tools/pre-execute` → 守卫 → `tools/execute` → 工具体 → `tools/post-execute` → `finalizeContent` → `tools/result`。
3. agent-loop 按**模型顺序**落 `tool/result`（dispatch 可以乱序完成，提交不行）。
4. 结果携带的 `additionalContexts` 在此之后作为 `user/message` 进入下一步的 inbox。
5. 取消时：未启动的调用补一条合成错误 `tool/result`——每个 `tool/call` 必有配对的 `tool/result`，回放永远自洽。

`ToolRuntime`（`packages/core/tools/src/index.ts:780`）是 `ctx.tools` 的实现。四个常用方法：

- `register(definition)`（:1028）——注册全局或调用方 agent 作用域的工具。校验在入口做完：output 必须声明 `{ schema, render, presentationMeta? }`、schema 必须是受支持的 JSON Schema、`timeoutMs` 必须是正有限数（:1037-1040）、`run_code` 是保留名（名字常量 `RUN_CODE_NAME` 在 `ptc.ts`）。返回 disposer——注册是 effect，卸载即注销。作用域工具 shadow 同名全局工具（就近优先）。
- `guard(guard)`（:1101）——注册单调守卫，见上。
- `get(name, scope?)`（:1195）——按某个作用域的视角查工具：被 restrict 掉的全局工具读起来就像不存在。
- `schemas(scope?)`（:1225）——把可见工具投影成模型可见 schema。注意「投影」二字：`schemas()` 只放行 name/description/parameters 等白名单字段，execute、presentation 这些回调永远不进模型请求。超时元数据也绝不模型可见（:247 附近）。

这个白名单投影值得停一下：它是「model-visible ⟺ logged」不变量在工具侧的守门员。注册时你交了完整 definition，但模型请求里出现的只是投影——所以请求可以从日志里的 `request/header` 事件精确重建，不会把函数引用这种不可序列化的东西带进 header。

作用域与限制（`restrict`，:1062）的组合规则：restriction 以**交集**组合，过滤的是继承来的全局工具集，且不影响保留的 PTC 传输工具；作用域自己的注册在过滤之后合并，不受自己的 restriction 影响——这保证了子代理的报告工具不会被父代理给它设的能力过滤器剥掉。

## defineTool 与类型推导

手写 `ToolDefinition` 时参数类型和 schema 容易脱节，`defineTool`（`packages/core/tools/src/schema.ts:545`）解决这个：用 schemastery 写参数 schema，`InferArgs` 从 schema 推导出 `execute(args)` 的精确参数类型——schema 是唯一事实源，类型是它的投影。执行前注册表会用同一 schema 校验模型给的 args，校验不过根本轮不到你的 `execute`。

顺带区分两个时间语义，常被混淆：

- `ToolDefinition.timeoutMs`（:247）只是**声明**：注册表校验它是正有限数，但注册表自己不执行超时。执行者是 `@deepseek-ai/dsh-tool-call-timeout-policy`——一个 `tools/execute` 的 around wrapper（`packages/guard/timeout-policy`）。声明与执行分离，所以超时策略是可替换的插件。
- 结果里的 `additionalContexts` 按活动批次的 FIFO 顺序落地：一批工具全部结算后，这些上下文作为 `user/message` 排进下一步的输入（`docs/tool-execution-pipeline.md` 的 context 节点）。它们不是工具结果的一部分，是结果**之后**的补充材料。

## 呈现：render intent 是纯函数

工具的 UI 呈现是设计期决策，不是事后补的（根 `AGENTS.md`：「A tool's UI render intent is part of its design」）。机制分两层：

- `presentationMeta(args, value): JsonValue`（:210）——结果落盘时投影出的**纯函数**元数据，存进 `tool/result` 事件的 `meta` 字段，回放时 UI 靠它重建卡片。
- `presentCall(args)` / `presentResult(args, result)`——返回 card 标签的 render intent：`generic`（默认）、`terminal`（bash 类命令）、`diff`（文件增改）、`search`（grep/glob 结果）、`read`、`web`（检索/抓取）。词汇定义在 `packages/core/tools/src/presentation.ts`。

纪律只有一条但很硬（`docs/cookbook/adding-a-tool.md` 的 purity 条目）：这两个方法会在**实时流和日志回放**两种路径上跑，所以必须是 args（+result）的纯函数——不许 I/O、不许读 session 状态、不许用时钟和随机数。你在 `presentCall` 里想要文件的旧内容？那属于 durable 结果元数据或 UI 适配器，不属于 presenter。

## 执行身份与并发模式

管线的输入 `ToolExecution` 有几个不可伪造的保证（`docs/cookbook/adding-a-tool.md` 的 execution identity 条目）：注册表把模型给的 `arguments` 一次性物化成无损 JSON 并冻结，分配给这次执行一个不透明的 `exec.token`；`callId`、`name`、`arguments`、`agent`、`token`、调用方提供的 `signal` 在整个 dispatch 期间不可变。只有 around wrapper 拿到可变视图（`ToolDispatchExecution`，:391），且只能替换并恢复 `signal` 来施加 deadline。

工具的并发模式（`ToolExecutionMode`）分 `parallel` 与独占两类，调度器（第 9 章 `tool-calls.ts`）在每次启动前重新查询 `ctx.tools.executionMode(exec)`——所以一个工具可以在注册表里声明「我副作用大，必须独占执行」，调度器会让它成为屏障：前面的并行池排空后它才启动，它跑完后后面的才能动。模式是按调用现场重新分类的，注册表运行期变更会影响尚未启动的调用。

## PTC 模式：一条管线，两种呈现

`ToolPresentationMode`（:644）有三个值：`native`（默认，每个可见工具一个 schema）、`ptc`（模型只看到一个 `run_code` 传输工具 + 生成的 SDK 提示词节，工具调用变成写程序）、`both`。

关键设计：`run_code` 不是一个旁路执行器——它的子调用带着 parent token **重进正常执行管线**（pre-execute、守卫、execute、post-execute 一级不少），所以权限策略对两种呈现一视同仁。注册表侧的配合：任何 agent 都可能给自己选 PTC 模式，所以 `run_code` 名字被无条件保留；`ptc` 模式下模型直接调用只能命中 `run_code`，嵌套子调用才能命中任意可见工具（`resolveExecution`，:1212）。PTC 模式需要一个带注册 SDK 渲染器（TypeScript 或 Python）的 `ctx.codeRuntime` 实现（如 worker-thread provider），缺席时提示词装配直接报可操作的配置错误（`requireCodeRuntime`，:1010）。

子调用与顶层调用的差异也被仔细设计过（`docs/tool-execution-pipeline.md:60`）：子调用带 parent token（`ToolExecution.parent`），日志里记 `tool/code-dispatch`，拒绝以「绑定拒绝」的形式返回给程序而不是吞掉，并且省略 `additionalContexts` 以保持 call/result 相邻。`Config`（:647）里的 `mode` 是部署默认值，agent preset 可以按 agent 覆盖；`run_code` 程序内重叠子调用的并发上限（`maxParallelSubCalls`，:666 的 Config 字段，默认 10）同样在注册表边界解析（:769-773）。

## 动手练习

**练习 1：写一个计时 around 插件。** 挂 `tools/execute`，给每次工具调用记录耗时：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'tool-timing'
export function apply(ctx: Context) {
  ctx.on('tools/execute', async (exec, next) => {
    const started = performance.now()
    try {
      return await next()
    } finally {
      console.error(`[tool-timing] ${exec.name} ${(performance.now() - started).toFixed(1)}ms`)
    }
  })
}
```

跑一次含多次工具调用的任务，观察输出顺序。然后思考：为什么计时适合 `tools/execute` 而不是 `tools/result`？（提示：around 能包住「执行」本身，result 只能看到结果。）

**练习 2：写一个守卫。** 用 `ctx.tools.guard()` 拒绝一切在工作目录之外写文件的工具调用（检查 `exec.name` 与 `exec.arguments`）。验证：被拒的调用产生了 `tool/result` 但 `isError: true`，且工具体从未执行（在你的工具里加个日志确认）。

**练习 3：追一条结果重写。** 在 `tools/post-execute` 里给某个工具的结果追加一段 `additionalContexts`，然后在会话 JSONL 里找到它落地成的 `user/message`（它在哪个 seq？相对 `tool/result` 的位置如何？）。

## 延伸阅读

- `docs/tool-execution-pipeline.md` — 权威流程图（生成文档）。
- `docs/cookbook/adding-a-tool.md` — 加工具的完整 cookbook，含 render intent 设计。
- `packages/core/tools/README.md` — PTC 模式与作用域注册的细节。
- 入门篇 [第 5 章 写你的第一个工具](../beginner/05-first-tool.md)；下一章 [第 13 章 写一个 LLM 适配器](./06-llm-adapter.md)。
