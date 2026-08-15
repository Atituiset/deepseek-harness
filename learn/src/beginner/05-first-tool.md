# 第 5 章 写你的第一个工具

## 本章目标

- 用 `defineTool` 写一个模型可调用的工具，并在真实 Agent 里跑通。
- 理解 `execute` 合约的四条核心规则：参数已校验、返回 canonical JSON、遵守 `exec.signal`、渲染与值分离。
- 理解 UI render intent 的设计：工具的卡片呈现是纯函数，与模型可见内容分开。

蓝本是 `docs/user/develop/basic/tool.md`（教程）和 `docs/cookbook/adding-a-tool.md`（合约参考）。本章把它们的内容串成一条线，细节以这两份文档为准。

## 最小工具长什么样

以下是 `docs/user/develop/basic/tool.md` 里的完整示例——一个 `greet` 工具：

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))
}
```

注意它就是一个普通 Cordis 插件（第 3 章的函数形态）：`inject = ['tools']` 声明等工具注册表就绪，`ctx.tools.register(...)` 把工具注册进去。注册是 effect——插件卸载时工具自动注销。`defineTool` 定义在 `packages/core/tools/src/schema.ts:545`，注册表服务 `ToolRuntime` 在 `packages/core/tools/src/index.ts:787`。

## 四个部分各自的合约

### parameters：类型从 schema 推导

`parameters` 用仓库自己的 `ParameterSchemaSpec` DSL 声明。关键事实：**`execute` 收到的 `args` 是从 schema 推导出的 TypeScript 类型**（`InferArgs`），上面的例子里是 `{ name: string }`。改 schema，args 类型跟着变；写错字段名，编译器当场报错。

### execute：进来的是已校验参数，出去的是 canonical JSON

`docs/cookbook/adding-a-tool.md` 的「Rules of the execute() contract」是权威清单，入门阶段先记住四条：

1. **参数已校验。** `defineTool` 在 `execute` 运行前就把模型生成的 JSON 按 schema 校验过了——类型、required 键、字面量约束、嵌套结构。`execute` 内部不用再做防御性检查，只需检查 DSL 表达不了的约束（比如「非空字符串」「正数」）。
2. **返回一个 canonical JSON 值。** `output.schema` 声明返回值的结构（对象、数组、标量、null 根都可以），`execute` 只返回这个值本身。不要在返回值里拼给模型看的散文，也不要让调用方从文本里解析 id。
3. **抛异常意味着 `isError`。** 基础设施失败（文件不存在、网络断了）就 throw，注册表会捕获并包装成错误结果。但「领域上成功、结果不理想」的情况（比如命令退出码非零）应该用 canonical 值正常返回，由渲染层解释。
4. **遵守 `exec.signal`。** `execute(args, exec)` 的第二个参数携带 `AbortSignal`；长操作（网络请求、子进程）必须把它传下去，这样取消和超时才能真正中断工作。

### output.schema 与 output.render：值和呈现的分离

这是最容易被初学者忽略、却最能体现设计水平的一点。工具的输出走两条路：

- `execute` 返回的 **canonical 值** 是程序可消费的真值：注册表把它快照为无损 JSON、校验、冻结。Code Mode 下其他代码调用你的工具时拿到的就是这个值。
- `output.render(args, value)` 把这个值转成**模型可见的内容块**（`ContentBlock[]`）。

分开的好处：canonical 值可以是干净的结构化数据（`{ exitCode: 1, stdout: "..." }`），而给模型看的渲染可以是带解释的散文。模型看到的是渲染结果；程序拿到的是结构。设计 `output.schema` 时要把它当作一个程序 API 来设计。

### UI render intent：卡片呈现是纯函数

工具在 UI 里的卡片是第三个独立维度。`docs/cookbook/adding-a-tool.md` 的「How your tool renders in a UI」一节定义了可选的 `presentCall(args)` / `presentResult(...)` 方法，返回带 `card` 标签的 render intent：

- `{ card: 'generic', ... }` — 默认卡片，可带 `kind` 图标和 `locations` 文件定位。
- `{ card: 'terminal', ... }` — 这个调用就是一条 shell 命令（tool-bash 用的就是它）。
- `{ card: 'diff', ... }` — 这个调用创建或修改文件（tool-fs 的 write/edit）。
- 还有 `search`、`web` 等完成态卡片。

两条硬规则：

1. **必须是纯函数。** 这些方法既在实时流式时运行，也在会话日志**回放**时运行，所以只能是 `args`（加结果）的纯函数——不许 I/O、不许读会话状态、不许用时钟或随机数。
2. **UI 格式不进模型结果。** 为了给卡片好看而往 canonical 值或模型内容里塞 diff、console 代码块，都是违规。`output.render` 管模型可见散文，卡片方法管 UI 状态，互不越界。

不写这两个方法的工具会落到 generic 卡片（标题 = 工具名，原始参数作输入），功能完全正常——卡片是增强，不是义务。入门阶段先不写，进阶篇[第 12 章](../advanced/05-tool-pipeline.md)再回来。

## 动手练习

练习沿用 `docs/user/develop/basic/tool.md` 的 scratch-plugin 流程（它也是第 6 章练习的底座，别删）。

1. **创建插件目录。** 从仓库根：

   ```sh
   mkdir -p scratch-plugin/src
   ```

2. **写工具插件。** 创建 `scratch-plugin/src/my-plugin.ts`，写一个 `word_count` 工具：接收 `text`（required string），返回 `{ characters, words, lines }`。照 `greet` 的结构写，注意 `output.schema` 要声明对象根：

   ```ts
   output: {
     schema: {
       type: 'object',
       properties: {
         characters: { type: 'number' },
         words: { type: 'number' },
         lines: { type: 'number' },
       },
     },
     render: (_args, value) => [{
       type: 'text',
       text: `${value.characters} characters, ${value.words} words, ${value.lines} lines`,
     }],
   },
   ```

   （schema DSL 的完整字段见 `docs/cookbook/adding-a-tool.md`；对象节点需要显式声明。）

3. **写 overlay 并启动。** 创建 `scratch-plugin/cordis.yml`（插件路径必须是绝对路径，用 `pwd` 的输出替换）：

   ```yaml
   - insert:
       - id: word-count
         name: '/绝对路径/deepseek-harness/scratch-plugin/src/my-plugin.ts'
   ```

   然后启动 Web UI（首次需要 `pnpm run build`，见第 2 章）：

   ```sh
   pnpm dsh web --patch ./scratch-plugin/cordis.yml
   ```

   打开 `http://127.0.0.1:3080`，对它说：「用 word_count 统计这段文字：……」。观察模型决定调用工具、传入参数、拿到结果的完整过程。

4. **（可选）headless 验证。** 同一个 overlay 也可以加在 headless 上：

   ```sh
   pnpm dsh --profile headless --patch ./scratch-plugin/cordis.yml "用 word_count 工具统计 hello world 这个词组"
   ```

5. **触发错误路径。** 让模型传一个不符合 schema 的调用（比如要求它「不传参数调用 word_count」），观察校验错误如何作为工具结果反馈给模型，而不是让进程崩溃。

## 延伸阅读

- `docs/cookbook/adding-a-tool.md` — 工具合约的完整参考：嵌套 schema、后台任务、策略钩子、Code Mode、UI 卡片。
- `docs/user/develop/basic/tool.md` — 官方教程版本，下一步指向插件配置教程。
- `packages/core/tools/README.md` — 工具注册表的扩展点定义（pre-execute / execute / post-execute / result）。
- `packages/shell/tool-bash/` — 生产级的三包工具示例（Definition / Provider / Consumer 完整接缝）。
- 进阶篇[第 12 章](../advanced/05-tool-pipeline.md)精读工具执行管线：从 `tool/call` 事件到 `tool/result` 事件的完整链路。
- 下一章：[第 6 章 写你的第一个插件](./06-first-plugin.md)。
