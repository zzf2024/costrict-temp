# OpenCode / CoStrict 与 Hermes Agent、LangChain、LangGraph 的 Agent 编排方式对比报告

调研对象：

- `zgsm-sangfor/opencode`：本地克隆路径 `/tmp/zgsm-sangfor-opencode`，分支 `dev`，commit `b158c71902a635994f578da65c7f824d391f7a3d`。
- `zgsm-ai/costrict`：本地克隆路径 `/tmp/zgsm-ai-costrict`，分支 `main`，commit `ffb7485b775925ecc3247d6bac735f4408f0c03a`。
- LangChain / LangGraph：以 2026-07-15 可访问的官方文档为准。
- Hermes Agent：名称存在歧义。本文按两个公开材料覆盖的含义讨论：
  - `Hermes: A Large Language Model Framework on the Journey to Autonomous Networks` 中的 Hermes，即面向网络数字孪生的 blueprint + LLM agent chain。
  - `Channel Fracture` 论文中提到的生产 Hermes Agent，即包含 profile、memory injection、scheduler、WebSocket ACK 等跨边界传递机制的多 agent 系统。

本文的“编排方式”指的是：系统如何组织 agent、任务、工具、上下文、状态、子任务、失败恢复和人工审批，而不仅是单个工具如何执行。

## 1. 总体结论

五套体系的定位不同：

| 系统 | 编排定位 | 核心抽象 | 最适合的场景 |
| --- | --- | --- | --- |
| OpenCode CLI | CLI/服务端 coding agent 的 session 编排 | `SessionPrompt.loop`、`Agent.Info`、`task` tool、child session、tool processor | 命令行交互、代码任务、可并行探索的 subagent |
| CoStrict VSCode 插件 | VSCode 内单活动任务 + mode + delegation 编排 | `Task`、mode、native tool parser、`new_task`、`delegateParentAndOpenChild` | IDE 内代码执行、用户审批、父子任务切换 |
| LangChain Agent | 通用 agent harness | model + prompt + tools + middleware + checkpointer | 快速构建工具调用 agent、加入中间件和重试 |
| LangGraph | 通用 stateful graph runtime | `StateGraph`、node、edge、checkpoint、interrupt | 长生命周期、强状态、可恢复、多节点 agent workflow |
| Hermes Agent | 领域型多 agent / profile / blueprint 编排 | blueprint、agent chain、profile、memory channel、delivery verification | 特定领域自治流程、跨 agent 边界传递可靠性 |

关键差异：

- OpenCode 比 CoStrict 更接近“主 agent 调 subagent as tool”的模型。`task` 工具会创建或复用子 session，完整运行子 agent，然后把 `<task_result>` 返回父 agent。
- CoStrict 更强调 IDE 单活动任务约束。`new_task` 会保存父任务、关闭父任务、切换 mode、创建子任务，子任务完成后再把结果写回父任务并重新打开父任务。
- LangChain 提供的是高层 agent harness，可通过 middleware 做上下文管理、重试、人工审批、子 agent 委派等。
- LangGraph 提供的是更底层的状态图运行时，优势是显式状态、checkpoint、interrupt、resume、可观测状态迁移。
- Hermes Agent 的可借鉴点不在通用工具调用 API，而在 blueprint 化流程、profile 边界、memory 注入、跨通道确认和“静默失败”验证。

## 2. OpenCode CLI 的编排方式

### 2.1 核心源码位置

```text
/tmp/zgsm-sangfor-opencode/packages/opencode/src/agent/agent.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/session/prompt.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/session/llm.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/session/processor.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/tool/task.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/tool/registry.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/session/compaction.ts
/tmp/zgsm-sangfor-opencode/packages/opencode/src/costrict/agent/builtin.ts
```

### 2.2 Agent 定义

OpenCode 的 agent 定义集中在 `agent/agent.ts`。

`Agent.Info` 包含：

```ts
{
  name: string
  description?: string
  mode: "subagent" | "primary" | "all"
  permission: Permission.Ruleset
  model?: { providerID, modelID }
  prompt?: string
  options: Record<string, any>
  steps?: number
}
```

内置 agent 包括：

- `build`：默认 primary agent，负责正常执行。
- `plan`：primary agent，但限制编辑能力。
- `general`：subagent，适合研究复杂问题和多步任务。
- `explore`：subagent，偏代码探索，默认只允许搜索、读取、webfetch、websearch、codesearch 等。
- `compaction`：隐藏 primary agent，用于上下文压缩。
- `title` / `summary`：隐藏 primary agent，用于标题和总结。

OpenCode 的 CoStrict 定制 agent 模板在：

```text
/tmp/zgsm-sangfor-opencode/packages/opencode/src/costrict/agent/builtin.ts
```

其中包括 wiki、plan、spec、strict、tdd 等内置 agent 文本，例如：

```text
plan-quick-explore
plan-sub-coding
plan-task-check
spec-design
spec-plan
spec-task
strict-plan
strict-spec
tdd
```

这些更像可加载的角色/工作流提示词，而不是独立的调度器节点。

### 2.3 主循环：SessionPrompt.loop

OpenCode 的核心循环在 `session/prompt.ts`。

简化链路：

