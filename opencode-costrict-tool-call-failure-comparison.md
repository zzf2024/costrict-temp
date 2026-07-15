# OpenCode CLI 与 CoStrict VSCode 插件工具调用与失败处理对比报告

调研对象：

- `zgsm-sangfor/opencode`：本地克隆路径 `/tmp/zgsm-sangfor-opencode`，分支 `dev`，commit `b158c71902a635994f578da65c7f824d391f7a3d`。
- `zgsm-ai/costrict`：本地克隆路径 `/tmp/zgsm-ai-costrict`，分支 `main`，commit `ffb7485b775925ecc3247d6bac735f4408f0c03a`。

## 1. 总体结论

两个仓库的工具调用架构差异很明显：

- OpenCode CLI 走“模型流事件 + AI SDK tool execute 回调 + session processor 持久化状态”的模式。工具失败通常表现为工具实现抛异常，然后由流事件转成 `tool-error`，最终落到会话里的 tool part `state.status = "error"`。
- CoStrict VSCode 插件走“API stream chunk -> NativeToolCallParser -> assistantMessageContent -> presentAssistantMessage 串行分发 -> userMessageContent tool_result”的模式。工具失败多数不会直接中断主循环，而是被包装成一个必须回传给模型的 `tool_result`。
- OpenCode 更偏服务端/CLI 的统一事件模型，工具执行结果和失败状态都作为 assistant message 的 part 存储。
- CoStrict 更偏 VSCode 交互产品，围绕原生工具协议做了大量兜底：tool_use/tool_result ID 去重、缺失 tool_result 自动补齐、用户拒绝后后续工具跳过但补错误结果、连续错误/重复工具调用检测、new_task 隔离等。

## 2. OpenCode CLI 工具调用链路

### 2.1 工具定义与注册

入口：

```text
packages/opencode/src/tool/registry.ts
packages/opencode/src/tool/tool.ts
```

调用链：

```text
Tool.define(...)
  -> ToolRegistry.tools(model, agent)
  -> SessionPrompt.resolveTools(...)
  -> AI SDK streamText({ tools, activeTools, toolChoice, ... })
  -> 模型返回 tool call
  -> AI SDK 调用对应 tool.execute(...)
  -> SessionProcessor 接收 tool-result / tool-error 事件
```

`Tool.define(...)` 会包装每个工具：

- 先用 Zod schema 校验参数。
- 参数不合法时抛 `Error`，如果工具提供 `formatValidationError` 则使用定制错误文案。
- 工具执行成功后统一做输出截断 `Truncate.output(...)`，除非工具自己已设置 `metadata.truncated`。

### 2.2 工具注入到模型请求

入口：

```text
packages/opencode/src/session/prompt.ts
```

核心函数：

```text
SessionPrompt.resolveTools(...)
```

这个函数会：

1. 从 `ToolRegistry.tools(...)` 取内置工具。
2. 把每个工具转成 AI SDK `tool(...)`。
3. 为每个工具注入统一执行上下文 `Tool.Context`。
4. 触发插件 hook：

```text
tool.execute.before
tool.execute.after
```

工具执行上下文包含：

```ts
{
  sessionID,
  messageID,
  callID,
  abort,
  agent,
  messages,
  metadata(...),
  ask(...)
}
```

其中：

- `metadata(...)` 可以在工具运行中更新 tool part 的标题和 metadata。
- `ask(...)` 走权限系统 `permission.ask(...)`，会把工具调用 ID、session、agent permission、session permission 合并后发起审批。

### 2.3 模型请求与 tool call 修复

入口：

```text
packages/opencode/src/session/llm.ts
```

OpenCode 调用 AI SDK：

```ts
streamText({
  tools,
  activeTools,
  toolChoice,
  experimental_repairToolCall,
  maxRetries,
  ...
})
```

关键逻辑：

- `activeTools` 会排除内部兜底工具 `invalid`。
- `experimental_repairToolCall(...)` 用于修复模型发出的坏工具调用：
  - 如果工具名只是大小写问题，例如模型返回 `Read` 而实际工具是 `read`，会转成小写工具名。
  - 如果无法修复，则改写成 `invalid` 工具，并把原工具名和错误原因作为输入。

这意味着“未知工具 / 参数结构不符合 SDK 预期”不会直接把整个请求打断，而是尽量转成 `invalid` 工具调用，让模型在下一轮看到错误。

### 2.4 SessionProcessor 如何记录工具状态

入口：

```text
packages/opencode/src/session/processor.ts
```

关键事件：

```text
tool-input-start
tool-call
tool-result
tool-error
finish-step
error
```

状态流转：

```text
tool-input-start
  -> 创建 MessageV2.ToolPart，state.status = "pending"

tool-call
  -> 更新为 state.status = "running"
  -> 写入 input、start time、provider metadata

tool-result
  -> 更新为 state.status = "completed"
  -> 写入 output、metadata、title、attachments、end time

tool-error
  -> 更新为 state.status = "error"
  -> 写入 error message、input、end time
```

如果工具处于 `pending` 或 `running`，但流被中断或清理，`cleanup()` 会把未完成工具标为：

```text
state.status = "error"
error = "Tool execution aborted"
```

在历史消息重新转换为模型输入时，OpenCode 还会把未完成工具变成模型可接受的错误结果：

```text
[Tool execution was interrupted]
```

对应入口：

```text
packages/opencode/src/session/message-v2.ts
```

这样可以避免 Anthropic/Claude 一类 API 因为有 `tool_use` 但没有对应 `tool_result` 而报协议错误。

### 2.5 OpenCode 失败处理分类

#### 2.5.1 参数校验失败

位置：

```text
packages/opencode/src/tool/tool.ts
```

处理方式：

- `toolInfo.parameters.parse(args)` 失败。
- 抛出带 schema 说明的 `Error`。
- AI SDK 将其作为工具执行错误向外发 `tool-error`。
- `SessionProcessor` 把 tool part 标成 `error`。
- 下一轮模型会在历史里看到该工具的 `output-error`。

#### 2.5.2 工具内部业务失败

例子：

```text
read.ts       文件不存在、二进制文件、offset 越界
grep.ts       ripgrep 失败
webfetch.ts   URL 非 http(s)、HTTP 状态错误、响应过大
apply_patch.ts patch 格式错误、校验失败、hunk 失败
edit.ts       未找到 oldString、多处匹配、文件不存在
```

处理方式：

- 多数工具直接 `throw new Error(...)`。
- 不在工具内部吞掉异常。
- 由统一流转进入 `tool-error`。

