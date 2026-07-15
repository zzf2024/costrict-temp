# OpenCode / CoStrict 上下文压缩功能说明

本文基于 2026-07-15 浅克隆源码检查结果整理：

- `zgsm-sangfor/opencode`：`dev` 分支，commit `b158c71902a635994f578da65c7f824d391f7a3d`
- `zgsm-ai/costrict`：`main` 分支，commit `ffb7485b775925ecc3247d6bac735f4408f0c03a`

结论：两个仓库都有压缩上下文能力。`opencode` 使用 `compaction` 命名，产品入口是 `/compact`、`/summarize` 和 `POST /session/:sessionID/summarize`。`costrict` 使用 `condense` / `context management` 命名，产品入口是 VS Code Webview 内部消息 `condenseTaskContextRequest` 和设置项 `autoCondenseContext`。

## 术语对照

| 项目 | 功能名 | 手动入口 | 自动入口 | 主要产物 |
| --- | --- | --- | --- | --- |
| `zgsm-sangfor/opencode` | `compaction` | `/compact`、`/summarize`、`POST /session/:sessionID/summarize` | `compaction.auto` 开启后，达到上下文上限前自动触发 | `message.part` 的 `type: "compaction"`，以及 `summary: true, mode: "compaction"` 的 assistant 摘要消息 |
| `zgsm-ai/costrict` | `condense context` / `context management` | Webview 发 `condenseTaskContextRequest`，后端执行 `Task.condenseContext()` | `autoCondenseContext` 开启后，达到阈值触发 `manageContext()` | UI 消息 `say: "condense_context"`，API 历史里的 `Summary` 消息 |

## OpenCode 使用方法

### 1. TUI / 前端手动压缩

在会话中执行：

```text
/compact
```

或别名：

```text
/summarize
```

TUI 命令定义在：

- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/cli/cmd/tui/routes/session/index.tsx#L464-L474>

Web App 的会话命令同样暴露 `session.compact`，调用 SDK：

```ts
await sdk.client.session.summarize({
  sessionID,
  modelID: model.id,
  providerID: model.provider.id,
})
```

源码位置：

- `packages/app/src/pages/session/use-session-commands.tsx`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/app/src/pages/session/use-session-commands.tsx#L329-L344>

Native App prompt input 里也注册了 `compact` 触发器：

- `packages/app-ai-native/src/components/prompt-input.tsx`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/app-ai-native/src/components/prompt-input.tsx#L580-L583>

### 2. 配置自动压缩

在 `opencode.json` 中配置：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 10000
  }
}
```

字段含义：

- `auto`：当上下文接近满载时自动压缩会话，默认 `true`。
- `prune`：清理旧工具输出，节省 token，默认 `true`。
- `reserved`：为压缩过程预留的 token 缓冲区，避免压缩本身溢出上下文窗口。

文档源码：

- `packages/web/src/content/docs/zh-cn/config.mdx`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/web/src/content/docs/zh-cn/config.mdx#L481-L498>

配置 schema：

- `packages/opencode/src/config/config.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/config/config.ts#L1085-L1095>

### 3. HTTP API

`SessionRoutes` 挂载在 `/session`：

- `packages/opencode/src/server/instance.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/server/instance.ts#L46>

手动压缩接口：

```http
POST /session/:sessionID/summarize
Content-Type: application/json

{
  "providerID": "anthropic",
  "modelID": "claude-sonnet-4-5",
  "auto": false
}
```

请求体 schema：

```ts
{
  providerID: ProviderID
  modelID: ModelID
  auto?: boolean // default false
}
```

返回：

```json
true
```

实现位置：

- `packages/opencode/src/server/routes/session.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/server/routes/session.ts#L496-L553>

接口内部流程：

1. 读取 `sessionID` 和请求体中的 `providerID/modelID/auto`。
2. `SessionRevert.cleanup(session)` 清理 revert 状态。
3. 找到最后一个 user message，推断当前 agent。
4. 调用 `SessionCompaction.create({ sessionID, agent, model, auto })`。
5. 调用 `SessionPrompt.loop({ sessionID })` 继续会话循环。
6. 返回 `true`。

### 4. 自动压缩执行链路

关键溢出判断：

- `packages/opencode/src/session/overflow.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/overflow.ts#L8-L22>

核心逻辑：

```ts
if (input.cfg.compaction?.auto === false) return false

const reserved =
  input.cfg.compaction?.reserved ??
  Math.min(COMPACTION_BUFFER, ProviderTransform.maxOutputTokens(input.model))

