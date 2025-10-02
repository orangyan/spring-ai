# Spring AI Advisor 机制详解

## 📋 目录
- [概述](#概述)
- [核心架构](#核心架构)
- [Advisor接口体系](#advisor接口体系)
- [AdvisorChain责任链](#advisorchain责任链)
- [执行流程](#执行流程)
- [内置Advisor实现](#内置advisor实现)
- [自定义Advisor](#自定义advisor)
- [最佳实践](#最佳实践)
- [实战示例](#实战示例)

---

## 概述

Advisor（顾问）是Spring AI中的核心设计模式，它基于**责任链模式（Chain of Responsibility）**和**拦截器模式（Interceptor）**，提供了强大的请求/响应拦截和增强能力。

### 核心价值

1. **解耦关注点**: 将横切关注点（如日志、记忆、安全）与核心业务逻辑分离
2. **可组合性**: 多个Advisor可以灵活组合，形成处理链
3. **可复用性**: Advisor可以在不同场景中重复使用
4. **可扩展性**: 易于添加新的处理逻辑，无需修改现有代码
5. **有序执行**: 通过Order控制Advisor的执行顺序

### 设计哲学

```
用户请求 → Advisor1 → Advisor2 → ... → ChatModel → ... → Advisor2 → Advisor1 → 响应
         ↓         ↓                              ↓         ↓
       before    before                        after     after
```

Advisor在请求到达AI模型之前和之后都可以进行处理：
- **Before**: 修改请求、增强提示词、添加上下文
- **After**: 处理响应、保存记忆、添加元数据

---

## 核心架构

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                      Advisor 体系                            │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐                                             │
│  │  Advisor    │  (根接口)                                   │
│  └──────┬──────┘                                             │
│         │                                                     │
│    ┌────┴────┐                                               │
│    │         │                                               │
│ ┌──▼───┐  ┌─▼────┐                                          │
│ │Call  │  │Stream│  (调用类型)                              │
│ │Advisor│  │Advisor│                                        │
│ └──┬───┘  └─┬────┘                                          │
│    │        │                                                │
│    └────┬───┘                                                │
│         │                                                    │
│    ┌────▼────────┐                                          │
│    │ BaseAdvisor │  (便利基类)                              │
│    └─────────────┘                                          │
│         │                                                    │
│  ┌──────┴───────────────────────┐                          │
│  │                                │                          │
│  ▼                                ▼                          │
│ 具体实现                        自定义实现                  │
│ - MessageChatMemoryAdvisor      - 日志Advisor               │
│ - PromptChatMemoryAdvisor       - 安全Advisor               │
│ - QuestionAnswerAdvisor         - 监控Advisor               │
│ - SafeGuardAdvisor              - ...                       │
│ - SimpleLoggerAdvisor                                       │
│ - ChatModelCallAdvisor                                      │
└─────────────────────────────────────────────────────────────┘
```

### 执行链路图

```
ChatClient.call()
    │
    ├─ buildAdvisorChain()
    │   │
    │   ├─ 添加用户Advisors (order: 小到大)
    │   ├─ 添加默认Advisors
    │   └─ 添加ChatModelCallAdvisor (最后执行，实际调用AI)
    │
    └─ DefaultAroundAdvisorChain.nextCall()
        │
        ├─ Advisor1.adviseCall()
        │   ├─ before()  - 预处理
        │   ├─ chain.nextCall()  - 调用下一个
        │   └─ after()   - 后处理
        │
        ├─ Advisor2.adviseCall()
        │   ├─ before()
        │   ├─ chain.nextCall()
        │   └─ after()
        │
        └─ ChatModelCallAdvisor.adviseCall()
            └─ chatModel.call()  - 实际AI调用
```

---

## Advisor接口体系

### 1. Advisor (根接口)

所有Advisor的根接口，继承自Spring的`Ordered`接口。

```java
public interface Advisor extends Ordered {
    
    /**
     * 默认记忆Advisor的优先级
     * 确保用户可以插入更高优先级的Advisor
     */
    int DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER = 
        Ordered.HIGHEST_PRECEDENCE + 1000;
    
    /**
     * 返回Advisor的名称
     */
    String getName();
    
    /**
     * 返回执行顺序
     * 数字越小，优先级越高，越早执行
     */
    int getOrder();
}
```

#### 优先级规则

```java
// Order值越小，优先级越高
Advisor1(order=0)   →  最先执行before，最后执行after
Advisor2(order=100) →  后执行before，先执行after
Advisor3(order=200) →  最后执行before，最先执行after

// 执行顺序示意
Before: Advisor1 → Advisor2 → Advisor3 → ChatModel
After:  Advisor3 ← Advisor2 ← Advisor1 ← ChatModel Response
```

### 2. CallAdvisor (同步调用Advisor)

处理同步调用的Advisor接口。

```java
public interface CallAdvisor extends Advisor {
    
    /**
     * 拦截并处理同步调用
     * @param chatClientRequest 请求对象
     * @param callAdvisorChain 责任链
     * @return 响应对象
     */
    ChatClientResponse adviseCall(
        ChatClientRequest chatClientRequest,
        CallAdvisorChain callAdvisorChain
    );
}
```

#### 典型实现模式

```java
public class MyCallAdvisor implements CallAdvisor {
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 1. Before 处理
        // 修改请求、添加上下文等
        ChatClientRequest modifiedRequest = modifyRequest(request);
        
        // 2. 调用链中的下一个Advisor
        ChatClientResponse response = chain.nextCall(modifiedRequest);
        
        // 3. After 处理
        // 处理响应、保存数据等
        ChatClientResponse modifiedResponse = modifyResponse(response);
        
        return modifiedResponse;
    }
    
    @Override
    public int getOrder() {
        return 0;  // 优先级
    }
    
    @Override
    public String getName() {
        return "MyCallAdvisor";
    }
}
```

### 3. StreamAdvisor (流式调用Advisor)

处理流式调用的Advisor接口。

```java
public interface StreamAdvisor extends Advisor {
    
    /**
     * 拦截并处理流式调用
     * @param chatClientRequest 请求对象
     * @param streamAdvisorChain 责任链
     * @return 响应流
     */
    Flux<ChatClientResponse> adviseStream(
        ChatClientRequest chatClientRequest,
        StreamAdvisorChain streamAdvisorChain
    );
}
```

#### 流式处理特点

流式调用需要处理：
1. **异步性**: 使用Reactor的Flux处理异步流
2. **背压**: 控制数据流速
3. **Scheduler**: 指定执行的调度器

```java
public class MyStreamAdvisor implements StreamAdvisor {
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        // Before处理（在流开始前）
        ChatClientRequest modifiedRequest = modifyRequest(request);
        
        // 获取响应流
        Flux<ChatClientResponse> responseFlux = 
            chain.nextStream(modifiedRequest);
        
        // After处理（对每个流元素进行处理）
        return responseFlux.map(response -> {
            // 处理每个响应片段
            return modifyStreamResponse(response);
        });
    }
}
```

### 4. BaseAdvisor (便利基类)

同时实现CallAdvisor和StreamAdvisor，简化开发。

```java
public interface BaseAdvisor extends CallAdvisor, StreamAdvisor {
    
    Scheduler DEFAULT_SCHEDULER = Schedulers.boundedElastic();
    
    /**
     * 默认的Call实现
     */
    @Override
    default ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 调用before方法
        ChatClientRequest processedRequest = before(request, chain);
        
        // 调用链中的下一个
        ChatClientResponse response = chain.nextCall(processedRequest);
        
        // 调用after方法
        return after(response, chain);
    }
    
    /**
     * 默认的Stream实现
     */
    @Override
    default Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        // 使用Reactor处理异步流
        Flux<ChatClientResponse> responseFlux = Mono.just(request)
            .publishOn(getScheduler())
            .map(req -> this.before(req, chain))
            .flatMapMany(chain::nextStream);
        
        // 只在流结束时调用after
        return responseFlux.map(response -> {
            if (AdvisorUtils.onFinishReason().test(response)) {
                response = after(response, chain);
            }
            return response;
        });
    }
    
    /**
     * Before处理逻辑（子类实现）
     */
    ChatClientRequest before(
        ChatClientRequest request,
        AdvisorChain chain
    );
    
    /**
     * After处理逻辑（子类实现）
     */
    ChatClientResponse after(
        ChatClientResponse response,
        AdvisorChain chain
    );
    
    /**
     * 流式调用的调度器
     */
    default Scheduler getScheduler() {
        return DEFAULT_SCHEDULER;
    }
}
```

#### 使用BaseAdvisor的优势

1. **代码简洁**: 只需实现`before()`和`after()`两个方法
2. **自动兼容**: 同时支持同步和流式调用
3. **模板实现**: 提供了标准的执行流程
4. **异步支持**: 内置Reactor支持

```java
// 使用BaseAdvisor的简化实现
public class SimpleAdvisor implements BaseAdvisor {
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        // 预处理逻辑
        return request;
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        // 后处理逻辑
        return response;
    }
    
    @Override
    public int getOrder() {
        return 0;
    }
}
```

---

## AdvisorChain责任链

### 1. AdvisorChain (根接口)

```java
public interface AdvisorChain {
    
    /**
     * 获取观测注册表（用于监控）
     */
    default ObservationRegistry getObservationRegistry() {
        return ObservationRegistry.NOOP;
    }
}
```

### 2. CallAdvisorChain (同步链)

```java
public interface CallAdvisorChain extends AdvisorChain {
    
    /**
     * 调用链中的下一个CallAdvisor
     */
    ChatClientResponse nextCall(ChatClientRequest request);
    
    /**
     * 获取链中所有的CallAdvisor
     */
    List<CallAdvisor> getCallAdvisors();
}
```

### 3. StreamAdvisorChain (流式链)

```java
public interface StreamAdvisorChain extends AdvisorChain {
    
    /**
     * 调用链中的下一个StreamAdvisor
     */
    Flux<ChatClientResponse> nextStream(ChatClientRequest request);
    
    /**
     * 获取链中所有的StreamAdvisor
     */
    List<StreamAdvisor> getStreamAdvisors();
}
```

### 4. DefaultAroundAdvisorChain (默认实现)

这是Spring AI提供的默认链实现，使用栈结构管理Advisor。

```java
public class DefaultAroundAdvisorChain 
    implements CallAdvisorChain, StreamAdvisorChain {
    
    // 使用栈存储Advisor（支持pop操作）
    private final Deque<CallAdvisor> callAdvisors;
    private final Deque<StreamAdvisor> streamAdvisors;
    private final ObservationRegistry observationRegistry;
    
    @Override
    public ChatClientResponse nextCall(ChatClientRequest request) {
        if (callAdvisors.isEmpty()) {
            throw new IllegalStateException(
                "No CallAdvisors available to execute"
            );
        }
        
        // 从栈中弹出下一个Advisor
        var advisor = callAdvisors.pop();
        
        // 创建观测上下文（用于监控）
        var observationContext = AdvisorObservationContext.builder()
            .advisorName(advisor.getName())
            .chatClientRequest(request)
            .order(advisor.getOrder())
            .build();
        
        // 使用观测机制执行Advisor
        return AdvisorObservationDocumentation.AI_ADVISOR
            .observation(null, DEFAULT_OBSERVATION_CONVENTION,
                        () -> observationContext, observationRegistry)
            .observe(() -> advisor.adviseCall(request, this));
    }
    
    @Override
    public Flux<ChatClientResponse> nextStream(ChatClientRequest request) {
        return Flux.deferContextual(contextView -> {
            if (streamAdvisors.isEmpty()) {
                return Flux.error(new IllegalStateException(
                    "No StreamAdvisors available to execute"
                ));
            }
            
            var advisor = streamAdvisors.pop();
            
            // 创建观测上下文
            var observationContext = AdvisorObservationContext.builder()
                .advisorName(advisor.getName())
                .chatClientRequest(request)
                .order(advisor.getOrder())
                .build();
            
            var observation = AdvisorObservationDocumentation.AI_ADVISOR
                .observation(null, DEFAULT_OBSERVATION_CONVENTION,
                            () -> observationContext, observationRegistry);
            
            observation.parentObservation(
                contextView.getOrDefault(
                    ObservationThreadLocalAccessor.KEY, null
                )
            ).start();
            
            // 执行并传播观测上下文
            return Flux.defer(() -> advisor.adviseStream(request, this)
                .doOnError(observation::error)
                .doFinally(s -> observation.stop())
                .contextWrite(ctx -> ctx.put(
                    ObservationThreadLocalAccessor.KEY, observation
                ))
            );
        });
    }
}
```

#### Chain构建过程

```java
// 在ChatClient中构建Advisor链
private BaseAdvisorChain buildAdvisorChain() {
    // 1. 添加用户定义的Advisors
    List<Advisor> advisors = new ArrayList<>(this.advisors);
    
    // 2. 在链的末尾添加模型调用Advisors
    //    这些是最后执行的，负责实际调用AI模型
    advisors.add(
        ChatModelCallAdvisor.builder()
            .chatModel(this.chatModel)
            .build()
    );
    advisors.add(
        ChatModelStreamAdvisor.builder()
            .chatModel(this.chatModel)
            .build()
    );
    
    // 3. 构建链（会自动按Order排序）
    return DefaultAroundAdvisorChain.builder(observationRegistry)
        .pushAll(advisors)
        .build();
}
```

---

## 执行流程

### 同步调用流程

```java
// 用户代码
String result = chatClient
    .prompt("Hello")
    .advisors(advisor1, advisor2)
    .call()
    .content();

// 内部执行流程
1. ChatClient.call()
   │
2. ├─ buildAdvisorChain()
   │   └─ 按Order排序: [Advisor1, Advisor2, ChatModelCallAdvisor]
   │
3. └─ chain.nextCall(request)
       │
4.     ├─ Advisor1.adviseCall()
       │   ├─ before(request)         → request1'
       │   ├─ chain.nextCall(request1')
       │   │   │
5.     │   │   └─ Advisor2.adviseCall()
       │   │       ├─ before(request1')    → request2'
       │   │       ├─ chain.nextCall(request2')
       │   │       │   │
6.     │   │       │   └─ ChatModelCallAdvisor.adviseCall()
       │   │       │       └─ chatModel.call(request2')  → response0
       │   │       │
7.     │   │       └─ after(response0)      → response2'
       │   │
8.     │   └─ after(response2')       → response1'
       │
9.     └─ return response1'
```

### 流式调用流程

```java
// 用户代码
Flux<String> stream = chatClient
    .prompt("Hello")
    .advisors(advisor1, advisor2)
    .stream()
    .content();

// 内部执行流程（异步）
1. ChatClient.stream()
   │
2. ├─ buildAdvisorChain()
   │
3. └─ chain.nextStream(request)
       │
4.     ├─ Advisor1.adviseStream()
       │   ├─ before(request) on Scheduler
       │   ├─ chain.nextStream()
       │   │   │
5.     │   │   └─ Advisor2.adviseStream()
       │   │       ├─ before(request)
       │   │       ├─ chain.nextStream()
       │   │       │   │
6.     │   │       │   └─ ChatModelStreamAdvisor.adviseStream()
       │   │       │       └─ chatModel.stream() → Flux<Response>
       │   │       │
7.     │   │       └─ map(response -> after(response))
       │   │
8.     │   └─ map(response -> after(response))
       │
9.     └─ return Flux<Response>
```

### 执行顺序可视化

```
Time ─────────────────────────────────────────────────►

Request
  │
  ├──► Advisor1.before()   (Order=0, 最高优先级)
  │      │
  │      ├──► Advisor2.before()   (Order=100)
  │      │      │
  │      │      ├──► ChatModelCallAdvisor
  │      │      │      │
  │      │      │      └──► AI Model Call
  │      │      │             │
  │      │      │      ┌──────┘
  │      │      │      │
  │      │      └──────┤
  │      │             │
  │      │      Advisor2.after()
  │      │             │
  │      └─────────────┤
  │                    │
  │      Advisor1.after()
  │                    │
  └────────────────────┘
                       │
                    Response
```

---

## 内置Advisor实现

Spring AI提供了多个开箱即用的Advisor实现。

### 1. MessageChatMemoryAdvisor

将对话历史存储为消息列表并注入到提示词中。

#### 功能

- 自动保存用户消息和AI响应
- 按会话ID隔离对话
- 支持消息窗口大小限制
- 将历史消息直接添加到Prompt的messages列表

#### 源码解析

```java
public class MessageChatMemoryAdvisor implements BaseAdvisor {
    
    private final ChatMemory chatMemory;
    private final String defaultConversationId;
    private final int order;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        String conversationId = getConversationId(
            request.context(),
            defaultConversationId
        );
        
        // 1. 从内存中获取历史消息
        List<Message> memoryMessages = chatMemory.get(conversationId);
        
        // 2. 将历史消息添加到当前Prompt
        List<Message> allMessages = new ArrayList<>();
        
        // 保留原有的System Message
        SystemMessage systemMessage = request.prompt().getSystemMessage();
        if (systemMessage != null && 
            StringUtils.hasText(systemMessage.getText())) {
            allMessages.add(systemMessage);
        }
        
        // 添加历史消息
        allMessages.addAll(memoryMessages);
        
        // 添加当前用户消息
        UserMessage currentUserMessage = 
            request.prompt().getUserMessage();
        allMessages.add(currentUserMessage);
        
        // 3. 创建新的Prompt
        Prompt newPrompt = new Prompt(
            allMessages,
            request.prompt().getOptions()
        );
        
        return request.mutate()
            .prompt(newPrompt)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        String conversationId = getConversationId(
            response.context(),
            defaultConversationId
        );
        
        // 保存AI响应到记忆
        AssistantMessage assistantMessage = 
            response.chatResponse().getResult().getOutput();
        
        chatMemory.add(conversationId, assistantMessage);
        
        return response;
    }
}
```

#### 使用示例

```java
// 1. 创建ChatMemory
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(new InMemoryChatMemoryRepository())
    .maxMessages(10)  // 最多保留10条消息
    .build();

// 2. 创建Advisor
MessageChatMemoryAdvisor memoryAdvisor = 
    MessageChatMemoryAdvisor.builder(chatMemory)
        .conversationId("session-123")
        .order(Advisor.DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER)
        .build();

// 3. 使用
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(memoryAdvisor)
    .build();

// 第一轮对话
chatClient.prompt("我叫张三").call().content();
// → "你好，张三！"

// 第二轮对话（会自动包含历史）
chatClient.prompt("我叫什么名字？").call().content();
// → "你叫张三。"
```

### 2. PromptChatMemoryAdvisor

将对话历史格式化为字符串，注入到系统提示词中。

#### 功能

- 将历史对话格式化为文本
- 使用模板注入到System Message
- 更灵活的格式控制

#### 源码解析

```java
public class PromptChatMemoryAdvisor implements BaseAdvisor {
    
    private static final PromptTemplate DEFAULT_SYSTEM_PROMPT_TEMPLATE = 
        new PromptTemplate("""
            {instructions}
            
            Use the conversation memory from the MEMORY section to provide 
            accurate answers.
            
            ---------------------
            MEMORY:
            {memory}
            ---------------------
            """);
    
    private final ChatMemory chatMemory;
    private final PromptTemplate systemPromptTemplate;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        String conversationId = getConversationId(
            request.context(),
            defaultConversationId
        );
        
        // 1. 获取历史消息
        List<Message> memoryMessages = chatMemory.get(conversationId);
        
        // 2. 格式化为字符串
        String memory = memoryMessages.stream()
            .filter(m -> m.getMessageType() == MessageType.USER ||
                        m.getMessageType() == MessageType.ASSISTANT)
            .map(m -> m.getMessageType() + ":" + m.getText())
            .collect(Collectors.joining(System.lineSeparator()));
        
        // 3. 使用模板增强系统提示词
        SystemMessage systemMessage = request.prompt().getSystemMessage();
        String augmentedSystemText = systemPromptTemplate.render(
            Map.of(
                "instructions", systemMessage.getText(),
                "memory", memory
            )
        );
        
        // 4. 更新请求
        ChatClientRequest processedRequest = request.mutate()
            .prompt(request.prompt()
                .augmentSystemMessage(augmentedSystemText))
            .build();
        
        // 5. 保存当前用户消息
        chatMemory.add(
            conversationId,
            processedRequest.prompt().getUserMessage()
        );
        
        return processedRequest;
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        String conversationId = getConversationId(
            response.context(),
            defaultConversationId
        );
        
        // 保存AI响应
        AssistantMessage assistantMessage = 
            response.chatResponse().getResult().getOutput();
        
        chatMemory.add(conversationId, assistantMessage);
        
        return response;
    }
}
```

#### 使用示例

```java
// 自定义提示词模板
PromptTemplate customTemplate = new PromptTemplate("""
    System: {instructions}
    
    Previous Conversation:
    {memory}
    
    Please maintain context from previous messages.
    """);

PromptChatMemoryAdvisor advisor = 
    PromptChatMemoryAdvisor.builder(chatMemory)
        .systemPromptTemplate(customTemplate)
        .conversationId("user-456")
        .build();
```

### 3. QuestionAnswerAdvisor (RAG)

从向量数据库检索相关文档并注入到提示词中。

详见[提示词工程文档](./Spring-AI提示词工程详解.md)的RAG章节。

```java
QuestionAnswerAdvisor qaAdvisor = 
    QuestionAnswerAdvisor.builder(vectorStore)
        .searchRequest(SearchRequest.builder()
            .topK(5)
            .similarityThreshold(0.75)
            .build())
        .build();

chatClient
    .prompt("Spring AI有哪些特性？")
    .advisors(qaAdvisor)
    .call()
    .content();
```

### 4. SimpleLoggerAdvisor

记录请求和响应日志。

#### 源码解析

```java
public class SimpleLoggerAdvisor implements CallAdvisor, StreamAdvisor {
    
    private static final Logger logger = 
        LoggerFactory.getLogger(SimpleLoggerAdvisor.class);
    
    private final Function<ChatClientRequest, String> requestToString;
    private final Function<ChatResponse, String> responseToString;
    private final int order;
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 记录请求
        logger.debug("request: {}", requestToString.apply(request));
        
        // 调用链
        ChatClientResponse response = chain.nextCall(request);
        
        // 记录响应
        logger.debug("response: {}", 
                    responseToString.apply(response.chatResponse()));
        
        return response;
    }
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        // 记录请求
        logRequest(request);
        
        // 获取响应流
        Flux<ChatClientResponse> responses = chain.nextStream(request);
        
        // 聚合流式响应并记录
        return new ChatClientMessageAggregator()
            .aggregateChatClientResponse(responses, this::logResponse);
    }
}
```

#### 使用示例

```java
// 基本用法
SimpleLoggerAdvisor loggerAdvisor = new SimpleLoggerAdvisor();

// 自定义格式化
SimpleLoggerAdvisor customLogger = SimpleLoggerAdvisor.builder()
    .requestToString(req -> "Request: " + req.prompt().getContents())
    .responseToString(resp -> "Response: " + resp.getResult())
    .order(Integer.MIN_VALUE)  // 最先执行
    .build();

chatClient
    .prompt("Hello")
    .advisors(customLogger)
    .call()
    .content();
```

### 5. SafeGuardAdvisor

内容安全检查，阻止敏感内容。

#### 源码解析

```java
public class SafeGuardAdvisor implements CallAdvisor, StreamAdvisor {
    
    private static final String DEFAULT_FAILURE_RESPONSE = 
        "I'm unable to respond to that due to sensitive content. " +
        "Could we rephrase or discuss something else?";
    
    private final List<String> sensitiveWords;
    private final String failureResponse;
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 检查是否包含敏感词
        boolean containsSensitiveWord = sensitiveWords.stream()
            .anyMatch(word -> 
                request.prompt().getContents().contains(word)
            );
        
        if (containsSensitiveWord) {
            // 返回失败响应，不调用AI模型
            return createFailureResponse(request);
        }
        
        // 通过检查，继续执行
        return chain.nextCall(request);
    }
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        boolean containsSensitiveWord = sensitiveWords.stream()
            .anyMatch(word -> 
                request.prompt().getContents().contains(word)
            );
        
        if (containsSensitiveWord) {
            return Flux.just(createFailureResponse(request));
        }
        
        return chain.nextStream(request);
    }
    
    private ChatClientResponse createFailureResponse(
            ChatClientRequest request) {
        return ChatClientResponse.builder()
            .chatResponse(ChatResponse.builder()
                .generations(List.of(new Generation(
                    new AssistantMessage(failureResponse)
                )))
                .build())
            .context(Map.copyOf(request.context()))
            .build();
    }
}
```

#### 使用示例

```java
SafeGuardAdvisor safeGuard = SafeGuardAdvisor.builder()
    .sensitiveWords(List.of(
        "password",
        "credit card",
        "social security"
    ))
    .failureResponse("请不要提供敏感信息")
    .order(Integer.MIN_VALUE)  // 最高优先级，最先检查
    .build();

chatClient
    .prompt("My password is 123456")
    .advisors(safeGuard)
    .call()
    .content();
// → "请不要提供敏感信息"
```

### 6. ChatModelCallAdvisor

实际调用AI模型的Advisor，总是最后执行。

```java
public final class ChatModelCallAdvisor implements CallAdvisor {
    
    private final ChatModel chatModel;
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 1. 增强输出格式指令
        ChatClientRequest formattedRequest = 
            augmentWithFormatInstructions(request);
        
        // 2. 实际调用AI模型
        ChatResponse chatResponse = 
            chatModel.call(formattedRequest.prompt());
        
        // 3. 构建响应
        return ChatClientResponse.builder()
            .chatResponse(chatResponse)
            .context(Map.copyOf(formattedRequest.context()))
            .build();
    }
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;  // 最低优先级
    }
}
```

---

## 自定义Advisor

### 1. 实现CallAdvisor（完全控制）

```java
public class CustomCallAdvisor implements CallAdvisor {
    
    private final SomeService service;
    private final int order;
    
    public CustomCallAdvisor(SomeService service, int order) {
        this.service = service;
        this.order = order;
    }
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // Before处理
        long startTime = System.currentTimeMillis();
        Map<String, Object> metrics = new HashMap<>();
        
        try {
            // 修改请求
            ChatClientRequest modifiedRequest = enhanceRequest(request);
            
            // 调用链
            ChatClientResponse response = chain.nextCall(modifiedRequest);
            
            // After处理
            long duration = System.currentTimeMillis() - startTime;
            metrics.put("duration", duration);
            
            // 修改响应
            return enhanceResponse(response, metrics);
            
        } catch (Exception e) {
            // 错误处理
            return handleError(request, e);
        }
    }
    
    private ChatClientRequest enhanceRequest(ChatClientRequest request) {
        // 自定义请求增强逻辑
        String enhancedText = service.enhance(
            request.prompt().getUserMessage().getText()
        );
        
        return request.mutate()
            .prompt(request.prompt()
                .augmentUserMessage(enhancedText))
            .build();
    }
    
    private ChatClientResponse enhanceResponse(
            ChatClientResponse response,
            Map<String, Object> metrics) {
        // 添加自定义元数据
        Map<String, Object> context = new HashMap<>(response.context());
        context.putAll(metrics);
        
        return ChatClientResponse.builder()
            .chatResponse(response.chatResponse())
            .context(context)
            .build();
    }
    
    @Override
    public String getName() {
        return "CustomCallAdvisor";
    }
    
    @Override
    public int getOrder() {
        return order;
    }
}
```

### 2. 实现BaseAdvisor（推荐）

```java
public class MyAdvisor implements BaseAdvisor {
    
