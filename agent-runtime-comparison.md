# Agent Runtime 对比报告：zgsm-sangfor/opencode vs zgsm-ai/costrict

## 1. 结论摘要

本报告对比以下两个仓库的 agent runtime 实现：

- `zgsm-sangfor/opencode`
  - 仓库：https://github.com/zgsm-sangfor/opencode
  - 对比提交：`b158c71902a635994f578da65c7f824d391f7a3d`
  - 本地检查目录：`/tmp/zgsm-opencode`
- `zgsm-ai/costrict`
  - 仓库：https://github.com/zgsm-ai/costrict
  - 对比提交：`ffb7485b775925ecc3247d6bac735f4408f0c03a`
  - 本地检查目录：`/tmp/zgsm-costrict`

核心结论：

**两者的 agent runtime 不一致。**

二者都具备“LLM + system prompt/mode + tools + permission/approval + 多轮循环 + 子任务/委派”的 agent 能力，但运行时架构、状态模型、工具调用链、stream 处理方式、子 agent 机制和上下文管理策略均不同。

`opencode` 更像一个基于 session 的通用 agent runtime：agent 是配置对象，session loop 统一调度，工具通过 Vercel AI SDK 的 `streamText` 工具机制注册和执行。

`costrict` 更像一个 VS Code/Roo Code 风格的 task runtime：`Task` 是长生命周期执行实体，runtime 自己解析 provider stream 中的 tool-call chunk，再由 `presentAssistantMessage` 顺序呈现和执行工具。

`opencode` 仓库中确实嵌入了一批 CoStrict 相关 agent prompt、工具和 provider 兼容逻辑，但这属于适配层和能力移植，不是把 `costrict` 的 `Task` runtime 原样搬过去。

## 2. 对比范围

本报告聚焦“agent runtime”，即一次用户请求从进入系统到模型调用、工具执行、状态落盘、继续下一轮或结束的完整运行链路。主要观察维度：

- agent/mode 抽象
- prompt 和上下文构建
- LLM 调用方式
- stream 事件处理
- tool 定义、过滤、执行和结果回填
- permission/approval 机制
- 子任务/子 agent 机制
- 状态持久化
- context overflow / condense / retry
- CoStrict 适配层是否等同于 runtime 一致

## 3. 总体架构差异

| 维度 | zgsm-sangfor/opencode | zgsm-ai/costrict |
| --- | --- | --- |
| runtime 核心实体 | `Session` + `Agent.Info` + `SessionPrompt` + `SessionProcessor` | `Task` |
| agent 定义 | 配置化 agent，支持 builtin/custom agent | mode/task 体系，基于 Roo Code `ModeConfig` |
| 主循环 | `SessionPrompt.runLoop` | `Task.recursivelyMakeClineRequests` |
| LLM 调用 | `LLM.stream` 调用 AI SDK `streamText` | `ApiHandler.createMessage` 返回自定义 `ApiStream` |
| stream 处理 | `SessionProcessor` 消费 AI SDK event | `Task` 手动 switch provider chunk |
| 工具执行 | 工具注册给 AI SDK，由 AI SDK 驱动 tool call/result event | 自己解析 tool call，`presentAssistantMessage` 顺序执行 |
| 工具定义 | `Tool.define(id, init)` + zod schema | `BaseTool<TName>` 子类 + nativeArgs |
| 子任务 | `task` tool 创建子 session，递归调用 `SessionPrompt.prompt` | `new_task` tool 通过 provider 委派 child task |
| 状态模型 | session/message/part 结构 | task/apiConversationHistory/clineMessages |
| 产品形态 | CLI/app/server 多端 runtime | VS Code extension/webview runtime |

## 4. opencode Agent Runtime

### 4.1 agent 抽象

`opencode` 的 agent 抽象集中在：

- `packages/opencode/src/agent/agent.ts`
  - `Agent.Info` 定义在第 26 行附近。
  - 包含 `name`、`description`、`mode`、`native`、`permission`、`model`、`prompt`、`options`、`steps` 等字段。

agent 在这里不是执行对象，而是“运行配置”。真正执行由 session runtime 负责。

默认内置 agent 包括：

- `build`
- `plan`
- `general`
- `explore`
- `compaction`
- `title`
- `summary`

参考位置：

- `/tmp/zgsm-opencode/packages/opencode/src/agent/agent.ts:26`
- `/tmp/zgsm-opencode/packages/opencode/src/agent/agent.ts:106`
- `/tmp/zgsm-opencode/packages/opencode/src/agent/agent.ts:236`