```text
SessionPrompt.prompt(...)
  -> SessionPrompt.loop(...)
  -> 读取 MessageV2.stream(sessionID)
  -> 找最近 user / assistant / finished assistant
  -> 优先处理 pending subtask 或 compaction part
  -> 否则创建新的 assistant message
  -> SessionProcessor.create(...)
  -> resolveTools(...)
  -> LLM.stream(...)
  -> processor 消费 text/tool/error/finish 事件
  -> 如果还有 tool calls，下一轮继续
```

主循环里有两个特殊任务会先于普通模型请求处理：

```text
MessageV2.SubtaskPart
MessageV2.CompactionPart
```

这意味着 OpenCode 把“子任务”和“压缩上下文”都建模成 session 历史里的特殊 part。loop 每轮会检查是否存在待处理的 subtask/compaction，如果有，则先处理这些 part，再进入普通 LLM tool-calling 循环。

### 2.4 `task` 工具：主 agent 调 subagent

`task` 工具在：

```text
/tmp/zgsm-sangfor-opencode/packages/opencode/src/tool/task.ts
```

参数：

```ts
{
  description: string
  prompt: string
  subagent_type: string
  task_id?: string
  command?: string
}
```

执行逻辑：

1. 根据调用方 agent 的 permission 过滤可访问 subagent。
2. 若不是由用户显式命令绕过，则调用 `ctx.ask(...)` 进行权限确认。
3. 获取目标 agent 配置。
4. 如果传入 `task_id` 且 session 存在，则复用旧子 session；否则创建新 child session：

```text
Session.create({ parentID: ctx.sessionID, title, permission })
```

5. 根据目标 agent 或父消息模型选择子任务模型。
6. 调用：

```text
SessionPrompt.prompt({
  sessionID: childSession.id,
  agent: agent.name,
  tools: ...,
  parts: promptParts,
})
```

7. 取子 session 最后一段文本，包装为：

```text
task_id: <child-session-id> (for resuming...)

<task_result>
...
</task_result>
```

8. 把结果作为工具输出返回给父 agent。

这个模型非常接近 LangChain 文档里的 “main agent coordinates subagents as tools”。父 agent 仍然是中心控制者，subagent 的执行结果回到父 agent 的工具观察里，父 agent 再决定下一步。

### 2.5 命令和 `@agent` 的子任务转换

`session/prompt.ts` 里还会把部分 command / agent reference 转成 `SubtaskPart`。

典型规则：

```text
isSubtask = (agent.mode === "subagent" && cmd.subtask !== false) || cmd.subtask === true
```

如果是 subtask，则用户消息里写入：

```ts
{
  type: "subtask",
  agent,
  description,
  command,
  model,
  prompt,
}
```

随后 `SessionPrompt.loop` 会检测到该 `SubtaskPart`，调用 `handleSubtask(...)`，本质上仍然走 `TaskTool.execute(...)`。

### 2.6 工具与权限如何参与编排

工具注册在：

```text
/tmp/zgsm-sangfor-opencode/packages/opencode/src/tool/registry.ts
```

`ToolRegistry.tools(model, agent)` 会：

- 注册基础工具：`bash`、`read`、`grep`、`edit`、`write`、`task`、`todowrite`、`webfetch`、`skill` 等。
- 注册 CoStrict 扩展工具：`sequential-thinking`、`file-outline`、`checkpoint`、`spec-manage`、`workflow` 等。
- 根据模型过滤 `apply_patch` / `edit` / `write`。
- 触发插件 hook `tool.definition`。

最终在 `LLM.stream(...)` 前还会结合 agent permission 和 session permission 再过滤：

```text
Permission.disabled(Object.keys(input.tools), Permission.merge(input.agent.permission, input.permission ?? []))
```

所以 OpenCode 的编排约束不是图边，而是：

```text
agent mode
+ agent permission
+ session permission
+ tool registry model filter
+ user/command subtask part
+ plugin hook
```

### 2.7 OpenCode 编排特征

优点：

- subagent 是一等能力，父子 session 有 `parentID`，可恢复旧 `task_id`。
- 父 agent 可以并行发多个 `task` 工具调用，适合探索类任务。
- 权限规则绑定 agent，适合给 explore/plan/general 不同工具能力。
- session 历史、tool part、compaction part、subtask part 都进入统一消息模型。
- 插件 hook 能改工具定义、chat params、headers、messages、system prompt。

限制：

- 它不是显式 graph runtime。子任务、压缩、普通 LLM 调用都由 `SessionPrompt.loop` 里的过程式逻辑调度。
- subagent 结果主要以文本 `<task_result>` 返回，缺少强类型输出契约。
- 多 subagent fan-out / fan-in 没有独立的调度状态机，主要依赖模型发起多个 `task` tool calls。
- 重试、循环检测、任务超时、递归深度等更分散，缺少统一 orchestration policy。

## 3. CoStrict VSCode 插件的编排方式

### 3.1 核心源码位置

```text
/tmp/zgsm-ai-costrict/src/core/task/Task.ts
/tmp/zgsm-ai-costrict/src/core/assistant-message/presentAssistantMessage.ts
/tmp/zgsm-ai-costrict/src/core/tools/NewTaskTool.ts
/tmp/zgsm-ai-costrict/src/core/webview/ClineProvider.ts
/tmp/zgsm-ai-costrict/src/core/task/build-tools.ts
/tmp/zgsm-ai-costrict/src/core/prompts/tools/filter-tools-for-mode.ts
/tmp/zgsm-ai-costrict/src/core/prompts/tools/native-tools/new_task.ts
/tmp/zgsm-ai-costrict/src/core/prompts/tools/native-tools/switch_mode.ts
/tmp/zgsm-ai-costrict/src/shared/modes.ts
```

