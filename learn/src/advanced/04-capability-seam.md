# 第 11 章 能力接缝：Definition / Provider / Consumer

## 本章目标

- 掌握全仓库最重要的结构模式：capability seam（能力接缝）的三角色。
- 用 shell 和 llm 两条真实接缝看清每个角色的代码长什么样。
- 理解「explicit > implicit」原则在接缝边界上的落地：request/spec 拆分。
- 能自己判断一个新能力该不该做成接缝、拆几个包。

## 定义：三角色缺一不可

`docs/glossary.md` 的 capability-seam 条目（`docs/glossary.md:9`）是权威定义，值得逐句拆：

- **Service Definition**：一个 Cordis `Service` 子类，占有自己的 `ctx.<key>`，定义词汇类型。它是**抽象类**（如 `ShellExecutor`）**或具体注册表**（如 `WebRuntime`），**绝不是一个裸 TypeScript `interface`**。
- **Service Provider**：实现 Definition 的插件，一个接缝可以有多个。
- **Consumer**：注入这个服务使用它的插件，通常是模型可见工具。

三条纪律：

1. **完整接缝缺一不可**。只有一个角色不构成接缝；加一个能力意味着设计全部三个角色。
2. **拆分时机是角色独立演化**。角色通常各占一个包；一个关注点内可以同时拥有多角色（`dsh-llm` 同时拥有它的 Service Definition 和 Consumer）。
3. **为什么不用裸 interface**：接口没有运行时可占有的 `ctx` 键、没有 effect 生命周期、没有 Cordis 的注入与追踪。`Service` 子类让「谁实现了这个能力」成为插件树上的可观察事实。

## 案例一：shell 接缝

shell 是仓库官方认定的 canonical 例子（glossary 原文点名）。全仓库的接缝清单见生成文档 `docs/capability-seams.md`——`fs`、`web`、`skill`、`compaction`、`code-runtime` 等都遵循同一形状。shell 的三个角色：

- **Definition**：`packages/shell/shell`（包名 `dsh-shell`）。`ShellExecutor` 定义在 `packages/shell/shell/src/index.ts:64`：
  ```ts
  export abstract class ShellExecutor extends Service {
    abstract resolve(request: ShellExecRequest): ShellExecSpec   // :84
    abstract run(spec: ShellExecSpec): Promise<ShellRunResult>   // :92
    abstract start(spec: ShellExecSpec): ShellProcess            // :99
  }
  ```
- **Providers**：`packages/shell/bash-local`（本机 bash）、`packages/shell/bash-sandbox`（沙箱内 bash）；同组还有 pwsh 的两个对应物。
- **Consumer**：`packages/shell/tool-bash`（`dsh-tool-bash`）——模型可见的 `bash` 工具，注入 `ctx.shell` 执行命令。

注意 Consumer 不认识 local 还是 sandbox：它只面向 `ShellExecutor` 这个抽象。把 provider 从 `dsh-bash-local` 换成 `dsh-bash-sandbox`，`bash` 工具一行不改就跑进了沙箱。这就是第 8 章说的「一次 provider 替换改变整个产品」。

Consumer 侧的接线就是一行 inject 声明（`packages/shell/tool-bash/src/index.ts:30`）：

```ts
export const inject = ['tools', 'shell', 'systemPrompt', 'shellEnv']
```

`inject` 声明了它消费哪些服务键：Cordis 保证这些服务就绪后才激活这个插件，插件卸载时它的工具注册（一个 effect）自动回收。Definition 同时**拥有词汇类型**：`ShellExecRequest`、`ShellExecSpec`、`ShellProcess`、`ShellRunResult` 都定义在 `dsh-shell` 的 `types.ts`，provider 和 consumer 都从 Definition 包 import 它们——词汇只有一个所有者，这是三方类型始终对齐的原因。

### explicit > implicit：request/spec 拆分

`ShellExecutor` 的 API 形状是这条仓库原则的模板（根 `AGENTS.md`：「Explicit > implicit at package boundaries」）。执行被拆成两步：

1. `resolve(request: ShellExecRequest): ShellExecSpec`（:84）——把「用户想干什么」（命令、cwd、环境等）显式解析成「确切要执行什么」（补全后的可执行路径、最终 argv、生效环境）。
2. `run(spec)` / `start(spec)`（:92、:99）——只接受已解析的 spec，里面**不允许**再藏着 `?? default` 之类的隐式补默认。

