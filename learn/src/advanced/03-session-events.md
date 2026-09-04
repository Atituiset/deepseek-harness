# 第 10 章 Session：事件溯源的真相之源

## 本章目标

- 理解 `Session` 这本只追加的账：append 时发生什么、为什么同步提交、为什么是冻结的。
- 理解 `deriveMessages()` 如何把日志投影成模型历史。
- 学会用 TS 声明合并扩展 `SessionEventMap`，加自己的持久事件。
- 搞清楚持久化为什么不在 core，以及 `SESSION_FORMAT_VERSION = 0` 的预发布语义。

源文件：`packages/core/session/src/index.ts`（1157 行）、`packages/core/session/src/types.ts`（436 行）。

## Session：一个类，一本账

`Session` 定义在 `packages/core/session/src/index.ts:425`。它不是 Cordis Service，而是普通类——活实例经 `ctx.sessions.create()` 创建，离线实例经 `Session.create()` 创建；用已有事件日志做种子就是重放/分叉一个会话。内部状态只有一个 `private log: SessionEvent[]`（:426），外加一个 `SurfaceManager`（:428）负责维护「表面」（surface，后面讲）。

这本账对外提供三种读法，对应三类消费者：

- **原始日志**：`session.snapshotEvents()` —— 日志的不可变快照（可按 seq 区间切片），持久化、遥测、transcript 导出走这里，一字节都不丢。
- **有序表面**：`session.surface` —— 只含产消息事件的当前视图，compaction 的 `replace` 改写的就是它。
- **模型历史**：`session.deriveMessages()` —— 表面的投影，agent-loop 每次请求前现算。

三者是同一份数据的三个视图，改任何历史都只能在 append 边界发生。

两个公开读口先记住：

- `session.snapshotEvents()`（:600-613）——日志的不可变快照，复用到下一次 append 为止；事件及其嵌套数据在入库时已深度冻结，任何代码都改不了历史。fork 出的会话还有 `ownEvents()`（:615-617）只取子代理自己产出的事件。
- `session.seq`（:629-631）——下一个事件的序号，**恒等于日志长度**。`seq = log.length` 这条连续性契约是全系统赖以生存的地基：持久化可以原样存储日志，重放可以拿 seq 当下标。

## append：校验、快照、冻结、同步提交

`append()` 在 `packages/core/session/src/index.ts:668-727`，是全部持久事实唯一的入口。它有资格被逐行读，因为「model-visible ⟺ logged」不变量就押在这 50 行上。流程：

1. **快照与校验（:673-686）**。`snapshotJsonValue(data)` 做一次递归拷贝，同时验证数据是无损 JSON——BigInt、函数、symbol、`undefined`、循环引用、Map/Set/Date 这类东西在这里直接抛错。文档注释（:654-666）解释了为什么：事件日志是持久的真相之源，坏事件必须在 append 现场爆炸，而不是等后端 flush 时才暴露。一次递归完成读取、校验、拷贝，有状态的 getter 没法给校验一个值、给存储另一个值。
2. **surface 校验（:698）**。`surfaceManager.validateNext(event)` 保证候选事件能合法进入表面——失败则日志不被改动，不存在部分变更。
3. **防重入（:687-690）**。append 的发布边界开着时不允许再次 append。
4. **提交与广播（:700-718）**。事件进 `log`，然后通过 store 私有的发布钩子**同步**通知 `session/event` 的订阅者（:705、:710）。注意顺序与容错：事件一旦进日志，append 就已提交——某个观察者抛错只会被逐监听者收容、记日志，不改变返回值，也不妨碍后面的监听者看到同一事件（:633-640 的契约注释）。

「同步提交、异步持久化」是关键设计：热路径从不阻塞在 I/O 上，持久化插件在后台缓冲。

append 的第三个参数 `opts` 是有类型条件的（:671）：只有**产消息的事件**（`SurfaceEventType`：`user/message`、`assistant/message`、`tool/result`，见 `types.ts:375-378`）才必须带 `surfaceOp`；`turn/start`、`assistant/chunk` 这类 log-only 事件带了反而被编译器拒绝。这条静态规则保证「如何进入模型历史」这个事实不可能漏记。

## 表面与 deriveMessages：日志如何变成模型历史

事件分两类：进历史的（surface 事件）和只进日志的（边界标记、chunk、header 等）。表面（surface）是日志之上的有序视图：每个产消息的 append 都记录自己的 `surfaceOp`——`'append'` 加到尾部，或 `{ op: 'replace', start, end }` 替换一段（`types.ts:404-406`，compaction 用它把旧历史换成摘要）。

