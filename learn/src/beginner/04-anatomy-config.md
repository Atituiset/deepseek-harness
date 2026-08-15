# 第 4 章 解剖一个真实 Agent 配置

## 本章目标

- 逐段读懂 `examples/headless-agent/cordis.yml`——一份 165 行、注释齐全的完整编码 Agent 配置。
- 能用能力接缝的三角色（Definition / Provider / Consumer）标注配置里的每一行。
- 理解 cordis.yml 的两条寻址规则：行按 `id` 寻址；`!!js` 只允许出现在 `config` 和 `disabled` 字段。
- 看懂「spine 留给 leaf 什么」这一分层思想。

## 先建立读法

一份 `cordis.yml` 是一个**插件条目列表**。每个条目最少两个字段：

```yaml
- id: bash                              # 行的名字，供 patch 寻址
  name: '@deepseek-ai/dsh-bash-local'   # 插件模块：npm 包名或路径
  config:                               # 可选：传给插件的经过校验的配置
    timeoutMs: 60000
```

三条规则（第 3 章和 `docs/cordis-primer.md` 都有出处）：

1. **行顺序无加载语义。** 启动顺序由 `inject` 依赖图驱动，文件里的排列只是给读者看的分组。`packages/bundle/base/cordis.patch.yml` 头部注释明说："Row order carries no load semantics"。
2. **`id` 是行的地址。** 后续的 patch 层通过 `id` 引用某一行、整体替换它的 `config`，或插入新行。第 7 章的组合覆盖全靠这个机制。
3. **`!!js` 是受限的逃生舱。** `@deepseek-ai/cordis-plugin-include` 会把 `!!js` 解析成表达式节点，但只允许出现在条目的 `config` 内部和 `disabled` 字段；其他元数据（`id`、`name`）保持字面量。想按环境条件换插件，用 overlay，不要在配置里写分支。

## 逐段解剖 headless-agent

打开 `examples/headless-agent/cordis.yml`（165 行），跟着下面的分组读。每段标题是功能域，括号里是该行在能力接缝中的角色。

### 用户面：settings 与 credentials（Provider）

```yaml
- id: settings
  name: '@deepseek-ai/dsh-settings-file'

- id: credentials
  name: '@deepseek-ai/dsh-credentials-local'
```

文件头注释点明了分工：「entry configs here are the composition base, while user-plane values resolve per request through the two providers below」。`settings` 读 `$DSH_HOME/settings.yaml`（热重载），`credentials` 管理 `$DSH_HOME/.credentials.yaml`。注意第 2 章的 `DEEPSEEK_API_KEY` 就是 `credentials` 在**每次请求时**解析的——所以配置里找不到任何 key 的字面量。这两个都是用户态能力的 Provider。

### LLM：一个适配器（Provider）

```yaml
- id: llm-deepseek
  name: '@deepseek-ai/dsh-llm-deepseek'
  config:
    thinking: enabled
    reasoningEffort: max
    models:
      - id: deepseek-v4-pro
        contextWindow: 128000
      - id: deepseek-v4-flash
        contextWindow: 128000
```

这是 LLM 接缝的 Provider：向 `ctx.llm` 注册 DeepSeek 适配器。注释提示了可替换性——换成 `@deepseek-ai/dsh-llm-pi-ai` 就是另一套后端。注意这里没有 LLM 的 Definition 行：Definition（`dsh-llm`，占据 `ctx.llm`）在下一节的 spine 内部。

### 执行底座：subprocess 与 bash（Provider）

```yaml
- id: subprocess
  name: '@deepseek-ai/dsh-subprocess-local'

- id: bash
  name: '@deepseek-ai/dsh-bash-local'
  config:
    timeoutMs: 60000
```

`subprocess` 管理子进程组（spawn/kill/输出管道），`bash-local` 是 shell 接缝的 Provider——真正的 bash 执行器。模型可见的 `bash` 工具（Consumer）不在这里，它在 spine 里。

### agent-spine：一行顶半个 Agent

```yaml
- id: agent-spine
  name: '@deepseek-ai/dsh-agent-spine-demo'
  config:
    agents:
      - id: main
        provider: deepseek-official
        model: deepseek-v4-flash
        cwd: !!js process.cwd()
    workspaceContext:
      maxBytes: 65536
    persona: |
      You are headless-agent, a coding assistant powered by the {{model}} model.
      ...
```

这是全文件信息量最大的一行。`dsh-agent-spine-demo` 是一个 **bundle 插件**：一个插件在代码里挂载一整套子插件树。打开 `packages/examples/agent-spine-demo/README.md` 的「The tree it loads」，它挂载的包括：`dsh-llm`（抽象 LLM 服务，Definition）、`dsh-session`（事件溯源日志）、`dsh-system-prompt`、`dsh-tools`（工具注册表）、`dsh-agent`、`dsh-agent-loop`（主循环本体）、`dsh-tool-bash`（bash 工具的 Consumer 部分）等十几个包。

这就解释了为什么这个文件只有 165 行：**Agent 的「脊柱」被收进了一个 bundle，leaf 配置只需提供脊柱刻意留空的东西**。README 的「What it deliberately leaves OUTSIDE the bundle」列得很清楚：