return count >= usable
```

也就是说：

- `compaction.auto === false` 会禁用自动压缩。
- 没有显式配置 `reserved` 时，会用 `20_000` 和模型最大输出 token 中较小者作为缓冲。
- token 使用量达到可用上下文窗口上限时，触发自动压缩。

会话循环中的自动触发点：

- `packages/opencode/src/session/prompt.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/prompt.ts#L1409-L1415>

当上一条 assistant 消息不是 summary 且 `compaction.isOverflow(...)` 返回 true 时，会创建一个 `auto: true` 的 compaction part：

```ts
yield* compaction.create({
  sessionID,
  agent: lastUser.agent,
  model: lastUser.model,
  auto: true,
})
```

如果 provider 调用过程中直接返回 `compact`，也会创建自动压缩任务，并记录 `overflow`：

- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/prompt.ts#L1541-L1548>

### 5. 压缩服务源码

核心服务：

- `packages/opencode/src/session/compaction.ts`
- 固定源码链接：<https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts>

主要方法：

| 方法 | 作用 |
| --- | --- |
| `isOverflow()` | 判断当前 token 用量是否应该触发自动压缩 |
| `create()` | 新增一个 user message 和 `type: "compaction"` 的 part，作为压缩任务边界 |
| `process()` | 使用 compaction agent 调用模型，总结前文并生成续接摘要 |
| `prune()` | 压缩完成后清理旧工具输出，减少持久化和后续上下文体积 |

`create()` 写入的 part 结构：

```ts
{
  type: "compaction",
  auto: input.auto,
  overflow: input.overflow
}
```

源码链接：

- <https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts#L344-L366>

`process()` 会生成 `summary: true` 的 assistant 消息：

```ts
{
  role: "assistant",
  mode: "compaction",
  agent: "compaction",
  summary: true
}
```

源码链接：

- <https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts#L214-L231>

### 6. 插件扩展点

压缩摘要生成前会触发插件 hook：

```ts
experimental.session.compacting
```

插件可以：

- 向默认压缩 prompt 注入额外上下文。
- 直接替换整个压缩 prompt。

源码链接：

- <https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts#L178-L214>

压缩完成后会发布事件：

```ts
session.compacted
```

源码链接：

- <https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts#L28-L29>
- <https://github.com/zgsm-sangfor/opencode/blob/b158c71902a635994f578da65c7f824d391f7a3d/packages/opencode/src/session/compaction.ts#L333>

## CoStrict 使用方法

### 1. 设置页自动压缩

CoStrict 在设置页暴露了上下文管理配置：

- 自动触发智能上下文压缩：`autoCondenseContext`
- 触发智能上下文压缩的阈值：`autoCondenseContextPercent`
- 上下文压缩 API 配置：`condensingApiConfiguration`
- 自定义上下文压缩提示词：`customCondensingPrompt`

中文文案位置：

- `webview-ui/src/i18n/locales/zh-CN/settings.json`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/webview-ui/src/i18n/locales/zh-CN/settings.json#L623-L644>

设置 UI 组件：

- `webview-ui/src/components/settings/ContextManagementSettings.tsx`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/webview-ui/src/components/settings/ContextManagementSettings.tsx#L477-L488>

手动设置建议：

- 长会话、模型上下文较小：开启 `autoCondenseContext`，阈值设置为 `70` 到 `90`。
- 希望尽量保留完整上下文：阈值可调高，但过高会增加请求溢出风险。
- 使用输出很长的模型：阈值应更保守，因为 CoStrict 会预留 `maxTokens/modelMaxTokens/settingsMaxTokens`。

### 2. 手动压缩入口

Webview 前端手动压缩时发送内部消息：

```ts
vscode.postMessage({
  type: "condenseTaskContextRequest",
  text: taskId,
})
```

源码位置：

- `webview-ui/src/components/chat/ChatView.tsx`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/webview-ui/src/components/chat/ChatView.tsx#L1904-L1913>

Extension 后端接收消息并调用 provider：

- `src/core/webview/webviewMessageHandler.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/webview/webviewMessageHandler.ts#L1034-L1036>

Provider 再调用任务实例：

- `src/core/webview/ClineProvider.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/webview/ClineProvider.ts#L2346-L2358>

最终执行：

