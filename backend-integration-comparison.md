# 后端接入对比报告：zgsm-sangfor/opencode vs zgsm-ai/costrict

## 1. 结论摘要

本报告对比两个项目对 CoStrict 后端的接入方式：

- `zgsm-sangfor/opencode`
  - 对比提交：`b158c71902a635994f578da65c7f824d391f7a3d`
  - 本地目录：`/tmp/zgsm-opencode`
- `zgsm-ai/costrict`
  - 对比提交：`ffb7485b775925ecc3247d6bac735f4408f0c03a`
  - 本地目录：`/tmp/zgsm-costrict`

核心结论：

**二者连接的是同一类 CoStrict 后端服务，但接入实现不一致。**

二者都使用以下后端路径：

- Chat/LLM 调用：`/chat-rag/api/v1`
- 模型列表：`/ai-gateway/api/v1/models`
- OAuth 登录：`/oidc-auth/api/v1/plugin/login`
- Token 获取/刷新：`/oidc-auth/api/v1/plugin/login/token`
- 默认 base url：`https://zgsm.sangfor.com`
- 环境变量覆盖：`COSTRICT_BASE_URL`

但是：

- `opencode` 把 CoStrict 后端封装成一个轻量的 OpenAI-compatible provider loader。
- `costrict` 使用专门的 `CostrictAiHandler`，在 OpenAI SDK 外层实现了大量业务协议：headers、metadata、prompt mode、workflow mode、quota identity、task id、prompt tags、workspace path、SSE 转换、Auto 模型选择、性能统计等。

所以如果问“后端是否同源”：**是，核心 endpoint 同源**。

如果问“接入代码、请求体、headers、stream 协议、业务 metadata 是否一致”：**否，不一致**。

## 2. 后端接口总览

| 能力 | 后端路径 | opencode | costrict |
| --- | --- | --- | --- |
| Chat/LLM | `/chat-rag/api/v1` | 作为 OpenAI-compatible baseURL | 作为 `OpenAI`/`AzureOpenAI` client baseURL |
| 模型列表 | `/ai-gateway/api/v1/models` | `fetchCoStrictModels` | `getCostrictModels` |
| OAuth 登录 | `/oidc-auth/api/v1/plugin/login` | `buildCoStrictLoginURL` | `CostrictAuthApi.loginUrl` |
| 登录轮询/token | `/oidc-auth/api/v1/plugin/login/token` | `pollLoginToken` / `refreshCoStrictToken` | `getRefreshUserToken` / login polling |
| 登录状态 | `/oidc-auth/api/v1/plugin/login/status` | 未看到等价主流程 | `CostrictAuthApi.statusUrl` |
| Logout | `/oidc-auth/api/v1/plugin/logout` | 未看到等价主流程 | `CostrictAuthApi.logoutUrl` |
| Quota | `/quota-manager/api/v1/quota` | 未看到 | `fetchCostrictQuotaInfo` |
| Invite code | `/oidc-auth/api/v1/manager/invite-code` | 未看到 | `fetchCostrictInviteCode` |

## 3. opencode 后端接入

### 3.1 入口文件

`opencode` 的 CoStrict provider 接入集中在：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/credentials.ts`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/token.ts`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/oauth-params.ts`

核心入口：

- `createCoStrictCustomLoader(provider)`  
  位置：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:148`
- `createCoStrictFetch()`  
  位置：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:28`

### 3.2 Provider loader 接口

`createCoStrictCustomLoader(provider)` 返回一个 provider loader 配置，核心结构如下：

```ts
{
  autoload: true,
  options: {
    baseURL: `${baseUrl}/chat-rag/api/v1`,
    fetch: createCoStrictFetch(),
  },
  models?: Record<string, Model>
}
```

代码位置：

- baseURL 构造：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:163`
- 有凭证时 baseURL 构造：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:216`
- fetch 注入：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:164`
- 动态 models 构造：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:223`

动态模型被转换为 `@ai-sdk/openai-compatible`：

```ts
api: {
  id: model.id,
  npm: "@ai-sdk/openai-compatible",
  url: `${baseUrl}/chat-rag/api/v1`,
}
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:229`

这说明 `opencode` 的后端接入本质是：**把 CoStrict 暴露为一个 OpenAI-compatible AI SDK provider**。

### 3.3 base URL 解析

函数：

```ts
getCoStrictBaseURL(providerApi?: string, credentialsBaseUrl?: string): string
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:51`

优先级：

1. `process.env.COSTRICT_BASE_URL`
2. `provider.api`
3. `credentials.base_url`
4. 默认值 `https://zgsm.sangfor.com`

