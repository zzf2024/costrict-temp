# OpenCode CLI 与 CoStrict VSCode 插件内置工具调研报告

调研对象：

- `zgsm-sangfor/opencode`：本地克隆路径 `/tmp/zgsm-sangfor-opencode`，分支 `dev`，commit `b158c71902a635994f578da65c7f824d391f7a3d`。
- `zgsm-ai/costrict`：本地克隆路径 `/tmp/zgsm-ai-costrict`，分支 `main`，commit `ffb7485b775925ecc3247d6bac735f4408f0c03a`。

结论摘要：

- `opencode` 是 CLI/服务端形态，工具由 `ToolRegistry` 在运行时注册，工具实现和执行函数基本合在同一份 `Tool.define(...)` 文件中。
- `costrict` 是 VSCode 插件形态，工具拆成“给模型看的 tool schema”和“插件端执行实现”两套目录，再通过 mode、实验开关、MCP、模型能力、用户禁用项做过滤。
- 两个仓库都有 CoStrict 定制工具，例如结构化思考、文件大纲、checkpoint、代码结构分析相关能力，但默认暴露范围不同。

## 1. OpenCode CLI 工具体系

### 1.1 核心入口

OpenCode 的工具注册入口在：

```text
packages/opencode/src/tool/registry.ts
```

这个文件承担三件事：

1. 引入内置工具，例如 `bash`、`read`、`edit`、`webfetch` 等。
2. 引入 CoStrict 扩展工具，例如 `sequential-thinking`、`file-outline`、`checkpoint`、`spec-manage`、`workflow`。
3. 扫描插件/配置目录里的自定义工具，并追加到工具列表。

关键接口：

```ts
ToolRegistry.register(tool)
ToolRegistry.ids()
ToolRegistry.tools(model, agent?)
```

`ToolRegistry.tools(...)` 返回的是最终可传给模型并可执行的工具定义数组，每项包含：

```ts
{
  id: string
  description: string
  parameters: z.ZodType
  execute: Function
  formatValidationError?: Function
}
```

工具本身遵循：

```text
packages/opencode/src/tool/tool.ts
```

核心定义方式是：

```ts
Tool.define("tool_id", ...)
```

### 1.2 内置工具目录

OpenCode 基础工具目录：

```text
packages/opencode/src/tool/
```

该目录下主要文件如下：

| 文件 | 工具 ID | 默认注册情况 | 说明 |
| --- | --- | --- | --- |
| `bash.ts` | `bash` | 默认注册 | 执行 shell 命令。 |
| `read.ts` | `read` | 默认注册 | 读取文件内容。 |
| `glob.ts` | `glob` | 默认注册 | 按 glob 模式找文件。 |
| `grep.ts` | `grep` | 默认注册 | 搜索文件内容。 |
| `edit.ts` | `edit` | 条件注册后可能被模型过滤 | 编辑已有文件。对支持 `apply_patch` 的 GPT 系模型会被过滤掉。 |
| `write.ts` | `write` | 条件注册后可能被模型过滤 | 写文件。对支持 `apply_patch` 的 GPT 系模型会被过滤掉。 |
| `apply_patch.ts` | `apply_patch` | 条件可用 | Codex 风格 patch 工具。仅在特定 GPT 系模型上保留。 |
| `task.ts` | `task` | 默认注册 | 启动/委派子任务。 |
| `todo.ts` | `todowrite` | 默认注册 | 写入待办列表。 |
| `webfetch.ts` | `webfetch` | 默认注册 | 拉取网页内容。 |
| `websearch.ts` | `websearch` | 默认注册后按模型过滤 | Web 搜索。只在 `opencode` provider 或启用 `OPENCODE_ENABLE_EXA` 时保留。 |
| `codesearch.ts` | `codesearch` | 默认注册后按模型过滤 | 代码搜索。过滤规则同 `websearch`。 |
| `skill.ts` | `skill` | 默认注册 | 加载/使用 skill。 |
| `question.ts` | `question` | 条件注册 | 当客户端是 `app`、`cli`、`desktop`，或 `OPENCODE_ENABLE_QUESTION_TOOL` 开启时注册。 |
| `invalid.ts` | `invalid` | 默认注册 | 内部兜底工具，提示无效参数。 |
| `lsp.ts` | `lsp` | 实验开关 | `OPENCODE_EXPERIMENTAL_LSP_TOOL` 开启时注册。 |
| `batch.ts` | `batch` | 实验开关 | `experimental.batch_tool === true` 时注册，可批量调用工具。 |
| `plan.ts` | `plan_exit` | 实验开关 | `OPENCODE_EXPERIMENTAL_PLAN_MODE` 且客户端为 `cli` 时注册。 |
| `ls.ts` | `list` | 源码存在，默认未注册 | 文件列表工具，当前未出现在主 registry 默认数组。 |
| `multiedit.ts` | `multiedit` | 源码存在，默认未注册 | 多处编辑工具，当前未出现在主 registry 默认数组。 |
| `plan.ts` | `plan_enter` | 源码存在，默认未注册 | 进入 plan 模式工具，当前 registry 只条件注册 `plan_exit`。 |