    private final MyService service;
    private final int order;
    private final Scheduler scheduler;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        // 1. 从上下文获取参数
        String param = (String) request.context().get("myParam");
        
        // 2. 执行业务逻辑
        String enhancement = service.process(param);
        
        // 3. 修改请求
        Map<String, Object> newContext = new HashMap<>(request.context());
        newContext.put("enhancement", enhancement);
        
        String augmentedText = String.format(
            "%s\nAdditional Info: %s",
            request.prompt().getUserMessage().getText(),
            enhancement
        );
        
        return request.mutate()
            .prompt(request.prompt().augmentUserMessage(augmentedText))
            .context(newContext)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        // 1. 处理响应
        String processedContent = service.postProcess(
            response.chatResponse().getResult().getOutput().getText()
        );
        
        // 2. 保存数据
        service.saveToDatabase(processedContent);
        
        // 3. 修改响应（可选）
        return response;
    }
    
    @Override
    public Scheduler getScheduler() {
        return scheduler;
    }
    
    @Override
    public int getOrder() {
        return order;
    }
    
    @Override
    public String getName() {
        return "MyAdvisor";
    }
}
```

### 3. 实现StreamAdvisor

```java
public class StreamProcessingAdvisor implements StreamAdvisor {
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        // Before: 修改请求
        ChatClientRequest modifiedRequest = modifyRequest(request);
        
