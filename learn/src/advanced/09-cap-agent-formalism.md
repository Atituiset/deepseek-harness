# 第 16 章 一个 Agent 的完整形式：六件套与状态机

## 本章目标

- 把全书读过的机制压缩成一个可检验的模型：Agent = Prompt + Loop + Tools + Context + Session + Model。
- 用状态机语言精化这个模型，并把预算闸、取消、重试三个介入点钉到源码的确切位置。
- 逐条验证四条不变量（I1–I4）：它们各自由哪些代码强制执行、违反时会在哪里爆炸。
- 学会用这套模型评估你自己的 agent 设计——六件套哪里缺了、不变量哪条没有执行者。

前面 15 章是自底向上读源码；本章自顶向下把结果收拢。读完它，你应该能在白板上画出这个系统，并对每一笔说「这行代码在哪」。

## 六件套：各自拥有什么

先按职责把六件套的**所有权边界**画清楚。一个常见的失败设计是让其中两件共用状态——这里每一件都有独占的领地：

| 件 | 唯一拥有的状态 | 仓库里的家 | 不许别人碰的东西 |
|---|---|---|---|
| **Prompt** | 提示词节排序、变量、工具 schema 的**装配视图** | `packages/core/system-prompt`（`ctx.systemPrompt`） | 装配结果是一次性快照（`PromptAssembly`），用后即弃 |
| **Context** | token 压力测量、压缩区间规划 | `packages/compaction/*` + `packages/llm/token-meter`（`ctx.tokenMeter`） | 压力是**测量值**，不持有历史本身 |
| **Session** | 追加型事件日志、表面（surface）、派生缓存 | `packages/core/session`（`ctx.sessions`） | 历史只在 append 边界变化；投影只读 |
| **Model** | 消息/流词汇、适配器注册表、请求绑定 | `packages/llm/llm`（`ctx.llm`） | 请求一经绑定（`prepareCall`）即冻结 |
| **Tools** | 注册表、五级执行管线、守卫 | `packages/core/tools`（`ctx.tools`） | 执行的时序编排归 Loop，不归注册表 |
| **Loop** | 相位（phase）、inbox、AbortController 链 | `packages/core/agent-loop`（`ctx.agentLoop`） | **Loop 自己不拥有任何六件套的状态**，只持有引用并编排它们 |

最后一行是精髓：`ReactLoopAgent` 的字段只有 `inbox`、`phase`、`activityDone`、缓存纪元号和 dispatch（`agent.ts:71-86`）——没有历史、没有工具、没有提示词。**Loop 是编排者，不是容器**。你写自己的 agent 时先问：我的 Loop 类里有没有本该属于别人的字段？

## 状态机

把六件套压进一个状态元组（State tuple）：

```text
State = (Assembly, Session, Inbox, Phase)
Loop : State → Model → (ToolCalls | Text) → Tools → State′
        │
        ├─ 预算闸（token 预算 + 并行预算）
        ├─ 取消（AbortController 链）
        └─ 重试（agent/request-error → llm-retry）
        （三者都在每一步可介入）
```

对应到源码，这个元组在**两个边界**上被读取：

1. **preStep**（`agent.ts:234-252`）：认领 inbox → `systemPrompt.assemble` 装配快照 → `runtimeContext.project` 投影运行时上下文。这一步产出 `PromptAssembly`（提示词节 + 工具 schema + 变量，`system-prompt/src/index.ts:114-119`）——**Prompt 与 ToolSpecs 在同一个快照对象里原子诞生**。
2. **step**（`agent.ts:341-438`）：`deriveMessages()`（Session 投影出 Context）→ `buildRequest`（组装请求 + 绑定适配器）→ 流式消费 → `executeToolCalls`（工具执行）→ 回到 step 边界。

每转一圈，五件都被**重新读取**而不是**携带缓存**——除了一个例外：`deriveMessages` 的增量缓存（按表面世代缓存，`index.ts:790-811`），而它恰好在 `replace`（压缩）发生时自动失效。**状态机没有隐藏状态**：任何一处的最新事实要么在元组四项里，要么在日志里。

图示化（对上你的示意图，标注源码位置）：

```text
用户输入 ──► Loop（kick/turn/step，见第 9 章）
              ├─► Model（buildRequest → prepareCall → stream，见第 13 章）
              ├─► Tools（executeToolCalls 调度器，见第 12 章）
              ├─► Context（tokenMeter 压力 + compaction 投影，见第 10 章）
              ├─► Session（append 唯一写路径，见第 10 章）
              └─► Prompt（preStep 装配，见第 9 章）
                    ▲
                    └─ 观测（agent/* 活体事件 + session/event 持久事件，见第 8 章）
```

## 四条不变量的执行者

不变量不是文档承诺，而是**有执行者的代码事实**。逐条找出谁在执行它、违反时在哪爆炸：

### I1：`Session.append` 为唯一写路径（可重放）

