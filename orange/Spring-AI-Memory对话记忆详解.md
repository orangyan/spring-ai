# Spring AI - Memory对话记忆详解

## 目录

- [一、核心概念](#一核心概念)
- [二、架构设计](#二架构设计)
- [三、核心接口](#三核心接口)
- [四、内存实现](#四内存实现)
- [五、存储实现](#五存储实现)
- [六、Memory Advisor](#六memory-advisor)
- [七、完整实战案例](#七完整实战案例)
- [八、最佳实践](#八最佳实践)

---

## 一、核心概念

### 1.1 为什么需要Memory？

**LLM的无状态特性**：
- 大语言模型本身是**无状态**的，不会记住之前的对话
- 每次请求都是独立的，无法维持对话上下文
- 需要通过外部机制来管理对话历史

**Spring AI的解决方案**：
```
┌─────────────────────────────────────────────┐
│          对话记忆管理 (Memory)               │
├─────────────────────────────────────────────┤
│  • 存储历史消息                              │
│  • 管理对话上下文                            │
│  • 支持多会话隔离                            │
│  • 自动裁剪和清理                            │
└─────────────────────────────────────────────┘
```

### 1.2 Chat Memory vs Chat History

Spring AI区分了两个重要概念：

| 概念 | 定义 | 用途 |
|------|------|------|
| **Chat Memory** | LLM用于维持上下文的信息 | 提供给AI模型的对话记忆 |
| **Chat History** | 完整的对话记录 | 用户查看、审计、分析 |

**关键区别**：
- `ChatMemory`：**选择性存储**，只保留相关的消息（如最近N条）
- `Chat History`：**完整存储**，记录所有交互（建议用Spring Data）

---

## 二、架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      ChatClient                              │
│                     (对话客户端)                              │
└──────────────────────┬──────────────────────────────────────┘
                       │ uses
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   Memory Advisor                             │
│  ┌──────────────────┬──────────────────┬──────────────────┐ │
│  │ Message          │ Prompt           │ VectorStore      │ │
│  │ ChatMemoryAdvisor│ ChatMemoryAdvisor│ ChatMemoryAdvisor│ │
│  └──────────────────┴──────────────────┴──────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │ uses
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    ChatMemory                                │
│              (内存管理策略接口)                                │
│  ┌──────────────────┬──────────────────┬──────────────────┐ │
│  │ Message          │ Token            │ Summary          │ │
│  │ WindowChatMemory │ WindowChatMemory │ ChatMemory       │ │
│  └──────────────────┴──────────────────┴──────────────────┘ │
└──────────────────────┬──────────────────────────────────────┘
                       │ uses
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              ChatMemoryRepository                            │
│                (存储持久化接口)                                │
│  ┌────────┬────────┬────────┬────────┬────────┬──────────┐ │
│  │InMemory│ JDBC   │Cassandra│ Neo4j │CosmosDB│  Redis   │ │
│  └────────┴────────┴────────┴────────┴────────┴──────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 分层职责

#### **第一层：Advisor（使用层）**
- 负责将Memory集成到对话流程
- 提供三种集成策略：消息列表、系统提示词、向量检索

#### **第二层：ChatMemory（策略层）**
- 定义内存管理策略
- 决定保留哪些消息、何时清理
- 处理SystemMessage的特殊逻辑

#### **第三层：Repository（存储层）**
- 负责数据持久化
- 支持多种存储后端
- 提供CRUD操作

---

## 三、核心接口

### 3.1 ChatMemory 接口

**核心职责**：管理对话记忆的生命周期

```java
package org.springframework.ai.chat.memory;

public interface ChatMemory {
    
    // 默认会话ID
    String DEFAULT_CONVERSATION_ID = "default";
    
    // 上下文键名
    String CONVERSATION_ID = "chat_memory_conversation_id";
    
    /**
     * 添加单条消息到记忆
     */
    default void add(String conversationId, Message message) {
        Assert.hasText(conversationId, "conversationId cannot be null or empty");
        Assert.notNull(message, "message cannot be null");
        this.add(conversationId, List.of(message));
    }
    
    /**
     * 添加多条消息到记忆
     */
    void add(String conversationId, List<Message> messages);
    
    /**
     * 获取指定会话的所有消息
     */
    List<Message> get(String conversationId);
    
    /**
     * 清除指定会话的所有消息
     */
    void clear(String conversationId);
}
```

**核心方法说明**：

| 方法 | 参数 | 返回值 | 说明 |
|------|------|--------|------|
| `add` | conversationId, messages | void | 添加消息到记忆 |
| `get` | conversationId | List\<Message\> | 获取记忆中的消息 |
| `clear` | conversationId | void | 清除会话记忆 |

### 3.2 ChatMemoryRepository 接口

**核心职责**：消息的存储和检索

```java
package org.springframework.ai.chat.memory;

public interface ChatMemoryRepository {
    
    /**
     * 查找所有会话ID
     */
    List<String> findConversationIds();
    
    /**
     * 查找指定会话的所有消息
     */
    List<Message> findByConversationId(String conversationId);
    
    /**
     * 保存消息（替换式保存）
     * 注意：会替换指定会话的所有现有消息
     */
    void saveAll(String conversationId, List<Message> messages);
    
    /**
     * 删除指定会话的所有消息
     */
    void deleteByConversationId(String conversationId);
}
```

**关键特性**：
- `saveAll`是**替换式保存**，不是追加
- 由`ChatMemory`实现负责决定保留哪些消息
- Repository只负责存取，不负责策略

---

## 四、内存实现

### 4.1 MessageWindowChatMemory（消息窗口）

**核心特性**：
- 维护固定数量的最近消息
- 默认保留最近20条消息
- 自动淘汰旧消息
- 特殊处理SystemMessage

#### **源码解析**

```java
package org.springframework.ai.chat.memory;

public final class MessageWindowChatMemory implements ChatMemory {
    
    // 默认最大消息数
    private static final int DEFAULT_MAX_MESSAGES = 20;
    
    private final ChatMemoryRepository chatMemoryRepository;
    private final int maxMessages;
    
    @Override
    public void add(String conversationId, List<Message> messages) {
        Assert.hasText(conversationId, "conversationId cannot be null or empty");
        Assert.notNull(messages, "messages cannot be null");
        
        // 1. 获取现有消息
        List<Message> memoryMessages = 
            this.chatMemoryRepository.findByConversationId(conversationId);
        
        // 2. 处理消息（合并、裁剪）
        List<Message> processedMessages = process(memoryMessages, messages);
        
        // 3. 保存处理后的消息
        this.chatMemoryRepository.saveAll(conversationId, processedMessages);
    }
    
    /**
     * 消息处理逻辑
     */
    private List<Message> process(
            List<Message> memoryMessages, 
            List<Message> newMessages) {
        
        List<Message> processedMessages = new ArrayList<>();
        
        // 1. 检查是否有新的SystemMessage
        Set<Message> memoryMessagesSet = new HashSet<>(memoryMessages);
        boolean hasNewSystemMessage = newMessages.stream()
            .filter(SystemMessage.class::isInstance)
            .anyMatch(message -> !memoryMessagesSet.contains(message));
        
        // 2. 如果有新SystemMessage，移除旧的SystemMessage
        memoryMessages.stream()
            .filter(message -> !(hasNewSystemMessage && 
                                 message instanceof SystemMessage))
            .forEach(processedMessages::add);
        
        // 3. 添加新消息
        processedMessages.addAll(newMessages);
        
        // 4. 如果消息数未超限，直接返回
        if (processedMessages.size() <= this.maxMessages) {
            return processedMessages;
        }
        
        // 5. 裁剪超出的消息（保留SystemMessage）
        int messagesToRemove = processedMessages.size() - this.maxMessages;
        List<Message> trimmedMessages = new ArrayList<>();
        int removed = 0;
        
        for (Message message : processedMessages) {
            // 保留SystemMessage，或者已经移除足够数量
            if (message instanceof SystemMessage || 
                removed >= messagesToRemove) {
                trimmedMessages.add(message);
            } else {
                removed++;
            }
        }
        
        return trimmedMessages;
    }
    
    @Override
    public List<Message> get(String conversationId) {
        Assert.hasText(conversationId, "conversationId cannot be null or empty");
        return this.chatMemoryRepository.findByConversationId(conversationId);
    }
    
    @Override
    public void clear(String conversationId) {
        Assert.hasText(conversationId, "conversationId cannot be null or empty");
        this.chatMemoryRepository.deleteByConversationId(conversationId);
    }
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static final class Builder {
        private ChatMemoryRepository chatMemoryRepository;
        private int maxMessages = DEFAULT_MAX_MESSAGES;
        
        public Builder chatMemoryRepository(
                ChatMemoryRepository chatMemoryRepository) {
            this.chatMemoryRepository = chatMemoryRepository;
            return this;
        }
        
        public Builder maxMessages(int maxMessages) {
            this.maxMessages = maxMessages;
            return this;
        }
        
        public MessageWindowChatMemory build() {
            // 如果没有指定Repository，使用内存实现
            if (this.chatMemoryRepository == null) {
                this.chatMemoryRepository = new InMemoryChatMemoryRepository();
            }
            return new MessageWindowChatMemory(
                this.chatMemoryRepository, 
                this.maxMessages
            );
        }
    }
}
```

#### **特性详解**

##### 1. **SystemMessage特殊处理**
```java
// 场景：当添加新的SystemMessage时
// 旧记忆：[SystemMessage("旧指令"), UserMessage, AssistantMessage]
// 新消息：[SystemMessage("新指令"), UserMessage]
// 结果：  [SystemMessage("新指令"), UserMessage, AssistantMessage, UserMessage]

// 原因：新的SystemMessage通常代表新的对话上下文，旧指令已过期
```

##### 2. **消息裁剪策略**
```java
// 场景：maxMessages=5，当前有7条消息
// 消息列表：
// [SystemMessage, User1, Assistant1, User2, Assistant2, User3, Assistant3]
//  ↓ 裁剪（保留SystemMessage + 最新4条）
// [SystemMessage, Assistant2, User3, Assistant3]

// 策略：
// - 始终保留SystemMessage
// - 按照FIFO（先进先出）淘汰旧消息
// - 优先淘汰非SystemMessage
```

#### **使用示例**

```java
// 1. 基础使用
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .maxMessages(10)  // 保留最近10条消息
    .build();

// 2. 使用自定义Repository
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(jdbcRepository)
    .maxMessages(20)
    .build();

// 3. 添加消息
chatMemory.add("user-123", new UserMessage("你好"));
chatMemory.add("user-123", new AssistantMessage("你好！有什么可以帮你的？"));

// 4. 获取记忆
List<Message> history = chatMemory.get("user-123");

// 5. 清除记忆
chatMemory.clear("user-123");
```

---

## 五、存储实现

### 5.1 InMemoryChatMemoryRepository（内存存储）

**特性**：
- 基于`ConcurrentHashMap`
- 线程安全
- 数据易失（重启丢失）
- 适合开发测试

#### **源码解析**

```java
package org.springframework.ai.chat.memory;

public final class InMemoryChatMemoryRepository 
        implements ChatMemoryRepository {
    
    // 使用线程安全的Map
    Map<String, List<Message>> chatMemoryStore = new ConcurrentHashMap<>();
    
    @Override
    public List<String> findConversationIds() {
        return new ArrayList<>(this.chatMemoryStore.keySet());
    }
    
    @Override
    public List<Message> findByConversationId(String conversationId) {
        Assert.hasText(conversationId, 
            "conversationId cannot be null or empty");
        List<Message> messages = this.chatMemoryStore.get(conversationId);
        // 返回副本，避免外部修改
        return messages != null ? new ArrayList<>(messages) : List.of();
    }
    
    @Override
    public void saveAll(String conversationId, List<Message> messages) {
        Assert.hasText(conversationId, 
            "conversationId cannot be null or empty");
        Assert.notNull(messages, "messages cannot be null");
        Assert.noNullElements(messages, 
            "messages cannot contain null elements");
        this.chatMemoryStore.put(conversationId, messages);
    }
    
    @Override
    public void deleteByConversationId(String conversationId) {
        Assert.hasText(conversationId, 
            "conversationId cannot be null or empty");
        this.chatMemoryStore.remove(conversationId);
    }
}
```

**使用场景**：
- ✅ 单机应用
- ✅ 开发测试
- ✅ 快速原型
- ❌ 生产环境
- ❌ 分布式系统
- ❌ 持久化需求

### 5.2 JdbcChatMemoryRepository（JDBC存储）

**特性**：
- 支持多种关系型数据库
- 数据持久化
- 支持事务
- 适合生产环境

#### **支持的数据库**

| 数据库 | Dialect类 | 特性 |
|--------|-----------|------|
| PostgreSQL | `PostgresChatMemoryRepositoryDialect` | 推荐，性能优秀 |
| MySQL/MariaDB | `MySqlChatMemoryRepositoryDialect` | 广泛使用 |
| SQL Server | `SqlServerChatMemoryRepositoryDialect` | 企业级 |
| HSQLDB | `HsqldbChatMemoryRepositoryDialect` | 内存数据库 |

#### **Maven依赖**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

#### **使用示例**

```java
// 1. Spring Boot自动配置（推荐）
@Autowired
private JdbcChatMemoryRepository chatMemoryRepository;

ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .chatMemoryRepository(chatMemoryRepository)
    .maxMessages(20)
    .build();

// 2. 手动创建
@Configuration
public class ChatMemoryConfig {
    
    @Bean
    public ChatMemoryRepository chatMemoryRepository(
            JdbcTemplate jdbcTemplate) {
        return JdbcChatMemoryRepository.builder()
            .jdbcTemplate(jdbcTemplate)
            .dialect(new PostgresChatMemoryRepositoryDialect())
            .build();
    }
    
    @Bean
    public ChatMemory chatMemory(
            ChatMemoryRepository chatMemoryRepository) {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(chatMemoryRepository)
            .maxMessages(20)
            .build();
    }
}
```

#### **数据库Schema**

```sql
-- PostgreSQL示例
CREATE TABLE chat_memory (
    conversation_id VARCHAR(255) NOT NULL,
    message_index INT NOT NULL,
    message_type VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (conversation_id, message_index)
);

CREATE INDEX idx_conversation_id ON chat_memory(conversation_id);
CREATE INDEX idx_created_at ON chat_memory(created_at);
```

### 5.3 CassandraChatMemoryRepository（Cassandra存储）

**特性**：
- 分布式NoSQL数据库
- 高可用、高扩展
- 适合大规模部署

#### **Maven依赖**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-cassandra</artifactId>
</dependency>
```

#### **配置示例**

```yaml
spring:
  cassandra:
    keyspace-name: spring_ai
    contact-points: localhost:9042
    local-datacenter: datacenter1
  ai:
    chat:
      memory:
        cassandra:
          table: chat_memory
```

### 5.4 Neo4jChatMemoryRepository（图数据库存储）

**特性**：
- 图数据库存储
- 适合关系复杂的对话
- 支持图查询

#### **Maven依赖**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-neo4j</artifactId>
</dependency>
```

### 5.5 CosmosDBChatMemoryRepository（Azure Cosmos DB存储）

**特性**：
- 全球分布式数据库
- 多模型支持
- Azure云原生

#### **Maven依赖**

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-cosmos-db</artifactId>
</dependency>
```

#### **使用示例**

```java
ChatMemoryRepository chatMemoryRepository = 
    CosmosDBChatMemoryRepository.create(
        CosmosDBChatMemoryRepositoryConfig.builder()
            .withCosmosClient(cosmosAsyncClient)
            .withDatabaseName("chat-memory-db")
            .withContainerName("conversations")
            .build()
    );
```

### 5.6 存储选择指南

| 存储类型 | 适用场景 | 优势 | 劣势 |
|----------|----------|------|------|
| **InMemory** | 开发测试、单机应用 | 快速简单 | 数据易失 |
| **JDBC** | 中小型应用、单机或主从 | 成熟稳定、事务支持 | 扩展性一般 |
| **Cassandra** | 大规模分布式系统 | 高可用、高扩展 | 运维复杂 |
| **Neo4j** | 关系复杂的对话系统 | 图查询强大 | 学习曲线陡 |
| **CosmosDB** | Azure云环境 | 全球分布、多模型 | 成本较高 |

---

## 六、Memory Advisor

Memory Advisor是将`ChatMemory`集成到`ChatClient`的桥梁。

### 6.1 三种Advisor对比

| Advisor | 集成方式 | 适用场景 | 优势 | 劣势 |
|---------|----------|----------|------|------|
| **MessageChatMemoryAdvisor** | 将历史消息添加到Prompt的messages列表 | 标准对话 | 结构化、上下文清晰 | Token消耗大 |
| **PromptChatMemoryAdvisor** | 将历史消息格式化为文本，追加到SystemMessage | 长对话、Token敏感 | Token效率高 | 丢失消息结构 |
| **VectorStoreChatMemoryAdvisor** | 使用向量检索获取相关历史 | 超长对话、知识库 | 可检索任意历史 | 需要向量数据库 |

### 6.2 MessageChatMemoryAdvisor

#### **工作原理**

```
┌─────────────────────────────────────────────────────────┐
│  1. Before: 获取历史消息                                 │
│     chatMemory.get(conversationId)                       │
│     → [UserMessage, AssistantMessage, ...]              │
├─────────────────────────────────────────────────────────┤
│  2. 构建完整消息列表                                      │
│     messages = [SystemMessage] + history + [newUserMsg]  │
├─────────────────────────────────────────────────────────┤
│  3. 调用AI模型                                           │
│     chatModel.call(new Prompt(messages))                 │
├─────────────────────────────────────────────────────────┤
│  4. After: 保存AI响应                                    │
│     chatMemory.add(conversationId, assistantMessage)     │
└─────────────────────────────────────────────────────────┘
```

#### **源码解析**

```java
package org.springframework.ai.chat.client.advisor;

public final class MessageChatMemoryAdvisor 
        implements BaseChatMemoryAdvisor {
    
    private final ChatMemory chatMemory;
    private final String defaultConversationId;
    private final int order;
    private final Scheduler scheduler;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest chatClientRequest, 
            AdvisorChain advisorChain) {
        
        // 1. 获取会话ID
        String conversationId = getConversationId(
            chatClientRequest.context(), 
            this.defaultConversationId
        );
        
        // 2. 从记忆中获取历史消息
        List<Message> memoryMessages = this.chatMemory.get(conversationId);
        
        // 3. 构建完整消息列表
        // 历史消息 + 当前Prompt的消息
        List<Message> processedMessages = new ArrayList<>(memoryMessages);
        processedMessages.addAll(chatClientRequest.prompt().getInstructions());
        
        // 4. 创建新的Request
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
        
        // 提取AI的响应消息
        List<Message> assistantMessages = new ArrayList<>();
        if (chatClientResponse.chatResponse() != null) {
            assistantMessages = chatClientResponse.chatResponse()
                .getResults()
                .stream()
                .map(g -> (Message) g.getOutput())
                .toList();
        }
        
        // 保存到记忆
        String conversationId = getConversationId(
            chatClientResponse.context(), 
            this.defaultConversationId
        );
        this.chatMemory.add(conversationId, assistantMessages);
        
        return chatClientResponse;
    }
    
    // Builder
    public static Builder builder(ChatMemory chatMemory) {
        return new Builder(chatMemory);
    }
}
```

#### **使用示例**

```java
// 1. 创建ChatMemory
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .maxMessages(10)
    .build();

// 2. 创建Advisor
MessageChatMemoryAdvisor memoryAdvisor = 
    MessageChatMemoryAdvisor.builder(chatMemory)
        .conversationId("user-123")  // 可选，默认为"default"
        .order(0)                     // 可选，默认为0
        .build();

// 3. 配置到ChatClient
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(memoryAdvisor)
    .build();

// 4. 开始对话
String response1 = chatClient.prompt()
    .user("我的名字是张三")
    .call()
    .content();
// → "你好，张三！很高兴认识你。"

String response2 = chatClient.prompt()
    .user("你还记得我的名字吗？")
    .call()
    .content();
// → "当然记得，你的名字是张三。"
```

#### **实际发送的消息**

```java
// 第一轮对话
Prompt {
    messages: [
        UserMessage("我的名字是张三")
    ]
}

// 第二轮对话（自动注入历史）
Prompt {
    messages: [
        UserMessage("我的名字是张三"),           // 历史
        AssistantMessage("你好，张三！..."),    // 历史
        UserMessage("你还记得我的名字吗？")      // 当前
    ]
}
```

### 6.3 PromptChatMemoryAdvisor

#### **工作原理**

```
┌─────────────────────────────────────────────────────────┐
│  1. Before: 获取历史消息                                 │
│     chatMemory.get(conversationId)                       │
├─────────────────────────────────────────────────────────┤
│  2. 格式化为文本                                         │
│     "USER: 你好\nASSISTANT: 你好！\nUSER: ..."         │
├─────────────────────────────────────────────────────────┤
│  3. 注入到SystemMessage                                  │
│     SystemMessage("""                                    │
│         {原始指令}                                        │
│         MEMORY:                                          │
│         {格式化的历史}                                    │
│     """)                                                 │
├─────────────────────────────────────────────────────────┤
│  4. 调用AI模型                                           │
│     chatModel.call(new Prompt([augmentedSystemMsg, ...]))│
└─────────────────────────────────────────────────────────┘
```

#### **源码解析**

```java
package org.springframework.ai.chat.client.advisor;

public final class PromptChatMemoryAdvisor 
        implements BaseChatMemoryAdvisor {
    
    // 默认系统提示词模板
    private static final PromptTemplate DEFAULT_SYSTEM_PROMPT_TEMPLATE = 
        new PromptTemplate("""
            {instructions}
            
            Use the conversation memory from the MEMORY section 
            to provide accurate answers.
            
            ---------------------
            MEMORY:
            {memory}
            ---------------------
            """);
    
    private final PromptTemplate systemPromptTemplate;
    private final ChatMemory chatMemory;
    private final String defaultConversationId;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest chatClientRequest, 
            AdvisorChain advisorChain) {
        
        String conversationId = getConversationId(
            chatClientRequest.context(), 
            this.defaultConversationId
        );
        
        // 1. 获取历史消息
        List<Message> memoryMessages = this.chatMemory.get(conversationId);
        
        // 2. 格式化为字符串
        String memory = memoryMessages.stream()
            .filter(m -> m.getMessageType() == MessageType.USER || 
                        m.getMessageType() == MessageType.ASSISTANT)
            .map(m -> m.getMessageType() + ":" + m.getText())
            .collect(Collectors.joining(System.lineSeparator()));
        
        // 3. 增强SystemMessage
        SystemMessage systemMessage = 
            chatClientRequest.prompt().getSystemMessage();
        String augmentedSystemText = this.systemPromptTemplate.render(
            Map.of(
                "instructions", systemMessage.getText(),
                "memory", memory
            )
        );
        
        // 4. 创建新的Request
        ChatClientRequest processedChatClientRequest = 
            chatClientRequest.mutate()
                .prompt(chatClientRequest.prompt()
                    .augmentSystemMessage(augmentedSystemText))
                .build();
        
        // 5. 保存当前用户消息
        UserMessage userMessage = 
            processedChatClientRequest.prompt().getUserMessage();
        this.chatMemory.add(conversationId, userMessage);
        
        return processedChatClientRequest;
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse chatClientResponse, 
            AdvisorChain advisorChain) {
        
        // 提取并保存AI响应
        List<Message> assistantMessages = Optional.ofNullable(chatClientResponse)
            .map(ChatClientResponse::chatResponse)
            .filter(response -> response.getResults() != null && 
                               !response.getResults().isEmpty())
            .map(response -> response.getResults()
                .stream()
                .map(g -> (Message) g.getOutput())
                .collect(Collectors.toList()))
            .orElse(List.of());
        
        if (!assistantMessages.isEmpty()) {
            String conversationId = getConversationId(
                chatClientResponse.context(), 
                this.defaultConversationId
            );
            this.chatMemory.add(conversationId, assistantMessages);
        }
        
        return chatClientResponse;
    }
}
```

#### **使用示例**

```java
// 1. 创建ChatMemory
ChatMemory chatMemory = MessageWindowChatMemory.builder()
    .maxMessages(10)
    .build();

// 2. 创建Advisor（使用默认模板）
PromptChatMemoryAdvisor memoryAdvisor = 
    PromptChatMemoryAdvisor.builder(chatMemory)
        .conversationId("user-123")
        .build();

// 3. 使用自定义模板
PromptTemplate customTemplate = new PromptTemplate("""
    你是一个智能助手。
    
    {instructions}
    
    ===== 对话历史 =====
    {memory}
    ========================
    
    请根据对话历史提供准确的回答。
    """);

PromptChatMemoryAdvisor customMemoryAdvisor = 
    PromptChatMemoryAdvisor.builder(chatMemory)
        .systemPromptTemplate(customTemplate)
        .conversationId("user-123")
        .build();

// 4. 配置到ChatClient
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(customMemoryAdvisor)
    .defaultSystem("你是一个友好的助手")
    .build();
```

#### **实际发送的消息**

```java
// 第一轮对话
Prompt {
    messages: [
        SystemMessage("你是一个友好的助手"),
        UserMessage("我的名字是张三")
    ]
}

// 第二轮对话
Prompt {
    messages: [
        SystemMessage("""
            你是一个友好的助手
            
            Use the conversation memory from the MEMORY section...
            
            ---------------------
            MEMORY:
            USER:我的名字是张三
            ASSISTANT:你好，张三！很高兴认识你。
            ---------------------
            """),
        UserMessage("你还记得我的名字吗？")
    ]
}
```

#### **优势**

1. **Token效率高**：历史以文本形式压缩在SystemMessage中
2. **减少API调用成本**：对于长对话特别有效
3. **简化消息结构**：只有当前的UserMessage，不需要完整历史

#### **劣势**

1. **丢失消息结构**：AI无法区分哪条是User，哪条是Assistant
2. **格式依赖**：依赖`USER:`, `ASSISTANT:`前缀来区分
3. **SystemMessage膨胀**：长对话会导致SystemMessage非常大

### 6.4 VectorStoreChatMemoryAdvisor

#### **工作原理**

```
┌─────────────────────────────────────────────────────────┐
│  1. Before: 使用向量检索获取相关历史                      │
│     vectorStore.similaritySearch(                        │
│         query: currentUserMessage,                       │
│         filter: "conversationId=='user-123'",           │
│         topK: 20                                         │
│     )                                                    │
├─────────────────────────────────────────────────────────┤
│  2. 格式化为文本并注入SystemMessage                      │
│     类似PromptChatMemoryAdvisor                          │
├─────────────────────────────────────────────────────────┤
│  3. 保存当前消息到向量数据库                             │
│     vectorStore.add(Document(userMessage))               │
├─────────────────────────────────────────────────────────┤
│  4. After: 保存AI响应到向量数据库                        │
│     vectorStore.add(Document(assistantMessage))          │
└─────────────────────────────────────────────────────────┘
```

#### **核心特性**

- **语义检索**：根据当前问题的语义检索相关历史
- **突破窗口限制**：可以检索任意久远的历史
- **智能过滤**：只检索与当前对话相关的内容

#### **源码解析**

```java
package org.springframework.ai.chat.client.advisor.vectorstore;

public final class VectorStoreChatMemoryAdvisor 
        implements BaseChatMemoryAdvisor {
    
    // 默认TopK
    private static final int DEFAULT_TOP_K = 20;
    
    // 元数据键
    private static final String DOCUMENT_METADATA_CONVERSATION_ID = 
        "conversationId";
    private static final String DOCUMENT_METADATA_MESSAGE_TYPE = 
        "messageType";
    
    // 默认模板
    private static final PromptTemplate DEFAULT_SYSTEM_PROMPT_TEMPLATE = 
        new PromptTemplate("""
            {instructions}
            
            Use the long term conversation memory from the 
            LONG_TERM_MEMORY section to provide accurate answers.
            
            ---------------------
            LONG_TERM_MEMORY:
            {long_term_memory}
            ---------------------
            """);
    
    private final PromptTemplate systemPromptTemplate;
    private final int defaultTopK;
    private final VectorStore vectorStore;
    
    @Override
    public ChatClientRequest before(
            ChatClientRequest request, 
            AdvisorChain advisorChain) {
        
        String conversationId = getConversationId(
            request.context(), 
            this.defaultConversationId
        );
        
        // 1. 获取当前用户消息作为查询
        String query = request.prompt().getUserMessage() != null 
            ? request.prompt().getUserMessage().getText() 
            : "";
        
        // 2. 获取TopK（可从上下文覆盖）
        int topK = getChatMemoryTopK(request.context());
        
        // 3. 构建过滤条件（只检索当前会话）
        String filter = DOCUMENT_METADATA_CONVERSATION_ID + 
                       "=='" + conversationId + "'";
        
        // 4. 向量检索相关历史
        SearchRequest searchRequest = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .filterExpression(filter)
            .build();
        List<Document> documents = 
            this.vectorStore.similaritySearch(searchRequest);
        
        // 5. 格式化为文本
        String longTermMemory = documents == null ? ""
            : documents.stream()
                .map(Document::getText)
                .collect(Collectors.joining(System.lineSeparator()));
        
        // 6. 增强SystemMessage
        SystemMessage systemMessage = request.prompt().getSystemMessage();
        String augmentedSystemText = this.systemPromptTemplate.render(
            Map.of(
                "instructions", systemMessage.getText(), 
                "long_term_memory", longTermMemory
            )
        );
        
        // 7. 创建新Request
        ChatClientRequest processedRequest = request.mutate()
            .prompt(request.prompt()
                .augmentSystemMessage(augmentedSystemText))
            .build();
        
        // 8. 保存当前用户消息到向量数据库
        UserMessage userMessage = processedRequest.prompt().getUserMessage();
        if (userMessage != null) {
            this.vectorStore.write(
                toDocuments(List.of(userMessage), conversationId)
            );
        }
        
        return processedRequest;
    }
    
    @Override
    public ChatClientResponse after(
            ChatClientResponse chatClientResponse, 
            AdvisorChain advisorChain) {
        
        // 提取AI响应
        List<Message> assistantMessages = new ArrayList<>();
        if (chatClientResponse.chatResponse() != null) {
            assistantMessages = chatClientResponse.chatResponse()
                .getResults()
                .stream()
                .map(g -> (Message) g.getOutput())
                .toList();
        }
        
        // 保存到向量数据库
        String conversationId = getConversationId(
            chatClientResponse.context(), 
            this.defaultConversationId
        );
        this.vectorStore.write(
            toDocuments(assistantMessages, conversationId)
        );
        
        return chatClientResponse;
    }
    
    /**
     * 将Message转换为Document
     */
    private List<Document> toDocuments(
            List<Message> messages, 
            String conversationId) {
        
        return messages.stream()
            .filter(m -> m.getMessageType() == MessageType.USER || 
                        m.getMessageType() == MessageType.ASSISTANT)
            .map(message -> {
                // 构建元数据
                Map<String, Object> metadata = new HashMap<>();
                metadata.put(DOCUMENT_METADATA_CONVERSATION_ID, 
                            conversationId);
                metadata.put(DOCUMENT_METADATA_MESSAGE_TYPE, 
                            message.getMessageType().name());
                
                // 创建Document
                if (message instanceof UserMessage userMessage) {
                    return Document.builder()
                        .text(userMessage.getText())
                        .metadata(metadata)
                        .build();
                } else if (message instanceof AssistantMessage assistantMessage) {
                    return Document.builder()
                        .text(assistantMessage.getText())
                        .metadata(metadata)
                        .build();
                }
                throw new RuntimeException(
                    "Unknown message type: " + message.getMessageType()
                );
            })
            .toList();
    }
    
    private int getChatMemoryTopK(Map<String, Object> context) {
        return context.containsKey(TOP_K) 
            ? Integer.parseInt(context.get(TOP_K).toString()) 
            : this.defaultTopK;
    }
}
```

#### **使用示例**

```java
// 1. 创建VectorStore
VectorStore vectorStore = new PgVectorStore(
    jdbcTemplate,
    embeddingModel
);

// 2. 创建Advisor
VectorStoreChatMemoryAdvisor memoryAdvisor = 
    VectorStoreChatMemoryAdvisor.builder(vectorStore)
        .conversationId("user-123")
        .defaultTopK(20)  // 检索20条最相关的历史
        .build();

// 3. 配置到ChatClient
ChatClient chatClient = ChatClient.builder(chatModel)
    .defaultAdvisors(memoryAdvisor)
    .build();

// 4. 对话（可以动态指定TopK）
String response = chatClient.prompt()
    .user("关于项目X的讨论内容是什么？")
    .context(Map.of(
        VectorStoreChatMemoryAdvisor.TOP_K, 30  // 覆盖默认TopK
    ))
    .call()
    .content();
```

#### **适用场景**

1. **超长对话**：对话轮次超过100轮
2. **知识库式对话**：需要记住很多历史细节
3. **多主题对话**：根据当前问题智能检索相关历史
4. **客服系统**：检索历史工单和解决方案

#### **优势**

| 特性 | 说明 |
|------|------|
| **突破窗口限制** | 不受消息窗口限制，可检索任意历史 |
| **语义智能** | 根据当前问题的语义检索相关历史 |
| **节省Token** | 只检索相关内容，不是全部历史 |
| **扩展性强** | 可以横向扩展向量数据库 |

#### **劣势**

| 问题 | 说明 |
|------|------|
| **需要向量数据库** | 增加系统复杂度和成本 |
| **检索延迟** | 向量检索有一定延迟 |
| **依赖Embedding质量** | 如果Embedding不准确，检索效果差 |
| **丢失时序信息** | 检索结果按相似度排序，不是时间顺序 |

---

## 七、完整实战案例

### 7.1 智能客服系统

#### **需求**

- 支持多用户并发对话
- 每个用户独立会话
- 保留最近20轮对话
- 持久化到数据库

#### **实现**

```java
@Configuration
public class CustomerServiceConfig {
    
    @Bean
    public ChatMemoryRepository chatMemoryRepository(
            JdbcTemplate jdbcTemplate) {
        return JdbcChatMemoryRepository.builder()
            .jdbcTemplate(jdbcTemplate)
            .dialect(new PostgresChatMemoryRepositoryDialect())
            .build();
    }
    
    @Bean
    public ChatMemory chatMemory(
            ChatMemoryRepository chatMemoryRepository) {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(chatMemoryRepository)
            .maxMessages(20)  // 保留最近20条消息
            .build();
    }
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ChatMemory chatMemory) {
        
        MessageChatMemoryAdvisor memoryAdvisor = 
            MessageChatMemoryAdvisor.builder(chatMemory)
                .build();
        
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                你是一个专业的客服助手。
                你的职责是：
                1. 回答用户关于产品的问题
                2. 解决用户的技术问题
                3. 引导用户完成操作
                
                回答要求：
                - 礼貌友好
                - 简洁明了
                - 提供具体步骤
                """)
            .defaultAdvisors(memoryAdvisor)
            .build();
    }
}

@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private ChatMemory chatMemory;
    
    /**
     * 发送消息
     */
    @PostMapping("/send")
    public ChatResponse sendMessage(@RequestBody ChatRequest request) {
        // 从请求中获取用户ID作为会话ID
        String conversationId = request.getUserId();
        
        // 调用ChatClient（会话ID通过context传递）
        String response = chatClient.prompt()
            .user(request.getMessage())
            .context(Map.of(
                ChatMemory.CONVERSATION_ID, conversationId
            ))
            .call()
            .content();
        
        return new ChatResponse(response);
    }
    
    /**
     * 获取对话历史
     */
    @GetMapping("/history/{userId}")
    public List<MessageDTO> getHistory(@PathVariable String userId) {
        List<Message> messages = chatMemory.get(userId);
        return messages.stream()
            .map(this::toDTO)
            .collect(Collectors.toList());
    }
    
    /**
     * 清除对话历史
     */
    @DeleteMapping("/history/{userId}")
    public void clearHistory(@PathVariable String userId) {
        chatMemory.clear(userId);
    }
    
    private MessageDTO toDTO(Message message) {
        return new MessageDTO(
            message.getMessageType().toString(),
            message.getText()
        );
    }
}

// DTOs
record ChatRequest(String userId, String message) {}
record ChatResponse(String message) {}
record MessageDTO(String type, String content) {}
```

#### **使用示例**

```bash
# 用户1的对话
curl -X POST http://localhost:8080/api/chat/send \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-001", "message": "我的订单号是12345，物流信息是什么？"}'

# 响应
{
  "message": "您好！订单12345当前状态是已发货，预计明天送达。"
}

# 继续对话
curl -X POST http://localhost:8080/api/chat/send \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-001", "message": "可以改地址吗？"}'

# 响应（AI记住了订单号）
{
  "message": "订单12345目前还在运输中，您可以联系物流公司修改配送地址..."
}

# 查看历史
curl http://localhost:8080/api/chat/history/user-001

# 响应
[
  {"type": "USER", "content": "我的订单号是12345，物流信息是什么？"},
  {"type": "ASSISTANT", "content": "您好！订单12345当前状态是已发货..."},
  {"type": "USER", "content": "可以改地址吗？"},
  {"type": "ASSISTANT", "content": "订单12345目前还在运输中，您可以..."}
]
```

### 7.2 长期知识库对话系统

#### **需求**

- 支持超长对话（>100轮）
- 智能检索相关历史
- 多主题对话

#### **实现**

```java
@Configuration
public class KnowledgeBaseConfig {
    
    @Bean
    public VectorStore vectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        return new PgVectorStore(
            jdbcTemplate,
            embeddingModel,
            PgVectorStore.PgVectorStoreConfig.builder()
                .dimensions(1536)
                .distanceType(PgDistanceType.COSINE_DISTANCE)
                .indexType(PgIndexType.HNSW)
                .build()
        );
    }
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            VectorStore vectorStore) {
        
        VectorStoreChatMemoryAdvisor memoryAdvisor = 
            VectorStoreChatMemoryAdvisor.builder(vectorStore)
                .defaultTopK(30)  // 检索30条相关历史
                .build();
        
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                你是一个知识库助手。
                你可以回答关于公司文档、项目、技术的问题。
                请根据历史对话和当前问题提供准确的答案。
                """)
            .defaultAdvisors(memoryAdvisor)
            .build();
    }
}

@Service
public class KnowledgeBaseService {
    
    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private VectorStore vectorStore;
    
    /**
     * 对话（自动检索相关历史）
     */
    public String chat(String userId, String question) {
        return chatClient.prompt()
            .user(question)
            .context(Map.of(
                ChatMemory.CONVERSATION_ID, userId
            ))
            .call()
            .content();
    }
    
    /**
     * 对话（自定义检索数量）
     */
    public String chatWithCustomTopK(
            String userId, 
            String question, 
            int topK) {
        return chatClient.prompt()
            .user(question)
            .context(Map.of(
                ChatMemory.CONVERSATION_ID, userId,
                VectorStoreChatMemoryAdvisor.TOP_K, topK
            ))
            .call()
            .content();
    }
    
    /**
     * 清除某个用户的所有历史
     */
    public void clearUserHistory(String userId) {
        String filter = "conversationId=='" + userId + "'";
        vectorStore.delete(filter);
    }
}
```

#### **使用示例**

```java
@RestController
@RequestMapping("/api/kb")
public class KnowledgeBaseController {
    
    @Autowired
    private KnowledgeBaseService kbService;
    
    @PostMapping("/ask")
    public String ask(@RequestBody AskRequest request) {
        return kbService.chat(
            request.userId(), 
            request.question()
        );
    }
}

// 示例对话
// 第1轮（讨论项目A）
POST /api/kb/ask
{
  "userId": "dev-001",
  "question": "项目A使用的技术栈是什么？"
}
→ "项目A使用Spring Boot、React、PostgreSQL..."

// 第2轮（讨论项目B）
POST /api/kb/ask
{
  "userId": "dev-001",
  "question": "项目B的部署流程是怎样的？"
}
→ "项目B使用Docker部署，具体步骤是..."

// ... 中间经过100+轮对话 ...

// 第150轮（回到项目A）
POST /api/kb/ask
{
  "userId": "dev-001",
  "question": "项目A的数据库是什么？"
}
→ "根据我们之前的对话，项目A使用PostgreSQL数据库..."
// ✅ 即使隔了100+轮，依然能检索到第1轮的内容！
```

### 7.3 混合策略：多Advisor组合

#### **需求**

- 短期记忆：保留最近5轮对话（结构化）
- 长期记忆：从向量数据库检索相关历史

#### **实现**

```java
@Configuration
public class HybridMemoryConfig {
    
    @Bean
    public ChatMemory shortTermMemory(
            ChatMemoryRepository repository) {
        return MessageWindowChatMemory.builder()
            .chatMemoryRepository(repository)
            .maxMessages(5)  // 短期：只保留5条
            .build();
    }
    
    @Bean
    public ChatClient chatClient(
            ChatModel chatModel,
            ChatMemory shortTermMemory,
            VectorStore vectorStore) {
        
        // 1. 短期记忆Advisor（高优先级）
        MessageChatMemoryAdvisor shortTermAdvisor = 
            MessageChatMemoryAdvisor.builder(shortTermMemory)
                .order(0)  // 先执行
                .build();
        
        // 2. 长期记忆Advisor（低优先级）
        VectorStoreChatMemoryAdvisor longTermAdvisor = 
            VectorStoreChatMemoryAdvisor.builder(vectorStore)
                .defaultTopK(20)
                .order(1)  // 后执行
                .build();
        
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                你是一个智能助手。
                你拥有短期记忆（最近5轮对话）和长期记忆（历史相关对话）。
                请综合两种记忆提供答案。
                """)
            .defaultAdvisors(shortTermAdvisor, longTermAdvisor)
            .build();
    }
}
```

#### **效果**

```
最终发送给AI的消息结构：

SystemMessage("""
    你是一个智能助手...
    
    ===== 长期记忆（向量检索） =====
    USER: 关于项目X的架构讨论...（30天前）
    ASSISTANT: 项目X采用微服务架构...
    ...
    ===========================
    """)

+ 短期记忆（最近5轮）
  UserMessage("昨天讨论的API接口是什么？")
  AssistantMessage("昨天我们讨论了用户登录接口...")
  UserMessage("登录接口的参数有哪些？")
  AssistantMessage("登录接口需要username和password...")
  UserMessage("密码需要加密吗？")
  
+ 当前消息
  UserMessage("项目X的登录接口需要加密吗？")
```

---

## 八、最佳实践

### 8.1 会话ID设计

#### **推荐方案**

```java
// ✅ 方案1：用户维度（适合多轮对话）
String conversationId = "user-" + userId;

// ✅ 方案2：会话维度（适合独立会话）
String conversationId = "session-" + sessionId;

// ✅ 方案3：业务维度（适合业务场景）
String conversationId = "order-" + orderId;
String conversationId = "ticket-" + ticketId;

// ❌ 不推荐：固定ID
String conversationId = "default";  // 所有用户共享记忆！
```

#### **动态会话ID**

```java
@Service
public class ConversationService {
    
    /**
     * 创建新会话
     */
    public String createConversation(String userId) {
        String conversationId = UUID.randomUUID().toString();
        // 存储到数据库
        conversationRepository.save(new Conversation(
            conversationId,
            userId,
            LocalDateTime.now()
        ));
        return conversationId;
    }
    
    /**
     * 获取用户的活跃会话
     */
    public String getActiveConversation(String userId) {
        return conversationRepository
            .findActiveByUserId(userId)
            .map(Conversation::getId)
            .orElseGet(() -> createConversation(userId));
    }
}

// 使用
@RestController
public class ChatController {
    
    @PostMapping("/chat")
    public String chat(
            @RequestParam String userId,
            @RequestParam String message) {
        
        String conversationId = 
            conversationService.getActiveConversation(userId);
        
        return chatClient.prompt()
            .user(message)
            .context(Map.of(
                ChatMemory.CONVERSATION_ID, conversationId
            ))
            .call()
            .content();
    }
}
```

### 8.2 内存窗口大小选择

| 窗口大小 | 适用场景 | Token消耗 | 上下文质量 |
|----------|----------|-----------|------------|
| 5-10条 | 简单问答、表单填写 | 低 | 基础 |
| 10-20条 | 客服对话、日常聊天 | 中 | 良好 |
| 20-50条 | 技术支持、长对话 | 高 | 优秀 |
| 50+条 | 深度咨询、复杂场景 | 很高 | 极佳 |

**动态调整示例**：

```java
@Service
public class AdaptiveMemoryService {
    
    /**
     * 根据对话复杂度动态调整窗口大小
     */
    public ChatMemory createAdaptiveMemory(String scenario) {
        int maxMessages = switch (scenario) {
            case "simple-qa" -> 5;
            case "customer-service" -> 15;
            case "technical-support" -> 30;
            case "consultation" -> 50;
            default -> 20;
        };
        
        return MessageWindowChatMemory.builder()
            .maxMessages(maxMessages)
            .build();
    }
}
```

### 8.3 存储选择策略

```java
@Configuration
public class StorageConfig {
    
    @Bean
    @ConditionalOnProperty(
        name = "spring.profiles.active", 
        havingValue = "dev"
    )
    public ChatMemoryRepository devRepository() {
        // 开发环境：内存存储
        return new InMemoryChatMemoryRepository();
    }
    
    @Bean
    @ConditionalOnProperty(
        name = "spring.profiles.active", 
        havingValue = "prod"
    )
    public ChatMemoryRepository prodRepository(
            JdbcTemplate jdbcTemplate) {
        // 生产环境：数据库存储
        return JdbcChatMemoryRepository.builder()
            .jdbcTemplate(jdbcTemplate)
            .dialect(new PostgresChatMemoryRepositoryDialect())
            .build();
    }
}
```

### 8.4 内存清理策略

#### **定时清理**

```java
@Component
public class MemoryCleanupScheduler {
    
    @Autowired
    private ChatMemoryRepository repository;
    
    /**
     * 每天凌晨2点清理30天前的对话
     */
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanupOldConversations() {
        LocalDateTime cutoffTime = LocalDateTime.now().minusDays(30);
        
        List<String> conversationIds = repository.findConversationIds();
        for (String conversationId : conversationIds) {
            List<Message> messages = 
                repository.findByConversationId(conversationId);
            
            // 检查最后一条消息的时间
            if (isOlderThan(messages, cutoffTime)) {
                repository.deleteByConversationId(conversationId);
                logger.info("Cleaned up conversation: {}", conversationId);
            }
        }
    }
    
    private boolean isOlderThan(
            List<Message> messages, 
            LocalDateTime cutoffTime) {
        // 实现时间判断逻辑
        // ...
    }
}
```

#### **主动清理**

```java
@Service
public class ConversationService {
    
    /**
     * 结束会话时清理
     */
    public void endConversation(String conversationId) {
        // 1. 归档到长期存储（可选）
        archiveConversation(conversationId);
        
        // 2. 清理记忆
        chatMemory.clear(conversationId);
        
        logger.info("Conversation {} ended and cleared", conversationId);
    }
    
    /**
     * 归档对话
     */
    private void archiveConversation(String conversationId) {
        List<Message> messages = chatMemory.get(conversationId);
        conversationArchiveRepository.save(
            new ConversationArchive(
                conversationId,
                messages,
                LocalDateTime.now()
            )
        );
    }
}
```

### 8.5 监控和日志

```java
@Component
@Aspect
public class MemoryMonitoringAspect {
    
    private final MeterRegistry meterRegistry;
    
    /**
     * 监控内存操作
     */
    @Around("execution(* org.springframework.ai.chat.memory.ChatMemory.add(..))")
    public Object monitorAdd(ProceedingJoinPoint joinPoint) throws Throwable {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            Object result = joinPoint.proceed();
            
            // 记录成功指标
            sample.stop(Timer.builder("chat.memory.add")
                .tag("status", "success")
                .register(meterRegistry));
            
            return result;
        } catch (Exception e) {
            // 记录失败指标
            sample.stop(Timer.builder("chat.memory.add")
                .tag("status", "error")
                .register(meterRegistry));
            throw e;
        }
    }
    
    /**
     * 监控内存大小
     */
    @Scheduled(fixedRate = 60000)  // 每分钟
    public void recordMemorySize() {
        if (repository instanceof InMemoryChatMemoryRepository inMemoryRepo) {
            int totalMessages = inMemoryRepo.chatMemoryStore.values()
                .stream()
                .mapToInt(List::size)
                .sum();
            
            meterRegistry.gauge(
                "chat.memory.total_messages", 
                totalMessages
            );
        }
    }
}
```

### 8.6 错误处理

```java
@Service
public class RobustChatService {
    
    @Autowired
    private ChatClient chatClient;
    
    @Autowired
    private ChatMemory chatMemory;
    
    /**
     * 带降级的对话
     */
    public String chatWithFallback(String userId, String message) {
        try {
            return chat(userId, message);
        } catch (Exception e) {
            logger.error("Chat failed, falling back", e);
            
            // 降级策略1：清除记忆后重试
            try {
                chatMemory.clear(userId);
                return chat(userId, message);
            } catch (Exception e2) {
                logger.error("Fallback also failed", e2);
                
                // 降级策略2：返回友好错误
                return "抱歉，系统暂时无法处理您的请求，请稍后再试。";
            }
        }
    }
    
    private String chat(String userId, String message) {
        return chatClient.prompt()
            .user(message)
            .context(Map.of(ChatMemory.CONVERSATION_ID, userId))
            .call()
            .content();
    }
}
```

---

## 总结

### 核心要点

1. **ChatMemory vs Repository**：
   - `ChatMemory`：管理策略（保留哪些消息）
   - `ChatMemoryRepository`：存储实现（如何持久化）

2. **三种Advisor**：
   - `MessageChatMemoryAdvisor`：标准方案，消息列表形式
   - `PromptChatMemoryAdvisor`：Token优化，文本形式
   - `VectorStoreChatMemoryAdvisor`：长期记忆，向量检索

3. **存储选择**：
   - 开发测试：`InMemoryChatMemoryRepository`
   - 单机生产：`JdbcChatMemoryRepository`
   - 分布式：`CassandraChatMemoryRepository`

4. **最佳实践**：
   - 合理设计会话ID
   - 根据场景选择窗口大小
   - 定期清理历史记忆
   - 添加监控和日志

### 学习路径

1. ✅ 理解核心概念：ChatMemory vs ChatHistory
2. ✅ 掌握基础实现：`MessageWindowChatMemory` + `InMemoryChatMemoryRepository`
3. ✅ 学习三种Advisor的区别和适用场景
4. ✅ 实践生产级配置：JDBC持久化
5. ✅ 进阶：向量检索长期记忆
6. ✅ 优化：混合策略、监控、降级

现在你已经完全掌握了Spring AI的Memory对话记忆机制！🎉