        // 获取响应流
        Flux<ChatClientResponse> responseFlux = 
            chain.nextStream(modifiedRequest);
        
        // After: 处理流中的每个元素
        return responseFlux
            .doOnNext(response -> {
                // 对每个响应片段进行处理
                logStreamChunk(response);
            })
            .map(response -> {
                // 转换响应
                return transformResponse(response);
            })
            .doOnComplete(() -> {
                // 流完成时的处理
                onStreamComplete();
            })
            .doOnError(error -> {
                // 错误处理
                handleStreamError(error);
            });
    }
    
    @Override
    public int getOrder() {
        return 0;
    }
    
    @Override
    public String getName() {
        return "StreamProcessingAdvisor";
    }
}
```

---

## 最佳实践

### 1. Advisor设计原则

#### 单一职责原则

```java
// ❌ 不好：一个Advisor做太多事
public class MegaAdvisor implements BaseAdvisor {
    @Override
    public ChatClientRequest before(ChatClientRequest request, ...) {
        // 日志记录
        logger.info("Request: {}", request);
        
        // 安全检查
        checkSecurity(request);
        
        // 添加记忆
        addMemory(request);
        
        // RAG增强
        enhanceWithRAG(request);
        
        return request;
    }
}

// ✅ 好：每个Advisor只做一件事
LoggerAdvisor loggerAdvisor = new LoggerAdvisor();
SecurityAdvisor securityAdvisor = new SecurityAdvisor();
MemoryAdvisor memoryAdvisor = new MemoryAdvisor();
RAGAdvisor ragAdvisor = new RAGAdvisor();
```

#### 明确的Order设置

```java
public class AdvisorOrders {
    // 安全检查应该最先执行
    public static final int SECURITY = Integer.MIN_VALUE;
    