### 3.2 Mode 是 CoStrict 的主要 agent 边界

CoStrict 的 mode 定义在：

```text
/tmp/zgsm-ai-costrict/src/shared/modes.ts
```

mode 决定：

- role definition / custom instructions。
- 可用 tool group。
- provider / `costrictCodeMode` 过滤。
- 是否有 `taskMode` 限制，即当前 mode 只能委派到某个指定 mode。

工具集合通过：

```text
getToolsForMode(groups)
```

构造。它会加入 mode 的 tool groups，再加入 `ALWAYS_AVAILABLE_TOOLS`。

具体模型请求工具数组在：

```text
/tmp/zgsm-ai-costrict/src/core/task/build-tools.ts
```

`buildNativeToolsArrayWithRestrictions(...)` 会组合：

- native tools。
- MCP server tools。
- custom tools。
- mode 过滤。
- experiment 过滤。
- disabled tools。
- modelInfo included/excluded tools。
- tool alias。
- Gemini 等 provider 的 `allowedFunctionNames` 限制。

所以 CoStrict 的“agent 编排”本质是：

```text
当前 Task
+ 当前 mode persona
+ mode/tool group 权限
+ native tool parser
+ presentAssistantMessage 工具分发
+ new_task / switch_mode 做任务或模式切换
```

### 3.3 Task 主循环与工具执行

CoStrict 的主循环在 `Task.ts`。模型输出会被 parser 转为 `assistantMessageContent`，其中包含 text 和 tool_use block。

关键执行链路：

```text
Task.recursivelyMakeClineRequests(...)
  -> API stream chunk
  -> NativeToolCallParser
  -> assistantMessageContent
  -> assistantContent 写入 API conversation history
  -> presentAssistantMessage(this)
  -> 根据 tool_use.name 分发到具体 Tool class
  -> pushToolResult(...) 写入 userMessageContent
  -> 等待 tool_result 完成后进入下一轮 API 请求
```

这里有一个重要顺序：

```text
先保存 assistant tool_use 到 API history
再执行工具并写 tool_result
```

这样可以避免 Anthropic/OpenAI 类工具协议出现 `tool_result` 早于 `tool_use` 的历史顺序错误。

### 3.4 `new_task` 工具

模型看到的 schema 在：

```text
/tmp/zgsm-ai-costrict/src/core/prompts/tools/native-tools/new_task.ts
```

描述里明确要求：

```text
new_task MUST be called alone
```

执行实现：

```text
/tmp/zgsm-ai-costrict/src/core/tools/NewTaskTool.ts
```

主要逻辑：

1. 校验 `mode`、`message`。
2. 根据 VSCode 设置判断 `todos` 是否必填。
3. 校验当前 mode 的 `taskMode` 限制。
4. 解析 markdown checklist 为 initial todos。
5. 校验目标 mode 是否存在。
6. 通过 `askApproval("tool", toolMessage)` 请求用户批准。
7. 调用：

```text
provider.delegateParentAndOpenChild({
  parentTaskId: task.taskId,
  message,
  initialTodos,
  mode,
})
```

8. 返回：

```text
Delegated to child task <child.taskId>
```

### 3.5 `new_task` 隔离机制

`Task.ts` 对 `new_task` 做了额外隔离：

```text
if new_task appears before later tool_use blocks:
  truncate all blocks after new_task
  truncate execution array
  inject error tool_results for truncated tool_use ids
```

目的：

- 防止模型在同一轮里先 `new_task`，后面又继续调用其它工具。
- 因为 `new_task` 会触发父任务被关闭/委派，如果后续工具继续执行，会造成 orphan tool_use、缺失 tool_result 或父任务历史无法恢复。

这说明 CoStrict 的编排不支持“new_task 后同轮继续执行”，它把 delegation 当成一个会改变活动任务所有权的边界。

### 3.6 父子任务切换：delegateParentAndOpenChild

核心实现：

```text
/tmp/zgsm-ai-costrict/src/core/webview/ClineProvider.ts
```

`delegateParentAndOpenChild(...)` 的关键步骤：

1. 确认 parent 是当前活动 task。
2. 再次校验 parent mode 的 `taskMode` 限制。
3. 调用 `parent.flushPendingToolResultsToHistory()`，在关闭父任务前把 pending tool_result 写入 API history。
4. 如果 flush 失败，尝试 `retrySaveApiConversationHistory()`。
5. `removeClineFromStack({ skipDelegationRepair: true })` 关闭/释放父任务，维护单活动任务约束。
6. `handleModeSwitch(mode)` 切到子任务 mode。
7. `createTask(...)` 创建子任务，但先 `startTask: false`，避免子任务抢先写 global state。
8. 更新父任务 history：

```ts
{
  status: "delegated",
  delegatedToId: child.taskId,
  awaitingChildId: child.taskId,
  childIds,
}
```