### 1.3 CoStrict 扩展工具目录

OpenCode 仓库里还有一组 CoStrict 定制工具：

```text
packages/opencode/src/costrict/tool/
```

主文件如下：

| 文件 | 工具 ID | 默认注册情况 | 说明 |
| --- | --- | --- | --- |
| `sequential-thinking.ts` | `sequential-thinking` | 默认注册 | 结构化思考工具。 |
| `file-outline.ts` | `file-outline` | 默认注册 | 提取代码文件结构、类、函数、方法、文档字符串。 |
| `checkpoint.ts` | `checkpoint` | 默认注册，可被配置关闭 | `experimental.checkpoint !== false` 时注册。 |
| `spec-manage.ts` | `spec-manage` | 默认注册，可被配置关闭 | `experimental.spec_manage !== false` 时注册。 |
| `workflow.ts` | `workflow` | 默认注册 | 提供工作流模式提示词。 |
| `call-graph.ts` | `call-graph` | 源码存在，默认未注册 | 调用图和继承链分析入口。 |
| `file-importance.ts` | 未作为工具注册 | 支撑目录存在 | 文件重要性分析相关实现，主 registry 当前没有注册该工具。 |

支撑实现目录：

```text
packages/opencode/src/costrict/tool/call-graph/
packages/opencode/src/costrict/tool/file-importance/
packages/opencode/src/costrict/tool/query/
packages/opencode/src/costrict/tool/service/
packages/opencode/src/costrict/tool/util/
packages/opencode/src/costrict/tool/workflow/
```

其中 `query/` 下按语言放置 Tree-sitter 查询文件，例如：

```text
c-tags.ts
cpp-tags.ts
go-tags.ts
java-tags.ts
javascript-tags.ts
python-tags.ts
typescript-tags.ts
```

### 1.4 默认工具列表

按 `registry.ts` 的 `all(custom)` 构造逻辑，OpenCode 默认工具大体如下：

```text
invalid
question                    # 条件注册
bash
read
glob
grep
edit                        # 后续可能按模型过滤
write                       # 后续可能按模型过滤
task
webfetch
todowrite
websearch                   # 后续可能按 provider/flag 过滤
codesearch                  # 后续可能按 provider/flag 过滤
skill
sequential-thinking
file-outline
checkpoint                  # experimental.checkpoint !== false
spec-manage                 # experimental.spec_manage !== false
workflow
apply_patch                 # 后续可能按模型过滤
lsp                         # OPENCODE_EXPERIMENTAL_LSP_TOOL
batch                       # experimental.batch_tool === true
plan_exit                   # OPENCODE_EXPERIMENTAL_PLAN_MODE && cli
custom tools
```

模型过滤规则：

- `codesearch`、`websearch`：仅当 provider 是 `opencode`，或启用 `OPENCODE_ENABLE_EXA` 时保留。
- `apply_patch`：当 `modelID` 包含 `gpt-`、不包含 `oss`、不包含 `gpt-4` 时保留。
- `edit`、`write`：与 `apply_patch` 互斥；当 `apply_patch` 可用时会过滤掉 `edit` 和 `write`。

### 1.5 插件和自定义工具

OpenCode 支持两类动态工具：

1. 配置目录扫描：

```text
{tool,tools}/*.{js,ts}
```

扫描目录来自 `config.directories()`。如果文件导出 `default`，工具 ID 使用文件名；如果导出具名对象，工具 ID 形如：

```text
文件名_导出名
```

2. 插件工具：

```ts
plugin.list()
p.tool
```

插件工具会经过 `fromPlugin(id, def)` 包装，转换成 OpenCode 内部 `Tool.Info`。