    // 日志记录在安全检查之后
    public static final int LOGGING = -1000;
    
    // 记忆管理
    public static final int MEMORY = 
        Advisor.DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER;
    
    // RAG增强在记忆之后
    public static final int RAG = 
        Advisor.DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER + 100;
    
    // 业务逻辑
    public static final int BUSINESS = 0;
}
```

### 2. 上下文传递

使用`ChatClientRequest.context()`和`ChatClientResponse.context()`传递数据。

```java
public class ContextPassingAdvisor implements BaseAdvisor {
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        // 从上下文读取数据
        String userId = (String) request.context().get("userId");
        
        // 添加数据到上下文
        Map<String, Object> newContext = new HashMap<>(request.context());
        newContext.put("timestamp", System.currentTimeMillis());
        newContext.put("requestId", UUID.randomUUID().toString());
        
        return request.mutate()
            .context(newContext)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        // 从上下文读取数据
        long timestamp = (long) response.context().get("timestamp");
        String requestId = (String) response.context().get("requestId");
        
        // 计算处理时间
        long duration = System.currentTimeMillis() - timestamp;
        
        // 添加到响应上下文
        Map<String, Object> newContext = new HashMap<>(response.context());
        newContext.put("duration", duration);
        
        return ChatClientResponse.builder()
            .chatResponse(response.chatResponse())
            .context(newContext)
            .build();
    }
}
```

### 3. 异常处理

```java
public class ResilientAdvisor implements CallAdvisor {
    