- `src/core/task/Task.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/task/Task.ts#L1929-L2004>

手动压缩核心行为：

1. `flushPendingToolResultsToHistory()`：先把 pending tool result 写入历史，避免 `tool_use/tool_result` 不成对。
2. 获取系统提示词、当前 mode、API 配置、自定义压缩提示词。
3. 构造工具 metadata，保证压缩请求也知道当前工具定义。
4. 生成环境信息 `getEnvironmentDetails(this, true)`。
5. 获取任务中读过的文件列表 `getFilesReadByRooSafely("condenseContext")`，传给摘要过程。
6. 调用 `summarizeConversation({ isAutomaticTrigger: false, ... })`。
7. 成功后覆盖 API conversation history，并发出 `say: "condense_context"` UI 消息。

### 3. 自动压缩执行链路

CoStrict 的自动压缩在每次 API 请求前检查上下文：

- `src/core/task/Task.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/task/Task.ts#L4554-L4630>

运行时读取：

```ts
autoCondenseContext = true
autoCondenseContextPercent = 90
profileThresholds = {}
```

然后调用 `willManageContext(...)` 判断是否即将进入压缩/截断流程。若预计会运行且自动压缩开启，会先给 Webview 发：

```ts
{
  type: "condenseTaskContextStarted",
  text: taskId
}
```

真正的上下文管理调用点：

- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/task/Task.ts#L4690-L4728>

```ts
const truncateResult = await manageContext({
  messages: this.apiConversationHistory,
  totalTokens: contextTokens,
  maxTokens,
  contextWindow,
  apiHandler: this.api,
  autoCondenseContext,
  autoCondenseContextPercent,
  systemPrompt,
  taskId: this.taskId,
  customCondensingPrompt,
  profileThresholds,
  currentProfileId,
  metadata: contextMgmtMetadata,
  environmentDetails: contextMgmtEnvironmentDetails,
  filesReadByRoo: contextMgmtFilesReadByRoo,
  cwd: this.cwd,
  rooIgnoreController: this.rooIgnoreController,
  modelMaxTokens: modelInfo.maxTokens,
  settingsMaxTokens: this.apiConfiguration.modelMaxTokens,
})
```

`manageContext()` 的核心逻辑：

- `src/core/context-management/index.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/context-management/index.ts#L278-L417>

关键行为：

1. 计算 `reservedTokens = max(maxTokens, modelMaxTokens, settingsMaxTokens, ANTHROPIC_DEFAULT_MAX_TOKENS)`。
2. 计算当前有效 token：`prevContextTokens = totalTokens + lastMessageTokens`。
3. 计算可用上限：`allowedTokens = contextWindow * 0.9 - reservedTokens`。
4. 若 `autoCondenseContext` 开启，且满足以下任一条件，则调用摘要压缩：
   - `contextPercent >= effectiveThreshold`
   - `prevContextTokens > allowedTokens`
5. 摘要成功则返回压缩后的 messages 和 summary。
6. 摘要失败但 token 仍超出 `allowedTokens`，退回 sliding-window truncation，隐藏部分旧消息。
7. 没达到阈值则返回原消息。

自动压缩调用 `summarizeConversation()` 时会设置：

```ts
isAutomaticTrigger: true
```

源码链接：

- <https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/context-management/index.ts#L348-L364>

### 4. 内部接口和消息结构

CoStrict 当前没有发现类似 `opencode` 的公开 HTTP 压缩接口；它主要是 VS Code Extension 内部 Webview 消息和任务方法。

前端发给 Extension：

```ts
{
  type: "condenseTaskContextRequest",
  text: taskId
}
```

Extension 发给前端：

```ts
{
  type: "condenseTaskContextStarted",
  text: taskId
}
```

```ts
{
  type: "condenseTaskContextResponse",
  text: taskId
}
```

Webview 对这两个响应的处理：

- `webview-ui/src/components/chat/ChatView.tsx`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/webview-ui/src/components/chat/ChatView.tsx#L1061-L1080>

压缩完成后通过 `say("condense_context", ..., contextCondense)` 写入 UI 消息。

`ContextCondense` schema：

```ts
{
  cost: number
  prevContextTokens: number
  newContextTokens: number
  summary: string
  condenseId?: string
}
```

源码位置：

- `packages/types/src/message.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/packages/types/src/message.ts#L199-L219>

### 5. 摘要生成源码

摘要入口：

- `src/core/condense/index.ts`
- 固定源码链接：<https://github.com/zgsm-ai/costrict/blob/ffb7485b775925ecc3247d6bac735f4408f0c03a/src/core/condense/index.ts#L256>

从调用侧看，它支持：