### 1.6 TDD 插件工具

OpenCode 仓库内置 TDD 插件目录：

```text
packages/opencode/src/plugin/tdd/
```

其中工具相关目录：

```text
packages/opencode/src/plugin/tdd/tools/
```

主要文件：

| 文件 | 说明 |
| --- | --- |
| `list-user-commands.tool.ts` | TDD 插件提供的用户命令列表工具。 |
| `bash.ts` | TDD 插件侧 bash 相关封装。 |
| `shell/` | shell 执行器、工具封装、编码处理等支撑实现。 |
| `constants.ts` | TDD 工具常量。 |

这部分不是 `packages/opencode/src/tool/registry.ts` 里的基础内置工具，而是插件体系的一部分。

## 2. CoStrict VSCode 插件工具体系

### 2.1 核心入口

CoStrict 的工具体系分为两层：

1. 给模型看的工具声明：

```text
src/core/prompts/tools/native-tools/
```

2. 插件侧真正执行工具调用的实现：

```text
src/core/tools/
```

构建工具数组的入口：

```text
src/core/task/build-tools.ts
```

该文件通过：

```ts
buildNativeToolsArray(...)
buildNativeToolsArrayWithRestrictions(...)
```

组合以下来源：

- `getNativeTools(...)`：本地原生工具声明。
- `getMcpServerTools(mcpHub)`：已连接 MCP server 的动态工具。
- `customToolRegistry`：实验开关 `experiments.customTools` 开启后的项目/用户自定义工具。

然后再通过：

```text
src/core/prompts/tools/filter-tools-for-mode.ts
```

按 mode、实验开关、模型配置、禁用工具、MCP 资源存在性进行过滤。

### 2.2 原生工具声明目录

原生工具声明在：

```text
src/core/prompts/tools/native-tools/
```

这些文件定义的是 OpenAI Chat Completions function tool schema，主要字段是：

```ts
{
  type: "function",
  function: {
    name,
    description,
    parameters
  }
}
```

默认由 `index.ts` 的 `getNativeTools()` 返回的工具如下：

| 声明文件 | 工具名 | 默认声明 | 说明 |
| --- | --- | --- | --- |
| `access_mcp_resource.ts` | `access_mcp_resource` | 是 | 访问 MCP resource。后续会在没有 MCP resource 时被过滤。 |
| `apply_diff.ts` | `apply_diff` | 是 | 使用 diff 修改文件。 |
| `apply_patch.ts` | `apply_patch` | 是 | 使用 Codex patch 格式修改文件。 |
| `ask_followup_question.ts` | `ask_followup_question` | 是 | 向用户追问。 |
| `ask_multiple_choice.ts` | `ask_multiple_choice` | 是 | 向用户发起多选/单选问题。 |
| `sequential_thinking.ts` | `sequential_thinking` | 是 | 结构化思考。 |
| `attempt_completion.ts` | `attempt_completion` | 是 | 提交任务完成结果。 |
| `execute_command.ts` | `execute_command` | 是 | 执行命令。 |
| `file_outline.ts` | `file_outline` | 是 | 读取代码文件结构。 |
| `generate_image.ts` | `generate_image` | 是 | 图片生成工具声明；默认会被实验开关过滤。 |
| `list_files.ts` | `list_files` | 是 | 列目录/文件。 |
| `new_task.ts` | `new_task` | 是 | 创建新任务。 |
| `read_command_output.ts` | `read_command_output` | 是 | 读取后台命令输出。 |
| `read_file.ts` | `read_file` | 是 | 读取文件，支持按模型是否支持图片调整描述。 |
| `run_slash_command.ts` | `run_slash_command` | 是 | 运行 slash command；默认会被实验开关过滤。 |
| `skill.ts` | `skill` | 是 | 加载/使用 skill。 |
| `search_replace.ts` | `search_replace` | 是 | 单处 search/replace。 |
| `edit_file.ts` | `edit_file` | 是 | search/replace 风格编辑。 |
| `edit.ts` | `edit` | 是 | 编辑工具，存在 alias 兼容。 |
| `search_files.ts` | `search_files` | 是 | 正则搜索文件内容。 |
| `switch_mode.ts` | `switch_mode` | 是 | 切换模式。 |
| `update_todo_list.ts` | `update_todo_list` | 是 | 更新 todo list。 |
| `write_to_file.ts` | `write_to_file` | 是 | 写文件。 |
| `codebase_search.ts` | `codebase_search` | 文件存在，默认未声明 | `index.ts` 中 import 和返回项被注释。 |
| `costrict_checkpoint.ts` | `costrict_checkpoint` | 文件存在，默认未声明 | `index.ts` 中 import 和返回项被注释。 |
| `mcp_server.ts` | 动态 MCP 工具 | 动态生成 | 不是单个固定工具；负责把 MCP server tools 转成 native tool schema。 |
| `converters.ts` | 转换器 | 非工具 | OpenAI tool 与 Anthropic tool 格式转换。 |