它会移除尾部 `/chat-rag/api/v1` 和尾部 `/`：

```ts
return baseUrl.replace(/\/chat-rag\/api\/v1$/, "").replace(/\/$/, "")
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:58`

### 3.4 凭证存储接口

凭证结构：

```ts
export interface CoStrictCredentials {
  id: string
  name: string
  access_token: string
  refresh_token?: string
  state?: string
  machine_id: string
  base_url: string
  expiry_date: number
  updated_at: string
  expired_at?: string
}
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/credentials.ts:21`

凭证文件：

```text
~/.costrict/share/auth.json
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/credentials.ts:38`

读写函数：

- `loadCoStrictCredentials()`  
  位置：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/credentials.ts:63`
- `saveCoStrictCredentials(credentials)`  
  位置：`/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/credentials.ts:107`

特征：

- CLI 直接读写文件。
- `refresh_token` 和 `state` 可选。
- 写文件权限使用 `0o600`。
- 可通过 `COSTRICT_TEST_HOME` 改写 home，用于测试。

### 3.5 OAuth 参数接口

函数：

```ts
buildOAuthParams(
  includeMachineCode: boolean,
  machineId?: string,
  state?: string,
): [string, string][]
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/oauth-params.ts:17`

公共参数：

```text
provider=casdoor
plugin_version=costrict-cli-${Installation.VERSION}
vscode_version=costrict-cli-${Installation.VERSION}
uri_scheme=costrict-cli
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/oauth-params.ts:38`

如果 `includeMachineCode = true`，还会传：

```text
machine_code={machineId}
```

### 3.6 登录接口

构建登录 URL：

```ts
buildCoStrictLoginURL(baseUrl, state, machineId)
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:71`

URL：

```text
{baseUrl}/oidc-auth/api/v1/plugin/login?...params
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:81`

轮询 token：

```ts
pollLoginToken(baseUrl, state, machineId, maxAttempts?, intervalMs?, abortSignal?)
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:96`

URL：

```text
{baseUrl}/oidc-auth/api/v1/plugin/login/token?...params
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:112`

完整登录流程：

```ts
loginCoStrict(openBrowser?)
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/auth.ts:213`

流程：

1. `getCoStrictBaseURL()`
2. 生成 state
3. 生成 machine id
4. 构建登录 URL
5. 打开浏览器
6. 轮询 token
7. 保存到 `~/.costrict/share/auth.json`

### 3.7 Token 刷新接口

函数：

```ts
refreshCoStrictToken(params: {
  baseUrl: string
  refreshToken: string
  state?: string
})
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/token.ts:166`

刷新请求：

```text
GET {baseUrl}/oidc-auth/api/v1/plugin/login/token?...params
Authorization: Bearer {refreshToken}
Accept: application/json
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/token.ts:178`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/token.ts:181`

注意：

- 刷新时 `buildOAuthParams(false, undefined, params.state)`，即不带 `machine_code`。
- 401/400 会转换为 `APICallError`。

### 3.8 Chat 请求 headers

`createCoStrictFetch()` 会对所有请求动态注入：

```text
Authorization: Bearer {access_token}
HTTP-Referer: https://github.com/zgsm-ai/costrict-cli
X-Title: CoStrict-CLI
X-Costrict-Version: costrict-cli-${Installation.VERSION}
X-Request-ID: {uuidv7}
zgsm-client-id: {Installation.getInstallationId()}
zgsm-client-ide: cli
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:78`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:88`

401 处理：

- 如果 response status 是 401 且有 `refresh_token`，强制刷新 token，然后重试一次。

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:96`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/index.ts:120`

### 3.9 模型列表接口

函数：

```ts
fetchCoStrictModels(baseUrl: string, accessToken: string)
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:41`

请求：

```text
GET {baseUrl}/ai-gateway/api/v1/models
Authorization: Bearer {accessToken}
Accept: application/json
```

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:52`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:56`

返回处理：

- 读取 `response.json().data`
- 1 小时内存缓存
- 失败时使用旧缓存
- 再失败时返回默认 `gpt-4` / `gpt-3.5-turbo`

位置：

- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:31`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:68`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:88`
- `/tmp/zgsm-opencode/packages/opencode/src/costrict/provider/models.ts:103`