9. `child.start()` 启动子任务。
10. emit `TaskDelegated`。

这个流程更像 UI/IDE 任务栈事务，而不是 LangGraph 那样在一个 runtime 内并行跑多个节点。

### 3.7 子任务完成后恢复父任务

`reopenParentFromDelegation(...)` 会：

1. 读取父任务 UI messages 和 API messages。
2. 写入 UI 侧 `subtask_result`。
3. 在父 API history 中寻找最近的 `new_task` tool_use id。
4. 如果找到，则注入匹配的 `tool_result`：

```text
Subtask <childTaskId> completed.

Result:
<completionResultSummary>
```

5. 如果找不到对应 tool_use，则降级写入普通 user text，避免构造非法 tool_result。
6. 保存 API messages。
7. 关闭 child。
8. 把 child history 标记为 `completed`。
9. 把 parent history 标记为 `active`，写入 `completedByChildId`、`completionResultSummary`。
10. emit `TaskDelegationCompleted`。
11. 用 `createTaskWithHistoryItem(...)` 重新打开父任务，调用 `resumeAfterDelegation()` 继续。

### 3.8 `switch_mode`

`switch_mode` 的 schema 在：

```text
/tmp/zgsm-ai-costrict/src/core/prompts/tools/native-tools/switch_mode.ts
```

它是 mode 内部请求切换到另一个 mode 的工具，必须说明 `mode_slug` 和 `reason`，并要求用户批准。

因此 CoStrict 有两类“编排边界”：

- `switch_mode`：当前 task 不变，切换 persona / tools。
- `new_task`：当前 task 委派出去，打开 child task，之后再恢复 parent。

### 3.9 CoStrict 编排特征

优点：

- 很适合 VSCode 插件形态，始终维护“单活动任务”。
- mode/tool group 权限清晰，用户审批强。
- `new_task` 前后有专门协议修复逻辑，避免 tool_use/tool_result 缺失。
- 父子任务历史有 metadata，UI 能展示 delegation 关系。
- 子任务完成后会把结果回灌到父任务 API history，父任务可以继续推理。

限制：

- 它不是通用 graph runtime，没有显式 node/edge/checkpoint state machine。
- `new_task` 是独占式 delegation，天然不适合并行子任务 fan-out / fan-in。
- 父子任务恢复依赖文件历史和 synthetic message 注入，容易出现状态顺序、tool id、mode 切换、flush 失败等边界问题。
- mode 与工具权限较分散，缺少一个统一的“编排策略层”来描述哪些 mode 可以转给哪些 mode、哪些失败可以 retry、哪些需要 human interrupt。

## 4. LangChain Agent 的编排方式

官方文档把 agent 定义为：

```text
model calling tools in a loop until a given task is complete
```

LangChain 的 agent 更像一个高层 harness：

```text
Agent = Model + Harness
Harness = model + prompt + tools + middleware
```

参考：

- https://docs.langchain.com/oss/python/langchain/agents

### 4.1 Agent harness

