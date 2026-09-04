# 第 4 章 解剖一个真实 Agent 配置

## 本章目标

- 逐段读懂 `packages/bundle/base/cordis.patch.yml`——产品每个 profile 的共享底座，近 500 行、注释齐全的真实编码 Agent 配置。
- 能用能力接缝的三角色（Definition / Provider / Consumer）标注配置里的每一行。
- 理解 cordis.yml 的两条寻址规则：行按 `id` 寻址；`!!js` 只允许出现在 `config` 和 `disabled` 字段。
- 看懂「base 留给 mode bundle 什么」这一分层思想。

## 先建立读法

一份 `cordis.patch.yml` 是一个**插件条目列表**。每个条目最少两个字段：

```yaml
- id: bash-sandbox                        # 行的名字，供 patch 寻址
  name: '@deepseek-ai/dsh-bash-sandbox'   # 插件模块：npm 包名或路径
  config:                                # 可选：传给插件的经过校验的配置
    timeoutMs: 60000
```

三条规则（第 3 章和 `docs/cordis-primer.md` 都有出处）：

1. **行顺序无加载语义。** 启动顺序由 `inject` 依赖图驱动，文件里的排列只是给读者看的分组。`packages/bundle/base/cordis.patch.yml` 头部注释明说："Row order carries no load semantics"。
2. **`id` 是行的地址。** 后续的 patch 层通过 `id` 引用某一行、整体替换它的 `config`，或插入新行。第 7 章的组合覆盖全靠这个机制。
3. **`!!js` 是受限的逃生舱。** `@deepseek-ai/cordis-plugin-include` 会把 `!!js` 解析成表达式节点，但只允许出现在条目的 `config` 内部和 `disabled` 字段；其他元数据（`id`、`name`）保持字面量。想按环境条件换插件，用 overlay，不要在配置里写分支。

## 逐段解剖 dsh-base

打开 `packages/bundle/base/cordis.patch.yml`（498 行），跟着下面的分组读。每段标题是功能域，括号里是该行在能力接缝中的角色。

头部注释先交代两条大纪律：整份文件是**对空 profile 根的一次 `insert`**，上层 bundle 和用户的 `cordis.patch.yml` 都按 `id` 逐行覆盖，同一行最后写入者胜；patch 是**整体替换 `config` 而非合并**——所以「各模式取值不同的行」根本不住在 base 里，它们属于每个 mode bundle，让任何一行最多只有「一个 bundle 层 + 用户层」两份定义。

### 框架底座与 LLM 接缝

```yaml
- id: timer
  name: '@deepseek-ai/cordis-plugin-timer'
- id: hmr
  name: '@deepseek-ai/cordis-plugin-hmr'
  disabled: true
- id: llm
  name: '@deepseek-ai/dsh-llm'
```

`llm` 行就是 LLM 接缝的 **Definition**：`dsh-llm` 占据 `ctx.llm`、定义消息与流词汇。注意它下面没有跟任何适配器——Provider 在文件末尾（见下），这种「按功能域分组、不按依赖序排列」正是「行顺序无语义」的体现。

### 会话与存储：session 一族

`session`（Definition，`ctx.sessions`）、`session-log-deepseek`、`session-title` + `session-title-llm`、`session-persistence-jsonl`（持久化 Provider）、`session-query-sqlite`、`session-projection`（投影注册表）、`storage` 一族（`storage-json`、`storage-domain`、`spill-local`）——这一段是第 10 章整章的主角：事件溯源日志加上围绕它的查询、投影、落盘。注意 `session-persistence-jsonl` 只是普通插件行：持久化不在 core，是订阅 `session/event` 的接缝另一头。

### 用户面：settings 与 credentials（Provider）

```yaml
- id: settings
  name: '@deepseek-ai/dsh-settings-file'
- id: credentials
  name: '@deepseek-ai/dsh-credentials-local'
```

`settings` 读 `$DSH_HOME/settings.yaml`（热重载），`credentials` 管理继承环境变量与 `$DSH_HOME/.credentials.yaml`。注释点明关键设计：settings 文档里的一个 `llm-deepseek:` 小节可以在**不重启**的情况下覆盖下面适配器行的配置；第 2 章的 `DEEPSEEK_API_KEY` 就是 `credentials` 在**每次请求时**解析的——所以整个产品树里找不到任何 key 的字面量。

紧随其后的 `llm-pi-ai` 行是多 provider 的例子：以「休眠」形态挂载（零路由、模型选择器里不出现），直到 settings 出现 `llm-pi-ai:` 小节才激活路由。「哪些适配器存在」是组合层的事，「哪些 provider 在跑」是用户 settings 的事——两层刻意分开。

### 执行底座：subprocess、沙箱与 bash

```yaml
- id: subprocess
  name: '@deepseek-ai/dsh-subprocess-local'
- id: sandbox
  name: '@deepseek-ai/dsh-sandbox-local'
- id: bash-sandbox
  name: '@deepseek-ai/dsh-bash-sandbox'
```