### 3.10 opencode 通用 LLM 层附加信息

`opencode` 在通用 `LLM.stream` 中也会补一些 provider-agnostic 信息：

- git remote repo 写到 `extra_body.repo`
  - `/tmp/zgsm-opencode/packages/opencode/src/session/llm.ts:161`
- headers 中包含：
  - `User-Agent: opencode/${Installation.VERSION}`
  - `X-Request-Id`
  - `agent-type`
  - `/tmp/zgsm-opencode/packages/opencode/src/session/llm.ts:328`

CoStrict thinking 控制在 `ProviderTransform.providerOptions` 中处理：

- `/tmp/zgsm-opencode/packages/opencode/src/provider/transform.ts:778`

## 4. costrict 后端接入

### 4.1 入口文件

`costrict` 的后端接入分散在：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts`
- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authConfig.ts`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authService.ts`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts`
- `/tmp/zgsm-costrict/src/utils/costrictUtils.ts`
- `/tmp/zgsm-costrict/src/api/providers/constants.ts`

核心类：

```ts
export class CostrictAiHandler extends BaseProvider implements SingleCompletionHandler
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:48`

核心方法：

- `createMessage(systemPrompt, messages, metadata?)`
  - `/tmp/zgsm-costrict/src/api/providers/costrict.ts:127`
- `buildHeaders(...)`
  - `/tmp/zgsm-costrict/src/api/providers/costrict.ts:358`
- `buildStreamingRequestOptions(...)`
  - `/tmp/zgsm-costrict/src/api/providers/costrict.ts:480`
- `handleOptimizedStream(...)`
  - `/tmp/zgsm-costrict/src/api/providers/costrict.ts:565`
- `processToolCalls(...)`
  - `/tmp/zgsm-costrict/src/api/providers/costrict.ts:887`

### 4.2 Client 初始化接口

`CostrictAiHandler` 构造函数中设置：

```ts
this.baseURL = `${options.costrictBaseUrl?.trim() || CostrictAuthConfig.getInstance().getDefaultApiBaseUrl()}/chat-rag/api/v1`
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:86`

根据 base URL 类型选择：

- `OpenAI`
- `AzureOpenAI`
- Azure AI Inference path

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:95`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:105`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:117`

默认 headers：

```ts
defaultHeaders: COSTRICT_DEFAULT_HEADERS
```

默认 header 常量：

```ts
{
  "HTTP-Referer": "https://github.com/RooVetGit/Roo-Cline",
  "X-Title": "Roo Code",
  "User-Agent": `RooCode/3.52.1`,
  "X-Costrict-Version": `${Package.version}`,
}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/constants.ts:3`

### 4.3 base URL 配置

默认 API base URL：

```ts
process.env.COSTRICT_BASE_URL || "https://zgsm.sangfor.com"
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authConfig.ts:28`

默认站点：

```ts
"https://costrict.ai"
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authConfig.ts:71`

### 4.4 Auth API 接口

类：

```ts
export class CostrictAuthApi
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:8`

内部路径：

```ts
loginUrl = "/oidc-auth/api/v1/plugin/login"
tokenUrl = "/oidc-auth/api/v1/plugin/login/token"
statusUrl = "/oidc-auth/api/v1/plugin/login/status"
logoutUrl = "/oidc-auth/api/v1/plugin/logout"
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:12`

配置来源：

```ts
clineProvider.getState().apiConfiguration
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:31`

刷新 token：

```ts
getRefreshUserToken(refreshToken, machineId, state)
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:112`

请求：

```text
GET {baseUrl}/oidc-auth/api/v1/plugin/login/token?...params
Authorization: Bearer {refreshToken}
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:117`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:118`