### 4.2 agent 配置加载

`opencode` 的 agent 配置来自多处：

- 默认内置 agent
- `cfg.agent`
- `.costrict/agent`
- `.costrict/agents`
- `agent`
- `agents`

CoStrict 内置 agent prompt 通过 `BUILTIN_AGENTS` 注入：

- `/tmp/zgsm-opencode/packages/opencode/src/config/config.ts:276`
- `/tmp/zgsm-opencode/packages/opencode/src/config/config.ts:281`
- `/tmp/zgsm-opencode/packages/opencode/src/config/config.ts:325`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/agent/builtin.ts:44`

这说明 `opencode` 中有 CoStrict agent prompt 适配，但这些 agent 仍然被转换成 `opencode` 自己的 `Agent.Info`，由 `SessionPrompt` runtime 执行。

### 4.3 主循环

`opencode` 的主循环在 `SessionPrompt.runLoop`：

- `/tmp/zgsm-opencode/packages/opencode/src/session/prompt.ts:1334`

核心流程：

1. 读取当前 session 历史。
2. 找到最后的 user/assistant message。
3. 判断是否已有非 tool-call 完成消息，若完成则退出 loop。
4. 处理 queued subtask 或 compaction。
5. 检查 context overflow，必要时创建 compaction。
6. 根据最后 user message 的 agent 字段加载 agent 配置。
7. 创建新的 assistant message。
8. 构建 tools、system prompt、skills、environment、instructions。
9. 将历史消息转换成模型消息。
10. 调用 `SessionProcessor.process`。
11. 如果结果是 `continue`，继续下一轮；如果是 `compact`，插入 compaction；如果是 `stop`，结束。

loop 外层通过 `Runner` 保证同一个 session 的并发控制：

- `/tmp/zgsm-opencode/packages/opencode/src/session/prompt.ts:1566`

### 4.4 LLM 调用

`opencode` 的 LLM 调用入口是：

- `/tmp/zgsm-opencode/packages/opencode/src/session/llm.ts:93`

其内部调用 AI SDK：

- `/tmp/zgsm-opencode/packages/opencode/src/session/llm.ts:291`

关键特征：

- 使用 `streamText`
- 统一拼装 system prompt
- 通过 `ProviderTransform` 转换模型参数和 schema
- 支持 provider/plugin hook
- 将 tools 作为 AI SDK tool set 传入
- 由 AI SDK 产生标准事件，如 `tool-input-start`、`tool-call`、`tool-result`、`finish-step`

### 4.5 stream 处理

`opencode` 的 stream 消费在 `SessionProcessor`：

- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:475`

典型事件处理：

- `tool-input-start`
- `tool-call`
- `tool-result`
- `tool-error`
- `start-step`
- `finish-step`
- `text-start`
- `text-delta`
- `text-end`

参考位置：

- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:158`
- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:179`
- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:220`
- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:239`
- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:272`
- `/tmp/zgsm-opencode/packages/opencode/src/session/processor.ts:319`

这里的重点是：`SessionProcessor` 不自己解析 OpenAI-style function call chunk，而是消费 AI SDK 已经标准化后的事件。

### 4.6 工具模型

工具注册中心：

- `/tmp/zgsm-opencode/packages/opencode/src/tool/registry.ts:41`
- `/tmp/zgsm-opencode/packages/opencode/src/tool/registry.ts:167`

工具定义接口：

- `/tmp/zgsm-opencode/packages/opencode/src/tool/tool.ts:8`

工具通过 `Tool.define` 定义，典型工具包括：

- `bash`
- `read`
- `edit`
- `write`
- `glob`
- `grep`
- `task`
- `todowrite`
- `webfetch`
- `websearch`
- `codesearch`
- `apply_patch`
- `skill`
- `checkpoint`
- `workflow`
- `sequential-thinking`
- `file-outline`
- `spec-manage`

工具被转换成 AI SDK `tool(...)`：

- `/tmp/zgsm-opencode/packages/opencode/src/session/prompt.ts:431`
- `/tmp/zgsm-opencode/packages/opencode/src/session/prompt.ts:436`
- `/tmp/zgsm-opencode/packages/opencode/src/session/prompt.ts:440`

执行时会通过 `ctx.ask` 触发权限判断，并通过 plugin hook 暴露执行前后事件：

- `tool.execute.before`
- `tool.execute.after`

### 4.7 子 agent / subtask

`opencode` 的子 agent 入口是 `task` tool：

- `/tmp/zgsm-opencode/packages/opencode/src/tool/task.ts:28`

它会：

1. 根据 `subagent_type` 找到 agent。
2. 创建或恢复子 session。
3. 继承或调整权限。
4. 调用 `SessionPrompt.prompt` 递归执行子 session。
5. 将子 session 最终文本包装成 `<task_result>` 返回给父 session。

关键位置：

- `/tmp/zgsm-opencode/packages/opencode/src/tool/task.ts:75`
- `/tmp/zgsm-opencode/packages/opencode/src/tool/task.ts:130`

这意味着 `opencode` 的子 agent 是“session 内嵌 session”的模型。

## 5. costrict Agent Runtime

### 5.1 runtime 核心实体

`costrict` 的核心执行对象是 `Task`：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:177`