#### 2.5.3 权限拒绝 / 用户拒绝

涉及：

```text
Tool.Context.ask(...)
Permission.RejectedError
Question.RejectedError
```

处理方式：

- 工具内调用 `ctx.ask(...)`。
- 如果用户拒绝，抛 `Permission.RejectedError` 或 `Question.RejectedError`。
- `SessionProcessor` 在 `tool-error` 里识别这两类错误。
- 如果配置 `experimental.continue_loop_on_deny !== true`，会设置 `ctx.blocked = true`。
- 当前 loop 停止，不再继续自动跑后续模型轮次。

#### 2.5.4 模型输出工具名错误

位置：

```text
packages/opencode/src/session/llm.ts
```

处理方式：

- 尝试大小写修复。
- 修复不了就转成 `invalid` 工具。
- `invalid` 工具输出类似“参数无效”的结果，供模型自我修正。

#### 2.5.5 模型/Provider API 错误

位置：

```text
packages/opencode/src/session/retry.ts
packages/opencode/src/session/processor.ts
```

处理方式：

- `SessionRetry.policy(...)` 根据错误类型决定是否重试。
- 会重试连接错误、rate limit、overloaded、部分 provider retryable API 错误。
- context overflow 不重试，而是触发 compact。
- 重试状态写入 `SessionStatus`。
- 最终失败时 `halt(...)` 把错误写入 assistant message，并发布 `Session.Event.Error`。

#### 2.5.6 Doom loop 检测

位置：

```text
packages/opencode/src/session/processor.ts
```

逻辑：

- 最近 3 个 tool part 如果是同一个工具、同一输入、且都不是 pending，就触发 `permission.ask({ permission: "doom_loop" })`。
- 这是对模型重复调用同一工具的保护。

#### 2.5.7 batch 工具的局部失败

位置：

```text
packages/opencode/src/tool/batch.ts
```

`batch` 是 OpenCode 里比较特殊的工具：

- 它内部最多执行 25 个子工具。
- 每个子调用单独 try/catch。
- 子调用失败不会让整个 batch 抛错。
- 它会为每个子调用创建独立 tool part：
  - 成功：`completed`
  - 失败：`error`
- batch 自己最终返回汇总：

```text
Executed X/Y tools successfully. N failed.
```

这说明 OpenCode 默认是“工具失败抛出并交给统一框架”，但 `batch` 是显式局部容错。

### 2.6 OpenCode 下一轮如何继续

OpenCode 的主循环在：

```text
packages/opencode/src/session/prompt.ts
```

核心逻辑：

- 如果 assistant message finish reason 是 `tool-calls`，继续 loop。
- 下一轮会通过 `MessageV2.toModelMessages(...)` 把工具成功或失败结果转换为模型历史。
- 如果 `ctx.blocked`、assistant message 有 error、或用户 abort，则停止。

因此 OpenCode 对普通工具失败的策略是：

```text
工具失败 -> 记录 error tool part -> 作为 tool error 回传模型 -> 模型下一轮自行修正
```

只有权限拒绝、问题拒绝、abort、provider hard error 等情况才会停止 loop。

## 3. CoStrict VSCode 插件工具调用链路

### 3.1 工具声明与构建

入口：

```text
src/core/prompts/tools/native-tools/index.ts
src/core/task/build-tools.ts
src/core/prompts/tools/filter-tools-for-mode.ts
```

调用链：

```text
getNativeTools(...)
  -> buildNativeToolsArrayWithRestrictions(...)
  -> filterNativeToolsForMode(...)
  -> getMcpServerTools(mcpHub)
  -> customToolRegistry
  -> API handler createMessage metadata.tools
```

CoStrict 传给模型的是 OpenAI/Anthropic 兼容的 native tool schema，而不是直接把执行函数放到 provider SDK 里。

### 3.2 模型 stream 到内部 ToolUse

入口：

```text
src/core/task/Task.ts
src/core/assistant-message/NativeToolCallParser.ts
```

模型返回工具调用时，CoStrict 处理两类 chunk：

```text
tool_call_partial
tool_call
```

流式工具调用的处理过程：

```text
tool_call_partial
  -> NativeToolCallParser.processRawChunk(...)
  -> tool_call_start / tool_call_delta / tool_call_end
  -> startStreamingToolCall(...)
  -> processStreamingChunk(...)
  -> finalizeStreamingToolCall(...)
  -> assistantMessageContent.push(ToolUse | McpToolUse)
  -> presentAssistantMessage(cline)
```

`NativeToolCallParser` 做了几类兼容：

- 用 `partial-json` 解析尚未完整的 JSON 参数。
- 解析完成后生成 `nativeArgs`，工具执行层强依赖 `nativeArgs`。
- 支持工具 alias，例如 `read` -> `read_file`。
- 支持动态 MCP 工具名，例如 `mcp--serverName--toolName`。
- 支持模型把 MCP 参数 double-wrap 到 `arguments` 字符串里的情况。
- 支持部分 CoStrict provider 特殊输出修正，例如 `<arg_key>`、`ask_multiple_choice` 格式修复。

### 3.3 工具分发执行

入口：

```text
src/core/assistant-message/presentAssistantMessage.ts
```

`presentAssistantMessage(cline)` 是 CoStrict 工具执行核心。它负责：

1. 用 lock 避免并发执行同一个 assistantMessageContent。
2. 按 `currentStreamingContentIndex` 顺序处理 block。
3. 对 text block 展示文本。
4. 对 `tool_use` block 做校验、审批、分发执行。
5. 对 `mcp_tool_use` block 转成 synthetic `use_mcp_tool` 执行。
6. 把工具结果写入 `cline.userMessageContent`，等待下一轮请求作为 user `tool_result` 发送给模型。

工具执行接口：

```text
src/core/tools/BaseTool.ts
```

每个工具继承 `BaseTool<TName>`，入口是：

```ts
tool.handle(task, block, callbacks)
```

`BaseTool.handle(...)` 做：

- partial block：调用 `handlePartial(...)`，主要用于 UI 预览。
- complete block：要求 `block.nativeArgs` 存在。
- 如果缺少 `nativeArgs`，报“Tool call is missing native arguments”。
- 然后调用具体工具的 `execute(...)`。

### 3.4 工具结果如何回写

CoStrict 不把工具结果写成 assistant part，而是写成下一条 user message 的 `tool_result`。

关键字段：

```text
Task.assistantMessageContent
Task.userMessageContent
Task.userMessageContentReady
```

入口：

```text
src/core/task/Task.ts
```

`pushToolResultToUserContent(...)` 会按 `tool_use_id` 去重：

