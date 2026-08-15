# 第 6 章 写你的第一个插件

## 本章目标

- 掌握两种最常见的 harness 插件形态：观察插件（监听 `session/event`）和拦截插件（`tools/pre-execute` 权限钩子）。
- 体会「注册即 effect、卸载自动回滚」在真实插件里的含义。
- 学会把插件挂进 cordis.yml 并被加载的完整流程。

第 5 章你写的工具其实已经是插件了——工具插件是「向注册表贡献条目」的插件。本章讲另外两类：**只看不改**的观察者和**参与决策**的拦截者。它们覆盖了 `docs/cookbook/extension-cookbook.md` 里大部分日常扩展场景。

## 挂载流程（两种形态共用）

无论写哪种插件，挂进 harness 的方式和第 5 章一样（完整流程见 `docs/user/develop/basic/index.md` 的 scratch-plugin 教程）：

1. 写一个导出 `apply` 的 TypeScript 模块。
2. 写一份 overlay 文件，用 `- insert:` 插入新行，`name` 是插件文件的**绝对路径**——patch 文件只贡献配置，不改变 loader 解析模块的基准目录：

   ```yaml
   - insert:
       - id: my-plugin
         name: '/绝对路径/deepseek-harness/scratch-plugin/src/my-plugin.ts'
   ```

3. 用 `--patch` 启动：`pnpm dsh web --patch ./scratch-plugin/cordis.yml`。

第 7 章会讲清楚 `--patch` 在整个组合层级里的位置，现在照做即可。

## 形态一：观察插件——监听 session/event

harness 里有一条「一切事实的广播总线」：`session/event`。会话里发生的每一件持久事实——用户消息、assistant 的每个流式 chunk、工具调用与结果、轮次边界——都会作为事件在这条总线上广播。UI、遥测、日志都靠订阅它工作。

一个把每条 assistant 文本记一行日志的插件（改编自 `docs/cookbook/extension-cookbook.md` 的 UI 插件模式）：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'assistant-logger'