登录状态：

```ts
getUserLoginState(state, access_token)
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:85`

Logout：

```ts
logoutUser(state?, access_token?)
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authApi.ts:67`

### 4.5 OAuth 参数接口

`costrict` 使用：

```ts
getParams(state: string, ignore: string[] = [])
```

位置：

- `/tmp/zgsm-costrict/src/utils/costrictUtils.ts:10`

参数：

```text
machine_code={getClientId()}
state={state}
provider=casdoor
plugin_version={Package.version}
vscode_version={vscode.version}
uri_scheme={vscode.env.uriScheme}
```

位置：

- `/tmp/zgsm-costrict/src/utils/costrictUtils.ts:11`

与 `opencode` 的差异：

- `opencode` 的 `plugin_version` / `vscode_version` 是 `costrict-cli-${Installation.VERSION}`。
- `costrict` 的 `plugin_version` 是 VS Code extension package version。
- `costrict` 的 `vscode_version` 是真实 `vscode.version`。
- `costrict` 的 `uri_scheme` 是 `vscode.env.uriScheme`。
- `opencode` 的 `uri_scheme` 固定为 `costrict-cli`。

### 4.6 Token 存储接口

类：

```ts
export class CostrictAuthStorage
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:8`

保存 token：

```ts
saveTokens(tokens: CostrictAuthTokens)
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:25`

它写入：

- 当前 provider profile
- VS Code extension state
- `costrictRefreshToken`
- `costrictAccessToken`
- `costrictState`
- `costrictApiKeyUpdatedAt`
- `costrictApiKeyExpiredAt`

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:43`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:57`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:62`

还会同步到其他 runtime：

```ts
sendCostrictTokens(tokens)
writeCostrictRuntimeAuth(tokens.access_token, tokens.refresh_token)
ensureCompletionRuntimeReady()
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:64`
- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:66`

读取 token：

```ts
getTokens()
```

位置：

- `/tmp/zgsm-costrict/src/core/costrict/auth/authStorage.ts:78`

与 `opencode` 的差异：

- `costrict` 不以 `~/.costrict/share/auth.json` 作为主存储。
- `costrict` 以 VS Code provider state/profile 为主。
- `costrict` 会额外同步 completion runtime。
- `opencode` 直接读写 `~/.costrict/share/auth.json`，更适合 CLI。

### 4.7 Chat 请求接口

方法：

```ts
createMessage(
  systemPrompt: string,
  messages: Anthropic.Messages.MessageParam[],
  metadata?: ApiHandlerCreateMessageMetadata,
): ApiStream
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:127`

调用链：

1. 生成 request id。
2. 判断 workflow/prompt mode。
3. 更新 model info。
4. 获取 client id / workspace path / git repo。
5. `buildHeaders(...)`
6. 从 `CostrictAuthService.getTokens()` 刷新 `client.apiKey`。
7. 构造 request options。
8. 调用 `client.chat.completions.create(...).withResponse()`。
9. 读取 response header 中的 Auto 模型选择信息。
10. 将 OpenAI stream 转换为 `ApiStream`。

关键位置：