注意：`getNativeTools()` 只是“声明层默认包含”。工具最终能不能给模型使用，还要经过 mode/实验开关/模型设置过滤。

### 2.3 执行实现目录

工具执行实现集中在：

```text
src/core/tools/
```

主要文件：

| 执行文件 | 对应工具名 | 说明 |
| --- | --- | --- |
| `ExecuteCommandTool.ts` | `execute_command` | 命令执行。 |
| `ReadFileTool.ts` | `read_file` | 文件读取。 |
| `ReadCommandOutputTool.ts` | `read_command_output` | 读取命令输出 artifact。 |
| `WriteToFileTool.ts` | `write_to_file` | 写文件。 |
| `ApplyDiffTool.ts` | `apply_diff` | 应用 diff。 |
| `ApplyPatchTool.ts` | `apply_patch` | 应用 Codex patch；底层支撑在 `apply-patch/`。 |
| `EditTool.ts` | `edit` / `search_and_replace` | 编辑工具及兼容 alias。 |
| `SearchReplaceTool.ts` | `search_replace` | 单处替换。 |
| `EditFileTool.ts` | `edit_file` | 文件编辑。 |
| `SearchFilesTool.ts` | `search_files` | 搜索文件内容。 |
| `ListFilesTool.ts` | `list_files` | 列文件。 |
| `UseMcpToolTool.ts` | `use_mcp_tool` / native MCP tool | 执行 MCP tool 调用。 |
| `accessMcpResourceTool.ts` | `access_mcp_resource` | 访问 MCP resource。 |
| `AskFollowupQuestionTool.ts` | `ask_followup_question` | 追问用户。 |
| `AskMultipleChoiceTool.ts` | `ask_multiple_choice` | 多选/单选问题。 |
| `AttemptCompletionTool.ts` | `attempt_completion` | 任务完成。 |
| `SwitchModeTool.ts` | `switch_mode` | 切换 mode。 |
| `NewTaskTool.ts` | `new_task` | 创建新任务。 |
| `UpdateTodoListTool.ts` | `update_todo_list` | 更新 todo。 |
| `RunSlashCommandTool.ts` | `run_slash_command` | 执行 slash command。 |
| `SkillTool.ts` | `skill` | skill 工具执行。 |
| `GenerateImageTool.ts` | `generate_image` | 图片生成。 |
| `SequentialThinking.ts` | `sequential_thinking` | 结构化思考执行。 |
| `FileOutline.ts` | `file_outline` | 文件结构提取。 |
| `CheckpointTool.ts` | `costrict_checkpoint` | checkpoint 执行实现，声明层当前默认注释。 |
| `CodebaseSearchTool.ts` | `codebase_search` | codebase search 执行实现，声明层当前默认注释。 |
| `BaseTool.ts` | 基类/通用逻辑 | 工具基类。 |
| `validateToolUse.ts` | 校验 | mode 权限和工具调用校验。 |
| `ToolRepetitionDetector.ts` | 重复调用检测 | 检测重复工具调用。 |

支撑目录：

```text
src/core/tools/apply-patch/
src/core/tools/helpers/
src/core/tools/query/
src/core/tools/service/
src/core/tools/util/
src/core/tools/__tests__/
```

### 2.4 调用分发入口

CoStrict 的工具调用分发入口在：

```text
src/core/assistant-message/presentAssistantMessage.ts
```

该文件会按模型返回的 tool call 名称进入不同分支。能看到的主要 case 包括：

```text
execute_command
read_file
write_to_file
apply_diff
edit
search_and_replace
search_replace
edit_file
apply_patch
list_files
use_mcp_tool
access_mcp_resource
ask_followup_question
ask_multiple_choice
attempt_completion
switch_mode
codebase_search
sequential_thinking
file_outline
read_command_output
update_todo_list
new_task
run_slash_command
skill
generate_image
costrict_checkpoint
```

