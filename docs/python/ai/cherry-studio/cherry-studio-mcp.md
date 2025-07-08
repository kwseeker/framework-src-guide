# Cherry-Studio 对接 MCP 原理

通过 Cherry-Studio 了解 AI 应用是怎么对接和使用 MCP Server 的。

## 1 MCP 功能测试

首先需要启用支持 Function Calling 的大模型，Cherry-Studio 中列举了几个：

```typescript
export const FUNCTION_CALLING_MODELS = [
  'gpt-4o',
  'gpt-4o-mini',
  'gpt-4',
  'gpt-4.5',
  'claude',
  'qwen',
  'hunyuan',
  'glm-4(?:-[\\w-]+)?',
  'learnlm(?:-[\\w-]+)?',
  'gemini(?:-[\\w-]+)?' // 提前排除了gemini的嵌入模型
]
```

配置中安装 MCP 服务器并启用，比如安装 `server-redis-mcp`，命令和参数：

```shell
npx
-y
@gongrzhe/server-redis-mcp
redis://127.0.0.1:6379
```

聊天对话框中启用 MCP 服务器，然后聊天，比如输入

```
> 本机 Redis 服务器中名为name的key的值是什么
```

响应：

```
> 本机 Redis 服务器中名为 name 的 key 的值是 arvin。
```

## 2 MCP 接入相关代码

这里主要关注功能提交：

```shell
feat: add Model Context Protocol (MCP) support (#2809)
```

从一些改动文件入手（从渲染进程到主进程排序）：

+ **package.json**

  引入了 `@modelcontextprotocol/sdk`，说明是使用官方 SDK对接的。

**渲染进程**：

+ **src/renderer/src/pages/settings/SettingsPage.tsx**

  MCP 配置管理的上一级配置页，配置菜单中包含 “MCP 服务器” 这一配置项。

  ```tsx
  <Container>
      <Navbar>
          {/* 如果当前 window.location 是 /settings/mcp 需要额外展示 MCP 配置的导航栏 
          	导航栏包含三个按钮：“搜索 MCP”、"编辑 MCP 配置"、"更多 MCP"
          */}
          <NavbarCenter style={{ borderRight: 'none' }}>{t('settings.title')}</NavbarCenter>
          {pathname === '/settings/mcp' && <McpSettingsNavbar />}
      </Navbar>
      <ContentContainer id="content-container">
          <SettingMenus>
              ...
              {/* MCP 服务器的菜单选项 */}
              <MenuItemLink to="/settings/mcp">
              	<MenuItem className={isRoute('/settings/mcp')}>
                		<CodeOutlined />
                		{t('settings.mcp.title')}
              	</MenuItem>
            	</MenuItemLink>
          </SettingMenus>
          <SettingContent>
              <Routes>
  			    ...
                  <Route path="mcp" element={<MCPSettings />} />
                  ...
              </Routes>
          </SettingContent>
      </ContentContainer>
  </Container>
  ```

+ **src/renderer/src/pages/settings/MCPSettings/MCPSettingsNavbar.tsx**

  导航栏包含三个按钮：“搜索 MCP”、"编辑 MCP 配置"、"更多 MCP"，主要看前两个按钮实现：

  ```typescript
  onClick={() => EventEmitter.emit('mcp:npx-search')}
  onClick={() => EditMcpJsonPopup.show()}
  ```

+ **src/renderer/src/pages/settings/MCPSettings.tsx**

  MCP 服务器配置的 React 函数组件，实现 MCP 的管理页面。

  页面包括**MCP服务器列表**和**被选中MCP服务器的配置页**。

+ **src/renderer/src/hooks/useMCPServers.ts**

  MCP Server 存取 Store 的钩子。

+ src/renderer/src/i18n/locales/en-us.json