- request id：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:134`
- workflow 判断：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:135`
- headers：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:168`
- token 获取：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:177`
- request options：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:198`
- OpenAI 调用：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:225`
- Auto 模型 header：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:236`
- stream 转换：`/tmp/zgsm-costrict/src/api/providers/costrict.ts:260`

### 4.8 Chat 请求体接口

构造函数：

```ts
buildStreamingRequestOptions(
  messages,
  isDeepseekReasoner,
  isGrokXAI,
  isMiniMax,
  isQwen3,
  reasoning,
  modelInfo,
  metadata?,
  isNative?,
  repoUrl?,
)
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:480`

基础请求体：

```ts
{
  model: modelInfo.id,
  temperature,
  messages,
  stream: true,
  enable_thinking,
  thinking_budget,
  stream_options: { include_usage: true },
  ...reasoning,
  thinking: { type: "enabled" },
}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:492`

native tool 字段：

```ts
tools: this.convertToolsForOpenAI(metadata.tools)
tool_choice: metadata.tool_choice
parallel_tool_calls: metadata.parallelToolCalls ?? true
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:505`

`extra_body`：

```ts
extra_body = {
  mode: metadata?.mode,
  promptTags: metadata?.promptTags,
  repo: repoUrl,
}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:514`

随后在 `createMessage` 中还会设置：

```ts
requestOptions.extra_body.prompt_mode = fromWorkflow ? (metadata?.costrictCodeMode ?? "vibe") : "vibe"
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:210`

非 streaming 请求也支持：

```ts
extra_body: {
  prompt_mode: metadata?.mode,
  promptTags: metadata?.promptTags,
}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:552`

### 4.9 Chat 请求 headers

函数：

```ts
buildHeaders(metadata, requestId, clientId, workspacePath, chatType?)
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:358`

输出 headers：

```text
Accept-Language: {metadata.language || "en"}
HTTP-Referer: https://github.com/RooVetGit/Roo-Cline
X-Title: Roo Code
User-Agent: RooCode/3.52.1 plugin_vscode/{Package.version}
X-Costrict-Version: {Package.version}
x-quota-identity: {chatType || "system"}
X-Request-ID: {requestId}
x-user-id: {metadata.userId || ""}
zgsm-task-id: {metadata.taskId || ""}
zgsm-request-id: {requestId}
zgsm-client-id: {clientId}
zgsm-provider: {metadata.provider}
x-costrict-idea: {getEditorType()}
zgsm-project-path: {encodeURI(workspacePath)}
zgsm-prompt-tags: {metadata.promptTags || ""}
agent-type: {metadata.mode || ""}
x-caller: review-checker | chat
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:365`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:369`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:370`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:371`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:372`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:373`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:374`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:375`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:376`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:377`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:378`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:379`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:380`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:381`

这比 `opencode` 的 headers 丰富得多，明显包含 VS Code task、workspace、quota、review、prompt tag 等业务上下文。

### 4.10 模型列表接口

函数：

```ts
getCostrictModels(baseUrl?, apiKey?, openAiHeaders?, timeout?)
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:8`

请求：

```text
GET {baseUrl.replace(/\/api\/v1$/, "")}/ai-gateway/api/v1/models
Authorization: Bearer {apiKey}
X-Request-ID: {uuidv7}
x-user-id: {CostrictAuthService.getUserInfo().id}
...COSTRICT_DEFAULT_HEADERS
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:27`
- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:32`
- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:47`

失败时：

- 从 disk model cache 读取 `readModels("costrict")`

位置：

- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:55`

与 `opencode` 的差异：

- `costrict` 会带 `x-user-id`。
- `costrict` 会带默认 Roo headers。
- `costrict` 失败时读模型磁盘缓存。
- `opencode` 使用简单内存缓存和默认模型兜底。

### 4.11 Quota 和邀请码接口

`costrict` 额外提供：

```ts
fetchCostrictQuotaInfo(baseUrl?, apiKey?)
```

请求：

```text
GET {baseUrl.replace(/\/api\/v1$/, "")}/quota-manager/api/v1/quota
Authorization: Bearer {apiKey}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:61`
- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:85`

邀请码：

```ts
fetchCostrictInviteCode(baseUrl?, apiKey?)
```

请求：

```text
GET {baseUrl.replace(/\/api\/v1$/, "")}/oidc-auth/api/v1/manager/invite-code
Authorization: Bearer {apiKey}
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:97`
- `/tmp/zgsm-costrict/src/api/providers/fetchers/costrict.ts:123`

这些接口在 `opencode` 的 CoStrict provider 接入中没有看到等价实现。

### 4.12 Stream 转换接口

`costrict` 自己将 OpenAI SSE stream 转换为内部 `ApiStream`。

入口：

```ts
handleOptimizedStream(...)
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:565`

输出 chunk 类型包括：

- `automodel`
- `reasoning`
- `text`
- `fake_tool_call`
- `tool_call_partial`
- `tool_call_end`
- `usage`

tool call 处理：

```ts
processToolCalls(delta, finishReason, activeToolCallIds, requestId?)
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:887`

它将 OpenAI delta 的：

```ts
delta.tool_calls[].function.name
delta.tool_calls[].function.arguments
```

转换为：

