# Spring AI - 大模型调用完整流程详解

> 从用户代码到AI响应，深度剖析每一步的执行过程

## 目录

- [一、调用流程总览](#一调用流程总览)
- [二、阶段一：请求构建](#二阶段一请求构建)
- [三、阶段二：Advisor链执行](#三阶段二advisor链执行)
- [四、阶段三：ChatModel调用](#四阶段三chatmodel调用)
- [五、阶段四：底层API交互](#五阶段四底层api交互)
- [六、阶段五：响应构建](#六阶段五响应构建)
- [七、阶段六：Advisor后处理](#七阶段六advisor后处理)
- [八、阶段七：响应返回](#八阶段七响应返回)
- [九、完整示例分析](#九完整示例分析)
- [十、流式调用流程](#十流式调用流程)

---

## 一、调用流程总览

### 1.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                     用户代码                                  │
│  chatClient.prompt().user("你好").call().content()           │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段1: 请求构建 (DefaultChatClientRequestSpec)              │
│  • 构建user/system消息                                        │
│  • 渲染模板参数                                               │
│  • 合并配置选项                                               │
│  • 创建ChatClientRequest                                      │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段2: Advisor链执行 (DefaultAroundAdvisorChain)            │
│  • Memory Advisor (添加对话历史)                             │
│  • RAG Advisor (检索知识库)                                   │
│  • Logger Advisor (记录日志)                                 │
│  • ... 其他自定义Advisor                                      │
│  • ChatModelCallAdvisor (终点)                                │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段3: ChatModel调用 (OpenAiChatModel)                       │
│  • 合并Prompt选项                                             │
│  • 创建ChatCompletionRequest                                  │
│  • 处理Tool调用                                               │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段4: 底层API交互 (OpenAiApi)                               │
│  • HTTP请求构建                                               │
│  • 调用OpenAI API                                             │
│  • 重试机制                                                   │
│  • 响应解析                                                   │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段5: 响应构建 (ChatResponse)                               │
│  • 解析Generation                                             │
│  • 提取Usage信息                                              │
│  • 构建Metadata                                               │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段6: Advisor后处理                                         │
│  • Memory Advisor保存响应                                     │
│  • Logger Advisor记录                                        │
│  • ... 其他后处理                                             │
└────────────┬─────────────────────────────────────────────────┘
             │
             ▼
┌──────────────────────────────────────────────────────────────┐
│  阶段7: 响应返回给用户                                        │
│  • 提取content字符串                                          │
│  • 或返回完整ChatResponse                                     │
│  • 或转换为实体对象                                           │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 时序图

```
用户代码                ChatClient         AdvisorChain       ChatModel          OpenAI API
   │                       │                   │                 │                   │
   │ call().content()      │                   │                 │                   │
   ├──────────────────────>│                   │                 │                   │
   │                       │                   │                 │                   │
   │                       │ 1. 构建Request    │                 │                   │
   │                       │──────────┐        │                 │                   │
   │                       │          │        │                 │                   │
   │                       │<─────────┘        │                 │                   │
   │                       │                   │                 │                   │
   │                       │ 2. nextCall()     │                 │                   │
   │                       ├──────────────────>│                 │                   │
   │                       │                   │                 │                   │
   │                       │                   │ 3. before()     │                   │
   │                       │                   │ (Memory, RAG...)│                   │
   │                       │                   │────────┐        │                   │
   │                       │                   │        │        │                   │
   │                       │                   │<───────┘        │                   │
   │                       │                   │                 │                   │
   │                       │                   │ 4. call(prompt) │                   │
   │                       │                   ├────────────────>│                   │
   │                       │                   │                 │                   │
   │                       │                   │                 │ 5. HTTP POST      │
   │                       │                   │                 ├──────────────────>│
   │                       │                   │                 │                   │
   │                       │                   │                 │ 6. JSON Response  │
   │                       │                   │                 │<──────────────────│
   │                       │                   │                 │                   │
   │                       │                   │ 7. ChatResponse │                   │
   │                       │                   │<────────────────│                   │
   │                       │                   │                 │                   │
   │                       │                   │ 8. after()      │                   │
   │                       │                   │ (保存Memory...)  │                   │
   │                       │                   │────────┐        │                   │
   │                       │                   │        │        │                   │
   │                       │                   │<───────┘        │                   │
   │                       │                   │                 │                   │
   │                       │ 9. Response       │                 │                   │
   │                       │<──────────────────│                 │                   │
   │                       │                   │                 │                   │
   │ 10. String content    │                   │                 │                   │
   │<──────────────────────│                   │                 │                   │
   │                       │                   │                 │                   │
```

---

## 二、阶段一：请求构建

### 2.1 用户代码入口

```java
// 用户代码
String response = chatClient.prompt()     // 返回 ChatClientRequestSpec
    .user("你好")                          // 设置user消息
    .call()                                // 返回 CallResponseSpec
    .content();                            // 获取响应文本
```

### 2.2 步骤详解

#### **步骤1：创建RequestSpec**

```java
// DefaultChatClient.java (L95-97)
@Override
public ChatClientRequestSpec prompt() {
    // 复制默认配置，创建新的RequestSpec
    return new DefaultChatClientRequestSpec(this.defaultChatClientRequest);
}
```

**关键点**：
- 每次调用`prompt()`都会创建一个**新的**`RequestSpec`实例
- 继承了Builder中设置的所有默认值（system、advisors、options等）
- 保证了线程安全和调用独立性

#### **步骤2：设置User消息**

```java
// DefaultChatClient.java (L943-947)
@Override
public ChatClientRequestSpec user(String text) {
    Assert.hasText(text, "text cannot be null or empty");
    this.userText = text;  // 保存user文本
    return this;            // 链式调用
}
```

**存储的数据**：
```java
private String userText = "你好";                  // 用户消息文本
private Map<String, Object> userParams = {};       // 模板参数（如果有）
private List<Media> media = [];                     // 多模态内容（如果有）
private Map<String, Object> userMetadata = {};     // 元数据（如果有）
```

#### **步骤3：调用call()方法**

```java
// DefaultChatClient.java (L990-994)
@Override
public CallResponseSpec call() {
    // 1. 构建Advisor链
    BaseAdvisorChain advisorChain = buildAdvisorChain();
    
    // 2. 将RequestSpec转换为ChatClientRequest
    ChatClientRequest chatClientRequest = 
        DefaultChatClientUtils.toChatClientRequest(this);
    
    // 3. 创建ResponseSpec
    return new DefaultCallResponseSpec(
        chatClientRequest, 
        advisorChain,
        this.observationRegistry, 
        this.observationConvention
    );
}
```

#### **步骤4：构建Advisor链**

```java
// DefaultChatClient.java (L1003-1010)
private BaseAdvisorChain buildAdvisorChain() {
    // 1. 在Advisor列表末尾添加ChatModelCallAdvisor
    // 这是最后执行的Advisor，负责实际调用ChatModel
    this.advisors.add(
        ChatModelCallAdvisor.builder()
            .chatModel(this.chatModel)
            .build()
    );
    
    // 2. 添加流式Advisor（用于stream()调用）
    this.advisors.add(
        ChatModelStreamAdvisor.builder()
            .chatModel(this.chatModel)
            .build()
    );
    
    // 3. 构建Advisor链
    return DefaultAroundAdvisorChain.builder(this.observationRegistry)
        .pushAll(this.advisors)
        .build();
}
```

**Advisor排序**：
```java
// 按Order值排序，值越小越先执行
advisors = [
    MemoryAdvisor (order=0),           // 先执行
    RAGAdvisor (order=100),
    LoggerAdvisor (order=200),
    ChatModelCallAdvisor (order=2147483647)  // 最后执行 (LOWEST_PRECEDENCE)
]
```

#### **步骤5：转换为ChatClientRequest**

```java
// DefaultChatClientUtils.java (L54-144)
static ChatClientRequest toChatClientRequest(
        DefaultChatClientRequestSpec inputRequest) {
    
    List<Message> processedMessages = new ArrayList<>();
    
    // 1. 处理SystemMessage（第一个）
    String processedSystemText = inputRequest.getSystemText();
    if (StringUtils.hasText(processedSystemText)) {
        // 如果有模板参数，渲染模板
        if (!CollectionUtils.isEmpty(inputRequest.getSystemParams())) {
            processedSystemText = PromptTemplate.builder()
                .template(processedSystemText)
                .variables(inputRequest.getSystemParams())
                .renderer(inputRequest.getTemplateRenderer())
                .build()
                .render();
        }
        processedMessages.add(
            SystemMessage.builder()
                .text(processedSystemText)
                .metadata(inputRequest.getSystemMetadata())
                .build()
        );
    }
    
    // 2. 添加中间的Messages（如果通过messages()方法添加）
    if (!CollectionUtils.isEmpty(inputRequest.getMessages())) {
        processedMessages.addAll(inputRequest.getMessages());
    }
    
    // 3. 处理UserMessage（最后一个）
    String processedUserText = inputRequest.getUserText();
    if (StringUtils.hasText(processedUserText)) {
        // 如果有模板参数，渲染模板
        if (!CollectionUtils.isEmpty(inputRequest.getUserParams())) {
            processedUserText = PromptTemplate.builder()
                .template(processedUserText)
                .variables(inputRequest.getUserParams())
                .renderer(inputRequest.getTemplateRenderer())
                .build()
                .render();
        }
        processedMessages.add(
            UserMessage.builder()
                .text(processedUserText)
                .media(inputRequest.getMedia())
                .metadata(inputRequest.getUserMetadata())
                .build()
        );
    }
    
    // 4. 处理ChatOptions（合并tool相关配置）
    ChatOptions processedChatOptions = inputRequest.getChatOptions();
    // ... 处理tool callbacks和tool context ...
    
    // 5. 构建最终的Prompt
    Prompt prompt = Prompt.builder()
        .messages(processedMessages)
        .chatOptions(processedChatOptions)
        .build();
    
    // 6. 构建ChatClientRequest
    return ChatClientRequest.builder()
        .prompt(prompt)
        .context(new ConcurrentHashMap<>(inputRequest.getAdvisorParams()))
        .build();
}
```

**生成的数据结构**：
```java
ChatClientRequest {
    prompt: Prompt {
        messages: [
            UserMessage {
                text: "你好",
                media: [],
                metadata: {}
            }
        ],
        chatOptions: OpenAiChatOptions {
            model: "gpt-4",
            temperature: 0.7,
            maxTokens: null,
            // ... 其他选项
        }
    },
    context: {
        // Advisor参数（如conversationId等）
    }
}
```

---

## 三、阶段二：Advisor链执行

### 3.1 启动Advisor链

#### **步骤1：调用content()方法**

```java
// DefaultChatClient.java (L486-490)
@Override
public String content() {
    // 调用内部方法获取ChatResponse
    ChatResponse chatResponse = 
        doGetObservableChatClientResponse(this.request).chatResponse();
    
    // 从ChatResponse提取文本内容
    return getContentFromChatResponse(chatResponse);
}
```

#### **步骤2：执行Advisor链**

```java
// DefaultChatClient.java (L496-520)
private ChatClientResponse doGetObservableChatClientResponse(
        ChatClientRequest chatClientRequest,
        @Nullable String outputFormat) {
    
    // 1. 将outputFormat添加到context
    if (outputFormat != null) {
        chatClientRequest.context().put(
            ChatClientAttributes.OUTPUT_FORMAT.getKey(), 
            outputFormat
        );
    }
    
    // 2. 创建观测上下文（用于监控）
    ChatClientObservationContext observationContext = 
        ChatClientObservationContext.builder()
            .request(chatClientRequest)
            .advisors(this.advisorChain.getCallAdvisors())
            .stream(false)
            .format(outputFormat)
            .build();
    
    // 3. 创建Observation（Micrometer）
    var observation = ChatClientObservationDocumentation.AI_CHAT_CLIENT
        .observation(
            this.observationConvention,
            DEFAULT_CHAT_CLIENT_OBSERVATION_CONVENTION,
            () -> observationContext,
            this.observationRegistry
        );
    
    // 4. 在Observation中执行Advisor链
    var chatClientResponse = observation.observe(() -> {
        // ⭐ 核心：启动Advisor链
        return this.advisorChain.nextCall(chatClientRequest);
    });
    
    // 5. 返回响应（如果为null，返回空响应）
    return chatClientResponse != null 
        ? chatClientResponse 
        : ChatClientResponse.builder().build();
}
```

### 3.2 Advisor链遍历

#### **Advisor链的执行机制**

```java
// DefaultAroundAdvisorChain.java (L85-103)
@Override
public ChatClientResponse nextCall(ChatClientRequest chatClientRequest) {
    Assert.notNull(chatClientRequest, "the chatClientRequest cannot be null");
    
    // 1. 检查是否还有Advisor
    if (this.callAdvisors.isEmpty()) {
        throw new IllegalStateException("No CallAdvisors available to execute");
    }
    
    // 2. 弹出下一个Advisor（FIFO栈）
    var advisor = this.callAdvisors.pop();
    
    // 3. 创建观测上下文（监控每个Advisor）
    var observationContext = AdvisorObservationContext.builder()
        .advisorName(advisor.getName())
        .chatClientRequest(chatClientRequest)
        .order(advisor.getOrder())
        .build();
    
    // 4. 在Observation中执行Advisor
    return AdvisorObservationDocumentation.AI_ADVISOR
        .observation(
            null, 
            DEFAULT_OBSERVATION_CONVENTION, 
            () -> observationContext, 
            this.observationRegistry
        )
        .observe(() -> advisor.adviseCall(chatClientRequest, this));
        //                                                      ^^^^
        //                        将this（链本身）传递给Advisor
}
```

**关键机制**：
- Advisor通过调用`chain.nextCall(request)`来触发**下一个**Advisor
- 这形成了**递归调用链**
- 最后一个Advisor（ChatModelCallAdvisor）不再调用nextCall，而是直接调用ChatModel

### 3.3 示例：Memory Advisor执行

```java
// MessageChatMemoryAdvisor.java (L78-114)
@Override
public ChatClientRequest before(
        ChatClientRequest chatClientRequest, 
        AdvisorChain advisorChain) {
    
    // 1. 获取会话ID
    String conversationId = getConversationId(
        chatClientRequest.context(), 
        this.defaultConversationId
    );
    
    // 2. 从ChatMemory获取历史消息
    List<Message> memoryMessages = this.chatMemory.get(conversationId);
    
    // 3. 构建完整消息列表
    List<Message> processedMessages = new ArrayList<>(memoryMessages);
    processedMessages.addAll(chatClientRequest.prompt().getInstructions());
    
    // 4. 创建新的Request（包含历史）
    ChatClientRequest processedChatClientRequest = 
        chatClientRequest.mutate()
            .prompt(chatClientRequest.prompt()
                .mutate()
                .messages(processedMessages)
                .build())
            .build();
    
    // 5. 将当前用户消息添加到记忆
    UserMessage userMessage = 
        processedChatClientRequest.prompt().getUserMessage();
    this.chatMemory.add(conversationId, userMessage);
    
    return processedChatClientRequest;
}

@Override
public ChatClientResponse after(
        ChatClientResponse chatClientResponse, 
        AdvisorChain advisorChain) {
    
    // 提取AI响应并保存到记忆
    List<Message> assistantMessages = /* 从response提取 */;
    
    String conversationId = getConversationId(
        chatClientResponse.context(), 
        this.defaultConversationId
    );
    
    this.chatMemory.add(conversationId, assistantMessages);
    
    return chatClientResponse;
}
```

**BaseAdvisor的默认实现**：

```java
// BaseAdvisor.java
@Override
public ChatClientResponse adviseCall(
        ChatClientRequest chatClientRequest, 
        CallAdvisorChain callAdvisorChain) {
    
    // 1. before: 前置处理
    ChatClientRequest processedRequest = 
        before(chatClientRequest, callAdvisorChain);
    
    // 2. ⭐ 调用链的下一个Advisor
    ChatClientResponse chatClientResponse = 
        callAdvisorChain.nextCall(processedRequest);
    
    // 3. after: 后置处理
    return after(chatClientResponse, callAdvisorChain);
}
```

**执行顺序示例**：

```
1. MemoryAdvisor.adviseCall()
   │
   ├─> before()  [添加历史消息]
   │
   ├─> chain.nextCall() ──────┐
   │                           │
   │   2. RAGAdvisor.adviseCall()
   │      │
   │      ├─> before()  [检索知识库]
   │      │
   │      ├─> chain.nextCall() ──────┐
   │      │                           │
   │      │   3. ChatModelCallAdvisor.adviseCall()
   │      │      │
   │      │      └─> chatModel.call(prompt)  [实际调用AI]
   │      │                           │
   │      │<──────────────────────────┘
   │      │
   │      └─> after()  [后处理]
   │                           │
   │<──────────────────────────┘
   │
   └─> after()  [保存AI响应到Memory]
```

---

## 四、阶段三：ChatModel调用

### 4.1 ChatModelCallAdvisor终点

```java
// ChatModelCallAdvisor.java (L49-59)
@Override
public ChatClientResponse adviseCall(
        ChatClientRequest chatClientRequest, 
        CallAdvisorChain callAdvisorChain) {
    
    Assert.notNull(chatClientRequest, "the chatClientRequest cannot be null");
    
    // 1. 如果需要结构化输出，增强UserMessage
    ChatClientRequest formattedChatClientRequest = 
        augmentWithFormatInstructions(chatClientRequest);
    
    // 2. ⭐ 实际调用ChatModel
    ChatResponse chatResponse = 
        this.chatModel.call(formattedChatClientRequest.prompt());
    
    // 3. 构建ChatClientResponse
    return ChatClientResponse.builder()
        .chatResponse(chatResponse)
        .context(Map.copyOf(formattedChatClientRequest.context()))
        .build();
}
```

**关键点**：
- `ChatModelCallAdvisor`是Advisor链的**终点**
- 它的`adviseCall`方法**不再**调用`chain.nextCall()`
- 直接调用`chatModel.call(prompt)`

### 4.2 OpenAiChatModel.call()

```java
// OpenAiChatModel.java (L178-183)
@Override
public ChatResponse call(Prompt prompt) {
    // 1. 构建最终的Prompt（合并默认选项和运行时选项）
    Prompt requestPrompt = buildRequestPrompt(prompt);
    
    // 2. 调用内部方法
    return this.internalCall(requestPrompt, null);
}
```

#### **步骤1：合并Prompt选项**

```java
private Prompt buildRequestPrompt(Prompt prompt) {
    // 如果prompt没有options，使用defaultOptions
    if (prompt.getOptions() == null) {
        return Prompt.builder()
            .messages(prompt.getInstructions())
            .chatOptions(this.defaultOptions)
            .build();
    }
    
    // 合并prompt的options和defaultOptions
    ChatOptions mergedOptions = ModelOptionsUtils.merge(
        prompt.getOptions(),      // 运行时选项（优先级高）
        this.defaultOptions,      // 默认选项
        ChatOptions.class
    );
    
    return Prompt.builder()
        .messages(prompt.getInstructions())
        .chatOptions(mergedOptions)
        .build();
}
```

#### **步骤2：创建OpenAI请求**

```java
// OpenAiChatModel.java (L185-200)
public ChatResponse internalCall(Prompt prompt, ChatResponse previousChatResponse) {
    
    // 1. 创建ChatCompletionRequest
    ChatCompletionRequest request = createRequest(prompt, false);
    
    // 2. 创建观测上下文（监控）
    ChatModelObservationContext observationContext = 
        ChatModelObservationContext.builder()
            .prompt(prompt)
            .provider(OpenAiApiConstants.PROVIDER_NAME)
            .build();
    
    // 3. 在Observation中执行HTTP调用
    ChatResponse response = ChatModelObservationDocumentation.CHAT_MODEL_OPERATION
        .observation(
            this.observationConvention,
            DEFAULT_OBSERVATION_CONVENTION,
            () -> observationContext,
            this.observationRegistry
        )
        .observe(() -> {
            // ⭐ 核心：HTTP调用OpenAI API
            ResponseEntity<ChatCompletion> completionEntity = 
                this.retryTemplate.execute(ctx -> 
                    this.openAiApi.chatCompletionEntity(
                        request, 
                        getAdditionalHttpHeaders(prompt)
                    )
                );
            
            // 解析响应并构建ChatResponse
            return /* 构建ChatResponse */;
        });
    
    // 4. 如果有Tool调用，处理Tool执行并递归调用
    if (isToolCall(response, ...)) {
        return handleToolCalls(prompt, response);
    }
    
    return response;
}
```

#### **步骤3：构建ChatCompletionRequest**

```java
ChatCompletionRequest createRequest(Prompt prompt, boolean stream) {
    
    // 1. 获取ChatOptions
    OpenAiChatOptions options = (OpenAiChatOptions) prompt.getOptions();
    
    // 2. 转换Messages
    List<ChatCompletionMessage> chatCompletionMessages = 
        prompt.getInstructions().stream()
            .map(this::toOpenAiMessage)
            .toList();
    
    // 3. 处理Tool定义
    List<FunctionTool> tools = null;
    if (options.getToolCallbacks() != null) {
        tools = options.getToolCallbacks().stream()
            .map(toolCallback -> {
                var toolDefinition = toolCallback.getToolDefinition();
                return new FunctionTool(
                    toolDefinition.name(),
                    toolDefinition.description(),
                    toolDefinition.inputSchema()
                );
            })
            .toList();
    }
    
    // 4. 构建请求
    return ChatCompletionRequest.builder()
        .model(options.getModel())                  // 模型名称
        .messages(chatCompletionMessages)           // 消息列表
        .temperature(options.getTemperature())      // 温度
        .maxTokens(options.getMaxTokens())          // 最大Token
        .topP(options.getTopP())                    // Top-P
        .frequencyPenalty(options.getFrequencyPenalty())
        .presencePenalty(options.getPresencePenalty())
        .stop(options.getStop())                    // 停止序列
        .stream(stream)                              // 是否流式
        .tools(tools)                                // 工具列表
        .toolChoice(options.getToolChoice())        // 工具选择策略
        .user(options.getUser())
        .build();
}
```

**生成的JSON请求示例**：

```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "user",
      "content": "你好"
    }
  ],
  "temperature": 0.7,
  "max_tokens": null,
  "stream": false
}
```

---

## 五、阶段四：底层API交互

### 5.1 OpenAiApi HTTP调用

```java
// OpenAiApi.java
public ResponseEntity<ChatCompletion> chatCompletionEntity(
        ChatCompletionRequest chatRequest,
        Map<String, String> additionalHeaders) {
    
    Assert.notNull(chatRequest, "chatRequest cannot be null");
    Assert.isTrue(!chatRequest.stream(), "stream must be false");
    
    // 使用RestClient发送HTTP POST请求
    return this.restClient.post()
        .uri("/v1/chat/completions")
        .headers(headers -> {
            headers.setAll(this.defaultHeaders);
            if (additionalHeaders != null) {
                headers.setAll(additionalHeaders);
            }
        })
        .body(chatRequest)
        .retrieve()
        .toEntity(ChatCompletion.class);
}
```

### 5.2 HTTP请求详情

#### **请求头**

```
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer sk-...
Content-Type: application/json
```

#### **请求体**

```json
{
  "model": "gpt-4",
  "messages": [
    {
      "role": "user",
      "content": "你好"
    }
  ],
  "temperature": 0.7
}
```

### 5.3 重试机制

```java
// 使用RetryTemplate进行重试
ResponseEntity<ChatCompletion> completionEntity = 
    this.retryTemplate.execute(ctx -> 
        this.openAiApi.chatCompletionEntity(request, headers)
    );
```

**默认重试策略**：
- 最多重试10次
- 初始退避时间2秒
- 最大退避时间3秒
- 指数退避倍数1.5

### 5.4 OpenAI API响应

#### **响应JSON**

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1699012345,
  "model": "gpt-4-0613",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "你好！我是一个AI助手，有什么可以帮助你的吗？"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 20,
    "total_tokens": 30
  }
}
```

---

## 六、阶段五：响应构建

### 6.1 解析ChatCompletion

```java
// OpenAiChatModel.java
ChatResponse response = /* ... 在observe()中 ... */ {
    
    // 1. 执行HTTP调用
    ResponseEntity<ChatCompletion> completionEntity = 
        this.retryTemplate.execute(ctx -> 
            this.openAiApi.chatCompletionEntity(request, headers)
        );
    
    // 2. 提取响应体和响应头
    ChatCompletion chatCompletion = completionEntity.getBody();
    MultiValueMap<String, String> headers = completionEntity.getHeaders();
    
    // 3. 解析ChatCompletion构建ChatResponse
    ChatResponse chatResponse = toChatResponse(chatCompletion);
    
    // 4. 添加RateLimit信息（从响应头）
    RateLimit rateLimit = OpenAiResponseHeaderExtractor
        .extractAiResponseHeaders(headers);
    chatResponse = chatResponse.mutate()
        .metadata(ChatResponseMetadata.builder()
            .rateLimit(rateLimit)
            .build())
        .build();
    
    // 5. 更新观测上下文
    observationContext.setResponse(chatResponse);
    
    return chatResponse;
};
```

### 6.2 构建ChatResponse

```java
private ChatResponse toChatResponse(ChatCompletion chatCompletion) {
    
    // 1. 解析所有Choice
    List<Generation> generations = chatCompletion.choices().stream()
        .map(choice -> {
            // 转换Message
            var message = choice.message();
            var assistantMessage = new AssistantMessage(
                message.content(),
                Map.of(),                      // metadata
                message.toolCalls()            // tool calls
            );
            
            // 构建GenerationMetadata
            var generationMetadata = ChatGenerationMetadata.builder()
                .finishReason(choice.finishReason())
                .build();
            
            // 构建Generation
            return new Generation(assistantMessage, generationMetadata);
        })
        .toList();
    
    // 2. 构建Usage
    var usage = chatCompletion.usage();
    Usage responseUsage = new DefaultUsage(
        usage.promptTokens(),
        usage.completionTokens(),
        usage.totalTokens()
    );
    
    // 3. 构建ChatResponseMetadata
    ChatResponseMetadata chatResponseMetadata = ChatResponseMetadata.builder()
        .id(chatCompletion.id())
        .model(chatCompletion.model())
        .usage(responseUsage)
        .build();
    
    // 4. 构建ChatResponse
    return ChatResponse.builder()
        .generations(generations)
        .metadata(chatResponseMetadata)
        .build();
}
```

### 6.3 ChatResponse数据结构

```java
ChatResponse {
    results: [
        Generation {
            output: AssistantMessage {
                text: "你好！我是一个AI助手，有什么可以帮助你的吗？",
                messageType: ASSISTANT,
                metadata: {},
                toolCalls: []
            },
            metadata: ChatGenerationMetadata {
                finishReason: "stop"
            }
        }
    ],
    metadata: ChatResponseMetadata {
        id: "chatcmpl-abc123",
        model: "gpt-4-0613",
        usage: DefaultUsage {
            promptTokens: 10,
            completionTokens: 20,
            totalTokens: 30
        },
        rateLimit: RateLimit {
            requestsLimit: 10000,
            requestsRemaining: 9999,
            tokensLimit: 150000,
            tokensRemaining: 149970
        }
    }
}
```

---

## 七、阶段六：Advisor后处理

### 7.1 Advisor链回溯

当`ChatModelCallAdvisor`返回`ChatResponse`后，执行栈开始**回溯**，依次执行各个Advisor的`after()`方法。

```
执行栈回溯：

1. ChatModelCallAdvisor.adviseCall()
   └─> return ChatClientResponse  ────┐
                                       │
2. RAGAdvisor.adviseCall()             │
   ├─> before()                        │
   ├─> chain.nextCall() ───────────────┘
   └─> after() ← 现在执行              │
       return response ────────────────┐
                                       │
3. MemoryAdvisor.adviseCall()          │
   ├─> before()                        │
   ├─> chain.nextCall() ───────────────┘
   └─> after() ← 现在执行
       - 保存AI响应到ChatMemory
       return response
```

### 7.2 Memory Advisor后处理

```java
// MessageChatMemoryAdvisor.java (L101-114)
@Override
public ChatClientResponse after(
        ChatClientResponse chatClientResponse, 
        AdvisorChain advisorChain) {
    
    // 1. 提取AI的响应消息
    List<Message> assistantMessages = new ArrayList<>();
    if (chatClientResponse.chatResponse() != null) {
        assistantMessages = chatClientResponse.chatResponse()
            .getResults()
            .stream()
            .map(g -> (Message) g.getOutput())  // AssistantMessage
            .toList();
    }
    
    // 2. 保存到ChatMemory
    String conversationId = getConversationId(
        chatClientResponse.context(), 
        this.defaultConversationId
    );
    this.chatMemory.add(conversationId, assistantMessages);
    
    // 3. 返回响应（不修改）
    return chatClientResponse;
}
```

**Memory中的状态**：

```java
// Before call:
memory.get("user-123") = [
    UserMessage("你好")
]

// After call:
memory.get("user-123") = [
    UserMessage("你好"),
    AssistantMessage("你好！我是一个AI助手，有什么可以帮助你的吗？")
]
```

### 7.3 其他Advisor后处理

```java
// SimpleLoggerAdvisor.java
@Override
public ChatClientResponse after(
        ChatClientResponse chatClientResponse, 
        AdvisorChain advisorChain) {
    
    // 记录响应日志
    logger.info("Response received: {}", 
        chatClientResponse.chatResponse().getResult().getOutput().getText()
    );
    
    return chatClientResponse;
}
```

---

## 八、阶段七：响应返回

### 8.1 提取Content

```java
// DefaultChatClient.java (L486-490)
@Override
public String content() {
    // 1. 获取ChatClientResponse
    ChatResponse chatResponse = 
        doGetObservableChatClientResponse(this.request).chatResponse();
    
    // 2. 从ChatResponse提取文本
    return getContentFromChatResponse(chatResponse);
}
```

### 8.2 提取文本内容

```java
// DefaultChatClient.java (L522-529)
@Nullable
private static String getContentFromChatResponse(
        @Nullable ChatResponse chatResponse) {
    
    return Optional.ofNullable(chatResponse)
        .map(ChatResponse::getResult)           // 获取第一个Generation
        .map(Generation::getOutput)              // 获取AssistantMessage
        .map(AbstractMessage::getText)           // 获取文本内容
        .orElse(null);                           // 如果为null返回null
}
```

### 8.3 其他响应格式

#### **返回完整ChatResponse**

```java
ChatResponse chatResponse = chatClient.prompt()
    .user("你好")
    .call()
    .chatResponse();

// ChatResponse {
//     results: [...],
//     metadata: {
//         usage: { promptTokens: 10, completionTokens: 20 }
//     }
// }
```

#### **返回结构化对象**

```java
record Person(String name, int age) {}

Person person = chatClient.prompt()
    .user("返回一个人的信息")
    .call()
    .entity(Person.class);

// Person { name="张三", age=25 }
```

#### **返回ChatClientResponse**

```java
ChatClientResponse response = chatClient.prompt()
    .user("你好")
    .call()
    .chatClientResponse();

// ChatClientResponse {
//     chatResponse: ChatResponse {...},
//     context: { conversationId: "user-123", ... }
// }
```

---

## 九、完整示例分析

### 9.1 示例代码

```java
@Service
public class ChatService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String chat(String userId, String message) {
        return chatClient.prompt()
            .user(message)
            .context(Map.of(ChatMemory.CONVERSATION_ID, userId))
            .call()
            .content();
    }
}

// 配置
@Configuration
public class ChatConfig {
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ChatMemory chatMemory) {
        
        return ChatClient.builder(chatModel)
            .defaultSystem("你是一个友好的助手")
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                SimpleLoggerAdvisor.builder().build()
            )
            .build();
    }
    
    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .maxMessages(10)
            .build();
    }
}
```

### 9.2 执行流程追踪

#### **第1次调用**

```java
chatService.chat("user-123", "我的名字是张三");
```

**流程**：

```
1. 请求构建
   ChatClientRequest {
       prompt: {
           messages: [
               SystemMessage("你是一个友好的助手"),
               UserMessage("我的名字是张三")
           ]
       },
       context: { conversationId: "user-123" }
   }

2. Advisor链执行
   
   2.1 MessageChatMemoryAdvisor.before()
       - memory.get("user-123") = []  (空的)
       - 添加UserMessage到memory
       - memory.add("user-123", UserMessage("我的名字是张三"))
   
   2.2 SimpleLoggerAdvisor.before()
       - logger.info("Request: 我的名字是张三")
   
   2.3 ChatModelCallAdvisor.adviseCall()
       - chatModel.call(prompt)

3. ChatModel调用
   
   3.1 OpenAiChatModel.call()
       - 构建ChatCompletionRequest
       - HTTP POST to OpenAI
   
   3.2 OpenAI响应
       {
           "choices": [{
               "message": {
                   "role": "assistant",
                   "content": "你好，张三！很高兴认识你。"
               }
           }]
       }
   
   3.3 构建ChatResponse
       ChatResponse {
           results: [
               Generation {
                   output: AssistantMessage("你好，张三！很高兴认识你。")
               }
           ]
       }

4. Advisor链回溯（after方法）
   
   4.1 SimpleLoggerAdvisor.after()
       - logger.info("Response: 你好，张三！很高兴认识你。")
   
   4.2 MessageChatMemoryAdvisor.after()
       - memory.add("user-123", AssistantMessage("你好，张三！很高兴认识你。"))
       - memory.get("user-123") = [
             UserMessage("我的名字是张三"),
             AssistantMessage("你好，张三！很高兴认识你。")
         ]

5. 返回给用户
   return "你好，张三！很高兴认识你。"
```

#### **第2次调用**

```java
chatService.chat("user-123", "你还记得我的名字吗？");
```

**流程**：

```
1. 请求构建
   ChatClientRequest {
       prompt: {
           messages: [
               SystemMessage("你是一个友好的助手"),
               UserMessage("你还记得我的名字吗？")
           ]
       },
       context: { conversationId: "user-123" }
   }

2. Advisor链执行
   
   2.1 MessageChatMemoryAdvisor.before()
       - memory.get("user-123") = [
             UserMessage("我的名字是张三"),
             AssistantMessage("你好，张三！很高兴认识你。")
         ]
       
       - 添加历史消息到prompt
       - processedMessages = [
             SystemMessage("你是一个友好的助手"),
             UserMessage("我的名字是张三"),           // 历史
             AssistantMessage("你好，张三！..."),     // 历史
             UserMessage("你还记得我的名字吗？")      // 当前
         ]
       
       - 添加当前消息到memory
       - memory.add("user-123", UserMessage("你还记得我的名字吗？"))

3. ChatModel调用
   
   3.1 发送给OpenAI的消息
       [
           { "role": "system", "content": "你是一个友好的助手" },
           { "role": "user", "content": "我的名字是张三" },
           { "role": "assistant", "content": "你好，张三！很高兴认识你。" },
           { "role": "user", "content": "你还记得我的名字吗？" }
       ]
   
   3.2 OpenAI响应
       "当然记得，你的名字是张三。"

4. Advisor链回溯
   
   4.1 MessageChatMemoryAdvisor.after()
       - memory.add("user-123", AssistantMessage("当然记得，你的名字是张三。"))
       - memory.get("user-123") = [
             UserMessage("我的名字是张三"),
             AssistantMessage("你好，张三！很高兴认识你。"),
             UserMessage("你还记得我的名字吗？"),
             AssistantMessage("当然记得，你的名字是张三。")
         ]

5. 返回给用户
   return "当然记得，你的名字是张三。"
```

---

## 十、流式调用流程

### 10.1 流式调用入口

```java
Flux<String> contentStream = chatClient.prompt()
    .user("写一篇关于Spring的文章")
    .stream()      // 返回 StreamResponseSpec
    .content();    // 返回 Flux<String>

// 订阅流
contentStream.subscribe(chunk -> {
    System.out.print(chunk);  // 逐块打印
});
```

### 10.2 流式Advisor链

```java
// DefaultAroundAdvisorChain.java (L106-134)
@Override
public Flux<ChatClientResponse> nextStream(
        ChatClientRequest chatClientRequest) {
    
    return Flux.deferContextual(contextView -> {
        
        // 1. 检查是否还有StreamAdvisor
        if (this.streamAdvisors.isEmpty()) {
            return Flux.error(
                new IllegalStateException("No StreamAdvisors available")
            );
        }
        
        // 2. 弹出下一个Advisor
        var advisor = this.streamAdvisors.pop();
        
        // 3. 创建观测上下文
        AdvisorObservationContext observationContext = 
            AdvisorObservationContext.builder()
                .advisorName(advisor.getName())
                .chatClientRequest(chatClientRequest)
                .order(advisor.getOrder())
                .build();
        
        // 4. 创建Observation
        var observation = AdvisorObservationDocumentation.AI_ADVISOR
            .observation(
                null,
                DEFAULT_OBSERVATION_CONVENTION,
                () -> observationContext,
                this.observationRegistry
            );
        
        observation.parentObservation(
            contextView.getOrDefault(ObservationThreadLocalAccessor.KEY, null)
        ).start();
        
        // 5. 执行Advisor并返回Flux
        return Flux.defer(() -> 
            advisor.adviseStream(chatClientRequest, this)
                .doOnError(observation::error)
                .doFinally(s -> observation.stop())
                .contextWrite(ctx -> 
                    ctx.put(ObservationThreadLocalAccessor.KEY, observation)
                )
        );
    });
}
```

### 10.3 ChatModel流式调用

```java
// OpenAiChatModel.java
@Override
public Flux<ChatResponse> stream(Prompt prompt) {
    
    // 1. 构建请求（stream=true）
    Prompt requestPrompt = buildRequestPrompt(prompt);
    ChatCompletionRequest request = createRequest(requestPrompt, true);
    
    // 2. 调用流式API
    Flux<ChatCompletion> chatCompletionFlux = 
        this.retryTemplate.execute(ctx ->
            this.openAiApi.chatCompletionStream(request)
        );
    
    // 3. 转换为ChatResponse流
    return chatCompletionFlux.map(chatCompletion -> 
        toChatResponse(chatCompletion)
    );
}
```

### 10.4 SSE响应处理

```
OpenAI SSE响应：

data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"Spring"}}]}