+ **src/renderer/src/providers/AiProvider.ts**

  这里面**实现 AI 模型调用方法**，传入 mcpTools。

  ```typescript
  public async completions({
      messages,
      assistant,
      mcpTools,
      onChunk,
      onFilterMessages
  }: CompletionsParams): Promise<void> {
      return this.sdk.completions({ messages, assistant, mcpTools, onChunk, onFilterMessages })
  }
  ```

+ **src/renderer/src/providers/OpenAiProvider.ts**

  实现了大模型 API 接口调用流程，包括 MCP 工具的使用。

+ **src/renderer/src/providers/index.d.ts**

  添加大模型调用参数定义，加入 MCPTool。

+ **src/renderer/src/services/ApiService.ts**

  导出了包含 MCPTool 参数的大模型调用接口，调用 AiProvider .ts 封装的 completions() 方法。

  ```
  export async function fetchChatCompletion({
    message,
    messages,
    assistant,
    onResponse
  }: {
    message: Message
    messages: Message[]
    assistant: Assistant
    onResponse: (message: Message) => void
  })
  ```

+ **src/renderer/src/store/index.ts**

  将 mcp 状态添加持久化支持。

+ **src/renderer/src/store/mcp.ts**

  mcp 状态管理。

+ **src/renderer/src/types/index.ts**

  添加 MCP 相关接口属性定义。

**IPC**：

+ **src/preload/index.d.ts**

  将 MCP IPC 接口暴露给渲染进程。

+ **src/preload/index.ts**

  将 MCP IPC 接口暴露给渲染进程。

  ```typescript
  mcp: {
      // 会被 ipc.ts 中对应的处理函数处理
      removeServer: (server: MCPServer) => ipcRenderer.invoke('mcp:remove-server', server),
      restartServer: (server: MCPServer) => ipcRenderer.invoke('mcp:restart-server', server),
      stopServer: (server: MCPServer) => ipcRenderer.invoke('mcp:stop-server', server),
      listTools: (server: MCPServer) => ipcRenderer.invoke('mcp:list-tools', server),
      callTool: ({ server, name, args }: { server: MCPServer; name: string; args: any }) =>
        ipcRenderer.invoke('mcp:call-tool', { server, name, args }),
      getInstallInfo: () => ipcRenderer.invoke('mcp:get-install-info')
  },
  ```

**主进程**：

+ **src/main/ipc.ts**

  通过 IPC 实现 MCP Server 的管理操作的传递，包括增删改查、激活，这些功能是借助调用 mcp.ts MCPService 方法实现。

  ```typescript
  ipcMain.handle('mcp:remove-server', mcpService.removeServer)
  ipcMain.handle('mcp:restart-server', mcpService.restartServer)
  ipcMain.handle('mcp:stop-server', mcpService.stopServer)
  ipcMain.handle('mcp:list-tools', mcpService.listTools)
  ipcMain.handle('mcp:call-tool', mcpService.callTool)
  ipcMain.handle('mcp:get-install-info', mcpService.getInstallInfo)
  ```

+ **src/main/service/mcp.ts**

  初始化 MCP 客户端、传输层实现、加载 MCP Server 配置信息、提供 MCP Server 管理接口。

  MCP Server 配置信息存在 electron-store。

  **激活某个 MCP Server  内部其实是创建一个 MCP Client 并连接到 MCP Server；相反失活某个 MCP Server 是借助删除对应的 MCP Client 实现**。

## 3 MCP 配置和调用流程

### 3.1 AI应用调用MCP流程

从后面的流程看，**Cherry Studio 集成 MCP 调用外部资源的处理流程是借助 function calling 实现的**，所以处理流程和 function calling 一样。

**原理**：向 AI 模型发送消息（请求），如果启用 MCP Tool，应用需要在消息中会附带 Tool 信息（会转换为模型 API Function Calling Tool 格式），当 AI 模型识别出自己需要调用 MCP Tool 会返回调用 MCP Tool 需要的参数信息，然后当前请求结束；AI 应用获取到模型返回的参数信息，使用 MCP 客户端向 MCP Server 发送调用请求，获取调用结果后将结果数据追加到之前的消息中，再次向 AI 模型发送消息，AI 模型会根据 MCP Tool 调用结果进行响应。