```text
同一个 tool_use_id 只能有一个 tool_result
```

`presentAssistantMessage.ts` 内部对每个工具调用还有一层 `hasToolResult`，防止同一个 tool call 多次调用 `pushToolResult`。

### 3.5 API 历史写入顺序

CoStrict 对 native tool protocol 很谨慎。流结束时：

1. 先把 assistant message 写入 API history，里面包含 `tool_use` blocks。
2. 再执行/等待工具，工具结果进入 `userMessageContent`。
3. 再把 user message 写入 API history，里面包含 `tool_result` blocks。

关键原因：

```text
tool_result 必须出现在对应 tool_use 之后。
```

相关入口：

```text
src/core/task/Task.ts
src/core/task/validateToolResultIds.ts
```

`validateAndFixToolResultIds(...)` 会在写入 user message 前做防御性修复：

- 去重重复 `tool_result`。
- 如果 `tool_result.tool_use_id` 错了，按位置尝试修正。
- 如果 assistant 有 `tool_use` 但 user 缺少对应 `tool_result`，自动补：

```text
Tool execution was interrupted before completion.
```

- 如果 previous effective message 不是 assistant，则把孤立 `tool_result` 转成普通 text，避免 provider 协议错误。

## 4. CoStrict 失败处理分类

### 4.1 缺少 tool_use.id

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
```

处理方式：

- 原生工具协议要求 tool_use 必须有 ID。
- 如果没有 ID，CoStrict 认为这是无效工具调用。
- UI 展示 error。
- 向 `userMessageContent` 追加普通 text 错误，而不是 structured tool_result。
- 设置 `didAlreadyUseTool = true`。

### 4.2 参数 JSON 不完整 / nativeArgs 缺失

位置：

```text
src/core/assistant-message/NativeToolCallParser.ts
src/core/assistant-message/presentAssistantMessage.ts
src/core/tools/BaseTool.ts
```

处理方式：

- 流式阶段尽量用 partial-json 解析。
- stream end 时 `finalizeStreamingToolCall(...)` 解析完整 JSON。
- 如果解析失败，Task 会把已有 ToolUse 标成 non-partial，交给 `presentAssistantMessage`。
- `presentAssistantMessage` 发现 known tool 但没有 `nativeArgs`，不会执行工具，而是写入一个 structured `tool_result`：

```text
Invalid tool call for '<tool>': missing nativeArgs...
```

这条结果带 `is_error: true`，用于满足 provider 的 tool_use/tool_result 配对要求。

### 4.3 未知工具 / 当前 mode 不允许

位置：

```text
src/core/tools/validateToolUse.ts
src/core/assistant-message/presentAssistantMessage.ts
```

处理方式：

- complete tool block 执行前调用 `validateToolUse(...)`。
- 校验内容包括：
  - 工具名是否存在。
  - 当前 mode 是否允许。
  - disabledTools 是否禁用。
  - edit group 的 fileRegex 限制。
  - dynamic MCP tool 是否属于 MCP group。
  - includedTools/alias 是否允许。
- 校验失败后：
  - 增加 `consecutiveMistakeCount`。
  - 记录 tool error。
  - 直接 push `tool_result`，内容是 `formatResponse.toolError(...)`。
  - 不设置 `didAlreadyUseTool`，避免中断 stream。

### 4.4 用户拒绝工具执行

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
```

工具执行前，很多工具会调用 `askApproval(...)`。

用户拒绝时：

- 如果用户有反馈文本，先 `say("user_feedback", ...)`。
- 把拒绝信息写成 tool result：

```text
toolDenied()
toolDeniedWithFeedback(...)
```

- 设置：

```text
cline.didRejectTool = true
```

后续工具处理：

- 如果同一 assistant message 后面还有 tool_use，CoStrict 不再执行。
- 但为了满足 native tool protocol，会为每个后续工具补一个 `is_error: true` 的 `tool_result`：

```text
Skipping tool ... due to user rejecting a previous tool.
```

这点比 OpenCode 更显式。OpenCode 是权限拒绝后可能直接 blocked 停止 loop；CoStrict 则优先保证当前 assistant message 内所有 tool_use 都有对应 tool_result。

### 4.5 工具内部参数缺失或业务失败

位置：

```text
src/core/tools/*.ts
```

多数工具自己处理参数和业务错误，不靠统一抛异常。

例子：

- `ReadFileTool.ts`
  - path 缺失：`sayAndCreateMissingParamError(...)`，push `Error: ...`。
  - offset/anchor_line 非法：push `Error: ...`。
  - rooignore 阻止：push rooignore error。
  - 目录、二进制、读取失败：写入对应 file result。
- `ApplyPatchTool.ts`
  - patch 缺失：missing param error。
  - parsePatch 失败：push `toolError(...)`。
  - processAllHunks 失败：push `toolError(...)`。
  - 用户拒绝 diff：push “Changes were rejected by the user.”。
- `UseMcpToolTool.ts`
  - server_name/tool_name 缺失：missing param error。
  - server/tool 不存在：push unknown MCP server/tool error。
  - tool disabled：push unknown/disabled tool error。
- `ExecuteCommandTool.ts`
  - command 缺失：missing param error。
  - command 访问 rooignore 文件：push rooignore error。
  - shell integration 失败：fallback 到禁用 shell integration 再跑。
  - 非零退出码：通常不作为异常，而是把 exit code 和 output 返回给模型。
  - 用户配置超时：终止命令并返回“不要重跑该命令”的结果。

这种设计导致 CoStrict 的“失败”经常是正常 tool_result 内容，而不是抛异常。