`Task` 同时承担多种职责：

- 保存 task id、parent/child task 信息
- 管理 VS Code/webview 消息
- 管理 API conversation history
- 管理 mode/profile/provider state
- 构建 system prompt 和 environment details
- 调用 provider
- 处理 stream
- 解析 native tool call
- 执行工具
- 保存 checkpoint
- 处理上下文压缩、retry、fallback、approval

相较 `opencode`，`costrict` 的 runtime 更集中在一个大型 `Task` 类中。

### 5.2 mode 抽象

`costrict` 使用 Roo Code 风格的 mode 体系：

- `/tmp/zgsm-costrict/packages/types/src/mode.ts:440`

默认 mode 包括：

- `code`
- `architect`
- `ask`
- `debug`
- `orchestrator`
- workflow modes

关键位置：

- `/tmp/zgsm-costrict/packages/types/src/mode.ts:442`
- `/tmp/zgsm-costrict/packages/types/src/mode.ts:490`

每个 mode 通过 `groups` 控制工具能力，例如：

- `read`
- `edit`
- `command`
- `mcp`

与 `opencode` 的差异是：`costrict` 的 mode 更接近“UI 可切换工作模式 + 工具组权限”，而 `opencode` 的 agent 是“prompt + permission + model/options”的可组合运行配置。

### 5.3 主循环

`costrict` 的主循环是：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:2815`

`recursivelyMakeClineRequests` 的核心流程：

1. 维护 stack，避免真正递归导致调用栈问题。
2. 处理 mistake limit。
3. 获取 provider state、mode、实验开关。
4. 处理 user content mentions。
5. 构建 environment details。
6. 把 user message 写入 API conversation history。
7. 调用 `attemptApiRequest` 获取 stream。
8. 逐 chunk 处理 reasoning、usage、tool_call、tool_call_partial、text 等。
9. stream 结束后保存 assistant message。
10. 等待 tool result ready。
11. 根据 tool result 构建下一轮 user content。
12. 继续下一轮或结束。

参考位置：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:2815`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3138`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3230`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3354`

### 5.4 API 调用

`costrict` 的 API 调用入口是：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:4544`

内部调用：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:4916`

即：

```ts
effectiveApi.createMessage(systemPrompt, cleanConversationHistory, metadata)
```

`metadata` 中会携带：

- mode
- prompt tags
- task id
- instance id
- tools
- tool choice
- parallel tool calls
- allowed function names
- costrict workflow mode
- language
- request headers callback
- abort controller

参考位置：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:4830`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:4891`

与 `opencode` 相比，`costrict` 没有把 runtime 直接交给 AI SDK 的 `streamText`，而是通过自身 `ApiHandler` 层适配各 provider。

### 5.5 stream 和 tool-call 解析

`costrict` 自己处理 provider stream chunk。关键分支包括：

- `reasoning`
- `usage`
- `grounding`
- `fake_tool_call`
- `tool_call_partial`
- `tool_call`
- `fake_reasoning`
- `automodel`
- `text`

关键位置：

- `/tmp/zgsm-costrict/src/core/task/Task.ts:3185`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3217`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3230`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3354`
- `/tmp/zgsm-costrict/src/core/task/Task.ts:3439`

native tool call parser：

- `/tmp/zgsm-costrict/src/core/assistant-message/NativeToolCallParser.ts:56`
- `/tmp/zgsm-costrict/src/core/assistant-message/NativeToolCallParser.ts:209`

`NativeToolCallParser` 负责：

