# 第 13 章 写一个 LLM 适配器

## 本章目标

- 掌握 `LlmAdapter.stream()` 合约与 `StreamChunk` 协议义务。
- 以 `llm-deepseek` 为参考布局，理解 wire 类型、序列化、传输、chunk 翻译、adapter 类的五文件分工。
- 学会用稳定错误码归一 provider 错误，并把密钥交给凭证接缝。
- 动手写一个免 key 的 echo adapter，跑通一个完整 turn。

蓝本是 `docs/cookbook/adding-an-llm-adapter.md`，参考实现是 `packages/llm/llm-deepseek/`。本章把它们连成一条可照做的路径。

## 接缝位置

LLM 是第 11 章讲过的能力接缝：Definition 是 `packages/llm/llm` 的 `LlmRuntime`（`packages/llm/llm/src/index.ts:284`），你的适配器是 provider，agent-loop 是 consumer。适配器基类：

```ts
// packages/llm/llm/src/index.ts:232（LlmAdapter 的抽象方法）
abstract stream(options: GenerateOptions): AsyncIterable<StreamChunk>
```

一个适配器插件的最小形状（cookbook 原文）：

```ts
class MyAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> { /* … */ }
}

export const name = 'llm-myprovider'
export const inject = ['llm']
export const Config: z<Config> = z.object({ apiKeyEnv: z.string(), /* … */ })

export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(/* … */))
}
```

注册纪律（`registerAdapter`，`packages/llm/llm/src/index.ts:338`）：一条 provider 路由一个 adapter，重复注册抛错；一次注册多条路由是全有或全无；注册是 effect，插件卸载即摘除，热替换成立。`options.provider` 选 adapter，`options.model` 是 provider 侧的模型 id——所以动态目录型 adapter 可以不重配生命周期就服务新模型。

## StreamChunk 协议义务

流的词汇在 `packages/llm/llm/src/types.ts:291-299`：

```ts
export type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: CallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | { type: 'finish'; /* … */ }
```

cookbook（`docs/cookbook/adding-an-llm-adapter.md:25-35`）把义务钉成了清单，逐条都有后果：

- **`usage` 必须先于 `finish`；`finish` 之后什么都不许再发。** 稳妥做法：把 finish/usage 缓冲到 provider 的流结束标记再一起 flush（有的 provider 会在末尾发只有 usage 的 chunk）。
- **工具调用的 `arguments` 全程是原始 JSON 字符串**，流式片段用 `argumentsDelta` 传。provider 如果给你解析好的对象，在 `block-end` 处重新 stringify。
- **block 的 `index` 按首次出现的流序分配**，同一 block 的每个 delta 复用同一 index。
- **错误只有两条合法出路**：从 `stream()` 抛出（传输与协议失败，用带稳定 code 的 `LlmError`），或以 `finish { kind: 'error' | 'aborted' }` 结束流（provider 的带内失败）。按失败类别二选一并在文档里写明。
- **尊重 `options.signal`**：传给 fetch 或你的 SDK。
- **provider 不支持的 `GenerateOptions` 字段**（比如没有 stop sequence 的 provider 收到 `stop`）：抛 `LlmError(..., 'UNSUPPORTED')`，不许悄悄丢弃。
- provider 需要响应 id / 签名等原生元数据做续接的，把最小无损 JSON 投影放进 `finish.replayState`，重建历史时校验它。

消费侧不需要你关心，但值得知道：agent-loop 把每个 chunk 落盘为 `assistant/chunk` 并喂给 `BlockAssembler`（`packages/llm/llm/src/assembler.ts:36`）——它对「只有 delta 没有 block-start/end」的协议是宽容的，对已关闭 block 的迟到 delta 直接忽略（:32-34 注释）。你的适配器越守规矩，装配器越无感。

## 参考布局：llm-deepseek 的五文件分工

`packages/llm/llm-deepseek/src/` 演示了 cookbook 推荐的职责拆分（`docs/cookbook/adding-an-llm-adapter.md:37-39`）：

| 文件 | 行数 | 职责 |
|---|---|---|
| `types.ts` | 152 | wire 类型：DeepSeek API 的请求/响应/SSE 载荷 |
| `serialize.ts` | 187 | 请求序列化：harness 消息词汇 → wire 格式 |
| `sse.ts` | 40 | 传输解析：SSE 帧解码 |
| `translate.ts` | 185 | chunk 翻译：wire 事件 → `StreamChunk` |
| `adapter.ts` | 346 | adapter 类：HTTP 调用、错误归一、`stream()` 主流程 |
| `index.ts` | 276 | 插件入口：Config、凭证解析、注册 |

看 `adapter.ts` 里错误归一的做法（:139-144）：HTTP 状态映射到稳定 code——401/403 → `'AUTH'`，配额特征 → `QUOTA_EXCEEDED_CODE`，429 → `'RATE_LIMIT'`，上下文窗口特征 → `CONTEXT_WINDOW_EXCEEDED_CODE`。这些 code 是给策略层（重试、熔断、UI 提示）用的，所以必须稳定，不能把 provider 的原话直接冒泡上去。

## Config、密钥与重试

三条纪律，全部有实证：

**1. Config 经 schemastery 校验。** `export const Config: z<Config> = z.object({...})`（`llm-deepseek/src/index.ts:91`），cordis.yml 里的配置在加载期就被 schema 拦住，不存在「运行到一半发现配置错了」。

**2. 密钥走凭证接缝，绝不写字面 key。** llm-deepseek 的 Config 里没有 `apiKey` 字段，只有 `apiKeyEnv`（:64，schema 里带 `role('credential-ref')`，:92）；每次请求时经 `credentialRef` + 可选的 `ctx.credentials` 服务解析（:228-242），解析失败时报错并指出该把凭证存到哪。仓库里任何代码都不该出现字面 API key。