### 4.6 工具执行异常

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
src/core/tools/BaseTool.ts
```

统一回调：

```ts
handleError(action, error)
```

处理方式：

- `AskIgnoredError` 静默忽略，这是内部控制流，表示旧 ask 被新 ask 覆盖。
- 其他异常：
  - UI 展示 `say("error", ...)`。
  - 写入 `formatResponse.toolError(...)`。
  - 通过 `pushToolResult(...)` 变成当前 tool_use 对应的 tool_result。

### 4.7 重复工具调用保护

位置：

```text
src/core/tools/ToolRepetitionDetector.ts
src/core/assistant-message/presentAssistantMessage.ts
```

逻辑：

- 对当前工具调用做稳定 JSON 序列化。
- 如果连续相同工具调用达到限制，拒绝执行。
- 向用户询问 `mistake_limit_reached`。
- 给模型返回 tool error：

```text
Tool call repetition limit reached for <tool>. Please try a different approach.
```

OpenCode 也有 doom loop 检测，但 CoStrict 是在工具执行前直接拦截，并回传 tool_result。

### 4.8 连续错误与 mistake limit

位置：

```text
src/core/task/Task.ts
```

CoStrict 维护：

```text
consecutiveMistakeCount
consecutiveMistakeLimit
SmartMistakeDetector
toolUsage[tool].attempts/failures
```

工具成功：

- `recordToolUsage(...)` 增加 attempts。
- 降低 `consecutiveMistakeCount`。
- `smartMistakeDetector.recordSuccess()`。

工具失败：

- `recordToolError(...)` 增加 failures。
- 增加 `consecutiveMistakeCount`。
- 上报 telemetry。
- 加入 `SmartMistakeDetector`。

达到阈值后：

- 向用户发 `mistake_limit_reached`。
- 用户反馈会被加入下一轮 user content。
- 对 `costrict` provider，还可能触发 model fallback。

### 4.9 没有工具调用 / 没有 assistant message

位置：

```text
src/core/task/Task.ts
```

如果模型返回文本但没有 tool_use：

- CoStrict 会追加 `formatResponse.noToolsUsed()` 给模型。
- 连续两次才展示 `MODEL_NO_TOOLS_USED`。

如果模型没有返回 assistant message：

- 连续计数。
- 第一次可静默重试。
- auto approval 开启时自动 backoff retry。
- 否则询问用户是否重试。
- 为避免连续 user message 导致 tool_result 校验异常，会先移除刚加入历史的 user message，再重试。

这属于模型输出失败，不是单个工具执行失败，但会直接影响工具调用链路。

### 4.10 new_task 隔离

位置：

```text
src/core/task/Task.ts
```

如果同一条 assistant message 同时调用 `new_task` 和后续其他工具：

- CoStrict 要求 `new_task` 必须是最后一个工具。
- 它会截断 `assistantContent` 中 `new_task` 后面的工具。
- 同时也截断 `assistantMessageContent`，防止后续工具被执行。
- 对被截断的工具预注入 `is_error: true` 的 tool_result：

```text
This tool was not executed because new_task was called in the same message turn...
```

这是为了避免父任务被 delegation dispose 后，后续工具缺失结果。

## 5. 两者失败处理机制对比

| 维度 | OpenCode CLI | CoStrict VSCode 插件 |
| --- | --- | --- |
| 工具执行触发 | AI SDK `streamText` 的 tool `execute` 回调 | `presentAssistantMessage` 读取 `assistantMessageContent` 后手动分发 |
| 工具参数校验 | `Tool.define` 统一 Zod parse | `NativeToolCallParser` 生成 `nativeArgs`，`BaseTool` 要求 nativeArgs，具体工具再校验业务参数 |
| 参数解析失败 | 抛异常，进入 `tool-error` | 生成 `is_error` tool_result，尽量不中断 stream |
| 未知工具 | `experimental_repairToolCall` 尝试修复，否则转 `invalid` 工具 | `validateToolUse` 或 default 分支返回 unknown tool 的 tool_result |
| 工具业务失败 | 多数工具 `throw Error`，统一变成 tool part error | 多数工具自己 push `toolError` 或普通错误结果 |
| 非零命令退出 | bash 工具内部决定输出/错误，失败可抛 | `execute_command` 通常把 exit code 和 output 作为 tool_result 返回，不一定算异常 |
| 权限/审批拒绝 | `Permission.RejectedError` / `Question.RejectedError`，可能 blocked 停止 loop | 当前工具返回拒绝结果，`didRejectTool = true`，后续工具补 skip error result |
| 重复工具调用 | `DOOM_LOOP_THRESHOLD = 3` 后触发 doom_loop permission | `ToolRepetitionDetector` 执行前拦截，返回 error tool_result，可询问用户 |
| 工具结果存储 | assistant message 的 tool part | 下一条 user message 的 `tool_result` block |
| tool_use/tool_result 协议修复 | `message-v2.ts` 对 pending/running 工具补 interrupted error | `pushToolResultToUserContent` 去重 + `validateAndFixToolResultIds` 修复/补齐 |
| 部分工具并行 | AI SDK/provider 支持并行；`batch` 工具内部 Promise.all | stream 中多个 tool_use 进入数组，但 `presentAssistantMessage` 按 index 串行执行 |
| 子任务/委派 | `task` tool 与 subtask part 机制；失败标 tool part error | `new_task` 工具；强制 new_task 后续工具截断并补 error result |
| Provider/API 错误重试 | `SessionRetry.policy`，按 retryable 错误重试 | Task 内部 backoff、用户确认重试、auto approval 自动重试 |
| 输出截断 | `Tool.define` 默认 `Truncate.output` | 各工具自行处理；命令输出可落 artifact，用 `read_command_output` 读取 |
| Telemetry | 主要通过 session/bus/plugin hook | toolUsage、TelemetryService、SmartMistakeDetector、PostHog exception |

## 6. 工具调用时序对比

### 6.1 OpenCode

```text
用户消息
  -> SessionPrompt.runLoop
  -> resolveTools
  -> MessageV2.toModelMessages
  -> LLM.stream / streamText
  -> model emits tool call
  -> AI SDK execute(tool args)
  -> Tool.define wrapper validates args
  -> concrete tool.execute
  -> tool-result or tool-error event
  -> SessionProcessor updates MessageV2.ToolPart
  -> finish reason tool-calls
  -> loop continues with tool result/error in history
```

### 6.2 CoStrict

```text
用户消息
  -> Task.recursivelyMakeClineRequests
  -> buildNativeToolsArrayWithRestrictions
  -> api.createMessage(metadata.tools)
  -> stream chunk: tool_call_partial / tool_call
  -> NativeToolCallParser
  -> assistantMessageContent.push(ToolUse/McpToolUse)
  -> presentAssistantMessage
  -> validateToolUse
  -> ToolRepetitionDetector
  -> BaseTool.handle
  -> concrete tool.execute
  -> pushToolResultToUserContent
  -> assistant message with tool_use saved to API history
  -> user message with tool_result saved to API history
  -> next API request
