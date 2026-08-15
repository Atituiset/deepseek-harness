# 第 15 章 测试体系：单测、快照与真实 API

## 本章目标

- 理清五层测试各自证明什么、命令是什么。
- 掌握仓库的测试政策：为什么 mock 只用在边界、为什么 e2e 要「验证世界而非自报」。
- 知道非平凡改动为什么必须在同 PR 带一个免 key 快照。
- 能为自己写的插件选对测试层并跑起来。

权威文档是 `docs/testing.md`，本章是它的导读；命令清单在根 `AGENTS.md`。

## 五层梯队

| 层 | 命令 | 证明什么 |
|---|---|---|
| 单测 | `pnpm run test` | vitest 跑 `tests/**` 下的包测试与 `scripts/**/*.spec.ts` |
| 覆盖率门禁 | `pnpm run test:coverage` | CI 门禁：`packages/*/*/src` 按文件 100% |
| 真实 API e2e | `pnpm run test:e2e` | 对真实 provider API 的带 key 测试 |
| 快照 | `pnpm run test:snapshot` | 免 key 回放录制的 ACP/headless transcript |
| Web 浏览器快照 | `pnpm run test:web` | Chromium 回放对比（Linux PR 门禁） |

逐层展开：

**单测（`pnpm run test`，即 `vitest run`）。** 测试跟代码走，每个注册表都要有 HMR 安全测试（dispose 贡献它的 fiber，断言清理干净）。偏好边界 case、错误路径、事件顺序、并发竞态；契约回归值得写永久测试，范例是 `packages/core/agent-loop/tests/contract-regressions.spec.ts`。

**覆盖率门禁（`pnpm run test:coverage`）。** 注意这是 CI 门禁而不是 `test`——`docs/testing.md:10` 明说了理由：一行没覆盖经常是死代码，门禁在正确地标记它去删除，而不是缺一个补上的测试。行覆盖是必要的，但从不充分——它证明行跑过，不证明功能如发布时那样工作。豁免是按文件显式声明的：`packages/shell/pwsh-local/src` 需要真实 `pwsh`，没有它的机器上套件自跳过且 `vitest.config.ts` 豁免该文件，CI runner 带着 pwsh 执行完整门槛。

**真实 API e2e（`pnpm run test:e2e`）。** 带 key 打真实 provider。每个套件在没有自己的 key 时自动 skip（`DEEPSEEK_API_KEY`、`EXA_API_KEY` 等各自独立），所以无 key 的 CI 保持绿。政策原文值得记住：「inference is cheap here」——无 key 测试只证明管道通，只有带 key 的运行证明 agent 对真实模型工作。最有价值的是 smoke：启动真实 example、发一个 prompt、检查世界。

**快照（`pnpm run test:snapshot`）。** 免 key 的外部行为门禁。ACP 场景启动真实的 automation-server example，回放录制的会话，diff 归一化后的 JSON-RPC 与重新持久化的日志；headless 场景经未导出的 JSONL 测试驱动启动显式的 example 组合。三个动词：

```sh
pnpm run test:snapshot          # 回放对比（免 key）
pnpm run test:snapshot:record   # 模型 transcript 变了，重新录制（需 key）
pnpm run test:snapshot:refresh  # 回放输入仍有效，只刷新期望输出
```

用 `-t <名字>` 过滤单个场景（vitest 的 testNamePattern）。录制的 diff 要逐行 review——快照测试的信任度全靠 review 纪律。

**Web 浏览器快照（`pnpm run test:web`）。** Chromium 回放浏览器输出并对比 `apps/web/tests/snapshots/`。CI 强制只读 `DSH_SNAPSHOT=replay`，从不写期望输出；录制和 refresh 只在本地做。它先跑 build（插件 CSS 需要构建产物），所以它同时也是一条「构建没坏」的信号。

## 快照的组织方式

场景各有归属（`docs/testing.md:49`）：ACP 自动化场景在 `examples/<name>/tests/snapshots/`（`examples/acp-agent` 是主场景表，经 `dsh-acp-snapshot` 套件工厂驱动）；`examples/headless-agent` 拥有内部 canonical-event JSONL 快照与回放 fixture；交互式终端旅程在 `apps/cli/tests/snapshots/`；浏览器渲染的 web GUI 旅程在 `apps/web/tests/snapshots/`。提交的 fixture 用 canonical packed-row 布局，免 key 门禁靠 `session` header 发现每个 fixture。一个 ACP 场景（`text-turn`）钉住完整系统提示词与工具 schema 内容，其它 fixture 把它 token 化——改一行提示词只搅乱一个文件，这是刻意的审查成本控制。

## 五条政策及其理由

**1. 测试描述行为，而非正确性。** 行为变了就连测试一起改，并在 PR 里解释为什么。「把旧测试改绿」不是作弊的前兆，而是行为变更的应有之义——前提是你说得清新行为为什么对。

