# 如何从 0 做一个自己的 Pi Agent：架构、方案与伪代码

## 一、概述

本文档基于 Pi Agent 源码的深入分析，提供一份从零构建类似系统的完整指南。我们将按照 Pi 的分层架构，从底层到上层逐步构建，每一层都提供架构设计、关键决策和核心伪代码。

**目标产出**：一个可扩展的终端 AI 编码助手，支持多 Provider、工具调用、会话管理和扩展系统。

**技术选型建议**：TypeScript + Node.js（与 Pi 一致），但架构设计是语言无关的。

---

## 二、整体架构设计

### 2.1 分层架构

```
┌─────────────────────────────────────────────────┐
│                    应用层                         │
│  CLI 入口 · 交互模式 · 非交互模式 · RPC 模式      │
├─────────────────────────────────────────────────┤
│                    编码层                         │
│  Agent Session · 工具集 · 会话管理 · 扩展系统      │
│  系统提示 · 模型管理 · 上下文压缩                  │
├─────────────────────────────────────────────────┤
│                    运行时层                       │
│  Agent 循环 · 状态管理 · 事件系统 · 工具编排       │
├─────────────────────────────────────────────────┤
│                    通信层                         │
│  Provider 注册表 · 流式 API · 消息格式 · 校验      │
├─────────────────────────────────────────────────┤
│                    UI 层                          │
│  终端渲染引擎 · 组件系统 · 输入处理                │
└─────────────────────────────────────────────────┘
```

### 2.2 包规划

```
my-agent-mono/
├── packages/
│   ├── llm/           # 第 1 层：LLM 通信
│   ├── agent-core/    # 第 2 层：Agent 运行时
│   ├── tui/           # 第 2 层：终端 UI
│   ├── coding-agent/  # 第 3 层：编码 Agent
│   └── cli/           # 第 4 层：CLI 入口
├── package.json       # workspace 配置
└── tsconfig.json      # TypeScript 配置
```

### 2.3 依赖关系

```
cli → coding-agent → agent-core → llm
                   → tui
```

---

## 三、第 1 层：LLM 通信层

### 3.1 架构设计

**核心问题**：如何用一套 API 统一 Anthropic、OpenAI、Google 等不同 LLM Provider 的通信协议。

**设计决策**：

| 决策 | 选项 | Pi 的选择 | 原因 |
|------|------|----------|------|
| API 统一方式 | 适配器模式 vs 中间件模式 | 适配器 + 注册表 | 运行时可扩展 |
| 流式处理 | 回调 vs AsyncIterator | AsyncIterator (EventStream) | 可 for-await 消费 |
| 类型校验 | zod vs TypeBox+AJV | TypeBox+AJV | 编译时+运行时双重校验 |
| Provider 加载 | 静态导入 vs 懒加载 | 部分懒加载 | 浏览器兼容 |

### 3.2 核心类型定义

```
// ========== 消息类型 ==========

TYPE TextContent {
    type: "text"
    text: string
}

TYPE ThinkingContent {
    type: "thinking"
    thinking: string
    signature?: string   // 用于跨轮次传递思考上下文
}

TYPE ImageContent {
    type: "image"
    data: string         // base64
    mimeType: string
}

TYPE ToolCall {
    type: "toolCall"
    id: string
    name: string
    arguments: Record<string, any>
}

TYPE UserMessage {
    role: "user"
    content: string | (TextContent | ImageContent)[]
    timestamp: number
}

TYPE AssistantMessage {
    role: "assistant"
    content: (TextContent | ThinkingContent | ToolCall)[]
    usage: Usage
    stopReason: "stop" | "length" | "toolUse" | "error" | "aborted"
    errorMessage?: string
    model: string
    provider: string
    timestamp: number
}

TYPE ToolResultMessage {
    role: "toolResult"
    toolCallId: string
    toolName: string
    content: (TextContent | ImageContent)[]
    isError: boolean
    timestamp: number
}

TYPE Message = UserMessage | AssistantMessage | ToolResultMessage

// ========== 工具类型 ==========

TYPE Tool {
    name: string
    description: string
    parameters: JSONSchema   // 使用 TypeBox 或 Zod 定义
}

// ========== 上下文 ==========

TYPE Context {
    systemPrompt?: string
    messages: Message[]
    tools?: Tool[]
}

// ========== 流式事件 ==========

TYPE AssistantMessageEvent =
    | { type: "start", partial: AssistantMessage }
    | { type: "text_delta", delta: string, partial: AssistantMessage }
    | { type: "thinking_delta", delta: string, partial: AssistantMessage }
    | { type: "toolcall_delta", delta: string, partial: AssistantMessage }
    | { type: "toolcall_end", toolCall: ToolCall, partial: AssistantMessage }
    | { type: "done", message: AssistantMessage }
    | { type: "error", error: AssistantMessage }

// ========== 模型 ==========

TYPE Model {
    id: string
    name: string
    api: string           // "anthropic" | "openai" | "google" | ...
    provider: string
    baseUrl: string
    reasoning: boolean
    contextWindow: number
    maxTokens: number
    cost: { input: number, output: number }  // $/M tokens
}
```

### 3.3 EventStream 实现

```
CLASS EventStream<Event, Result>:
    PRIVATE queue: Event[] = []
    PRIVATE waiter: Promise<void> | null = null
    PRIVATE resolveWaiter: (() => void) | null = null
    PRIVATE ended: boolean = false
    PRIVATE finalResult: Result | null = null
    PRIVATE isTerminal: (event: Event) => boolean
    PRIVATE extractResult: (event: Event) => Result

    CONSTRUCTOR(isTerminal, extractResult):
        this.isTerminal = isTerminal
        this.extractResult = extractResult

    METHOD push(event: Event):
        this.queue.push(event)
        IF this.isTerminal(event):
            this.ended = true
            this.finalResult = this.extractResult(event)
        IF this.resolveWaiter:
            this.resolveWaiter()
            this.resolveWaiter = null

    METHOD end(result: Result):
        this.ended = true
        this.finalResult = result
        IF this.resolveWaiter:
            this.resolveWaiter()

    ASYNC METHOD result(): Result:
        // 等待流结束，返回最终结果
        WHILE NOT this.ended:
            AWAIT this.waitForNext()
        RETURN this.finalResult

    METHOD [Symbol.asyncIterator]():
        RETURN {
            next: ASYNC () => {
                WHILE this.queue.length === 0 AND NOT this.ended:
                    AWAIT this.waitForNext()
                IF this.queue.length > 0:
                    RETURN { done: false, value: this.queue.shift() }
                RETURN { done: true }
            }
        }

    PRIVATE ASYNC METHOD waitForNext():
        IF this.queue.length > 0 OR this.ended:
            RETURN
        this.waiter = NEW Promise(resolve => this.resolveWaiter = resolve)
        AWAIT this.waiter
```

