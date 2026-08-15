# 第 8 章 架构总览：事件溯源与扩展点

## 本章目标

- 建立全仓库的心智地图：组合层级、核心包分工、三类事件域。
- 理解一个 turn 从输入到落盘的完整事件流水线。
- 掌握「model-visible ⟺ logged」这条最重要的不变量。
- 学会用扩展点地图回答「新功能该挂在哪里」。

本章以 `docs/architecture.md` 为主线，它是仓库钦定的架构文档——改动 `packages/` 之前官方要求先读它。本章是它的导读与注解，而不是替代品。

## 一切皆插件，没有特权核心

`docs/architecture.md:11` 开宗明义：Cordis 是 dsh 底下的框架，插件向共享 context 贡献服务、类型化事件和可逆的 effect。模型适配器、工具注册表、会话日志、Agent 循环本身都是插件，因此每一部分都可以从配置里替换。

这句话的实践含义在 `docs/architecture.md:13`：**没有可以打补丁的特权核心**。你扩展 dsh 的方式是把一个插件挂到其它插件旁边；所有注册都是 effect，插件卸载时随 effect 一起回收。入门篇第 6 章你写过第一个插件，已经见过 `ctx.effect()` / `ctx.on()` 的用法；进阶篇要理解的是：这套机制支撑了整棵产品树。

## 组合层级：profile、bundle、patch

运行中的 `dsh` 是一棵在启动时按有序层级组合出来的插件树（`docs/architecture.md:15-37`）：

- **profile**：存在 Harness home 里的命名组合。它列出叠放的 bundle、安装的树外插件，以及用户自己的 `cordis.patch.yml`。`web` 和 `headless` 是随仓库发布的模板。
- **bundle**：Cordis 配置行和所挂载代码的分发格式，保证它插入的每一行仍可被上层 patch 修改。`dsh-base` 是每个 profile 的第一层（模型适配器、工具、持久化、沙箱与审批策略、设置、凭证、遥测），`dsh-web-app` 加浏览器应用，`dsh-headless` 加一个无 server 的一次性运行器。
- **patch**：层级按序作用于空入口列表——profile 里列出的每个 bundle → profile 的 `cordis.patch.yml` → home 级 patch → `--patch` 覆盖。patch 按 id 定位一行并整体替换其 config，或插入新行。

想看你机器上实际启动的那棵树：

```sh
dsh --profile web --dump-config
```

它打印的每一行都可以被你自己的 patch 替换。入门篇第 7 章的组合与覆盖练习就是在这条规则上玩的。

## 核心包：各自的 `ctx` 钥匙

`docs/architecture.md:39-51` 的表列出了核心包的分工，记住这张表就记住了系统的骨架：