**2. 真实实现优先于 mock。** mock 只用在昂贵或不确定的边界（LLM adapter、网络、时钟），下游全用真的。手工替身只能证明「桥会传字节」，证明不了发布的工具如断言般行为。范例：bridge 的工具调用测试用脚本化 mock 模型 + 真实工具和 executor（`makeBridgeHarness({ withBash: true })` 插的是 `dsh-bash-local` 和 `dsh-tool-bash`，跑真的 `echo`）。

**3. 验证世界，而非自报。** e2e 断言要在外部重跑命令、重读文件；对 agent 自己输出的关键词探针会让作弊的 agent 通过。断言未触碰的文件字节级一致。e2e 测试拥有自己的资源：在测试里建 harness，`afterEach` 里 dispose（失败/retry/超时也要）。

**4. 走真实入口路径。** 产品可见的插件必须有非单测的真实组合测试：经 Loader 和 app/process 启动 test-only `cordis.yml`，只 mock 外部服务，断言模型可见请求、持久状态或用户可见输出。且「真实入口」指发布产物：包的 `bin` 要用普通 `node` 跑构建后的 `lib/bin.js`——tsx 会掩盖 settle 竞态、模块解析、被吞的加载失败。

**5. 守卫要真的守得住。** 对没有 `inject` 的组合类插件，一个 Loader smoke 在「默认导出顶替了必需具名导出」时照样是绿的——所以要显式断言 `expect('default' in mod).toBe(false)` 并做 `unwrapExports` 往返，而且要**证明**它：引入回归、看它变红、再还原（`docs/testing.md:34`）。一条从没红过的守卫不是守卫。

## 跑哪些：证据匹配表面

不是每个改动都跑全套。根 `AGENTS.md` 的选择规则是「证据匹配表面」：

- 行为改动 → 相关包的 focused 单测。
- 模型输出或用户可见输出改动 → 快照。
- 文档改动 → `pnpm run doc-sync`。
- 发布路径改动 → build/hygiene 加 built smokes。
- provider 行为改动 → 真实 API e2e。

不要默认跑全量，也不要为了一次提交重复跑已经绿的检查——穷举覆盖和平台矩阵是 CI 的职责。推送前的最小检查清单见 `.agents/skills/dsh-pre-push-checks/`。

## 两个容易踩的工程约束

**密钥与环境。** 真实 API 测试读根 `.env` 的 `DEEPSEEK_API_KEY`，可选 `DEEPSEEK_BASE_URL` 指向代理或 mock server。没有 key 时 e2e 自跳过——这是特性不是失败。绝不提交凭证。

**测试解析只在源码平面。** 每个 vitest 配置用 vite-tsconfig-paths 指向 `tsconfig.base.json`，裸 workspace 导入解析到 `src`，永不经过 package `exports` 到构建产物 `lib/`——那里的陈旧产物会加载出模块单例的第二份拷贝。构建产物只被显式消费：`lib` 模式的子进程和 built smokes。

**测试子进程的启动模式。** CI 和带构建的测试通道里，example/Cordis 配置子进程一律从构建的 `lib/` 经共享双模 launcher 启动，不要手写 `--import tsx`。而 `dsh` CLI 的**源码启动**走 tsx 的 ESM-only hook（`node --import tsx/esm`）：它能摸到的模块必须保持 ESM，不能有 CJS-only 导出，因为在 engines 范围内 Node 的原生 TypeScript 模式不可用（契约见 `.agents/notes/implemented/architecture/2026-07-29-dsh-source-launch-tsx-esm.md`）。两条规则合起来的记忆法：**测试里跑产物，源码启动走 tsx，互不混用。**

## 一次改动的测试计划模板

给非平凡改动写测试计划时，照这个顺序自问：

1. **行为层**：改动的逻辑有 focused 单测吗？错误路径、事件顺序、并发竞态各覆盖了吗？
2. **组合层**：这是产品可见插件吗？是则要有经 Loader 启动 test-only `cordis.yml` 的真实组合测试。
3. **transcript 层**：改动模型可见、协议可见或人类可见输出吗？是则同 PR 必须有免 key 快照场景（见下节）。
4. **provider 层**：碰了真实 provider 行为吗？是则补带 key 的 e2e smoke。
5. **发布层**：动了 bin 或构建产物路径吗？是则确认 built smokes 覆盖。

五问都能在动手前回答——`docs/testing.md:49` 要求「计划阶段点名每一层覆盖，并在实现前验证 harness 能表达它」。

## 本地与 CI 的分工

仓库对「谁跑什么」有明确分工，理解它能省下大量等待：

- **本地**：只跑与你改动表面匹配的最小检查（见上节「证据匹配表面」）。推送前过一遍 `.agents/skills/dsh-pre-push-checks/` 的选择流程。
- **CI**：拥有穷举覆盖和平台矩阵——全量单测、覆盖率门禁、快照、web 快照、跨平台。不要为了「保险」在本地预演整个 CI。
- **e2e**：有 key 的本地环境和有 secret 的 CI 都跑；无 key 的环境自动跳过且保持绿。跳过不是失败信号，是设计好的行为。
- **重复跑已绿的检查没有价值**：测试的信任来自「它在改动时变红」，不来自跑的次数。

