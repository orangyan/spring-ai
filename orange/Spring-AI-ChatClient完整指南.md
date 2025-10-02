# Spring AI ChatClient 完整指南

## 📋 目录
- [概述](#概述)
- [创建ChatClient](#创建chatclient)
- [Fluent API详解](#fluent-api详解)
- [响应处理](#响应处理)
- [高级功能](#高级功能)
- [Advisor集成](#advisor集成)
- [实战场景](#实战场景)
- [性能优化](#性能优化)
- [最佳实践](#最佳实践)

---

## 概述

`ChatClient` 是Spring AI提供的高级流式API，用于与AI模型进行交互。它在`ChatModel`的基础上，提供了更优雅、更强大的编程体验。

### 核心优势

1. **流式API**: 链式调用，代码更简洁优雅
2. **类型安全**: 强类型支持，减少错误
3. **结构化输出**: 自动将AI响应转换为Java对象
4. **Advisor机制**: 内置拦截器链，支持内存管理、RAG等
5. **多模态支持**: 图片、音频等多种媒体类型
6. **模板渲染**: 内置模板引擎支持
7. **可观测性**: 集成Micrometer观测
8. **默认配置**: 支持全局默认配置

### 架构层次

```
ChatClient (高级API)
    ↓
Advisor Chain (拦截器链)
    ↓
ChatModel (底层模型接口)
    ↓
AI Provider (OpenAI、Ollama等)
```

### ChatClient vs ChatModel

| 特性 | ChatModel | ChatClient |
|------|-----------|-----------|
| API风格 | 命令式 | 流式/声明式 |
| 配置方式 | 每次调用传入 | 支持默认配置 |
| 消息构建 | 手动创建Message对象 | Fluent API构建 |
| 结构化输出 | 手动解析JSON | 自动转换为对象 |
| 模板支持 | 需手动使用PromptTemplate | 内置模板渲染 |
| Advisor | 不支持 | 内置支持 |
| 多模态 | 需手动构建UserMessage | Fluent API支持 |
| 适用场景 | 底层控制、集成开发 | 应用开发、快速迭代 |

---

## 创建ChatClient

### 1. 使用Spring Boot自动配置

在Spring Boot环境中，最简单的方式是注入自动配置的`ChatClient.Builder`。

```java
@RestController
public class ChatController {
    
    private final ChatClient chatClient;
    
    // 注入Builder，然后构建ChatClient
    public ChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }
    
    @GetMapping("/chat")
    public String chat(@RequestParam String message) {
        return chatClient
            .prompt(message)
            .call()
            .content();
    }
}
```

#### 配置文件

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
```

### 2. 手动创建（单个模型）

```java
@Configuration
public class ChatClientConfig {
    
    @Bean
    public ChatClient chatClient(ChatModel chatModel) {
        // 方式1: 最简单的创建
        return ChatClient.create(chatModel);
        
        // 方式2: 使用Builder，设置默认配置
        return ChatClient.builder(chatModel)
            .defaultSystem("你是一个专业的Java开发助手")
            .defaultOptions(ChatOptions.builder()
                .temperature(0.7)
                .maxTokens(1000)
                .build())
            .build();
    }
}
```

### 3. 多模型配置

当需要使用多个AI模型时，需要禁用自动配置。

```yaml
spring:
  ai:
    chat:
      client:
        enabled: false  # 禁用自动配置
```

```java
@Configuration
public class MultiChatClientConfig {
    
    // GPT-4 模型（复杂任务）
    @Bean
    public ChatClient gpt4ChatClient(
            @Qualifier("gpt4Model") ChatModel gpt4Model) {
        return ChatClient.builder(gpt4Model)
            .defaultSystem("你是一个专业的技术顾问，擅长复杂问题分析")
            .defaultOptions(ChatOptions.builder()
                .model("gpt-4")
                .temperature(0.3)  // 低温度，更准确
                .maxTokens(2000)
                .build())
            .build();
    }
    
    // GPT-3.5 模型（简单任务）
    @Bean
    public ChatClient gpt35ChatClient(
            @Qualifier("gpt35Model") ChatModel gpt35Model) {
        return ChatClient.builder(gpt35Model)
            .defaultSystem("你是一个友好的助手")
            .defaultOptions(ChatOptions.builder()
                .model("gpt-3.5-turbo")
                .temperature(0.7)
                .maxTokens(500)
                .build())
            .build();
    }
    
    // Ollama 本地模型（离线场景）
    @Bean
    public ChatClient ollamaChatClient(
            @Qualifier("ollamaModel") ChatModel ollamaModel) {
        return ChatClient.builder(ollamaModel)
            .defaultSystem("你是一个本地运行的AI助手")
            .build();
    }
}
```

### 4. Builder详细配置

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    // 默认系统消息
    .defaultSystem("你是一个专业的Java开发助手")
    .defaultSystem(systemSpecConsumer -> systemSpecConsumer
        .text("你是一个{role}")
        .param("role", "Java开发助手")
    )
    
    // 默认用户消息
    .defaultUser("请用中文回答")
    
    // 默认配置
    .defaultOptions(ChatOptions.builder()
        .temperature(0.7)
        .maxTokens(1000)
        .build())
    
    // 默认Advisors
    .defaultAdvisors(
        new MessageChatMemoryAdvisor(chatMemory),
        new SimpleLoggerAdvisor()
    )
    
    // 默认工具
    .defaultTools(weatherService)  // 自动注册@Tool方法
    .defaultToolCallbacks(weatherToolCallback)  // 手动注册工具
    .defaultToolContext(Map.of("userId", "12345"))  // 工具上下文
    
    // 默认模板渲染器
    .defaultTemplateRenderer(customRenderer)
    
    .build();
```

### 5. 使用Observation（可观测性）

```java
@Configuration
public class ObservableChatClientConfig {
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ObservationRegistry observationRegistry) {
        
        return ChatClient.create(
            chatModel,
            observationRegistry,
            new CustomChatClientObservationConvention()
        );
    }
}
```

---

## Fluent API详解

### 1. 基本使用流程

```java
ChatClient chatClient = ...;

String response = chatClient
    .prompt()              // 1. 开始构建请求
    .user("你好")          // 2. 设置用户消息
    .call()                // 3. 执行调用
    .content();            // 4. 获取响应内容
```

#### 流程图

```
chatClient
    ↓
prompt() / prompt(String) / prompt(Prompt)
    ↓
ChatClientRequestSpec (构建请求)
    ├─ system()          - 设置系统消息
    ├─ user()            - 设置用户消息
    ├─ messages()        - 直接添加消息
    ├─ options()         - 设置配置选项
    ├─ advisors()        - 设置Advisors
    ├─ tools()           - 设置工具
    └─ toolContext()     - 设置工具上下文
    ↓
call() / stream() (执行调用)
    ↓
CallResponseSpec / StreamResponseSpec (处理响应)
    ├─ content()         - 获取文本内容
    ├─ chatResponse()    - 获取完整响应
    ├─ entity()          - 转换为对象
    └─ responseEntity()  - 获取响应实体
```

### 2. prompt() 方法详解

#### 2.1 无参数 prompt()

最灵活的方式，从零开始构建。

```java
String response = chatClient
    .prompt()
    .system("你是一个专业的Java开发助手")
    .user("什么是Spring Boot?")
    .call()
    .content();
```

#### 2.2 prompt(String)

便捷方式，直接传入用户消息。

```java
String response = chatClient
    .prompt("什么是Spring Boot?")
    .call()
    .content();
```

#### 2.3 prompt(Prompt)

传入已构建的Prompt对象。

```java
Prompt prompt = new Prompt(
    List.of(
        new SystemMessage("你是一个助手"),
        new UserMessage("你好")
    ),
    ChatOptions.builder().temperature(0.7).build()
);

String response = chatClient
    .prompt(prompt)
    .call()
    .content();
```

### 3. 设置系统消息

#### 3.1 简单文本

```java
chatClient
    .prompt()
    .system("你是一个专业的Java开发助手")
    .user("什么是Spring?")
    .call()
    .content();
```

#### 3.2 使用Resource

```java
// 从classpath读取系统消息
chatClient
    .prompt()
    .system(new ClassPathResource("prompts/system.txt"))
    .user("什么是Spring?")
    .call()
    .content();
```

#### 3.3 使用Consumer（高级）

```java
chatClient
    .prompt()
    .system(spec -> spec
        .text("你是一个{role}，擅长{skill}")
        .param("role", "Java开发助手")
        .param("skill", "Spring框架")
        .metadata("priority", "high")
    )
    .user("什么是Spring Boot?")
    .call()
    .content();
```

### 4. 设置用户消息

#### 4.1 简单文本

```java
chatClient
    .prompt()
    .user("什么是Spring Boot?")
    .call()
    .content();
```

#### 4.2 使用模板

```java
chatClient
    .prompt()
    .user(spec -> spec
        .text("请介绍{topic}的{aspect}")
        .param("topic", "Spring Boot")
        .param("aspect", "核心特性")
    )
    .call()
    .content();
```

#### 4.3 多模态（图片+文本）

```java
// 方式1: 使用Media对象
chatClient
    .prompt()
    .user(spec -> spec
        .text("这张图片里有什么？")
        .media(new Media(
            MimeTypeUtils.IMAGE_PNG,
            new ClassPathResource("images/photo.png")
        ))
    )
    .call()
    .content();

// 方式2: 使用URL
chatClient
    .prompt()
    .user(spec -> spec
        .text("分析这张图片")
        .media(
            MimeTypeUtils.IMAGE_JPEG,
            new URL("https://example.com/image.jpg")
        )
    )
    .call()
    .content();

// 方式3: 多个图片
chatClient
    .prompt()
    .user(spec -> spec
        .text("比较这些图片")
        .media(
            new Media(MimeTypeUtils.IMAGE_PNG, image1),
            new Media(MimeTypeUtils.IMAGE_PNG, image2),
            new Media(MimeTypeUtils.IMAGE_PNG, image3)
        )
    )
    .call()
    .content();
```

#### 4.4 附加元数据

```java
chatClient
    .prompt()
    .user(spec -> spec
        .text("查询订单状态")
        .metadata("userId", "12345")
        .metadata("sessionId", "abc-123")
        .metadata(Map.of(
            "requestId", UUID.randomUUID().toString(),
            "timestamp", System.currentTimeMillis()
        ))
    )
    .call()
    .content();
```

### 5. 直接添加消息

```java
// 添加单个消息
chatClient
    .prompt()
    .messages(new UserMessage("你好"))
    .call()
    .content();

// 添加多个消息
chatClient
    .prompt()
    .messages(
        new SystemMessage("你是一个助手"),
        new UserMessage("我叫张三"),
        new AssistantMessage("你好，张三！"),
        new UserMessage("我叫什么名字？")
    )
    .call()
    .content();

// 添加消息列表（对话历史）
List<Message> history = loadChatHistory();
chatClient
    .prompt()
    .messages(history)
    .user("继续我们的对话")
    .call()
    .content();
```

### 6. 设置配置选项

```java
// 方式1: 使用通用ChatOptions
chatClient
    .prompt()
    .user("生成一段代码")
    .options(ChatOptions.builder()
        .temperature(0.2)  // 低温度，更准确
        .maxTokens(500)
        .build())
    .call()
    .content();

// 方式2: 使用特定模型的Options
OpenAiChatOptions openAiOptions = OpenAiChatOptions.builder()
    .model("gpt-4")
    .temperature(0.7)
    .maxTokens(1000)
    .responseFormat(ResponseFormat.builder()
        .type("json_object")
        .build())
    .seed(12345L)  // 确定性输出
    .build();

chatClient
    .prompt()
    .user("生成JSON格式的数据")
    .options(openAiOptions)
    .call()
    .content();
```

### 7. 设置Advisors

```java
// 方式1: 直接传入Advisor实例
chatClient
    .prompt()
    .advisors(
        new MessageChatMemoryAdvisor(chatMemory),
        new SimpleLoggerAdvisor()
    )
    .user("你好")
    .call()
    .content();

// 方式2: 使用Consumer配置
chatClient
    .prompt()
    .advisors(spec -> spec
        .advisors(
            new MessageChatMemoryAdvisor(chatMemory),
            new QuestionAnswerAdvisor(vectorStore)
        )
        .param("advisorKey", "advisorValue")
    )
    .user("查询文档内容")
    .call()
    .content();

// 方式3: 传入List
List<Advisor> advisors = List.of(
    new MessageChatMemoryAdvisor(chatMemory),
    new SafeGuardAdvisor(sensitiveWords)
);

chatClient
    .prompt()
    .advisors(advisors)
    .user("敏感内容检测")
    .call()
    .content();
```

### 8. 设置工具（Function Calling）

#### 8.1 使用@Tool注解的服务

```java
@Service
public class WeatherService {
    
    @Tool(description = "获取指定城市的天气信息")
    public WeatherInfo getWeather(
            @ToolParam(description = "城市名称") String city) {
        // 实现获取天气逻辑
        return new WeatherInfo(city, 25, "晴天");
    }
}

// 使用工具
chatClient
    .prompt()
    .user("北京今天天气怎么样？")
    .tools(weatherService)  // 自动注册@Tool方法
    .call()
    .content();
```

#### 8.2 使用ToolCallback

```java
// 定义工具回调
ToolCallback weatherToolCallback = ToolCallback.builder()
    .name("get_weather")
    .description("获取天气信息")
    .inputTypeSchema(JsonSchemaGenerator.generate(WeatherRequest.class))
    .function((WeatherRequest request) -> {
        // 执行逻辑
        return new WeatherInfo(request.city(), 25, "晴天");
    })
    .build();

// 使用工具
chatClient
    .prompt()
    .user("上海今天天气如何？")
    .toolCallbacks(weatherToolCallback)
    .call()
    .content();
```

#### 8.3 指定工具名称

```java
// 只使用特定的工具
chatClient
    .prompt()
    .user("查询天气")
    .toolNames("get_weather", "get_forecast")  // 只启用这些工具
    .call()
    .content();
```

#### 8.4 工具上下文

```java
// 传递上下文信息给工具
chatClient
    .prompt()
    .user("查询我的订单")
    .tools(orderService)
    .toolContext(Map.of(
        "userId", "12345",
        "sessionId", "abc-123",
        "permissions", List.of("READ", "WRITE")
    ))
    .call()
    .content();

// 在工具中使用上下文
@Tool
public Order getOrder(
        String orderId,
        @ToolContext Map<String, Object> context) {
    String userId = (String) context.get("userId");
    // 使用userId验证权限
    return orderRepository.findByIdAndUserId(orderId, userId);
}
```

### 9. 模板渲染器

```java
// 使用默认的StringTemplate渲染器
chatClient
    .prompt()
    .user(spec -> spec
        .text("你好，{name}！今天是{date}")
        .param("name", "张三")
        .param("date", LocalDate.now())
    )
    .call()
    .content();

// 使用自定义渲染器
TemplateRenderer customRenderer = new TemplateRenderer() {
    @Override
    public String render(String template, Map<String, Object> model) {
        // 自定义模板渲染逻辑（如Thymeleaf、Freemarker等）
        return thymeleafEngine.process(template, model);
    }
};

chatClient
    .prompt()
    .templateRenderer(customRenderer)
    .user(spec -> spec
        .text("[[${greeting}]], [[${name}]]!")  // Thymeleaf语法
        .param("greeting", "你好")
        .param("name", "李四")
    )
    .call()
    .content();
```

---

## 响应处理

### 1. content() - 获取文本内容

最简单常用的方式，直接获取AI返回的文本。

```java
// 同步调用
String content = chatClient
    .prompt("什么是Spring Boot?")
    .call()
    .content();

System.out.println(content);
// 输出: "Spring Boot是一个快速开发框架..."

// 流式调用
Flux<String> contentStream = chatClient
    .prompt("写一篇文章")
    .stream()
    .content();

contentStream.subscribe(
    chunk -> System.out.print(chunk),  // 逐块打印
    error -> System.err.println("Error: " + error),
    () -> System.out.println("\n[完成]")
);
```

### 2. chatResponse() - 获取完整响应

获取包含元数据的完整响应对象。

```java
// 同步调用
ChatResponse chatResponse = chatClient
    .prompt("你好")
    .call()
    .chatResponse();

// 访问响应内容
String content = chatResponse.getResult()
    .getOutput()
    .getText();

// 访问元数据
ChatResponseMetadata metadata = chatResponse.getMetadata();
Usage usage = metadata.getUsage();

System.out.println("Prompt Tokens: " + usage.getPromptTokens());
System.out.println("Completion Tokens: " + usage.getCompletionTokens());
System.out.println("Total Tokens: " + usage.getTotalTokens());
System.out.println("Model: " + metadata.getModel());
System.out.println("Finish Reason: " + metadata.getFinishReason());

// 流式调用
Flux<ChatResponse> responseStream = chatClient
    .prompt("写一首诗")
    .stream()
    .chatResponse();

responseStream.subscribe(
    response -> {
        String chunk = response.getResult().getOutput().getText();
        System.out.print(chunk);
    }
);
```

### 3. entity() - 结构化输出

自动将AI响应转换为Java对象（最强大的功能之一）。

#### 3.1 使用Class

```java
// 定义数据类
record Person(String name, int age, String occupation) {}

// 自动转换
Person person = chatClient
    .prompt("提取人物信息：张三，35岁，软件工程师")
    .call()
    .entity(Person.class);

System.out.println(person.name());  // "张三"
System.out.println(person.age());   // 35
System.out.println(person.occupation());  // "软件工程师"
```

#### 3.2 使用ParameterizedTypeReference（泛型类型）

```java
// 复杂泛型类型
record ActorFilms(String actor, List<String> movies) {}

ActorFilms actorFilms = chatClient
    .prompt("列出成龙的5部电影")
    .call()
    .entity(new ParameterizedTypeReference<ActorFilms>() {});

System.out.println(actorFilms.actor());  // "成龙"
actorFilms.movies().forEach(System.out::println);
// 醉拳
// 警察故事
// 红番区
// ...

// List类型
List<String> cities = chatClient
    .prompt("列出中国的5个一线城市")
    .call()
    .entity(new ParameterizedTypeReference<List<String>>() {});

cities.forEach(System.out::println);
// 北京、上海、广州、深圳、杭州

// Map类型
Map<String, Integer> population = chatClient
    .prompt("返回中国一线城市的人口数量（万人）")
    .call()
    .entity(new ParameterizedTypeReference<Map<String, Integer>>() {});

population.forEach((city, pop) -> 
    System.out.println(city + ": " + pop + "万人")
);
```

#### 3.3 使用StructuredOutputConverter（自定义转换器）

```java
// 自定义转换器
StructuredOutputConverter<Person> converter = 
    new BeanOutputConverter<>(Person.class);

Person person = chatClient
    .prompt("提取：李四，28岁，设计师")
    .call()
    .entity(converter);
```

#### 3.4 JSON模式输出

某些模型（如OpenAI）支持强制JSON输出。

```java
record BookReview(
    String title,
    String author,
    int rating,
    String summary
) {}

BookReview review = chatClient
    .prompt("评价《三体》这本书")
    .options(OpenAiChatOptions.builder()
        .responseFormat(ResponseFormat.builder()
            .type("json_object")
            .build())
        .build())
    .call()
    .entity(BookReview.class);

System.out.println(review.title());   // "三体"
System.out.println(review.rating());  // 5
```

### 4. chatClientResponse() - 获取ChatClient响应

获取包含上下文的响应对象。

```java
// 同步调用
ChatClientResponse clientResponse = chatClient
    .prompt("你好")
    .call()
    .chatClientResponse();

// 访问ChatResponse
ChatResponse chatResponse = clientResponse.chatResponse();
String content = chatResponse.getResult().getOutput().getText();

// 访问上下文（Advisor可能添加的数据）
Map<String, Object> context = clientResponse.context();
Object memoryData = context.get("memory");
Object ragData = context.get("retrievedDocuments");

// 流式调用
Flux<ChatClientResponse> responseStream = chatClient
    .prompt("写一篇文章")
    .stream()
    .chatClientResponse();

responseStream.subscribe(
    clientResponse -> {
        // 处理每个响应块
        ChatResponse chatResponse = clientResponse.chatResponse();
        if (chatResponse != null) {
            System.out.print(
                chatResponse.getResult().getOutput().getText()
            );
        }
    }
);
```

### 5. responseEntity() - 获取响应实体

同时获取ChatResponse和转换后的实体。

```java
record Summary(String title, List<String> keyPoints) {}

// 获取响应实体
ResponseEntity<ChatResponse, Summary> responseEntity = chatClient
    .prompt("总结这篇文章：...")
    .call()
    .responseEntity(Summary.class);

// 访问ChatResponse
ChatResponse chatResponse = responseEntity.chatResponse();
Usage usage = chatResponse.getMetadata().getUsage();
System.out.println("Tokens: " + usage.getTotalTokens());

// 访问实体
Summary summary = responseEntity.entity();
System.out.println("Title: " + summary.title());
summary.keyPoints().forEach(System.out::println);
```

### 6. 流式响应高级用法

#### 6.1 实时打印

```java
chatClient
    .prompt("写一首诗")
    .stream()
    .content()
    .subscribe(
        chunk -> System.out.print(chunk),  // 打印每个块
        error -> System.err.println("Error: " + error),
        () -> System.out.println("\n[完成]")
    );
```

#### 6.2 累积内容

```java
// 累积所有块
String fullContent = chatClient
    .prompt("写一篇文章")
    .stream()
    .content()
    .collectList()
    .block()
    .stream()
    .collect(Collectors.joining());

System.out.println(fullContent);
```

#### 6.3 Server-Sent Events (SSE)

```java
@RestController
public class StreamController {
    
    private final ChatClient chatClient;
    
    @GetMapping(value = "/stream", 
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> stream(
            @RequestParam String message) {
        
        return chatClient
            .prompt(message)
            .stream()
            .content()
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

#### 6.4 WebFlux响应

```java
@RestController
public class ReactiveController {
    
    private final ChatClient chatClient;
    
    @GetMapping(value = "/chat/stream",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String message) {
        return chatClient
            .prompt(message)
            .stream()
            .content()
            .onErrorResume(error -> {
                logger.error("Stream error", error);
                return Flux.just("[Error occurred]");
            })
            .timeout(Duration.ofSeconds(30));
    }
}
```

---

## 高级功能

### 1. 默认配置继承

ChatClient支持多层配置，优先级从高到低：

```
请求级配置 > ChatClient级配置 > ChatModel级配置
```

```java
// 1. ChatModel级别（最低优先级）
ChatModel chatModel = new OpenAiChatModel(...);
chatModel.getDefaultOptions();  // 模型默认配置

// 2. ChatClient级别（中等优先级）
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultSystem("你是一个助手")
    .defaultOptions(ChatOptions.builder()
        .temperature(0.7)
        .build())
    .build();

// 3. 请求级别（最高优先级）
String response = chatClient
    .prompt()
    .system("你是一个专家")  // 覆盖默认system
    .user("你好")
    .options(ChatOptions.builder()
        .temperature(0.5)  // 覆盖默认temperature
        .build())
    .call()
    .content();
```

### 2. mutate() - 动态调整

在已有ChatClient基础上创建新的实例。

```java
// 基础ChatClient
ChatClient baseChatClient = ChatClient.builder(chatModel)
    .defaultSystem("你是一个助手")
    .defaultOptions(ChatOptions.builder()
        .temperature(0.7)
        .build())
    .build();

// 派生ChatClient（用于代码生成）
ChatClient codeChatClient = baseChatClient
    .mutate()
    .defaultSystem("你是一个代码生成专家")
    .defaultOptions(ChatOptions.builder()
        .temperature(0.2)  // 低温度
        .maxTokens(2000)
        .build())
    .build();

// 派生ChatClient（用于创意写作）
ChatClient creativeChatClient = baseChatClient
    .mutate()
    .defaultSystem("你是一个创意写作助手")
    .defaultOptions(ChatOptions.builder()
        .temperature(0.9)  // 高温度
        .build())
    .build();

// 请求级别的mutate
ChatClientRequestSpec spec = chatClient
    .prompt()
    .user("原始请求");

// 基于原始请求创建新请求
ChatClientRequestSpec newSpec = spec
    .mutate()
    .defaultSystem("新的系统消息")
    .build();
```

### 3. 对话历史管理

#### 3.1 手动管理

```java
@Service
public class ConversationService {
    
    private final ChatClient chatClient;
    private final Map<String, List<Message>> conversations = 
        new ConcurrentHashMap<>();
    
    public String chat(String userId, String userMessage) {
        // 获取历史
        List<Message> history = conversations
            .computeIfAbsent(userId, k -> new ArrayList<>());
        
        // 添加用户消息
        UserMessage currentMessage = new UserMessage(userMessage);
        history.add(currentMessage);
        
        // 调用AI
        ChatResponse response = chatClient
            .prompt()
            .messages(history)
            .call()
            .chatResponse();
        
        // 保存AI响应
        AssistantMessage assistantMessage = 
            response.getResult().getOutput();
        history.add(assistantMessage);
        
        return assistantMessage.getText();
    }
    
    public void clearHistory(String userId) {
        conversations.remove(userId);
    }
}
```

#### 3.2 使用MessageChatMemoryAdvisor

```java
@Configuration
public class ChatMemoryConfig {
    
    @Bean
    public ChatMemory chatMemory() {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(new InMemoryChatMemoryRepository())
            .maxMessages(10)  // 保留最近10条消息
            .build();
    }
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ChatMemory chatMemory) {
        
        return ChatClient.builder(chatModel)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .promptTemplate("Chat history:\n{history}\n\n")
                    .build()
            )
            .build();
    }
}

// 使用（自动管理历史）
@Service
public class ChatService {
    
    private final ChatClient chatClient;
    
    public String chat(String userId, String message) {
        // Advisor自动处理历史
        return chatClient
            .prompt()
            .advisors(spec -> spec
                .param("conversationId", userId)  // 会话ID
            )
            .user(message)
            .call()
            .content();
    }
}
```

### 4. RAG (检索增强生成)

```java
@Configuration
public class RAGConfig {
    
    @Bean
    public QuestionAnswerAdvisor ragAdvisor(
            VectorStore vectorStore) {
        
        return QuestionAnswerAdvisor.builder()
            .vectorStore(vectorStore)
            .searchRequest(SearchRequest.defaults()
                .withTopK(5)
                .withSimilarityThreshold(0.7)
            )
            .userTextAdvise("Context:\n{context}\n\nQuestion: {question}")
            .build();
    }
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            QuestionAnswerAdvisor ragAdvisor) {
        
        return ChatClient.builder(chatModel)
            .defaultSystem("根据提供的上下文回答问题")
            .defaultAdvisors(ragAdvisor)
            .build();
    }
}

// 使用RAG
@Service
public class DocumentQAService {
    
    private final ChatClient chatClient;
    
    public String askQuestion(String question) {
        // Advisor自动从VectorStore检索相关文档
        return chatClient
            .prompt(question)
            .call()
            .content();
    }
}
```

### 5. 多模态处理

#### 5.1 图像分析

```java
@Service
public class ImageAnalysisService {
    
    private final ChatClient chatClient;
    
    public String analyzeImage(String imageUrl, String question) {
        return chatClient
            .prompt()
            .user(spec -> spec
                .text(question)
                .media(
                    MimeTypeUtils.IMAGE_JPEG,
                    new URL(imageUrl)
                )
            )
            .options(OpenAiChatOptions.builder()
                .model("gpt-4-vision-preview")
                .build())
            .call()
            .content();
    }
    
    public ImageDescription describeImage(MultipartFile imageFile) 
            throws IOException {
        
        return chatClient
            .prompt()
            .user(spec -> spec
                .text("详细描述这张图片")
                .media(new Media(
                    MimeTypeUtils.IMAGE_PNG,
                    imageFile.getResource()
                ))
            )
            .call()
            .entity(ImageDescription.class);
    }
    
    record ImageDescription(
        String mainSubject,
        List<String> objects,
        String scene,
        String mood
    ) {}
}
```

#### 5.2 音频处理

```java
@Service
public class AudioService {
    
    private final ChatClient chatClient;
    
    public String transcribeAudio(Resource audioFile) {
        return chatClient
            .prompt()
            .user(spec -> spec
                .text("转录这段音频")
                .media(new Media(
                    new MimeType("audio", "mp3"),
                    audioFile
                ))
            )
            .call()
            .content();
    }
}
```

---

## Advisor集成

Advisor是ChatClient最强大的功能之一，提供拦截器链机制。

### 1. 内置Advisors

#### 1.1 MessageChatMemoryAdvisor（对话记忆）

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(new InMemoryChatMemoryRepository())
    .maxMessages(10)
    .build();

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        MessageChatMemoryAdvisor.builder(chatMemory)
            .conversationIdProvider(request -> 
                (String) request.context().get("userId")
            )
            .promptTemplate("""
                Previous conversation:
                {history}
                
                Current request:
                {question}
                """)
            .build()
    )
    .build();

// 使用
String response = chatClient
    .prompt()
    .advisors(spec -> spec
        .param("userId", "user-123")  // 会话ID
    )
    .user("我叫张三")
    .call()
    .content();

String response2 = chatClient
    .prompt()
    .advisors(spec -> spec
        .param("userId", "user-123")
    )
    .user("我叫什么名字？")  // AI会记住"张三"
    .call()
    .content();
```

#### 1.2 QuestionAnswerAdvisor（RAG）

```java
QuestionAnswerAdvisor ragAdvisor = QuestionAnswerAdvisor.builder()
    .vectorStore(vectorStore)
    .searchRequest(SearchRequest.defaults()
        .withTopK(5)
        .withSimilarityThreshold(0.75)
        .withFilterExpression("type == 'documentation'")
    )
    .userTextAdvise("""
        Use the following context to answer the question:
        
        {context}
        
        Question: {question}
        """)
    .build();

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(ragAdvisor)
    .build();

// 使用
String answer = chatClient
    .prompt("什么是Spring Boot?")  // 自动检索相关文档
    .call()
    .content();
```

#### 1.3 SimpleLoggerAdvisor（日志记录）

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(new SimpleLoggerAdvisor())
    .build();

// 自动记录请求和响应
String response = chatClient
    .prompt("你好")
    .call()
    .content();
// 日志输出:
// Request: [SystemMessage: ..., UserMessage: 你好]
// Response: 你好！有什么我可以帮助你的吗？
```

#### 1.4 SafeGuardAdvisor（内容审核）

```java
SafeGuardAdvisor safeGuardAdvisor = SafeGuardAdvisor.builder()
    .sensitiveWords(Set.of("敏感词1", "敏感词2"))
    .blockOnSensitiveContent(true)
    .replacementText("[已屏蔽]")
    .build();

ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(safeGuardAdvisor)
    .build();

// 使用
try {
    String response = chatClient
        .prompt("包含敏感词的内容")
        .call()
        .content();
} catch (ContentBlockedException e) {
    System.out.println("内容被屏蔽: " + e.getMessage());
}
```

### 2. 自定义Advisor

#### 2.1 实现BaseAdvisor

```java
public class CustomLoggingAdvisor implements BaseAdvisor {
    
    private static final Logger logger = 
        LoggerFactory.getLogger(CustomLoggingAdvisor.class);
    
    @Override
    public String getName() {
        return "CustomLoggingAdvisor";
    }
    
    @Override
    public int getOrder() {
        return 0;  // 执行顺序
    }
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request,
            AdvisorChain advisorChain) {
        
        // 请求前拦截
        logger.info("Before request: {}", 
            request.prompt().getInstructions());
        
        // 可以修改请求
        Prompt modifiedPrompt = request.prompt()
            .augmentSystemMessage(systemMsg -> 
                SystemMessage.from(
                    systemMsg.getText() + "\n[Timestamp: " + 
                    Instant.now() + "]"
                )
            );
        
        return ChatClientRequest.builder()
            .prompt(modifiedPrompt)
            .context(request.context())
            .build();
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse response,
            AdvisorChain advisorChain) {
        
        // 响应后拦截
        if (response.chatResponse() != null) {
            String content = response.chatResponse()
                .getResult()
                .getOutput()
                .getText();
            
            logger.info("After response: {}", content);
            
            // 记录Token使用
            Usage usage = response.chatResponse()
                .getMetadata()
                .getUsage();
            
            logger.info("Tokens used: {}", usage.getTotalTokens());
        }
        
        return response;
    }
}
```

#### 2.2 只拦截同步调用

```java
public class SyncOnlyAdvisor implements CallAdvisor {
    
    @Override
    public String getName() {
        return "SyncOnlyAdvisor";
    }
    
    @Override
    public ChatClientResponse adviseCall(
            ChatClientRequest request,
            CallAdvisorChain chain) {
        
        // 只在同步调用时执行
        System.out.println("Sync call intercepted");
        
        // 调用链的下一个Advisor
        ChatClientResponse response = chain.nextAroundCall(request);
        
        // 后处理
        System.out.println("Sync call completed");
        
        return response;
    }
}
```

#### 2.3 只拦截流式调用

```java
public class StreamOnlyAdvisor implements StreamAdvisor {
    
    @Override
    public String getName() {
        return "StreamOnlyAdvisor";
    }
    
    @Override
    public Flux<ChatClientResponse> adviseStream(
            ChatClientRequest request,
            StreamAdvisorChain chain) {
        
        // 只在流式调用时执行
        return Flux.deferContextual(contextView -> {
            System.out.println("Stream call intercepted");
            
            // 调用链的下一个Advisor
            return chain.nextAroundStream(request)
                .doOnComplete(() -> 
                    System.out.println("Stream completed")
                )
                .doOnError(error -> 
                    System.err.println("Stream error: " + error)
                );
        });
    }
}
```

### 3. Advisor顺序控制

```java
public class HighPriorityAdvisor implements BaseAdvisor {
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // 最高优先级
    }
    
    // ... 其他方法
}

public class LowPriorityAdvisor implements BaseAdvisor {
    
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;  // 最低优先级
    }
    
    // ... 其他方法
}

// 使用
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(
        new LowPriorityAdvisor(),   // 后执行
        new HighPriorityAdvisor()   // 先执行（虽然写在后面）
    )
    .build();

// 执行顺序:
// HighPriorityAdvisor.before()
// LowPriorityAdvisor.before()
// ... AI调用 ...
// LowPriorityAdvisor.after()
// HighPriorityAdvisor.after()
```

---

## 实战场景

### 1. REST API集成

```java
@RestController
@RequestMapping("/api/chat")
public class ChatRestController {
    
    private final ChatClient chatClient;
    
    @PostMapping
    public ResponseEntity<ChatResponseDTO> chat(
            @RequestBody ChatRequestDTO request) {
        
        try {
            ChatResponse response = chatClient
                .prompt()
                .system(request.systemMessage())
                .user(request.userMessage())
                .options(ChatOptions.builder()
                    .temperature(request.temperature())
                    .maxTokens(request.maxTokens())
                    .build())
                .call()
                .chatResponse();
            
            return ResponseEntity.ok(
                new ChatResponseDTO(
                    response.getResult().getOutput().getText(),
                    response.getMetadata().getUsage().getTotalTokens()
                )
            );
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new ChatResponseDTO(
                    "Error: " + e.getMessage(),
                    0
                ));
        }
    }
    
    @GetMapping("/stream")
    public Flux<ServerSentEvent<String>> streamChat(
            @RequestParam String message) {
        
        return chatClient
            .prompt(message)
            .stream()
            .content()
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
    
    record ChatRequestDTO(
        String systemMessage,
        String userMessage,
        Double temperature,
        Integer maxTokens
    ) {}
    
    record ChatResponseDTO(
        String content,
        int totalTokens
    ) {}
}
```

### 2. 文档问答系统

```java
@Service
public class DocumentQAService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final DocumentReader documentReader;
    
    public void ingestDocuments(List<Resource> documents) {
        // 读取文档
        List<Document> docs = documents.stream()
            .flatMap(resource -> 
                documentReader.read(resource).stream()
            )
            .toList();
        
        // 存入向量数据库
        vectorStore.add(docs);
    }
    
    public String askQuestion(String question) {
        return chatClient
            .prompt(question)
            .advisors(
                QuestionAnswerAdvisor.builder()
                    .vectorStore(vectorStore)
                    .searchRequest(SearchRequest.defaults()
                        .withTopK(5)
                        .withSimilarityThreshold(0.7)
                    )
                    .build()
            )
            .call()
            .content();
    }
    
    public QuestionAnswer askWithSources(String question) {
        // 手动检索文档
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.query(question)
                .withTopK(5)
                .withSimilarityThreshold(0.7)
        );
        
        // 构建上下文
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        // 询问AI
        String answer = chatClient
            .prompt()
            .system("根据以下上下文回答问题")
            .user(spec -> spec
                .text("""
                    Context:
                    {context}
                    
                    Question: {question}
                    """)
                .param("context", context)
                .param("question", question)
            )
            .call()
            .content();
        
        // 返回答案和来源
        return new QuestionAnswer(
            answer,
            docs.stream()
                .map(doc -> doc.getMetadata().get("source"))
                .map(Object::toString)
                .toList()
        );
    }
    
    record QuestionAnswer(String answer, List<String> sources) {}
}
```

### 3. 对话式Agent

```java
@Service
public class ConversationalAgent {
    
    private final ChatClient chatClient;
    private final ChatMemory chatMemory;
    private final ToolCallbackProvider toolProvider;
    
    public ConversationalAgent(
            ChatModel chatModel,
            ChatMemory chatMemory,
            List<Object> tools) {
        
        this.chatMemory = chatMemory;
        this.toolProvider = new DefaultToolCallbackProvider(tools);
        
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("""
                你是一个智能助手，可以使用工具来帮助用户。
                当需要执行操作时，请使用可用的工具。
                """)
            .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory)
                    .build()
            )
            .defaultToolCallbacks(toolProvider)
            .build();
    }
    
    public String chat(String userId, String message) {
        return chatClient
            .prompt()
            .advisors(spec -> spec
                .param("conversationId", userId)
            )
            .user(message)
            .toolContext(Map.of("userId", userId))
            .call()
            .content();
    }
    
    public void resetConversation(String userId) {
        chatMemory.clear(userId);
    }
}