典型写法：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-5.5",
    tools=tools,
    system_prompt="...",
)
```

它默认提供工具调用循环；复杂能力通过 middleware 增强，而不是把所有逻辑写死在主循环。

### 4.2 状态与恢复

LangChain agent 可以通过 `thread_id` 和 `checkpointer` 持久化会话历史。

官方文档强调：

- `thread_id` 负责圈定 conversation / checkpoint。
- `context` 负责传递 per-run 数据，例如 user id、API key、feature flag。
- 配置 checkpointer 后才能持久化并恢复 history。

这对 OpenCode / CoStrict 的启发是：应该明确区分“长期会话状态”和“本次运行上下文”，不要把 mode、工具权限、用户状态、任务元数据全部混在消息历史里。

### 4.3 Middleware

LangChain 的 middleware 覆盖：

- execution environment。
- context management。
- planning and delegation。
- fault tolerance。
- guardrails。
- steering。

其中 fault tolerance 包括 model retry 和 tool retry；steering 支持在破坏性写入、昂贵 API 调用等点暂停，等待人工批准、修改或拒绝。

对 OpenCode / CoStrict 来说，middleware 是很值得借鉴的分层方式。当前两者有很多分散逻辑：

- 工具参数校验。
- 权限审批。
- tool_result 补齐。
- mode/tool 过滤。
- 上下文压缩。
- 连续失败计数。
- 用户拒绝后的后续工具跳过。
- delegation 前后 flush/retry。

这些都可以抽象成可组合 middleware，而不是散在主循环、工具实现和 provider 里。

### 4.4 Multi-agent patterns

LangChain 官方多 agent 文档列出常见模式：

- subagents：主 agent 把 subagent 当工具调度，所有 routing 回到主 agent。
- handoffs：agent 之间通过 tool call 转交控制权，接收方可以直接面对用户。
- skills：单 agent 按需加载专门 prompt / knowledge。
- router：先分类，再把输入路由到专门 agent，最后合成结果。
- custom workflow：用 LangGraph 构造更明确的工作流。

参考：

- https://docs.langchain.com/oss/python/langchain/multi-agent

OpenCode 的 `task` 更接近 subagents。CoStrict 的 `new_task` 更接近 handoff，但它不是直接“另一个 agent 接管并面向用户”，而是“IDE 单活动 task 切换到 child，完成后再恢复 parent”。

## 5. LangGraph 的编排方式

LangGraph 官方文档把它定位为：

```text
low-level orchestration framework and runtime for long-running, stateful agents
```

核心能力包括：

- durable execution。
- streaming。
- human-in-the-loop。
- persistence。
- short-term / long-term memory。
- debug / trace / state transition visibility。

参考：

- https://docs.langchain.com/oss/python/langgraph/overview

### 5.1 Graph primitive

LangGraph 的基本形态是：

```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(State)
graph.add_node("planner", planner)
graph.add_node("executor", executor)
graph.add_edge(START, "planner")
graph.add_edge("planner", "executor")
graph.add_edge("executor", END)
graph = graph.compile()
```

它强调显式 state 和显式 transition。

### 5.2 与 OpenCode / CoStrict 的差异

OpenCode 和 CoStrict 都有“循环”，但都不是“图”：

- OpenCode 的 loop 在代码里按顺序判断 subtask、compaction、普通 LLM 调用。
- CoStrict 的 loop 在 Task 内处理 stream、tool_use、tool_result、delegation。
- LangGraph 会把 planner、tool executor、reviewer、human interrupt、subagent、compaction 等拆成 graph node，并让 state 在 node 之间流动。

这带来的差异：

| 维度 | OpenCode / CoStrict | LangGraph |
| --- | --- | --- |
| 状态 | 主要散落在 session/task history、message part、metadata、provider state | 显式 state schema |
| 路由 | 过程式 if/else + 模型 tool call | edge / conditional edge |
| 恢复 | 依赖 session/task 持久化和历史修复 | checkpoint/resume 是核心能力 |
| 人工审批 | 工具内或 provider 层 ask/approval | interrupt/human-in-the-loop 是 runtime 能力 |
| 观测 | 依赖日志、message history、tool part | 可追踪 node path 和 state transition |
| 失败边界 | 各工具/各环节自行处理 | 可在 node / edge / checkpoint 层定义 |

## 6. Hermes Agent 的编排方式

Hermes 的公开资料不是一个统一的通用 SDK，因此不能把它等同于 LangChain/LangGraph。

### 6.1 Blueprint + agent chain

论文 `Hermes: A Large Language Model Framework on the Journey to Autonomous Networks` 介绍 Hermes 是一条 LLM agent chain，并使用 blueprint 构造 Network Digital Twin instances。它强调：

- 面向网络自治场景。
- 用 blueprint 把复杂建模流程拆成结构化、可解释步骤。
- 多 agent 按流程协作，不是单 agent 随意工具调用。

参考：

- https://arxiv.org/abs/2411.06490

对 OpenCode / CoStrict 的启发：对于复杂代码任务，可以把“探索、设计、实现、验证、复核”做成可配置 blueprint，而不是只靠 system prompt 建议模型自觉遵循。

### 6.2 Cross-boundary channel verification

`Channel Fracture` 论文报告了生产 Hermes Agent 中的三类静默传递失败：

- cron memory injection 被 scheduler barrier 阻断。
- cross-profile skill routing 因递归目录遍历产生 fracture。
- WebSocket delivery confirmation fallback fracture 导致消息重复确认。

论文提出 CADVP 验证协议，强调 inverse verification、channel matching、PIP protection 等原则。

参考：

- https://arxiv.org/abs/2606.04896

这类问题和 CoStrict 的父子任务恢复、OpenCode 的子 session 结果回传很相关：只要跨 agent、跨 profile、跨 session、跨 transport，就不能只相信“我发了”，还要验证“对方以正确身份、正确顺序、正确 channel 收到了，并且没有重复或静默丢失”。

## 7. 五者详细对比

### 7.1 编排 primitive

| 系统 | 编排 primitive | 说明 |
| --- | --- | --- |
| OpenCode | session、agent、tool part、subtask part、compaction part | 以 session history 为中心，subagent 通过 `task` tool 启动 |
| CoStrict | Task、mode、tool_use/tool_result、task history metadata | 以 VSCode 当前 Task 为中心，`new_task` 切换活动任务 |
| LangChain | agent harness、middleware、tool、checkpointer | 高层抽象，便于组合能力 |
| LangGraph | graph、node、edge、state、checkpoint、interrupt | 低层 runtime，显式状态迁移 |
| Hermes | blueprint、agent chain、profile、memory channel | 领域流程 + 跨边界通信可靠性 |

### 7.2 子任务 / 多 agent

| 系统 | 子任务模型 | 控制权 |
| --- | --- | --- |
| OpenCode | `task` tool 创建 child session，运行 subagent 后返回结果 | 父 agent 始终是中心 |
| CoStrict | `new_task` 保存父任务并打开 child task，完成后恢复 parent | 当前活动权转给 child，之后再回 parent |
| LangChain | subagents / handoffs / router / skills | 可选，取决于模式 |
| LangGraph | subgraph / graph node / conditional routing | 由 graph 明确控制 |
| Hermes | agent chain / profile routing | 由 blueprint 或 profile 调度逻辑控制 |

### 7.3 并发

| 系统 | 并发能力 |
| --- | --- |
| OpenCode | 父 agent 可以在一轮里发多个 `task` tool calls，AI SDK/tool processor 可处理并发工具调用；具体并发策略仍受模型和 processor 约束 |
| CoStrict | 明确维护单活动任务，`new_task` 还要求单独调用，不适合并行子任务 |
| LangChain | 支持并行 subagent / tool 模式，但要开发者配置 |
| LangGraph | 可通过 graph / subgraph / runtime 做更明确的并发和 fan-in |
| Hermes | 视具体实现，重点不是通用并发 API，而是跨 profile/channel 的可靠传递 |

### 7.4 状态与持久化

| 系统 | 状态保存方式 |
| --- | --- |
| OpenCode | MessageV2、session 数据、tool part state、parentID、compaction part |
| CoStrict | API conversation history、Cline messages、task history、delegation metadata、mode state |
| LangChain | thread_id + checkpointer + runtime context |
| LangGraph | graph state + checkpoint + persistence |
| Hermes | memory/profile/scheduler/channel 状态，依具体实现 |

### 7.5 工具失败处理

| 系统 | 失败处理特点 |
| --- | --- |
| OpenCode | tool error 进入 tool part state；中断工具会标记 error；坏 tool call 尝试 repair 到 `invalid` |
| CoStrict | 工具失败多包装为 tool_result；补齐缺失 tool_result；`new_task` 后续工具截断并注入错误结果 |
| LangChain | ToolRetryMiddleware、ModelRetryMiddleware、guardrails、HITL |
| LangGraph | 通过 checkpoint/resume、node retry、interrupt、人审恢复 |
| Hermes | 关注 channel fracture：静默阻断、错路由、重复 ACK，需要协议级验证 |

### 7.6 人工审批

| 系统 | 人工审批方式 |
| --- | --- |
| OpenCode | `ctx.ask(...)` + permission ruleset |
| CoStrict | `askApproval(...)`、mode switch approval、工具审批 |
| LangChain | HumanInTheLoopMiddleware / steering |
| LangGraph | interrupt 任意状态点，人工查看/修改 state 后继续 |
| Hermes | 公开资料更偏自治网络和验证协议，人工审批不是核心抽象 |

### 7.7 上下文管理

| 系统 | 上下文管理 |
| --- | --- |
| OpenCode | compaction agent、MessageV2.filterCompacted、SessionCompaction、summary/title agent |
| CoStrict | condense context、API history 修复、file context tracker、mode prompt |
| LangChain | middleware + context_schema + checkpointer + memory |
| LangGraph | short-term / long-term memory + checkpointed graph state |
| Hermes | blueprint 中的结构化步骤和 memory injection |

## 8. 后三者对前两者的可借鉴点

### 8.1 LangChain 可借鉴点

适用程度：高。

LangChain 不一定要作为依赖引入，但它的 harness / middleware 分层很适合 OpenCode 和 CoStrict。

#### 对 OpenCode 的建议

1. 把 `SessionPrompt.loop` 里的横切逻辑拆成 middleware：
   - tool call validation。
   - permission / approval。
   - context compression。
   - retry policy。
   - loop guard。
   - user interruption。
   - trace enrichment。
2. 给 `task` tool 增加强类型 result schema，而不只是 `<task_result>` 文本。
3. 引入 subagent policy：
   - 最大递归深度。
   - 最大并发 subtask 数。
   - subagent 超时。
   - subagent 可调用工具白名单。
   - subagent 输出验证。
4. 学习 LangChain 的 `context` 与 `thread_id` 分离：
   - `thread_id/sessionID` 表示会话。
   - runtime context 表示本次调用的 cwd、permission、feature flag、provider headers、user intent。

#### 对 CoStrict 的建议

1. 把散落在 `Task.ts`、`presentAssistantMessage`、Tool class、`ClineProvider` 里的逻辑抽象为 middleware：
   - native tool call normalize。
   - alias resolve。
   - mode permission check。
   - approval。
   - tool retry。
   - tool_result repair。
   - delegation transaction。
2. 对工具失败引入统一 taxonomy：
   - `validation_error`
   - `permission_denied`
   - `user_rejected`
   - `transient_io_error`
   - `protocol_repair`
   - `delegation_state_error`
3. 学习 HumanInTheLoopMiddleware 的配置化方式，把“哪些工具必须审批”从工具内部提到策略层。
4. 对 `switch_mode` 和 `new_task` 设计统一的 transition middleware，减少 mode 切换与任务切换分散处理。

### 8.2 LangGraph 可借鉴点

适用程度：高，但建议借鉴架构思想，不建议直接全量引入。

OpenCode 和 CoStrict 都是 TypeScript 项目，直接依赖 Python LangGraph 不现实。更可行的是实现一个轻量 graph/state-machine 层。

#### 对 OpenCode 的建议

可以把 `SessionPrompt.loop` 显式化为状态机：

```text
Idle
  -> LoadHistory
  -> HandlePendingSubtask
  -> HandlePendingCompaction
  -> BuildPrompt
  -> StreamLLM
  -> ExecuteTools
  -> PersistToolResults
  -> DecideContinueOrFinish