这说明有些工具即使声明层默认没暴露，执行层和分发层也已经预留了实现，例如 `codebase_search`、`costrict_checkpoint`。

### 2.5 MCP 动态工具

MCP 工具声明由：

```text
src/core/prompts/tools/native-tools/mcp_server.ts
```

动态生成。生成逻辑：

1. 从 `mcpHub.getServers()` 读取所有 server。
2. 遍历每个 server 的 `tools`。
3. 跳过 `tool.enabledForPrompt === false` 的工具。
4. 使用 `buildMcpToolName(server.name, tool.name)` 生成符合模型 API 要求的函数名。
5. 对输入 schema 做 normalize。
6. 如果 server config 中配置了 `asyncPolling.tools[toolName].initialArgsTemplate`，会从模型可见 schema 中过滤部分已预填或敏感参数。
7. 通过 `seenToolNames` 去重。

因此 CoStrict 有两种 MCP 相关用法：

- `access_mcp_resource`：访问 MCP resource。
- 动态 MCP tool：把 MCP server 暴露的具体工具直接作为 native function tool 给模型调用。

执行侧由：

```text
src/core/tools/UseMcpToolTool.ts
src/core/tools/accessMcpResourceTool.ts
```

负责。

### 2.6 自定义工具

CoStrict 支持 custom tools，但由实验开关控制：

```ts
experiments?.customTools
```

开启后，`build-tools.ts` 会从当前工作目录对应的 Roo/CoStrict 配置目录下找：

```text
tools/
```

并通过：

```ts
customToolRegistry.loadFromDirectoriesIfStale(toolDirs)
customToolRegistry.getAllSerialized()
formatNative(...)
```

转成 native tool schema。

### 2.7 工具类型、分组和别名

工具名称类型定义在：

```text
packages/types/src/tool.ts
```

当前工具名全集包括：

```text
fake_tool_call
execute_command
read_file
read_command_output
write_to_file
apply_diff
edit
search_and_replace
search_replace
edit_file
apply_patch
search_files
list_files
use_mcp_tool
access_mcp_resource
ask_followup_question
ask_multiple_choice
attempt_completion
switch_mode
new_task
codebase_search
update_todo_list
run_slash_command
skill
generate_image
custom_tool
sequential_thinking
file_outline
costrict_checkpoint
```

工具分组和别名定义在：

```text
src/shared/tools.ts
```

分组：

| 组 | 工具 |
| --- | --- |
| `read` | `read_file`、`search_files`、`list_files`、`codebase_search` |
| `edit` | `apply_diff`、`write_to_file`、`generate_image` |
| `edit.customTools` | `edit`、`search_replace`、`edit_file`、`apply_patch` |
| `command` | `execute_command`、`read_command_output` |
| `mcp` | `use_mcp_tool`、`access_mcp_resource` |
| `modes` | `switch_mode`、`new_task` |
| `question` | `ask_multiple_choice` |
| `sequential_thinking` | `sequential_thinking` |
| `file_outline` | `file_outline` |

始终可用工具：

```text
ask_followup_question
ask_multiple_choice
attempt_completion
switch_mode
new_task
update_todo_list
run_slash_command
skill
costrict_checkpoint
```

注意：`ALWAYS_AVAILABLE_TOOLS` 是 mode 权限层面的定义，不等于 `getNativeTools()` 默认声明。比如 `costrict_checkpoint` 在权限表中始终可用，但声明层当前被注释，所以默认不会出现在模型可见工具列表里。

别名：

| alias | canonical |
| --- | --- |
| `write_file` | `write_to_file` |
| `read` | `read_file` |
| `apply` | `apply_diff` |
| `search` | `search_files` |
| `list` | `list_files` |
| `search_and_replace` | `edit` |

### 2.8 过滤规则

过滤入口：

```text
src/core/prompts/tools/filter-tools-for-mode.ts
```

主要规则：

