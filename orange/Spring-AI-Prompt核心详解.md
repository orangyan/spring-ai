# Spring AI - Prompt核心详解

> 从基础到高级，全面掌握Spring AI的Prompt体系

## 目录

- [一、核心概念](#一核心概念)
- [二、Prompt类详解](#二prompt类详解)
- [三、PromptTemplate模板系统](#三prompttemplate模板系统)
- [四、Message体系](#四message体系)
- [五、高级特性](#五高级特性)
- [六、完整实战案例](#六完整实战案例)
- [七、最佳实践](#七最佳实践)

---

## 一、核心概念

### 1.1 什么是Prompt？

**Prompt**是发送给AI模型的完整请求对象，包含：
- **消息列表**（Messages）：System、User、Assistant等
- **配置选项**（ChatOptions）：temperature、maxTokens等

```
Prompt = Messages + ChatOptions
```

### 1.2 Prompt在Spring AI中的位置

```
┌─────────────────────────────────────────────┐
│              用户代码                        │
│  chatClient.prompt().user("...").call()     │
└────────────┬────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│          ChatClientRequest                   │
│  包含: Prompt + Context                     │
└────────────┬────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│              Prompt                          │
│  ├─ messages: List<Message>                │
│  └─ chatOptions: ChatOptions                │
└────────────┬────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────┐
│           ChatModel                          │
│  OpenAI / Ollama / Azure / ...              │
└─────────────────────────────────────────────┘
```

### 1.3 核心类图

```
┌─────────────────────────────────────┐
│           Prompt                     │
│  - messages: List<Message>          │
│  - chatOptions: ChatOptions         │
│                                      │
│  + getSystemMessage()                │
│  + getUserMessage()                  │
│  + augmentSystemMessage()            │
│  + augmentUserMessage()              │
└──────────────┬──────────────────────┘
               │
               │ creates
               ▼
┌────────────────────────────────────────────┐
│          PromptTemplate                    │
│  - template: String                        │
│  - variables: Map<String, Object>         │
│  - renderer: TemplateRenderer             │
│                                            │
│  + render()                                │
│  + create()                                │
│  + createMessage()                         │
└─────────────┬──────────────────────────────┘
              │
              │ subclasses
              ▼
┌─────────────────────────────────────────────┐
│    SystemPromptTemplate                     │
│    UserPromptTemplate                       │
│    AssistantPromptTemplate                  │
│    ChatPromptTemplate                       │
└─────────────────────────────────────────────┘
```

---

## 二、Prompt类详解

### 2.1 核心接口定义

```java
package org.springframework.ai.chat.prompt;

public class Prompt implements ModelRequest<List<Message>> {
    
    private final List<Message> messages;
    
    @Nullable
    private ChatOptions chatOptions;
    
    // ... 构造方法和其他方法
}
```

### 2.2 多种构造方式

#### **方式1：简单字符串**

```java
// 自动转换为UserMessage
Prompt prompt = new Prompt("你好，请介绍一下Spring AI");

// 等价于
Prompt prompt = new Prompt(new UserMessage("你好，请介绍一下Spring AI"));
```

**使用场景**：最简单的单轮对话

#### **方式2：单个Message**

```java
// SystemMessage
Prompt prompt = new Prompt(
    new SystemMessage("你是一个Java专家")
);

// UserMessage
Prompt prompt = new Prompt(
    new UserMessage("解释一下Spring Boot")
);
```

**使用场景**：单一角色的简单提示

#### **方式3：多个Messages（可变参数）**

```java
Prompt prompt = new Prompt(
    new SystemMessage("你是一个友好的助手"),
    new UserMessage("你好")
);
```

**使用场景**：需要System + User的基本对话

#### **方式4：Message列表**

```java
List<Message> messages = List.of(
    new SystemMessage("你是一个Python专家"),
    new UserMessage("什么是装饰器？"),
    new AssistantMessage("装饰器是Python的一个特性..."),
    new UserMessage("能举个例子吗？")
);

Prompt prompt = new Prompt(messages);
```

**使用场景**：多轮对话、包含上下文的复杂交互

#### **方式5：带ChatOptions**

```java
// 字符串 + Options
Prompt prompt = new Prompt(
    "写一首诗",
    OpenAiChatOptions.builder()
        .model("gpt-4")
        .temperature(0.8)
        .maxTokens(500)
        .build()
);

// Message + Options
Prompt prompt = new Prompt(
    new UserMessage("写一首诗"),
    chatOptions
);

// Message列表 + Options
Prompt prompt = new Prompt(messages, chatOptions);
```

**使用场景**：需要自定义模型参数的场景

#### **方式6：Builder模式**

```java
Prompt prompt = Prompt.builder()
    .messages(
        new SystemMessage("你是一个助手"),
        new UserMessage("你好")
    )
    .chatOptions(OpenAiChatOptions.builder()
        .temperature(0.7)
        .build())
    .build();

// 或者使用content（自动转为UserMessage）
Prompt prompt = Prompt.builder()
    .content("你好")
    .chatOptions(chatOptions)
    .build();
```

**使用场景**：配置复杂的Prompt

### 2.3 核心方法详解

#### **获取消息方法**

```java
// 1. 获取第一个SystemMessage
SystemMessage systemMessage = prompt.getSystemMessage();
// 如果没有，返回空的SystemMessage("")

// 2. 获取最后一个UserMessage
UserMessage userMessage = prompt.getUserMessage();
// 如果没有，返回空的UserMessage("")

// 3. 获取所有UserMessage
List<UserMessage> userMessages = prompt.getUserMessages();

// 4. 获取所有消息
List<Message> allMessages = prompt.getInstructions();

// 5. 获取所有消息的文本内容（拼接）
String content = prompt.getContents();
```

**示例**：

```java
List<Message> messages = List.of(
    new SystemMessage("你是助手"),
    new UserMessage("问题1"),
    new AssistantMessage("回答1"),
    new UserMessage("问题2")
);
Prompt prompt = new Prompt(messages);

// 获取System消息
SystemMessage sys = prompt.getSystemMessage();
// → SystemMessage("你是助手")

// 获取最后一个User消息
UserMessage user = prompt.getUserMessage();
// → UserMessage("问题2")

// 获取所有User消息
List<UserMessage> users = prompt.getUserMessages();
// → [UserMessage("问题1"), UserMessage("问题2")]
```

#### **增强消息方法**

##### **augmentSystemMessage() - 增强系统消息**

```java
// 源码
public Prompt augmentSystemMessage(String newSystemText) {
    return augmentSystemMessage(
        systemMessage -> systemMessage.mutate()
            .text(newSystemText)
            .build()
    );
}

public Prompt augmentSystemMessage(
        Function<SystemMessage, SystemMessage> systemMessageAugmenter) {
    var messagesCopy = new ArrayList<>(this.messages);
    boolean found = false;
    
    for (int i = 0; i < messagesCopy.size(); i++) {
        Message message = messagesCopy.get(i);
        if (message instanceof SystemMessage systemMessage) {
            // 找到第一个SystemMessage，应用增强函数
            messagesCopy.set(i, systemMessageAugmenter.apply(systemMessage));
            found = true;
            break;
        }
    }
    
    if (!found) {
        // 如果没有SystemMessage，创建一个新的并添加到开头
        messagesCopy.add(0, systemMessageAugmenter.apply(new SystemMessage("")));
    }
    
    return new Prompt(messagesCopy, this.chatOptions.copy());
}
```

**使用示例**：

```java
// 示例1：简单字符串增强
Prompt originalPrompt = new Prompt(
    new SystemMessage("你是一个助手"),
    new UserMessage("你好")
);

Prompt newPrompt = originalPrompt.augmentSystemMessage(
    "你是一个友好的Python专家助手"
);

// 结果：
// messages = [
//   SystemMessage("你是一个友好的Python专家助手"),
//   UserMessage("你好")
// ]

// 示例2：使用函数增强（追加内容）
Prompt newPrompt = originalPrompt.augmentSystemMessage(
    sysMsg -> sysMsg.mutate()
        .text(sysMsg.getText() + "\n\n请用简洁的语言回答。")
        .build()
);

// 结果：
// messages = [
//   SystemMessage("你是一个助手\n\n请用简洁的语言回答。"),
//   UserMessage("你好")
// ]

// 示例3：如果原Prompt没有SystemMessage
Prompt promptWithoutSystem = new Prompt(
    new UserMessage("你好")
);

Prompt newPrompt = promptWithoutSystem.augmentSystemMessage(
    "你是一个助手"
);

// 结果：
// messages = [
//   SystemMessage("你是一个助手"),  // 新增在开头
//   UserMessage("你好")
// ]
```

##### **augmentUserMessage() - 增强用户消息**

```java
// 源码
public Prompt augmentUserMessage(String newUserText) {
    return augmentUserMessage(
        userMessage -> userMessage.mutate()
            .text(newUserText)
            .build()
    );
}

public Prompt augmentUserMessage(
        Function<UserMessage, UserMessage> userMessageAugmenter) {
    var messagesCopy = new ArrayList<>(this.messages);
    
    // 从后往前找最后一个UserMessage
    for (int i = messagesCopy.size() - 1; i >= 0; i--) {
        Message message = messagesCopy.get(i);
        if (message instanceof UserMessage userMessage) {
            messagesCopy.set(i, userMessageAugmenter.apply(userMessage));
            break;
        }
        if (i == 0) {
            // 如果没找到，添加新的UserMessage到末尾
            messagesCopy.add(userMessageAugmenter.apply(new UserMessage("")));
        }
    }
    
    return new Prompt(messagesCopy, this.chatOptions.copy());
}
```

**使用示例**：

```java
// 示例1：追加格式要求
Prompt originalPrompt = new Prompt(
    new SystemMessage("你是助手"),
    new UserMessage("解释Spring Boot")
);

Prompt newPrompt = originalPrompt.augmentUserMessage(
    userMsg -> userMsg.mutate()
        .text(userMsg.getText() + "\n\n请用JSON格式回答。")
        .build()
);

// 结果：
// messages = [
//   SystemMessage("你是助手"),
//   UserMessage("解释Spring Boot\n\n请用JSON格式回答。")
// ]

// 示例2：替换用户消息
Prompt newPrompt = originalPrompt.augmentUserMessage(
    "详细解释Spring Boot的自动配置原理"
);

// 结果：
// messages = [
//   SystemMessage("你是助手"),
//   UserMessage("详细解释Spring Boot的自动配置原理")
// ]
```

**实际应用场景**：

```java
// 场景：ChatClient中自动添加格式说明
ChatClientRequest formattedRequest = 
    chatClientRequest.mutate()
        .prompt(chatClientRequest.prompt()
            .augmentUserMessage(userMsg -> userMsg.mutate()
                .text(userMsg.getText() + 
                      System.lineSeparator() + 
                      outputFormat)  // 添加JSON Schema等
                .build()))
        .build();
```

#### **复制方法**

```java
// 深拷贝Prompt
Prompt copy = prompt.copy();
```

**源码实现**：

```java
public Prompt copy() {
    return new Prompt(
        instructionsCopy(),  // 深拷贝所有Message
        this.chatOptions == null ? null : this.chatOptions.copy()
    );
}

private List<Message> instructionsCopy() {
    List<Message> messagesCopy = new ArrayList<>();
    this.messages.forEach(message -> {
        if (message instanceof UserMessage userMessage) {
            messagesCopy.add(userMessage.copy());
        }
        else if (message instanceof SystemMessage systemMessage) {
            messagesCopy.add(systemMessage.copy());
        }
        else if (message instanceof AssistantMessage assistantMessage) {
            messagesCopy.add(AssistantMessage.builder()
                .content(assistantMessage.getText())
                .properties(assistantMessage.getMetadata())
                .toolCalls(assistantMessage.getToolCalls())
                .build());
        }
        else if (message instanceof ToolResponseMessage toolResponseMessage) {
            messagesCopy.add(new ToolResponseMessage(
                new ArrayList<>(toolResponseMessage.getResponses()),
                new HashMap<>(toolResponseMessage.getMetadata())
            ));
        }
        else {
            throw new IllegalArgumentException(
                "Unsupported message type: " + message.getClass().getName()
            );
        }
    });
    return messagesCopy;
}
```

**使用场景**：

```java
// 场景：需要基于原Prompt创建变体
Prompt basePrompt = new Prompt(
    new SystemMessage("你是助手"),
    new UserMessage("解释Spring")
);

// 创建变体1：高创造性
Prompt creativePrompt = basePrompt.copy();
creativePrompt = Prompt.builder()
    .messages(creativePrompt.getInstructions())
    .chatOptions(OpenAiChatOptions.builder()
        .temperature(0.9)
        .build())
    .build();

// 创建变体2：低创造性
Prompt conservativePrompt = basePrompt.copy();
conservativePrompt = Prompt.builder()
    .messages(conservativePrompt.getInstructions())
    .chatOptions(OpenAiChatOptions.builder()
        .temperature(0.1)
        .build())
    .build();
```

#### **mutate() - 可变Builder**

```java
// 基于现有Prompt创建Builder
Prompt.Builder builder = prompt.mutate();

// 修改后构建新的Prompt
Prompt newPrompt = builder
    .messages(newMessages)
    .chatOptions(newOptions)
    .build();
```

**使用场景**：

```java
// 场景：在Advisor中修改Prompt
public ChatClientRequest before(
        ChatClientRequest request, 
        AdvisorChain chain) {
    
    // 获取原Prompt
    Prompt originalPrompt = request.prompt();
    
    // 通过mutate()修改
    Prompt newPrompt = originalPrompt.mutate()
        .messages(/* 添加历史消息 */)
        .build();
    
    return request.mutate()
        .prompt(newPrompt)
        .build();
}
```

---

## 三、PromptTemplate模板系统

### 3.1 PromptTemplate核心概念

**PromptTemplate**提供参数化Prompt的能力，支持：
- **变量占位符**：`{name}`、`{age}`等
- **模板渲染**：将变量替换为实际值
- **多种输出**：String、Message、Prompt

### 3.2 基础用法

#### **创建和渲染**

```java
// 1. 创建模板
PromptTemplate template = new PromptTemplate(
    "你好，{name}！今天是{date}，天气{weather}。"
);

// 2. 方式1：添加变量后渲染
template.add("name", "张三");
String rendered = template.render(Map.of(
    "date", "2025年10月5日",
    "weather", "晴朗"
));
// → "你好，张三！今天是2025年10月5日，天气晴朗。"

// 3. 方式2：直接传入所有变量
String rendered = template.render(Map.of(
    "name", "张三",
    "date", "2025年10月5日",
    "weather", "晴朗"
));
```

#### **从Resource创建**

```java
// 从classpath加载模板文件
Resource resource = new ClassPathResource("prompts/greeting.txt");
PromptTemplate template = new PromptTemplate(resource);

// greeting.txt内容：
// 你好，{name}！
// 欢迎来到{company}。
// 今天我们将讨论{topic}。

String rendered = template.render(Map.of(
    "name", "李四",
    "company", "Spring AI",
    "topic", "提示词工程"
));
```

#### **使用Builder**

```java
PromptTemplate template = PromptTemplate.builder()
    .template("分析以下代码：\n{code}\n\n语言：{language}")
    .variables(Map.of(
        "language", "Java"
    ))
    .build();

String rendered = template.render(Map.of(
    "code", "public class Hello { ... }"
));
```

### 3.3 输出形式

#### **1. 渲染为String**

```java
String text = template.render();
String text = template.render(variables);
```

#### **2. 创建Message**

```java
// 默认创建UserMessage
Message message = template.createMessage();
// 等价于: new UserMessage(template.render())

Message message = template.createMessage(variables);

// 带Media（多模态）
List<Media> mediaList = List.of(
    new Media(MimeTypeUtils.IMAGE_PNG, imageUrl)
);
Message message = template.createMessage(mediaList);
```

#### **3. 创建Prompt**

```java
// 不带Options
Prompt prompt = template.create();
Prompt prompt = template.create(variables);

// 带Options
Prompt prompt = template.create(chatOptions);
Prompt prompt = template.create(variables, chatOptions);
```

**完整示例**：

```java
PromptTemplate template = new PromptTemplate(
    "你是{role}。请{task}：{content}"
);

// 方式1：渲染为字符串
String text = template.render(Map.of(
    "role", "Java专家",
    "task", "解释",
    "content", "Spring Boot自动配置"
));
// → "你是Java专家。请解释：Spring Boot自动配置"

// 方式2：创建Message
UserMessage message = (UserMessage) template.createMessage(Map.of(
    "role", "Java专家",
    "task", "解释",
    "content", "Spring Boot自动配置"
));

// 方式3：创建Prompt
Prompt prompt = template.create(
    Map.of(
        "role", "Java专家",
        "task", "解释",
        "content", "Spring Boot自动配置"
    ),
    OpenAiChatOptions.builder()
        .temperature(0.7)
        .build()
);
```

### 3.4 高级特性

#### **Resource变量处理**

```java
// 模板中的变量可以是Resource
Resource codeFile = new ClassPathResource("code/Example.java");

PromptTemplate template = new PromptTemplate(
    "请分析以下代码：\n\n{code}\n\n找出潜在的问题。"
);

// Resource会自动读取文件内容
String rendered = template.render(Map.of(
    "code", codeFile
));
```

**源码实现**：

```java
private String renderResource(Resource resource) {
    if (resource == null) {
        return "";
    }
    
    try {
        // 特殊处理ByteArrayResource
        if (resource instanceof ByteArrayResource byteArrayResource) {
            return new String(
                byteArrayResource.getByteArray(), 
                StandardCharsets.UTF_8
            );
        }
        
        // 检查资源是否存在和非空
        if (!resource.exists() || resource.contentLength() == 0) {
            return "";
        }
        
        // 读取资源内容
        return resource.getContentAsString(StandardCharsets.UTF_8);
    }
    catch (IOException e) {
        log.warn("Failed to render resource: {}", 
                 resource.getDescription(), e);
        return "[Unable to render resource: " + 
               resource.getDescription() + "]";
    }
}
```

#### **自定义TemplateRenderer**

```java
// 默认使用StringTemplate 4引擎
TemplateRenderer defaultRenderer = StTemplateRenderer.builder().build();

// 自定义Renderer
TemplateRenderer customRenderer = new TemplateRenderer() {
    @Override
    public String apply(String template, Map<String, Object> variables) {
        // 自定义模板渲染逻辑
        // 例如使用Mustache、Velocity等
        return /* rendered content */;
    }
};

PromptTemplate template = PromptTemplate.builder()
    .template("{name} says: {message}")
    .renderer(customRenderer)
    .build();
```

### 3.5 SystemPromptTemplate

**专门用于创建SystemMessage**

```java
SystemPromptTemplate template = new SystemPromptTemplate(
    "你是一个{expertise}专家。你的任务是{task}。"
);

// 创建SystemMessage
SystemMessage message = (SystemMessage) template.createMessage(Map.of(
    "expertise", "Python",
    "task", "帮助用户调试代码"
));

// 或者直接创建Prompt
Prompt prompt = template.create(Map.of(
    "expertise", "Python",
    "task", "帮助用户调试代码"
));
// prompt.messages = [SystemMessage(...)]
```

**与PromptTemplate的区别**：

| 特性 | PromptTemplate | SystemPromptTemplate |
|------|----------------|----------------------|
| 默认Message类型 | UserMessage | SystemMessage |
| createMessage() | 返回UserMessage | 返回SystemMessage |
| 使用场景 | 用户输入 | 系统指令 |

### 3.6 UserPromptTemplate和AssistantPromptTemplate

```java
// UserPromptTemplate（实际上就是PromptTemplate）
UserPromptTemplate userTemplate = new UserPromptTemplate(
    "我想了解{topic}"
);
UserMessage userMsg = (UserMessage) userTemplate.createMessage(
    Map.of("topic", "Spring Cloud")
);

// AssistantPromptTemplate
AssistantPromptTemplate assistantTemplate = new AssistantPromptTemplate(
    "关于{topic}，我可以告诉你..."
);
AssistantMessage assistantMsg = (AssistantMessage) assistantTemplate.createMessage(
    Map.of("topic", "Spring Cloud")
);
```

---

## 四、Message体系

### 4.1 Message类型层次

```
Message (抽象基类)
│
├── SystemMessage          系统提示词
│   └── 定义AI的角色和行为规则
│
├── UserMessage            用户消息
│   ├── text: String       文本内容
│   └── media: List<Media> 多模态内容（图片、音频等）
│
├── AssistantMessage       AI助手回复
│   ├── text: String       回复文本
│   └── toolCalls: List    工具调用请求
│
└── ToolResponseMessage    工具调用结果
    └── responses: List    工具执行结果
```

### 4.2 SystemMessage - 系统提示词

**作用**：定义AI的角色、行为和规则

```java
// 基础创建
SystemMessage msg = new SystemMessage("你是一个友好的助手");

// 使用Builder
SystemMessage msg = SystemMessage.builder()
    .text("你是一个Python专家")
    .metadata(Map.of("version", "1.0"))
    .build();

// 从PromptTemplate创建
SystemPromptTemplate template = new SystemPromptTemplate(
    """
    你是一个{role}。
    你的专长是{expertise}。
    你应该{behavior}。
    """
);

SystemMessage msg = (SystemMessage) template.createMessage(Map.of(
    "role", "技术顾问",
    "expertise", "云原生架构",
    "behavior", "提供专业且易懂的建议"
));
```

**典型模式**：

```java
// 模式1：角色定义
SystemMessage msg = new SystemMessage(
    """
    你是一个专业的Java代码审查员。
    你的职责是：
    1. 检查代码质量
    2. 发现潜在bug
    3. 提供改进建议
    """
);

// 模式2：行为约束
SystemMessage msg = new SystemMessage(
    """
    你是一个客服助手。
    
    规则：
    - 始终保持礼貌和专业
    - 不要透露公司内部信息
    - 如果不确定，建议联系人工客服
    - 每次回复不超过200字
    """
);

// 模式3：输出格式要求
SystemMessage msg = new SystemMessage(
    """
    你是一个数据分析师。
    
    输出要求：
    - 使用JSON格式
    - 包含分析结论和置信度
    - 格式：{"conclusion": "...", "confidence": 0.95}
    """
);
```

### 4.3 UserMessage - 用户消息

**作用**：表示用户的输入，可包含文本和多模态内容

#### **文本消息**

```java
// 简单文本
UserMessage msg = new UserMessage("什么是Spring Boot？");

// 使用Builder
UserMessage msg = UserMessage.builder()
    .text("解释一下Spring Boot的自动配置原理")
    .metadata(Map.of("requestId", "req-123"))
    .build();
```

#### **多模态消息**

```java
// 文本 + 图片
UserMessage msg = UserMessage.builder()
    .text("这张图片中的代码有什么问题？")
    .media(new Media(
        MimeTypeUtils.IMAGE_PNG, 
        new URL("https://example.com/code.png")
    ))
    .build();

// 文本 + 多张图片
UserMessage msg = UserMessage.builder()
    .text("比较这两张架构图的区别")
    .media(
        new Media(MimeTypeUtils.IMAGE_PNG, imageUrl1),
        new Media(MimeTypeUtils.IMAGE_PNG, imageUrl2)
    )
    .build();

// 使用Base64编码的图片
String base64Image = "iVBORw0KGgo...";
UserMessage msg = UserMessage.builder()
    .text("分析这张图片")
    .media(new Media(
        MimeTypeUtils.IMAGE_PNG,
        Base64.getDecoder().decode(base64Image)
    ))
    .build();
```

### 4.4 AssistantMessage - AI回复

```java
// 简单回复
AssistantMessage msg = new AssistantMessage("Spring Boot是一个...");

// 使用Builder
AssistantMessage msg = AssistantMessage.builder()
    .content("Spring Boot是一个简化Spring应用开发的框架...")
    .properties(Map.of("confidence", 0.95))
    .build();

// 带Tool调用的回复
AssistantMessage msg = AssistantMessage.builder()
    .content("让我查询一下天气信息...")
    .toolCalls(List.of(
        new AssistantMessage.ToolCall(
            "get_weather",
            "{\"location\": \"北京\"}"
        )
    ))
    .build();
```

### 4.5 ToolResponseMessage - 工具响应

```java
ToolResponseMessage msg = new ToolResponseMessage(
    List.of(
        new ToolResponseMessage.ToolResponse(
            "tool_call_123",
            "get_weather",
            "{\"temperature\": 22, \"condition\": \"晴\"}"
        )
    )
);
```

### 4.6 消息的复制和修改

#### **copy() - 复制消息**

```java
UserMessage original = new UserMessage("你好");
UserMessage copy = original.copy();
```

#### **mutate() - 可变Builder**

```java
UserMessage original = new UserMessage("你好");

// 修改文本
UserMessage modified = original.mutate()
    .text("你好，我想了解Spring AI")
    .build();

// 添加metadata
UserMessage withMetadata = original.mutate()
    .metadata(Map.of("userId", "user-123"))
    .build();

// 添加media
UserMessage withMedia = original.mutate()
    .media(new Media(MimeTypeUtils.IMAGE_PNG, imageUrl))
    .build();
```

---

## 五、高级特性

### 5.1 ChatPromptTemplate - 多角色模板

**ChatPromptTemplate**用于组合多个不同角色的模板

```java
// 创建多个PromptTemplate
SystemPromptTemplate systemTemplate = new SystemPromptTemplate(
    "你是一个{role}专家"
);

UserPromptTemplate userTemplate = new UserPromptTemplate(
    "请解释{concept}"
);

// 组合为ChatPromptTemplate
ChatPromptTemplate chatTemplate = new ChatPromptTemplate(
    List.of(systemTemplate, userTemplate)
);

// 渲染所有模板
List<Message> messages = chatTemplate.createMessages(Map.of(
    "role", "Java",
    "concept", "Spring Bean生命周期"
));
// 结果：
// [
//   SystemMessage("你是一个Java专家"),
//   UserMessage("请解释Spring Bean生命周期")
// ]

// 创建Prompt
Prompt prompt = chatTemplate.create(
    Map.of("role", "Java", "concept", "Spring Bean生命周期"),
    chatOptions
);
```

**实战示例：Few-shot Learning**

```java
// 构建Few-shot示例
SystemPromptTemplate systemTemplate = new SystemPromptTemplate(
    "你是一个代码生成助手，根据需求生成代码"
);

UserPromptTemplate example1User = new UserPromptTemplate(
    "创建一个计算两数之和的函数"
);
AssistantPromptTemplate example1Assistant = new AssistantPromptTemplate(
    """
    ```python
    def add(a, b):
        return a + b
    ```
    """
);

UserPromptTemplate example2User = new UserPromptTemplate(
    "创建一个判断数字是否为偶数的函数"
);
AssistantPromptTemplate example2Assistant = new AssistantPromptTemplate(
    """
    ```python
    def is_even(n):
        return n % 2 == 0
    ```
    """
);

// 实际请求
UserPromptTemplate actualRequest = new UserPromptTemplate(
    "{request}"
);

// 组合所有模板
ChatPromptTemplate fewShotTemplate = new ChatPromptTemplate(List.of(
    systemTemplate,
    example1User,
    example1Assistant,
    example2User,
    example2Assistant,
    actualRequest
));

// 使用
Prompt prompt = fewShotTemplate.create(Map.of(
    "request", "创建一个计算列表平均值的函数"
));
```

### 5.2 动态Prompt构建

#### **场景1：根据条件添加消息**

```java
public Prompt buildPrompt(String userInput, boolean includeContext) {
    List<Message> messages = new ArrayList<>();
    
    // 始终添加System消息
    messages.add(new SystemMessage("你是一个助手"));
    
    // 条件添加上下文
    if (includeContext) {
        messages.add(new UserMessage("上下文：这是关于Spring AI的讨论"));
    }
    
    // 添加用户输入
    messages.add(new UserMessage(userInput));
    
    return new Prompt(messages);
}
```

#### **场景2：动态选择System提示词**

```java
public Prompt buildPromptForRole(String userInput, String role) {
    Map<String, String> rolePrompts = Map.of(
        "developer", "你是一个经验丰富的软件开发工程师",
        "architect", "你是一个精通系统架构设计的架构师",
        "tester", "你是一个注重质量的测试工程师"
    );
    
    String systemPrompt = rolePrompts.getOrDefault(role, "你是一个通用助手");
    
    return new Prompt(
        new SystemMessage(systemPrompt),
        new UserMessage(userInput)
    );
}
```

#### **场景3：基于历史构建Prompt**

```java
public Prompt buildPromptWithHistory(
        String userInput, 
        List<Message> history) {
    
    List<Message> messages = new ArrayList<>();
    
    // System消息
    messages.add(new SystemMessage("你是一个助手"));
    
    // 添加历史（最多保留10条）
    int historySize = Math.min(history.size(), 10);
    messages.addAll(history.subList(
        Math.max(0, history.size() - historySize),
        history.size()
    ));
    
    // 当前用户输入
    messages.add(new UserMessage(userInput));
    
    return new Prompt(messages);
}
```

### 5.3 Prompt验证和优化

#### **验证Prompt长度**

```java
public class PromptValidator {
    
    private static final int MAX_TOKENS = 4000;
    
    public boolean validatePromptLength(Prompt prompt) {
        // 简单估算：1个token ≈ 0.75个英文单词或1个中文字
        int estimatedTokens = 0;
        
        for (Message message : prompt.getInstructions()) {
            String text = message.getText();
            // 中文字符数
            int chineseChars = text.replaceAll("[^\\u4e00-\\u9fa5]", "").length();
            // 英文单词数
            int englishWords = text.split("\\s+").length - chineseChars;
            
            estimatedTokens += chineseChars + (int)(englishWords * 0.75);
        }
        
        return estimatedTokens <= MAX_TOKENS;
    }
    
    public Prompt truncateIfNeeded(Prompt prompt) {
        if (validatePromptLength(prompt)) {
            return prompt;
        }
        
        // 裁剪历史消息，保留System和最新的User消息
        List<Message> messages = new ArrayList<>();
        SystemMessage systemMessage = prompt.getSystemMessage();
        if (systemMessage != null && !systemMessage.getText().isEmpty()) {
            messages.add(systemMessage);
        }
        
        UserMessage userMessage = prompt.getUserMessage();
        messages.add(userMessage);
        
        return new Prompt(messages, prompt.getOptions());
    }
}
```

---

## 六、完整实战案例

### 6.1 智能代码审查助手

```java
@Service
public class CodeReviewService {
    
    private final ChatModel chatModel;
    
    public CodeReviewService(ChatModel chatModel) {
        this.chatModel = chatModel;
    }
    
    public String reviewCode(String code, String language) {
        // 创建System提示词
        SystemPromptTemplate systemTemplate = new SystemPromptTemplate(
            """
            你是一个专业的{language}代码审查员。
            
            审查标准：
            1. 代码质量和可读性
            2. 性能问题
            3. 安全隐患
            4. 最佳实践
            
            输出格式：
            - 总体评分（1-10分）
            - 问题列表（按严重程度排序）
            - 改进建议
            """
        );
        
        // 创建User提示词
        UserPromptTemplate userTemplate = new UserPromptTemplate(
            """
            请审查以下{language}代码：
            
            ```{language}
            {code}
            ```
            """
        );
        
        // 组合模板
        ChatPromptTemplate chatTemplate = new ChatPromptTemplate(
            List.of(systemTemplate, userTemplate)
        );
        
        // 创建Prompt
        Prompt prompt = chatTemplate.create(
            Map.of(
                "language", language,
                "code", code
            ),
            OpenAiChatOptions.builder()
                .model("gpt-4")
                .temperature(0.3)  // 低温度，更客观
                .build()
        );
        
        // 调用AI
        ChatResponse response = chatModel.call(prompt);
        return response.getResult().getOutput().getText();
    }
}
```

**使用**：

```java
String javaCode = """
    public class UserService {
        private List<User> users = new ArrayList<>();
        
        public void addUser(User user) {
            users.add(user);
        }
        
        public User findUser(String name) {
            for (User u : users) {
                if (u.getName().equals(name)) {
                    return u;
                }
            }
            return null;
        }
    }
    """;

String review = codeReviewService.reviewCode(javaCode, "Java");
```

### 6.2 多语言翻译服务

```java
@Service
public class TranslationService {
    
    private final ChatModel chatModel;
    
    public String translate(
            String text, 
            String sourceLanguage, 
            String targetLanguage,
            TranslationStyle style) {
        
        // 根据风格选择System提示词
        String styleInstruction = switch (style) {
            case FORMAL -> "使用正式、专业的语言";
            case CASUAL -> "使用轻松、口语化的表达";
            case TECHNICAL -> "保持技术术语的准确性";
            case LITERARY -> "注重文学性和美感";
        };
        
        SystemPromptTemplate systemTemplate = new SystemPromptTemplate(
            """
            你是一个专业的{sourceLanguage}到{targetLanguage}翻译。
            
            翻译要求：
            - {styleInstruction}
            - 保持原文的语气和情感
            - 注意文化差异
            - 只输出翻译结果，不要解释
            """
        );
        
        UserPromptTemplate userTemplate = new UserPromptTemplate(
            "请翻译：{text}"
        );
        
        ChatPromptTemplate chatTemplate = new ChatPromptTemplate(
            List.of(systemTemplate, userTemplate)
        );
        
        Prompt prompt = chatTemplate.create(Map.of(
            "sourceLanguage", sourceLanguage,
            "targetLanguage", targetLanguage,
            "styleInstruction", styleInstruction,
            "text", text
        ));
        
        ChatResponse response = chatModel.call(prompt);
        return response.getResult().getOutput().getText();
    }
    
    public enum TranslationStyle {
        FORMAL, CASUAL, TECHNICAL, LITERARY
    }
}
```

### 6.3 智能文档问答（RAG简化版）

```java
@Service
public class DocumentQAService {
    
    private final ChatModel chatModel;
    
    public String answerQuestion(
            String question, 
            List<String> relevantDocuments) {
        
        // 构建上下文
        StringBuilder context = new StringBuilder();
        for (int i = 0; i < relevantDocuments.size(); i++) {
            context.append(String.format(
                "文档%d：\n%s\n\n", 
                i + 1, 
                relevantDocuments.get(i)
            ));
        }
        
        // System提示词
        SystemMessage systemMsg = new SystemMessage(
            """
            你是一个文档问答助手。
            
            规则：
            1. 只基于提供的文档回答问题
            2. 如果文档中没有相关信息，明确告知
            3. 引用文档时标注文档编号
            4. 回答要准确且完整
            """
        );
        
        // User提示词（使用模板）
        UserPromptTemplate userTemplate = new UserPromptTemplate(
            """
            文档内容：
            {context}
            
            问题：{question}
            """
        );
        
        UserMessage userMsg = (UserMessage) userTemplate.createMessage(Map.of(
            "context", context.toString(),
            "question", question
        ));
        
        // 创建Prompt
        Prompt prompt = new Prompt(
            List.of(systemMsg, userMsg),
            OpenAiChatOptions.builder()
                .temperature(0.1)  // 低温度，更准确
                .build()
        );
        
        ChatResponse response = chatModel.call(prompt);
        return response.getResult().getOutput().getText();
    }
}
```

### 6.4 动态Few-shot学习

```java
@Service
public class FewShotLearningService {
    
    private final ChatModel chatModel;
    
    public String generateWithExamples(
            String task,
            List<Example> examples,
            String input) {
        
        List<Message> messages = new ArrayList<>();
        
        // System消息
        messages.add(new SystemMessage(
            String.format("你的任务是：%s", task)
        ));
        
        // 添加示例（Few-shot）
        for (Example example : examples) {
            messages.add(new UserMessage(example.input()));
            messages.add(new AssistantMessage(example.output()));
        }
        
        // 实际请求
        messages.add(new UserMessage(input));
        
        Prompt prompt = new Prompt(messages);
        ChatResponse response = chatModel.call(prompt);
        return response.getResult().getOutput().getText();
    }
    
    record Example(String input, String output) {}
}

// 使用
List<Example> examples = List.of(
    new Example(
        "将'Hello World'转换为Camel Case",
        "helloWorld"
    ),
    new Example(
        "将'spring boot application'转换为Camel Case",
        "springBootApplication"
    )
);

String result = service.generateWithExamples(
    "将文本转换为Camel Case格式",
    examples,
    "open ai chat model"
);
// → "openAiChatModel"
```

---

## 七、最佳实践

### 7.1 Prompt设计原则

#### **原则1：清晰具体**

```java
// ❌ 不好：模糊不清
Prompt bad = new Prompt("解释Spring");

// ✅ 好：清晰具体
Prompt good = new Prompt(
    new SystemMessage("你是一个Java框架专家"),
    new UserMessage("""
        请用300字左右解释Spring Framework的核心特性，
        重点说明依赖注入（DI）和面向切面编程（AOP）。
        请使用简单易懂的语言，并提供一个简单示例。
        """)
);
```

#### **原则2：角色定义**

```java
// ✅ 明确定义AI的角色
SystemMessage roleDefinition = new SystemMessage("""
    你是一个高级Java架构师，拥有15年的企业级应用开发经验。
    你擅长：
    - Spring生态系统
    - 微服务架构
    - 云原生技术
    
    你的回答风格：
    - 专业但易懂
    - 提供实际可行的建议
    - 包含代码示例
    """);
```

#### **原则3：示例引导**

```java
// ✅ 使用Few-shot提供示例
Prompt fewShot = new Prompt(
    new SystemMessage("将英文技术术语翻译成中文"),
    // 示例1
    new UserMessage("Dependency Injection"),
    new AssistantMessage("依赖注入"),
    // 示例2
    new UserMessage("Aspect-Oriented Programming"),
    new AssistantMessage("面向切面编程"),
    // 实际请求
    new UserMessage("Inversion of Control")
);
```

#### **原则4：输出格式约束**

```java
// ✅ 明确输出格式
SystemMessage formatConstraint = new SystemMessage("""
    你的回答必须使用以下JSON格式：
    {
      "summary": "一句话总结",
      "explanation": "详细解释",
      "example": "代码示例",
      "references": ["参考链接1", "参考链接2"]
    }
    """);
```

### 7.2 性能优化

#### **1. Prompt缓存**

```java
@Service
public class PromptCacheService {
    
    private final Cache<String, Prompt> promptCache = Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(1, TimeUnit.HOURS)
        .build();
    
    public Prompt getOrCreatePrompt(String key, Supplier<Prompt> creator) {
        return promptCache.get(key, k -> creator.get());
    }
}

// 使用
Prompt cached = promptCacheService.getOrCreatePrompt(
    "code-review-java",
    () -> buildCodeReviewPrompt("Java")
);
```

#### **2. 模板复用**

```java
@Configuration
public class PromptTemplateConfig {
    
    @Bean
    public SystemPromptTemplate codeReviewSystemTemplate() {
        return new SystemPromptTemplate(
            """
            你是一个{language}代码审查专家。
            请按照以下标准审查代码：
            {standards}
            """
        );
    }
    
    @Bean
    public UserPromptTemplate codeReviewUserTemplate() {
        return new UserPromptTemplate(
            "请审查以下代码：\n{code}"
        );
    }
}
```

#### **3. 批量处理**

```java
public List<String> batchTranslate(List<String> texts) {
    // 合并为单个Prompt，减少API调用
    StringBuilder combined = new StringBuilder();
    for (int i = 0; i < texts.size(); i++) {
        combined.append(String.format("%d. %s\n", i + 1, texts.get(i)));
    }
    
    Prompt prompt = new Prompt(
        new SystemMessage("将以下文本逐条翻译成英文，保持编号"),
        new UserMessage(combined.toString())
    );
    
    String response = chatModel.call(prompt)
        .getResult()
        .getOutput()
        .getText();
    
    // 解析响应
    return parseNumberedResponses(response);
}
```

### 7.3 错误处理

```java
public class RobustPromptService {
    
    private final ChatModel chatModel;
    
    public String safeCall(Prompt prompt) {
        try {
            // 验证Prompt
            validatePrompt(prompt);
            
            // 调用AI
            ChatResponse response = chatModel.call(prompt);
            
            // 验证响应
            String content = response.getResult().getOutput().getText();
            if (content == null || content.trim().isEmpty()) {
                throw new IllegalStateException("Empty response");
            }
            
            return content;
            
        } catch (Exception e) {
            logger.error("Failed to call AI model", e);
            
            // 降级策略
            return fallbackResponse(prompt);
        }
    }
    
    private void validatePrompt(Prompt prompt) {
        if (prompt.getInstructions().isEmpty()) {
            throw new IllegalArgumentException("Prompt has no messages");
        }
        
        // 检查长度
        int totalLength = prompt.getContents().length();
        if (totalLength > 10000) {
            throw new IllegalArgumentException("Prompt too long: " + totalLength);
        }
    }
    
    private String fallbackResponse(Prompt prompt) {
        return "抱歉，系统暂时无法处理您的请求，请稍后再试。";
    }
}
```

### 7.4 测试策略

```java
@SpringBootTest
class PromptTemplateTest {
    
    @Test
    void testPromptTemplateRendering() {
        PromptTemplate template = new PromptTemplate(
            "Hello, {name}! You are {age} years old."
        );
        
        String rendered = template.render(Map.of(
            "name", "Alice",
            "age", 25
        ));
        
        assertEquals("Hello, Alice! You are 25 years old.", rendered);
    }
    
    @Test
    void testPromptCreation() {
        SystemMessage systemMsg = new SystemMessage("You are a helper");
        UserMessage userMsg = new UserMessage("Hello");
        
        Prompt prompt = new Prompt(systemMsg, userMsg);
        
        assertEquals(2, prompt.getInstructions().size());
        assertEquals("You are a helper", 
                     prompt.getSystemMessage().getText());
        assertEquals("Hello", 
                     prompt.getUserMessage().getText());
    }
    
    @Test
    void testPromptAugmentation() {
        Prompt original = new Prompt(
            new SystemMessage("Original system message"),
            new UserMessage("Original user message")
        );
        
        Prompt augmented = original.augmentSystemMessage(
            "New system message"
        );
        
        assertEquals("New system message", 
                     augmented.getSystemMessage().getText());
        assertEquals("Original user message", 
                     augmented.getUserMessage().getText());
    }
}
```

---

## 总结

### 核心要点

1. **Prompt是AI请求的完整封装**
   - Messages：消息列表
   - ChatOptions：配置选项

2. **PromptTemplate提供参数化能力**
   - 变量占位符
   - 多种输出形式
   - Resource支持

3. **Message体系丰富**
   - SystemMessage：角色定义
   - UserMessage：用户输入（支持多模态）
   - AssistantMessage：AI回复
   - ToolResponseMessage：工具结果

4. **高级特性强大**
   - ChatPromptTemplate：多角色组合
   - augment方法：动态修改
   - mutate：可变Builder

### 学习路径

```
Level 1: 基础
  ├─ 理解Prompt的概念和作用
  ├─ 掌握多种Prompt创建方式
  └─ 熟悉Message体系

Level 2: 模板
  ├─ 掌握PromptTemplate基础用法
  ├─ 学习SystemPromptTemplate等变体
  └─ 理解模板渲染机制

Level 3: 高级
  ├─ ChatPromptTemplate组合使用
  ├─ 动态Prompt构建
  └─ Prompt增强技巧

Level 4: 实战
  ├─ Few-shot Learning
  ├─ RAG集成
  └─ 性能优化和错误处理
```

### 推荐实践

1. ✅ 使用SystemMessage明确定义角色
2. ✅ 使用PromptTemplate实现复用
3. ✅ 提供Few-shot示例引导
4. ✅ 明确指定输出格式
5. ✅ 验证Prompt长度避免超限
6. ✅ 添加错误处理和降级策略

恭喜你！现在你已经完全掌握了Spring AI的Prompt体系！🎉