MCP Server 在配置启用时通过命令行启动，比如 `npx -y @gongrzhe/server-redis-mcp redis://127.0.0.1:6379`。

**实现分析**：

CherryStudio 启动默认打开的 HomePage，其实就是聊天助手页面。

打开 HomePage 会从持久化状态中获取历史添加的助手、使用的模型以及对话主题。会加载活跃的助手、对话主题和历史消息信息并展示。

聊天对话输入框位于：HomePage 组件 -> Chat 组件

直接在聊天对话框输入一条聊天消息并发送，使用到 Textarea 组件 和 SendMessageButton 组件，分别位于：
Chat 组件 ->  Inputbar 组件 -> Textarea 组件
Chat 组件 ->  Inputbar 组件 -> ToolbarMenu 组件 -> SendMessageButton 组件

SendMessageButton 组件定义：

```tsx
// i 是 React.DetailedHTMLProps<React.HTMLAttributes<HTMLElement>, HTMLElement> 类型
const SendMessageButton: FC<Props> = ({ disabled, sendMessage }) => {
  return (
    <i
      className="iconfont icon-ic_send"
      onClick={sendMessage}
      style={{
        ...
      }}
    />
  )
}
```

sendMessage 函数定义：

```typescript
// Inputbar.tsx
// useCallback 是 React 提供的一个 Hook，用于优化函数引用稳定性，避免不必要的子组件重渲染
// 只要依赖项不变，函数引用就不变，不会重新渲染
const sendMessage = useCallback(async () => {
  // 内容、限流等检查
  // EventEmitter 是 Emittery 实例，emittery 是一个事件发布订阅库
  // 事件监听：emitter.on(EVENT_NAMES.SEND_MESSAGE, (data) => {...});
  EventEmitter.emit(EVENT_NAMES.SEND_MESSAGE)

  try {
    // 加载聊天文件、历史消息、知识库、使用的模型信息等，暂时忽略
    const uploadedFiles = await FileManager.uploadFiles(files)
    const userMessage = getUserMessage({ assistant, topic, type: 'text', content: text })
    if (uploadedFiles) {
      userMessage.files = uploadedFiles
    }
    const knowledgeBaseIds = selectedKnowledgeBases?.map((base) => base.id)
    if (knowledgeBaseIds) {
      userMessage.knowledgeBaseIds = knowledgeBaseIds
    }
    if (mentionModels) {
      userMessage.mentions = mentionModels
    }
    // 是否启用MCP服务
    if (enabledMCPs) {
      userMessage.enabledMCPs = enabledMCPs
    }
	// TODO
    userMessage.usage = await estimateMessageUsage(userMessage)
    currentMessageId.current = userMessage.id
	
    dispatch(
      // 
      _sendMessage(userMessage, assistant, topic, {
        mentions: mentionModels
      })
    )

    // 发送完成清理输入框
    ...
  } catch (error) {
    console.error('Failed to send message:', error)
  }
}, [ //依赖项
  assistant,
  dispatch,
  enabledMCPs,
  files,
  inputEmpty,
  loading,
  mentionModels,
  resizeTextArea,
  selectedKnowledgeBases,
  text,
  topic
])
```

发送消息：