- tracking streaming tool calls
- tracking raw chunk index
- 累积 arguments delta
- 处理 `tool_call_start`
- 处理 `tool_call_delta`
- 处理 `tool_call_end`
- 将 native tool call 转换成 `ToolUse`
- 处理 MCP 动态工具名
- 兼容 `fake_tool_call`

这是与 `opencode` 非常关键的差异：`costrict` 自己维护 tool-call streaming 状态机。

### 5.6 工具呈现和执行

`costrict` 的工具执行入口在：

- `/tmp/zgsm-costrict/src/core/assistant-message/presentAssistantMessage.ts:80`

该函数负责：

- 防止并发执行同一个 assistant message block
- 按 `currentStreamingContentIndex` 顺序消费 block
- 展示 text block
- 处理 `tool_use`
- 处理 `mcp_tool_use`
- 请求用户 approval
- 执行具体工具
- 将 tool result 放入下一轮 user content
- 控制是否继续下一轮 API 请求

工具基类：

- `/tmp/zgsm-costrict/src/core/tools/BaseTool.ts:29`

工具执行统一入口：

- `/tmp/zgsm-costrict/src/core/tools/BaseTool.ts:113`

`BaseTool.handle` 会：

1. 如果是 partial block，调用 `handlePartial`。
2. 从 `nativeArgs` 中取 typed parameters。
3. 如果缺少 native args，报错。
4. 调用具体工具的 `execute`。

这说明 `costrict` 的工具执行是 runtime 自己调度的，不依赖 AI SDK tool executor。

### 5.7 工具定义和过滤

native tool 构建入口：

- `/tmp/zgsm-costrict/src/core/task/build-tools.ts:83`

它会：

1. 获取 MCP hub。
2. 获取 code index manager。
3. 构建 native tools。
4. 按 mode/custom mode/experiments/filter settings 过滤。
5. 构建 MCP dynamic tools。
6. 可选加载 custom tools。
7. 对 Gemini 等 provider 支持 “all tools + allowedFunctionNames” 模式。

工具参数类型在：

- `/tmp/zgsm-costrict/src/shared/tools.ts:105`

典型工具包括：

- `read_file`
- `read_command_output`
- `execute_command`
- `write_to_file`
- `apply_diff`
- `edit`
- `edit_file`
- `search_replace`
- `apply_patch`
- `list_files`
- `search_files`
- `use_mcp_tool`
- `access_mcp_resource`
- `new_task`
- `attempt_completion`
- `switch_mode`
- `update_todo_list`
- `skill`
- `generate_image`
- `costrict_checkpoint`

这套工具名和 `opencode` 的工具名不是一一相同。例如：

| 能力 | opencode 工具名 | costrict 工具名 |
| --- | --- | --- |
| shell 命令 | `bash` | `execute_command` |
| 读文件 | `read` | `read_file` |
| 写文件 | `write` | `write_to_file` |
| 编辑 | `edit` / `apply_patch` | `edit` / `edit_file` / `apply_patch` / `apply_diff` |
| 搜索 | `grep` / `glob` | `search_files` / `list_files` |
| 子任务 | `task` | `new_task` |
| 完成任务 | 普通 assistant finish 或 structured output | `attempt_completion` |
| todo | `todowrite` | `update_todo_list` |
| checkpoint | `checkpoint` | `costrict_checkpoint` |

### 5.8 子任务/委派

`costrict` 的子任务工具是：

- `/tmp/zgsm-costrict/src/core/tools/NewTaskTool.ts:19`

它会：

1. 校验 `mode` 和 `message`。
2. 检查父 mode 是否限制可委派 mode。
3. 可选校验 todos。
4. 找到 target mode。
5. 请求 approval。
6. 调用 provider 的 `delegateParentAndOpenChild`。

关键位置：

- `/tmp/zgsm-costrict/src/core/tools/NewTaskTool.ts:126`

这与 `opencode` 的 `task` tool 明显不同。`costrict` 的 child task 是 VS Code/webview 任务模型的一部分，而不是简单创建子 session 并同步返回结果。

## 6. 关键运行链路对照

### 6.1 opencode 请求链路

简化链路：

```text
User input
  -> SessionPrompt.prompt
  -> create user message
  -> SessionPrompt.loop / runLoop
  -> load session history
  -> select Agent.Info
  -> resolveTools
  -> build system/env/skills/instructions
  -> MessageV2.toModelMessages
  -> SessionProcessor.process
  -> LLM.stream
  -> AI SDK streamText
  -> SessionProcessor.handleEvent
  -> update MessageV2 parts
  -> continue/compact/stop
```