- **LLM 适配器** — spine 只有抽象的 `ctx.llm`，leaf 提供具体的 `llm-deepseek`。
- **bash 执行器** — spine 只有工具 schema（Consumer），leaf 提供 `ctx.shell` 的实现 `bash-local`。
- **入口与每应用基础设施** — 传输、stdout、重载策略由 app 包决定。

这正是能力接缝思想在**组合层面**的应用：bundle 拥有共享脊柱，leaf 拥有可换后端，app 包拥有入口。

`cwd: !!js process.cwd()` 是 `!!js` 的标准用法：配置在激活时对插件上下文求值。

### 持久化与压缩

```yaml
- id: persistence
  name: '@deepseek-ai/dsh-session-persistence-jsonl'
  config:
    root: './.sessions'
    compression: !!js "process.env.DSH_SNAPSHOT === undefined ? 'zstd' : 'none'"

- id: checkpoint-policy
  name: '@deepseek-ai/dsh-session-checkpoint-policy'

- id: token-meter
  name: '@deepseek-ai/dsh-token-meter'

- id: compaction-basic
  name: '@deepseek-ai/dsh-compaction-basic'
  config:
    thresholdRatio: 0.8
    ...
```

`persistence` 是会话持久化的 Provider（JSONL 落盘）；它的 `compression` 行展示了 `!!js` 的另一个典型用法——快照录制时关闭压缩，让录制文件可读。`compaction-basic` 是压缩接缝的 Provider：历史接近上下文窗口时摘要旧内容。`token-meter` 统计 token 用量，是压缩决策的输入。

### 子代理与工作流

`session-projection`、`subagent`、`subagent-spawn-in-process`、`subagent-fork-in-process` 四行搭起子代理能力：projection 提供持久身份，`dsh-subagent` 是 provider 注册表（Definition），两个 in-process 行是 `spawn` 和 `fork` 两个 Provider。随后的 `tool-subagent` 和 `tool-subagent-fork` 是 Consumer——把两个 provider 分别以 `subagent`、`subagent_fork` 工具名暴露给模型。同一个 Consumer 包被配置了两次，用不同的 `id` 区分行。

`workflow-worker-thread` + `tool-workflow` 是同样的接缝：worker 线程引擎（Provider）+ 模型可见的 `workflow` 工具（Consumer）。

### 剩余工具行

`tool-todo`（todo_write）、`fs-local`（文件系统 Provider）+ `fs-observation-policy`（写入前必须先观察过文件的策略插件）+ `tool-fs`（文件工具 Consumer）收尾。注意注释强调的加载顺序依赖："Policy loads before the model-facing filesystem tools"——但这指的依然是 inject 依赖，不是行顺序。

## 对照完整底座

这份文件是**示例组合**，不是产品默认。产品 profile 的第一层是 `packages/bundle/base/cordis.patch.yml`（dsh-base bundle），比这里更大：模型适配器、全套工具、持久化、沙箱与审批策略、settings、credentials、遥测都在里面。`docs/architecture.md` 第 27 行描述了层级顺序：

> Layers apply to an empty entry list in this order: each bundle in the profile's listed order, then the profile's `cordis.patch.yml`, then the home-level one, then any `--patch` overlay. A patch targets a row by id and replaces its whole config, or inserts new rows.

第 7 章会完整展开这个分层。

## 动手练习

1. **看你机器上真实的组合树。** `dsh` 启动器自带 dump 功能（`apps/cli/src/args.ts` 的 `--dump-config` flag）：

   ```sh
   pnpm dsh --profile headless --dump-config
   ```

   它打印组合后的完整插件树然后退出（不启动 Agent）。再加一个 `--dump-default-config` 对比，看用户层和 `--patch` 层被排除后的 bundle 纯层。

2. **给每行标角色。** 打开 `examples/headless-agent/cordis.yml`，给每个条目标注它在能力接缝中的角色：Definition / Provider / Consumer / 纯插件（不占接缝，如 `checkpoint-policy`）。做完和上面的解剖对照。你会发现绝大多数行是 Provider 或 Consumer——Definition 几乎都收在 spine 里。
3. **追踪一个 `!!js`。** 找到 `cwd: !!js process.cwd()`，回答：这个表达式什么时候求值、在哪个上下文里求值？（提示：读 `docs/cordis-primer.md` 的 Loader Configuration 一节，再对照 `docs/cordis-tutorial/05-config.md`。）
4. **故意改坏它。** 把 `llm-deepseek` 行的 `name` 改成一个不存在的包名，重跑 headless（第 2 章的命令），观察报错方式；再把某行的 `id` 改成和另一行重复，看 loader 怎么说。体会「配置错误要尽早炸出来」的设计。

## 延伸阅读

- `examples/headless-agent/cordis.yml` — 本章解剖对象，注释本身就是最好的文档。
- `packages/examples/agent-spine-demo/README.md` — spine 挂载的完整子树和它留给 leaf 的清单。
- `packages/bundle/base/cordis.patch.yml` — 产品级 base bundle，注释解释了 patch 语义。
- `docs/architecture.md` — profile/bundle 分层；进阶篇[第 8 章](../advanced/01-architecture.md)精读。
- 进阶篇[第 11 章](../advanced/04-capability-seam.md)会把三角色模式推广到设计你自己的能力接缝。
- 下一章：[第 5 章 写你的第一个工具](./05-first-tool.md)。