```typescript
// messages.ts
export const sendMessage =
  (
    userMessage: Message,
    assistant: Assistant,
    topic: Topic,
    options?: {
      resendAssistantMessage?: Message | Message[]
      isMentionModel?: boolean
      mentions?: Model[]
    }
  ) => async (dispatch: AppDispatch, getState: () => RootState) => {	// 返回一个异步函数
  // 消息发送前通过 dispatch 缓存很多状态数据
  ...
  // 整合当前助手当前主题的历史消息，最终调用 fetchChatCompletion
  await fetchChatCompletion({
      message: { ...assistantMessage },
      messages: handleMessages(),
      assistant: assistantWithModel,
      onResponse: async (msg) => {
          // 允许在回调外维护一个最新的消息状态，每次都更新这个对象，但只通过节流函数分发到Redux
          const updateMessage = { ...msg, status: msg.status || 'pending', content: msg.content || '' }
          // 使用节流函数更新Redux
          throttledDispatch(
              assistant,
              {
                  ...assistantMessage,
                  ...updateMessage
              },
              topic.id,
              dispatch,
              getState
          )
      }
  })
}
```

fetchChatCompletion 实现：

```typescript
// ApiService.ts
export async function fetchChatCompletion({
  message,
  messages,
  assistant,
  onResponse
}: {
  message: Message
  messages: Message[]
  assistant: Assistant
  onResponse: (message: Message) => void
}) {

  const provider = getAssistantProvider(assistant)
  const webSearchProvider = WebSearchService.getWebSearchProvider()
  const AI = new AiProvider(provider)

  try {
    let _messages: Message[] = []
    let isFirstChunk = true
    let query = ''

    // 从网络搜索相关内容扩展请求消息
    if (WebSearchService.isWebSearchEnabled() && assistant.enableWebSearch && assistant.model) {
      ...
    }

    const lastUserMessage = findLast(messages, (m) => m.role === 'user')
    // Get MCP tools
    const mcpTools: MCPTool[] = []
    const enabledMCPs = lastUserMessage?.enabledMCPs

    if (enabledMCPs && enabledMCPs.length > 0) {
      for (const mcpServer of enabledMCPs) {
        const tools = await window.api.mcp.listTools(mcpServer)
        console.debug('tools', tools)
        mcpTools.push(...tools)
      }
    }
	
    // 同步调用 AI模型 completions 接口
    await AI.completions({
      messages: filterUsefulMessages(messages),
      assistant,
      onFilterMessages: (messages) => (_messages = messages),
      onChunk: ({ text, reasoning_content, usage, metrics, search, citations, mcpToolResponse, generateImage }) => {
        message.content = message.content + text || ''
        message.usage = usage
        message.metrics = metrics

        if (reasoning_content) {
          message.reasoning_content = (message.reasoning_content || '') + reasoning_content
        }

        if (search) {
          message.metadata = { ...message.metadata, groundingMetadata: search }
        }

        if (mcpToolResponse) {
          message.metadata = { ...message.metadata, mcpTools: cloneDeep(mcpToolResponse) }
        }

        if (generateImage && generateImage.images.length > 0) {
          const existingImages = message.metadata?.generateImage?.images || []
          generateImage.images = [...existingImages, ...generateImage.images]
          console.log('generateImage', generateImage)
          message.metadata = {
            ...message.metadata,
            generateImage: generateImage
          }
        }

        // Handle citations from Perplexity API
        if (isFirstChunk && citations) {
          message.metadata = {
            ...message.metadata,
            citations
          }
          isFirstChunk = false
        }

        onResponse({ ...message, status: 'pending' })
      },
      mcpTools: mcpTools
    })

    message.status = 'success'
    message = withGenerateImage(message)

    if (!message.usage || !message?.usage?.completion_tokens) {
      message.usage = await estimateMessagesUsage({
        assistant,
        messages: [..._messages, message]
      })
      // Set metrics.completion_tokens
      if (message.metrics && message?.usage?.completion_tokens) {
        if (!message.metrics?.completion_tokens) {
          message = {
            ...message,
            metrics: {
              ...message.metrics,
              completion_tokens: message.usage.completion_tokens
            }
          }
        }
      }
    }
    console.log('message', message)
  } catch (error: any) {
    if (isAbortError(error)) {
      message.status = 'paused'
    } else {
      message.status = 'error'
      message.error = formatMessageError(error)
    }
  }

  // Emit chat completion event
  EventEmitter.emit(EVENT_NAMES.RECEIVE_MESSAGE, message)
  onResponse(message)

  // Reset generating state
  store.dispatch(setGenerating(false))
  return message
}
```