    private final RetryTemplate retryTemplate;
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        try {
            // 使用重试机制
            return retryTemplate.execute(context -> {
                try {
                    return chain.nextCall(request);
                } catch (Exception e) {
                    // 记录错误
                    logger.error("Error in advisor chain", e);
                    
                    // 决定是否重试
                    if (isRetryable(e)) {
                        throw e;  // 重试
                    } else {
                        // 返回降级响应
                        return createFallbackResponse(request, e);
                    }
                }
            });
            
        } catch (Exception e) {
            // 最终失败，返回错误响应
            return createErrorResponse(request, e);
        }
    }
    
    private boolean isRetryable(Exception e) {
        // 判断异常是否可重试
        return e instanceof TimeoutException ||
               e instanceof IOException;
    }
    
    private ChatClientResponse createFallbackResponse(
            ChatClientRequest request,
            Exception e) {
        // 创建降级响应
        return ChatClientResponse.builder()
            .chatResponse(ChatResponse.builder()
                .generations(List.of(new Generation(
                    new AssistantMessage(
                        "抱歉，服务暂时不可用，请稍后重试。"
                    )
                )))
                .build())
            .context(Map.of("error", e.getMessage()))
            .build();
    }
}
```

### 4. 性能优化

```java
public class CachedAdvisor implements BaseAdvisor {
    