一个反直觉但重要的纪律：如果你怀疑某个测试其实守不住它声称的东西（比如换了实现它也不红），**先引入回归看它变红再还原**——这是仓库对「守卫有效性」的标准验证法（`docs/testing.md:34`）。

## 读一份快照 fixture

快照 fixture 是本书最好的「真实 transcript」语料，读法：

- 文件开头是 `session` header——免 key 门禁靠它发现 fixture，里面有 `version`（就是第 10 章的 `SESSION_FORMAT_VERSION`）。
- 之后每行一个 packed 的事件行，事件类型对照第 10 章的 `SessionEventMap`。
- ACP 场景另有归一化后的 JSON-RPC 期望输出：归一化抹掉时间戳、id 这类非确定字段，剩下的是协议契约。
- headless 场景是 canonical-event JSONL：一个事件一行，可以直接用第 9 章的流水线对照着走一遍。

想亲眼看「行为变更如何体现为快照 diff」：改一句 headless persona（`examples/headless-agent/cordis.yml` 的 `persona`），`pnpm run test:snapshot:refresh`，然后 `git diff` 看哪些 fixture 动了——这就是评审者审查一次模型可见改动时看到的东西。

## 常见问题

- **没有 API key 能干什么？** 单测、快照、web 快照全都可以；e2e 自跳过。第 13 章的 mock server 练习也是免 key 的。
- **覆盖率红了先干什么？** 先问那行是不是死代码——门禁的设计意图就是先删再补测（`docs/testing.md:10`）。
- **快照 diff 很大怎么办？** 那是给评审看的改动证据，不是噪音；逐行 review，确认每一处变化都能对应到你的意图。对应不上的就是意外行为变更。
- **测试在本地绿、CI 红？** 先查平台差异（pwsh 是否存在、快照录制环境），再查是不是走了源码/产物两条不同路径——参见「测试子进程的启动模式」。
- **能不能为图省事只加单测？** 模型或用户可见的改动不能——见下节，快照是同 PR 的硬性要求。

## 什么时候必须加快照

`docs/testing.md:47-49` 的规则没有例外口：**每个非平凡的模型可见、协议可见或人类可见的改动，都要在同一个 PR 里经一个可运行 example 的所属快照套件新增或更新一个免 key 场景。** 包测试、e2e 断言、mock/test-only 组合、PR 论述都不能替代装配后的 transcript；harness 表达不了你的场景时，在同一个改动里扩展 harness。规划能力接缝、生命周期变体或 transcript 表面时，要在计划阶段就点名每一层覆盖，并在实现前验证 harness 能表达它。

这条规则的直觉解释：单测证明零件对，快照证明整机说人话。Agent 产品的回归大多数是「零件全绿、整机说胡话」，只有装配后的 transcript 抓得住。

## 动手练习

**练习 1：跑快照全量。** 先 `pnpm install && pnpm run build`，然后 `pnpm run test:snapshot`——不需要任何 API key。观察输出里 ACP 与 headless 场景的列表，挑一个名字用 `pnpm run test:snapshot -- -t <名字>` 单跑，然后打开它对应的 fixture 与期望输出文件（`examples/acp-agent/tests/snapshots/` 或 `examples/headless-agent` 的快照目录），对照第 10 章的事件词汇读懂录制的 JSONL。

**练习 2：跑一个你读过的包的单测。** 本书精读过 agent-loop，跑：

```sh
pnpm run test -- packages/core/agent-loop
```

（`pnpm run test` 即 `vitest run`，位置参数是 vitest 的文件路径过滤。）再试 `-t` 按测试名过滤：`pnpm run test -- packages/core/agent-loop -t "turn"`。打开 `packages/core/agent-loop/tests/` 找一个你能在源码里指出对应行为的测试读一遍。

**练习 3：给第 12 章的计时插件选测试层。** 回到你写的 `tools/execute` around 插件：它该有哪几层测试？写出计划——单测断言什么（耗时被记录）、真实组合测试断言什么（经 Loader 启动后日志行出现）、需不需要快照（它改变模型或用户可见输出吗？）。按 `docs/testing.md:49` 的标准回答最后一个问题。

## 延伸阅读

- `docs/testing.md` — 测试政策原文，每层都有 Agent Note 链接承载理由。
- 根 `AGENTS.md` 的 Commands 一节 — 全部命令与各自的用途边界。
- `examples/AGENTS.md` — 每个 example 的免 key 与带 key smoke 约定。
- `packages/core/agent-loop/tests/contract-regressions.spec.ts` — 契约回归测试的范例。
- 上一章 [第 14 章 子代理与工作流](./07-subagent-workflow.md)；回到 [第 8 章 架构总览](./01-architecture.md) 复习全图。
