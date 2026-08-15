# 第 2 章 环境准备：跑通你的第一个 Agent

## 本章目标

- 装好前置环境，从源码直接启动一个真实 Agent。
- 用一条命令让 Agent 完成一个任务，并观察它的工具调用过程。
- 没有 API key 也能继续学习：掌握快照回放这条替代路径。

## 前置条件

根 `package.json` 的 `engines` 字段写得很明确：

- Node.js `^22.19.0 || >=24.0.0`
- pnpm 11（`packageManager` 字段钉在 `pnpm@11.7.0`）

检查版本：

```sh
node --version   # 需要 v22.19+ 或 v24+
pnpm --version   # 需要 11.x
```

如果 pnpm 版本不对，最简单的办法是用 Corepack 让仓库自己声明的版本生效：

```sh
corepack enable
```

## 安装依赖

```sh
git clone https://github.com/Atituiset/deepseek-harness.git
cd deepseek-harness
pnpm install
```

`pnpm install` 会装好所有 workspace 依赖，并通过 `postinstall` 脚本装好 git hooks。注意这里**不需要** `pnpm run build`：接下来用的 `dsh` 命令直接从源码运行。

## 配置 API Key

在仓库根目录创建 `.env` 文件（它已被 git 忽略，不会进版本控制）：

```sh
DEEPSEEK_API_KEY=sk-你的key
# 可选：自定义接入点
# DEEPSEEK_BASE_URL=https://api.deepseek.com
```

启动时 `apps/cli/src/bin.ts` 会调用 `loadLayeredEnv('dsh')` 把这个文件加载进进程环境。Key 不会出现在任何配置文件里——第 4 章你会看到，配置里只有一个 `credentials` 插件负责在每次请求时解析它。

## 跑通第一个 Agent

根 `package.json` 里的 `dsh` 脚本是：

```json
"dsh": "node --import tsx/esm apps/cli/src/bin.ts"
```

`--import tsx/esm` 让 Node 直接执行 TypeScript 源码，所以下面这条命令跑的就是你目录里的代码，改源码立即生效：

```sh
pnpm dsh --profile headless "用一段话解释这个仓库"
```

拆解这条命令：

- `--profile headless` — 启动名为 `headless` 的 profile：一个一次性执行器，没有服务器、没有 UI，回答完任务就退出。profile 的概念（命名的插件组合）会在第 7 章展开。
- 引号里的字符串是交给 Agent 的任务。`dsh` 启动器只解析自己的 flag（`apps/cli/src/args.ts`），其余参数原样交给启动后的应用。

第一次运行会初始化 harness home 和 profile，然后你会看到 Agent 开始工作：它会真的去读这个仓库的文件来回答你的问题。

预期结果：几十秒内输出一段对仓库的解释，然后进程退出。如果报 API 错误，先检查 `.env` 里的 key 是否生效。

## 观察输出中的工具调用

headless 输出不是只有最终答案。Agent 在回答过程中会调用工具——bash、文件读写等——每次调用及其结果都会出现在输出里。再跑一个明确需要工具的任务：

```sh
pnpm dsh --profile headless "数一数 packages/core 下有几个子目录，列出它们"
```

注意观察：

1. Agent 决定调用 `bash` 工具（而不是凭空回答）。
2. 工具执行的参数和结果被完整打印。
3. Agent 基于工具返回的真实结果组织最终回答。

这个「模型决定 → 工具执行 → 结果回喂 → 模型继续」的循环就是 Agent Loop，进阶篇[第 9 章](../advanced/02-agent-loop-source.md)会精读它的源码。现在你只需建立直觉：**工具调用是这个系统的一等公民，一切对模型的可见内容都被记录在案**。

## 没有 Key 怎么办

两条路，都不需要联网调模型。

**快照回放。** 仓库带了一套免 key 的快照测试，回放预先录制好的真实 transcript：

```sh
pnpm run test:snapshot
```

它用录制文件代替真实 API，跑完整的 ACP/headless 流程并对比预期输出。这是观察「真实 Agent 行为长什么样」的零成本方式，也是仓库测试体系的一部分（进阶篇[第 15 章](../advanced/08-testing.md)）。

**Cordis 教程。** `docs/cordis-tutorial/` 是一套七章的动手教程，全程在临时目录里写自己的插件，不需要任何 key。第 3 章会引导你做它的前六章。

## 顺带认识：Web UI

有 key 之后，也可以启动带界面的版本：

```sh
pnpm run build   # Web 前端需要构建产物
pnpm dsh web
```

浏览器打开 `http://127.0.0.1:3080`。`web` 是 `--profile web` 的硬编码别名（见 `apps/cli/src/args.ts` 第 13 行注释）。第 5、6 章的练习会在 Web UI 里验证你写的工具和插件。

## 动手练习

1. **跑通 headless。** 完成上面的 `pnpm dsh --profile headless "用一段话解释这个仓库"`，确认进程正常退出。
2. **观察工具调用。** 运行「数一数 `packages/core` 下有几个子目录」的任务，在输出中找出每一次工具调用：它用了什么工具、传了什么参数、拿到什么结果。尝试回答：如果 Agent 不调用工具直接回答，会错在哪？
3. **体验快照回放。** 运行 `pnpm run test:snapshot`，确认全部通过。然后挑一个 `packages/` 下名字含 `snapshot` 的测试文件，找到它录制的 transcript，读一遍——那就是一次真实会话的完整记录。
4. **看一眼帮助。** 运行 `pnpm dsh --profile headless --help`（注意：帮助由被启动的应用自己打印，不是启动器），看看 headless 应用还支持哪些参数。

## 延伸阅读

- `README.md` — 仓库首页的 run-from-source 路径。
- `apps/cli/src/args.ts` — `dsh` 命令行的完整解析逻辑，头部注释讲清了 flag 归属规则。
- `docs/user/guide/index.md` — Web UI 使用指南。
- `docs/testing.md` — 测试政策，解释快照测试的地位；进阶篇[第 15 章](../advanced/08-testing.md)展开。
- 下一章：[第 3 章 Cordis 基础：五个核心概念](./03-cordis-basics.md)。