### 3.4 API 注册表实现

```
// ========== Provider 注册表 ==========

TYPE ProviderEntry {
    api: string
    stream: (model, context, options) => EventStream
    streamSimple: (model, context, simpleOptions) => EventStream
}

GLOBAL providerRegistry: Map<string, ProviderEntry> = new Map()

FUNCTION registerProvider(entry: ProviderEntry):
    providerRegistry.set(entry.api, entry)

FUNCTION stream(model: Model, context: Context, options?):
    provider = providerRegistry.get(model.api)
    IF NOT provider:
        THROW "No provider registered for API: " + model.api
    RETURN provider.stream(model, context, options)

FUNCTION streamSimple(model: Model, context: Context, options?):
    provider = providerRegistry.get(model.api)
    IF NOT provider:
        THROW "No provider registered for API: " + model.api
    RETURN provider.streamSimple(model, context, options)
```

### 3.5 Anthropic Provider 适配器伪代码

```
FUNCTION streamAnthropic(model, context, options):
    eventStream = NEW EventStream(
        isTerminal: (e) => e.type === "done" OR e.type === "error",
        extractResult: (e) => e.type === "done" ? e.message : e.error
    )

    ASYNC RUN:
        // 转换消息格式
        anthropicMessages = convertToAnthropicFormat(context.messages)
        anthropicTools = convertToolsToAnthropic(context.tools)

        // 构建请求
        client = NEW Anthropic({ apiKey: options.apiKey })

        partialMessage = createEmptyAssistantMessage(model)
        eventStream.push({ type: "start", partial: partialMessage })

        TRY:
            stream = client.messages.stream({
                model: model.id,
                messages: anthropicMessages,
                tools: anthropicTools,
                system: context.systemPrompt,
                max_tokens: options.maxTokens OR model.maxTokens,
                // 如果支持 thinking，添加 thinking 配置
                ...(model.reasoning ? { thinking: { type: "enabled", budget_tokens: ... } } : {})
            })

            FOR EACH event IN stream:
                SWITCH event.type:
                    CASE "content_block_start":
                        IF event.content_block.type === "text":
                            eventStream.push({ type: "text_start", ... })
                        ELSE IF event.content_block.type === "thinking":
                            eventStream.push({ type: "thinking_start", ... })
                        ELSE IF event.content_block.type === "tool_use":
                            eventStream.push({ type: "toolcall_start", ... })

                    CASE "content_block_delta":
                        // 更新 partialMessage 并推送 delta 事件
                        updatePartialMessage(partialMessage, event)
                        eventStream.push(convertDeltaEvent(event, partialMessage))

                    CASE "message_stop":
                        finalMessage = buildFinalMessage(partialMessage, stream.usage)
                        eventStream.push({ type: "done", message: finalMessage })

        CATCH error:
            errorMessage = buildErrorMessage(partialMessage, error)
            eventStream.push({ type: "error", error: errorMessage })

    RETURN eventStream

// 注册
registerProvider({
    api: "anthropic",
    stream: streamAnthropic,
    streamSimple: (model, context, options) => {
        // 将 SimpleStreamOptions.reasoning 转换为 Anthropic 特有选项
        anthropicOptions = mapReasoningToAnthropicThinking(options)
        RETURN streamAnthropic(model, context, anthropicOptions)
    }
})
```

### 3.6 工具参数校验

```
FUNCTION validateToolArguments(tool: Tool, toolCall: ToolCall):
    schema = tool.parameters
    validator = ajv.compile(schema)
    
    IF NOT validator(toolCall.arguments):
        errors = formatValidationErrors(validator.errors)
        THROW "Invalid tool arguments for " + tool.name + ": " + errors
    
    RETURN toolCall.arguments  // 校验通过，返回参数
```

---

## 四、第 2 层：Agent 运行时

### 4.1 架构设计

**核心问题**：如何实现有状态的 Agent 循环，支持工具调用、中断和恢复。

**设计决策**：

| 决策 | 选项 | 建议 | 原因 |
|------|------|------|------|
| 循环模式 | 单层 vs 双层 | 双层 | 支持 steering 和 follow-up |
| 消息类型 | 固定 vs 可扩展 | 声明合并可扩展 | 上层自定义消息 |
| 状态管理 | 可变 vs 不可变 | 混合（状态可变，消息列表不可变） | 性能与安全平衡 |

### 4.2 Agent 状态

```
TYPE AgentState {
    systemPrompt: string
    model: Model
    thinkingLevel: "off" | "low" | "medium" | "high"
    tools: AgentTool[]
    messages: AgentMessage[]
    isStreaming: boolean
    streamMessage: AgentMessage | null
    pendingToolCalls: Set<string>
    error?: string
}

TYPE AgentTool EXTENDS Tool {
    label: string
    execute: (toolCallId, params, signal?, onUpdate?) => Promise<AgentToolResult>
}

TYPE AgentToolResult {
    content: (TextContent | ImageContent)[]
    details: any
}

// 可扩展的消息类型
TYPE AgentMessage = Message | CustomMessages[keyof CustomMessages]

// 上层通过声明合并扩展
INTERFACE CustomMessages {
    // 默认为空，上层添加自定义类型
}
```

### 4.3 Agent 事件