    private final Cache<String, String> cache;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        String userText = request.prompt().getUserMessage().getText();
        String cacheKey = generateCacheKey(userText);
        
        // 尝试从缓存获取增强内容
        String cachedEnhancement = cache.getIfPresent(cacheKey);
        
        if (cachedEnhancement != null) {
            // 使用缓存的内容
            return request.mutate()
                .prompt(request.prompt()
                    .augmentUserMessage(cachedEnhancement))
                .context(Map.of("cache_hit", true))
                .build();
        }
        
        // 缓存未命中，继续处理
        String enhancement = performExpensiveOperation(userText);
        cache.put(cacheKey, enhancement);
        
        return request.mutate()
            .prompt(request.prompt()
                .augmentUserMessage(enhancement))
            .context(Map.of("cache_hit", false))
            .build();
    }
}
```

### 5. 测试Advisor

```java
@ExtendWith(MockitoExtension.class)
class MyAdvisorTest {
    
    @Mock
    private CallAdvisorChain chain;
    
    @Mock
    private MyService service;
    
    private MyAdvisor advisor;
    
    @BeforeEach
    void setUp() {
        advisor = new MyAdvisor(service);
    }
    
    @Test
    void testBeforeProcessing() {
        // 准备测试数据
        ChatClientRequest request = ChatClientRequest.builder()
            .prompt(new Prompt("test"))
            .context(Map.of("key", "value"))
            .build();
        
        given(service.process(any()))
            .willReturn("enhanced");
        
        given(chain.nextCall(any()))
            .willReturn(mockResponse());
        
        // 执行
        ChatClientResponse response = advisor.adviseCall(request, chain);
        
        // 验证
        ArgumentCaptor<ChatClientRequest> captor = 
            ArgumentCaptor.forClass(ChatClientRequest.class);
        verify(chain).nextCall(captor.capture());
        
        ChatClientRequest capturedRequest = captor.getValue();
        assertThat(capturedRequest.prompt().getUserMessage().getText())
            .contains("enhanced");
        assertThat(capturedRequest.context())
            .containsKey("enhancement");
    }
    