`subprocess` 管子进程组（spawn/kill/输出管道），`sandbox` 是沙箱接缝的 Provider，`bash-sandbox` 是 shell 接缝的 Provider——沙箱内 bash。随后的 `approval`（审批 Provider）、`permission`、`shell-env` 是策略与审批域；`tool-bash`、`tool-pwsh`、`tool-jobs`、`fs-observation-policy` + `tool-fs` + `tool-fs-search` 是模型可见的工具 **Consumer** 行。注意注释强调的加载顺序依赖："Policy loads before the model-facing filesystem tools"——但这指的依然是 inject 依赖，不是行顺序。

### 子代理与工作流

`subagent`（Provider 注册表的 Definition）、`subagent-spawn-in-process` / `subagent-fork-in-process`（两个 Provider）、`tool-subagent-control`（`send_message`）、`tool-subagent-list-agents`、`tool-subagent`（配 `provider: spawn`，暴露成 `subagent` 工具）、`tool-subagent-fork`（**同一个 Consumer 包**配 `provider: fork`，暴露成 `subagent_fork`）——一个 Consumer 接不同 provider 产出不同模型工具，接缝的「一次替换、全局生效」写在一行配置里。`workflow-worker-thread` + `tool-workflow` 同构。第 14 章整章精读这段。

### 提示词装配与 loop 本体

文件末尾的三行值得单独看：

```yaml
- id: tools
  name: '@deepseek-ai/dsh-tools'
- id: system-prompt
  name: '@deepseek-ai/dsh-system-prompt'
- id: agent-loop
  name: '@deepseek-ai/dsh-agent-loop'
  config:
    agents: []
```

你印象中「Agent 框架本体」该有的东西——工具注册表、提示词装配、驱动模型一轮轮干活的主循环——在这里全是普通插件行，和 `timer` 平起平坐。这就是第 1 章「一切皆插件」在配置层面的直接证据。

最后一行是 LLM 接缝的 Provider：

```yaml
- id: llm-deepseek
  name: '@deepseek-ai/dsh-llm-deepseek'
```

注释写明：不内联任何 key 或 endpoint，两者每次请求都从 `llm-deepseek:` settings 小节与凭据存储解析。想换后端，把这一行换成 `llm-pi-ai` 激活、或在 settings 里写新的路由——Definition 与 Consumer 一行不动。

## mode bundle：headless 怎么长出来

base 只是底座。`packages/bundle/headless/cordis.patch.yml` 只有 30 行：改写 `system-prompt` 的 persona、调整 `tools` 的 mode，再 `insert` 一个 `headless-startup`（解析任务参数）和 `headless-runner`（一次性驱动 + 打印结果）。「一次性执行器」不是另一套 Agent，而是 base 之上薄薄一层。

`docs/architecture.md` 第 27 行描述了层级顺序：

> Layers apply to an empty entry list in this order: each bundle in the profile's listed order, then the profile's `cordis.patch.yml`, then the home-level one, then any `--patch` overlay. A patch targets a row by id and replaces its whole config, or inserts new rows.

第 7 章会完整展开这个分层。

## 动手练习

1. **看你机器上真实的组合树。** `dsh` 启动器自带 dump 功能（`apps/cli/src/args.ts` 的 `--dump-config` flag）：

   ```sh
   pnpm dsh --profile headless --dump-config
   ```

   它打印组合后的完整插件树然后退出（不启动 Agent）。再加一个 `--dump-default-config` 对比，看用户层和 `--patch` 层被排除后的 bundle 纯层。

2. **给每行标角色。** 打开 `packages/bundle/base/cordis.patch.yml`，给每个条目标注它在能力接缝中的角色：Definition / Provider / Consumer / 纯插件（不占接缝，如 `session-checkpoint-policy`）。做完和上面的解剖对照。你会发现绝大多数行是 Provider 或 Consumer，但 Definition 也都在这份文件里（`llm`、`session`、`tools`、`subagent`……）——产品把共享脊柱整个收进了 base。
3. **追踪一个 `!!js`。** 在 headless bundle 里找到 `mode: !!js process.env.DSH_TOOLS_MODE`，回答：这个表达式什么时候求值、在哪个上下文里求值？（提示：读 `docs/cordis-primer.md` 的 Loader Configuration 一节，再对照 `docs/cordis-tutorial/05-config.md`。）
4. **故意改坏它。** 写一个 overlay 把 `llm-deepseek` 行的 `name` 改成一个不存在的包名，重跑 headless（第 2 章的命令）并 `--dump-config`，观察报错方式；再把某行的 `id` 改成和另一行重复，看 loader 怎么说。体会「配置错误要尽早炸出来」的设计。

## 延伸阅读

- `packages/bundle/base/cordis.patch.yml` — 本章解剖对象，注释本身就是最好的文档。
- `packages/bundle/README.md` 与各 bundle 的 `README.md` — 每个 bundle 的边界与用法。
- `packages/bundle/headless/cordis.patch.yml` — 一次性执行器层的最小形态。
- `docs/architecture.md` — profile/bundle 分层；进阶篇[第 8 章](../advanced/01-architecture.md)精读。
- 进阶篇[第 11 章](../advanced/04-capability-seam.md)会把三角色模式推广到设计你自己的能力接缝。
- 下一章：[第 5 章 写你的第一个工具](./05-first-tool.md)。