// 工具定义
@Component
public class AgentTools {
    
    @Tool(description = "获取当前时间")
    public String getCurrentTime() {
        return LocalDateTime.now().toString();
    }
    
    @Tool(description = "搜索信息")
    public SearchResult search(
            @ToolParam(description = "搜索查询") String query) {
        // 实现搜索逻辑
        return new SearchResult(query, List.of("结果1", "结果2"));
    }
    
    record SearchResult(String query, List<String> results) {}
}
```

### 4. 内容生成服务

```java
@Service
public class ContentGenerationService {
    
    private final ChatClient creativeChatClient;
    private final ChatClient factualChatClient;
    
    public ContentGenerationService(ChatModel chatModel) {
        // 创意写作客户端
        this.creativeChatClient = ChatClient.builder(chatModel)
            .defaultSystem("你是一个创意写作专家")
            .defaultOptions(ChatOptions.builder()
                .temperature(0.9)  // 高创造性
                .maxTokens(2000)
                .build())
            .build();
        
        // 事实性内容客户端
        this.factualChatClient = ChatClient.builder(chatModel)
            .defaultSystem("你是一个专业的技术写作专家")
            .defaultOptions(ChatOptions.builder()
                .temperature(0.3)  // 低温度，更准确
                .maxTokens(1500)
                .build())
            .build();
    }
    