    @Test
    void testAfterProcessing() {
        // 类似的测试after逻辑
    }
    
    @Test
    void testErrorHandling() {
        // 测试异常情况
        given(chain.nextCall(any()))
            .willThrow(new RuntimeException("Test error"));
        
        assertThatThrownBy(() -> advisor.adviseCall(mockRequest(), chain))
            .isInstanceOf(RuntimeException.class);
    }
}
```

---

## 实战示例

### 示例1: 审计日志Advisor

完整的审计日志实现，记录所有请求和响应。

```java
@Component
public class AuditLogAdvisor implements BaseAdvisor {
    
    private final AuditLogService auditLogService;
    private final int order;
    
    public AuditLogAdvisor(AuditLogService auditLogService) {
        this.auditLogService = auditLogService;
        this.order = Integer.MIN_VALUE + 100;  // 高优先级
    }
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        // 创建审计日志
        AuditLog log = AuditLog.builder()
            .requestId(UUID.randomUUID().toString())
            .timestamp(Instant.now())
            .userId(extractUserId(request))
            .promptContent(request.prompt().getContents())
            .metadata(request.context())
            .build();
        
        // 保存到数据库
        auditLogService.save(log);
        
        // 将审计日志ID添加到上下文
        Map<String, Object> context = new HashMap<>(request.context());
        context.put("auditLogId", log.getId());
        
        return request.mutate()
            .context(context)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        // 获取审计日志ID
        String auditLogId = (String) response.context().get("auditLogId");
        
        // 更新审计日志
        auditLogService.updateResponse(
            auditLogId,
            response.chatResponse().getResult().getOutput().getText(),
            Instant.now()
        );
        
        return response;
    }
    
    @Override
    public int getOrder() {
        return order;
    }
    
    private String extractUserId(ChatClientRequest request) {
        return (String) request.context()
            .getOrDefault("userId", "anonymous");
    }
}
```

### 示例2: 速率限制Advisor

防止API滥用的速率限制。

```java
@Component
public class RateLimitAdvisor implements CallAdvisor {
    
    private final RateLimiter rateLimiter;
    private final int order;
    
    public RateLimitAdvisor(RateLimiter rateLimiter) {
        this.rateLimiter = rateLimiter;
        this.order = Integer.MIN_VALUE + 50;  // 非常高的优先级
    }
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        String userId = extractUserId(request);
        
        // 尝试获取许可
        if (!rateLimiter.tryAcquire(userId)) {
            // 超过速率限制
            return createRateLimitResponse(request);
        }
        
        // 继续执行
        return chain.nextCall(request);
    }
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        String userId = extractUserId(request);
        
        if (!rateLimiter.tryAcquire(userId)) {
            return Flux.just(createRateLimitResponse(request));
        }
        
        return chain.nextStream(request);
    }
    
    private ChatClientResponse createRateLimitResponse(
            ChatClientRequest request) {
        return ChatClientResponse.builder()
            .chatResponse(ChatResponse.builder()
                .generations(List.of(new Generation(
                    new AssistantMessage(
                        "您的请求过于频繁，请稍后再试。"
                    )
                )))
                .build())
            .context(Map.of(
                "rateLimited", true,
                "retryAfter", rateLimiter.getRetryAfterSeconds()
            ))
            .build();
    }
    
    @Override
    public int getOrder() {
        return order;
    }
    
    @Override
    public String getName() {
        return "RateLimitAdvisor";
    }
    
    private String extractUserId(ChatClientRequest request) {
        return (String) request.context()
            .getOrDefault("userId", "anonymous");
    }
}
```

### 示例3: 多语言翻译Advisor

自动翻译用户输入和AI响应。

```java
@Component
public class TranslationAdvisor implements BaseAdvisor {
    
    private final TranslationService translationService;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        String userText = request.prompt().getUserMessage().getText();
        String targetLanguage = (String) request.context()
            .getOrDefault("targetLanguage", "en");
        
        // 检测源语言
        String sourceLanguage = translationService.detectLanguage(userText);
        
        // 如果不是英语，翻译为英语
        String translatedText = userText;
        if (!"en".equals(sourceLanguage)) {
            translatedText = translationService.translate(
                userText,
                sourceLanguage,
                "en"
            );
        }
        
        // 保存原始语言和文本
        Map<String, Object> context = new HashMap<>(request.context());
        context.put("sourceLanguage", sourceLanguage);
        context.put("originalText", userText);
        
        return request.mutate()
            .prompt(request.prompt().augmentUserMessage(translatedText))
            .context(context)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        String sourceLanguage = (String) response.context()
            .get("sourceLanguage");
        
        // 如果原始语言不是英语，翻译回原语言
        if (!"en".equals(sourceLanguage)) {
            String aiResponse = response.chatResponse()
                .getResult()
                .getOutput()
                .getText();
            
            String translatedResponse = translationService.translate(
                aiResponse,
                "en",
                sourceLanguage
            );
            
            // 创建新的响应
            ChatResponse newChatResponse = ChatResponse.builder()
                .from(response.chatResponse())
                .generations(List.of(new Generation(
                    new AssistantMessage(translatedResponse)
                )))
                .build();
            
            return ChatClientResponse.builder()
                .chatResponse(newChatResponse)
                .context(response.context())
                .build();
        }
        
        return response;
    }
    
    @Override
    public int getOrder() {
        return 100;  // 相对较低的优先级
    }
}
```

### 示例4: Token计数和成本追踪Advisor

跟踪Token使用和API成本。

```java
@Component
public class TokenCostAdvisor implements BaseAdvisor {
    