`deriveMessages()` 在 `packages/core/session/src/index.ts:790-811`：

```ts
deriveMessages(): Message[] {
  const surface = this.surface
  const nodes = surface.nodes
  const generation = surface.replaceGeneration
  if (generation !== this.derivedGeneration) { /* 表面被 replace 改写过，重建缓存 */ }
  for (const seq of nodes.slice(this.derivedNodes)) {
    const msg = this.deriveEventMessage(this.log[seq]!)
    if (msg) this.derived.push(msg)
  }
  ...
}
```

要点：

- **增量缓存**：每个表面节点只在第一次见到时投影一次，一次调用只花 O（新节点）。发生 `replace`（`replaceGeneration` 变化）时整体重建。
- **返回值的契约**：每次返回新数组（调用方持有的数组不会被后来的 append 顶长），但里面的 `Message` 对象是共享且深度冻结的——缓存不需要第二次克隆，消费方也改不了日志（:781-788）。
- 空内容的 `assistant/message`（max-tokens 的 step 只承载 usage）投影为 `null`，不进历史（:806-807 注释）。

回头看第 9 章：`agent.ts:351` 每次请求前调的就是这个方法。**模型历史没有第二份内存副本，每次都是从日志现投影的**——这就是事件溯源在这套系统里的字面含义。

## SessionEventMap：用声明合并扩展词汇

所有事件类型集中在 `packages/core/session/src/types.ts:261` 的 `SessionEventMap` 接口：`turn/start`、`turn/end`、`step/start`、`step/end`、`user/message`、`assistant/chunk`、`assistant/message`、`tool/call`、`tool/result`、`request/header`、`request/context`、`session/end-seed` 等。

这个接口故意设计成**可被声明合并（declaration merging）扩展**——插件在自己的包里往同一个接口里 merge 自己的事件。两个真实例子：

- compaction：`packages/compaction/compaction/src/types.ts:24` merge 了 `compaction/start`、`compaction/summary` 等事件（log-only，不进 surface；真正的历史替换由紧随其后的 `user/message` 完成）。
- llm-retry：`packages/llm/llm-retry/src/types.ts:9-11` merge 了 `llm/retry` 和 `llm/retry-started`，把一次重试的调度与开始变成持久记录。

合并后 `append('compaction/start', ...)` 就是全类型安全的：payload 类型、surface 规则都由编译器强制。新增模型可见输入的正确姿势由此确定：**先扩展 `SessionEventMap`，再从日志渲染**——这正是根 `AGENTS.md` 里「model-visible ⟺ logged」规则的落地机制。

## required-on-read 与 ignorable：读者协议

`SessionEvent` 的信封上有一个可选标记 `ignorable?: true`（`types.ts:444-454`），它定义了读者协议：

- **默认必填（required-on-read）**：读者遇到不认识且没有 `ignorable` 标记的事件类型，必须拒绝重建这个会话，而不是悄悄丢掉它——因为不认识的必填事件可能改变对日志其余部分的解释。
- 写入方只能给「纯信息性记录」标 `ignorable: true`——丢掉它不影响重建。默认必填意味着忘记标 marker 的后果是「过度拒绝」（不方便），而不是「悄悄恢复出一个被掏空的会话」。

配合 `SESSION_FORMAT_VERSION = 0`（`types.ts:87`）的预发布语义：格式版本在第一个带 tag 的发布前保持 0，不做兼容承诺；只有结构性格式变化才 bump 它。词汇增长（新事件类型）由 `ignorable` 协议覆盖，不 bump 版本。机制细节见 `.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md`。

## 持久化不在 core

`SessionStore`（`packages/core/session/src/index.ts:857`）是内存 store（`ctx.sessions`），它的类注释把边界写得很直白（:851-856）：

> Persistence is intentionally not implemented here — persistence plugins subscribe to `session/event` and flush on `session/flush` / dispose.

持久化是 `packages/session/` 组的插件（如 `dsh-session-persistence-jsonl`）：订阅 `session/event` 消防带，把事件缓冲起来，在 `session/flush`（:1075-1090，store 代发的耐久检查点）或 dispose 时落盘。core 不认识文件、不认识磁盘——**session 包只保证事件流自洽，存储是接缝另一头的事**。这就是第 8 章说的 capability event 域的典型用法。

