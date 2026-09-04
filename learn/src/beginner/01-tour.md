# 第 1 章 仓库导览：一切皆插件

## 本章目标

- 建立对 deepseek-harness 仓库的整体地图：哪个目录装什么。
- 理解「一切皆插件」这句口号的字面含义——它不是修辞，是事实陈述。
- 认识贯穿全仓库的设计模式：能力接缝（capability seam）的三个角色。
- 拿到一份「什么需求去哪个目录」的速查表，之后每章都会用到。

## 一切皆插件

打开根目录的 `AGENTS.md`，第一句话就是仓库的宪法：

> DeepSeek Harness is a plugin-based agent harness on vendored Cordis: **everything is a plugin**.

这句话要按字面理解。在 `docs/architecture.md` 里有更具体的版本（第 11 行）：

> Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so every part is replaceable from configuration.

也就是说，这个仓库里**没有一个享有特权的核心**。你印象中「Agent 框架本体」该有的东西——调模型的代码、工具注册表、会话日志、驱动模型一轮一轮干活的主循环——在这里全都是普通插件，挂在同一棵插件树上，和你的插件遵守同一套规则。第 13 行补上了推论：

> There is no privileged core to patch: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads.

扩展这个系统的方式只有一种：把你的插件挂到别的插件旁边。想换掉某个部件的方式也只有一种：在配置里把它那行改掉。

支撑这一切的框架是 Cordis，它以 vendored 源码的形式放在 `vendor/cordis/`。第 3 章会专门讲它；现在只需要记住：**Cordis 提供插件机制，harness 用插件机制搭出 Agent**。

## 仓库布局

根目录 `AGENTS.md` 的「Repository layout」一节是权威清单，这里按学习视角重新组织一遍。

### `vendor/` — 框架底座

`vendor/cordis/` 是插件框架 Cordis 的源码副本（pinned source copy），不是 npm 依赖。为什么 vendored 而不是引用上游包，`vendor/README.md` 里有清单和同步流程。学习时这很重要：**框架源码就在仓库里，可以直接读**。第 3 章会带你读它的五个核心文件，总共约 1700 行。

同目录还有 `cosmokit`、`loader`、`hmr` 等 Cordis 生态包，同样是 vendored 副本。

### `packages/` — 产品本体

所有产品代码都在 `packages/<分组>/<包>/` 下，每个包名都是 `@deepseek-ai/dsh-<名字>`。分组（`packages/README.md` 有完整说明）里最值得先记住的：

- `packages/core/` — 产品 API 脊柱：session（事件溯源日志）、system-prompt（提示词装配）、tools（工具注册表与执行管线）、agent（Agent 接口与注册表）、agent-loop（默认主循环）。进阶篇主要读这里。
- `packages/llm/` — LLM 能力：抽象服务定义 + DeepSeek 适配器。
- `packages/shell/`、`packages/fs/`、`packages/web/` 等 — 各个能力域，每个都按能力接缝组织（下文讲）。
- `packages/bundle/` — 可安装的 bundle：把一组插件打包成一个配置层。`packages/bundle/base/cordis.patch.yml` 是每个 profile 的第一层。
- `packages/examples/` — 剩余的演示包（`acp-demo`、`jsonrpc-demo`），现代产品组合已收进 `packages/bundle/`。
- `packages/bundle/` — 产品的全部 bundle 层：`base`（每个 profile 的共享底座）、`headless`、`web-app`、`sdk-minimal` 等。`packages/bundle/base/cordis.patch.yml` 是每个 profile 的第一层，也是第 4 章的解剖对象。
- `packages/boot/` — 应用启动的共享胶水。

### `packages/bundle/` — 可运行组合的权威来源

产品自己发布的组合就是最好的「配置样板间」，全在 `packages/bundle/` 下，每个 bundle 一份 `cordis.patch.yml`：

- `packages/bundle/base/cordis.patch.yml` — 每个底座型 profile 的第一层（498 行、注释齐全）：LLM 接缝、工具、持久化、沙箱与审批、settings、credentials、遥测都在里面。第 4 章解剖它。
- `packages/bundle/headless/cordis.patch.yml` — 一次性执行器层，叠在 base 上就是 `dsh --profile headless`。
- `packages/bundle/sdk-minimal/cordis.patch.yml` — 第 7 章的最小 Agent 蓝本：不叠 base，自成完整组合。

另外 `apps/cli/config/examples/` 下有一组小型 overlay 示例（`cordis`、`schedule`、`mcp-memory` 等），第 7 章的 overlay 实战素材。

学习时把 `packages/bundle/` 当作「配置样板间」：它们展示了真实 Agent 是怎么由插件清单组装出来的。

### 其余目录

- `apps/cli/` — `dsh` 命令行入口。根 `package.json` 的 `dsh` 脚本指向 `apps/cli/src/bin.ts`，第 2 章就会用到。
- `apps/web/` — Web 应用。
- `docs/` — 文档主库：`cordis-primer.md`（框架概念）、`cordis-tutorial/`（免 key 动手教程）、`cookbook/`（扩展模式参考）、`architecture.md`（架构权威文档）。
- `python/` — Python SDK 及其打包运行时。
- `scripts/` — 仓库质量门禁与生成器，进阶篇测试章会涉及。
- `.agents/notes/` — Agent Notes：仓库的设计决策档案。读到某个设计想不明白「为什么」时，来这里找。