```
TYPE AgentEvent =
    | { type: "agent_start" }
    | { type: "agent_end", messages: AgentMessage[] }
    | { type: "turn_start" }
    | { type: "turn_end", message: AgentMessage, toolResults: ToolResultMessage[] }
    | { type: "message_start", message: AgentMessage }
    | { type: "message_update", message: AgentMessage, event: AssistantMessageEvent }
    | { type: "message_end", message: AgentMessage }
    | { type: "tool_execution_start", toolCallId: string, toolName: string, args: any }
    | { type: "tool_execution_update", toolCallId: string, partialResult: any }
    | { type: "tool_execution_end", toolCallId: string, result: any, isError: boolean }
```

### 4.4 双层 Agent 循环伪代码

```
FUNCTION agentLoop(prompts, context, config, signal):
    stream = NEW EventStream<AgentEvent>(
        isTerminal: (e) => e.type === "agent_end",
        extractResult: (e) => e.messages
    )

    ASYNC RUN:
        newMessages = [...prompts]
        currentContext = { ...context, messages: [...context.messages, ...prompts] }

        stream.push({ type: "agent_start" })
        stream.push({ type: "turn_start" })

        // 推送 prompt 消息事件
        FOR EACH prompt IN prompts:
            stream.push({ type: "message_start", message: prompt })
            stream.push({ type: "message_end", message: prompt })

        // 进入主循环
        AWAIT runLoop(currentContext, newMessages, config, signal, stream)

    RETURN stream

FUNCTION runLoop(context, newMessages, config, signal, stream):
    firstTurn = true
    pendingMessages = AWAIT config.getSteeringMessages() OR []

    // ===== 外层循环：处理 follow-up =====
    WHILE true:
        hasMoreToolCalls = true
        steeringAfterTools = null

        // ===== 内层循环：处理 tool calls + steering =====
        WHILE hasMoreToolCalls OR pendingMessages.length > 0:

            IF NOT firstTurn:
                stream.push({ type: "turn_start" })
            ELSE:
                firstTurn = false

            // 注入 pending messages
            IF pendingMessages.length > 0:
                FOR EACH msg IN pendingMessages:
                    stream.push({ type: "message_start", message: msg })
                    stream.push({ type: "message_end", message: msg })
                    context.messages.push(msg)
                    newMessages.push(msg)
                pendingMessages = []

            // 调用 LLM
            assistantMsg = AWAIT streamAssistantResponse(context, config, signal, stream)
            newMessages.push(assistantMsg)

            // 检查是否出错/中止
            IF assistantMsg.stopReason === "error" OR assistantMsg.stopReason === "aborted":
                stream.push({ type: "turn_end", message: assistantMsg, toolResults: [] })
                stream.push({ type: "agent_end", messages: newMessages })
                stream.end(newMessages)
                RETURN

            // 提取工具调用
            toolCalls = assistantMsg.content.filter(c => c.type === "toolCall")
            hasMoreToolCalls = toolCalls.length > 0

            toolResults = []
            IF hasMoreToolCalls:
                // 执行工具
                execution = AWAIT executeToolCalls(context.tools, assistantMsg, signal, stream, config.getSteeringMessages)
                toolResults = execution.toolResults
                steeringAfterTools = execution.steeringMessages

                // 将工具结果加入上下文
                FOR EACH result IN toolResults:
                    context.messages.push(result)
                    newMessages.push(result)

            stream.push({ type: "turn_end", message: assistantMsg, toolResults: toolResults })

            // 检查 steering
            IF steeringAfterTools AND steeringAfterTools.length > 0:
                pendingMessages = steeringAfterTools
                steeringAfterTools = null
            ELSE:
                pendingMessages = AWAIT config.getSteeringMessages() OR []

        // ===== 外层：检查 follow-up =====
        followUpMessages = AWAIT config.getFollowUpMessages() OR []
        IF followUpMessages.length > 0:
            pendingMessages = followUpMessages
            CONTINUE
        BREAK

    stream.push({ type: "agent_end", messages: newMessages })
    stream.end(newMessages)

ASYNC FUNCTION streamAssistantResponse(context, config, signal, stream):
    // 阶段 1：上下文转换
    messages = context.messages
    IF config.transformContext:
        messages = AWAIT config.transformContext(messages, signal)

    // 阶段 2：转换为 LLM 格式
    llmMessages = AWAIT config.convertToLlm(messages)

    // 阶段 3：构建 LLM 上下文
    llmContext = {
        systemPrompt: context.systemPrompt,
        messages: llmMessages,
        tools: context.tools
    }

    // 阶段 4：动态获取 API Key
    apiKey = config.getApiKey ? AWAIT config.getApiKey(config.model.provider) : config.apiKey

    // 阶段 5：调用 LLM
    response = AWAIT streamSimple(config.model, llmContext, { apiKey, signal, ... })

    // 阶段 6：消费事件流
    partialMessage = null
    FOR EACH event IN response:
        SWITCH event.type:
            CASE "start":
                partialMessage = event.partial
                context.messages.push(partialMessage)
                stream.push({ type: "message_start", message: partialMessage })

            CASE "text_delta", "thinking_delta", "toolcall_delta", ...:
                partialMessage = event.partial
                context.messages[context.messages.length - 1] = partialMessage
                stream.push({ type: "message_update", message: partialMessage, event: event })

            CASE "done", "error":
                finalMessage = AWAIT response.result()
                context.messages[context.messages.length - 1] = finalMessage
                stream.push({ type: "message_end", message: finalMessage })
                RETURN finalMessage

    RETURN AWAIT response.result()

ASYNC FUNCTION executeToolCalls(tools, assistantMessage, signal, stream, getSteeringMessages):
    toolCalls = assistantMessage.content.filter(c => c.type === "toolCall")
    results = []
    steeringMessages = null

    FOR EACH toolCall IN toolCalls:
        tool = tools.find(t => t.name === toolCall.name)

        stream.push({ type: "tool_execution_start", toolCallId: toolCall.id, toolName: toolCall.name, args: toolCall.arguments })

        TRY:
            IF NOT tool:
                THROW "Tool not found: " + toolCall.name

            validatedArgs = validateToolArguments(tool, toolCall)

            result = AWAIT tool.execute(toolCall.id, validatedArgs, signal, (partialResult) => {
                stream.push({ type: "tool_execution_update", toolCallId: toolCall.id, partialResult: partialResult })
            })

            isError = false

        CATCH error:
            result = { content: [{ type: "text", text: error.message }], details: {} }
            isError = true

        stream.push({ type: "tool_execution_end", toolCallId: toolCall.id, result: result, isError: isError })

        toolResult = {
            role: "toolResult",
            toolCallId: toolCall.id,
            toolName: toolCall.name,
            content: result.content,
            isError: isError,
            timestamp: Date.now()
        }
        results.push(toolResult)

        stream.push({ type: "message_start", message: toolResult })
        stream.push({ type: "message_end", message: toolResult })

        // 检查 steering
        IF getSteeringMessages:
            steering = AWAIT getSteeringMessages()
            IF steering.length > 0:
                steeringMessages = steering
                // 跳过剩余工具，生成 "Skipped" 结果
                FOR EACH remainingCall IN toolCalls.slice(currentIndex + 1):
                    results.push(createSkippedToolResult(remainingCall))
                BREAK

    RETURN { toolResults: results, steeringMessages: steeringMessages }
```