    public String generateStory(String theme, String style) {
        return creativeChatClient
            .prompt(spec -> spec
                .text("写一个关于{theme}的{style}风格故事")
                .param("theme", theme)
                .param("style", style)
            )
            .call()
            .content();
    }
    
    public TechnicalArticle generateTechnicalArticle(String topic) {
        return factualChatClient
            .prompt(spec -> spec
                .text("写一篇关于{topic}的技术文章，包括标题、摘要和章节")
                .param("topic", topic)
            )
            .call()
            .entity(TechnicalArticle.class);
    }
    
    public Flux<String> generateLongFormContent(String prompt) {
        return creativeChatClient
            .prompt(prompt)
            .stream()
            .content();
    }
    
    record TechnicalArticle(
        String title,
        String summary,
        List<Section> sections
    ) {}
    
    record Section(
        String heading,
        String content
    ) {}
}
```

### 5. 数据提取服务

```java
@Service
public class DataExtractionService {
    
    private final ChatClient chatClient;
    
    public DataExtractionService(ChatModel chatModel) {
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("你是一个数据提取专家，请准确提取信息")
            .defaultOptions(OpenAiChatOptions.builder()
                .responseFormat(ResponseFormat.builder()
                    .type("json_object")
                    .build())
                .temperature(0.1)  // 低温度，更准确
                .build())
            .build();
    }
    