另外两个值得知道的派生能力：

- **fork**：`ctx.sessions.fork(source, boundary?, childSessionId?)` 用日志前缀做种子造出子会话，分叉会校验边界不落在打开的 turn 中间（错误码见 `index.ts:835-840` 的 `SessionForkErrorCode`）。
- **end-seed 标记**：种子（重放/fork/resume 进来的历史）与本次进程产出的 live 事件之间，由 `session/end-seed` 事件分隔（`types.ts:340-364`）；进程内的精确边界是 `Session.firstLiveSeq`（`index.ts:475`）。

## SessionStore：创建、挂载与宣布

`SessionStore`（`ctx.sessions`）的创建路径分两步，拆开的原因很讲究：

- `prepare(id?, options?)`（:928）——只做校验与构造：id 冲突抛错、`meta.cwd` 必须是绝对路径（存储后端以它建目录）、种子走 `Session.fromRestore` 或快照校验。返回的 session **尚未进 store**。
- `create(id?, options?)`（:895）——`prepare` 之后在一个 effect 里 `enter`（挂进 store、接好发布钩子）再 `announce`（发 `session/created`）。

为什么拆？JSDoc（:907-924）写得很清楚：agent 工厂需要把 session 生命周期折叠进 agent 自己的那一个 effect 里，fiber 卸载时 session 和 agent 按**有序**链条拆除；如果 create/announce 是并列的兄弟 effect，拆除竞态会让发布钩子在驱动的收尾事件提交之前就被摘掉，丢事件。这是「注册即 effect」原则在生命周期编排上的直接推论。

构造时的种子校验和 live append 执行**同一套不变量**（构造函数注释，:535-564）：种子里每个事件的数据必须无损 JSON 可序列化、seq 必须从 0 连续——否则一个坏种子会潜伏到后端拒收或内存日志与磁盘分叉时才暴露。种子不是绕过校验的后门，而是校验的另一个入口。

## 不变量守卫：把账本纪律变成断言

`packages/core/session/src/invariant.ts` 是一个 companion 插件（`session-invariant`，挂在 `dsh-invariants` 服务旁）。它为每个 session 维护一份 trace（`SessionTrace`，`invariant.ts:23-31`：最近 seq、打开的 turn/step、下一个 turn/step 编号、未配对的 `pendingCalls`），对每个入库事件做关系校验：turn/step 必须配对开关、编号必须递增、`tool/call` 必须有 `tool/result` 配对。模型可见性不变量（「到达模型请求的东西必须能从日志重建」）也由这类守卫在运行中断言——它不是测试里的期望，而是产品代码里的活体检查。读这个文件是理解「账本纪律」最快的路径：每条断言对应一条你可以依赖的假设。

## 动手练习

**练习 1：把事件流打到控制台。** 写一个插件订阅 `session/event`，打印每个事件的 `seq` 和 `type`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'session-tap'
export function apply(ctx: Context) {
  ctx.on('session/event', (session, event) => {
    console.error(`[session ${session.id}] #${event.seq} ${event.type}`)
  })
}
```

挂到 headless 组合上跑一次任务，观察输出顺序与第 9 章练习 1 的 JSONL 日志是否一致（它们是同一条流：一个在内存里同步广播，一个被持久化插件落盘）。

**练习 2：merge 一个自己的事件。** 仿照 `llm-retry` 的做法，在你的插件里写：

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    'demo/marker': { note: string }
  }
}
```

然后在 `agent/pre-step` 里 `session.append('demo/marker', { note: '...' })`（注意拿 session 的正确姿势是 `agent.session`）。跑一次并在 JSONL 里找到它。再想想：这个事件该标 `ignorable: true` 吗？判据是「一个不认识它的读者丢掉它之后，重建出的会话还完整吗」。

**练习 3：验证 required-on-read。** 手工编辑一份会话 JSONL，把某个事件的 `type` 改成不存在的值（不加 `ignorable`），然后尝试 resume 这个会话，观察加载方如何拒绝。再把 `ignorable: true` 加到信封上重复实验，对比行为差异。

## 延伸阅读

- `docs/subsystems/session.md` — session 子系统的完整参考。
- `packages/core/session/src/surface.ts` — SurfaceManager 与投影规则的实现。
- `.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md` — 版本机制与 required-on-read 的决策记录。
- 上一章 [第 9 章 Agent Loop](./02-agent-loop-source.md)；下一章 [第 11 章 能力接缝](./04-capability-seam.md)。
