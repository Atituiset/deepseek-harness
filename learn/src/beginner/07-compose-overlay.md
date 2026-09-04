# 第 7 章 组合与覆盖：打造自己的 Agent

## 本章目标

- 理解 dsh 的组合层级：profile → bundles → cordis.patch.yml → home patch → `--patch` overlay。
- 掌握 patch 的三种操作：按 `id` 整体替换一行、`disabled: true` 禁用一行、`- insert:` 插入新行。
- 参照三个实战蓝本（最小 Agent、定时提醒、MCP 记忆），组合出自己的 Agent 并跑通。

## 组合层级

`docs/architecture.md` 第 15–37 行是这一章的权威依据。核心事实：

一次 `dsh` 启动，是把若干**有序的层**依次应用到一个空的条目列表上：

```text
profile 列出的每个 bundle（按列出顺序）
  → profile 自己的 cordis.patch.yml
    → home 级 cordis.patch.yml（$DSH_HOME/cordis.patch.yml）
      → 命令行的 --patch overlay（可重复，按 argv 顺序）
```

几个名词（`docs/architecture.md` 第 19–25 行）：

- **profile** 是存在 harness home 里的命名组合：它列出自己叠加哪些 bundle、持有用户自己的 `cordis.patch.yml`。`web`、`headless`、`sdk`、`sdk-minimal`、`acp` 是随产品发布的模板。
- **bundle** 是「Cordis 配置行 + 这些行挂载的代码」的分发格式。`dsh-base`（`packages/bundle/base/cordis.patch.yml`）是 `web`/`headless`/`sdk`/`acp` 这些底座型 profile 的共享第一层；`dsh-web-app` 加浏览器应用、`dsh-headless` 加一次性执行器、`dsh-sdk-app` 加 SDK JSON-RPC 服务器、`dsh-acp-app` 加 ACP 服务器。`dsh-sdk-minimal` 是刻意的例外：它不叠 base，一个 bundle 拥有完整独立的 SDK 组合树。
- 每个 profile/bundle 在自己的 `package.json` 的 `dsh` 字段里声明身份：`dsh.profile` 列出 profile 的 bundle 清单，`dsh.bundle` 指向 bundle 的 patch 文件。

看你机器上真实启动的那棵树（这条命令第 4 章用过）：

```sh
pnpm dsh --profile headless --dump-config
```

它打印出的任何一行，都可以被你自己的 patch 替换。

## patch 的三种操作

`docs/architecture.md` 第 27 行："A patch targets a row by id and replaces its whole config, or inserts new rows." 具体是三种：

**1. 按 id 替换整行。** patch 里出现一个与下层同 `id` 的行，就整体替换那一行——注意是**替换整个 `config`，不是合并**。想改一个字段，也要把整份 config 写全。

**2. 禁用一行。** 同 `id` 的行加上 `disabled: true`：

```yaml
- id: tool-todo
  disabled: true
```

`disabled` 字段还允许 `!!js` 表达式（每次挂载决策时求值），这是除 `config` 之外唯一允许 `!!js` 的地方（第 4 章的规则）。

**3. 插入新行。** 用 `- insert:` 列表：

```yaml
- insert:
    - id: word-count
      name: '/绝对路径/scratch-plugin/src/my-plugin.ts'
```

第 5、6 章的 scratch-plugin overlay 用的就是它。「三种操作全用上」的更大示范就在产品自己身上：`packages/bundle/base` 里 `hmr` 行 `disabled: true`（禁用）、`sdk-minimal` 不叠 base 而用 `insert` 自建全树（插入）、mode bundle 改写 base 的 `system-prompt`/`tools` 行（替换）——能力接缝 + 组合覆盖的威力写在每个 bundle 里。

## 三个实战蓝本

### 蓝本一：最小 Agent —— `packages/bundle/sdk-minimal/cordis.patch.yml`

168 行，是所有 shipped 组合里唯一**不叠 `dsh-base`** 的：一个 bundle 的 `insert` 就是完整的 Cordis 树。模型可见的工具**只有两个**：持久 `bash`（`dsh-tool-bash-persistent`，行内可数）和 `str_replace_editor`。它值得逐行读，因为它展示了「最小」不是删出来的，而是选出来的：

- 没有 settings、遥测、审批、web、skills——每缺席一行都是一次有意的取舍，决策记录见 `.agents/notes/implemented/feature/2026-08-11-minimal-profiles-bare-two-tool-runtime.md`。
- `llm-deepseek` 的 `defaultContextWindow` 用 `!!js Number(process.env.DSH_CONTEXT_WINDOW ?? 1000000)` 从环境取值——`!!js` 在 `config` 里的标准用法。