**执行者**：`Session` 类自己。`log` 是 `private`（`index.ts:426`），全仓库没有任何其它代码直接写数组——持久化插件（`dsh-session-persistence-jsonl`）订阅 `session/event` 落盘，fork/恢复走 `Session.fromRestore` 种子校验，UI 走投影。

**违反时在哪爆炸**：种子校验在构造期拒绝（`index.ts:535-564` 的构造注释）；`session-invariant` companion（`invariant.ts`）在运行中对每个入库事件做关系断言（turn/step 配对、`callId` 配对、seq 连续）——**不是测试里的期望，是产品代码里的活体检查**。

**为什么重要**：重放（快照测试、resume、SDK 投影）完全建立在「日志 ⇒ 重建一切」上。第 15 章的免 key 快照测试，本质是 I1 的最大规模日常验证。

### I2：Prompt 与 ToolSpecs 同快照（原子性）

**执行者**：`PromptAssembly` 的类型形状（`system-prompt/src/index.ts:114-119`）——`sections`、`contexts`、`tools`、`variables` 住在同一个冻结对象里。`headerEquals`（`session/src/request-header.ts:44-53`）比较时同时覆盖 config、system、**逐条** tool schema（`sameSchema`）。

**违反时在哪爆炸**：`request/header` 落盘在 `buildRequest`（`agent.ts:503-518`）——提示词变了或任何一个 tool schema 变了，`headerEquals` 判不等，新 header 必然落盘。重建方从日志折叠 `EpochHeader`（`foldRequestHeader`，`request-header.ts:59`）恢复的就是这个原子对。

**为什么重要**：没有它，模型可能用新人格配旧工具清单（或反之），而日志无法证明它看到了什么。第 8 章「model-visible ⟺ logged」在 (Prompt, Tools) 这个序偶上的完整落实就是 I2。

### I3：Context = project(Session)，投影非重写

**执行者**：`deriveMessages` 只读 surface（`index.ts:790-811`）；投影规则集中在纯函数 `deriveEventMessage`（`surface.ts:90-120`，第 10 章深读过）；「重写」历史的唯一合法手段是 append 一条带 `replace` 的 surfaceOp（压缩），**replace 不删除任何事件**，只改变投影视图。运行时上下文同样走投影：`RuntimeContextProjection.project`（`runtime-context.ts`）从装配的节里投影，不反写装配。

**违反时在哪爆炸**：`snapshotEvents`/`deriveMessages` 返回的 Message 是深冻结的共享对象——试图改历史在任何消费方当场 TypeError。

**为什么重要**：Context 若是独立存储，就需要双向同步协议（经典 bug 源）；投影模型里「Context 过时」在结构上不可能发生——它每次从 Session 现算，缓存有世代号护体。

### I4：每次 Model 调用受 token 预算约束

这条的执行结构最分散，值得拼成一张完整的地图。**预算约束不是一个闸门，而是三层防线**：

**第一层：请求前的压力测量（主动闸）。** `tokenMeter.measure(session, requestHeader?)`（`token-meter/src/index.ts:120-140`）在每次 step 边界现算当前表面压力——路由相关的逐节点计价（image pricing）、provider usage 锚定（上次成功调用的 usage 与启发式估计取更保守的基线）。`compaction-basic` 在 `agent/pre-step` 里消费它（`index.ts:148-165`）：`thresholdRatio`（默认 0.8，`config.ts:20`）超限就**在 step 开始前**压缩，摘要以 `user/message + replace` 落盘。压力压缩失败**不阻断 turn**（日志 warn 后 `next()` 继续）——预算闸失效宁可多花 token，不能死循环。

**第二层：请求内的输出预算（被动闸）。** `maxTokens` 进请求头（`llm-deepseek` 默认 `DEFAULT_MAX_TOKENS = 256_000`，`adapter.ts:142`）；撞墙时流以 `finish.kind === 'max-tokens'` 结束，`step` 返回 `{ kind: 'max-tokens' }`，turn 结局**粘性**（第 9 章深读二）。

**第三层：溢出后的恢复（兜底闸）。** provider 真的返回 `CONTEXT_WINDOW_EXCEEDED` 时，`compaction-basic` 的 `agent/request-error` 监听器（`index.ts:180-226`）接管：压缩到 `replaceGeneration` 前进，然后返回 `{ kind: 'retry' }` 让 Loop 重发；重试次数有上限（`maxOverflowRetries`），到顶把原始错误放行。注意一个精致的处理：压缩第二阶段抛错但表面**已经**前进了一代（模型无关的 prune 先落盘），仍然算恢复证据、仍然 retry（:207-214 的注释讲了这个边界）。

三层合起来才是 I4 的完整含义：**预算约束 = 测量（主动）+ 上限（被动）+ 溢出恢复（兜底）**，缺一层就有可构造的死路（比如超限后无恢复 = 会话报废；只有恢复无上限 = 压缩循环烧钱）。

## 三个介入点的解剖

你图上Loop 箭头旁的三个标注——预算闸、取消、重试——在源码里有完全不同的介入机制，这个对比本身就是设计课：