```

## 7. 关键源码路径

### OpenCode CLI

| 目的 | 路径 |
| --- | --- |
| 工具抽象与参数校验 | `packages/opencode/src/tool/tool.ts` |
| 工具注册 | `packages/opencode/src/tool/registry.ts` |
| 工具注入模型请求 | `packages/opencode/src/session/prompt.ts` |
| LLM stream 与 tool call repair | `packages/opencode/src/session/llm.ts` |
| tool part 状态机 | `packages/opencode/src/session/processor.ts` |
| 工具结果转模型历史 | `packages/opencode/src/session/message-v2.ts` |
| provider/API 错误重试 | `packages/opencode/src/session/retry.ts` |
| batch 局部容错 | `packages/opencode/src/tool/batch.ts` |
| 权限系统入口 | `packages/opencode/src/permission/` |

### CoStrict VSCode 插件

| 目的 | 路径 |
| --- | --- |
| 工具 schema 构建 | `src/core/task/build-tools.ts` |
| 原生工具声明 | `src/core/prompts/tools/native-tools/` |
| 工具过滤 | `src/core/prompts/tools/filter-tools-for-mode.ts` |
| 工具调用解析 | `src/core/assistant-message/NativeToolCallParser.ts` |
| 工具分发与失败包装 | `src/core/assistant-message/presentAssistantMessage.ts` |
| 工具基类 | `src/core/tools/BaseTool.ts` |
| 具体工具实现 | `src/core/tools/` |
| 工具权限校验 | `src/core/tools/validateToolUse.ts` |
| tool_result ID 修复 | `src/core/task/validateToolResultIds.ts` |
| tool_result 入队去重 | `src/core/task/Task.ts` |
| 重复工具调用检测 | `src/core/tools/ToolRepetitionDetector.ts` |
| MCP 工具执行 | `src/core/tools/UseMcpToolTool.ts` |
| 命令工具特殊处理 | `src/core/tools/ExecuteCommandTool.ts` |

## 8. 设计取舍分析

### 8.1 OpenCode 的取舍

优点：

- 架构更短，工具执行与模型流事件天然绑定。
- 工具实现只需要抛异常或返回结果，统一状态机负责持久化。
- `Tool.define` 提供统一 schema 校验和输出截断。
- 插件 hook 可以在 before/after 做横切扩展。

不足：

- 普通工具失败主要靠异常传播，缺少 CoStrict 那样细粒度的 `tool_result` 协议修复层。
- 对多工具同轮复杂协议问题的防御主要依赖 AI SDK 和 `message-v2.ts` 的 pending/running 兜底。
- 用户拒绝后更偏“停止/blocked”，不是补齐当前所有未执行工具结果。

### 8.2 CoStrict 的取舍

优点：

- 对 VSCode 交互场景更稳，用户审批、diff 预览、终端输出、MCP 状态都能精细反馈。
- 对 native tool calling 的协议边界处理很强，尤其是 tool_use/tool_result ID 去重、修复和补齐。
- 工具失败多数能反馈给模型并继续下一轮，而不是让外层流直接崩。
- 有连续错误、重复调用、no tool use、no assistant message 等多层恢复策略。

不足：

- 调用链更长：parser、Task、presentAssistantMessage、BaseTool、具体工具、history validator 都参与同一次工具调用。
- 错误处理分散在具体工具中，不同工具对“失败”的定义不完全一致。
- 有些失败会作为普通成功 tool_result 返回，调用方需要看内容判断是否失败。
- 错误计数存在多处递增路径，维护时需要小心避免重复计数。

## 9. 迁移或复用建议

如果要把两边机制互相借鉴：

1. 从 CoStrict 迁到 OpenCode：
   - 可以借鉴 `validateAndFixToolResultIds` 的协议兜底思想，尤其是 pending/running tool call 中断后的完整配对检查。
   - 可以借鉴 `ToolRepetitionDetector`，在工具执行前直接拦截重复调用，而不仅是触发 permission。
   - 可以借鉴 `new_task` 隔离策略，防止委派任务后同轮后续工具产生孤儿结果。

2. 从 OpenCode 迁到 CoStrict：
   - 可以借鉴 `Tool.define` 的统一参数校验与输出截断，把具体工具里重复的 missing param / truncate / result 包装收敛。
   - 可以借鉴 `tool.execute.before/after` hook，让工具执行横切逻辑更集中。
   - 可以借鉴 OpenCode 的 session part 状态机，把 UI 展示状态和模型历史状态拆得更清楚。

3. 做稳定性增强时优先关注：
   - malformed tool args。
   - unknown tool / alias mismatch。
   - 用户拒绝后同轮后续工具的结果补齐。
   - 工具执行中 abort。
   - 多工具同轮、重复 tool_use_id、缺失 tool_result。
   - 子任务/委派导致父任务提前释放。

## 10. 一句话对比

OpenCode 的工具失败处理是“执行异常进入统一 session 状态机，再由下一轮模型读取错误状态”；CoStrict 的工具失败处理是“尽量在插件层把每个失败都包装成合法 tool_result，确保 native tool protocol 不断裂，并通过 UI、telemetry、错误计数引导恢复”。

## 11. 逻辑漏洞排查与加固建议

本节目标是降低两类问题的发生概率：

1. 死循环：模型反复调用同一工具、同一失败参数、同一被拒绝动作，或者反复产生无工具/坏工具调用。
2. 工具调用失败放大：一次工具解析、审批、执行、历史写入失败，进一步导致 provider 400、tool_use/tool_result 协议断裂、任务挂起或重复重试。

这里的“漏洞”指架构风险点和防线薄弱点，不等同于已确认线上 bug。

### 11.1 排查口径

建议用五层防线排查工具调用逻辑：

| 层级 | 关注点 | 失败后果 |
| --- | --- | --- |
| 工具暴露层 | 模型看到的工具是否和实际可执行工具一致 | 模型调用不存在或当前模式不允许的工具 |
| 解析校验层 | 工具名、参数 JSON、schema、mode 权限是否提前拦截 | 工具还没执行就失败，或错误工具被执行 |
| 执行层 | 权限审批、超时、abort、业务异常是否稳定返回 | 卡住、重复审批、工具结果缺失 |
| 结果回写层 | tool_result 是否唯一、配对、排序正确 | provider 400、历史污染、下一轮无法继续 |
| 恢复控制层 | 重复失败是否有预算、熔断、用户确认 | 模型持续重试同一失败路径 |

### 11.2 OpenCode 高风险排查点

#### OC-1：Doom loop 检测只覆盖“最近 3 次完全相同工具 + 完全相同输入”

位置：

```text
packages/opencode/src/session/processor.ts
```

当前保护：

- `DOOM_LOOP_THRESHOLD = 3`。
- 最近 3 个 tool part 都是同一工具、同一 `JSON.stringify(input)`，才触发 `doom_loop` 权限询问。

风险：

- A/B/A/B 交替工具循环检测不到。
- 同一语义参数但 key 顺序不同、路径等价写法不同、默认参数显式/隐式不同，检测不到。
- 同一工具连续失败但参数有微小变化，例如 edit 每次多加一点上下文，可能长期循环。
- 如果用户/配置允许 doom_loop，后续仍可能继续。

建议：

- 引入 `toolCallFingerprint`：工具名 + 规范化参数 + 失败类型 + 目标文件/命令。
- 维护滑动窗口，不只看最近连续 3 次。
- 对失败工具单独计数，例如同一 `tool + args_hash + error_code` 连续失败 2 次后，强制要求模型换策略或询问用户。
- 对 edit/apply_patch/read/grep/command 这类高频工具设置不同阈值。

#### OC-2：未知工具修复为 `invalid` 后缺少失败预算

位置：

```text
packages/opencode/src/session/llm.ts
```

当前保护：

- `experimental_repairToolCall(...)` 会尝试把工具名转小写。
- 修复不了时，把调用改成 `invalid` 工具。

风险：

- 如果模型反复输出同一个未知工具，系统会不断生成 `invalid` 工具结果，让模型继续自我修正。
- `invalid` 能把错误反馈给模型，但没有“同一未知工具最多修复 N 次”的熔断。

建议：

- 为 repair 增加计数：`unknown_tool_name + model + session`。
- 同一未知工具连续 2 次后，不再继续自动 loop，改为 assistant 明确说明可用工具或触发用户确认。
- `invalid` 输出中加入可用工具候选和“不要再次调用该工具”的机器可读字段。

#### OC-3：权限拒绝后的继续策略可能再次触发同类工具

位置：

```text
packages/opencode/src/session/processor.ts
packages/opencode/src/session/prompt.ts
```

当前保护：

- `Permission.RejectedError` / `Question.RejectedError` 会被识别。
- 默认 `experimental.continue_loop_on_deny !== true` 时会 blocked。

风险：

- 如果开启 `continue_loop_on_deny`，模型可能在下一轮换一种参数继续调用被拒绝的工具。
- OpenCode 没有像 CoStrict 一样给同一 assistant message 后续工具补“skipped due to rejection”的结果，因为它更依赖流和 session 状态。

建议：

- 用户拒绝后生成结构化 deny marker，包含 `tool`、`permission`、`patterns`、`scope`。
- 下一轮 prompt 注入短期规则：同一 scope 的工具不得再次调用，除非用户显式改变决定。
- 对 deny 后继续 loop 的场景，最多允许一次模型解释/改计划，不允许直接再次执行高风险工具。

#### OC-4：`batch` 工具重新取 registry，可能与当前模型可用工具集不一致

位置：

```text
packages/opencode/src/tool/batch.ts
packages/opencode/src/tool/registry.ts
```

当前逻辑：

- `batch` 内部调用 `ToolRegistry.tools({ modelID: "", providerID: "" })`。
- 子工具执行失败会被 batch 局部捕获，并写入独立 tool part。

风险：

- 它没有直接复用当前 `SessionPrompt.resolveTools(...)` 已经算好的 active tool map。
- 当前 provider/model 下可用工具与 batch 内部重新算出的工具可能不一致。
- 子工具执行没有走 `SessionPrompt.resolveTools(...)` 外层的 `tool.execute.before/after` 包装，插件 hook 和部分横切逻辑可能不一致。
- 如果未来 registry 过滤规则增加，batch 更容易和主工具列表漂移。

建议：

- batch 执行时传入当前 resolved tool map，而不是重新用空 model/provider 获取 registry。
- 子工具也走统一 before/after hook。
- batch 输出中为每个失败子工具加入 `error_code`、`retryable`、`suggested_next_action`。
- 对 batch 中同类失败做合并，避免 25 个相同错误污染上下文。

#### OC-5：工具错误多为自由文本，缺少统一错误码

位置：

```text
packages/opencode/src/tool/*.ts
packages/opencode/src/session/processor.ts
```

当前逻辑：

- 工具失败多为 `throw new Error("...")`。
- session 最终保存 `state.error = error.message`。

风险：

- 模型难以稳定判断“是否应该重试同一参数”。
- 同类错误文本不一致，会削弱 doom loop/failure budget 判断。
- 例如 patch no hunk、file not found、permission denied、timeout，本质恢复策略不同，但都只是文本。

建议：

- 定义统一 `ToolExecutionError`：

```ts
{
  code: "FILE_NOT_FOUND" | "PATCH_NO_MATCH" | "PERMISSION_DENIED" | "TIMEOUT" | ...
  message: string
  retryable: boolean
  sameArgsRetryable: boolean
  suggestedNextAction?: string
}
```

- `SessionProcessor` 保存结构化 error metadata。
- 给模型的 tool error 文案保持简洁，但保留机器可读 code。

#### OC-6：中断清理错误过于泛化

位置：

```text
packages/opencode/src/session/processor.ts
packages/opencode/src/session/message-v2.ts
```

当前保护：

- pending/running 工具清理时写：

```text
Tool execution aborted
[Tool execution was interrupted]
```

风险：

- abort、provider stream error、用户取消、工具超时、进程崩溃都会被压成类似信息。
- 模型可能不知道应该重试、询问用户，还是放弃。

建议：

- 区分 `aborted_by_user`、`stream_error`、`tool_timeout`、`permission_denied`、`session_compacted`。
- 对用户主动 abort，下一轮不应自动重试同一工具。
- 对 provider stream error，可以允许一次 retry，但不要重复执行已产生副作用的工具。

### 11.3 CoStrict 高风险排查点

#### CS-1：工具名 substring 解析可能误匹配

位置：

```text
src/core/assistant-message/NativeToolCallParser.ts
```

当前逻辑里存在类似匹配：

```text
name === resolvedName || resolvedName.indexOf(name) > -1
```

风险：

- `read_file_extra`、`my_read_file` 这类异常名称可能被解析成 `read_file`。
- 模型输出拼接污染的工具名时，有机会执行错误工具。
- 这会降低 unknown tool 防线的价值。

建议：

- 工具名解析只允许：
  - 精确匹配。
  - 显式 alias 表匹配。
  - 动态 MCP 工具名按固定格式解析。
  - custom tool 精确匹配。
- 删除 substring fuzzy matching。
- 对“近似工具名”只返回 unknown tool error，并给出候选，不执行。

#### CS-2：错误计数可能被重复递增

位置：

```text
src/core/task/Task.ts
src/core/tools/*.ts
```

当前现象：

- 多个工具里先手动执行：

```ts
task.consecutiveMistakeCount++
task.recordToolError(...)
```

- 但 `recordToolError(...)` 内部也会 `consecutiveMistakeCount++`。

风险：

- 单次工具失败可能计为两次 mistake。
- 过早触发 `mistake_limit_reached`。
- 误判模型质量或触发 model fallback。

建议：

- 规定只有 `recordToolError(...)` 修改 mistake count。
- 工具内部不直接写 `consecutiveMistakeCount++`。
- 如果确实需要不同权重，用 `recordToolError(tool, error, { weight: 2 })` 这种显式接口。
- 增加单测：每类失败只增加预期次数。

#### CS-3：`userMessageContentReady` 等待缺少硬超时

位置：

```text
src/core/task/Task.ts
src/core/assistant-message/presentAssistantMessage.ts
```

当前逻辑：

```text
await pWaitFor(() => this.userMessageContentReady)
```

风险：

- 如果某个工具没有调用 `pushToolResult`，也没有正确推进 `currentStreamingContentIndex`，主循环可能卡住。
- 如果 `presentAssistantMessageLocked` 因异常路径未释放，也可能挂起。
- 这类问题会表现为“任务一直转圈”，比明确失败更难排查。

建议：

- 给等待加超时，例如 60s 或按工具类型配置。
- 超时后：
  - 找出所有未配对 tool_use。
  - 注入 `is_error: true` 的 tool_result。
  - 释放 lock。
  - 记录 telemetry 和 raw state。
- 对长时间运行命令例外处理，必须有 execution status 或后台 artifact。

#### CS-4：partial tool 处理失败可能提前产生 tool_result

位置：

```text
src/core/tools/BaseTool.ts
src/core/assistant-message/presentAssistantMessage.ts
```

当前逻辑：

- partial block 会调用 `handlePartial(...)`。
- `handlePartial` 抛错时，`BaseTool` 调用 `callbacks.handleError(...)`。
- `handleError(...)` 会 push tool result。

风险：

- partial 阶段参数还没完整，理论上不应该产生最终 tool_result。
- 如果 partial 阶段先写了错误结果，complete 阶段再执行时会遇到 duplicate tool_result，被跳过或污染结果。

建议：

- partial 阶段错误只写 UI/debug log，不写模型 `tool_result`。
- 只有 complete block 才允许写 tool_result。
- 如果 partial 错误严重到不能继续，应标记该 tool call final failure，并阻止 complete 再执行。

#### CS-5：`validateAndFixToolResultIds` 会修复协议，但也可能掩盖根因

位置：

```text
src/core/task/validateToolResultIds.ts
```

当前保护：

- 去重重复 tool_result。
- 修正错误 tool_use_id。
- 自动补缺失 tool_result。

风险：

- 自动补 “Tool execution was interrupted before completion.” 可以避免 provider 400，但模型下一轮可能继续重试同一工具。
- 如果频繁进入修复逻辑，说明上游 parser/dispatcher/history 顺序有 bug，但任务仍会继续，根因可能被掩盖。

建议：

- 对每次修复写入 `protocol_repair_count`。
- 同一 task 超过阈值后停止自动继续，要求用户/开发日志介入。
- 修复补齐的 tool_result 应带更明确的 code，例如 `TOOL_RESULT_MISSING_AUTOFIXED`。
- 在 telemetry 中记录 assistant tool_use IDs、user tool_result IDs、currentStreamingContentIndex、didRejectTool、didAlreadyUseTool。

#### CS-6：工具失败的 `is_error` 标记不一致

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
src/core/tools/*.ts
```

当前现象：

- unknown tool、validation error 等直接写 `is_error: true`。
- 但很多工具通过 `pushToolResult(formatResponse.toolError(...))` 返回错误文本时，不一定设置 `is_error: true`。
- 命令非零退出码通常作为普通结果返回。

风险：

- provider 和模型只看到文本，不一定能区分成功/失败。
- 失败预算、telemetry、下一轮策略更难统一。

建议：

- `pushToolResult` 支持结构化参数：

```ts
pushToolResult({
  content,
  isError,
  errorCode,
  retryable,
})
```

- `formatResponse.toolError(...)` 默认映射为 `is_error: true`。
- 对命令非零退出码至少设置 `errorCode = COMMAND_NON_ZERO_EXIT`，但可标注 `retryable` 由模型判断。

#### CS-7：用户拒绝一个工具后跳过同轮后续所有工具，可能过度抑制

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
src/core/tools/ReadFileTool.ts
src/core/tools/ExecuteCommandTool.ts
```

当前保护：

- `didRejectTool = true` 后，同一 assistant message 后续工具不执行，但会补 skipped tool_result。

优点：

- 避免用户已经拒绝危险操作后，模型继续执行同一轮后续工具。
- 保证 native tool protocol 配对完整。

风险：

- 用户拒绝一个读文件或某个子文件后，后续无害工具也会被跳过。
- 模型下一轮可能重新发起一组工具，造成“拒绝 -> 跳过 -> 重试”的循环。

建议：

- 拒绝结果带 scope：`tool`、`resource`、`permission type`。
- 后续工具如果明显无关且只读，可选择继续；高风险工具继续跳过。
- 下一轮 prompt 明确写入“用户拒绝的是 X，不要再次请求 X；如需替代方案请说明”。

#### CS-8：custom tools 当前模式限制偏弱

位置：

```text
src/core/task/build-tools.ts
src/core/tools/validateToolUse.ts
```

当前逻辑：

- `experiments.customTools` 开启后加载 custom tools。
- `validateToolUse` 中 custom tool 基本允许所有 mode。

风险：

- 自定义工具可能绕过 mode 的读/写/命令边界。
- 模型在受限模式下仍能调用高风险 custom tool。

建议：

- custom tool manifest 增加：

```text
groups
allowedModes
requiresApproval
sideEffects
```

- `validateToolUse` 对 custom tool 应按 manifest 校验。
- 没有 manifest 的 custom tool 默认只允许最低风险组，或必须显式 includedTools。

#### CS-9：MCP 校验异常时 fail-open

位置：

```text
src/core/tools/UseMcpToolTool.ts
```

当前逻辑：

- `validateToolExists(...)` 内部校验 MCP server/tool。
- 如果校验过程本身异常，会 log 后返回 `{ isValid: true }`，继续执行。

风险：

- MCP hub 状态异常时，本该 fail-fast 的问题会推迟到执行期。
- 执行期错误更难给出可用工具列表和恢复建议。

建议：

- 区分两类异常：
  - MCP hub 暂不可用：返回明确 tool error。
  - 校验列表读取失败但 server/tool 已知：允许谨慎继续。
- fail-open 需要 telemetry 标记，连续出现则停用 MCP tool 并提示用户检查 server。

#### CS-10：同一响应内多个 malformed tool call 缺少 per-turn 熔断

位置：

```text
src/core/assistant-message/presentAssistantMessage.ts
src/core/assistant-message/NativeToolCallParser.ts
```

当前保护：

- 每个 malformed known tool 会得到一个 error tool_result。

风险：

- 如果模型一次输出很多 malformed tool calls，系统会逐个补 error result，污染上下文。
- 下一轮模型可能根据大量错误继续生成坏调用。

建议：

- 每个 assistant message 设置 malformed tool call 上限，例如 3。
- 超过上限后，后续工具统一补一个简短错误，并要求模型重新生成有效工具调用。
- 将 parser error 合并摘要化，不要把大量重复错误全部塞进上下文。

### 11.4 共同死循环模式

| 模式 | 典型触发 | 建议防线 |
| --- | --- | --- |
| 同工具同参数重复失败 | edit 找不到 old_string、patch hunk 失败、read 路径不存在 | `tool + args_hash + error_code` 失败预算 |
| 工具名反复错误 | 模型持续调用不存在工具或旧 alias | 精确 alias 表、unknown tool 熔断、候选工具提示 |
| 参数 JSON 反复畸形 | stream 丢 chunk、模型输出半截 JSON | malformed tool per-turn 上限、parser telemetry |
| 用户拒绝后模型继续尝试 | permission denied、diff rejected、command denied | deny scope 写入下一轮 prompt，短期禁用同 scope |
| provider 协议断裂 | tool_use 缺 result、重复 tool_use_id、孤立 tool_result | preflight 校验 + 自动修复 + 修复次数熔断 |
| 长命令/长工具挂起 | command 不退出、MCP async 卡住、审批 promise 被覆盖 | 工具级 timeout、后台 artifact、ready 等待超时 |
| 压缩/截断后信息丢失 | tool error 被压缩成泛化文本 | 保留结构化 error code 和最近失败摘要 |

### 11.5 推荐的统一加固方案

建议两边都引入一个工具调用安全层，职责不要散落在具体工具中：

```text
ToolCallGuard
  -> normalizeToolName
  -> normalizeArgs
  -> validateActiveTool
  -> validateModeAndPermission
  -> checkFailureBudget
  -> executeWithTimeout
  -> normalizeResult
  -> recordTelemetry
  -> repairProtocolIfNeeded
```

核心数据结构建议：

```ts
type ToolFailureRecord = {
  sessionID: string
  messageID: string
  callID: string
  tool: string
  argsHash: string
  errorCode: string
  retryable: boolean
  sameArgsRetryable: boolean
  countInTurn: number
  countInSession: number
}
```

关键策略：

- 同一 `tool + argsHash + errorCode` 连续失败 2 次：禁止同参数自动重试。
- 同一 assistant message malformed tool call 超过 3 次：停止执行后续工具，要求模型重新生成。
- 同一 session unknown tool 超过 2 次：注入可用工具列表并停止当前 loop。
- 用户拒绝同 scope 后：本轮跳过同 scope，下一轮禁止同 scope 自动执行。
- tool_result 协议自动修复超过 1 次：继续前写入 telemetry；超过 3 次停止任务并提示协议异常。

### 11.6 优先级建议

| 优先级 | 建议 | 主要收益 |
| --- | --- | --- |
| P0 | CoStrict 删除工具名 substring fuzzy match，只保留精确匹配和显式 alias | 防止错误工具被误执行 |
| P0 | CoStrict 给 `pWaitFor(() => userMessageContentReady)` 加超时和兜底 tool_result | 防止任务无反馈挂死 |
| P0 | 两边增加 `tool + args_hash + error_code` 失败预算 | 直接降低同参数死循环 |
| P1 | OpenCode batch 复用当前 resolved tool map，并补齐 before/after hook | 避免 batch 与主工具集漂移 |
| P1 | CoStrict 统一 `recordToolError`，避免 mistake count 双重递增 | 降低误触发 mistake limit |
| P1 | 两边统一结构化工具错误码和 retryable 标记 | 让模型更容易选择下一步 |
| P1 | CoStrict partial 阶段禁止写最终 tool_result | 避免 partial/final 重复结果 |
| P2 | 协议修复次数熔断和开发日志快照 | 避免修复层掩盖根因 |
| P2 | custom tool manifest 加 mode/sideEffects/approval 元数据 | 降低自定义工具越权 |

### 11.7 建议补充的测试矩阵

| 场景 | OpenCode 测试点 | CoStrict 测试点 |
| --- | --- | --- |
| 未知工具连续 3 次 | repair 到 `invalid` 后是否熔断 | unknown tool 是否只产生配对 tool_result，且计数正确 |
| 同一 edit 参数连续失败 | doom loop/failure budget 是否触发 | ToolRepetitionDetector + error budget 是否触发 |
| 用户拒绝命令后模型继续调用命令 | 是否 blocked 或禁止同 scope | 后续工具是否补 skipped result，下一轮是否禁止同 scope |
| 工具执行中 abort | pending/running 是否变 interrupted error | 是否补缺失 tool_result，任务是否不挂起 |
| duplicate tool_use_id | AI SDK/history 是否能避免协议错误 | preflight dedup + validateAndFix 是否有效 |
| 缺失 tool_result | message-v2 是否补 interrupted | validateAndFix 是否补齐并上报 |
| malformed JSON tool args | repair/invalid 是否给出可恢复错误 | missing nativeArgs 是否单次结果且不重复 |
| batch/new_task 同轮多工具 | batch 子工具结果和 hook 是否一致 | new_task 后续工具是否截断并补 result |
| MCP server/tool 不存在 | MCP tool 是否 fail-fast | unknown MCP server/tool 是否有候选列表 |
| 长命令超时 | bash/abort 状态是否清晰 | user timeout、agent timeout、artifact 是否稳定 |

### 11.8 最小落地路径

如果只做一轮低风险改造，建议顺序如下：

1. 先加观测：记录 `tool`、`args_hash`、`error_code`、`tool_call_id`、`repair_count`。
2. 再加熔断：同一失败签名重复时，不再让模型无条件继续。
3. 再统一错误格式：让模型看到明确的 `retryable` 和 `sameArgsRetryable`。
4. 最后收敛工具实现：把参数缺失、权限拒绝、业务失败、异常捕获从具体工具中逐步迁到统一 wrapper。

这样可以先降低死循环和挂起概率，同时不大规模重构现有工具实现。