### 4.5 Agent 类伪代码

```
CLASS Agent:
    PRIVATE state: AgentState
    PRIVATE listeners: Set<(AgentEvent) => void>
    PRIVATE abortController: AbortController
    PRIVATE steeringQueue: AgentMessage[]
    PRIVATE followUpQueue: AgentMessage[]
    PRIVATE convertToLlm: (AgentMessage[]) => Message[]
    PRIVATE transformContext: (AgentMessage[], signal?) => Promise<AgentMessage[]>

    CONSTRUCTOR(options):
        this.state = { ...defaultState, ...options.initialState }
        this.convertToLlm = options.convertToLlm OR defaultConvertToLlm
        this.transformContext = options.transformContext

    METHOD subscribe(fn): () => void:
        this.listeners.add(fn)
        RETURN () => this.listeners.delete(fn)

    METHOD steer(message: AgentMessage):
        this.steeringQueue.push(message)

    METHOD followUp(message: AgentMessage):
        this.followUpQueue.push(message)

    METHOD abort():
        this.abortController?.abort()

    ASYNC METHOD prompt(input: string | AgentMessage):
        IF this.state.isStreaming:
            THROW "Agent is busy. Use steer() or followUp()."

        messages = normalizeInput(input)
        AWAIT this._runLoop(messages)

    PRIVATE ASYNC METHOD _runLoop(messages?):
        this.abortController = NEW AbortController()
        this.state.isStreaming = true

        context = {
            systemPrompt: this.state.systemPrompt,
            messages: this.state.messages.slice(),
            tools: this.state.tools
        }

        config = {
            model: this.state.model,
            reasoning: this.state.thinkingLevel !== "off" ? this.state.thinkingLevel : undefined,
            convertToLlm: this.convertToLlm,
            transformContext: this.transformContext,
            getSteeringMessages: () => this.dequeueSteering(),
            getFollowUpMessages: () => this.dequeueFollowUp(),
        }

        TRY:
            stream = messages
                ? agentLoop(messages, context, config, this.abortController.signal)
                : agentLoopContinue(context, config, this.abortController.signal)

            FOR EACH event IN stream:
                this.updateState(event)
                this.emit(event)

        CATCH error:
            this.state.error = error.message
            errorMsg = createErrorAssistantMessage(error)
            this.appendMessage(errorMsg)
            this.emit({ type: "agent_end", messages: [errorMsg] })

        FINALLY:
            this.state.isStreaming = false
            this.state.streamMessage = null
```

---

## 五、第 2 层（并行）：终端 UI

### 5.1 架构设计

**核心问题**：如何高效渲染终端 UI，避免闪烁和性能问题。

**关键设计**：差分渲染——只更新屏幕上变化的行。

### 5.2 渲染引擎伪代码

```
CLASS TUI:
    PRIVATE terminal: Terminal
    PRIVATE rootContainer: Container
    PRIVATE previousBuffer: string[] = []
    PRIVATE dirtyFlag: boolean = false
    PRIVATE renderScheduled: boolean = false

    METHOD invalidate():
        this.dirtyFlag = true
        IF NOT this.renderScheduled:
            this.renderScheduled = true
            setImmediate(() => this.renderFrame())

    PRIVATE METHOD renderFrame():
        this.renderScheduled = false
        IF NOT this.dirtyFlag:
            RETURN
        this.dirtyFlag = false

        width = this.terminal.columns
        height = this.terminal.rows

        // 收集所有组件的渲染输出
        newBuffer = this.rootContainer.render(width)

        // 裁剪到屏幕高度
        newBuffer = newBuffer.slice(0, height)

        // 差分更新
        IF this.previousBuffer.length === 0 OR sizeChanged:
            // 首次渲染或尺寸变化：全量重绘
            this.terminal.write(CSI_CLEAR_SCREEN)
            FOR row = 0 TO newBuffer.length:
                this.terminal.write(moveCursor(row, 0) + newBuffer[row])
        ELSE:
            // 增量更新：只写变化行
            // 如果终端支持同步输出，开启原子更新
            IF this.terminal.supportsSyncOutput:
                this.terminal.write(CSI_BEGIN_SYNC)

            FOR row = 0 TO max(newBuffer.length, previousBuffer.length):
                IF newBuffer[row] !== previousBuffer[row]:
                    this.terminal.write(moveCursor(row, 0))
                    this.terminal.write(CSI_CLEAR_LINE)
                    IF row < newBuffer.length:
                        this.terminal.write(newBuffer[row])

            IF this.terminal.supportsSyncOutput:
                this.terminal.write(CSI_END_SYNC)

        // 定位光标
        cursorPos = findCursorMarker(newBuffer)
        IF cursorPos:
            this.terminal.write(moveCursor(cursorPos.row, cursorPos.col))
            this.terminal.write(CSI_SHOW_CURSOR)

        this.previousBuffer = newBuffer

INTERFACE Component:
    render(width: number): string[]
    handleInput?(key: Key): boolean
    invalidate?(): void
```