核心特征：

- runtime 是 session-centric。
- agent 是运行配置。
- tool calling 主要走 AI SDK event。
- message state 被拆成 assistant message 和 parts。
- loop 以 assistant finish reason 和 tool-call 状态决定是否继续。

### 6.2 costrict 请求链路

简化链路：

```text
User input
  -> Task
  -> recursivelyMakeClineRequests
  -> process mentions/environment details
  -> add user message to apiConversationHistory
  -> attemptApiRequest
  -> ApiHandler.createMessage
  -> provider ApiStream
  -> Task switch(chunk.type)
  -> NativeToolCallParser
  -> assistantMessageContent blocks
  -> presentAssistantMessage
  -> BaseTool.handle / concrete tool.execute
  -> push tool_result to userMessageContent
  -> next request or attempt_completion
```

核心特征：

- runtime 是 task-centric。
- mode 是 UI 和工具权限的核心抽象。
- tool calling 由 runtime 自己解析和执行。
- assistant content 和 tool result 通过 `assistantMessageContent` / `userMessageContent` 两个缓冲区衔接。
- 结束通常依赖 `attempt_completion` 等工具语义。

## 7. 相似点

两者在能力层面有明显相似：

1. 都支持多轮 agent loop。
2. 都支持工具调用。
3. 都支持权限/审批。
4. 都支持 MCP。
5. 都支持子任务/委派。
6. 都支持上下文压缩或管理。
7. 都有 CoStrict 相关 prompt/workflow/tool 能力。
8. 都会将工具执行结果回填给模型，驱动下一轮推理。

这些相似点说明二者目标相近，但不能推出 runtime 实现一致。

## 8. 主要不一致点

### 8.1 runtime 所有权不同

`opencode`：

- runtime 分散在 `SessionPrompt`、`SessionProcessor`、`LLM`、`ToolRegistry`。
- agent 配置是输入。
- session/message/part 是状态主体。

`costrict`：

- runtime 高度集中在 `Task`。
- task 是执行实体。
- VS Code provider/webview 状态深度参与 runtime。

### 8.2 tool-call 状态机不同

`opencode`：

- 将 tools 注册给 AI SDK。
- AI SDK 输出标准事件。
- `SessionProcessor` 消费标准事件。
- 不直接维护 raw tool-call chunk index/arguments buffer。

`costrict`：

- provider stream 原始 chunk 进入 `Task`。
- `NativeToolCallParser` 维护 streaming state。
- `presentAssistantMessage` 顺序消费 block。
- 工具执行和 UI 呈现绑定更紧密。

### 8.3 工具 API 不同

`opencode` 工具签名：

```text
Tool.define(id, init)
  -> parameters: zod schema
  -> execute(args, ctx)
  -> returns { title, metadata, output, attachments? }
```

`costrict` 工具签名：

```text
class XTool extends BaseTool<"tool_name">
  -> handle(task, block, callbacks)
  -> execute(params, task, callbacks)
  -> pushToolResult(...)
```

这会导致工具迁移时不能简单复用，需要做适配。

### 8.4 子任务模型不同

`opencode` 子任务：

- `task` tool
- 创建子 session
- 同步执行子 session prompt
- 返回 `<task_result>`

`costrict` 子任务：

- `new_task` tool
- 调用 VS Code provider 委派 child task
- 父任务进入 delegated/child active 语义
- 不只是同步函数调用

### 8.5 agent/mode 权限模型不同

`opencode`：

- agent 带 `permission` ruleset。
- permission 以具体 tool/pattern/action 表达。
- agent 可是 primary/subagent/all。

`costrict`：

- mode 带 `groups`。
- group 映射到工具集合。
- custom mode 可限制文件 regex、taskMode 等。

### 8.6 上下文管理不同

`opencode`：

- `SessionCompaction`
- `SessionSummary`
- token overflow 后创建 compaction part
- loop 内处理 compaction task

`costrict`：

- `manageContext`
- auto condense/truncation
- profile thresholds
- context management metadata 中还会带 tools
- API 请求前可能主动压缩或截断历史

### 8.7 产品集成边界不同

`opencode`：

- 面向 CLI/app/server/desktop。
- runtime 较平台无关。

`costrict`：