**3. `retryPolicy` 属于 provider 配置，由独立插件执行。** adapter 只声明重试策略（`llm-deepseek/src/index.ts:80` 的 `retryPolicy?: RetryPolicyConfig`），真正执行重试的是 `packages/llm/llm-retry`：它监听 `agent/request-error` waterfall（`llm-retry/src/index.ts:210`），按策略返回 `{ kind: 'retry' }` 让 agent-loop 的 step 循环再来一圈（回到第 9 章 `agent.ts:367-370`），并把调度事实落盘为 `llm/retry` 事件。把 `retryPolicy` 放在 llm-retry 自己的 Config 里会直接抛错（`llm-retry/src/index.ts:32-34`）——策略归 provider，执行归策略插件，这是故意的分层。

## 模型元数据：resolveModel

provider 特有的思考模式开关留在 adapter 自己的 Config 里；而精确的模型元数据走一条 provider 中立的接缝：实现 `LlmAdapter.resolveModel()`，返回 provider/model 身份加上可选的 `context`（上下文窗口等）和 `reasoning` 字段，仅在确有默认时声明 `defaultEffort`，并尊重可选的 `AbortSignal`（cookbook :35）。推理档位（reasoning effort）是有序的不透明 id，由 adapter 映射到 provider 请求——adapter 保留权威的可选列表（包括自己定义的 `off`），不暴露最终 wire 拼写，也不削掉不支持的值。agent-loop 的 `prepareCall` 绑定的「adapter 精确默认值」（第 9 章的 `adapterDefaults`）就从这里来。

## 适配器验收清单

写完适配器、跑练习之前，对照这张清单（全部来自两个已验证实现的共同义务）：

- `usage` 在 `finish` 之前发出；`finish` 之后流里没有任何东西。
- 工具参数全程是原始 JSON 字符串；provider 给了对象就在 `block-end` 重新 stringify。
- block `index` 按首见顺序分配并复用。
- 传输/协议失败抛带稳定 code 的 `LlmError`；provider 带内失败以 `finish { kind: 'error' | 'aborted' }` 收尾。二选一，不混用。
- `options.signal` 传进了 fetch/SDK；abort 时归一为 `ABORTED`。
- 不支持的 `GenerateOptions` 字段抛 `UNSUPPORTED`，不静默丢弃。
- 密钥只出现为 credential 引用，经 `ctx.credentials` 解析；Config 里没有字面 key 的位置。
- 需要原生元数据续接的，`finish.replayState` 只放最小无损 JSON 投影，重建时校验。

## 动手练习

**练习 1：写一个 echo adapter。** 不联网，把最后一条用户消息回显为流式 chunk，用来免 key 跑通完整 turn：

```ts
import { LlmAdapter } from '@deepseek-ai/dsh-llm'
import type { GenerateOptions, StreamChunk } from '@deepseek-ai/dsh-llm'

class EchoAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> {
    const last = options.messages.at(-1)
    const text = JSON.stringify(last?.content ?? '(empty)')
    yield { type: 'block-start', index: 0, blockType: 'text' }
    // 按小块吐 delta，模拟流式
    for (let i = 0; i < text.length; i += 16) {
      options.signal.throwIfAborted()
      yield { type: 'text-delta', index: 0, text: text.slice(i, i + 16) }
    }
    yield { type: 'block-end', index: 0, block: { type: 'text', text } }
    yield { type: 'usage', usage: { inputTokens: 0, outputTokens: 0 } }
    yield { type: 'finish', kind: 'stop' }
  }
}

export const name = 'llm-echo'
export const inject = ['llm']
export function apply(ctx: Context) {
  ctx.llm.registerAdapter(['echo'], new EchoAdapter())
}
```

（`usage`/`finish` 字段的确切形状以 `packages/llm/llm/src/types.ts` 为准，照着改。）把插件挂进组合、agent 的 `provider` 配成 `echo`，跑一个 headless 任务：会话日志里应出现完整的 `assistant/chunk*` → `assistant/message` → `turn/end { kind: 'completed' }`，全程无网络。注意 chunk 顺序：usage 先于 finish——故意交换它们，看看哪里炸。

**练习 2：用 mock server 测真 adapter。** 仓库自带可编程的 OpenAI 兼容 mock server（`packages/test-support/llm-mock-server`），免 key 演练真实 adapter 与恢复策略：

```sh
pnpm run mock:llm -- --port 8000 --api-key mock-key --sequence partial_disconnect,success
DEEPSEEK_BASE_URL=http://127.0.0.1:8000/v1 DEEPSEEK_API_KEY=mock-key \
  pnpm dsh --profile headless "test provider recovery"
```

观察第一次（连接中断）如何经 `agent/request-error` 和 llm-retry 变成第二次成功的请求，日志里应出现 `llm/retry`。

**练习 3：归一一种错误。** 给你的 echo adapter 加一条规则：当 prompt 含特定关键词时抛 `LlmError('...', 'RATE_LIMIT')`。配上 llm-retry 和很小的重试次数，观察 `llm/retry` 事件与最终 `turn/end { kind: 'error' }` 里的 code 是否原样保留。

## 延伸阅读

- `docs/cookbook/adding-an-llm-adapter.md` — 本章蓝本，协议义务的权威清单。
- `packages/llm/llm/src/types.ts` 的 `StreamChunk` 文档注释 — 两个已验证实现共同确认的协议约定。
- `docs/subsystems/llm-streaming.md` — 流式子系统参考。
- 上一章 [第 12 章 工具执行管线](./05-tool-pipeline.md)；下一章 [第 14 章 子代理与工作流](./07-subagent-workflow.md)。