默认值的填充被强制摆在一个显式的、可审查的步骤里，而不是散落在 `run()` 的深处。你写自己的接缝时照抄这个形状：请求类型进 `resolve`，spec 类型进执行方法。

## 案例二：llm 接缝

llm 接缝展示「Definition 与 Consumer 同包」的变体（角色没有独立演化，就住在一起）：

- **Definition**：`packages/llm/llm` 的 `LlmRuntime`（`packages/llm/llm/src/index.ts:326`），占有 `ctx.llm`，同时定义消息与流词汇（`Message`、`StreamChunk` 等在 `types.ts`）。适配器基类 `LlmAdapter` 在同文件（抽象方法 `stream` 在 :274）。
- **Providers**：`packages/llm/llm-deepseek`（直连 HTTP + SSE）、`packages/llm/llm-pi-ai`（包装现成 LLM 库）。第 13 章会精读前者。
- **Consumer**：`LlmRuntime` 自己就是主要 consumer（`stream`/`prepareCall`），agent-loop 通过它发起请求；`llm/stream` waterfall（`index.ts:67`）把每次调用暴露给策略插件。

`LlmRuntime.registerAdapter`（:338）的注册纪律直接来自接缝的完整性要求：

- **一条 provider 路由一个 adapter**——重复注册抛错。
- **多路由注册全有或全无**——一个 adapter 注册三条路由，中途失败则整体回滚，不留半个注册。
- **HMR 安全**：注册是 effect，返回 disposer（`AdapterRegistrationHandle`）；插件卸载时 adapter 随 effect 移除，热替换成立。

agent-loop 侧的消费姿势（第 9 章见过）：`ctx.llm.prepareCall(config, signal)`（:890）把一次调用绑定到具体 adapter 注册，拿回带着精确默认值和 `retryPolicy` 的 `PreparedLlmCall`；不用 prepare 的直接 `ctx.llm.stream(request)`（:1051）走注册解析加上 `llm/stream` waterfall。两条路最终都汇到 adapter 的 `stream()`。

这条接缝还有一层值得单独看：`llm/stream` waterfall（`index.ts:67`）把**每一次**流式调用暴露成一个能力事件——重试、录制回放、路由都可以作为不 import loop 的普通插件挂在上面。录制回放（`packages/test-support/llm-replay`）支撑了第 15 章的免 key 快照，就是这条 waterfall 的消费者。接缝设计得好，测试基础设施都只是普通插件。

## 两条接缝对照

把两个案例并排，能看出接缝模式的弹性：

| | shell | llm |
|---|---|---|
| Definition | `ShellExecutor`（抽象类） | `LlmRuntime`（具体注册表）+ `LlmAdapter`（抽象类） |
| 包分布 | 三角色各占一包 | Definition 与主 Consumer 同包，provider 独立 |
| 替换粒度 | 换一个 executor 类 | 按 provider 路由注册 adapter，多路由并存 |
| 策略挂点 | Consumer 与守卫 | `llm/stream` waterfall、`agent/request-error` |

`docs/architecture.md:102` 还点了一个更隐蔽的接缝关系：文件系统和子进程 provider 共享同一个执行世界——把它们指向远程沙箱，Bash、PTY、LSP 会一起跟着走。接缝不是孤立的，执行环境类接缝（fs、subprocess、sandbox）是其它接缝的地基。

## 接缝与包组织

仓库的包目录就是按接缝分组的（`packages/README.md` 有完整分组表）：`packages/shell/` 一个目录装下 shell 接缝的全部角色，`packages/llm/` 装下 llm 接缝，`packages/subagent/`、`packages/workflow/` 同构。这个布局把一个设计问题变成了物理问题：**你想加的东西属于哪条接缝的哪个角色？** 回答不了，多半是该立新接缝；答得出来，包放哪里、依赖谁、README 写哪个角色，全都顺理成章。

命名纪律也来自 glossary：seam 这个词专指完整能力，构成它的部件按角色称呼——「shell 接缝的 provider `dsh-bash-local`」，而不是「`dsh-bash-local` 这条 seam」。词汇精确，讨论才不会滑。

## 什么时候不该立接缝

接缝有成本：三个角色、词汇类型、注册纪律。以下信号说明该克制：

- 只有一个实现且看不到第二个的需求来源——先写具体类，把可替换性留给真正的第二个实现出现时再抽（预发布期 rename 是自由的）。
- 差异无法被抽象吃掉——provider 之间的差异会漏进词汇类型，迫使 Consumer 分支。这时要么重新设计词汇，要么承认它们是两个能力。
- 没有 Consumer——一个没人注入的服务不是能力，是库。纯工具函数放 `packages/util/`，别包装成服务。