---

## 六、第 3 层：编码 Agent

### 6.1 架构设计

```
AgentSession（核心控制器）
├── Agent（from agent-core）
├── SessionManager（JSONL 持久化）
├── ToolSet
│   ├── ReadTool
│   ├── WriteTool
│   ├── EditTool（含模糊匹配）
│   ├── BashTool（含输出管理）
│   ├── GrepTool（ripgrep 封装）
│   └── FindTool（fd 封装）
├── ModelRegistry（多源模型注册）
├── ExtensionRunner（扩展运行时）
├── SystemPromptBuilder（动态提示构建）
└── CompactionEngine（上下文压缩）
```

### 6.2 Edit 工具（含模糊匹配）

```
FUNCTION createEditTool(cwd, operations?):
    ops = operations OR defaultFileOps

    RETURN {
        name: "edit",
        label: "edit",
        description: "Replace exact text in a file",
        parameters: { path: string, oldText: string, newText: string },

        execute: ASYNC (toolCallId, { path, oldText, newText }, signal):
            absolutePath = resolve(cwd, path)

            // 读取文件
            rawContent = AWAIT ops.readFile(absolutePath)

            // 处理 BOM
            { bom, text: content } = stripBom(rawContent)

            // 统一行尾符
            originalEnding = detectLineEnding(content)
            normalizedContent = normalizeToLF(content)
            normalizedOldText = normalizeToLF(oldText)
            normalizedNewText = normalizeToLF(newText)

            // 尝试匹配
            matchResult = fuzzyFindText(normalizedContent, normalizedOldText)

            IF NOT matchResult.found:
                THROW "Could not find the exact text in " + path

            // 检查唯一性
            fuzzyContent = normalizeForFuzzyMatch(normalizedContent)
            fuzzyOldText = normalizeForFuzzyMatch(normalizedOldText)
            occurrences = fuzzyContent.split(fuzzyOldText).length - 1

            IF occurrences > 1:
                THROW "Found " + occurrences + " occurrences. Text must be unique."

            // 执行替换
            base = matchResult.contentForReplacement
            newContent = base.substring(0, matchResult.index)
                       + normalizedNewText
                       + base.substring(matchResult.index + matchResult.matchLength)

            // 恢复格式并写入
            finalContent = bom + restoreLineEndings(newContent, originalEnding)
            AWAIT ops.writeFile(absolutePath, finalContent)

            // 生成 diff
            diff = generateUnifiedDiff(base, newContent)

            RETURN {
                content: [{ type: "text", text: "Successfully replaced text in " + path }],
                details: { diff: diff }
            }
    }

FUNCTION fuzzyFindText(content, pattern):
    // 第一步：精确匹配
    exactIndex = content.indexOf(pattern)
    IF exactIndex >= 0:
        RETURN { found: true, index: exactIndex, matchLength: pattern.length, contentForReplacement: content }

    // 第二步：Unicode 规范化匹配
    fuzzyContent = normalizeForFuzzyMatch(content)
    fuzzyPattern = normalizeForFuzzyMatch(pattern)
    fuzzyIndex = fuzzyContent.indexOf(fuzzyPattern)
    IF fuzzyIndex >= 0:
        RETURN { found: true, index: fuzzyIndex, matchLength: fuzzyPattern.length, contentForReplacement: fuzzyContent }

    RETURN { found: false }

FUNCTION normalizeForFuzzyMatch(text):
    // 替换所有 Unicode 引号变体为 ASCII
    text = text.replace(/[""]/g, '"')
    text = text.replace(/['']/g, "'")
    // 替换 Unicode 连字符为 ASCII
    text = text.replace(/[–—−]/g, "-")
    // 规范化空白
    text = text.replace(/\t/g, "    ")
    RETURN text
```

### 6.3 Bash 工具（含输出管理）

```
FUNCTION createBashTool(cwd, options?):
    ops = options?.operations OR defaultBashOps
    MAX_BYTES = 30720  // 30KB
    MAX_LINES = 200

    RETURN {
        name: "bash",
        label: "bash",
        description: "Execute bash command",
        parameters: { command: string, timeout?: number },

        execute: ASYNC (toolCallId, { command, timeout }, signal, onUpdate):
            chunks = []
            chunksBytes = 0
            totalBytes = 0
            tempFilePath = null
            tempFileStream = null

            handleData = (data):
                totalBytes += data.length
                chunks.push(data)
                chunksBytes += data.length

                // 超阈值时写入临时文件
                IF totalBytes > MAX_BYTES AND NOT tempFilePath:
                    tempFilePath = createTempFile()
                    tempFileStream = createWriteStream(tempFilePath)
                    FOR EACH chunk IN chunks:
                        tempFileStream.write(chunk)

                IF tempFileStream:
                    tempFileStream.write(data)

                // 修剪旧 chunk
                WHILE chunksBytes > MAX_BYTES * 2 AND chunks.length > 1:
                    removed = chunks.shift()
                    chunksBytes -= removed.length

                // 流式更新
                IF onUpdate:
                    truncated = truncateTail(Buffer.concat(chunks).toString())
                    onUpdate({ content: [{ type: "text", text: truncated.content }], details: {} })

            { exitCode } = AWAIT ops.exec(command, cwd, { onData: handleData, signal, timeout })

            // 最终输出处理
            fullOutput = Buffer.concat(chunks).toString()
            truncation = truncateTail(fullOutput)
            outputText = truncation.content OR "(no output)"

            IF truncation.truncated:
                outputText += "\n\n[Output truncated. Full output: " + tempFilePath + "]"

            IF exitCode !== 0:
                THROW outputText + "\n\nCommand exited with code " + exitCode

            RETURN { content: [{ type: "text", text: outputText }], details: {} }
    }

FUNCTION truncateTail(text):
    lines = text.split("\n")
    bytes = Buffer.byteLength(text)

    IF lines.length <= MAX_LINES AND bytes <= MAX_BYTES:
        RETURN { content: text, truncated: false }

    // 从尾部保留
    keptLines = []
    keptBytes = 0
    FOR i = lines.length - 1 DOWNTO 0:
        lineBytes = Buffer.byteLength(lines[i])
        IF keptBytes + lineBytes > MAX_BYTES:
            BREAK
        keptLines.unshift(lines[i])
        keptBytes += lineBytes
        IF keptLines.length >= MAX_LINES:
            BREAK

    RETURN {
        content: keptLines.join("\n"),
        truncated: true,
        totalLines: lines.length,
        outputLines: keptLines.length
    }
```