completions:

```typescript
// AiProvider.ts
public async completions({
    messages,
    assistant,
    mcpTools,
    onChunk,
    onFilterMessages
}: CompletionsParams): Promise<void> {
    // sdk 是 BaseProvider 类型，有三种实现，这里只分析 OpenAIProvider (默认使用 OpenAIProvider)
    return this.sdk.completions({ messages, assistant, mcpTools, onChunk, onFilterMessages })
}

//OpenAIProvider.ts
async completions({ messages, assistant, mcpTools, onChunk, onFilterMessages }: CompletionsParams): Promise<void> {
  // 1 模型选择
  const defaultModel = getDefaultModel()
  const model = assistant.model || defaultModel
  // 2 聊天设置：上下文数量限制、最大token数量、是否启用流式输出
  const { contextCount, maxTokens, streamOutput } = getAssistantSettings(assistant)
  // 3 将图片编码后加入消息体
  messages = addImageFileToContents(messages)
  // 4 如果助手提示词存在则创建系统消息
  let systemMessage = assistant.prompt ? { role: 'system', content: assistant.prompt } : undefined
  // OpenAI特定模型的特殊处理
  if (isOpenAIoSeries(model)) {
    ...
  }

  const userMessages: ChatCompletionMessageParam[] = []
  // 5 对用户消息进行过滤，保留最新的有效消息
  const _messages = filterUserRoleStartMessages(
    filterEmptyMessages(filterContextMessages(takeRight(messages, contextCount + 1)))
  )
  // 将过滤后的消息传给上一层调用
  onFilterMessages(_messages)

  // 6 将消息转换成模型需要的格式，然后加入 userMessages
  for (const message of _messages) {
    userMessages.push(await this.getMessageParam(message, model))
  }
  // 判断模型是否是 OpenAI 的推理模型
  const isOpenAIReasoning = this.isOpenAIReasoning(model)
  const isSupportStreamOutput = () => {
    if (isOpenAIReasoning) {// 推理模型不支持流式输出，改为不使用流式输出
      return false
    }
    return streamOutput
  }

  let hasReasoningContent = false
  let lastChunk = ''
  // 定义用于检测推理是否结束的函数
  const isReasoningJustDone = (
    delta: OpenAI.Chat.Completions.ChatCompletionChunk.Choice.Delta & {
      reasoning_content?: string
      reasoning?: string
      thinking?: string
    }
  ) => {
    if (!delta?.content) return false

    // 检查当前chunk和上一个chunk的组合是否形成###Response标记
    const combinedChunks = lastChunk + delta.content
    lastChunk = delta.content

    // 检测思考结束
    if (combinedChunks.includes('###Response') || delta.content === '</think>') {
      return true
    }

    // 如果有reasoning_content或reasoning，说明是在思考中
    if (delta?.reasoning_content || delta?.reasoning || delta?.thinking) {
      hasReasoningContent = true
    }

    // 如果之前有reasoning_content或reasoning，现在有普通content，说明思考结束
    if (hasReasoningContent && delta.content) {
      return true
    }

    return false
  }

  let time_first_token_millsec = 0
  let time_first_content_millsec = 0
  const start_time_millsec = new Date().getTime()
  const lastUserMessage = _messages.findLast((m) => m.role === 'user')
  const { abortController, cleanup, signalPromise } = this.createAbortController(lastUserMessage?.id, true)
  const { signal } = abortController
  await this.checkIsCopilot()
  
  // 8 MCP Tools 列表不为空，先将 MCP Tool 转换为 OpenAI Tool 
  const tools = mcpTools && mcpTools.length > 0 ? mcpToolsToOpenAITools(mcpTools) : undefined
  // 真正的AI接口请求消息
  const reqMessages: ChatCompletionMessageParam[] = [systemMessage, ...userMessages].filter(
    Boolean
  ) as ChatCompletionMessageParam[]

  const toolResponses: MCPToolResponse[] = []
  
  // 定义 AI 响应流式输出的处理函数
  const processStream = async (stream: any, idx: number) => {
    // 非流式响应处理
    if (!isSupportStreamOutput()) {
      const time_completion_millsec = new Date().getTime() - start_time_millsec
      return onChunk({
        text: stream.choices[0].message?.content || '',
        usage: stream.usage,
        metrics: {
          completion_tokens: stream.usage?.completion_tokens,
          time_completion_millsec,
          time_first_token_millsec: 0
        }
      })
    }
      
    // 流式响应处理
    // 存储工具调用相关信息
    const final_tool_calls = {} as Record<number, ChatCompletionMessageToolCall>
    // 11 遍历响应流数据块
    for await (const chunk of stream) {
      if (window.keyv.get(EVENT_NAMES.CHAT_COMPLETION_PAUSED)) { // 检测到全局状态CHAT_COMPLETION_PAUSED为true, 中断处理
        break
      }
	  // 12 读取当前数据块中的增量数据
      const delta = chunk.choices[0]?.delta
	  // 如果增量内容中包含推理相关字段，则标记当前流正在进行推理
      if (delta?.reasoning_content || delta?.reasoning) {
        hasReasoningContent = true
      }
      
      // 耗时统计
      if (time_first_token_millsec == 0) {
        time_first_token_millsec = new Date().getTime() - start_time_millsec
      }
      if (time_first_content_millsec == 0 && isReasoningJustDone(delta)) {
        time_first_content_millsec = new Date().getTime()
      }
      const time_completion_millsec = new Date().getTime() - start_time_millsec
      const time_thinking_millsec = time_first_content_millsec ? time_first_content_millsec - start_time_millsec : 0

      // 13 提取数据块中的引用信息，TODO 什么是引用信息？
      const citations = (chunk as OpenAI.Chat.Completions.ChatCompletionChunk & { citations?: string[] })?.citations
      // TODO finish_reason 字段
      const finishReason = chunk.choices[0]?.finish_reason
	  // 14 如果数据块中增量数据包含工具调用，则解析工具调用信息并存储到 final_tool_calls
      if (delta?.tool_calls?.length) {
        // 工具调用信息在 chunk.choices[0]?.delta.tool_calls 字段中
        const chunkToolCalls = delta.tool_calls
        // 可能有多个工具调用要求
        for (const t of chunkToolCalls) {
          const { index, id, function: fn, type } = t
          const args = fn && typeof fn.arguments === 'string' ? fn.arguments : ''
          if (!(index in final_tool_calls)) {
            final_tool_calls[index] = {
              id,
              function: {
                name: fn?.name,
                arguments: args
              },
              type
            } as ChatCompletionMessageToolCall
          } else {
            final_tool_calls[index].function.arguments += args
          }
        }
        if (finishReason !== 'tool_calls') {
          continue
        }
      }

      // 15 里面执行 MCP Tool 调用
      if (finishReason === 'tool_calls' || (finishReason === 'stop' && Object.keys(final_tool_calls).length > 0)) {
        const toolCalls = Object.values(final_tool_calls).map(this.cleanToolCallArgs)
        console.log('start invoke tools', toolCalls)
        if (this.isZhipuTool(model)) {
          reqMessages.push({
            role: 'assistant',
            content: `argments=${JSON.stringify(toolCalls[0].function.arguments)}`
          })
        } else {
          reqMessages.push({
            role: 'assistant',
            tool_calls: toolCalls
          } as ChatCompletionAssistantMessageParam)
        }

        for (const toolCall of toolCalls) {
          const mcpTool = openAIToolsToMcpTool(mcpTools, toolCall)

          if (!mcpTool) {
            continue
          }

          upsertMCPToolResponse(toolResponses, { tool: mcpTool, status: 'invoking', id: toolCall.id }, onChunk)
		  // 16 调用 MCP Tool
          const toolCallResponse = await callMCPTool(mcpTool)
          // 存储 MCP Tool 调用结果
          const toolResponsContent: { type: string; text?: string; image_url?: { url: string } }[] = []
          for (const content of toolCallResponse.content) {
            if (content.type === 'text') {
              toolResponsContent.push({
                type: 'text',
                text: content.text
              })
            } else if (content.type === 'image') {
              toolResponsContent.push({
                type: 'image_url',
                image_url: { url: `data:${content.mimeType};base64,${content.data}` }
              })
            } else {
              console.warn('Unsupported content type:', content.type)
              toolResponsContent.push({
                type: 'text',
                text: 'unsupported content type: ' + content.type
              })
            }
          }

          const provider = lastUserMessage?.model?.provider
          const modelName = lastUserMessage?.model?.name

          if (
            modelName?.toLocaleLowerCase().includes('gpt') ||
            (provider === 'dashscope' && modelName?.toLocaleLowerCase().includes('qwen'))
          ) {
            // 17 将工具调用结果添加到 AI API 请求体
            reqMessages.push({
              role: 'tool',
              content: toolResponsContent,
              tool_call_id: toolCall.id
            } as ChatCompletionToolMessageParam)
          } else {
            reqMessages.push({
              role: 'tool',
              content: JSON.stringify(toolResponsContent),
              tool_call_id: toolCall.id
            } as ChatCompletionToolMessageParam)
          }
          upsertMCPToolResponse(
            toolResponses,
            { tool: mcpTool, status: 'done', response: toolCallResponse, id: toolCall.id },
            onChunk
          )
        }
        // 18 调用 OpenAI API 重新发起请求
        const newStream = await this.sdk.chat.completions
          // @ts-ignore key is not typed
          .create(
            {
              model: model.id,
              messages: reqMessages,
              temperature: this.getTemperature(assistant, model),
              top_p: this.getTopP(assistant, model),
              max_tokens: maxTokens,
              keep_alive: this.keepAliveTime,
              stream: isSupportStreamOutput(),
              tools: tools,
              ...getOpenAIWebSearchParams(assistant, model),
              ...this.getReasoningEffort(assistant, model),
              ...this.getProviderSpecificParameters(assistant, model),
              ...this.getCustomParameters(assistant)
            },
            {
              signal
            }
          )
        // 19 等待嵌套请求的处理完成
        await processStream(newStream, idx + 1)
      }

      onChunk({
        text: delta?.content || '',
        reasoning_content: delta?.reasoning_content || delta?.reasoning || '',
        usage: chunk.usage,
        metrics: {
          completion_tokens: chunk.usage?.completion_tokens,
          time_completion_millsec,
          time_first_token_millsec,
          time_thinking_millsec
        },
        citations,
        mcpToolResponse: toolResponses
      })
    }
  }
  
  // 9 调用 OpenAI API 发送请求
  const stream = await this.sdk.chat.completions
    // @ts-ignore key is not typed
    .create(
      {
        model: model.id,
        messages: reqMessages,
        temperature: this.getTemperature(assistant, model),
        top_p: this.getTopP(assistant, model),
        max_tokens: maxTokens,
        keep_alive: this.keepAliveTime,
        stream: isSupportStreamOutput(),
        tools: tools, // 将 MCP Tool 信息传给 API 接口，这样 AI 才知道当前有哪些工具可以使用
        ...getOpenAIWebSearchParams(assistant, model),
        ...this.getReasoningEffort(assistant, model),
        ...this.getProviderSpecificParameters(assistant, model),
        ...this.getCustomParameters(assistant)
      },
      {
        signal
      }
    )
  // 10 同步等待处理流式输出
  await processStream(stream, 0).finally(cleanup)
  // 捕获signal的错误
  await signalPromise?.promise?.catch((error) => {
    throw error
  })
}
```

