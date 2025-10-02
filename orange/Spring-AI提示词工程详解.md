# Spring AI 提示词工程详解

## 📋 目录
- [概述](#概述)
- [核心架构](#核心架构)
- [提示词模板系统](#提示词模板系统)
- [ChatClient中的提示词处理](#chatclient中的提示词处理)
- [RAG中的提示词增强](#rag中的提示词增强)
- [Advisor中的提示词操作](#advisor中的提示词操作)
- [最佳实践](#最佳实践)
- [实战示例](#实战示例)

---

## 概述

Spring AI 提供了一套完整的提示词工程（Prompt Engineering）解决方案，涵盖从模板定义、变量注入、到动态增强的全流程。

### 设计理念

1. **分离关注点**: 提示词内容与业务逻辑分离
2. **模板化**: 支持参数化模板，避免字符串拼接
3. **可复用**: 提示词模板可以存储为文件并重复使用
4. **可扩展**: 支持自定义模板渲染器
5. **类型安全**: 基于Java类型系统，编译期检查

### 核心组件

```
提示词工程组件架构
├── Prompt (提示词对象)
│   ├── List<Message> (消息列表)
│   └── ChatOptions (调用选项)
├── PromptTemplate (模板引擎)
│   ├── Template String (模板字符串)
│   ├── Variables (变量映射)
│   └── TemplateRenderer (渲染器)
├── ChatClient (流式API)
│   ├── system() - 系统提示词
│   ├── user() - 用户提示词
│   └── params() - 参数注入
└── Advisor (增强器)
    ├── QuestionAnswerAdvisor (RAG增强)
    ├── PromptChatMemoryAdvisor (记忆增强)
    └── VectorStoreChatMemoryAdvisor (向量记忆)
```

---

## 核心架构

### 1. Prompt - 提示词对象

`Prompt` 是Spring AI中提示词的核心数据结构。

#### 类定义

```java
public class Prompt implements ModelRequest<List<Message>> {
    private final List<Message> messages;
    private ChatOptions chatOptions;
}
```

#### Message类型体系

```
Message (抽象基类)
├── SystemMessage      - 系统提示词，定义AI行为
├── UserMessage        - 用户消息
├── AssistantMessage   - AI助手回复
└── ToolResponseMessage - 工具调用结果
```

#### 创建Prompt的多种方式

```java
// 1. 简单字符串
Prompt prompt = new Prompt("你好");

// 2. 单个消息
Prompt prompt = new Prompt(new UserMessage("你好"));

// 3. 多个消息
Prompt prompt = new Prompt(
    new SystemMessage("你是一个友好的助手"),
    new UserMessage("你好")
);

// 4. 消息列表
List<Message> messages = List.of(
    new SystemMessage("你是一个友好的助手"),
    new UserMessage("你好")
);
Prompt prompt = new Prompt(messages);

// 5. 带选项
Prompt prompt = new Prompt(
    messages,
    ChatOptions.builder().temperature(0.8).build()
);

// 6. 使用Builder
Prompt prompt = Prompt.builder()
    .messages(messages)
    .chatOptions(chatOptions)
    .build();
```

#### Prompt的重要方法

```java
// 获取第一个系统消息
SystemMessage systemMessage = prompt.getSystemMessage();

// 获取最后一个用户消息
UserMessage userMessage = prompt.getUserMessage();

// 获取所有用户消息
List<UserMessage> userMessages = prompt.getUserMessages();

// 增强系统消息
Prompt newPrompt = prompt.augmentSystemMessage("新的系统提示");

// 增强用户消息
Prompt newPrompt = prompt.augmentUserMessage("补充内容");

// 使用函数增强
Prompt newPrompt = prompt.augmentSystemMessage(
    systemMsg -> systemMsg.mutate()
        .text("新内容: " + systemMsg.getText())
        .build()
);
```

---

## 提示词模板系统

### 1. PromptTemplate - 核心模板类

`PromptTemplate` 提供了参数化提示词的能力。

#### 基本用法

```java
// 1. 创建模板
PromptTemplate template = new PromptTemplate(
    "你好，{name}！今天天气{weather}。"
);

// 2. 添加变量
template.add("name", "张三");

// 3. 渲染为字符串
String rendered = template.render(Map.of("weather", "晴朗"));
// 结果: "你好，张三！今天天气晴朗。"

// 4. 创建消息
Message message = template.createMessage(Map.of("weather", "晴朗"));

// 5. 创建Prompt
Prompt prompt = template.create(Map.of("weather", "晴朗"));
```

#### 从资源文件加载

```java
// 从类路径加载
Resource resource = new ClassPathResource("prompts/system-message.st");
PromptTemplate template = new PromptTemplate(resource);

// 从文件系统加载
Resource resource = new FileSystemResource("templates/prompt.st");
PromptTemplate template = new PromptTemplate(resource);
```

#### Builder模式

```java
PromptTemplate template = PromptTemplate.builder()
    .template("你好，{name}！")
    .variables(Map.of("name", "张三"))
    .build();

// 或者从资源
PromptTemplate template = PromptTemplate.builder()
    .resource(resource)
    .variables(variables)
    .build();
```

### 2. 角色特定的模板

Spring AI 为不同角色提供了专门的模板类。

#### SystemPromptTemplate

```java
SystemPromptTemplate systemTemplate = new SystemPromptTemplate(
    "你是一个{role}。你的专长是{specialty}。"
);

Message systemMessage = systemTemplate.createMessage(
    Map.of(
        "role", "软件工程师",
        "specialty", "Java开发"
    )
);

Prompt prompt = systemTemplate.create(variables);
```

#### UserPromptTemplate

```java
UserPromptTemplate userTemplate = new UserPromptTemplate(
    "请帮我{action}关于{topic}的内容。"
);

Message userMessage = userTemplate.createMessage(
    Map.of(
        "action", "解释",
        "topic", "Spring AI"
    )
);
```

#### AssistantPromptTemplate

```java
AssistantPromptTemplate assistantTemplate = 
    new AssistantPromptTemplate("我理解了，{response}");

Message assistantMessage = assistantTemplate.createMessage(
    Map.of("response", "让我为您详细说明")
);
```

### 3. ChatPromptTemplate - 多消息模板

用于组合多个角色的提示词模板。

```java
List<PromptTemplate> templates = List.of(
    new SystemPromptTemplate("你是{role}"),
    new UserPromptTemplate("请{action}"),
    new AssistantPromptTemplate("好的，{response}")
);

ChatPromptTemplate chatTemplate = new ChatPromptTemplate(templates);

// 渲染所有消息
List<Message> messages = chatTemplate.createMessages(
    Map.of(
        "role", "助手",
        "action", "解释",
        "response", "我来说明"
    )
);

// 创建Prompt
Prompt prompt = chatTemplate.create(variables);
```

### 4. 模板渲染器

Spring AI 支持可插拔的模板渲染器。

#### TemplateRenderer接口

```java
@FunctionalInterface
public interface TemplateRenderer 
    extends BiFunction<String, Map<String, Object>, String> {
    
    String apply(String template, Map<String, Object> variables);
}
```

#### StTemplateRenderer (默认)

基于StringTemplate 4 (ST4) 的渲染器。

```java
StTemplateRenderer renderer = StTemplateRenderer.builder()
    .startDelimiterToken('{')      // 开始分隔符
    .endDelimiterToken('}')        // 结束分隔符
    .validationMode(ValidationMode.THROW)  // 验证模式
    .build();

PromptTemplate template = PromptTemplate.builder()
    .template("Hello {name}!")
    .renderer(renderer)
    .build();
```

**ST4模板语法特性**:

```java
// 1. 简单变量替换
"Hello {name}!"

// 2. 条件渲染
"Hello {if(name)}{name}{else}Guest{endif}!"

// 3. 列表迭代
"Items: {items; separator=', '}"

// 4. 对象属性访问
"User: {user.name}, Age: {user.age}"

// 5. 内置函数
"{first(items)}"  // 第一个元素
"{rest(items)}"   // 除第一个外的元素
"{length(items)}" // 列表长度
```

#### NoOpTemplateRenderer

不做任何处理，直接返回原模板字符串。

```java
TemplateRenderer renderer = new NoOpTemplateRenderer();

PromptTemplate template = PromptTemplate.builder()
    .template("This is raw text")
    .renderer(renderer)
    .build();

String result = template.render(); // 返回 "This is raw text"
```

#### 自定义渲染器

```java
public class MustacheTemplateRenderer implements TemplateRenderer {
    
    private final MustacheFactory mf = new DefaultMustacheFactory();
    
    @Override
    public String apply(String template, Map<String, Object> variables) {
        Mustache mustache = mf.compile(
            new StringReader(template), 
            "template"
        );
        
        StringWriter writer = new StringWriter();
        mustache.execute(writer, variables);
        return writer.toString();
    }
}
```

### 5. 验证模式

控制模板变量验证的行为。

```java
public enum ValidationMode {
    THROW,  // 缺少变量时抛出异常（默认）
    WARN,   // 缺少变量时记录警告
    NONE    // 不进行验证
}

// 使用示例
StTemplateRenderer renderer = StTemplateRenderer.builder()
    .validationMode(ValidationMode.WARN)
    .build();
```

### 6. Resource变量支持

模板变量可以是Resource类型，会自动读取内容。

```java
Map<String, Object> variables = new HashMap<>();
variables.put("content", new ClassPathResource("data/content.txt"));
variables.put("name", "Spring AI");

PromptTemplate template = new PromptTemplate(
    "Name: {name}\nContent: {content}"
);

String rendered = template.render(variables);
// Resource会被自动读取并替换
```

---

## ChatClient中的提示词处理

`ChatClient` 提供了流畅的API来构建和处理提示词。

### 1. 基本提示词设置

```java
ChatClient chatClient = ChatClient.builder(chatModel).build();

// 方式1: 直接字符串
String response = chatClient
    .prompt("你好，世界")
    .call()
    .content();

// 方式2: 完整Prompt对象
Prompt prompt = new Prompt("你好");
String response = chatClient
    .prompt(prompt)
    .call()
    .content();
```

### 2. 系统提示词 (system)

系统提示词用于定义AI的行为、角色和规则。

```java
// 简单文本
chatClient
    .prompt()
    .system("你是一个友好的AI助手")
    .user("你好")
    .call()
    .content();

// 从资源文件加载
chatClient
    .prompt()
    .system(new ClassPathResource("prompts/system.txt"))
    .user("你好")
    .call()
    .content();

// 指定字符集
chatClient
    .prompt()
    .system(resource, StandardCharsets.UTF_8)
    .user("你好")
    .call()
    .content();

// 使用Consumer配置
chatClient
    .prompt()
    .system(s -> s
        .text("你是一个{role}")
        .param("role", "软件工程师")
        .metadata("priority", "high")
    )
    .user("你好")
    .call()
    .content();
```

### 3. 用户提示词 (user)

用户提示词包含用户的查询或指令。

```java
// 简单文本
chatClient
    .prompt()
    .user("解释什么是Spring AI")
    .call()
    .content();

// 带参数的模板
chatClient
    .prompt()
    .user(u -> u
        .text("请解释{topic}，重点关注{aspect}")
        .param("topic", "Spring AI")
        .param("aspect", "架构设计")
    )
    .call()
    .content();

// 多模态消息（带媒体）
chatClient
    .prompt()
    .user(u -> u
        .text("这张图片里有什么？")
        .media(new Media(MimeTypeUtils.IMAGE_PNG, 
                        new ClassPathResource("images/photo.png")))
    )
    .call()
    .content();

// 添加元数据
chatClient
    .prompt()
    .user(u -> u
        .text("查询内容")
        .metadata("userId", "12345")
        .metadata("sessionId", "abc-123")
    )
    .call()
    .content();
```

### 4. 参数化提示词

参数会自动应用到system和user提示词中的模板。

```java
// 在system中使用参数
chatClient
    .prompt()
    .system(s -> s
        .text("你是一个{role}，擅长{skill}")
        .param("role", "Java开发者")
        .param("skill", "Spring框架")
    )
    .user(u -> u
        .text("如何使用{framework}实现{feature}？")
        .param("framework", "Spring AI")
        .param("feature", "RAG")
    )
    .call()
    .content();
```

### 5. 默认提示词

可以在Builder中设置默认的系统和用户提示词。

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultSystem("你是一个专业的技术助手")
    .defaultUser("请用简洁的语言回答")
    .build();

// 后续调用会自动应用默认提示词
String response = chatClient
    .prompt("什么是Spring？")
    .call()
    .content();
```

### 6. 自定义模板渲染器

```java
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultTemplateRenderer(customRenderer)
    .build();

// 或在请求时指定
chatClient
    .prompt()
    .templateRenderer(customRenderer)
    .system("...")
    .user("...")
    .call()
    .content();
```

### 7. 内部处理流程

`ChatClient` 内部是如何处理提示词的：

```java
// DefaultChatClientUtils.toChatClientRequest()
static ChatClientRequest toChatClientRequest(
        DefaultChatClientRequestSpec inputRequest) {
    
    List<Message> processedMessages = new ArrayList<>();
    
    // 1. 处理系统提示词
    String systemText = inputRequest.getSystemText();
    if (StringUtils.hasText(systemText)) {
        // 如果有参数，使用模板渲染
        if (!CollectionUtils.isEmpty(inputRequest.getSystemParams())) {
            systemText = PromptTemplate.builder()
                .template(systemText)
                .variables(inputRequest.getSystemParams())
                .renderer(inputRequest.getTemplateRenderer())
                .build()
                .render();
        }
        processedMessages.add(SystemMessage.builder()
            .text(systemText)
            .metadata(inputRequest.getSystemMetadata())
            .build());
    }
    
    // 2. 添加其他消息
    processedMessages.addAll(inputRequest.getMessages());
    
    // 3. 处理用户提示词
    String userText = inputRequest.getUserText();
    if (StringUtils.hasText(userText)) {
        // 如果有参数，使用模板渲染
        if (!CollectionUtils.isEmpty(inputRequest.getUserParams())) {
            userText = PromptTemplate.builder()
                .template(userText)
                .variables(inputRequest.getUserParams())
                .renderer(inputRequest.getTemplateRenderer())
                .build()
                .render();
        }
        processedMessages.add(UserMessage.builder()
            .text(userText)
            .media(inputRequest.getMedia())
            .metadata(inputRequest.getUserMetadata())
            .build());
    }
    
    return new ChatClientRequest(processedMessages, options, context);
}
```

---

## RAG中的提示词增强

RAG（Retrieval Augmented Generation）通过检索相关文档来增强提示词。

### 1. QuestionAnswerAdvisor

自动从向量数据库检索相关文档并注入到用户提示词中。

#### 默认提示词模板

```java
private static final PromptTemplate DEFAULT_PROMPT_TEMPLATE = 
    new PromptTemplate("""
        {query}
        
        Context information is below, surrounded by ---------------------
        
        ---------------------
        {question_answer_context}
        ---------------------
        
        Given the context and provided history information and not prior knowledge,
        reply to the user comment. If the answer is not in the context, inform
        the user that you can't answer the question.
        """);
```

#### 工作流程

```java
@Override
public ChatClientRequest before(
        ChatClientRequest request, 
        AdvisorChain chain) {
    
    // 1. 从向量数据库检索相关文档
    SearchRequest searchRequest = SearchRequest.from(this.searchRequest)
        .query(request.prompt().getUserMessage().getText())
        .filterExpression(filterExpression)
        .build();
    
    List<Document> documents = vectorStore.similaritySearch(searchRequest);
    
    // 2. 将文档转换为上下文字符串
    String documentContext = documents.stream()
        .map(Document::getText)
        .collect(Collectors.joining(System.lineSeparator()));
    
    // 3. 使用模板渲染增强后的提示词
    UserMessage userMessage = request.prompt().getUserMessage();
    String augmentedUserText = this.promptTemplate.render(
        Map.of(
            "query", userMessage.getText(),
            "question_answer_context", documentContext
        )
    );
    
    // 4. 返回增强后的请求
    return request.mutate()
        .prompt(request.prompt().augmentUserMessage(augmentedUserText))
        .context(context)
        .build();
}
```

#### 使用示例

```java
// 1. 基本用法
QuestionAnswerAdvisor qaAdvisor = 
    new QuestionAnswerAdvisor(vectorStore);

chatClient
    .prompt("Spring AI有哪些特性？")
    .advisors(qaAdvisor)
    .call()
    .content();

// 2. 自定义配置
QuestionAnswerAdvisor qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .searchRequest(SearchRequest.builder()
        .topK(5)
        .similarityThreshold(0.7)
        .build())
    .promptTemplate(customTemplate)
    .build();

// 3. 自定义提示词模板
PromptTemplate customTemplate = new PromptTemplate("""
    User Question: {query}
    
    Relevant Documents:
    {question_answer_context}
    
    Please provide a detailed answer based on the documents above.
    If the information is not available, say "I don't know".
    """);

QuestionAnswerAdvisor qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
    .promptTemplate(customTemplate)
    .build();
```

#### 真实示例：产品目录QA

```java
// prompts/acme/system-qa.st
"""
You're assisting with questions about products in a bicycle catalog.
Use the information from the DOCUMENTS section to provide accurate answers.
If the answer involves referring to the price or the dimension of the bicycle,
include the bicycle name in the response.
If unsure, simply state that you don't know.

DOCUMENTS:
{documents}
"""

// 使用
PromptTemplate productQaTemplate = new PromptTemplate(
    new ClassPathResource("prompts/acme/system-qa.st")
);

QuestionAnswerAdvisor advisor = QuestionAnswerAdvisor.builder(vectorStore)
    .promptTemplate(productQaTemplate)
    .build();
```

### 2. 文档格式化

可以自定义文档如何格式化注入到提示词中。

```java
// 默认格式：直接连接文档文本
String context = documents.stream()
    .map(Document::getText)
    .collect(Collectors.joining(System.lineSeparator()));

// 自定义格式：包含元数据
String context = documents.stream()
    .map(doc -> String.format(
        "Title: %s\nSource: %s\nContent: %s",
        doc.getMetadata().get("title"),
        doc.getMetadata().get("source"),
        doc.getText()
    ))
    .collect(Collectors.joining("\n\n---\n\n"));
```

### 3. Document的ContentFormatter

`Document` 类支持自定义内容格式化。

```java
ContentFormatter formatter = DefaultContentFormatter.builder()
    .metadataTemplate("{key}: {value}")
    .metadataSeparator("\n")
    .textTemplate("{metadata_string}\n\n{content}")
    .excludedInferenceMetadataKeys(List.of("embedding"))
    .build();

String formattedContent = formatter.format(document, MetadataMode.INFERENCE);
```

---

## Advisor中的提示词操作

Advisor是Spring AI中的拦截器模式，可以在调用AI模型前后修改提示词。

### 1. Advisor接口

```java
public interface Advisor {
    
    // 在调用模型前执行
    ChatClientRequest before(
        ChatClientRequest request, 
        AdvisorChain chain
    );
    
    // 在调用模型后执行
    ChatClientResponse after(
        ChatClientResponse response, 
        AdvisorChain chain
    );
    
    int getOrder();  // 执行顺序
}
```

### 2. PromptChatMemoryAdvisor

将对话历史注入到系统提示词中。

#### 默认提示词模板

```java
private static final PromptTemplate DEFAULT_SYSTEM_PROMPT_TEMPLATE = 
    new PromptTemplate("""
        {instructions}
        
        Use the conversation memory from the MEMORY section to provide accurate answers.
        
        ---------------------
        MEMORY:
        {memory}
        ---------------------
        """);
```

#### 工作流程

```java
@Override
public ChatClientRequest before(
        ChatClientRequest request, 
        AdvisorChain chain) {
    
    // 1. 获取对话历史
    List<Message> memoryMessages = chatMemory.get(conversationId);
    
    // 2. 格式化为字符串
    String memory = memoryMessages.stream()
        .filter(m -> m.getMessageType() == MessageType.USER 
                  || m.getMessageType() == MessageType.ASSISTANT)
        .map(m -> m.getMessageType() + ":" + m.getText())
        .collect(Collectors.joining(System.lineSeparator()));
    
    // 3. 使用模板增强系统提示词
    SystemMessage systemMessage = request.prompt().getSystemMessage();
    String augmentedSystemText = this.systemPromptTemplate.render(
        Map.of(
            "instructions", systemMessage.getText(),
            "memory", memory
        )
    );
    
    // 4. 更新请求
    ChatClientRequest processedRequest = request.mutate()
        .prompt(request.prompt().augmentSystemMessage(augmentedSystemText))
        .build();
    
    // 5. 保存当前用户消息到记忆
    chatMemory.add(conversationId, 
                   processedRequest.prompt().getUserMessage());
    
    return processedRequest;
}
```

#### 使用示例

```java
// 1. 基本用法
ChatMemory chatMemory = new InMemoryChatMemory();
PromptChatMemoryAdvisor memoryAdvisor = 
    new PromptChatMemoryAdvisor(chatMemory);

chatClient
    .prompt("我之前问过你什么？")
    .advisors(memoryAdvisor)
    .call()
    .content();

// 2. 指定会话ID
chatClient
    .prompt("继续之前的话题")
    .advisors(memoryAdvisor)
    .advisorParams(Map.of(
        CONVERSATION_ID_KEY, "session-123"
    ))
    .call()
    .content();

// 3. 自定义提示词模板
PromptTemplate customMemoryTemplate = new PromptTemplate("""
    System Instructions: {instructions}
    
    Previous Conversation:
    {memory}
    
    Remember to maintain context from previous messages.
    """);

PromptChatMemoryAdvisor advisor = PromptChatMemoryAdvisor.builder()
    .chatMemory(chatMemory)
    .systemPromptTemplate(customMemoryTemplate)
    .build();
```

### 3. VectorStoreChatMemoryAdvisor

使用向量数据库存储和检索对话历史。

#### 默认提示词模板

```java
private static final PromptTemplate DEFAULT_SYSTEM_PROMPT_TEMPLATE = 
    new PromptTemplate("""
        {instructions}
        
        Use the long term conversation memory from the LONG_TERM_MEMORY section
        to provide accurate answers.
        
        ---------------------
        LONG_TERM_MEMORY:
        {long_term_memory}
        ---------------------
        """);
```

#### 工作流程

```java
@Override
public ChatClientRequest before(
        ChatClientRequest request, 
        AdvisorChain chain) {
    
    String conversationId = getConversationId(request.context());
    String query = request.prompt().getUserMessage().getText();
    
    // 1. 从向量存储检索相关历史
    String filter = "conversationId=='" + conversationId + "'";
    SearchRequest searchRequest = SearchRequest.builder()
        .query(query)
        .topK(topK)
        .filterExpression(filter)
        .build();
    
    List<Document> documents = vectorStore.similaritySearch(searchRequest);
    
    // 2. 格式化长期记忆
    String longTermMemory = documents.stream()
        .map(Document::getText)
        .collect(Collectors.joining(System.lineSeparator()));
    
    // 3. 增强系统提示词
    SystemMessage systemMessage = request.prompt().getSystemMessage();
    String augmentedSystemText = this.systemPromptTemplate.render(
        Map.of(
            "instructions", systemMessage.getText(),
            "long_term_memory", longTermMemory
        )
    );
    
    // 4. 保存当前消息到向量存储
    UserMessage userMessage = request.prompt().getUserMessage();
    vectorStore.write(toDocuments(List.of(userMessage), conversationId));
    
    return request.mutate()
        .prompt(request.prompt().augmentSystemMessage(augmentedSystemText))
        .build();
}
```

### 4. 自定义Advisor

创建自定义Advisor来实现特定的提示词增强逻辑。

```java
public class SentimentAnalysisAdvisor implements Advisor {
    
    private static final PromptTemplate SENTIMENT_TEMPLATE = 
        new PromptTemplate("""
            Original Query: {query}
            Detected Sentiment: {sentiment}
            
            Please respond appropriately considering the user's sentiment.
            """);
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request, 
            AdvisorChain chain) {
        
        // 1. 分析用户情绪
        String userText = request.prompt().getUserMessage().getText();
        String sentiment = analyzeSentiment(userText);
        
        // 2. 增强提示词
        String augmentedText = SENTIMENT_TEMPLATE.render(
            Map.of(
                "query", userText,
                "sentiment", sentiment
            )
        );
        
        return request.mutate()
            .prompt(request.prompt().augmentUserMessage(augmentedText))
            .build();
    }
    
    private String analyzeSentiment(String text) {
        // 情绪分析逻辑
        return "positive";
    }
}
```

### 5. ChatModelCallAdvisor - 输出格式增强

自动添加输出格式指令到用户提示词。

```java
private static ChatClientRequest augmentWithFormatInstructions(
        ChatClientRequest request) {
    
    String outputFormat = (String) request.context()
        .get(ChatClientAttributes.OUTPUT_FORMAT.getKey());
    
    if (!StringUtils.hasText(outputFormat)) {
        return request;
    }
    
    // 将格式指令追加到用户消息
    Prompt augmentedPrompt = request.prompt()
        .augmentUserMessage(userMessage -> userMessage.mutate()
            .text(userMessage.getText() + 
                  System.lineSeparator() + 
                  outputFormat)
            .build());
    
    return ChatClientRequest.builder()
        .prompt(augmentedPrompt)
        .context(request.context())
        .build();
}
```

---

## 最佳实践

### 1. 提示词组织

#### 文件组织结构

```
src/main/resources/prompts/
├── system/
│   ├── default.st           # 默认系统提示词
│   ├── expert.st            # 专家角色
│   └── assistant.st         # 助手角色
├── user/
│   ├── query.st             # 查询模板
│   └── command.st           # 命令模板
└── rag/
    ├── qa.st                # 问答模板
    └── summarize.st         # 摘要模板
```

#### 配置类管理

```java
@Configuration
public class PromptConfiguration {
    
    @Bean
    public PromptTemplate systemPromptTemplate() {
        return new PromptTemplate(
            new ClassPathResource("prompts/system/default.st")
        );
    }
    
    @Bean
    public PromptTemplate qaPromptTemplate() {
        return new PromptTemplate(
            new ClassPathResource("prompts/rag/qa.st")
        );
    }
}
```

### 2. 提示词版本控制

```java
@Service
public class PromptVersionService {
    
    private final Map<String, PromptTemplate> templates = new HashMap<>();
    
    public PromptTemplate getTemplate(String name, String version) {
        String key = name + ":" + version;
        return templates.computeIfAbsent(key, k -> loadTemplate(name, version));
    }
    
    private PromptTemplate loadTemplate(String name, String version) {
        String path = String.format(
            "prompts/%s/%s.st", 
            version, 
            name
        );
        return new PromptTemplate(new ClassPathResource(path));
    }
}
```

### 3. 提示词测试

```java
@SpringBootTest
class PromptTemplateTest {
    
    @Test
    void testSystemPromptRendering() {
        PromptTemplate template = new PromptTemplate(
            "You are a {role} specialized in {domain}"
        );
        
        String rendered = template.render(Map.of(
            "role", "software engineer",
            "domain", "AI development"
        ));
        
        assertThat(rendered).contains("software engineer");
        assertThat(rendered).contains("AI development");
    }
    
    @Test
    void testMissingVariableValidation() {
        PromptTemplate template = PromptTemplate.builder()
            .template("Hello {name}!")
            .renderer(StTemplateRenderer.builder()
                .validationMode(ValidationMode.THROW)
                .build())
            .build();
        
        assertThatThrownBy(() -> template.render(Map.of()))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("name");
    }
}
```

### 4. 多语言支持

```java
@Service
public class I18nPromptService {
    
    @Autowired
    private MessageSource messageSource;
    
    public PromptTemplate getLocalizedTemplate(
            String templateKey, 
            Locale locale) {
        
        String template = messageSource.getMessage(
            templateKey, 
            null, 
            locale
        );
        
        return new PromptTemplate(template);
    }
}

// messages_zh_CN.properties
prompt.greeting=你好，{name}！
prompt.system=你是一个{role}

// messages_en_US.properties
prompt.greeting=Hello, {name}!
prompt.system=You are a {role}
```

### 5. 提示词缓存

```java
@Service
public class CachedPromptService {
    
    private final LoadingCache<String, PromptTemplate> cache;
    
    public CachedPromptService() {
        this.cache = CacheBuilder.newBuilder()
            .maximumSize(100)
            .expireAfterAccess(1, TimeUnit.HOURS)
            .build(new CacheLoader<String, PromptTemplate>() {
                @Override
                public PromptTemplate load(String path) {
                    return new PromptTemplate(
                        new ClassPathResource(path)
                    );
                }
            });
    }
    
    public PromptTemplate getTemplate(String path) {
        return cache.getUnchecked(path);
    }
}
```

### 6. 提示词安全

#### 防止注入攻击

```java
public class SecurePromptBuilder {
    
    public static PromptTemplate build(
            String template, 
            Map<String, Object> userInputs) {
        
        // 1. 清理用户输入
        Map<String, Object> sanitized = new HashMap<>();
        for (Map.Entry<String, Object> entry : userInputs.entrySet()) {
            sanitized.put(
                entry.getKey(), 
                sanitize(entry.getValue())
            );
        }
        
        // 2. 验证模板
        validateTemplate(template);
        
        return PromptTemplate.builder()
            .template(template)
            .variables(sanitized)
            .build();
    }
    
    private static Object sanitize(Object value) {
        if (value instanceof String str) {
            // 移除潜在的恶意内容
            return str.replaceAll("[<>\\{\\}\\$]", "")
                     .trim();
        }
        return value;
    }
    
    private static void validateTemplate(String template) {
        // 检查模板是否包含敏感操作
        if (template.contains("System.") || template.contains("Runtime.")) {
            throw new SecurityException("Template contains forbidden operations");
        }
    }
}
```

### 7. 提示词监控

```java
@Aspect
@Component
public class PromptMonitoringAspect {
    
    private final MeterRegistry meterRegistry;
    
    @Around("execution(* org.springframework.ai.chat.prompt.PromptTemplate.render(..))")
    public Object monitorPromptRendering(ProceedingJoinPoint joinPoint) 
            throws Throwable {
        
        long startTime = System.currentTimeMillis();
        
        try {
            Object result = joinPoint.proceed();
            
            // 记录成功指标
            meterRegistry.counter("prompt.render.success").increment();
            
            // 记录渲染时间
            long duration = System.currentTimeMillis() - startTime;
            meterRegistry.timer("prompt.render.duration")
                .record(duration, TimeUnit.MILLISECONDS);
            
            return result;
        } catch (Exception e) {
            // 记录失败指标
            meterRegistry.counter("prompt.render.failure").increment();
            throw e;
        }
    }
}
```

---

## 实战示例

### 示例1: 简单问答系统

```java
@Service
public class SimpleQAService {
    
    private final ChatClient chatClient;
    
    public SimpleQAService(ChatModel chatModel) {
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("你是一个知识渊博的助手")
            .build();
    }
    
    public String ask(String question) {
        return chatClient
            .prompt(question)
            .call()
            .content();
    }
}
```

### 示例2: 角色扮演对话

```java
@Service
public class RolePlayService {
    
    private final ChatClient chatClient;
    private final PromptTemplate roleTemplate;
    
    public RolePlayService(ChatModel chatModel) {
        this.chatClient = ChatClient.create(chatModel);
        
        this.roleTemplate = new PromptTemplate("""
            You are roleplaying as {character}.
            Personality: {personality}
            Background: {background}
            
            Stay in character and respond accordingly.
            """);
    }
    
    public String chat(String character, String userMessage) {
        Map<String, Object> roleInfo = getRoleInfo(character);
        
        return chatClient
            .prompt()
            .system(roleTemplate.render(roleInfo))
            .user(userMessage)
            .call()
            .content();
    }
    
    private Map<String, Object> getRoleInfo(String character) {
        return switch (character) {
            case "sherlock" -> Map.of(
                "character", "Sherlock Holmes",
                "personality", "Analytical, observant, logical",
                "background", "Famous detective from Victorian London"
            );
            case "yoda" -> Map.of(
                "character", "Yoda",
                "personality", "Wise, patient, speaks in reversed syntax",
                "background", "Jedi Master from Star Wars"
            );
            default -> Map.of(
                "character", "Assistant",
                "personality", "Helpful and friendly",
                "background", "AI assistant"
            );
        };
    }
}
```

### 示例3: RAG文档问答

```java
@Service
public class DocumentQAService {
    
    private final ChatClient chatClient;
    private final VectorStore vectorStore;
    private final QuestionAnswerAdvisor qaAdvisor;
    
    public DocumentQAService(
            ChatModel chatModel,
            VectorStore vectorStore) {
        
        // 自定义QA提示词模板
        PromptTemplate qaTemplate = new PromptTemplate("""
            Question: {query}
            
            Relevant Information:
            {question_answer_context}
            
            Based on the information above, please provide a comprehensive answer.
            If you cannot find the answer in the provided context, say so.
            Include relevant details and examples where appropriate.
            """);
        
        this.qaAdvisor = QuestionAnswerAdvisor.builder(vectorStore)
            .searchRequest(SearchRequest.builder()
                .topK(5)
                .similarityThreshold(0.75)
                .build())
            .promptTemplate(qaTemplate)
            .build();
        
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("""
                You are a helpful assistant that answers questions based on 
                the provided documentation. Be accurate and cite sources when possible.
                """)
            .defaultAdvisors(qaAdvisor)
            .build();
        
        this.vectorStore = vectorStore;
    }
    
    public String askDocument(String question, String documentFilter) {
        return chatClient
            .prompt(question)
            .advisorParams(Map.of(
                QuestionAnswerAdvisor.FILTER_EXPRESSION, documentFilter
            ))
            .call()
            .content();
    }
    
    public void addDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }
}
```

### 示例4: 带记忆的对话系统

```java
@Service
public class ConversationalService {
    
    private final ChatClient chatClient;
    private final ChatMemory chatMemory;
    
    public ConversationalService(
            ChatModel chatModel,
            ChatMemory chatMemory) {
        
        PromptTemplate memoryTemplate = new PromptTemplate("""
            {instructions}
            
            Previous Conversation:
            {memory}
            
            Please maintain context and refer to previous messages when relevant.
            """);
        
        PromptChatMemoryAdvisor memoryAdvisor = 
            PromptChatMemoryAdvisor.builder()
                .chatMemory(chatMemory)
                .systemPromptTemplate(memoryTemplate)
                .build();
        
        this.chatClient = ChatClient.builder(chatModel)
            .defaultSystem("You are a friendly conversational assistant")
            .defaultAdvisors(memoryAdvisor)
            .build();
        
        this.chatMemory = chatMemory;
    }
    
    public String chat(String userId, String message) {
        return chatClient
            .prompt(message)
            .advisorParams(Map.of(
                PromptChatMemoryAdvisor.CONVERSATION_ID_KEY, userId
            ))
            .call()
            .content();
    }
    
    public void clearHistory(String userId) {
        chatMemory.clear(userId);
    }
}
```

### 示例5: 多步骤任务处理

```java
@Service
public class TaskProcessingService {
    
    private final ChatClient chatClient;
    
    public TaskProcessingService(ChatModel chatModel) {
        this.chatClient = ChatClient.create(chatModel);
    }
    
    public TaskResult processTask(Task task) {
        // Step 1: 分析任务
        String analysis = chatClient
            .prompt()
            .system("""
                You are a task analyzer. Break down the task into steps.
                Output format: Step 1: ..., Step 2: ..., etc.
                """)
            .user("Analyze this task: " + task.getDescription())
            .call()
            .content();
        
        // Step 2: 生成计划
        PromptTemplate planTemplate = new PromptTemplate("""
            Task: {task}
            Analysis: {analysis}
            
            Create a detailed execution plan with timeline and resources.
            """);
        
        String plan = chatClient
            .prompt()
            .system("You are a project planner")
            .user(planTemplate.render(Map.of(
                "task", task.getDescription(),
                "analysis", analysis
            )))
            .call()
            .content();
        
        // Step 3: 生成代码/实现
        PromptTemplate implTemplate = new PromptTemplate("""
            Task: {task}
            Plan: {plan}
            
            Generate implementation code or detailed instructions.
            """);
        
        String implementation = chatClient
            .prompt()
            .system("You are a senior developer")
            .user(implTemplate.render(Map.of(
                "task", task.getDescription(),
                "plan", plan
            )))
            .call()
            .content();
        
        return new TaskResult(analysis, plan, implementation);
    }
}
```

### 示例6: 内容生成器

```java
@Service
public class ContentGeneratorService {
    
    private final ChatClient chatClient;
    private final PromptTemplate blogTemplate;
    
    public ContentGeneratorService(ChatModel chatModel) {
        this.chatClient = ChatClient.create(chatModel);
        
        // 从文件加载模板
        this.blogTemplate = new PromptTemplate(
            new ClassPathResource("prompts/blog-generator.st")
        );
    }
    
    public BlogPost generateBlogPost(BlogRequest request) {
        // 渲染提示词
        String prompt = blogTemplate.render(Map.of(
            "topic", request.getTopic(),
            "keywords", String.join(", ", request.getKeywords()),
            "tone", request.getTone(),
            "length", request.getLength(),
            "targetAudience", request.getTargetAudience()
        ));
        
        // 生成标题
        String title = chatClient
            .prompt()
            .system("Generate a catchy blog post title")
            .user(prompt)
            .call()
            .content();
        
        // 生成大纲
        String outline = chatClient
            .prompt()
            .system("Create a detailed outline for the blog post")
            .user("Title: " + title + "\n" + prompt)
            .call()
            .content();
        
        // 生成完整内容
        String content = chatClient
            .prompt()
            .system("Write a comprehensive blog post based on the outline")
            .user("Outline:\n" + outline + "\n\nExpand each section with details.")
            .call()
            .content();
        
        return BlogPost.builder()
            .title(title)
            .outline(outline)
            .content(content)
            .build();
    }
}
```

### 示例7: AI评估器

```java
@Service
public class AIEvaluatorService {
    
    private final ChatClient chatClient;
    private final PromptTemplate accuracyTemplate;
    
    public AIEvaluatorService(ChatModel chatModel) {
        this.chatClient = ChatClient.create(chatModel);
        
        // 加载评估模板
        this.accuracyTemplate = new PromptTemplate(
            new ClassPathResource(
                "prompts/spring/test/evaluation/qa-evaluator-accurate-answer.st"
            )
        );
    }
    
    public EvaluationResult evaluate(String question, String answer) {
        String evaluationPrompt = accuracyTemplate.render(Map.of(
            "question", question,
            "answer", answer
        ));
        
        String result = chatClient
            .prompt()
            .user(evaluationPrompt)
            .call()
            .content();
        
        boolean isAccurate = result.toLowerCase().contains("yes");
        
        return new EvaluationResult(isAccurate, result);
    }
}
```

---

## 总结

Spring AI的提示词工程系统具有以下特点：

### 核心优势

1. **模板化**: 使用 `PromptTemplate` 实现参数化提示词
2. **可插拔**: 支持自定义 `TemplateRenderer`
3. **类型安全**: 基于Java类型系统，编译期检查
4. **流式API**: `ChatClient` 提供优雅的API
5. **增强机制**: `Advisor` 模式支持动态提示词增强
6. **资源管理**: 支持从文件加载提示词模板

### 关键组件

- **Prompt**: 提示词数据结构
- **PromptTemplate**: 模板引擎
- **ChatClient**: 流式构建器
- **Advisor**: 拦截器/增强器
- **TemplateRenderer**: 渲染引擎

### 最佳实践

1. 将提示词模板存储为文件
2. 使用参数化避免字符串拼接
3. 利用Advisor实现复杂的提示词增强
4. 为不同场景准备专用模板
5. 实施提示词版本控制
6. 添加安全验证和监控

通过掌握这套提示词工程体系，你可以构建强大、灵活、可维护的AI应用。

---

## 参考资源

- Spring AI官方文档: https://docs.spring.io/spring-ai/reference/
- StringTemplate 4文档: https://github.com/antlr/stringtemplate4
- 源码位置:
  - `spring-ai-model/src/main/java/org/springframework/ai/chat/prompt/`
  - `spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/`
  - `spring-ai-template-st/src/main/java/org/springframework/ai/template/st/`

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