### 6.4 JSONL 会话管理器

```
CLASS SessionManager:
    PRIVATE filePath: string
    PRIVATE entries: Map<string, SessionEntry>  // id → entry
    PRIVATE leafId: string  // 当前叶节点

    METHOD createSession(cwd):
        sessionId = generateUUID()
        header = {
            type: "session",
            id: sessionId,
            timestamp: now(),
            cwd: cwd
        }
        this.filePath = getSessionPath(cwd, sessionId)
        this.appendEntry(header)
        this.leafId = sessionId
        RETURN sessionId

    METHOD append(entry):
        entry.id = generateUUID()
        entry.parentId = this.leafId
        entry.timestamp = now()
        this.entries.set(entry.id, entry)
        this.appendToFile(entry)
        this.leafId = entry.id

    METHOD fork(fromEntryId):
        // 不改变文件，只移动 leafId 指针
        this.leafId = fromEntryId
        // 后续 append 会以 fromEntryId 为 parent，形成分支

    METHOD buildContext():
        // 从 leafId 回溯到根
        path = []
        current = this.leafId
        WHILE current !== null:
            entry = this.entries.get(current)
            path.unshift(entry)
            current = entry.parentId

        // 找到最后一个 compaction 节点
        lastCompaction = findLast(path, e => e.type === "compaction")
        IF lastCompaction:
            path = path.slice(path.indexOf(lastCompaction))

        // 构建消息列表
        messages = []
        FOR EACH entry IN path:
            message = extractMessage(entry)
            IF message:
                messages.push(message)

        RETURN messages

    PRIVATE METHOD appendToFile(entry):
        line = JSON.stringify(entry) + "\n"
        appendFileSync(this.filePath, line)

    METHOD loadFromFile(filePath):
        content = readFileSync(filePath, "utf-8")
        lines = content.trim().split("\n")
        FOR EACH line IN lines:
            entry = JSON.parse(line)
            this.entries.set(entry.id, entry)
        // leafId = 最后一行的 id
        this.leafId = entries[entries.length - 1].id
```

### 6.5 上下文压缩引擎

```
FUNCTION shouldCompact(contextTokens, contextWindow, settings):
    IF NOT settings.enabled:
        RETURN false
    RETURN contextTokens > contextWindow - settings.reserveTokens

FUNCTION estimateTokens(message):
    // 简单但保守的估算：字符数 / 4
    chars = countTextCharacters(message)
    RETURN ceil(chars / 4)

FUNCTION prepareCompaction(entries, settings):
    // 找到上一次 compaction
    prevCompaction = findLast(entries, e => e.type === "compaction")
    startIndex = prevCompaction ? indexOf(prevCompaction) + 1 : 0

    // 估算总 Token
    messages = extractMessages(entries, startIndex)
    totalTokens = sum(messages.map(estimateTokens))

    // 找切割点（从尾部保留 keepRecentTokens）
    accumulated = 0
    cutIndex = startIndex
    FOR i = entries.length - 1 DOWNTO startIndex:
        IF entries[i].type !== "message":
            CONTINUE
        accumulated += estimateTokens(entries[i].message)
        IF accumulated >= settings.keepRecentTokens:
            cutIndex = findNearestValidCutPoint(entries, i, startIndex)
            BREAK

    // 收集待摘要消息
    messagesToSummarize = extractMessages(entries, startIndex, cutIndex)
    keptMessages = extractMessages(entries, cutIndex)

    RETURN {
        firstKeptEntryId: entries[cutIndex].id,
        messagesToSummarize: messagesToSummarize,
        tokensBefore: totalTokens,
        previousSummary: prevCompaction?.summary
    }

ASYNC FUNCTION compact(preparation, model, apiKey):
    // 构建摘要提示
    conversationText = serializeConversation(preparation.messagesToSummarize)

    prompt = "<conversation>\n" + conversationText + "\n</conversation>\n\n"
    IF preparation.previousSummary:
        prompt += "<previous-summary>\n" + preparation.previousSummary + "\n</previous-summary>\n\n"
        prompt += UPDATE_SUMMARIZATION_PROMPT
    ELSE:
        prompt += INITIAL_SUMMARIZATION_PROMPT

    // 调用 LLM 生成摘要
    response = AWAIT completeSimple(model, {
        systemPrompt: "You are a conversation summarizer...",
        messages: [{ role: "user", content: prompt, timestamp: now() }]
    }, { apiKey })

    summary = extractText(response)

    RETURN {
        summary: summary,
        firstKeptEntryId: preparation.firstKeptEntryId,
        tokensBefore: preparation.tokensBefore
    }
```

### 6.6 系统提示构建器

```
FUNCTION buildSystemPrompt(options):
    tools = options.selectedTools OR ["read", "bash", "edit", "write"]
    cwd = options.cwd OR process.cwd()

    // 构建工具列表
    toolsList = tools.map(name => "- " + name + ": " + getToolDescription(name)).join("\n")

    // 构建使用指南
    guidelines = []
    IF tools.includes("bash") AND NOT tools.includes("grep"):
        guidelines.push("Use bash for file operations like ls, grep, find")
    IF tools.includes("read") AND tools.includes("edit"):
        guidelines.push("Use read to examine files before editing")
    IF tools.includes("edit"):
        guidelines.push("Use edit for precise changes (old text must match exactly)")
    guidelines.push("Be concise in your responses")
    guidelines.push("Show file paths clearly when working with files")

    guidelinesText = guidelines.map(g => "- " + g).join("\n")

    // 组合提示
    prompt = "You are an expert coding assistant.\n\n"
    prompt += "Available tools:\n" + toolsList + "\n\n"
    prompt += "Guidelines:\n" + guidelinesText + "\n\n"

    // 附加项目上下文 (AGENTS.md)
    contextFiles = findContextFiles(cwd)  // 从 cwd 向上查找
    IF contextFiles.length > 0:
        prompt += "# Project Context\n\n"
        FOR EACH file IN contextFiles:
            prompt += "## " + file.path + "\n\n" + file.content + "\n\n"

    // 附加技能
    IF options.skills AND options.skills.length > 0:
        prompt += formatSkillsForPrompt(options.skills)

    prompt += "\nCurrent date and time: " + formatDateTime(now())
    prompt += "\nCurrent working directory: " + cwd

    RETURN prompt
```

