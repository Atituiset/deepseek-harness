# 第 3 章 Cordis 基础：五个核心概念

## 本章目标

- 掌握 Cordis 的五个核心概念：插件、Context 与服务、inject、类型化事件、effect 生命周期。
- 完成 `docs/cordis-tutorial/` 的第 1–6 章（全程免 API key）。
- 按推荐顺序通读一遍 Cordis 框架源码（约 1700 行），建立「框架就这么点东西」的底气。

本章的概念部分对应 `docs/cordis-primer.md`，动手部分对应 `docs/cordis-tutorial/`。这本教程不再重复它们的内容，而是帮你把三者（概念、动手、源码）对齐。

## 概念一：插件是一个函数（加两个可选导出）

一个 Cordis 插件最常见的形态就是一个模块，导出 `apply` 函数：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'
export const inject = ['tools']   // 可选：声明依赖的服务

export function apply(ctx: Context) {
  // 在这里注册本插件贡献的一切
}
```

框架加载插件时调用 `apply(ctx)`；插件卸载时，`apply` 里通过 `ctx` 注册的所有东西被自动回收。除了函数形态，还有对象形态（`export default { name, inject, apply }`）和类形态（`extends Service`，用于提供服务，见概念三）。`docs/cordis-tutorial/01-first-plugin.md` 有三种形态的完整对照。

要点：**插件里没有框架启动代码**。你的文件只描述「我贡献什么」，组合和装配交给配置文件。

## 概念二：Context 是服务的仓库

`ctx` 是插件之间相遇的唯一场所。一个服务占据 `ctx` 上的一个稳定键——`ctx.tools`、`ctx.llm`、`ctx.sessions`——其他插件通过这个键找到它，而**不需要 import 它的具体实现**。

这就是第 1 章能力接缝能成立的原因：Consumer 只认识 `ctx.shell` 这个键，Provider 把实现挂到这个键下，换掉 Provider 时 Consumer 无感知。类型层面，`ctx` 上有什么键靠 TypeScript 的**声明合并**扩展——每个服务包向 `@deepseek-ai/cordis` 的接口合并自己的条目。

## 概念三：inject 声明依赖，加载顺序由服务可用性驱动

```ts
export const inject = ['tools']
```

写了这行，Cordis 就保证 `ctx.tools` 存在之后才调用你的 `apply`。没写而直接访问 `ctx.tools`，就是拿一个可能 undefined 的值去赌。

推论很重要：**`cordis.yml` 里的行顺序不决定加载顺序**。`packages/bundle/base/cordis.patch.yml` 的头部注释把这一点写死了："Row order carries no load semantics (activation is service-availability driven)"。配置里把行排成什么顺序只是给人看的；真正的启动顺序是一张由 `inject` 声明构成的依赖图。

这也意味着：一个插件如果永远等不到它 inject 的服务，它就一直不加载——排查「我的插件怎么没生效」时，先检查 inject 声明拼写，再检查服务是否真的有人提供。`docs/cordis-tutorial/06-composition-and-hmr.md` 专门讲怎么诊断这种情况。

## 概念四：类型化事件是插件间的通信协议

服务方法用于「直接调用能力」，事件用于「观察和拦截」。Cordis 有四种分发模式（`docs/cordis-primer.md` 的表格）：

| 模式 | 等待完成？ | 有返回值？ | 用途 |
|---|---|---|---|
| `emit` | 否 | 无 | 广播一个事实，监听器只观察 |
| `waterfall` | 否 | 有 | 中间件式拦截，可改写或短路 |
| `parallel` | 是 | 无 | 并发通知所有监听器 |
| `serial` | 是 | 有 | 按注册顺序依次执行 |

事件名和载荷类型同样通过声明合并注册，所以 `ctx.on('tools/pre-execute', ...)` 的参数类型是推导出来的，不是手写的。

**waterfall 语义是全仓库最容易踩的坑**，`docs/cordis-primer.md` 用了一节专门讲，这里原文强调：

> A listener receives `(...args, next)`. Call `next()` to delegate...; return without `next()` to short-circuit.

一个 waterfall 监听器收到 `(...args, next)`：**调用 `next()` 是把（可能被改写过的）流程委托给下一个监听器；不调用 `next()` 直接返回，就是短路**——后面的监听器和默认实现都不会执行。这不是可选项。一个只想「看一眼」的监听器如果忘了调 `next()`，它会静默地掐断整条链。仓库的 AGENTS.md 把这条列为铁律："Waterfall listeners MUST call `next()`"。第 6 章写权限钩子时你会亲手用到这个语义：拒绝时短路返回 `{ kind: 'deny' }`，放行时 `return next()`。

## 概念五：注册即 effect，卸载自动回滚

Cordis 里一切注册都是**可逆的 effect**：

- `ctx.on('event', listener)` — 返回一个 disposer；插件卸载时自动调用。
- `ctx.tools.register(tool)` — 同上，卸载即注销工具。
- `ctx.effect(() => { ...; return dispose })` — 通用形式：执行一段建立逻辑的代码，返回它的逆操作。

```ts
export function apply(ctx: Context) {
  ctx.effect(() => {
    const timer = setInterval(() => console.log('heartbeat'), 5000)
    return () => clearInterval(timer)   // 插件卸载时执行
  })
}
```

这条纪律的实践意义：**你不需要写清理代码的对账逻辑**。事件监听、定时器、工具注册、prompt 段——只要是通过 `ctx` 注册的，卸载时全部自动回滚。这也是热重载（HMR）能安全工作的前提：`docs/cordis-tutorial/06-composition-and-hmr.md` 会演示改一行插件代码、框架自动卸载旧版挂载新版，中间没有任何状态泄漏。

## 动手练习：Cordis 教程第 1–6 章

这是本章的主练习，预计 1–2 小时。全程免 API key。

1. 按 `docs/cordis-tutorial/index.md` 的「准备工作」建立临时目录：

   ```sh
   pnpm install   # 如果第 2 章没做过
   mkdir -p tmp/cordis-tutorial
   cd tmp/cordis-tutorial
   ```

   `tmp/` 已被 git 忽略，随便写。

2. 依次完成第 1–6 章。每章都是同一个启动命令：

   ```sh
   node --import tsx ../../vendor/cordis/bin.js
   ```

   这个单文件启动器（`vendor/cordis/bin.js`）创建根 Context、挂载 Loader、从当前目录的 `./cordis.yml` 读插件清单。章节顺序：第一个插件 → 生命周期与 effect → 服务 → 事件 → 配置 → 组合与 HMR。

3. 做第 4 章（事件）时，**故意写一个不调 `next()` 的 waterfall 监听器**，观察下游监听器收不到事件，确认你理解了短路语义。
4. 第 7 章（进入 harness）会把你写的插件接到真实 harness 服务上，可以现在做，也可以学完第 5 章再回来。

## 源码导读顺序

Cordis 的全部核心就是 `vendor/cordis/src/` 下五个文件，按这个顺序读（行数为写作时实测，会漂移）：

1. **`service.ts`（115 行）** — `Service` 基类。核心发现：构造函数里 `super(ctx, name)` 一调用，服务就注册进 Context，并随所属 fiber 自动注销。服务的整个生命周期就这一个动作。
2. **`context.ts`（146 行）** — `Context` 类。看它如何用 mixin 把 events/registry/fiber 三套能力组合到一个 `ctx` 上。
3. **`events.ts`（352 行）** — `emit`/`waterfall`/`parallel`/`serial` 的实现。重点看 waterfall 的 `next()` 链是怎么串起来的。
4. **`registry.ts`（337 行）** — 插件注册表：插件怎么被解析、加载、按 inject 等待。
5. **`fiber.ts`（754 行）** — 最长的文件，effect 的账本。每个插件对应一个 fiber，记录它注册的一切，卸载时逐条回滚。

读完这五个文件，你对框架的神秘感应该已经消失——剩下的都是这五个概念的组合运用。

## 延伸阅读

- `docs/cordis-primer.md` — 五个概念的官方精简版，值得原文读一遍。
- `docs/cordis-tutorial/` — 本章练习的教材，第 7 章接入真实 harness。
- `docs/cordis-api/` — 生成的 Cordis API 参考。
- `docs/user/develop/framework/service.md` — harness 视角的服务与依赖讲解。
- 进阶篇[第 8 章](../advanced/01-architecture.md)会讲 harness 在 Cordis 之上建立的事件域分层（session 事件 / agent 事件 / 能力事件）。
- 下一章：[第 4 章 解剖一个真实 Agent 配置](./04-anatomy-config.md)。