    public ContactInfo extractContactInfo(String text) {
        return chatClient
            .prompt(spec -> spec
                .text("从以下文本中提取联系信息：\n{text}")
                .param("text", text)
            )
            .call()
            .entity(ContactInfo.class);
    }
    
    public List<Product> extractProducts(String webpage) {
        return chatClient
            .prompt(spec -> spec
                .text("从以下网页内容中提取所有产品信息：\n{content}")
                .param("content", webpage)
            )
            .call()
            .entity(new ParameterizedTypeReference<List<Product>>() {});
    }
    
    public InvoiceData parseInvoice(String invoiceText) {
        return chatClient
            .prompt()
            .user(spec -> spec
                .text("""
                    Extract invoice information from the following text:
                    {invoice}
                    
                    Return JSON with fields: invoiceNumber, date, 
                    vendor, total, items
                    """)
                .param("invoice", invoiceText)
            )
            .call()
            .entity(InvoiceData.class);
    }
    
    record ContactInfo(
        String name,
        String email,
        String phone,
        String address
    ) {}
    
    record Product(
        String name,
        String description,
        Double price,
        String currency
    ) {}
    
    record InvoiceData(
        String invoiceNumber,
        String date,
        String vendor,
        Double total,
        List<InvoiceItem> items
    ) {}
    