| 介入点 | 机制 | 介入位置 | 语义 |
|---|---|---|---|
| **取消** | `AbortController` 链 + `signal.throwIfAborted()` 派遣点 | step 循环每圈头部、每个 chunk 落盘后、pre-execute 等待点 | **同步传播**：signal 是每步必查的派遣点，取消在毫秒级生效 |
| **重试** | `agent/request-error` waterfall | 仅流以 `error/aborted` finish 结束时 | **策略化**：默认放行，`llm-retry` 按策略接管；「谁来决定」是可插拔的 |
| **预算闸** | 三个不同位置（见 I4） | pre-step / 请求内 / request-error | **分层的**：不是一个组件，是一组防线 |

注意**不对称性**：取消是基础设施（Loop 内建，没有插件能绕过），重试是策略（不挂 llm-retry 插件就没有重试），预算是混合体（测量在服务、阈值在配置、恢复在插件）。**哪个介入点该是基础设施、哪个该是可插拔策略**，是你设计自己的 agent 时最重要的分层决策之一——这个仓库的答案：可靠性内建（取消）、演进性外挂（重试）、成本控制混合（预算）。

## 用这个模型审计你自己的 Agent 设计

拿这六件套加四条不变量当清单，逐项过你的设计：

1. **六件套完整性**：你的 Loop 类持有它不该持有的状态吗？（常见病：Loop 里塞了个 messages 数组——那是 Session 的职责，I3 直接被破坏。）
2. **I1**：你有第二条写历史的路径吗？（常见病：UI 直接改内存会话对象「顺便」保存。）
3. **I2**：提示词和工具清单同一次落盘吗？还是两次写、中间能观测到不一致状态？（常见病：system prompt 单独存一份，工具清单另一份。）
4. **I3**：上下文是投影还是副本？（常见病：维护一个 context 列表并手动同步——两份真相。）
5. **I4**：三层预算闸都有执行者吗？（常见病：只有 maxTokens，没有 pre-request 测量也没有溢出恢复——超限即死。）
6. **三个介入点**：取消内建了吗？重试有策略所有者吗？取消和重试的语义边界写清了吗（取消不重试、重试不吞取消）？

这不是理论洁癖。六件套缺一件、四不变量缺一个执行者，都对应着一类**已经在这个仓库的设计里被显式解决过的生产事故形态**：丢上下文（I1）、人格与工具错配（I2）、会话漂移（I3）、超限死循环（I4）。

## 与全书的对照

- 六件套的所有权重 → 第 8 章扩展点地图、第 11 章能力接缝
- Loop 的并发协议与失败分类 → 第 9 章三节深读
- Session 的账本与投影 → 第 10 章
- Tools 管线与调度不变量 → 第 12 章
- Model 的接缝与预算词汇 → 第 13 章
- Context 的压缩机制（本章的展开）→ 第 10 章 replace 语义的实战消费者
- 状态机的可重放性 → 第 15 章快照测试（I1 的最高强度日常验证）

## 动手练习

**练习 1：审计一个真实配置。** 打开 `packages/bundle/base/cordis.patch.yml`，把每一行归到六件套之一（或「编排 glue」）。数一数：哪件套的行最多？哪件最少？你会发现 Tools 占了压倒性多数——这是生产 agent 的普遍形态，工具生态是复杂度的主要归宿。

**练习 2：构造 I4 的死路。** 写一个 overlay：禁用 `compaction-basic`，把 agent-loop 的 `maxParallelToolCalls` 保持默认，用很小的模型上下文配置（或 mock 一个极小 `contextWindow` 的 adapter）跑一个长任务。观察 `CONTEXT_WINDOW_EXCEEDED` 后的行为——没有恢复闸时错误如何终结 turn？再恢复 compaction，对比 `agent/request-error` 日志里的 retry 链。

**练习 3：证明 I2 的原子性。** 写一个 `agent/request` waterfall 监听器，记录每次收到的 config。然后在另一个插件里注册一个新工具。跑一个任务，在会话日志里找到那次 `request/header`（`reason: 'change'`）事件，验证：system 与**完整的** tools 数组在同一条事件里——不存在只有工具变了而 system 没变（或反之）的中间状态。

**练习 4：给状态机画时序图。** 用 `session-tap` 插件（第 10 章练习 1）把事件流打到 stderr，跑一个含工具调用的任务，对照本章的状态机把每个事件归到「读元组哪一项 → 写元组哪一项」。特别标注三个介入点各自出现的位置（如果预算闸没触发，想想为什么——压力没到 0.8）。

## 延伸阅读

- `docs/architecture.md` — 官方架构文档，本章是它的形式化视角。
- `.agents/notes/implemented/architecture/2026-06-18-compaction-capability-seam.md` — 压缩接缝的决策记录（I4 第一层的设计理由）。
- `packages/llm/token-meter/README.md` — 压力测量服务的完整契约。
- `packages/compaction/compaction-basic/README.md` — 三层预算闸的实战实现。
- 上一章 [第 15 章 测试体系](./08-testing.md)；读完本章，回到 [第 8 章](./01-architecture.md) 用新框架复习全图。