- 深度绑定 VS Code extension。
- `Task` 直接处理 webview、workspace、terminal、diff view、checkpoint provider 等。

## 9. CoStrict 适配层分析

`opencode` 中存在明显 CoStrict 相关目录：

- `packages/opencode/src/costrict/agent`
- `packages/opencode/src/costrict/tool`
- `packages/opencode/src/costrict/provider`
- `packages/opencode/src/costrict/review`
- `packages/opencode/src/costrict/utils`

其中 agent prompt 通过 `BUILTIN_AGENTS` 注入，工具如 `sequential-thinking`、`file-outline`、`checkpoint`、`spec-manage`、`workflow` 被注册进 `ToolRegistry`。

这说明 `opencode` 已吸收或适配 CoStrict 的一部分能力：

- strict/spec/workflow prompt
- CoStrict provider 兼容
- CoStrict 工具
- review/skills 相关能力
- `<think>` 等 provider 文本兼容处理

但这不是 runtime 一致，原因是：

1. CoStrict prompt 被转成 `opencode` agent 配置执行。
2. CoStrict 工具被实现为 `opencode` `Tool.define` 风格。
3. 执行仍然走 `SessionPrompt`、`SessionProcessor`、`LLM.stream`。
4. 不存在 `costrict` 的 `Task.recursivelyMakeClineRequests` 在 `opencode` 中作为核心 loop 复用。

因此更准确的判断是：

**opencode 包含 CoStrict 能力适配层，但 agent runtime 仍是 opencode 自己的 runtime。**

## 10. 迁移/兼容影响

如果目标是让两边 agent 行为保持一致，需要注意以下风险：

### 10.1 prompt 一致不等于行为一致

即使两边使用同一份 strict/spec prompt，由于工具名、工具描述、工具 schema、审批策略、上下文注入格式不同，模型实际行为仍可能不同。

### 10.2 工具名差异会影响模型选择

例如模型在 `costrict` 中学习到 `read_file`、`execute_command`、`attempt_completion`，但在 `opencode` 中对应工具是 `read`、`bash`、普通 finish 或 structured output。工具名和参数 schema 差异会直接影响 tool-call 成功率。

### 10.3 子任务语义不同

`opencode` 的 `task` 更像同步 subagent 调用；`costrict` 的 `new_task` 更像 UI task delegation。多 agent 协作策略不能直接等价。

### 10.4 stream 中间态不同

`costrict` 支持 partial tool call 呈现和工具参数稳定性处理；`opencode` 依赖 AI SDK 标准事件。某些 provider 的非标准 tool streaming 行为可能在两边表现不同。

### 10.5 上下文压缩时机不同

同样的长任务，在两边可能因 compaction/condense 时机不同而给模型不同历史，从而产生不同行为。

## 11. 建议的验证方式

如果需要进一步证明“行为是否一致”，建议做一组黑盒回归用例，而不只看代码：

1. 固定同一个 provider/model。
2. 固定同一份 system prompt。
3. 固定同一仓库和初始 git 状态。
4. 禁用或对齐自动压缩策略。
5. 对齐工具集合和工具描述。
6. 设计以下任务：
   - 只读代码问答
   - 修改单文件
   - 多文件重构
   - 命令执行 + 测试修复
   - 子任务委派
   - MCP 工具调用
   - 上下文接近窗口上限
7. 比较：
   - 模型请求 payload
   - tool call 序列
   - tool result 格式
   - 最终 diff
   - 失败/重试路径
   - token/context 管理行为

仅从代码结构判断，二者 runtime 不一致；若要评估用户可见行为是否接近，需要上述黑盒用例。

## 12. 最终判断

| 判断项 | 结论 |
| --- | --- |
| agent/runtime 代码是否同源同构 | 否 |
| 主循环是否一致 | 否 |
| tool-call 执行机制是否一致 | 否 |
| agent/mode 数据模型是否一致 | 否 |
| 子任务机制是否一致 | 否 |
| 是否有 CoStrict 能力在 opencode 中适配 | 是 |
| 能否认为高层 agent 能力相似 | 是 |
| 能否认为 runtime 行为完全一致 | 否 |

一句话总结：

**`zgsm-sangfor/opencode` 是一个带 CoStrict 能力适配的 opencode session runtime；`zgsm-ai/costrict` 是 Roo Code/VS Code task runtime。二者高层能力相似，但 agent runtime 实现和运行语义不一致。**