    private final TokenCounter tokenCounter;
    private final CostCalculator costCalculator;
    private final UsageRepository usageRepository;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        // 计算输入Token数
        String promptText = request.prompt().getContents();
        int inputTokens = tokenCounter.countTokens(promptText);
        
        // 添加到上下文
        Map<String, Object> context = new HashMap<>(request.context());
        context.put("inputTokens", inputTokens);
        context.put("startTime", System.currentTimeMillis());
        
        return request.mutate()
            .context(context)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        // 获取输入Token数
        int inputTokens = (int) response.context().get("inputTokens");
        long startTime = (long) response.context().get("startTime");
        
        // 计算输出Token数
        String responseText = response.chatResponse()
            .getResult()
            .getOutput()
            .getText();
        int outputTokens = tokenCounter.countTokens(responseText);
        
        // 计算总成本
        String model = response.chatResponse()
            .getMetadata()
            .getModel();
        
        double cost = costCalculator.calculate(
            model,
            inputTokens,
            outputTokens
        );
        
        // 保存使用记录
        UsageRecord record = UsageRecord.builder()
            .userId((String) response.context().get("userId"))
            .model(model)
            .inputTokens(inputTokens)
            .outputTokens(outputTokens)
            .totalTokens(inputTokens + outputTokens)
            .cost(cost)
            .duration(System.currentTimeMillis() - startTime)
            .timestamp(Instant.now())
            .build();
        
        usageRepository.save(record);
        
        // 添加到响应上下文
        Map<String, Object> context = new HashMap<>(response.context());
        context.put("outputTokens", outputTokens);
        context.put("totalTokens", inputTokens + outputTokens);
        context.put("cost", cost);
        
        return ChatClientResponse.builder()
            .chatResponse(response.chatResponse())
            .context(context)
            .build();
    }
    
    @Override
    public int getOrder() {
        return Integer.MAX_VALUE - 1;  // 低优先级，最后执行
    }
}
```

### 示例5: 智能缓存Advisor

基于语义相似度的智能缓存。

```java
@Component
public class SemanticCacheAdvisor implements BaseAdvisor {
    
    private final EmbeddingModel embeddingModel;
    private final VectorStore cacheStore;
    private final double similarityThreshold = 0.95;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain chain) {
        
        String userText = request.prompt().getUserMessage().getText();
        
        // 生成查询的向量
        List<Double> queryEmbedding = embeddingModel.embed(userText);
        
        // 在缓存中搜索相似的查询
        SearchRequest searchRequest = SearchRequest.builder()
            .query(userText)
            .topK(1)
            .similarityThreshold(similarityThreshold)
            .build();
        
        List<Document> similar = cacheStore.similaritySearch(searchRequest);
        
        if (!similar.isEmpty()) {
            // 找到缓存的响应
            Document cached = similar.get(0);
            String cachedResponse = 
                (String) cached.getMetadata().get("response");
            
            // 添加缓存命中标记
            Map<String, Object> context = 
                new HashMap<>(request.context());
            context.put("cacheHit", true);
            context.put("cachedResponse", cachedResponse);
            
            return request.mutate()
                .context(context)
                .build();
        }
        
        // 缓存未命中
        Map<String, Object> context = new HashMap<>(request.context());
        context.put("cacheHit", false);
        
        return request.mutate()
            .context(context)
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain chain) {
        
        boolean cacheHit = (boolean) response.context()
            .getOrDefault("cacheHit", false);
        
        if (cacheHit) {
            // 返回缓存的响应
            String cachedResponse = (String) response.context()
                .get("cachedResponse");
            
            ChatResponse chatResponse = ChatResponse.builder()
                .generations(List.of(new Generation(
                    new AssistantMessage(cachedResponse)
                )))
                .build();
            
            return ChatClientResponse.builder()
                .chatResponse(chatResponse)
                .context(response.context())
                .build();
        } else {
            // 保存到缓存
            saveToCacheSave(response);
            return response;
        }
    }
    
    private void saveToCache(ChatClientResponse response) {
        // 将查询和响应保存到向量存储
        String query = response.context().get("originalQuery").toString();
        String aiResponse = response.chatResponse()
            .getResult()
            .getOutput()
            .getText();
        
        Document cacheDoc = new Document(
            query,
            Map.of(
                "response", aiResponse,
                "timestamp", Instant.now().toString()
            )
        );
        
        cacheStore.add(List.of(cacheDoc));
    }
    
    @Override
    public int getOrder() {
        return -100;  // 高优先级，尽早检查缓存
    }
}
```

---

## 总结

### Advisor机制的核心优势

1. **模块化**: 每个Advisor负责一个独立的功能
2. **可组合**: 多个Advisor可以灵活组合
3. **有序执行**: 通过Order控制执行顺序
4. **请求/响应拦截**: 在AI调用前后都可以处理
5. **上下文传递**: 通过context在Advisor间传递数据
6. **支持同步和流式**: 统一的API支持两种调用方式

### 关键接口

- **Advisor**: 根接口，定义名称和顺序
- **CallAdvisor**: 同步调用的Advisor
- **StreamAdvisor**: 流式调用的Advisor
- **BaseAdvisor**: 便利基类，简化开发

### 内置实现

- **MessageChatMemoryAdvisor**: 消息级别的对话记忆
- **PromptChatMemoryAdvisor**: 提示词级别的对话记忆
- **QuestionAnswerAdvisor**: RAG文档检索
- **SimpleLoggerAdvisor**: 日志记录
- **SafeGuardAdvisor**: 内容安全检查
- **ChatModelCallAdvisor**: 实际AI模型调用

### 最佳实践

1. 遵循单一职责原则
2. 明确设置Order
3. 使用context传递数据
4. 妥善处理异常
5. 考虑性能优化（缓存等）
6. 编写单元测试

通过Advisor机制，Spring AI实现了真正的横切关注点分离，让开发者能够轻松扩展和定制AI应用的行为。

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