export function apply(ctx: Context) {
  ctx.on('session/event', (_session, event) => {
    if (event.type === 'assistant/chunk' && event.data.chunk.type === 'text-delta') {
      process.stderr.write(event.data.chunk.text)
    }
  })
}
```

三个要点：

- **事件名和载荷类型是推导出来的。** `session/event` 的监听器签名来自声明合并，`event.type` 可判别到一个封闭联合，IDE 能补全所有事件类型。这就是为什么仓库约定「类型化事件用声明合并」。
- **`emit` 模式的监听器只观察。** 它没有 `next`，也不该有副作用之外的企图。想拦截，去找 waterfall 事件。
- **常用观察事件速查**：`assistant/chunk`（流式输出）、`tool/call` 与 `tool/result`（工具活动）、`turn/end`（一轮结束）。完整的 session 事件清单在进阶篇[第 10 章](../advanced/03-session-events.md)。

## 形态二：拦截插件——tools/pre-execute 权限钩子

观察不够时，就要参与决策。工具执行管线的第一道门是 `tools/pre-execute`：一个 waterfall 事件，监听器返回类型化的放行/拒绝决定。`docs/cookbook/extension-cookbook.md` 的 permission-gate 模式：

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

export const name = 'permission-gate'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (exec.name === 'bash' && /* 你的判定逻辑 */ false) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

回顾第 3 章的 waterfall 语义，这里两个分支各自体现了它的一半：

- **拒绝 = 短路。** 返回 `{ kind: 'deny' }` 而不调 `next()`，后续监听器和工具本体都不会执行。对单决策事件，短路就是设计意图——这个监听器拥有这个决定。
- **放行 = 委托。** `return next()` 把决定权交给链上的下一个监听器。只观察、不打算拦的监听器**必须**走这条路，否则它会静默掐断整条策略链。

工具管线上还有一串扩展点，按职责选用（选择规则见 `docs/cookbook/adding-a-tool.md` 的「Execution policy and observation」）：

| 扩展点 | 模式 | 用途 |
|---|---|---|
| `tools/pre-execute` | waterfall | 放行/拒绝/询问的策略门 |
| `ctx.tools.guard()` | — | 单调终局拒绝，后续监听器无法撤销 |
| `tools/execute` | waterfall | 包裹真实执行：超时、重试、指标 |
| `tools/post-execute` | waterfall | 改写结果呈现或附加模型可见上下文 |
| `tools/result` | emit | 观察不可变的最终结果（审计、指标） |

## 注册即 effect：卸载会发生什么

本章两个插件都没有写任何清理代码，这不是偷懒。第 3 章说过：`ctx.on()` 返回 disposer，插件卸载时 Cordis 自动调用。实践含义：

- **改代码即热重载。** base bundle 挂了 `cordis-plugin-hmr`，保存插件文件，旧监听器撤销、新版本挂载，不会重复注册。
- **配置里删掉一行即拆除。** 把 overlay 里的行删掉重启，插件的监听器、工具、定时器全部消失，不留残骸。
- **需要手动管理的资源用 `ctx.effect()` 包起来**，返回它的逆操作（第 3 章的定时器例子）。

这条纪律反过来也是约束：**永远不要绕过 `ctx` 直接改全局状态**。直接 `process.on(...)`、修改别的模块的单例，这些都不会被回滚，是仓库级 bug。

## 动手练习

继续用第 5 章的 `scratch-plugin/` 目录。

1. **观察插件。** 新建 `scratch-plugin/src/observer.ts`，用上面的 `assistant-logger` 代码，再加一个分支：遇到 `tool/result` 事件时把工具名和 `isError` 打到 stderr。在 `scratch-plugin/cordis.yml` 的 `insert` 列表里加一行挂它（`id: observer`），跑：

   ```sh
   pnpm dsh --profile headless --patch ./scratch-plugin/cordis.yml "列出当前目录的文件"
   ```

   在 stderr 里找到 assistant 的流式文本和工具结果日志。

2. **权限钩子。** 新建 `scratch-plugin/src/gate.ts`，写一个 `tools/pre-execute` 监听器：当 `exec.name === 'bash'` 且参数中的命令包含 `rm ` 时返回 `{ kind: 'deny', reason: 'no rm allowed' }`，否则 `return next()`。挂进 overlay，然后让 Agent 执行一个含 `rm` 的任务，确认：工具被拒绝、拒绝理由作为工具结果反馈给模型、模型换用别的方式或报告无法完成。

3. **验证短路语义。** 在 gate 插件里注册**两个** `tools/pre-execute` 监听器：第一个只打日志然后 `return next()`，第二个在日志里写「我收到了」。跑一次确认两条日志都出现。然后把第一个监听器的 `return next()` 改成直接 `return { kind: 'deny', reason: 'test' }`，再跑一次：第二个监听器的日志消失了——这就是短路。体会：第一个监听器如果只想观察却忘了 `next()`，后面的策略链会被它静默掐断。
4. **验证回滚。** 在 headless 运行期间不可行（一次性进程），改用 `pnpm dsh web --patch ./scratch-plugin/cordis.yml` 启动后，编辑 `observer.ts` 保存，观察 HMR 日志确认插件被卸载重载，且日志没有重复输出。

## 延伸阅读

- `docs/cookbook/extension-cookbook.md` — 扩展模式参考：UI 插件、协议驱动、「feature → mechanism」总表。
- `docs/user/develop/basic/index.md` — scratch-plugin 流程的官方教程。
- `packages/core/tools/README.md` — 工具管线各扩展点的输入、顺序、返回值、失败行为。
- `docs/architecture.md` 的 Events 一节 — session 事件 / agent 事件 / 能力事件的分层；进阶篇[第 8 章](../advanced/01-architecture.md)展开。
- 进阶篇[第 12 章](../advanced/05-tool-pipeline.md)精读工具执行管线，你会看到自己挂的钩子在整个链路中的确切位置。
- 下一章：[第 7 章 组合与覆盖：打造自己的 Agent](./07-compose-overlay.md)。