data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"是"}}]}

data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"一个"}}]}

data: {"id":"chatcmpl-abc","choices":[{"delta":{"content":"强大"}}]}

data: [DONE]


转换为Flux：

"Spring" → "是" → "一个" → "强大" → ...
```

### 10.5 流式Memory处理

```java
// BaseAdvisor.java
@Override
public Flux<ChatClientResponse> adviseStream(
        ChatClientRequest chatClientRequest,
        StreamAdvisorChain streamAdvisorChain) {
    
    // 1. before: 前置处理（同步）
    return Mono.just(chatClientRequest)
        .publishOn(this.getScheduler())
        .map(request -> this.before(request, streamAdvisorChain))
        
        // 2. 调用下一个Advisor（返回Flux）
        .flatMapMany(streamAdvisorChain::nextStream)
        
        // 3. 聚合流式响应
        .transform(flux -> 
            new ChatClientMessageAggregator()
                .aggregateChatClientResponse(
                    flux,
                    // 4. after: 后置处理（在聚合完成后）
                    response -> this.after(response, streamAdvisorChain)
                )
        );
}
```

**关键机制**：
- `before()`在流开始前执行（添加历史）
- 流式响应逐块发送给客户端
- `after()`在流结束后执行（保存AI响应）

---

## 总结

### 核心要点

1. **7个关键阶段**：
   - 请求构建 → Advisor链 → ChatModel → API交互 → 响应构建 → 后处理 → 返回

2. **Advisor链机制**：
   - 递归调用链
   - before → nextCall → after
   - ChatModelCallAdvisor是终点

3. **关键组件**：
   - `DefaultChatClient`：请求构建和调度
   - `DefaultAroundAdvisorChain`：Advisor链管理
   - `ChatModelCallAdvisor`：实际调用ChatModel
   - `OpenAiChatModel`：HTTP调用和响应解析
   - `OpenAiApi`：底层HTTP客户端

4. **数据转换流程**：
   ```
   用户输入 → ChatClientRequest → Prompt → ChatCompletionRequest
   → HTTP JSON → ChatCompletion → ChatResponse → String content
   ```

5. **可扩展点**：
   - 自定义Advisor（扩展处理逻辑）
   - 自定义ChatModel（支持其他AI提供商）
   - 自定义TemplateRenderer（扩展模板引擎）
   - 自定义ObservationConvention（自定义监控）

### 性能考虑

1. **Advisor顺序**：Order值影响执行顺序
2. **Memory大小**：历史消息数影响Token消耗
3. **重试策略**：影响失败恢复时间
4. **流式响应**：减少首字延迟

### 调试建议

1. **开启日志**：
   ```yaml
   logging:
     level:
       org.springframework.ai: DEBUG
   ```

2. **使用SimpleLoggerAdvisor**：记录所有请求和响应

3. **使用Observation**：集成Micrometer监控

4. **断点调试位置**：
   - `DefaultChatClientRequestSpec.call()`
   - `DefaultAroundAdvisorChain.nextCall()`
   - `ChatModelCallAdvisor.adviseCall()`
   - `OpenAiChatModel.call()`

---

恭喜你！现在你已经完全理解了Spring AI中一次大模型调用的完整流程！🎉