| 包 | 拥有 | `ctx` 键 |
|---|---|---|
| `core/session` | 只追加的 `SessionEvent` 日志与内存 store | `ctx.sessions` |
| `core/system-prompt` | 提示词节与工具 schema 装配 | `ctx.systemPrompt` |
| `core/tools` | 带作用域的工具注册表与受防护的执行管线 | `ctx.tools` |
| `core/agent` | `Agent` 接口、活体注册表、`agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动 | `ctx.agentLoop` |
| `llm/llm` | 消息与流词汇、适配器接缝 | `ctx.llm` |

注意每个包拥有的东西都不大：session 只管日志，tools 只管注册与执行，agent-loop 只管驱动。**复杂度在组合里，不在任何单个包里**。第 9、10、12 章会分别精读 agent-loop、session、tools。

## 三类事件域：扩展点的第一决策

`docs/architecture.md:53-61` 把事件分成三个域，并明说「选对事件域是大多数改动的第一个决策」：

- **Session events**：追加到日志、经 `session/event` 广播的持久事实。当这个事实必须在 reload 后存活时用它。
- **Agent events**（`agent/*`）：携带活体 `Agent` 的事件——inbox、step、status、request、validation、continuation。用来观察或拦截进行中的工作。
- **Capability events**：把策略和适配器挂到能力接缝上（`fs/*`、`tools/*`、`telemetry/*`），不需要 import loop。

三者的判别标准很干脆：要不要持久化？要，就是 session event；不要但要在运行时拦截 Agent 行为，就是 agent event；只是给某个能力缝加策略，就是 capability event。

## step / turn / round：先统一词汇

读事件流水线前，先把层级词汇定死（`docs/glossary.md` 的 loop hierarchy 条目）：

- **step**：一次模型请求，加上它的响应引起的工具执行。
- **turn**：一次会话中被接纳输入的完整排空（drain），以模型和工具停止或终态策略介入为结束。一个 turn 含零个或多个 step。
- **round**：外层策略迭代，包含一个 turn——例如一个 goal round。round 计数器属于那个策略，不对会话里每个 turn 计数。

「零个 step 的 turn」不是修辞：被 `agent/pre-step` 拒绝的输入照样开关一个 turn，只是不消耗模型调用。下一章你会在源码里看到这个分支。

## 一个 turn 的完整流水线

`docs/architecture.md:63-90` 给出了权威的事件序列。把它画成图：

```text
turn/start                              ← 持久事件：turn 打开
  ├─ 认领 next-step 输入 + 一条排队消息
  ├─ 装配提示词节 + 工具 schema
  ├─ agent/pre-step (waterfall)         ← 活体扩展点：改写或拒绝
  │    ├─ reject → turn/end { kind: 'blocked' }，不消耗 step
  │    └─ 首轮 enter 被改写为空 → turn/end { kind: 'completed' }
  ├─ step/start                         ← 持久事件
  ├─ user/message (surfaceOp: append)   ← 持久事件：进入模型历史
  ├─ 从日志 derive 模型历史
  ├─ agent/request (waterfall) → llm/stream (waterfall)
  │    └─ assistant/chunk*              ← 持久事件：token 级回放保真
  ├─ assistant/message                  ← 持久事件：装配后的助手消息
  ├─ tool/call* → tools/pre-execute → tools/execute
  │    → tools/post-execute → tool/result*   ← call/result 均持久
  ├─ step/end                           ← 持久事件
  └─ 工具欠一次新请求，或 next-step 输入到达 → 认领 → 下一个 step
agent/turn-stopping (serial)            ← 活体扩展点：可阻止收尾
turn/end { reason }                     ← 持久事件：结构化结束原因
```

三个要点：

1. **持久事件与活体扩展点交错出现**。`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*` 是持久 session 事件；`agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 是活体扩展点（`docs/architecture.md:84`）。
2. **waterfall 必须调 `next()`**。`agent/pre-step`、`agent/request`、`llm/stream` 和三个 `tools/*` 事件都是 waterfall，监听者不调用 `next()` 就会短路整条链（语义见 `docs/cordis-primer.md` 的 waterfall 章节）。`agent/turn-stopping` 是 serial，没有 `next()`。
3. **输入走同一个 inbox**。有些消息立即唤醒驱动（`followup`、`steer`），注入的上下文（`inject`）躺在 inbox 里等下一条唤醒消息（`docs/architecture.md:86`）。

`agent/pre-step` 决定模型看到什么：监听者可以改写被认领的消息，也可以 outright 拒绝；被拒绝或首轮被清空的认领仍然会关闭一个没消耗 step 的持久 turn——日志要记录这次尝试（`docs/architecture.md:88`）。

## 会话日志与「model-visible ⟺ logged」

`docs/architecture.md:92-96` 是全书最重要的一段：

> 会话日志是模型所见上下文的来源。`deriveMessages()` 从日志投影模型历史，原始 `assistant/chunk` 事件保留回放与 UI 保真。fork、resume、transcript、遥测、持久化全都从这条流派生。
>
> **Model-visible means logged.** 任何到达模型请求的东西都必须能从日志重建，并有运行时不变量断言它。这就是为什么新的模型可见输入需要一个新的 session 事件：扩展 `SessionEventMap`，从日志渲染。

这条不变量是整个事件溯源设计的「为什么」：模型上下文不是内存里某个易失数组，而是日志的**投影**。投影意味着任何来源（重启、fork、另一个进程）只要拿到同一份日志就能重建完全相同的模型输入。反过来，如果某个输入偷偷绕过日志直接进了请求，重放就会撒谎——所以这是被断言的不变量，不是风格建议。

第 10 章专门讲 session 包怎么实现这条不变量。

## 扩展点地图：新行为挂哪里

`docs/architecture.md:104-129` 的表是日常开发最有用的一页，这里摘录最常用的几行（完整表共 19 行，建议打开原文对照）：

| 目标 | 机制 |
|---|---|
| 加模型 provider | 在 `ctx.llm` 上注册适配器（第 13 章） |
| 加模型可见能力 | 注册到 `ctx.tools`，其 schema 进入提示词装配（第 12 章） |
| 给单个会话换能力集 | 组合一个 agent preset |
| 拦截请求、工具或 turn | 用对应的 `agent/*` 或 `tools/*` 事件；`agent/turn-stopping` 可停 turn |
| 加模型可见上下文 | 调 `agent.inject()`，在下一次被接纳的请求里落地 |
| 加 UI/编辑器集成 | 驱动 `ctx.agents`，从 `session/event` 渲染 |
| 加持久会话状态 | 扩展 `SessionEventMap`，从日志渲染与重放（第 10 章） |
| 把注册限定到单个 agent | 用该 agent 的 `agent.ctx`（第 14 章） |
| fork 一个活会话 | `ctx.sessions.fork(source, boundary?, childSessionId?)` |

表的标题句同样重要：「新行为挂在文档化的扩展点上。改动 loop 本身就要更新这张地图。」也就是说，**表里没有的需求才轮到改 loop**——而且改 loop 是架构级变更，要同步更新 `docs/architecture.md`。

## 能力接缝：一次替换，全局生效

`docs/architecture.md:98-102` 定义了贯穿全仓库的设计模式——**capability seam（能力接缝）**：一个可替换能力由三个角色组成：声明接口的 Service Definition、实现它的 Service Provider、使用它的 Consumer（通常是一个模型可见工具）。一个包可以身兼多角色，但只有一个角色不构成接缝。

接缝的威力在于「一次 provider 替换改变整个产品」：文件系统和子进程 provider 共享同一个执行世界，把它们指向远程沙箱，Bash、PTY、LSP 会一起跟着走，不需要 fork 任何 provider。第 11 章用 shell 和 llm 两条真实接缝把这个模式讲透。

## 动手练习

**练习 1：画出你机器上的插件树。** 跑 `pnpm dsh --profile headless --dump-config`（或 `web`），数出配置行数，并按 `docs/architecture.md:39-51` 的核心包表给其中 5 行归类：这行挂的是哪个 `ctx` 键？再任选一行，写出你会用哪个 patch 文件、以哪个 id 覆盖它。

**练习 2：给事件域分类。** 打开 `docs/event-producer-consumer.md`（每个事件的 producer/consumer 清单），任选 6 个事件，把它们分进本章的三类事件域，并写一句话说明归类依据（持久？拦截？策略？）。其中至少一个应该是你之前没听过的。

**练习 3：用不变量找 bug。** 假设有人提了一个 PR：在 `agent/request` waterfall 里直接向 `options.messages` 追加一条从未落盘的 synthetic 消息。根据「model-visible ⟺ logged」，这个 PR 违反了什么？运行时哪个机制会抓到它？把你的推理写下来，然后在第 9、10 章读完源码后回来对答案。

## 延伸阅读

- `docs/architecture.md` — 本章的主线原文，改 `packages/` 前必读。
- `docs/glossary.md` — 全仓库词汇表，capability-seam、agent-scope、loop hierarchy 三个条目与本章直接相关。
- `docs/event-producer-consumer.md` — 每个事件的 producer/consumer 清单。
- `docs/agent-lifecycle.md` — turn 生命周期的时序图。
- 下一章 [第 9 章 Agent Loop 源码精读](./02-agent-loop-source.md) 把本章的流水线落到每一行代码上。