```

每个状态记录：

```ts
{
  sessionID,
  messageID,
  agent,
  step,
  pendingParts,
  activeToolCalls,
  checkpointID,
  outcome
}
```

收益：

- 失败后可以从明确 checkpoint 恢复。
- 更容易定位死循环发生在 prompt、tool、compaction 还是 subtask。
- 可以对每个状态设置最大重试、超时、跳转条件。
- 可以把 compaction/subtask 从 loop 内 if/else 提升为显式节点。

#### 对 CoStrict 的建议

CoStrict 最需要 LangGraph 风格的是 delegation state machine。

建议把 `new_task` 建模为显式事务：

```text
ParentActive
  -> AssistantToolUseSaved
  -> PendingToolResultsFlushed
  -> ParentDisposed
  -> ModeSwitched
  -> ChildCreated
  -> ParentMetadataPersisted
  -> ChildStarted
  -> ChildCompleted
  -> ParentToolResultInjected
  -> ParentReopened
  -> ParentResumed
```

每一步都应有：

```ts
{
  delegationId,
  parentTaskId,
  childTaskId?,
  parentToolUseId,
  expectedMode,
  completedStep,
  retryCount,
  lastError?
}
```

收益：

- 当前 `delegateParentAndOpenChild` / `reopenParentFromDelegation` 中的顺序要求可以机器校验。
- VSCode 退出、扩展 reload、磁盘写失败后可以知道卡在哪一步。
- 可以避免 parent 已被 dispose 但 child 未创建、child 完成但 parent tool_result 未注入等半完成状态。
- 可以降低 `new_task` 相关死循环和工具协议错误。

### 8.3 Hermes Agent 可借鉴点

适用程度：中到高，尤其适合父子任务和跨 profile/channel 的可靠性。

Hermes 的启发不是“照搬一个框架”，而是两个工程原则：

1. 用 blueprint 显式描述复杂流程。
2. 对跨边界传递做 verification，而不是只做 best-effort send。

#### 对 OpenCode 的建议

1. 给复杂 coding workflow 增加 blueprint：

```text
explore -> plan -> implement -> verify -> review -> summarize
```

每个阶段声明：

```ts
{
  allowedAgents,
  allowedTools,
  requiredArtifacts,
  exitCriteria,
  maxAttempts
}
```

2. 对 `task` 子 session 返回做 channel matching：
   - 父 session 发起的 `task.callID`。
   - child session id。
   - child final assistant id。
   - `<task_result>` 对应的 parent tool part id。
   - result 是否已被 parent 消费。
3. 对 subagent 输出做 inverse verification：
   - 父 agent 不只接收结果，还要检查子任务是否满足原始 description/prompt。
   - 对关键任务要求 subagent 返回 structured evidence，例如文件路径、测试命令、失败原因。

#### 对 CoStrict 的建议

1. 为 `new_task` 引入 `delegationId`，所有跨边界消息都带这个 ID：

```text
parent tool_use
parent pending tool_result flush
parent history metadata
child history metadata
child completion summary
parent injected tool_result
TaskDelegated / TaskDelegationCompleted event
```

2. 做 channel matching：
   - parent API history 中必须有 `new_task` tool_use。
   - 注入的 tool_result 必须紧跟对应 assistant message 之后，或者经过明确 repair。
   - child completion 必须匹配 `awaitingChildId`。
   - 当前 provider mode 必须匹配 child mode 或 parent restored mode。
3. 做 ACK 去重：
   - `TaskDelegationCompleted` 事件应有幂等 key。
   - parent tool_result 注入前检查是否已注入同一 `delegationId`。
   - child completed 状态写入要幂等。
4. 做 silent failure audit：
   - flush 失败但继续创建 child 时，记录可恢复 checkpoint。
   - mode switch 失败不能只 log 后继续，至少应进入 degraded state。
   - 如果找不到 parent `new_task` tool_use，降级为 text 是合理兜底，但应把该事件纳入 telemetry/diagnostic。

## 9. 两个本地代码库可以优先改造的方向

### 9.1 OpenCode 优先级

P0：降低死循环和子任务失败概率。

- 给 `task` 增加 `maxDepth`、`maxChildrenPerTurn`、`timeoutMs`。
- 给 subagent result 增加 JSON schema，可选保留文本结果。
- 在 `SessionPrompt.loop` 记录结构化 step trace：

```text
step, phase, active agent, pending subtask count, pending compaction count, tool count, finish reason
```

- 对 repeated same tool error / repeated same subtask prompt 做 loop guard。

P1：提升状态恢复能力。

- 为 subtask part 增加状态字段：

```text
pending -> running -> completed | failed | cancelled
```

- 把 `handleSubtask` 的运行状态写入 parent session，而不只是 tool part。
- 对 child session resume 使用明确 result consumption marker，避免重复消费。

P2：增强 workflow 能力。

- 为 CoStrict 内置 plan/spec/tdd agent 增加 workflow manifest。
- 支持 router agent 或 blueprint 自动选择 explore/general/plan-sub-coding。
- 对多 subagent fan-in 增加合成节点，而不是完全交给父 LLM 自行总结。

### 9.2 CoStrict 优先级

P0：把 delegation 做成可恢复事务。

- 引入 `delegationId`。
- 记录 delegation step checkpoint。
- `delegateParentAndOpenChild` 中每个关键步骤写入事务状态。
- `reopenParentFromDelegation` 使用事务状态判断是否可继续、可重试或必须人工修复。

P0：强化 `new_task` 协议边界。

- 当前已有“必须单独调用”的 prompt + truncate 机制，建议继续补：
  - 如果同轮有 text after new_task，记录 diagnostic。
  - 如果同轮多个 `new_task`，明确只允许第一个，其余作为 protocol error。
  - 对被截断工具的 error tool_result 统一 error code。

P1：mode transition policy。

- 把 `taskMode`、`switch_mode`、provider allowed mode、costrictCodeMode 合并成一个 mode transition policy。
- 显式声明：

```text
from mode
to mode
allowed action: switch | delegate | return
requires approval
allowed tools after transition
```

P1：把工具执行横切逻辑中间件化。

- `validateParamsMiddleware`
- `approvalMiddleware`
- `modePermissionMiddleware`
- `toolResultRepairMiddleware`
- `retryMiddleware`
- `telemetryMiddleware`
- `delegationMiddleware`

P2：谨慎引入并行子任务。

CoStrict 当前单活动任务模型适合 IDE，不建议直接照搬 OpenCode 的并行 subagent。但可以先引入“只读并行探索”：

- 仅允许 read/search/code-index 类工具。
- 子任务不能编辑文件。
- 子任务只返回 structured findings。
- 父任务统一合成，不直接让多个 child 修改 workspace。

## 10. 不建议照搬的地方

### 10.1 不建议把 LangGraph 直接硬塞进 TypeScript 运行时

原因：

- 两个仓库都是 TypeScript 生态。
- OpenCode 已有 Effect service / Runner / SessionProcessor。
- CoStrict 深度耦合 VSCode task、webview、history、provider state。

建议借鉴 LangGraph 的概念，实现轻量状态机，而不是引入跨语言 runtime。

### 10.2 不建议把 Hermes 的领域 blueprint 直接照搬

Hermes 的 NDT blueprint 是电信网络领域建模流程，不适合直接套到 coding agent。

可借鉴的是：

- blueprint 化。
- 阶段 exit criteria。
- cross-boundary verification。
- channel matching。
- silent failure audit。

### 10.3 不建议 CoStrict 直接开放无约束多 agent 并行编辑

CoStrict 在 VSCode 里面对真实工作区，多个子任务并行编辑会带来：

- 文件冲突。
- diff view 冲突。
- 用户审批混乱。
- task history 顺序混乱。
- tool_use/tool_result 对应关系复杂化。

更安全的路径是先开放只读并行探索，再考虑受控编辑。

## 11. 推荐的目标架构

### 11.1 OpenCode：session graph-lite

建议在现有架构上加一层 graph-lite，不改变工具系统：

```text
SessionRuntime
  State:
    sessionID
    agent
    step
    phase
    pendingParts
    activeTools
    checkpoints

  Nodes:
    LoadHistory
    HandleSubtask
    HandleCompaction
    BuildPrompt
    StreamModel
    ExecuteTools
    PersistResults
    DecideNext

  Policies:
    maxSteps
    maxSubtasks
    maxDepth
    retry
    timeout
    loopGuard