- `messages`
- `apiHandler`
- `systemPrompt`
- `taskId`
- `isAutomaticTrigger`
- `customCondensingPrompt`
- `metadata`
- `environmentDetails`
- `filesReadByRoo`
- `cwd`
- `rooIgnoreController`

手动和自动压缩都会走这个函数，但手动压缩直接调用 `summarizeConversation()`，自动压缩先经过 `manageContext()` 的阈值判断和失败回退逻辑。

## 两者实现差异

### 对外接口

- `opencode` 有明确 HTTP API：`POST /session/:sessionID/summarize`，也有 SDK 方法 `sdk.client.session.summarize(...)`。
- `costrict` 没发现公开 HTTP 压缩 API，主要通过 VS Code Webview 消息触发。

### 自动触发策略

- `opencode` 根据模型上下文窗口、输入限制、最大输出和 `compaction.reserved` 判断是否触发；触发点在会话 loop 中，上一轮 assistant 完成后检查。
- `costrict` 根据 `autoCondenseContextPercent`、profile 阈值、预留输出 token、当前上下文 token 判断；触发点在 API 请求前。

### 失败回退

- `opencode` 如果压缩时仍然超模型限制，会报 `Conversation history too large to compact` 或处理媒体附件过大场景。
- `costrict` 如果摘要压缩失败且上下文仍超限，会退回 sliding-window truncation，隐藏一部分旧消息以继续请求。

### 压缩结果存储

- `opencode` 用 `type: "compaction"` 的 part 标记压缩边界，生成 `summary: true, mode: "compaction"` 的 assistant 消息。
- `costrict` 用 `condense_context` UI 消息和 API 历史中的 `Summary` 消息关联，`condenseId` 用于删除/回滚时同步维护。

### 扩展能力

- `opencode` 有 `experimental.session.compacting` hook，可注入上下文或替换 prompt。
- `costrict` 有 `customCondensingPrompt`、可配置压缩 API 配置，并会把环境信息、工具 metadata、读过的文件上下文传给摘要过程。

## 注意事项

1. `opencode` 的 `compaction.auto = false` 只禁用自动压缩；手动 `/compact` 或 `POST /session/:sessionID/summarize` 仍可触发。
2. `opencode` 的 `reserved` 不建议设太小。压缩本身也需要上下文窗口，过小可能导致压缩请求也溢出。
3. `costrict` 的自动压缩阈值在不同层有兜底值：`Task` 层未拿到状态时默认 `90`，部分 provider state 兜底逻辑使用 `100`。实际运行以持久化设置和前端状态为准。
4. `costrict` 自动压缩失败后可能进入 sliding-window truncation。这个行为不是摘要，而是隐藏一部分旧消息来降低上下文。
5. 两个项目的压缩摘要都会消耗一次模型调用，需要把 token 成本和延迟纳入产品设计。

## 快速定位表

### OpenCode

| 目的 | 文件 |
| --- | --- |
| 配置 schema | `packages/opencode/src/config/config.ts` |
| 自动溢出判断 | `packages/opencode/src/session/overflow.ts` |
| 会话 loop 自动触发 | `packages/opencode/src/session/prompt.ts` |
| 压缩服务 | `packages/opencode/src/session/compaction.ts` |
| HTTP API | `packages/opencode/src/server/routes/session.ts` |
| 路由挂载 | `packages/opencode/src/server/instance.ts` |
| TUI `/compact` 命令 | `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx` |
| Web App SDK 调用 | `packages/app/src/pages/session/use-session-commands.tsx` |
| Native App 命令入口 | `packages/app-ai-native/src/components/prompt-input.tsx` |
| 中文文档 | `packages/web/src/content/docs/zh-cn/config.mdx` |

### CoStrict

| 目的 | 文件 |
| --- | --- |
| 手动压缩任务方法 | `src/core/task/Task.ts` 的 `condenseContext()` |
| 自动压缩请求前检查 | `src/core/task/Task.ts` 的 API request 逻辑 |
| 阈值判断和失败回退 | `src/core/context-management/index.ts` |
| 摘要生成 | `src/core/condense/index.ts` |
| Webview 消息入口 | `src/core/webview/webviewMessageHandler.ts` |
| Provider 调用入口 | `src/core/webview/ClineProvider.ts` |
| 前端触发按钮/消息 | `webview-ui/src/components/chat/ChatView.tsx` |
| 设置 UI | `webview-ui/src/components/settings/ContextManagementSettings.tsx` |
| 中文设置文案 | `webview-ui/src/i18n/locales/zh-CN/settings.json` |
| 结果 schema | `packages/types/src/message.ts` |