### 6.7 扩展系统

```
CLASS ExtensionRunner:
    PRIVATE extensions: Extension[]
    PRIVATE eventHandlers: Map<string, Function[]>

    ASYNC METHOD loadExtension(path):
        // 使用 jiti 运行时转译 TypeScript
        jiti = createJiti({ virtualModules: { "my-agent": currentPackageExports } })
        module = jiti(path)
        factory = module.default OR module.activate

        // 创建扩展 API
        api = {
            registerTool: (tool) => this.registerTool(tool),
            registerCommand: (cmd) => this.registerCommand(cmd),
            registerShortcut: (shortcut) => this.registerShortcut(shortcut),
            on: (event, handler) => this.addEventListener(event, handler),
        }

        // 初始化扩展
        AWAIT factory(api)

    METHOD wrapTools(tools):
        RETURN tools.map(tool => ({
            ...tool,
            execute: ASYNC (id, args, signal, onUpdate) => {
                // 前置拦截
                FOR EACH handler IN this.eventHandlers.get("tool_call"):
                    result = AWAIT handler({ toolName: tool.name, args, toolCallId: id })
                    IF result.block:
                        THROW "Blocked by extension: " + result.reason

                // 执行原始工具
                toolResult = AWAIT tool.execute(id, args, signal, onUpdate)

                // 后置拦截
                FOR EACH handler IN this.eventHandlers.get("tool_result"):
                    toolResult = AWAIT handler({ toolName: tool.name, result: toolResult }) OR toolResult

                RETURN toolResult
            }
        }))
```

---

## 七、第 4 层：CLI 入口

### 7.1 主流程

```
FUNCTION main():
    // 阶段 1：基础参数解析
    args = parseBasicArgs(process.argv)

    // 阶段 2：加载扩展（发现额外 CLI flags）
    extensions = loadExtensions(args.extensions)
    extraFlags = collectExtensionFlags(extensions)

    // 阶段 3：完整参数解析（包含扩展 flags）
    fullArgs = parseFullArgs(process.argv, extraFlags)

    // 阶段 4：初始化
    modelRegistry = createModelRegistry()
    sessionManager = NEW SessionManager()
    settingsManager = NEW SettingsManager()
    agent = NEW Agent({ ... })

    agentSession = createAgentSession({
        agent, sessionManager, settingsManager,
        cwd: process.cwd(),
        modelRegistry,
        resourceLoader: NEW ResourceLoader()
    })

    // 阶段 5：启动交互模式
    SWITCH fullArgs.mode:
        CASE "interactive":
            tui = NEW TUI()
            interactiveMode = NEW InteractiveMode(tui, agentSession)
            AWAIT interactiveMode.run()

        CASE "print":
            AWAIT runPrintMode(agentSession, fullArgs.prompt)

        CASE "rpc":
            AWAIT runRpcMode(agentSession)
```

---

## 八、开发路线图

### 8.1 Phase 1：最小可用（2-3 周）

**目标**：能在终端中与一个 LLM Provider 进行对话。

**任务**：
1. 实现 EventStream 基础设施
2. 实现一个 Provider（建议从 Anthropic 或 OpenAI 开始）
3. 实现基础消息类型和序列化
4. 实现简单的 REPL 循环（不需要 TUI，用 readline 即可）

### 8.2 Phase 2：Agent 能力（2-3 周）

**目标**：Agent 能够调用工具完成简单的编码任务。

**任务**：
1. 实现 Agent 循环（先单层，不需要 steering/follow-up）
2. 实现 read 和 write 工具
3. 实现 bash 工具（含输出截断）
4. 实现 edit 工具（先精确匹配，后加模糊匹配）
5. 实现基础系统提示

### 8.3 Phase 3：持久化（1-2 周）

**目标**：对话可以持久化和恢复。

**任务**：
1. 实现 JSONL 会话管理器
2. 实现会话恢复（`--resume`）
3. 实现基础的上下文压缩

### 8.4 Phase 4：交互体验（2-3 周）

**目标**：全屏终端 UI，差分渲染。

**任务**：
1. 实现基础 TUI 渲染引擎（差分渲染）
2. 实现 Markdown 渲染组件
3. 实现多行输入编辑器
4. 实现消息列表组件
5. 实现 diff 预览（edit 工具结果）

### 8.5 Phase 5：多 Provider（1-2 周）

**目标**：支持多个 LLM Provider。

**任务**：
1. 实现 API 注册表
2. 添加 Anthropic、OpenAI、Google Provider
3. 实现跨 Provider 消息转换
4. 实现运行时模型切换

### 8.6 Phase 6：扩展系统（2-3 周）

**目标**：支持 TypeScript 扩展。

**任务**：
1. 集成 jiti 加载器
2. 设计扩展 API 接口
3. 实现工具包装（tool_call/tool_result 拦截）
4. 实现事件系统
5. 编写示例扩展

### 8.7 Phase 7：高级功能（持续迭代）

- 双层 Agent 循环（steering + follow-up）
- 树形会话（fork、树导航）
- OAuth 认证
- RPC 模式
- Web UI
- 自动补全
- 主题系统
- 包管理器

---

## 九、关键技术选型建议

### 9.1 语言选择