注意它的运行方式：这是 Python SDK 打包运行时的组合，`python/sdk/examples/minimal.py` 是驱动脚本，SDK 延迟启动 `dsh --profile sdk-minimal` 进程并复用到上下文管理器退出（`docs/user/guide/python-sdk.md`）。也可以直接 `pnpm dsh --profile sdk-minimal --dump-config` 看它的树。

### 蓝本二：overlay 加能力 —— `apps/cli/config/examples/schedule/cordis.yml`

一份对现成 Web 组合的 opt-in patch：一次 `- insert:`（`time-context` + `schedule` 两个插件行）加一行 `ui-schedule: disabled: false`，给 `dsh web` 加上定时提醒能力（`schedule_create` / `schedule_list` / `schedule_delete` 三个模型工具）：

```sh
pnpm dsh web --patch apps/cli/config/examples/schedule/cordis.yml
```

这是 overlay 的标准用法：**不改任何交付物，把能力叠上去**。`docs/user/guide/schedule.md` 有完整的语义说明。

### 蓝本三：overlay 接外部系统 —— `apps/cli/config/examples/mcp-memory/`

三份默认关闭的参考配置（`engram` / `mcp-reference-memory` / `memorix`），各自通过 `@deepseek-ai/dsh-mcp-client` 把一个记忆 MCP server 接进来。模板本身（任一文件打开即见）就是接任意 MCP server 的配方：

```yaml
- insert:
    - id: memory-my-server
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: my-memory
        transport: stdio
        command: my-memory-mcp
        args: []
```

一个插件行 = 启动 MCP 子进程 + 发现它的工具 + 以 `mcp__<serverName>__<tool>` 注册给模型。想跨次运行保留，把这段合并进 `$DSH_HOME/profiles/<名字>/cordis.patch.yml`（只对一个 profile 生效）或 `$DSH_HOME/cordis.patch.yml`（全机生效）。

## 组合 vs 写代码：什么时候用哪招

到这一章，你手里有三种扩展手段，选择规则：

- **配置能表达的，不写代码。** 换模型、换 Provider、改超时、关工具、改 persona——改配置行。
- **加现成能力，用 insert。** 别人写好的插件（MCP client、schedule），overlay 一行挂上。
- **没有现成插件，才写插件。** 第 5、6 章的技能用在刀刃上：新工具、新策略、新观察点。

仓库自己的纪律与此同构："Plugins, not loop changes"——新行为走文档化的扩展点，而不是去改 agent-loop。

## 动手练习

1. **看清你脚下的树。** 运行 `pnpm dsh --profile headless --dump-config`，从输出里找出第 4 章解剖过的几个 id（`llm-deepseek`、`bash-sandbox`、`agent-loop`……）。再用 `--dump-default-config` 对比，确认差异来自用户层。

2. **替换一行。** 新建 `my-agent/cordis.yml`，把 `bash-sandbox` 行的 `timeoutMs` 改成 10000（`config` 要写全整份，不能省略）：

   ```yaml
   - id: bash-sandbox
     name: '@deepseek-ai/dsh-bash-sandbox'
     disabled: !!js process.platform === 'win32'
     config:
       timeoutMs: 10000
   ```

   先验证组合结果：`pnpm dsh --profile headless --patch ./my-agent/cordis.yml --dump-config`，确认那行的 config 已被整体替换。再跑一个会超时的任务（如让 Agent `sleep 30`）确认行为变化。

3. **禁用 + 插入。** 在同一个 overlay 里禁用 `tool-todo` 行，并 insert 第 5 章的 `word_count` 工具。运行 headless，先确认 Agent 声称自己没有 `todo_write`，再让它调用 `word_count`。

4. **向最小 Agent 靠拢。** 打开 `packages/bundle/sdk-minimal/cordis.patch.yml`，对照 headless 的 `--dump-config` 输出，给你的 overlay 追加若干 `disabled: true` 行，把 headless profile 裁到只剩 bash + 文件工具。每裁一刀跑一次简单任务验证没裁坏。体会：sdk-minimal 是从空列表**加**出来的最小集，你这里是从完整 profile **减**出来的最小集——同一份组合语义的两个方向。

## 延伸阅读

- `docs/architecture.md` — 组合层级的权威描述；进阶篇[第 8 章](../advanced/01-architecture.md)精读全篇。
- `packages/boot/app-boot/README.md` — profile 机制的完整细节（组合如何落地到 `$DSH_HOME`）。
- `apps/cli/config/examples/` — 一组 opt-in overlay 示例（schedule、mcp-memory、github-review、cordis）。
- `packages/bundle/sdk-minimal/cordis.patch.yml` 与 `docs/user/guide/schedule.md` — 最小组合与 overlay 加能力的两个蓝本。
- 进阶篇[第 11 章](../advanced/04-capability-seam.md) — 当你组合出的 Agent 需要换掉某个能力域时，这一章讲接缝的设计与拆分。
- 入门篇到此结束。进阶篇从[第 8 章 架构总览](../advanced/01-architecture.md)开始。