```

保留现有：

- `ToolRegistry`
- `SessionProcessor`
- `TaskTool`
- `MessageV2`
- plugin hooks

新增：

- structured trace。
- subtask lifecycle state。
- child result schema。
- fan-in summary node。

### 11.2 CoStrict：delegation transaction + mode transition policy

建议在现有 `Task` / `ClineProvider` 上加两层：

```text
DelegationTransactionManager
ModeTransitionPolicy
```

`DelegationTransactionManager` 管：

```text
delegationId
parentTaskId
childTaskId
toolUseId
step
checkpoint
retry
idempotency
repair
```

`ModeTransitionPolicy` 管：

```text
from mode
to mode
action type
approval rule
tool restrictions
provider restrictions
return mode
```

这样可以把 `new_task`、`switch_mode`、parent restore、tool_result repair 的状态边界收束起来。

## 12. 结论

OpenCode 和 CoStrict 都已经有 agent 编排能力，但两者不是同一种编排：

- OpenCode 是 session-centered subagent-as-tool 编排，适合 CLI 中并行探索和子 session 执行。
- CoStrict 是 task-centered single-active-task 编排，适合 VSCode 里强用户审批和父子任务切换。

LangChain 值得借鉴的是 harness/middleware 分层和多 agent pattern；LangGraph 值得借鉴的是显式 state graph、checkpoint、interrupt 和可恢复执行；Hermes Agent 值得借鉴的是 blueprint 化和跨边界 channel verification。

如果目标是降低死循环和工具调用失败概率，优先级最高的不是增加更多 agent，而是：

1. 把编排状态显式化。
2. 把父子任务/子 session 交接做成幂等事务。
3. 把工具失败分类、重试、审批、协议修复做成统一 middleware。
4. 对跨 agent 边界引入 channel matching 和 inverse verification。

## 13. 参考资料

- LangGraph overview: https://docs.langchain.com/oss/python/langgraph/overview
- LangChain agents: https://docs.langchain.com/oss/python/langchain/agents
- LangChain multi-agent: https://docs.langchain.com/oss/python/langchain/multi-agent
- Hermes: A Large Language Model Framework on the Journey to Autonomous Networks: https://arxiv.org/abs/2411.06490
- Channel Fracture: Three Instances of Cross-Boundary Silent Delivery Reliability Failures in Multi-Agent Systems: https://arxiv.org/abs/2606.04896