- 先按当前 mode 的 tool groups 得到允许工具。
- `ALWAYS_AVAILABLE_TOOLS` 会加入 mode 可用集合。
- `modelInfo.excludedTools` 会移除工具。
- `modelInfo.includedTools` 可以额外加入当前 mode 允许组里的工具，并支持 alias 重命名。
- `modeConfig.disableSwitchMode === true` 时移除 `switch_mode`。
- `codebase_search` 只有在 code index feature enabled、configured、initialized 全满足时才保留。
- `update_todo_list` 会受 `todoListEnabled` 影响。
- `generate_image` 需要 `experiments.imageGeneration`。
- `run_slash_command` 需要 `experiments.runSlashCommand`。
- `disabledTools` 会按 canonical name 移除工具。
- `access_mcp_resource` 在没有 MCP resource 时移除。
- 动态 MCP tools 只有在当前 mode 允许 `use_mcp_tool` 时才保留。

## 3. 两个仓库的主要差异

| 维度 | OpenCode CLI | CoStrict VSCode 插件 |
| --- | --- | --- |
| 工具注册入口 | `packages/opencode/src/tool/registry.ts` | `src/core/task/build-tools.ts` |
| 工具声明形态 | `Tool.define(id, ...)` 返回内部工具定义 | OpenAI/Anthropic 兼容 function tool schema |
| 执行实现位置 | 多数与声明同文件 | `src/core/tools/` 单独实现 |
| 默认工具过滤 | 按 provider、model、flag、experimental config | 按 mode、实验开关、模型配置、MCP 状态、disabledTools |
| 自定义工具 | 扫描 `{tool,tools}/*.{js,ts}`，以及 plugin `tool` | `experiments.customTools` 开启后扫描配置目录 `tools/` |
| MCP | 不是当前 registry 主线能力 | 原生支持 MCP resource 和 MCP server dynamic tools |
| CoStrict 定制工具 | `sequential-thinking`、`file-outline`、`checkpoint`、`spec-manage`、`workflow` 等 | `sequential_thinking`、`file_outline`、`costrict_checkpoint`、`codebase_search` 等，但后两者默认声明层注释 |

## 4. 快速定位表

### OpenCode

| 目的 | 路径 |
| --- | --- |
| 工具注册主入口 | `packages/opencode/src/tool/registry.ts` |
| 工具抽象 | `packages/opencode/src/tool/tool.ts` |
| 基础工具目录 | `packages/opencode/src/tool/` |
| CoStrict 扩展工具目录 | `packages/opencode/src/costrict/tool/` |
| TDD 插件工具目录 | `packages/opencode/src/plugin/tdd/tools/` |
| 自定义工具扫描逻辑 | `packages/opencode/src/tool/registry.ts` |

### CoStrict

| 目的 | 路径 |
| --- | --- |
| 工具数组构建入口 | `src/core/task/build-tools.ts` |
| 原生工具声明目录 | `src/core/prompts/tools/native-tools/` |
| 工具执行实现目录 | `src/core/tools/` |
| 工具调用分发 | `src/core/assistant-message/presentAssistantMessage.ts` |
| mode/filter 规则 | `src/core/prompts/tools/filter-tools-for-mode.ts` |
| 工具名称类型 | `packages/types/src/tool.ts` |
| 工具分组、始终可用、alias | `src/shared/tools.ts` |
| MCP 动态工具 schema | `src/core/prompts/tools/native-tools/mcp_server.ts` |
| MCP 工具执行 | `src/core/tools/UseMcpToolTool.ts` |
| MCP resource 执行 | `src/core/tools/accessMcpResourceTool.ts` |

## 5. 判断“是否内置”的口径

建议后续讨论时分三类说清楚：

1. 源码存在：仓库里有声明或执行实现文件。
2. 默认声明/注册：不额外配置时会进入工具注册表或 schema 数组。
3. 最终可调用：经过模型、mode、实验开关、MCP 状态、用户禁用项过滤后，实际传给模型并允许执行。

按这个口径：

- OpenCode 的 `call-graph`、`list`、`multiedit` 属于源码存在，但当前主 registry 默认未注册。
- CoStrict 的 `codebase_search`、`costrict_checkpoint` 属于声明文件和执行实现都存在，但 `getNativeTools()` 中默认注释；其中 `costrict_checkpoint` 又出现在权限层 `ALWAYS_AVAILABLE_TOOLS` 中，说明代码结构上为后续启用预留了路径。
- CoStrict 的 `generate_image`、`run_slash_command` 默认在声明层存在，但最终可调用依赖实验开关。
- CoStrict 的 MCP 动态工具不固定在源码工具清单里，而是由已连接 MCP server 的实际 tool 列表生成。
