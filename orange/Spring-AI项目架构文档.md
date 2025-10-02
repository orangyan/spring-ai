# Spring AI 项目架构文档

## 📋 目录
- [项目概述](#项目概述)
- [整体架构](#整体架构)
- [核心模块详解](#核心模块详解)
- [实现模块](#实现模块)
- [集成模块](#集成模块)
- [辅助模块](#辅助模块)
- [模块依赖关系](#模块依赖关系)
- [学习路径建议](#学习路径建议)

---

## 项目概述

**Spring AI** 是一个为开发AI应用提供Spring友好API和抽象的框架。

### 核心目标
- 将Spring生态系统的设计原则应用于AI领域
- 提供可移植性和模块化设计
- 使用POJO作为应用的构建块
- 连接企业数据和API与AI模型

### 核心特性
- 支持所有主流AI模型提供商（OpenAI、Anthropic、Google、Amazon、Azure等）
- 支持所有主流向量数据库（PGVector、Chroma、Redis、Pinecone等）
- 提供统一的、可移植的API
- 流式和同步调用支持
- 函数调用（Function Calling/Tools）
- RAG（检索增强生成）
- 对话记忆管理
- 可观测性支持
- Spring Boot自动配置

---

## 整体架构

Spring AI 采用经典的分层架构设计：

```
┌─────────────────────────────────────────────────────────┐
│           Spring Boot Starters & AutoConfiguration      │  ← Spring Boot集成层
├─────────────────────────────────────────────────────────┤
│  AI Model Implementations | Vector Store Implementations │  ← 实现层
│  (OpenAI, Anthropic, etc) | (PGVector, Chroma, etc)     │
├─────────────────────────────────────────────────────────┤
│    ChatClient API    |    Vector Store API   |   RAG    │  ← 应用层
├─────────────────────────────────────────────────────────┤
│       Model Abstractions      |   Commons & Utils        │  ← 抽象层
│  (ChatModel, EmbeddingModel)  |   (基础设施)             │
└─────────────────────────────────────────────────────────┘
```

### 核心设计原则

1. **接口抽象**: 所有AI模型和向量存储都基于统一接口
2. **可移植性**: 可轻松切换不同的AI提供商而不改变业务代码
3. **Spring集成**: 深度集成Spring Boot生态
4. **响应式支持**: 支持同步和流式（Reactor）调用
5. **可扩展性**: 易于添加新的模型提供商和功能

---

## 核心模块详解

### 1. spring-ai-model（最核心）

**位置**: `spring-ai-model/`

**功能**: 定义所有AI模型的抽象接口和核心数据结构

#### 核心接口

##### Model<TReq, TRes>
- 所有AI模型的根接口
- 定义了统一的 `call(request)` 方法
- 泛型设计，支持不同类型的请求和响应

##### ChatModel
```java
public interface ChatModel extends Model<Prompt, ChatResponse>, StreamingChatModel {
    // 同步调用
    ChatResponse call(Prompt prompt);
    
    // 流式调用
    Flux<ChatResponse> stream(Prompt prompt);
    
    // 便捷方法
    String call(String message);
}
```

**功能**:
- 聊天对话模型的抽象
- 支持单轮和多轮对话
- 支持流式响应
- 支持函数调用（Tools）

##### EmbeddingModel
```java
public interface EmbeddingModel {
    EmbeddingResponse call(EmbeddingRequest request);
    List<Double> embed(String text);
    List<Double> embed(Document document);
}
```

**功能**:
- 文本嵌入模型的抽象
- 将文本转换为向量表示
- 支持批量嵌入

##### ImageModel
**功能**:
- 图像生成模型的抽象（如DALL-E、Stable Diffusion）
- 文本到图像生成

##### TranscriptionModel & TextToSpeechModel
**功能**:
- 语音转文本（Whisper等）
- 文本转语音

##### ModerationModel
**功能**:
- 内容审核模型
- 检测不当内容

#### 核心数据结构

##### Prompt（请求）
```
Prompt
├── List<Message>  (消息列表)
│   ├── SystemMessage    (系统提示)
│   ├── UserMessage      (用户消息)
│   ├── AssistantMessage (助手回复)
│   └── ToolMessage      (工具调用结果)
└── ChatOptions          (调用选项)
    ├── model
    ├── temperature
    ├── maxTokens
    └── ...
```

##### ChatResponse（响应）
```
ChatResponse
├── List<Generation>     (生成结果列表)
│   └── AssistantMessage (实际内容)
└── ResponseMetadata     (元数据)
    ├── usage (token使用量)
    └── finishReason
```

#### 包结构

```
org.springframework.ai
├── model/              # 核心模型接口
│   ├── Model.java
│   ├── ModelRequest.java
│   ├── ModelResponse.java
│   └── tool/          # 工具/函数调用支持
├── chat/              # 聊天模型相关
│   ├── model/         # ChatModel接口
│   ├── messages/      # 消息类型
│   ├── prompt/        # 提示词相关
│   └── metadata/      # 元数据
├── embedding/         # 嵌入模型相关
├── image/            # 图像模型相关
├── audio/            # 音频模型相关
│   ├── transcription/ # 语音转文本
│   └── tts/          # 文本转语音
├── moderation/       # 内容审核
└── tool/             # 工具/函数定义和执行
```

---

### 2. spring-ai-client-chat

**位置**: `spring-ai-client-chat/`

**功能**: 提供流畅的ChatClient API，简化AI模型调用

#### ChatClient API

这是一个类似于WebClient/RestClient的流式API设计：

```java
// 创建客户端
ChatClient chatClient = ChatClient.builder(chatModel).build();

// 简单调用
String response = chatClient
    .prompt("你好，世界")
    .call()
    .content();

// 复杂调用
ChatResponse response = chatClient
    .prompt()
    .system("你是一个友好的助手")
    .user("讲个笑话")
    .options(ChatOptionsBuilder.builder()
        .withTemperature(0.8)
        .withMaxTokens(100)
        .build())
    .call()
    .chatResponse();

// 流式调用
Flux<String> stream = chatClient
    .prompt("写一首诗")
    .stream()
    .content();
```

#### Advisor机制

**Advisor** 是一种强大的拦截器模式，可以在调用AI模型前后进行处理：

##### 内置Advisors

1. **MessageChatMemoryAdvisor**: 对话记忆管理
   - 自动保存和加载历史对话
   - 支持不同的存储后端

2. **QuestionAnswerAdvisor**: RAG问答增强
   - 自动检索相关文档
   - 将文档注入到上下文中

3. **LoggingAdvisor**: 日志记录
   - 记录请求和响应

4. **PromptChatMemoryAdvisor**: 提示词记忆

##### Advisor使用示例

```java
chatClient
    .prompt("继续我们之前的话题")
    .advisors(new MessageChatMemoryAdvisor(chatMemory))
    .call()
    .content();
```

#### 核心类

- `ChatClient`: 主接口
- `DefaultChatClient`: 默认实现
- `ChatClientRequest`: 请求封装
- `Advisor`: 顾问/拦截器接口

---

### 3. spring-ai-vector-store

**位置**: `spring-ai-vector-store/`

**功能**: 向量数据库的抽象层

#### VectorStore接口

```java
public interface VectorStore extends DocumentWriter, VectorStoreRetriever {
    // 添加文档
    void add(List<Document> documents);
    
    // 删除文档
    void delete(List<String> idList);
    void delete(Filter.Expression filterExpression);
    
    // 相似度搜索
    List<Document> similaritySearch(SearchRequest request);
    List<Document> similaritySearch(String query);
}
```

#### Document结构

```java
public class Document {
    private String id;              // 文档ID
    private String text;            // 文档内容
    private Map<String, Object> metadata;  // 元数据
    private float[] embedding;      // 向量（可选）
}
```

#### SearchRequest

```java
SearchRequest request = SearchRequest.builder()
    .query("Spring AI教程")          // 查询文本
    .topK(5)                        // 返回前5个结果
    .similarityThreshold(0.7)       // 相似度阈值
    .filterExpression("type == 'tutorial'")  // 元数据过滤
    .build();
```

#### 元数据过滤器

支持类SQL的过滤表达式：

```java
// 简单条件
"type == 'tutorial'"
"price > 100"

// 复杂条件
"(type == 'tutorial' AND language == 'zh') OR priority > 5"

// IN操作
"category IN ['java', 'spring', 'ai']"
```

#### VectorStoreRetriever

这是一个只读接口，用于RAG场景：

```java
@FunctionalInterface
public interface VectorStoreRetriever {
    List<Document> similaritySearch(SearchRequest request);
}
```

---

### 4. spring-ai-commons

**位置**: `spring-ai-commons/`

**功能**: 提供公共基础设施和工具类

#### 主要内容

1. **文档处理**
   - `Document`: 文档类
   - `DocumentReader`: 文档读取器接口
   - `DocumentWriter`: 文档写入器接口
   - `DocumentTransformer`: 文档转换器

2. **模板引擎**
   - `PromptTemplate`: 提示词模板
   - 支持参数化提示词

3. **工具类**
   - JSON处理
   - 异常定义
   - 通用工具

---

### 5. spring-ai-rag

**位置**: `spring-ai-rag/`

**功能**: RAG（检索增强生成）框架

#### ETL Pipeline

提供文档处理流水线：

```
Documents → Transform → Split → Embed → Store
```

**典型流程**:
1. **读取**: 从各种来源读取文档（PDF、网页等）
2. **转换**: 清洗和标准化文档
3. **分块**: 将大文档分割成小块
4. **嵌入**: 生成向量表示
5. **存储**: 保存到向量数据库

#### 核心组件

- `DocumentReader`: 文档读取
- `DocumentTransformer`: 文档转换
- `TokenTextSplitter`: 文本分块
- `VectorStore`: 向量存储

---

### 6. spring-ai-retry

**位置**: `spring-ai-retry/`

**功能**: 重试机制支持

- 基于Spring Retry
- 处理AI API的瞬时故障
- 指数退避策略
- 速率限制处理

---

### 7. spring-ai-test

**位置**: `spring-ai-test/`

**功能**: 测试工具和辅助类

- AI模型评估工具
- 测试数据生成
- Mock实现

---

## 实现模块

### AI模型实现（models/）

每个模型提供商都有独立的模块：

#### spring-ai-openai ⭐（最完整的实现）

**位置**: `models/spring-ai-openai/`

**支持的模型**:
- GPT-4, GPT-4 Turbo, GPT-5
- GPT-3.5 Turbo
- DALL-E 3 (图像生成)
- Whisper (语音转文本)
- TTS (文本转语音)
- Embeddings (text-embedding-ada-002等)
- Moderation

**核心类**:
- `OpenAiChatModel`: ChatModel实现
- `OpenAiEmbeddingModel`: EmbeddingModel实现
- `OpenAiImageModel`: ImageModel实现
- `OpenAiAudioTranscriptionModel`: 语音转文本
- `OpenAiAudioSpeechModel`: 文本转语音

**特色功能**:
- 完整的流式支持
- Function Calling
- Vision（图像理解）
- JSON模式输出
- Seed支持（可重复生成）

#### spring-ai-anthropic

**支持的模型**: Claude 3 系列

#### spring-ai-ollama

**支持的模型**: 本地运行的开源模型
- Llama 2/3
- Mistral
- Gemma
- 等

#### spring-ai-azure-openai

**支持的模型**: Azure OpenAI Service

#### spring-ai-google-genai

**支持的模型**: Gemini系列

#### spring-ai-bedrock

**支持的模型**: AWS Bedrock上的各种模型

#### 其他模型提供商

- `spring-ai-mistral-ai`: Mistral AI
- `spring-ai-zhipuai`: 智谱AI（GLM）
- `spring-ai-deepseek`: DeepSeek
- `spring-ai-minimax`: MiniMax
- `spring-ai-huggingface`: HuggingFace
- `spring-ai-vertex-ai-gemini`: Google Vertex AI
- `spring-ai-stability-ai`: Stable Diffusion
- `spring-ai-elevenlabs`: 语音生成

---

### 向量数据库实现（vector-stores/）

#### spring-ai-pgvector-store ⭐（推荐用于生产）

**技术栈**: PostgreSQL + pgvector扩展

**特点**:
- 成熟稳定
- 支持ACID事务
- 丰富的查询能力
- 性能优秀

#### spring-ai-chroma-store

**技术栈**: ChromaDB

**特点**:
- 轻量级
- 易于开发和测试
- 支持Docker部署

#### spring-ai-redis-store

**技术栈**: Redis + RedisSearch

**特点**:
- 高性能
- 低延迟
- 支持混合查询

#### spring-ai-milvus-store

**技术栈**: Milvus

**特点**:
- 专为向量搜索设计
- 可扩展性强
- 支持GPU加速

#### 其他向量存储

- `spring-ai-pinecone-store`: Pinecone（托管服务）
- `spring-ai-weaviate-store`: Weaviate
- `spring-ai-qdrant-store`: Qdrant
- `spring-ai-elasticsearch-store`: Elasticsearch
- `spring-ai-opensearch-store`: OpenSearch
- `spring-ai-mongodb-atlas-store`: MongoDB Atlas
- `spring-ai-neo4j-store`: Neo4j
- `spring-ai-cassandra-store`: Cassandra
- `spring-ai-azure-store`: Azure Cognitive Search
- `spring-ai-azure-cosmos-db-store`: Azure Cosmos DB
- `spring-ai-oracle-store`: Oracle Database
- `spring-ai-mariadb-store`: MariaDB
- `spring-ai-coherence-store`: Oracle Coherence
- `spring-ai-gemfire-store`: VMware GemFire
- `spring-ai-hanadb-store`: SAP HANA
- `spring-ai-couchbase-store`: Couchbase
- `spring-ai-typesense-store`: Typesense

---

### 文档读取器（document-readers/）

#### spring-ai-pdf-reader

**功能**: 读取PDF文档

**特点**:
- 支持多种PDF格式
- 提取文本和元数据
- 支持OCR（可选）

#### spring-ai-tika-reader

**功能**: 使用Apache Tika读取多种格式

**支持格式**:
- Microsoft Office (Word, Excel, PowerPoint)
- PDF
- HTML
- XML
- 等

#### spring-ai-markdown-reader

**功能**: 读取Markdown文档

#### spring-ai-jsoup-reader

**功能**: 读取和解析HTML/网页

---

## 集成模块

### Auto Configuration（auto-configurations/）

Spring Boot自动配置模块，为每个AI提供商和向量存储提供开箱即用的配置。

#### 结构

```
auto-configurations/
├── common/
│   └── spring-ai-autoconfigure-retry/          # 重试配置
├── models/
│   ├── chat/
│   │   ├── client/                              # ChatClient配置
│   │   ├── memory/                              # 对话记忆配置
│   │   └── observation/                         # 可观测性配置
│   ├── spring-ai-autoconfigure-model-openai/   # OpenAI配置
│   ├── spring-ai-autoconfigure-model-anthropic/ # Anthropic配置
│   └── ...
├── vector-stores/
│   ├── spring-ai-autoconfigure-vector-store-pgvector/
│   ├── spring-ai-autoconfigure-vector-store-chroma/
│   └── ...
└── mcp/                                         # MCP配置
```

#### 典型的AutoConfiguration

```java
@AutoConfiguration
@ConditionalOnClass(OpenAiApi.class)
@EnableConfigurationProperties(OpenAiChatProperties.class)
public class OpenAiAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public OpenAiChatModel openAiChatModel(
            OpenAiChatProperties properties) {
        return OpenAiChatModel.builder()
            .apiKey(properties.getApiKey())
            .model(properties.getModel())
            .build();
    }
}
```

---

### Spring Boot Starters（spring-ai-spring-boot-starters/）

预配置的依赖包，简化项目配置。

#### 模型Starters

```xml
<!-- OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-openai</artifactId>
</dependency>

<!-- Ollama -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-ollama</artifactId>
</dependency>
```

#### 向量存储Starters

```xml
<!-- PGVector -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>

<!-- Chroma -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-chroma</artifactId>
</dependency>
```

#### 配置示例

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4
          temperature: 0.7
          max-tokens: 500
    vectorstore:
      pgvector:
        host: localhost
        port: 5432
        database: vectordb
```

---

### Docker Compose & Testcontainers

#### spring-ai-spring-boot-docker-compose

**功能**: Docker Compose集成
- 自动启动依赖服务（PostgreSQL、Redis等）
- 开发环境配置简化

#### spring-ai-spring-boot-testcontainers

**功能**: Testcontainers集成
- 集成测试支持
- 自动化测试环境

---

## 辅助模块

### Memory（memory/）

对话记忆管理，支持多种存储后端。

#### 实现

- `spring-ai-model-chat-memory-repository-jdbc`: 基于JDBC
- `spring-ai-model-chat-memory-repository-cassandra`: 基于Cassandra
- `spring-ai-model-chat-memory-repository-neo4j`: 基于Neo4j

#### 使用示例

```java
ChatMemory chatMemory = new CassandraChatMemory(cassandraTemplate);

chatClient
    .prompt("你好")
    .advisors(new MessageChatMemoryAdvisor(chatMemory))
    .call()
    .content();
```

---

### Advisors（advisors/）

#### spring-ai-advisors-vector-store

提供开箱即用的RAG顾问实现。

---

### MCP - Model Context Protocol（mcp/）

**新特性**: 模型上下文协议支持

#### 模块

- `mcp/common`: 公共定义
- `mcp/mcp-annotations-spring`: Spring注解支持

#### AutoConfiguration

- `spring-ai-autoconfigure-mcp-client-*`: MCP客户端配置
- `spring-ai-autoconfigure-mcp-server-*`: MCP服务端配置

---

### BOM（spring-ai-bom/）

**功能**: 统一版本管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>1.1.0-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

### 文档（spring-ai-docs/）

**功能**: 官方文档源码

使用Antora构建，包含：
- API参考文档
- 使用指南
- 示例代码
- 最佳实践

---

### Spring Cloud Bindings（spring-ai-spring-cloud-bindings/）

**功能**: Cloud Foundry和Kubernetes绑定支持

自动从平台服务绑定中读取配置。

---

## 模块依赖关系

### 分层依赖

```
┌─────────────────────────────────────────┐
│          Starters                       │
└────────────┬────────────────────────────┘
             │ depends on
┌────────────▼────────────────────────────┐
│      Auto Configurations                │
└────────────┬────────────────────────────┘
             │ depends on
┌────────────▼────────────────────────────┐
│    Model/VectorStore Implementations    │
└────────────┬────────────────────────────┘
             │ depends on
┌────────────▼────────────────────────────┐
│  ChatClient, VectorStore, RAG           │
└────────────┬────────────────────────────┘
             │ depends on
┌────────────▼────────────────────────────┐
│    spring-ai-model, spring-ai-commons   │
└─────────────────────────────────────────┘
```

### 核心依赖链

1. **最底层**: `spring-ai-commons` - 无依赖
2. **抽象层**: `spring-ai-model` - 依赖commons
3. **应用层**: `spring-ai-client-chat`, `spring-ai-vector-store`, `spring-ai-rag`
4. **实现层**: 各具体实现依赖相应的抽象
5. **集成层**: AutoConfiguration依赖具体实现
6. **入口层**: Starters依赖AutoConfiguration

---

## 学习路径建议

### 第一周：核心抽象（理解框架设计）

#### Day 1-2: Model抽象层
**阅读文件**:
1. `spring-ai-model/src/main/java/org/springframework/ai/model/Model.java`
2. `spring-ai-model/src/main/java/org/springframework/ai/chat/model/ChatModel.java`
3. `spring-ai-model/src/main/java/org/springframework/ai/chat/prompt/Prompt.java`
4. `spring-ai-model/src/main/java/org/springframework/ai/chat/model/ChatResponse.java`
5. `spring-ai-model/src/main/java/org/springframework/ai/embedding/EmbeddingModel.java`

**理解要点**:
- 泛型设计如何提供灵活性
- 请求-响应模式
- 同步vs流式调用

#### Day 3-4: ChatClient API
**阅读文件**:
1. `spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/ChatClient.java`
2. `spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/DefaultChatClient.java`
3. `spring-ai-client-chat/src/main/java/org/springframework/ai/chat/client/advisor/api/Advisor.java`

**理解要点**:
- 流式API设计
- Builder模式
- Advisor拦截器模式

#### Day 5-7: VectorStore & RAG
**阅读文件**:
1. `spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/VectorStore.java`
2. `spring-ai-vector-store/src/main/java/org/springframework/ai/vectorstore/SearchRequest.java`
3. `spring-ai-commons/src/main/java/org/springframework/ai/document/Document.java`
4. `spring-ai-rag/` 目录浏览

**理解要点**:
- 向量存储的工作原理
- 文档分块策略
- RAG流程

---

### 第二周：具体实现（了解如何实现接口）

#### Day 8-10: OpenAI实现
**阅读文件**:
1. `models/spring-ai-openai/src/main/java/org/springframework/ai/openai/OpenAiChatModel.java`
2. `models/spring-ai-openai/src/main/java/org/springframework/ai/openai/api/OpenAiApi.java`
3. 测试用例：`models/spring-ai-openai/src/test/java/`

**理解要点**:
- HTTP客户端调用
- 流式响应处理（SSE）
- 错误处理和重试
- Function Calling实现

#### Day 11-12: PGVector实现
**阅读文件**:
1. `vector-stores/spring-ai-pgvector-store/src/main/java/org/springframework/ai/vectorstore/PgVectorStore.java`

**理解要点**:
- SQL查询构建
- 向量相似度计算
- 元数据过滤实现

#### Day 13-14: 其他实现浏览
- 选择1-2个感兴趣的实现快速浏览
- 对比不同实现的差异

---

### 第三周：Spring Boot集成（了解开箱即用）

#### Day 15-17: AutoConfiguration
**阅读文件**:
1. `auto-configurations/models/spring-ai-autoconfigure-model-openai/`
2. 查看配置属性类
3. 查看自动配置类

**理解要点**:
- `@AutoConfiguration`
- `@ConditionalOnClass`
- `@EnableConfigurationProperties`
- 配置优先级

#### Day 18-19: Starter使用
**实践**:
1. 创建一个简单的Spring Boot项目
2. 添加OpenAI Starter
3. 配置application.yml
4. 编写简单的Controller调用AI

#### Day 20-21: 高级特性
**学习内容**:
1. Advisors深入
2. Memory实现
3. Observability（可观测性）
4. MCP协议

---

## 调试和实践建议

### 1. 从单元测试入手

每个模块都有丰富的测试用例，是学习API最好的材料：

```bash
# OpenAI测试
models/spring-ai-openai/src/test/java/

# VectorStore测试  
vector-stores/spring-ai-pgvector-store/src/test/java/
```

### 2. 使用调试器

在关键方法设置断点：
- `ChatModel.call()`
- `VectorStore.similaritySearch()`
- `Advisor.aroundCall()`

跟踪完整的调用链路。

### 3. 画图理解

- 画类图：理解继承和实现关系
- 画时序图：理解方法调用流程
- 画架构图：理解模块依赖关系

### 4. 编写示例代码

创建自己的测试项目：

```java
@SpringBootApplication
public class SpringAiLearningApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(SpringAiLearningApplication.class, args);
    }
    
    @Bean
    CommandLineRunner runner(ChatClient chatClient) {
        return args -> {
            String response = chatClient
                .prompt("解释Spring AI的核心设计")
                .call()
                .content();
            System.out.println(response);
        };
    }
}
```

### 5. 阅读官方文档

配合源码阅读官方文档：
- https://docs.spring.io/spring-ai/reference/

---

## 关键技术点总结

### 1. 设计模式

- **模板方法**: `Model`接口
- **策略模式**: 不同的AI提供商实现
- **建造者模式**: `ChatClient.Builder`
- **责任链模式**: `Advisor`链
- **适配器模式**: 各种模型适配统一接口
- **装饰器模式**: `Advisor`装饰调用

### 2. Spring技术栈

- **依赖注入**: 所有组件都是Bean
- **自动配置**: `@AutoConfiguration`
- **条件装配**: `@Conditional*`
- **配置属性**: `@ConfigurationProperties`
- **事件机制**: 可观测性事件

### 3. 响应式编程

- **Reactor**: `Flux<ChatResponse>`
- **背压**: 流式响应控制
- **非阻塞**: WebFlux支持

### 4. 可观测性

- **Micrometer**: 指标收集
- **Observation API**: 统一的观测接口
- **分布式追踪**: Trace支持

---

## 常见问题解答

### Q1: 如何选择AI模型提供商？

**开发环境**:
- 推荐：Ollama（本地免费）
- 备选：OpenAI（需要API Key）

**生产环境**:
- 性能要求高：OpenAI GPT-4
- 成本敏感：OpenAI GPT-3.5 或 Anthropic Claude
- 国内部署：智谱AI、DeepSeek等

### Q2: 如何选择向量数据库？

**小型项目**: 
- SimpleVectorStore（内存）
- ChromaDB（轻量级）

**中型项目**:
- PGVector（PostgreSQL扩展）
- Redis

**大型项目**:
- Milvus
- Pinecone（托管服务）

### Q3: RAG的最佳实践？

1. **文档分块**:
   - 大小：500-1000 tokens
   - 重叠：50-100 tokens

2. **检索策略**:
   - Top-K: 3-5个文档
   - 相似度阈值: 0.7+

3. **提示词工程**:
   - 明确指示使用检索到的内容
   - 要求引用来源

### Q4: 如何处理超长上下文？

1. **分段处理**: 将长文本分段
2. **摘要**: 先总结再处理
3. **滑动窗口**: 保留最近的N轮对话
4. **向量检索**: 使用RAG检索相关内容

---

## 项目统计信息

- **版本**: 1.1.0-SNAPSHOT
- **Java版本**: 17+
- **Spring Boot版本**: 3.5.6
- **总模块数**: 100+
- **支持的AI提供商**: 15+
- **支持的向量数据库**: 20+
- **代码行数**: 约100万行（包括测试）

---

## 参考资源

### 官方资源
- **官方文档**: https://docs.spring.io/spring-ai/reference/
- **GitHub仓库**: https://github.com/spring-projects/spring-ai
- **示例项目**: https://github.com/spring-projects/spring-ai-examples

### 社区资源
- **Awesome Spring AI**: https://github.com/spring-ai-community/awesome-spring-ai
- **Spring AI社区**: https://github.com/spring-ai-community

### 博客文章
- **Why Spring AI**: https://spring.io/blog/2024/11/19/why-spring-ai

---

## 更新日志

- **2025-10-02**: 初始版本，基于Spring AI 1.1.0-SNAPSHOT

---

## 文档维护

本文档由AI助手生成，建议根据项目实际情况更新。

**维护建议**:
1. 跟随Spring AI版本更新
2. 补充实际使用经验
3. 添加性能优化建议
4. 记录遇到的问题和解决方案