    record InvoiceItem(
        String description,
        Integer quantity,
        Double unitPrice,
        Double total
    ) {}
}
```

---

## 性能优化

### 1. 批处理

```java
@Service
public class BatchProcessingService {
    
    private final ChatClient chatClient;
    
    public List<String> processBatch(List<String> inputs) {
        return inputs.parallelStream()
            .map(input -> {
                try {
                    return chatClient
                        .prompt(input)
                        .call()
                        .content();
                } catch (Exception e) {
                    logger.error("Error processing: {}", input, e);
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .toList();
    }
    
    public Flux<ProcessResult> processBatchAsync(List<String> inputs) {
        return Flux.fromIterable(inputs)
            .flatMap(input -> 
                Mono.fromCallable(() -> {
                    long start = System.currentTimeMillis();
                    String result = chatClient
                        .prompt(input)
                        .call()
                        .content();
                    long duration = System.currentTimeMillis() - start;
                    
                    return new ProcessResult(input, result, duration);
                })
                .subscribeOn(Schedulers.boundedElastic())
            )
            .onErrorContinue((error, obj) -> 
                logger.error("Failed: {}", obj, error)
            );
    }
    
    record ProcessResult(
        String input,
        String output,
        long durationMs
    ) {}
}
```

### 2. 缓存策略

```java
@Service
public class CachedChatService {
    
    private final ChatClient chatClient;
    private final Cache<String, String> responseCache;
    
    public CachedChatService(ChatClient chatClient) {
        this.chatClient = chatClient;
        this.responseCache = CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(1, TimeUnit.HOURS)
            .recordStats()
            .build();
    }
    
    public String chatWithCache(String message) {
        try {
            return responseCache.get(message, () -> 
                chatClient
                    .prompt(message)
                    .call()
                    .content()
            );
        } catch (ExecutionException e) {
            throw new RuntimeException(e);
        }
    }
    
    @Scheduled(fixedDelay = 60000)
    public void logCacheStats() {
        CacheStats stats = responseCache.stats();
        logger.info("Cache stats - Hits: {}, Misses: {}, Hit rate: {}",
            stats.hitCount(),
            stats.missCount(),
            stats.hitRate()
        );
    }
}
```

### 3. 异步处理

```java
@Service
public class AsyncChatService {
    
    private final ChatClient chatClient;
    private final Executor asyncExecutor;
    
    @Async
    public CompletableFuture<String> chatAsync(String message) {
        return CompletableFuture.supplyAsync(() -> 
            chatClient
                .prompt(message)
                .call()
                .content(),
            asyncExecutor
        );
    }
    
    public Flux<String> chatReactive(String message) {
        return Mono.fromCallable(() -> 
            chatClient
                .prompt(message)
                .call()
                .content()
        )
        .subscribeOn(Schedulers.boundedElastic())
        .flux();
    }
}
```

---

## 最佳实践

### 1. 错误处理

```java
@Service
public class RobustChatService {
    
    private final ChatClient chatClient;
    private final RetryTemplate retryTemplate;
    
    public String chatWithRetry(String message) {
        return retryTemplate.execute(context -> {
            try {
                return chatClient
                    .prompt(message)
                    .call()
                    .content();
            } catch (RateLimitException e) {
                logger.warn("Rate limit hit, retrying...");
                throw e;  // 触发重试
            } catch (TimeoutException e) {
                logger.warn("Timeout, retrying...");
                throw e;
            } catch (Exception e) {
                logger.error("Unrecoverable error", e);
                return "抱歉，服务暂时不可用";
            }
        });
    }
    
    public Mono<String> chatWithFallback(String message) {
        return Mono.fromCallable(() -> 
            chatClient
                .prompt(message)
                .call()
                .content()
        )
        .timeout(Duration.ofSeconds(30))
        .retry(3)
        .onErrorResume(error -> {
            logger.error("All retries failed", error);
            return Mono.just("抱歉，我现在无法回答");
        });
    }
}
```

### 2. 监控和日志

```java
@Component
public class ChatClientMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter successCounter;
    private final Counter errorCounter;
    private final Timer responseTimer;
    
    public ChatClientMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.successCounter = Counter.builder("chatclient.requests.success")
            .register(meterRegistry);
        this.errorCounter = Counter.builder("chatclient.requests.error")
            .register(meterRegistry);
        this.responseTimer = Timer.builder("chatclient.response.time")
            .register(meterRegistry);
    }
    
    public String chatWithMetrics(ChatClient chatClient, String message) {
        return responseTimer.record(() -> {
            try {
                String response = chatClient
                    .prompt(message)
                    .call()
                    .content();
                successCounter.increment();
                return response;
            } catch (Exception e) {
                errorCounter.increment();
                throw e;
            }
        });
    }
}
```

### 3. 配置管理

```java
@ConfigurationProperties(prefix = "app.chat")
@Data
public class ChatProperties {
    