```ts
{
  type: "tool_call_partial",
  index,
  id,
  name,
  arguments,
}
```

并在：

```ts
finishReason === "tool_calls"
```

时输出：

```ts
{ type: "tool_call_end", id }
```

位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:896`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:920`

与 `opencode` 的差异：

- `opencode` 由 AI SDK 消费 OpenAI-compatible stream 并输出标准 AI SDK events。
- `costrict` 自己把 provider stream 转为 `ApiStream`，再交给 `Task` runtime 消费。

## 5. 接口差异对照

### 5.1 Chat 接入

| 项 | opencode | costrict |
| --- | --- | --- |
| baseURL | `${baseUrl}/chat-rag/api/v1` | `${costrictBaseUrl}/chat-rag/api/v1` |
| SDK | AI SDK + `@ai-sdk/openai-compatible` | OpenAI SDK / AzureOpenAI |
| provider 类型 | custom loader | `CostrictAiHandler` |
| 请求体构建 | AI SDK + ProviderTransform | 手写 request options |
| stream 消费 | AI SDK 标准事件 | 自定义 `ApiStream` |
| tools | AI SDK tools | OpenAI tools，经 `convertToolsForOpenAI` |
| tool stream | AI SDK 处理 | 手动解析 `delta.tool_calls` |

### 5.2 Auth 接入

| 项 | opencode | costrict |
| --- | --- | --- |
| 登录 URL | `/oidc-auth/api/v1/plugin/login` | 同 |
| token URL | `/oidc-auth/api/v1/plugin/login/token` | 同 |
| status URL | 未看到主流程 | `/oidc-auth/api/v1/plugin/login/status` |
| logout URL | 未看到主流程 | `/oidc-auth/api/v1/plugin/logout` |
| 参数构造 | `buildOAuthParams` | `getParams` |
| uri scheme | `costrict-cli` | `vscode.env.uriScheme` |
| plugin version | `costrict-cli-${Installation.VERSION}` | `Package.version` |
| vscode version | `costrict-cli-${Installation.VERSION}` | `vscode.version` |
| token 存储 | `~/.costrict/share/auth.json` | VS Code provider state/profile |

### 5.3 Headers

`opencode` Chat headers 主要是：

```text
Authorization
HTTP-Referer
X-Title
X-Costrict-Version
X-Request-ID
zgsm-client-id
zgsm-client-ide
User-Agent
agent-type
```

`costrict` Chat headers 主要是：

```text
Authorization
HTTP-Referer
X-Title
X-Costrict-Version
User-Agent
Accept-Language
x-quota-identity
X-Request-ID
x-user-id
zgsm-task-id
zgsm-request-id
zgsm-client-id
zgsm-provider
x-costrict-idea
zgsm-project-path
zgsm-prompt-tags
agent-type
x-caller
```

因此 costrict 的后端接入提供了更多后端可识别的业务上下文。

### 5.4 extra_body

`opencode`：

- 通用层会加入 `repo`。
- CoStrict provider transform 支持 `chat_template_kwargs.enable_thinking`。

关键位置：

- `/tmp/zgsm-opencode/packages/opencode/src/session/llm.ts:161`
- `/tmp/zgsm-opencode/packages/opencode/src/provider/transform.ts:778`

`costrict`：

```ts
extra_body = {
  mode,
  promptTags,
  repo,
  prompt_mode,
}
```

关键位置：

- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:514`
- `/tmp/zgsm-costrict/src/api/providers/costrict.ts:210`

`costrict` 的 `extra_body` 更业务化，服务端可根据 mode/workflow/prompt tags 做路由或统计。

## 6. 是否可以视为同一套接入

### 6.1 可以视为一致的部分

以下部分可以认为是同一后端协议族：

1. base URL 默认值一致。
2. Chat endpoint 都是 `/chat-rag/api/v1`。
3. 模型列表 endpoint 都是 `/ai-gateway/api/v1/models`。
4. OAuth login/token endpoint 一致。
5. 都使用 bearer token。
6. 都支持 refresh token。
7. 都支持 OpenAI-compatible chat completions 形态。

### 6.2 不能视为一致的部分

以下部分不一致：

1. Provider client 类型不同。
2. 请求体构造不同。
3. headers 不同。
4. token 存储不同。
5. OAuth 参数中的版本和 uri scheme 不同。
6. stream 转换不同。
7. tool call 传入和解析不同。
8. 模型信息缓存和 fallback 不同。
9. costrict 多了 quota、invite、login status、logout 等接口。
10. costrict 多了 task/workspace/prompt/workflow 业务 metadata。

## 7. 对行为的影响

### 7.1 服务端路由和统计可能不同

`costrict` 会传：

- `zgsm-task-id`
- `zgsm-provider`
- `zgsm-project-path`
- `zgsm-prompt-tags`
- `x-quota-identity`
- `x-caller`
- `agent-type`
- `x-user-id`

`opencode` 没有等价完整字段。因此即使请求同一个模型，后端统计、quota、日志、路由、workflow 识别可能不同。

### 7.2 prompt mode 可能不同

`costrict` 明确传：

```ts
extra_body.prompt_mode = fromWorkflow ? (metadata?.costrictCodeMode ?? "vibe") : "vibe"
```

`opencode` 没有看到同等的 `prompt_mode` 注入。它更多依赖 agent prompt 和通用 provider options。

### 7.3 工具协议可能不同

`opencode` 由 AI SDK 将 tools 交给 OpenAI-compatible provider。

`costrict` 自己把 native tools 转成 OpenAI tools，并自己解析 `delta.tool_calls`。这会影响：

- tool name
- tool schema
- partial arguments
- tool end 判断
- fake tool call fallback

### 7.4 登录态共享并不完全对称

`opencode` 的注释写明凭证文件与 CoStrict IDE 插件兼容，但当前检查中：

- `opencode` 主路径读写 `~/.costrict/share/auth.json`。
- `costrict` 主路径读写 VS Code provider state/profile，并同步 completion runtime auth。

因此两边能否共享登录态，取决于插件是否额外写入了同一 `auth.json` 或 runtime auth 文件；仅从这两套主流程看，主存储并不一致。

## 8. 迁移建议

如果目标是让 `opencode` 与 `costrict` 后端行为更一致，建议优先补齐这些接口层差异：

1. 统一 OAuth 参数：
   - `plugin_version`
   - `vscode_version`
   - `uri_scheme`
   - `machine_code`

2. 统一 Chat headers：
   - `zgsm-client-id`
   - `zgsm-provider`
   - `zgsm-task-id` 或 session id 映射
   - `zgsm-prompt-tags`
   - `x-quota-identity`
   - `x-caller`
   - `x-user-id`
   - `zgsm-project-path`

3. 统一 `extra_body`：
   - `mode`
   - `promptTags`
   - `repo`
   - `prompt_mode`

4. 明确 token 主存储：
   - CLI 是否继续使用 `~/.costrict/share/auth.json`
   - VS Code 是否需要同步该文件
   - completion runtime auth 是否也需要 CLI 侧同步

5. 对齐模型列表处理：
   - 动态模型字段映射
   - context window
   - max tokens
   - supportsImages
   - preserveReasoning
   - fallback cache

6. 对齐 stream/tool 协议：
   - 是否完全使用 OpenAI native tool call
   - 是否需要 `tool_call_partial`/`tool_call_end`
   - fake tool call fallback 是否需要在 CLI 侧实现

## 9. 最终判断

| 判断项 | 结论 |
| --- | --- |
| 是否访问同一类 CoStrict 后端 | 是 |
| Chat endpoint 是否一致 | 基本一致 |
| Auth endpoint 是否一致 | 核心一致 |
| 模型列表 endpoint 是否一致 | 是 |
| 请求 headers 是否一致 | 否 |
| 请求 body / extra_body 是否一致 | 否 |
| token 存储是否一致 | 否 |
| stream 转换是否一致 | 否 |
| tool call 协议处理是否一致 | 否 |
| 后端业务 metadata 是否一致 | 否 |

一句话总结：

**两边的 CoStrict 后端“服务入口”基本一致，但“客户端接入协议层”不一致。`opencode` 是轻量 OpenAI-compatible 接入，`costrict` 是带完整 VS Code/task/workflow 业务语义的深度接入。**