## 设计一条新接缝的检查清单

把全仓库接缝的共性抽出来，设计时逐项过：

- **Definition** 是 `Service` 子类，占有 `ctx.<key>`，拥有全部词汇类型（请求、spec、结果、错误码）。裸 interface 出局。
- **注册是 effect**：`register*` 返回 disposer，重名 fail loud，卸载即摘除，HMR 安全。
- **默认显式化**：request/spec 拆分，`resolve(request): Spec` 是显式步骤，`run()` 里没有隐式默认。
- **Config 经 schemastery 校验**，部署相关的可调项都是 cordis.yml 可改的 Config 字段，没有硬编码常量。
- **误配置 fail loud**：缺 provider、缺运行时、缺凭证，在加载期或最早可解析点抛带行动指引的错误，不静默跳过。
- **能力前置**：provider 用 capabilities 广告自己支持什么，服务在 start 之前就能拒绝不支持的请求。
- **Consumer 只面向 Definition**：grep 一下 Consumer 包，不应 import 任何 provider 包。

`dsh-shell`、`dsh-llm`、`dsh-subagent` 都过了这张表；你自己的接缝也应该能过。

## 注册即 effect：provider 的生与死

所有接缝共享同一条生命周期纪律，看一遍 llm 的就够了（`packages/llm/llm/src/index.ts:380-400`）：

- `registerAdapter` 返回的不只是 disposer，而是一个 `AdapterRegistrationHandle`——摘除是显式动作，但插件卸载时 effect 兜底自动摘除。
- 重复路由在注册点抛错；多路由注册中途失败整体回滚，系统里不存在「半个 adapter」。
- provider 包自己也是普通插件：`apply(ctx, config)` 里做注册，没有特权入口。

推论：provider 可以热替换。卸载旧 provider 的 fiber，注册 effect 回收；挂载新 provider，下一次 `prepareCall`/`stream` 解析到新注册。正在进行的调用持有旧的 `PreparedLlmCall` 不受影响——绑定发生在调用前，不在调用中。

## 怎么认出一条接缝

在仓库里辨认接缝有一个快速方法：找「`ctx.<key>` + 多个 provider 包 + 至少一个 tool 包」的三件套。包组目录通常就是按接缝组织的——`packages/shell/` 一个目录装下了 shell 接缝的全部角色，`packages/subagent/` 装下了子代理接缝（第 14 章）。

反向判据同样有用：

- 如果某个「能力」只有 Definition 没有第二个 provider，问自己：它真的需要可替换吗？预发布期的仓库文化（根 `AGENTS.md`「foundation over blast radius」）鼓励先把接缝设计对，而不是先堆 provider。
- 如果你发现自己在 Consumer 里 `if (provider === 'xxx')` 分支，说明词汇类型没设计好——provider 差异应该被 Definition 的抽象吃掉。

## 动手练习

**练习 1：找第三条接缝。** 在 `packages/` 里任选一组——`fs/`、`web/`、`skill/`、`compaction/` 都可以——仿照本章对 shell 的分析，列出它的三个角色分别在哪个包：Definition 的 Service 类名与 `ctx` 键、provider 包、consumer 包。验证方法：打开 Definition 包确认存在 `extends Service` 的类，再 grep 谁 `inject` 了这个服务键。

**练习 2：画出替换路径。** 对你选的接缝回答：把 provider A 换成 provider B，需要动哪些文件？如果答案包含「Consumer 的包」，说明这条接缝有泄漏——找到泄漏点（通常是某个 provider 特有的类型溜进了共享词汇）。

**练习 3：request/spec 拆分体检。** 打开 `packages/shell/shell/src/types.ts`，对比 `ShellExecRequest` 与 `ShellExecSpec` 两个类型的字段差异，列出 `resolve()` 到底「决定」了哪些事。然后看你入门篇写的工具（第 5 章）：它的参数解析里有没有该显式化的隐式默认？

## 延伸阅读

- `docs/glossary.md` — capability-seam 与 agent-scope 条目。
- `docs/capability-seams.md` — 全仓库接缝清单（生成文档）。
- `packages/README.md` — 包分组表，按接缝组织目录的实证。
- `docs/subsystems/llm-streaming.md` — llm 接缝的流式细节。
- 入门篇 [第 5 章 写你的第一个工具](../beginner/05-first-tool.md) 的 Consumer 视角；下一章 [第 12 章 工具执行管线](./05-tool-pipeline.md)。
