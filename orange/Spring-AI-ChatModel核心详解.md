# Spring AI ChatModel 核心详解

## 📋 目录
- [概述](#概述)
- [接口体系](#接口体系)
- [核心概念](#核心概念)
- [ChatOptions配置](#chatoptions配置)
- [实现ChatModel](#实现chatmodel)
- [OpenAI实现剖析](#openai实现剖析)
- [最佳实践](#最佳实践)
- [实战示例](#实战示例)

---

## 概述

`ChatModel` 是Spring AI中最核心的接口之一，它定义了与AI聊天模型交互的统一API。通过这个抽象层，开发者可以轻松切换不同的AI提供商（OpenAI、Anthropic、Ollama等），而无需修改业务代码。

### 核心价值

1. **统一抽象**: 为所有AI提供商提供一致的API
2. **可移植性**: 轻松切换不同的AI模型
3. **同步和异步**: 支持阻塞式调用和流式响应
4. **类型安全**: 基于Java泛型的强类型API
5. **扩展性**: 易于添加新的模型提供商

### 设计哲学

```
应用层
    ↓
ChatModel (统一接口)
    ↓
┌──────┴──────┬──────┬──────┬──────┐
│             │      │      │      │
OpenAI    Anthropic Ollama Azure  自定义
```

---

## 接口体系

### 1. Model<TReq, TRes> (根接口)

所有AI模型的根接口，使用泛型定义请求和响应类型。

```java
public interface Model<TReq extends ModelRequest<?>, 
                       TRes extends ModelResponse<?>> {
    
    /**
     * 执行AI模型调用
     * @param request 请求对象
     * @return 响应对象
     */
    TRes call(TReq request);
}
```

#### 设计特点

- **泛型设计**: 支持不同类型的请求和响应
- **单一方法**: 保持接口简洁
- **同步调用**: 阻塞直到获得完整响应

### 2. StreamingModel<TReq, TRes> (流式接口)

支持流式响应的模型接口。

```java
public interface StreamingModel<TReq extends ModelRequest<?>, 
                                TRes extends ModelResponse<?>> {
    
    /**
     * 执行流式调用
     * @param request 请求对象
     * @return 响应流
     */
    Flux<TRes> stream(TReq request);
}
```

#### 流式特点

- **Reactor支持**: 使用Flux处理异步流
- **实时响应**: 逐步返回生成的内容
- **背压控制**: 支持流量控制
- **更好的用户体验**: 适合长文本生成

### 3. StreamingChatModel (聊天流式接口)

专门用于聊天的流式模型接口。

```java
@FunctionalInterface
public interface StreamingChatModel 
    extends StreamingModel<Prompt, ChatResponse> {
    
    /**
     * 流式调用（便捷方法）
     */
    default Flux<String> stream(String message) {
        Prompt prompt = new Prompt(message);
        return stream(prompt)
            .map(response -> Optional.ofNullable(response.getResult())
                .map(Generation::getOutput)
                .map(AssistantMessage::getText)
                .orElse(""));
    }
    
    /**
     * 流式调用（多消息）
     */
    default Flux<String> stream(Message... messages) {
        Prompt prompt = new Prompt(Arrays.asList(messages));
        return stream(prompt)
            .map(response -> Optional.ofNullable(response.getResult())
                .map(Generation::getOutput)
                .map(AssistantMessage::getText)
                .orElse(""));
    }
    
    /**
     * 流式调用（核心方法）
     */
    @Override
    Flux<ChatResponse> stream(Prompt prompt);
}
```

### 4. ChatModel (聊天模型接口)

整合同步和流式调用的完整聊天模型接口。

```java
public interface ChatModel 
    extends Model<Prompt, ChatResponse>, StreamingChatModel {
    
    /**
     * 同步调用（便捷方法 - 字符串输入）
     * @param message 用户消息
     * @return AI响应文本
     */
    default String call(String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        Generation generation = call(prompt).getResult();
        return (generation != null) ? generation.getOutput().getText() : "";
    }
    
    /**
     * 同步调用（便捷方法 - 多消息）
     * @param messages 消息列表
     * @return AI响应文本
     */
    default String call(Message... messages) {
        Prompt prompt = new Prompt(Arrays.asList(messages));
        Generation generation = call(prompt).getResult();
        return (generation != null) ? generation.getOutput().getText() : "";
    }
    
    /**
     * 同步调用（核心方法）
     * @param prompt 提示词对象
     * @return 完整响应
     */
    @Override
    ChatResponse call(Prompt prompt);
    
    /**
     * 获取默认配置
     */
    default ChatOptions getDefaultOptions() {
        return ChatOptions.builder().build();
    }
    
    /**
     * 流式调用（默认抛出异常，子类可覆盖）
     */
    default Flux<ChatResponse> stream(Prompt prompt) {
        throw new UnsupportedOperationException(
            "streaming is not supported"
        );
    }
}
```

#### 接口层次结构

```
Model<Prompt, ChatResponse>
    ↑
    │ (同步调用)
    │
ChatModel ←────── StreamingChatModel
    │                    ↑
    │                    │ (流式调用)
    │                    │
    │             StreamingModel<Prompt, ChatResponse>
    │
    └─ 便捷方法: call(String), call(Message...)
    └─ 核心方法: call(Prompt)
    └─ 流式方法: stream(Prompt)
    └─ 配置方法: getDefaultOptions()
```

---

## 核心概念

### 1. Prompt (提示词对象)

`Prompt` 是发送给AI模型的请求封装。

```java
public class Prompt implements ModelRequest<List<Message>> {
    
    private final List<Message> messages;
    private ChatOptions chatOptions;
    
    // 构造方法
    public Prompt(String contents);
    public Prompt(Message message);
    public Prompt(List<Message> messages);
    public Prompt(List<Message> messages, ChatOptions chatOptions);
    
    // 核心方法
    public List<Message> getInstructions();
    public ChatOptions getOptions();
    public String getContents();
    public SystemMessage getSystemMessage();
    public UserMessage getUserMessage();
}
```

#### Message类型

```java
// Message类型体系
Message (抽象类)
├── SystemMessage      - 系统提示词，定义AI行为
├── UserMessage        - 用户输入消息
├── AssistantMessage   - AI助手的回复
└── ToolResponseMessage - 工具调用的结果
```

#### 创建Prompt示例

```java
// 1. 简单字符串
Prompt prompt = new Prompt("你好，世界");

// 2. 单个消息
Prompt prompt = new Prompt(new UserMessage("你好"));

// 3. 多个消息（完整对话）
Prompt prompt = new Prompt(
    new SystemMessage("你是一个友好的助手"),
    new UserMessage("你好")
);

// 4. 带配置选项
Prompt prompt = new Prompt(
    List.of(new UserMessage("你好")),
    ChatOptions.builder()
        .temperature(0.8)
        .maxTokens(100)
        .build()
);
```

### 2. ChatResponse (响应对象)

`ChatResponse` 封装了AI模型的响应。

```java
public class ChatResponse implements ModelResponse<Generation> {
    
    private final List<Generation> generations;
    private final ChatResponseMetadata metadata;
    
    // 获取所有生成结果
    public List<Generation> getResults();
    
    // 获取第一个结果（最常用）
    public Generation getResult();
    
    // 获取元数据
    public ChatResponseMetadata getMetadata();
    
    // 检查是否有工具调用
    public boolean hasToolCalls();
}
```

#### Generation (生成结果)

```java
public class Generation {
    
    private final AssistantMessage assistantMessage;
    private final ChatGenerationMetadata metadata;
    
    // 获取AI的输出消息
    public AssistantMessage getOutput();
    
    // 获取生成元数据
    public ChatGenerationMetadata getMetadata();
}
```

#### ChatResponseMetadata (响应元数据)

```java
public class ChatResponseMetadata {
    
    // Token使用情况
    private Usage usage;
    
    // 速率限制信息
    private RateLimit rateLimit;
    
    // 模型名称
    private String model;
    
    // 提示词过滤结果（内容审核）
    private List<PromptFilterMetadata> promptFilterMetadata;
    
    // 完成原因（finish_reason）
    private String finishReason;
}
```

#### 使用响应示例

```java
ChatResponse response = chatModel.call(prompt);

// 获取文本内容
String text = response.getResult()
    .getOutput()
    .getText();

// 获取Token使用量
Usage usage = response.getMetadata().getUsage();
int promptTokens = usage.getPromptTokens();
int completionTokens = usage.getCompletionTokens();
int totalTokens = usage.getTotalTokens();

// 获取模型名称
String model = response.getMetadata().getModel();

// 获取完成原因
String finishReason = response.getMetadata().getFinishReason();
// 可能的值: "stop", "length", "tool_calls", "content_filter"
```

### 3. Message体系

#### SystemMessage (系统消息)

定义AI的行为、角色和规则。

```java
SystemMessage systemMessage = new SystemMessage(
    "你是一个专业的Java开发助手"
);

// 使用Builder
SystemMessage systemMessage = SystemMessage.builder()
    .text("你是一个专业的Java开发助手")
    .metadata(Map.of("priority", "high"))
    .build();
```

#### UserMessage (用户消息)

用户的输入或查询。

```java
// 纯文本消息
UserMessage userMessage = new UserMessage("什么是Spring Boot?");

// 多模态消息（文本+图片）
UserMessage userMessage = UserMessage.builder()
    .text("这张图片里有什么？")
    .media(new Media(
        MimeTypeUtils.IMAGE_PNG,
        new ClassPathResource("images/photo.png")
    ))
    .build();

// 带元数据
UserMessage userMessage = UserMessage.builder()
    .text("查询内容")
    .metadata(Map.of(
        "userId", "12345",
        "sessionId", "abc-123"
    ))
    .build();
```

#### AssistantMessage (助手消息)

AI的回复，也可能包含工具调用。

```java
AssistantMessage assistantMessage = new AssistantMessage(
    "Spring Boot是一个快速开发框架..."
);

// 包含工具调用
AssistantMessage assistantMessage = AssistantMessage.builder()
    .content("让我帮你查询天气")
    .toolCalls(List.of(
        new ToolCall("get_weather", Map.of("city", "北京"))
    ))
    .build();
```

#### ToolResponseMessage (工具响应消息)

工具执行的结果。

```java
ToolResponseMessage toolResponse = new ToolResponseMessage(
    List.of(
        new ToolResponse("call_123", "工具名称", "执行结果")
    )
);
```

---

## ChatOptions配置

`ChatOptions` 定义了AI模型调用的参数配置。

### ChatOptions接口

```java
public interface ChatOptions extends ModelOptions {
    
    // 模型名称
    String getModel();
    
    // 频率惩罚 (-2.0 到 2.0)
    Double getFrequencyPenalty();
    
    // 最大Token数
    Integer getMaxTokens();
    
    // 存在惩罚 (-2.0 到 2.0)
    Double getPresencePenalty();
    
    // 停止序列
    List<String> getStopSequences();
    
    // 温度 (0.0 到 2.0)
    Double getTemperature();
    
    // Top-K采样
    Integer getTopK();
    
    // Top-P采样 (0.0 到 1.0)
    Double getTopP();
    
    // 复制配置
    <T extends ChatOptions> T copy();
}
```

### 参数详解

#### 1. Temperature (温度)

控制输出的随机性。

```java
// temperature: 0.0 - 2.0
// - 0.0: 确定性输出，每次结果相同
// - 0.5: 平衡创造性和一致性
// - 1.0: 默认值，平衡
// - 2.0: 最大随机性，创造性输出

ChatOptions options = ChatOptions.builder()
    .temperature(0.7)  // 适度创造性
    .build();
```

**使用场景**:
- **低温度(0.0-0.3)**: 代码生成、数学计算、事实查询
- **中温度(0.5-0.8)**: 一般对话、内容创作
- **高温度(0.9-2.0)**: 创意写作、头脑风暴

#### 2. MaxTokens (最大Token数)

限制生成的最大Token数量。

```java
ChatOptions options = ChatOptions.builder()
    .maxTokens(500)  // 限制输出长度
    .build();
```

**注意事项**:
- 包括输入和输出的总Token数不能超过模型限制
- GPT-3.5-turbo: 4,096 tokens
- GPT-4: 8,192 tokens (基础版)
- GPT-4-32k: 32,768 tokens

#### 3. FrequencyPenalty (频率惩罚)

降低模型重复相同内容的倾向。

```java
// frequencyPenalty: -2.0 到 2.0
// - 正值: 减少重复
// - 负值: 增加重复
// - 0: 不惩罚

ChatOptions options = ChatOptions.builder()
    .frequencyPenalty(0.5)  // 轻微减少重复
    .build();
```

#### 4. PresencePenalty (存在惩罚)

鼓励模型谈论新话题。

```java
// presencePenalty: -2.0 到 2.0
// - 正值: 鼓励新话题
// - 负值: 倾向于现有话题
// - 0: 不惩罚

ChatOptions options = ChatOptions.builder()
    .presencePenalty(0.6)  // 鼓励多样性
    .build();
```

#### 5. TopP (核采样)

控制输出的多样性，替代temperature。

```java
// topP: 0.0 到 1.0
// - 1.0: 考虑所有可能的词
// - 0.1: 只考虑概率最高的10%的词

ChatOptions options = ChatOptions.builder()
    .topP(0.9)  // 考虑前90%概率的词
    .build();
```

**建议**: 通常只调整temperature或topP其中之一。

#### 6. StopSequences (停止序列)

定义停止生成的标记。

```java
ChatOptions options = ChatOptions.builder()
    .stopSequences(List.of(
        "\n\n",      // 遇到两个换行停止
        "User:",     // 遇到"User:"停止
        "Assistant:" // 遇到"Assistant:"停止
    ))
    .build();
```

### DefaultChatOptions (默认实现)

```java
public class DefaultChatOptions implements ChatOptions {
    
    private String model;
    private Double frequencyPenalty;
    private Integer maxTokens;
    private Double presencePenalty;
    private List<String> stopSequences;
    private Double temperature;
    private Integer topK;
    private Double topP;
    
    // Builder模式创建
    public static Builder builder() {
        return new Builder();
    }
}
```

### 创建ChatOptions示例

```java
// 1. 使用Builder
ChatOptions options = ChatOptions.builder()
    .model("gpt-4")
    .temperature(0.7)
    .maxTokens(1000)
    .frequencyPenalty(0.5)
    .presencePenalty(0.6)
    .topP(0.9)
    .stopSequences(List.of("\n\n"))
    .build();

// 2. 特定模型的Options（如OpenAI）
OpenAiChatOptions openAiOptions = OpenAiChatOptions.builder()
    .model("gpt-4")
    .temperature(0.7)
    .maxTokens(1000)
    // OpenAI特有选项
    .seed(12345)  // 确定性输出
    .user("user-123")  // 用户标识
    .n(1)  // 生成数量
    .responseFormat(ResponseFormat.JSON)  // JSON输出
    .build();

// 3. 在Prompt中使用
Prompt prompt = new Prompt(
    List.of(new UserMessage("你好")),
    options
);
```

### 配置优先级

```java
// 优先级：请求级 > 客户端级 > 模型默认
// 
// 1. 模型默认配置
ChatModel chatModel = new OpenAiChatModel(...);
chatModel.getDefaultOptions(); // 模型级默认

// 2. ChatClient级配置
ChatClient client = ChatClient.builder(chatModel)
    .defaultOptions(ChatOptions.builder()
        .temperature(0.8)
        .build())
    .build();

// 3. 请求级配置（优先级最高）
String response = client
    .prompt("你好")
    .options(ChatOptions.builder()
        .temperature(0.5)  // 覆盖默认的0.8
        .build())
    .call()
    .content();
```

---

## 实现ChatModel

### 基本实现结构

```java
public class MyChatModel implements ChatModel {
    
    private final MyApiClient apiClient;
    private final ChatOptions defaultOptions;
    private final RetryTemplate retryTemplate;
    private final ObservationRegistry observationRegistry;
    
    @Override
    public ChatResponse call(Prompt prompt) {
        // 1. 合并配置
        ChatOptions options = mergeOptions(
            prompt.getOptions(), 
            defaultOptions
        );
        
        // 2. 构建API请求
        ApiRequest request = buildRequest(prompt, options);
        
        // 3. 调用API（带重试）
        ApiResponse apiResponse = retryTemplate.execute(ctx -> 
            apiClient.chat(request)
        );
        
        // 4. 转换响应
        return toChatResponse(apiResponse);
    }
    
    @Override
    public Flux<ChatResponse> stream(Prompt prompt) {
        // 1. 合并配置
        ChatOptions options = mergeOptions(
            prompt.getOptions(), 
            defaultOptions
        );
        
        // 2. 构建API请求
        ApiRequest request = buildRequest(prompt, options);
        request.setStream(true);
        
        // 3. 调用流式API
        return apiClient.chatStream(request)
            .map(this::toChatResponse);
    }
    
    @Override
    public ChatOptions getDefaultOptions() {
        return defaultOptions;
    }
    
    // 辅助方法
    private ChatOptions mergeOptions(
            ChatOptions requestOptions,
            ChatOptions defaultOptions) {
        // 请求级配置优先
        if (requestOptions != null) {
            return mergeWithDefaults(requestOptions, defaultOptions);
        }
        return defaultOptions;
    }
    
    private ApiRequest buildRequest(
            Prompt prompt, 
            ChatOptions options) {
        // 转换Prompt和Options为API请求格式
        return ApiRequest.builder()
            .messages(toApiMessages(prompt.getInstructions()))
            .temperature(options.getTemperature())
            .maxTokens(options.getMaxTokens())
            .build();
    }
    
    private ChatResponse toChatResponse(ApiResponse apiResponse) {
        // 转换API响应为ChatResponse
        Generation generation = new Generation(
            new AssistantMessage(apiResponse.getText())
        );
        
        ChatResponseMetadata metadata = ChatResponseMetadata.builder()
            .usage(new DefaultUsage(
                apiResponse.getPromptTokens(),
                apiResponse.getCompletionTokens()
            ))
            .model(apiResponse.getModel())
            .finishReason(apiResponse.getFinishReason())
            .build();
        
        return new ChatResponse(
            List.of(generation),
            metadata
        );
    }
}
```

### 支持的功能清单

实现ChatModel时应考虑支持的功能：

```java
public class FullFeaturedChatModel implements ChatModel {
    
    // ✅ 必须实现
    @Override
    public ChatResponse call(Prompt prompt) { }
    
    // ✅ 强烈推荐（流式支持）
    @Override
    public Flux<ChatResponse> stream(Prompt prompt) { }
    
    // ✅ 推荐（默认配置）
    @Override
    public ChatOptions getDefaultOptions() { }
    
    // ✅ 可选功能
    // - Function Calling (工具调用)
    // - Multi-modal (多模态)
    // - JSON Mode (JSON输出)
    // - Vision (图像理解)
    // - Audio (语音输入/输出)
    // - Retry (重试机制)
    // - Observation (可观测性)
    // - Rate Limiting (速率限制)
}
```

---

## OpenAI实现剖析

让我们深入分析OpenAI的ChatModel实现。

### OpenAiChatModel结构

```java
public class OpenAiChatModel implements ChatModel {
    
    // 依赖
    private final OpenAiApi openAiApi;  // API客户端
    private final OpenAiChatOptions defaultOptions;  // 默认配置
    private final RetryTemplate retryTemplate;  // 重试模板
    private final ObservationRegistry observationRegistry;  // 监控
    private final ToolCallingManager toolCallingManager;  // 工具调用管理
    private final ToolExecutionEligibilityPredicate eligibilityPredicate;
    private ChatModelObservationConvention observationConvention;
    
    @Override
    public ChatResponse call(Prompt prompt) {
        // 1. 构建请求Prompt（合并配置）
        Prompt requestPrompt = buildRequestPrompt(prompt);
        
        // 2. 调用内部实现
        return internalCall(requestPrompt, null);
    }
    
    public ChatResponse internalCall(
            Prompt prompt, 
            ChatResponse previousResponse) {
        
        // 1. 创建API请求
        ChatCompletionRequest request = createRequest(prompt, false);
        
        // 2. 创建观测上下文（用于监控）
        ChatModelObservationContext observationContext = 
            ChatModelObservationContext.builder()
                .prompt(prompt)
                .provider("OpenAI")
                .build();
        
        // 3. 使用观测和重试执行API调用
        return ChatModelObservationDocumentation.CHAT_MODEL_OPERATION
            .observation(observationConvention, 
                        DEFAULT_OBSERVATION_CONVENTION,
                        () -> observationContext,
                        observationRegistry)
            .observe(() -> {
                // 执行API调用
                ResponseEntity<ChatCompletion> completionEntity = 
                    retryTemplate.execute(ctx -> 
                        openAiApi.chatCompletionEntity(
                            request, 
                            getAdditionalHttpHeaders(prompt)
                        )
                    );
                
                // 4. 处理响应
                ChatCompletion chatCompletion = completionEntity.getBody();
                
                // 5. 转换为ChatResponse
                List<Generation> generations = 
                    chatCompletion.choices().stream()
                        .map(choice -> buildGeneration(
                            choice, 
                            metadata,
                            request
                        ))
                        .toList();
                
                // 6. 提取元数据
                RateLimit rateLimit = 
                    OpenAiResponseHeaderExtractor
                        .extractAiResponseHeaders(completionEntity);
                
                Usage usage = toSpringAiUsage(chatCompletion.usage());
                
                ChatResponseMetadata responseMetadata = 
                    ChatResponseMetadata.builder()
                        .usage(usage)
                        .rateLimit(rateLimit)
                        .model(chatCompletion.model())
                        .build();
                
                // 7. 创建响应
                ChatResponse chatResponse = new ChatResponse(
                    generations,
                    responseMetadata
                );
                
                // 8. 处理Function Calling
                if (isToolCall(chatResponse)) {
                    return handleToolCalls(prompt, chatResponse);
                }
                
                return chatResponse;
            });
    }
    
    @Override
    public Flux<ChatResponse> stream(Prompt prompt) {
        // 1. 构建请求
        Prompt requestPrompt = buildRequestPrompt(prompt);
        
        // 2. 调用流式实现
        return internalStream(requestPrompt, null);
    }
    
    public Flux<ChatResponse> internalStream(
            Prompt prompt,
            ChatResponse previousResponse) {
        
        return Flux.deferContextual(contextView -> {
            // 1. 创建流式请求
            ChatCompletionRequest request = createRequest(prompt, true);
            
            // 2. 调用流式API
            Flux<ChatCompletionChunk> completionChunks = 
                openAiApi.chatCompletionStream(
                    request,
                    getAdditionalHttpHeaders(prompt)
                );
            
            // 3. 创建观测上下文
            ChatModelObservationContext observationContext = 
                ChatModelObservationContext.builder()
                    .prompt(prompt)
                    .provider("OpenAI")
                    .build();
            
            // 4. 启动观测
            Observation observation = 
                ChatModelObservationDocumentation.CHAT_MODEL_OPERATION
                    .observation(observationConvention,
                                DEFAULT_OBSERVATION_CONVENTION,
                                () -> observationContext,
                                observationRegistry);
            
            observation.parentObservation(
                contextView.getOrDefault(
                    ObservationThreadLocalAccessor.KEY, 
                    null
                )
            ).start();
            
            // 5. 处理流式响应
            // 角色映射（第一个chunk包含角色，后续chunk共享）
            ConcurrentHashMap<String, String> roleMap = 
                new ConcurrentHashMap<>();
            
            return completionChunks
                // 转换chunk为ChatResponse
                .map(chunk -> toChatResponse(chunk, roleMap))
                // 聚合工具调用
                .windowUntil(AdvisorUtils.onFinishReason())
                .flatMap(window -> MessageAggregator.aggregate(
                    window, 
                    chatResponse -> {
                        // 处理Function Calling
                        if (isToolCall(chatResponse)) {
                            return handleToolCalls(prompt, chatResponse);
                        }
                        return Flux.just(chatResponse);
                    }
                ))
                // 完成观测
                .doOnError(observation::error)
                .doFinally(s -> observation.stop())
                .contextWrite(ctx -> ctx.put(
                    ObservationThreadLocalAccessor.KEY, 
                    observation
                ));
        });
    }
    
    // 处理Function Calling
    private ChatResponse handleToolCalls(
            Prompt prompt,
            ChatResponse chatResponse) {
        
        // 1. 提取工具调用
        List<ToolCall> toolCalls = extractToolCalls(chatResponse);
        
        // 2. 执行工具
        List<ToolResponse> toolResponses = 
            toolCallingManager.executeMethods(toolCalls);
        
        // 3. 构建新的Prompt（包含工具执行结果）
        List<Message> newMessages = new ArrayList<>(
            prompt.getInstructions()
        );
        newMessages.add(chatResponse.getResult().getOutput());
        newMessages.add(new ToolResponseMessage(toolResponses));
        
        Prompt newPrompt = new Prompt(
            newMessages,
            prompt.getOptions()
        );
        
        // 4. 递归调用（使用工具执行结果）
        return internalCall(newPrompt, chatResponse);
    }
}
```

### OpenAiChatOptions (OpenAI特有配置)

```java
public class OpenAiChatOptions implements ChatOptions {
    
    // 通用配置
    private String model;
    private Double temperature;
    private Integer maxTokens;
    private Double frequencyPenalty;
    private Double presencePenalty;
    private Double topP;
    private List<String> stopSequences;
    
    // OpenAI特有配置
    private Integer n;  // 生成数量
    private String user;  // 用户标识
    private Long seed;  // 随机种子（确定性输出）
    private ResponseFormat responseFormat;  // 输出格式（JSON等）
    private Double logitBias;  // Token偏置
    private Boolean logprobs;  // 返回概率
    private Integer topLogprobs;  // 返回Top N概率
    private Boolean stream;  // 流式输出
    private List<OpenAiApi.FunctionTool> functions;  // 函数定义
    private String toolChoice;  // 工具选择策略
    
    // Vision相关
    private String imageDetail;  // 图片细节级别
    
    // Audio相关
    private List<OutputModality> outputModalities;  // 输出模态
    private AudioParameters audioParameters;  // 音频参数
}
```

### 使用OpenAI特有功能

```java
// 1. JSON模式输出
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .model("gpt-4")
    .responseFormat(OpenAiApi.ResponseFormat.builder()
        .type("json_object")
        .build())
    .build();

// 2. 确定性输出（使用seed）
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .model("gpt-4")
    .seed(12345L)  // 相同seed产生相同结果
    .temperature(0.0)
    .build();

// 3. Function Calling
OpenAiChatOptions options = OpenAiChatOptions.builder()
    .model("gpt-4")
    .functions(List.of(
        OpenAiApi.FunctionTool.builder()
            .name("get_weather")
            .description("获取天气信息")
            .parameters(JsonSchemaGenerator.generate(
                WeatherRequest.class
            ))
            .build()
    ))
    .toolChoice("auto")  // "auto", "none", 或具体函数名
    .build();

// 4. Vision（图像理解）
UserMessage visionMessage = UserMessage.builder()
    .text("这张图片里有什么？")
    .media(new Media(
        MimeTypeUtils.IMAGE_PNG,
        new URL("https://example.com/image.png")
    ))
    .build();

OpenAiChatOptions options = OpenAiChatOptions.builder()
    .model("gpt-4-vision-preview")
    .imageDetail("high")  // "auto", "low", "high"
    .build();
```

---

## 最佳实践

### 1. 配置管理

```java
@Configuration
public class ChatModelConfig {
    
    // 1. 基础配置
    @Bean
    public ChatOptions defaultChatOptions() {
        return ChatOptions.builder()
            .temperature(0.7)
            .maxTokens(1000)
            .build();
    }
    
    // 2. 针对不同场景的配置
    @Bean("creativeOptions")
    public ChatOptions creativeChatOptions() {
        return ChatOptions.builder()
            .temperature(0.9)  // 高创造性
            .maxTokens(2000)
            .frequencyPenalty(0.5)
            .presencePenalty(0.6)
            .build();
    }
    
    @Bean("factualOptions")
    public ChatOptions factualChatOptions() {
        return ChatOptions.builder()
            .temperature(0.1)  // 低温度，更准确
            .maxTokens(500)
            .build();
    }
    
    @Bean("codeGenerationOptions")
    public ChatOptions codeGenerationOptions() {
        return ChatOptions.builder()
            .temperature(0.2)  // 代码生成需要准确性
            .maxTokens(2000)
            .stopSequences(List.of("```"))
            .build();
    }
}
```

### 2. 异常处理

```java
@Service
public class ResilientChatService {
    
    private final ChatModel chatModel;
    private final RetryTemplate retryTemplate;
    
    public String chat(String message) {
        try {
            return retryTemplate.execute(context -> {
                try {
                    return chatModel.call(message);
                } catch (Exception e) {
                    // 日志记录
                    logger.error("Chat failed", e);
                    
                    // 根据异常类型决定是否重试
                    if (isRetryable(e)) {
                        throw e;  // 重试
                    } else {
                        return getFallbackResponse();
                    }
                }
            });
        } catch (Exception e) {
            // 最终失败，返回友好错误消息
            return "抱歉，服务暂时不可用，请稍后重试。";
        }
    }
    
    private boolean isRetryable(Exception e) {
        // 判断异常是否可重试
        return e instanceof TimeoutException ||
               e instanceof IOException ||
               e.getMessage().contains("rate_limit");
    }
    
    private String getFallbackResponse() {
        return "抱歉，我现在无法处理您的请求。";
    }
}
```

### 3. Token管理

```java
@Service
public class TokenAwareChatService {
    
    private final ChatModel chatModel;
    private final TokenCounter tokenCounter;
    
    public String chat(
            String userMessage,
            List<Message> history) {
        
        // 1. 计算当前Token数
        int totalTokens = tokenCounter.countTokens(userMessage);
        for (Message msg : history) {
            totalTokens += tokenCounter.countTokens(msg.getText());
        }
        
        // 2. 如果超过限制，裁剪历史
        int maxContextTokens = 3000;  // 留1000给响应
        if (totalTokens > maxContextTokens) {
            history = truncateHistory(history, maxContextTokens);
        }
        
        // 3. 构建Prompt
        List<Message> messages = new ArrayList<>(history);
        messages.add(new UserMessage(userMessage));
        
        // 4. 调用模型
        ChatResponse response = chatModel.call(
            new Prompt(messages)
        );
        
        // 5. 记录Token使用
        Usage usage = response.getMetadata().getUsage();
        logger.info("Token usage: prompt={}, completion={}, total={}",
            usage.getPromptTokens(),
            usage.getCompletionTokens(),
            usage.getTotalTokens()
        );
        
        return response.getResult().getOutput().getText();
    }
    
    private List<Message> truncateHistory(
            List<Message> history,
            int maxTokens) {
        // 保留最近的消息，直到Token数在限制内
        List<Message> truncated = new ArrayList<>();
        int currentTokens = 0;
        
        for (int i = history.size() - 1; i >= 0; i--) {
            Message msg = history.get(i);
            int msgTokens = tokenCounter.countTokens(msg.getText());
            
            if (currentTokens + msgTokens <= maxTokens) {
                truncated.add(0, msg);  // 添加到开头
                currentTokens += msgTokens;
            } else {
                break;
            }
        }
        
        return truncated;
    }
}
```

### 4. 流式处理最佳实践

```java
@Service
public class StreamingChatService {
    
    private final ChatModel chatModel;
    
    public Flux<String> chatStream(String message) {
        return chatModel.stream(new Prompt(message))
            // 提取文本内容
            .mapNotNull(response -> {
                Generation result = response.getResult();
                return result != null ? 
                    result.getOutput().getText() : null;
            })
            // 过滤空内容
            .filter(StringUtils::hasText)
            // 错误处理
            .onErrorResume(error -> {
                logger.error("Stream error", error);
                return Flux.just("[Error occurred]");
            })
            // 超时处理
            .timeout(Duration.ofSeconds(30))
            // 日志记录
            .doOnComplete(() -> 
                logger.info("Stream completed")
            )
            .doOnError(error -> 
                logger.error("Stream failed", error)
            );
    }
    
    // Server-Sent Events示例
    @GetMapping(value = "/chat/stream",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamChat(
            @RequestParam String message) {
        
        return chatStream(message)
            .map(content -> ServerSentEvent.<String>builder()
                .data(content)
                .build())
            .concatWith(Flux.just(
                ServerSentEvent.<String>builder()
                    .event("end")
                    .data("[DONE]")
                    .build()
            ));
    }
}
```

---

## 实战示例

### 示例1: 多模型策略

根据任务选择不同的模型。

```java
@Service
public class MultiModelChatService {
    
    private final Map<String, ChatModel> models;
    
    public MultiModelChatService(
            @Qualifier("gpt4Model") ChatModel gpt4,
            @Qualifier("gpt35Model") ChatModel gpt35,
            @Qualifier("ollamaModel") ChatModel ollama) {
        
        this.models = Map.of(
            "complex", gpt4,      // 复杂任务
            "simple", gpt35,      // 简单任务
            "local", ollama       // 本地任务
        );
    }
    
    public String chat(String message, TaskComplexity complexity) {
        // 根据复杂度选择模型
        String modelKey = switch (complexity) {
            case COMPLEX -> "complex";
            case SIMPLE -> "simple";
            case LOCAL -> "local";
        };
        
        ChatModel model = models.get(modelKey);
        return model.call(message);
    }
    
    public String smartChat(String message) {
        // 智能选择模型
        TaskComplexity complexity = analyzeComplexity(message);
        return chat(message, complexity);
    }
    
    private TaskComplexity analyzeComplexity(String message) {
        // 分析任务复杂度
        if (message.length() > 500 || 
            message.contains("分析") ||
            message.contains("详细")) {
            return TaskComplexity.COMPLEX;
        } else if (message.contains("本地") ||
                   message.contains("离线")) {
            return TaskComplexity.LOCAL;
        } else {
            return TaskComplexity.SIMPLE;
        }
    }
    
    enum TaskComplexity {
        SIMPLE, COMPLEX, LOCAL
    }
}
```

### 示例2: 批量处理

高效处理多个请求。

```java
@Service
public class BatchChatService {
    
    private final ChatModel chatModel;
    private final Scheduler scheduler = 
        Schedulers.boundedElastic();
    
    public List<String> batchChat(List<String> messages) {
        return Flux.fromIterable(messages)
            .parallel()
            .runOn(scheduler)
            .map(message -> {
                try {
                    return chatModel.call(message);
                } catch (Exception e) {
                    logger.error("Failed to process: {}", message, e);
                    return "Error: " + e.getMessage();
                }
            })
            .sequential()
            .collectList()
            .block();
    }
    
    public Flux<ChatResult> batchChatAsync(List<String> messages) {
        return Flux.fromIterable(messages)
            .flatMap(message -> 
                Mono.fromCallable(() -> {
                    long startTime = System.currentTimeMillis();
                    String response = chatModel.call(message);
                    long duration = System.currentTimeMillis() - startTime;
                    
                    return new ChatResult(
                        message,
                        response,
                        duration
                    );
                })
                .subscribeOn(scheduler)
            )
            .onErrorContinue((error, obj) -> 
                logger.error("Failed to process: {}", obj, error)
            );
    }
    
    record ChatResult(
        String input,
        String output,
        long durationMs
    ) {}
}
```

### 示例3: 缓存实现

减少API调用，提高响应速度。

```java
@Service
public class CachedChatService {
    
    private final ChatModel chatModel;
    private final Cache<String, String> cache;
    
    public CachedChatService(ChatModel chatModel) {
        this.chatModel = chatModel;
        this.cache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(1, TimeUnit.HOURS)
            .recordStats()
            .build();
    }
    
    public String chat(String message) {
        // 尝试从缓存获取
        try {
            return cache.get(message, () -> {
                logger.info("Cache miss, calling AI model");
                return chatModel.call(message);
            });
        } catch (ExecutionException e) {
            logger.error("Cache error", e);
            return chatModel.call(message);
        }
    }
    
    public CacheStats getCacheStats() {
        return cache.stats();
    }
    
    public void clearCache() {
        cache.invalidateAll();
    }
}
```

---

## 总结

### ChatModel核心要点

1. **统一抽象**: 为所有AI模型提供一致的API
2. **同步+流式**: 支持阻塞式和响应式两种调用方式
3. **配置灵活**: 通过ChatOptions控制模型行为
4. **可扩展**: 易于实现新的模型提供商
5. **类型安全**: 基于强类型的Java API

### 接口层次

```
Model<TReq, TRes>               (根接口 - 同步)
    ↑
ChatModel  ←── StreamingChatModel  (流式接口)
    │               ↑
    │               │
    │      StreamingModel<TReq, TRes>
    │
    └─ 便捷方法 + 核心方法 + 配置方法
```

### 关键组件

- **Prompt**: 请求封装（消息列表 + 配置）
- **ChatResponse**: 响应封装（生成结果 + 元数据）
- **ChatOptions**: 配置选项（温度、Token限制等）
- **Message**: 消息类型（System、User、Assistant、Tool）

### 最佳实践

1. 根据场景选择合适的配置
2. 妥善处理异常和重试
3. 管理Token使用
4. 利用流式API提升用户体验
5. 实现缓存减少API调用

通过掌握ChatModel，你可以轻松地集成和切换不同的AI模型，构建强大的AI应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