## 能力接缝：三个角色

翻 `packages/` 时你会反复看到同一种三分结构，它叫**能力接缝（capability seam）**，`docs/glossary.md` 有正式定义。直觉版本：

一个可替换的能力被拆成三个角色：

1. **Service Definition（服务定义）** — 定义这个能力的抽象：它占据 `ctx` 上的哪个键（比如 `ctx.shell`）、它的方法签名是什么。它自己不干活。
2. **Service Provider（服务提供方）** — 真正干活的实现。同一个 Definition 可以有多个 Provider，互相可替换。
3. **Consumer（消费方）** — 使用这个能力的插件。它只依赖 Definition 声明的 `ctx.<key>`，不关心背后是哪个 Provider。

`docs/glossary.md` 给的标准例子是 shell 能力：

| 角色 | 包 | 干什么 |
|---|---|---|
| Definition | `dsh-shell` | 声明 `ctx.shell` 的抽象接口 |
| Provider | `dsh-bash-local`、`dsh-bash-sandbox` | 本地执行 / 沙箱执行，二选一可替换 |
| Consumer | `dsh-tool-bash` | 模型可见的 bash 工具，调用 `ctx.shell` |

这个结构的意义在**可替换性**：想把 bash 从「直接在本机跑」换成「在沙箱里跑」，只需在配置里换一个 Provider 行，工具（Consumer）和接口（Definition）一行代码都不用动。同一个模式贯穿 LLM（`ctx.llm`）、文件系统、子代理、压缩等所有能力域。第 4 章解剖真实配置时，你会逐行看到这三个角色。

一条重要纪律（同样来自 `AGENTS.md`）：一个能力接缝是 Definition / Provider / Consumer 的**完整组合**，永远不要只交付其中一个角色。

## 建议阅读顺序

这个仓库的文档是为不同读者写的，第一次通读建议这个顺序：

1. **`docs/cordis-primer.md`** — 一页纸讲完 Cordis 的五个核心概念。先读它建立词汇表。
2. **`docs/cordis-tutorial/index.md`** — 免 API key 的七章动手教程，从零目录开始写插件。第 3 章会引导你做前六章。
3. **`docs/architecture.md`** — 架构权威文档，讲清 profile/bundle 组合层、核心包、事件域。
4. **`docs/glossary.md`** — 术语表，遇到不懂的词回来查。
5. **`docs/cookbook/`** — 写具体扩展时按需查阅：加工具、加权限钩子、加 LLM 适配器等。

遇到「这个设计为什么是这样」的问题，去 `.agents/notes/` 按主题找 Agent Note——比如能力接缝的理由记在 `.agents/notes/implemented/architecture/2026-06-13-capability-seams.md`。

## 速查表：什么需求去哪个目录

| 我想…… | 去哪里 |
|---|---|
| 理解插件框架本身 | `docs/cordis-primer.md` → `vendor/cordis/src/` |
| 跑一个现成 Agent | `pnpm dsh --profile headless "任务"`（第 2 章） |
| 看一份完整 Agent 配置 | `packages/bundle/base/cordis.patch.yml`（第 4 章） |
| 加一个模型可调用的工具 | `docs/cookbook/adding-a-tool.md`（第 5 章） |
| 加权限/拦截逻辑 | `docs/cookbook/extension-cookbook.md`（第 6 章） |
| 组合出自己的 Agent | `packages/bundle/` 与 `apps/cli/config/examples/` 的 patch 文件（第 7 章） |
| 读 Agent 主循环源码 | `packages/core/agent-loop/`（进阶篇第 9 章） |
| 读会话/事件溯源 | `packages/core/session/`（进阶篇第 10 章） |
| 换掉模型适配器 | `packages/llm/`（进阶篇第 13 章） |
| 查某个设计决策的理由 | `.agents/notes/` |

## 动手练习

1. **定位脊柱。** 打开 `packages/core/`，列出它的子目录，对照 `docs/architecture.md` 第 43–51 行的核心包表格，把每个子目录和它占据的 `ctx` 键（`ctx.sessions`、`ctx.tools` 等）一一对应。
2. **找一个接缝。** 打开 `packages/shell/` 目录，确认里面能分出 Definition、Provider、Consumer 三类包；再打开 `packages/llm/`，做同样的分类。注意 `docs/glossary.md` 提到的一个例外：`dsh-llm` 一个包同时拥有 Definition 和 Consumer 两个角色——想想为什么这两个角色可以不分家。
3. **验证「一切皆插件」。** 打开 `packages/bundle/base/cordis.patch.yml`，数一下前 20 行里出现了哪些你以为会是「框架本体」的东西（比如 `llm`、`session`、`agent`）。它们全都是配置里的普通插件行。

## 延伸阅读

- `AGENTS.md`（仓库根目录）— 仓库宪法，所有贡献约定的单一权威来源。
- `docs/architecture.md` — 架构总览；进阶篇[第 8 章](../advanced/01-architecture.md)会逐节精读它。
- `docs/glossary.md` — 能力接缝的正式定义；进阶篇[第 11 章](../advanced/04-capability-seam.md)专门讲这个模式。
- `vendor/README.md` — vendored 包的清单与同步流程。
- 下一章：[第 2 章 环境准备：跑通你的第一个 Agent](./02-setup-run.md)。
