# dsh-routed-subagent

![License](https://img.shields.io/badge/license-MIT-blue) ![DeepSeek Harness](https://img.shields.io/badge/DeepSeek%20Harness-0.1.5--rc.1-2f6feb) ![CI](https://github.com/Xiao-Hong/dsh-routed-subagent/actions/workflows/ci.yml/badge.svg)

一个面向 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 的全局插件。它提供统一的 `subagent_routed` 工具，让任意会话都可以把一次子代理任务挂载到指定的 agent preset 上，并按调用覆盖模型、provider、工作目录和执行引擎。

> **兼容性基线：DeepSeek Harness `0.1.5-rc.1`。**
>
> 本仓库当前以 `0.1.5-rc.1` 作为适配和端到端验证版本。已验证 preset 挂载、跨 preset 委派、后台任务与 `job_output` 进度、按次模型/provider 覆盖，以及 continuable 续话链路。较早的 rc 版本缺少本插件依赖的接口，不在支持范围内。

## 为什么需要它

DeepSeek Harness 自带的 `subagent` / `subagent_fork` 会让子代理继承父会话的 preset。这样一来，调用方无法把任务交给另一个具有不同 persona、提示词、技能和工具集合的 preset。

本插件注册了一个自定义 subagent provider。创建子代理时，它会异步执行：

```js
await agentPresets.mount(childCtx, targetPreset)
```

因此子代理获得的是目标 preset 的完整 standing composition，而不是对父 preset 做一次 persona 拷贝。插件仍沿用 Harness 官方的一次性子代理生命周期、事件、轨迹和 UI 展示方式。

## 功能概览

- **指定任意 preset**：在当前会话中选择 roster 里的任意 preset，子代理使用该 preset 的身份、提示词段、技能和工具。
- **后台并行执行**：默认立即返回 job id；主会话可以继续工作、并行派发更多任务，再通过 `job_output` 读取结果或用 `job_kill` 停止任务。
- **前台执行**：设置 `run_in_background: false`，等待子代理完成并直接取得最终输出。
- **上下文 fork**：设置 `fork: true`，把当前会话已经完成的轮次作为种子，再叠加目标 preset。
- **可续话子代理**：设置 `continuable: true`，取得持久的 subagent id，之后通过 `send_message(subagentId, ...)` 继续对话。
- **按次模型路由**：`model`、`provider` 和 `max_tokens` 只作用于本次子代理调用。
- **模型可用性预检**：DSH 引擎在派发前校验模型；模型不存在时尽早返回 provider 的候选列表。
- **工具权限收窄**：使用仅支持 deny 的 `tool_filter`，在目标 preset 的工具面上额外移除工具。
- **外部 CLI 引擎**：可把任务交给 Codex、Claude Code 或 CodeBuddy，而不是在 Harness 内创建 preset 子代理。
- **幂等注册**：多个 preset 同时挂载本插件时，host 平面的 provider 不会重复注册。

## 兼容性与运行要求

| 项目 | 要求 |
| --- | --- |
| DeepSeek Harness | **`0.1.5-rc.1`（当前验证基线）** |
| Node.js | `>=22` |
| 安装方式 | GitHub 仓库安装；本插件不发布到 npm |
| DSH 依赖 | 由 Harness 的共享 `node_modules` 提供 `@deepseek-ai/*` 包 |
| 外部引擎 | 仅在使用对应 `engine` 时需要安装其 CLI/SDK |

`package.json` 中的 `peerDependencies` 是包管理元数据；它们的 semver 范围不等于所有 Harness 版本都经过验证。升级 Harness 后，请重新运行语法检查和实际派发验证。

## 安装

### 推荐：通过 dsh CLI 安装

确保 `dsh` 已在 `PATH` 中，然后执行：

```sh
dsh plugin --profile <profile-name> add github:Xiao-Hong/dsh-routed-subagent
```

重启对应的 `dsh web` 或 Harness 进程。插件会通过仓库中的 `cordis.patch.yml` 注册；装配完成后，挂载该 bundle 的会话应能看到 `subagent_routed`。

### 手动挂载（共享依赖不可见时）

插件是纯 ESM 包，没有构建步骤。若插件目录无法解析 Harness 的共享 `@deepseek-ai/*` 依赖：

1. 在插件目录创建指向 Harness 共享依赖的 `node_modules` junction/软链接。

   ```bat
   :: Windows
   mklink /J "<plugin-dir>\node_modules" "<harness>\resources\host\node_modules"
   ```

   ```sh
   # Linux/macOS
   ln -s "<harness>/resources/host/node_modules" "<plugin-dir>/node_modules"
   ```

2. 把插件加入 profile 的 `dsh.profile.bundles`：

   ```json
   {
     "dependencies": {
       "dsh-routed-subagent": "link:<plugin-dir>"
     },
     "dsh": {
       "profile": {
         "bundles": ["...", "dsh-routed-subagent"]
       }
     }
   }
   ```

3. 重启 Harness，并检查启动日志中是否出现 provider 注册信息。

如果部署提供 `dev_install_package(dir=...)` 一类的热装配工具，也可以用它代替手动链接；重启后仍以 profile 的 `bundles` 列表为准。

## 快速开始

### 默认后台 one-shot

```js
await subagent_routed({
  preset: 'dev',
  prompt: '审查当前仓库的错误处理，并给出按优先级排序的改进建议。',
  description: '错误处理审查',
})
```

默认返回后台 job id。随后使用 Harness 的 `job_output` 查看实时快照和最终输出，必要时使用 `job_kill` 停止任务。

### 前台等待结果

```js
await subagent_routed({
  preset: 'researcher',
  prompt: '比较这两个实现方案，并明确推荐其中一个。',
  description: '方案比较',
  run_in_background: false,
})
```

### 继承当前会话上下文并切换 preset

```js
await subagent_routed({
  preset: 'reviewer',
  prompt: '基于上文讨论继续完成代码审查。',
  description: '继续代码审查',
  fork: true,
})
```

`fork` 只带入当前会话已经完成的轮次，不包含正在发起本次工具调用的轮次。若任务需要完整、自包含的上下文，应直接把必要信息写进 `prompt`。

### 创建可续话子代理

```js
const result = await subagent_routed({
  preset: 'dev',
  prompt: '建立一个持续跟踪本仓库测试失败的工作线程。',
  description: '持续测试跟踪',
  continuable: true,
})

// 之后通过 Harness 的 send_message 继续：
// send_message(result.subagentId, '现在检查最新一轮 CI 日志。')
```

`fork: true` 与 `continuable: true` 可以组合使用：子代理先继承已完成的会话轮次，再以指定 preset 持久运行。

## `subagent_routed` 参数

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `prompt` | string，必填 | 子代理要执行的完整任务。普通 one-shot 不共享当前会话上下文，请写成自包含指令。 |
| `preset` | string | DSH 引擎必填，目标 agent preset id；外部引擎没有 preset 概念，会忽略此参数。 |
| `description` | string，必填 | 用于 job/UI 展示的简短任务名称。 |
| `engine` | string | `dsh`（默认）、`codex`、`claude` 或 `codebuddy`。 |
| `model` | string | 本次子代理使用的模型。DSH 引擎走 Harness LLM 路由；外部引擎传给对应 CLI。 |
| `provider` | string | DSH 引擎的模型 provider；省略时继承当前会话的 provider。外部引擎忽略。 |
| `max_tokens` | positive integer | 子代理 LLM 输出上限。 |
| `max_depth` | positive integer | DSH 子代理可继续委派的递归预算；默认值为 3，最小值为 1。外部引擎不使用该字段。 |
| `tool_filter` | `{ deny: string[] }` | 仅支持 deny；在目标 preset 的工具面上移除指定全局工具名，不支持 `allow`。 |
| `run_in_background` | boolean | 默认 `true`，立即返回 job id；设为 `false` 后前台等待最终结果。 |
| `fork` | boolean | 默认 `false`。DSH 引擎设为 `true` 时继承当前会话已完成轮次。 |
| `continuable` | boolean | 默认 `false`。返回可通过 `send_message` 继续的持久 subagent id。 |
| `cwd` | string | 外部引擎的工作目录。代码审查或运行命令时建议显式传入目标仓库路径。DSH 引擎忽略。 |
| `timeout_ms` | positive integer | 外部引擎单轮超时，默认 1,800,000 ms（30 分钟）。DSH 引擎忽略。 |

### 运行模式一览

| 模式 | 设置 | 返回/行为 |
| --- | --- | --- |
| 后台 one-shot | 默认，或 `run_in_background: true` | 立即返回 job id；用 `job_output` 读取，用 `job_kill` 停止。 |
| 前台 one-shot | `run_in_background: false` | 等待子代理完成后返回最终输出。 |
| fork | `fork: true` | 以当前会话已完成轮次为种子，再挂载目标 preset。 |
| continuable | `continuable: true` | 返回持久 subagent id，后续用 `send_message` 发送新任务。 |

后台任务与主会话相互独立：主会话中止不会自动取消子代理；需要停止时请显式调用 `job_kill`。

## 外部引擎

外部引擎由各自的 CLI/SDK 驱动，不使用 DSH preset。它们仍共享 `subagent_routed` 的后台、前台、进度和停止语义。

| 引擎 | 驱动方式 | 模型参数 | 续话方式 | 前置条件 |
| --- | --- | --- | --- | --- |
| `dsh`（默认） | Harness preset-mount provider | `model` + `provider` | DSH continuable | DeepSeek Harness 0.1.5-rc.1 |
| `codex` | `codex app-server --stdio` | `thread/start` 的 `model` | 持久化 thread，同一 `threadId` 继续 | 安装并登录 Codex CLI（`codex login`） |
| `claude` | `@anthropic-ai/claude-agent-sdk` | SDK model | SDK session resume | 安装可选 peer 依赖；可靠续话建议使用官方 Anthropic API |
| `codebuddy` | CodeBuddy CLI `--print` | `--model` | `--session-id` / `--resume` | 安装 CodeBuddy CLI（`codebuddy --version`） |

### Codex 示例

```js
await subagent_routed({
  engine: 'codex',
  cwd: 'E:/projects/my-repo',
  model: 'gpt-5.6-sol',
  prompt: '检查这个仓库最近的回归风险，并输出审查报告。',
  description: 'Codex 回归审查',
  run_in_background: true,
})
```

Codex 进程按需启动并复用 app-server。`CODEX_BIN` 可显式指定二进制或 `codex.js` 路径；未指定时插件会尝试探测 npm 全局安装。无人值守运行默认使用 `approval_policy: never`，请先确认工作目录和 CLI 权限范围。

### Claude Code 示例

```sh
npm install -g @anthropic-ai/claude-agent-sdk
```

```js
await subagent_routed({
  engine: 'claude',
  cwd: 'E:/projects/my-repo',
  model: 'claude-sonnet-4-5',
  prompt: '为这个项目整理一份迁移风险清单。',
  description: 'Claude 迁移分析',
})
```

当 Claude CLI 使用第三方或本地兼容后端时，`sessionId + persistSession` 可能无法可靠续话；需要 continuable 时请使用官方 Anthropic API。

### CodeBuddy 示例

```js
await subagent_routed({
  engine: 'codebuddy',
  cwd: 'E:/projects/my-repo',
  model: 'hy3',
  prompt: '运行测试并定位失败原因。',
  description: 'CodeBuddy 测试诊断',
})
```

可用 `CODEBUDDY_BIN` 指定 CLI 路径，`CODEBUDDY_MODEL` 指定默认模型；未传 `model` 时默认使用 `hy3`。该引擎会以非交互方式运行并跳过权限询问，请只在可信工作目录中使用。

## 模型路由与预检

对 DSH 引擎：

1. 未提供 `model` / `provider` 时，子代理继承父会话的模型路由。
2. 提供任一覆盖参数后，插件会通过 Harness 的 `resolveChildAgentOptions` 解析本次调用的配置。
3. 当 Harness 暴露 `llm` 服务且 provider 路由可用时，插件会先调用模型目录进行预检；模型不存在会快速失败，并尽量列出该 provider 的候选模型。
4. 预检通过不代表实际调用一定成功；provider 网络、API key、额度和模型权限仍取决于运行环境。

外部引擎的模型由其 CLI/SDK 解析，跳过 DSH 的模型目录预检。

## 平台补丁：让 continuable 保持目标 preset

`continuable` 需要在多次恢复之间记住目标 preset。DeepSeek Harness 0.1.5-rc.1 的官方 `@deepseek-ai/dsh-subagent` 并未在所有路径保存这项信息，因此本插件配套了一个**纯增量补丁 fork**。该补丁不是 npm 依赖，属于 Harness 安装级装配要求。

补丁的关键行为：

- `composition.preset` 会在创建和冷恢复时挂载目标 preset，而不是重新加入父 preset。
- `materializeTracked` setup 改为异步，并由 agent factory 等待，保留原有 `{ commit }` 契约。
- continuable descriptor 增加可选 `preset` 字段；旧版 descriptor 仍可解析，one-shot descriptor 版本不变。
- `coldResume` 从 descriptor 中恢复同一个 preset；preset 缺失时返回带名称的错误。
- 外部引擎 continuable 依赖 `registerExternalContinuation`；补丁未生效时任务可能能启动，但不能通过 `send_message` 续话。

安装级链接应指向补丁 fork：

```text
<harness>/resources/host/node_modules/@deepseek-ai/dsh-subagent
  -> <patched-fork>/@deepseek-ai/dsh-subagent
```

不要再给 `@deepseek-ai/dsh-subagent` 添加 profile-local `link:` 依赖，否则会产生重复模块身份，导致补丁和 Harness 实际加载的实例不一致。插件启动时会检查补丁标记；如果日志提示 assertion 失败，`continuable + preset` 不应视为可靠可用。

回滚时删除安装级 junction，将备份的原始包恢复到原路径并重启 Harness。已经创建的 preset-continuable 子代理在回滚后变为 `NOT_RESUMABLE`，这是预期行为。

## 官方 subagent 工具的处理

插件默认启用 `disableStockSubagent: true`。因此挂载本插件的 preset 中，官方 `subagent` / `subagent_fork` 会在执行时被拦截，并提示改用 `subagent_routed`，避免同一 Harness 同时存在两套不同的 preset 继承语义。

如需暂时保留官方工具，可在插件配置中设置：

```js
{ disableStockSubagent: false }
```

该 guard 只覆盖挂载本插件行的 preset；没有挂载本插件的 preset 不受影响。

## 已知限制与排查

- **Preset 代际漂移**：每次创建/恢复都会按 id 重新解析 preset。如果两次 continuable 调用之间修改了 preset 文件，后续轮次会使用新一代 preset。需要保持一致时，请在整个会话期间固定 preset 内容。
- **失败语义**：子代理以 `error`、`refusal` 或 `max-tokens` 结束时，工具调用会抛错，并尽量附带部分输出；`completed` 和调用方主动取消的 `aborted` 才会作为正常结果返回。
- **后台能力依赖 jobs 服务**：如果 Harness 未加载 `@deepseek-ai/dsh-jobs`，请使用 `run_in_background: false`，或先补齐 jobs 服务。
- **外部引擎的 cwd**：父会话 cwd 往往是对话目录，不一定是目标仓库。代码审查、运行测试等任务建议始终显式传 `cwd`。
- **外部引擎权限**：Codex、Claude 和 CodeBuddy 都是独立进程；它们的登录状态、API key、网络和文件权限不会由 DSH 模型预检替代。

建议按以下顺序排查：确认 Harness 版本为 `0.1.5-rc.1`，确认目标 preset id 存在，再检查启动日志中的 provider 注册和补丁 assertion，最后使用 `run_in_background: false` 获取完整错误。

## 开发与验证

本项目是纯 ESM 插件，零构建步骤：核心逻辑位于 `lib/index.js`，外部引擎位于 `lib/engines/`。

```sh
# 语法检查
node --check lib/index.js
node --check lib/engines/codex.js
node --check lib/engines/claude.js
node --check lib/engines/codebuddy.js

# npm script（等价于核心检查）
npm run check
```

真实功能验证需要运行在已安装 DeepSeek Harness `0.1.5-rc.1` 的环境中，并准备对应的 provider、preset 及外部 CLI 登录状态。

## 来源与版权说明

本仓库基于上游代码修改而来，仓库中继承的原始代码版权声明为：

```text
Copyright (c) 2026 bpc-oss
```

原始代码及其衍生部分遵循 MIT License，完整条款见仓库根目录的 `LICENSE` 文件。本仓库的主要修改方向是适配 DeepSeek Harness `0.1.5-rc.1`，并补充 preset 路由、模型/provider 覆盖、后台任务、continuable 续话和 Codex/Claude/CodeBuddy 外部引擎支持；这些修改由 Xiao-Hong 维护。

发布、复制或再分发本项目时，请保留原始版权声明和 MIT 许可文本。若某个文件或第三方依赖带有单独的许可证或版权声明，应同时遵守其适用条款。

## 许可证

MIT