    private String defaultModel = "gpt-4";
    private Double defaultTemperature = 0.7;
    private Integer defaultMaxTokens = 1000;
    private String defaultSystemMessage = "你是一个助手";
    private Integer maxRetries = 3;
    private Duration timeout = Duration.ofSeconds(30);
    private CacheConfig cache = new CacheConfig();
    
    @Data
    public static class CacheConfig {
        private boolean enabled = true;
        private long maxSize = 1000;
        private Duration ttl = Duration.ofHours(1);
    }
}

@Configuration
@EnableConfigurationProperties(ChatProperties.class)
public class ChatClientAutoConfig {
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ChatProperties properties) {
        
        return ChatClient.builder(chatModel)
            .defaultSystem(properties.getDefaultSystemMessage())
            .defaultOptions(ChatOptions.builder()
                .model(properties.getDefaultModel())
                .temperature(properties.getDefaultTemperature())
                .maxTokens(properties.getDefaultMaxTokens())
                .build())
            .build();
    }
}
```

---

## 总结

### ChatClient核心特性

1. **流式API**: 优雅的链式调用
2. **默认配置**: 多层配置继承
3. **结构化输出**: 自动对象转换
4. **Advisor机制**: 强大的拦截器链
5. **多模态支持**: 图片、音频等
6. **可观测性**: Micrometer集成
7. **工具调用**: Function Calling支持

### API流程

```
ChatClient
    ↓
prompt() / prompt(String) / prompt(Prompt)
    ↓
system() / user() / messages() / options() / advisors() / tools()
    ↓
call() / stream()
    ↓
content() / chatResponse() / entity() / responseEntity()
```

### 选择ChatClient还是ChatModel？

| 场景 | 推荐 |
|------|------|
| 应用开发 | **ChatClient** |
| 快速原型 | **ChatClient** |
| 需要Advisor | **ChatClient** |
| 需要默认配置 | **ChatClient** |
| 结构化输出 | **ChatClient** |
| 模板渲染 | **ChatClient** |
| 底层集成 | ChatModel |
| 自定义模型实现 | ChatModel |

### 最佳实践清单

- ✅ 使用默认配置简化代码
- ✅ 利用entity()实现结构化输出
- ✅ 使用Advisor实现横切关注点
- ✅ 流式响应提升用户体验
- ✅ 实现错误处理和重试机制
- ✅ 添加监控和日志
- ✅ 使用缓存减少API调用
- ✅ 异步处理提高性能
- ✅ 合理管理Token使用

通过掌握ChatClient，你可以快速构建强大、优雅、可维护的AI应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