| 语言 | 优势 | 劣势 | 适合场景 |
|------|------|------|---------|
| TypeScript | 类型安全、npm 生态、浏览器兼容 | 需要编译/转译 | Web 集成、多平台 |
| Python | AI 生态最强、LLM SDK 最全 | 类型系统弱、打包困难 | 快速原型、数据处理 |
| Rust | 性能、单二进制分发 | 开发周期长、LLM SDK 少 | 性能敏感场景 |
| Go | 编译快、单二进制、并发好 | 泛型弱、类型系统受限 | CLI 工具 |

**建议**：如果你的目标是一个完整的编码 Agent 产品，TypeScript 是最平衡的选择。如果只是个人工具，Python 开发最快。

### 9.2 依赖库选型

| 功能 | 推荐库 | 备选 |
|------|-------|------|
| LLM SDK | 各 Provider 官方 SDK | 自己封装 HTTP |
| Schema 校验 | TypeBox + AJV | Zod |
| 终端 UI | 自己实现（参考 Pi TUI） | Ink (React)、Blessed |
| Markdown 渲染 | marked | markdown-it |
| Diff 生成 | diff (npm) | jsdiff |
| 文件搜索 | ripgrep (外部命令) | node:fs/glob |
| TypeScript 运行时 | jiti | tsx、ts-node |
| 测试 | Vitest | Jest |

### 9.3 项目结构建议

```
my-coding-agent/
├── packages/
│   ├── llm/                    # LLM 通信层
│   │   ├── src/
│   │   │   ├── types.ts        # 核心类型
│   │   │   ├── stream.ts       # EventStream
│   │   │   ├── registry.ts     # API 注册表
│   │   │   ├── validate.ts     # 参数校验
│   │   │   └── providers/
│   │   │       ├── anthropic.ts
│   │   │       ├── openai.ts
│   │   │       └── google.ts
│   │   └── package.json
│   │
│   ├── agent/                  # Agent 运行时
│   │   ├── src/
│   │   │   ├── types.ts        # Agent 类型
│   │   │   ├── agent.ts        # Agent 类
│   │   │   └── loop.ts         # Agent 循环
│   │   └── package.json
│   │
│   ├── tools/                  # 编码工具
│   │   ├── src/
│   │   │   ├── read.ts
│   │   │   ├── write.ts
│   │   │   ├── edit.ts
│   │   │   ├── edit-diff.ts
│   │   │   ├── bash.ts
│   │   │   └── truncate.ts
│   │   └── package.json
│   │
│   ├── session/                # 会话管理
│   │   ├── src/
│   │   │   ├── manager.ts
│   │   │   ├── compaction.ts
│   │   │   └── types.ts
│   │   └── package.json
│   │
│   ├── tui/                    # 终端 UI
│   │   ├── src/
│   │   │   ├── engine.ts       # 渲染引擎
│   │   │   ├── components/
│   │   │   │   ├── markdown.ts
│   │   │   │   ├── editor.ts
│   │   │   │   └── message-list.ts
│   │   │   └── input.ts        # 输入处理
│   │   └── package.json
│   │
│   └── cli/                    # CLI 入口
│       ├── src/
│       │   ├── main.ts
│       │   ├── interactive.ts
│       │   └── system-prompt.ts
│       └── package.json
│
├── package.json
└── tsconfig.json
```

---

## 十、避坑指南

### 10.1 LLM Provider 兼容性

**坑**：不同 Provider 对消息格式的要求差异巨大。

**建议**：
- 实现 `transformMessages()` 函数处理跨 Provider 消息兼容
- 维护每个 Provider 的兼容性标志（如 `supportsThinking`、`requiresAlternating`）
- 对 OpenAI 兼容 API 使用 feature detection 而非硬编码

### 10.2 流式解析

**坑**：工具调用参数是 JSON，但在流式传输中是逐字到达的，不完整的 JSON 无法解析。

**建议**：
- 使用 `partial-json` 类库解析不完整的 JSON
- 在 `toolcall_delta` 事件中只传递 delta 文本，在 `toolcall_end` 时再解析完整参数

### 10.3 终端渲染

**坑**：宽字符（CJK）、ANSI 转义序列、emoji 的宽度计算。

**建议**：
- 使用 `get-east-asian-width` 库计算字符宽度
- 在计算显示宽度时剥离 ANSI 转义序列
- 测试时使用包含 CJK 字符的内容

### 10.4 进程管理

**坑**：bash 工具创建的子进程可能产生孙进程，kill 子进程不会 kill 孙进程。

**建议**：
- 使用 `detached: true` 创建独立进程组
- 实现 `killProcessTree()` 函数递归 kill 所有后代进程
- 在 Linux/macOS 上使用 `process.kill(-pid, signal)` kill 进程组

### 10.5 会话文件损坏

**坑**：程序崩溃时 JSONL 文件可能写入不完整的行。

**建议**：
- 读取时忽略最后一行如果它无法解析为有效 JSON
- 使用 `appendFileSync`（小于 PIPE_BUF 的写入是原子的）
- 考虑定期写入校验和

### 10.6 上下文溢出

**坑**：不同 Provider 的上下文溢出错误格式不同，有些返回 400，有些返回 500。

**建议**：
- 实现 `isContextOverflow()` 函数，匹配各 Provider 的错误格式
- 溢出时自动触发压缩然后重试
- 保留足够的 token 余量（reserveTokens）

---

## 十一、总结

构建一个类 Pi 的编码 Agent 是一个有挑战但可行的项目。关键是遵循分层架构原则：

1. **先做通信层**：确保可以稳定地与至少一个 LLM Provider 流式通信
2. **再做运行时**：实现 Agent 循环和工具执行
3. **然后做工具**：read/write/edit/bash 四个核心工具
4. **接着做持久化**：JSONL 会话管理和上下文压缩
5. **最后做交互**：TUI 或其他交互界面

每一层都可以独立开发和测试，不需要从第一天就构建完整系统。从最小可用版本开始，逐步添加功能。

Pi 源码最值得学习的不是具体实现，而是它的架构哲学：**将复杂问题分解为独立的可组合模块，每个模块解决一个明确的问题，通过清晰的接口协作**。
