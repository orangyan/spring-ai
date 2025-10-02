# Spring AI RAG 检索增强生成详解

## 📋 目录
- [概述](#概述)
- [RAG核心概念](#rag核心概念)
- [RAG工作流程](#rag工作流程)
- [QuestionAnswerAdvisor](#questionansweradvisor)
- [VectorStoreChatMemoryAdvisor](#vectorstorechatmemoryadvisor)
- [自定义RAG实现](#自定义rag实现)
- [文档处理](#文档处理)
- [高级RAG模式](#高级rag模式)
- [性能优化](#性能优化)
- [实战案例](#实战案例)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是RAG？

**RAG (Retrieval Augmented Generation)** 即检索增强生成，是一种结合**信息检索**和**大语言模型生成**的技术模式。

```
传统LLM:
用户问题 → LLM → 回答
问题: 只能基于训练数据，无法获取最新信息

RAG:
用户问题 → 检索相关文档 → 构建上下文 → LLM + 上下文 → 回答
优势: 可以利用外部知识库，回答更准确、更新
```

### 为什么需要RAG？

#### 大语言模型的局限性

1. **知识截止日期**: 训练数据有时间限制
2. **幻觉问题**: 可能生成不存在的事实
3. **领域知识不足**: 专业领域内容不够深入
4. **无法访问私有数据**: 不知道企业内部信息

#### RAG的优势

1. ✅ **知识实时更新**: 向量库可随时更新
2. ✅ **回答有据可查**: 提供信息来源
3. ✅ **降低幻觉**: 基于真实文档回答
4. ✅ **支持私有数据**: 可用于企业知识库
5. ✅ **成本更低**: 无需重新训练模型

### Spring AI的RAG支持

Spring AI提供两种RAG实现方式：

1. **Advisor API**: 开箱即用的RAG组件（推荐）
2. **手动实现**: 完全自定义的RAG流程

---

## RAG核心概念

### 1. RAG架构

```
┌─────────────────────────────────────────────────────────────┐
│                        RAG系统                                │
└─────────────────────────────────────────────────────────────┘

第一阶段：文档处理和索引
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│原始文档  │ => │文档分块  │ => │生成向量  │ => │存入向量库│
│(PDF等)   │    │(Chunking)│    │(Embedding)│   │(VectorDB)│
└──────────┘    └──────────┘    └──────────┘    └──────────┘

第二阶段：查询和生成
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│用户问题  │ => │向量检索  │ => │构建Prompt│ => │LLM生成  │
│(Query)   │    │(Retrieval)│   │(Augment) │    │(Generate)│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                      ↑
                      │
                ┌─────┴──────┐
                │  VectorDB  │
                └────────────┘
```

### 2. RAG的三个核心步骤

#### Retrieval（检索）

从向量数据库中检索与问题相关的文档。

```java
// 1. 将问题转换为向量
float[] queryEmbedding = embeddingModel.embed(question);

// 2. 相似性搜索
List<Document> relevantDocs = vectorStore.similaritySearch(
    SearchRequest.builder()
        .query(question)
        .topK(5)
        .similarityThreshold(0.7)
        .build()
);
```

#### Augmentation（增强）

将检索到的文档作为上下文，增强用户的原始问题。

```java
// 构建上下文
String context = relevantDocs.stream()
    .map(Document::getText)
    .collect(Collectors.joining("\n\n"));

// 增强Prompt
String augmentedPrompt = """
    根据以下上下文回答问题：
    
    上下文：
    %s
    
    问题：%s
    
    请基于上下文回答，如果上下文中没有相关信息，请说明。
    """.formatted(context, question);
```

#### Generation（生成）

使用LLM根据增强后的Prompt生成答案。

```java
String answer = chatModel.call(augmentedPrompt);
```

### 3. RAG vs Fine-tuning

| 特性 | RAG | Fine-tuning |
|------|-----|-------------|
| **知识更新** | 实时更新向量库 | 需要重新训练 |
| **成本** | 低（只需API调用） | 高（需要GPU训练） |
| **实施难度** | 简单 | 复杂 |
| **可解释性** | 好（可追溯来源） | 差（黑盒） |
| **专业性** | 中等 | 高（深度定制） |
| **适用场景** | 知识库问答、文档搜索 | 特定领域任务 |

---

## RAG工作流程

### 完整流程图

```
离线阶段（数据准备）:
┌────────────┐
│ 1. 收集文档 │
│ (PDF/Word) │
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 2. 文档分块 │
│ (Chunking) │
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 3. 生成向量 │
│ (Embedding)│
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 4. 存入向量库│
│ (VectorDB) │
└────────────┘

在线阶段（查询响应）:
┌────────────┐
│ 1. 用户提问 │
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 2. 查询向量化│
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 3. 相似性搜索│
│ (Top K)    │
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 4. 重排序   │
│ (Reranking)│ (可选)
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 5. 构建上下文│
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 6. LLM生成  │
└──────┬─────┘
       │
       ▼
┌────────────┐
│ 7. 返回答案 │
│ + 来源     │
└────────────┘
```

### 简单实现示例

```java
@Service
public class SimpleRAGService {
    
    private final VectorStore vectorStore;
    private final ChatModel chatModel;
    
    /**
     * RAG完整流程
     */
    public String askQuestion(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );
        
        // 2. 构建上下文
        String context = relevantDocs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        // 3. 构建增强Prompt
        String augmentedPrompt = buildPrompt(question, context);
        
        // 4. 生成答案
        return chatModel.call(augmentedPrompt);
    }
    
    private String buildPrompt(String question, String context) {
        return """
            你是一个专业的问答助手。请根据以下上下文回答用户的问题。
            
            上下文信息：
            %s
            
            用户问题：%s
            
            回答要求：
            1. 仅基于上下文信息回答
            2. 如果上下文中没有相关信息，请明确说明
            3. 回答要准确、简洁
            
            回答：
            """.formatted(context, question);
    }
}
```

---

## QuestionAnswerAdvisor

`QuestionAnswerAdvisor`是Spring AI提供的开箱即用的RAG实现，基于Advisor机制。

### 核心特性

1. ✅ 自动检索相关文档
2. ✅ 自动构建增强Prompt
3. ✅ 支持动态过滤
4. ✅ 可自定义Prompt模板
5. ✅ 集成ChatClient

### 基本使用

```java
@Configuration
public class RAGConfig {
    
    @Bean
    public ChatClient ragChatClient(
            ChatModel chatModel,
            VectorStore vectorStore) {
        
        // 创建QuestionAnswerAdvisor
        QuestionAnswerAdvisor qaAdvisor = 
            QuestionAnswerAdvisor.builder(vectorStore)
                .searchRequest(SearchRequest.builder()
                    .topK(5)
                    .similarityThreshold(0.7)
                    .build())
                .build();
        
        // 配置ChatClient
        return ChatClient.builder(chatModel)
            .defaultSystem("你是一个专业的问答助手")
            .defaultAdvisors(qaAdvisor)
            .build();
    }
}

@Service
public class QAService {
    
    private final ChatClient chatClient;
    
    /**
     * 使用RAG回答问题
     */
    public String answer(String question) {
        // Advisor自动处理RAG流程
        return chatClient
            .prompt(question)
            .call()
            .content();
    }
}
```

### 高级配置

```java
@Configuration
public class AdvancedRAGConfig {
    
    @Bean
    public QuestionAnswerAdvisor questionAnswerAdvisor(
            VectorStore vectorStore) {
        
        return QuestionAnswerAdvisor.builder(vectorStore)
            // 1. 配置搜索参数
            .searchRequest(SearchRequest.builder()
                .topK(5)
                .similarityThreshold(0.75)
                .build())
            
            // 2. 自定义Prompt模板
            .promptTemplate(new PromptTemplate("""
                {query}
                
                以下是相关的背景信息：
                ---------------------
                {question_answer_context}
                ---------------------
                
                请基于以上信息回答问题。如果信息不足，请说明。
                """))
            
            // 3. 设置执行顺序
            .order(0)
            
            // 4. 配置调度器（防止阻塞）
            .protectFromBlocking(true)
            
            .build();
    }
}
```

### 动态过滤

```java
@Service
public class FilteredRAGService {
    
    private final ChatClient chatClient;
    
    /**
     * 按部门过滤
     */
    public String answerByDepartment(
            String question,
            String department) {
        
        return chatClient
            .prompt(question)
            .advisors(spec -> spec
                .param(
                    QuestionAnswerAdvisor.FILTER_EXPRESSION,
                    "department == '" + department + "'"
                )
            )
            .call()
            .content();
    }
    
    /**
     * 复合过滤
     */
    public String answerWithComplexFilter(
            String question,
            String tenantId,
            String category,
            int minYear) {
        
        String filter = String.format(
            "tenantId == '%s' && category == '%s' && year >= %d",
            tenantId, category, minYear
        );
        
        return chatClient
            .prompt(question)
            .advisors(spec -> spec
                .param(
                    QuestionAnswerAdvisor.FILTER_EXPRESSION,
                    filter
                )
            )
            .call()
            .content();
    }
}
```

### 获取检索的文档

```java
@Service
public class RAGWithSourcesService {
    
    private final ChatClient chatClient;
    
    /**
     * 获取答案和来源
     */
    public AnswerWithSources answerWithSources(String question) {
        // 执行RAG查询
        ChatClientResponse response = chatClient
            .prompt(question)
            .call()
            .chatClientResponse();
        
        // 获取答案
        String answer = response.chatResponse()
            .getResult()
            .getOutput()
            .getText();
        
        // 获取检索的文档（从上下文中）
        @SuppressWarnings("unchecked")
        List<Document> retrievedDocs = 
            (List<Document>) response.context()
                .get(QuestionAnswerAdvisor.RETRIEVED_DOCUMENTS);
        
        // 提取来源
        List<String> sources = retrievedDocs.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .distinct()
            .toList();
        
        return new AnswerWithSources(answer, sources, retrievedDocs);
    }
    
    record AnswerWithSources(
        String answer,
        List<String> sources,
        List<Document> documents
    ) {}
}
```

### 自定义Prompt模板

```java
@Configuration
public class CustomPromptRAGConfig {
    
    @Bean
    public QuestionAnswerAdvisor customPromptAdvisor(
            VectorStore vectorStore) {
        
        // 中文Prompt模板
        PromptTemplate chineseTemplate = new PromptTemplate("""
            问题：{query}
            
            参考资料：
            ━━━━━━━━━━━━━━━━━━━━━
            {question_answer_context}
            ━━━━━━━━━━━━━━━━━━━━━
            
            请基于上述参考资料回答问题：
            1. 如果资料中有相关信息，请直接引用并回答
            2. 如果资料中没有相关信息，请明确告知
            3. 保持回答的专业性和准确性
            4. 可以适当补充你的理解
            """);
        
        return QuestionAnswerAdvisor.builder(vectorStore)
            .promptTemplate(chineseTemplate)
            .searchRequest(SearchRequest.builder()
                .topK(5)
                .similarityThreshold(0.7)
                .build())
            .build();
    }
    
    @Bean
    public QuestionAnswerAdvisor structuredPromptAdvisor(
            VectorStore vectorStore) {
        
        // 结构化输出Prompt
        PromptTemplate structuredTemplate = new PromptTemplate("""
            问题：{query}
            
            上下文：
            {question_answer_context}
            
            请按以下格式回答：
            
            【答案】
            （简洁的答案）
            
            【详细解释】
            （详细的解释）
            
            【相关信息】
            （其他相关的补充信息）
            
            【可信度】
            （高/中/低，基于上下文信息的充分程度）
            """);
        
        return QuestionAnswerAdvisor.builder(vectorStore)
            .promptTemplate(structuredTemplate)
            .build();
    }
}
```

---

## VectorStoreChatMemoryAdvisor

`VectorStoreChatMemoryAdvisor`使用向量库存储长期对话记忆，实现基于向量检索的对话上下文管理。

### 与MessageChatMemoryAdvisor的区别

| 特性 | MessageChatMemoryAdvisor | VectorStoreChatMemoryAdvisor |
|------|--------------------------|------------------------------|
| **存储方式** | 列表（FIFO） | 向量数据库 |
| **检索方式** | 按时间顺序 | 按相似度 |
| **容量** | 有限（通常10-20条） | 大容量 |
| **适用场景** | 短期对话 | 长期记忆 |

### 基本使用

```java
@Configuration
public class VectorMemoryConfig {
    
    @Bean
    public ChatClient memoryAwareChatClient(
            ChatModel chatModel,
            VectorStore memoryVectorStore) {
        
        VectorStoreChatMemoryAdvisor memoryAdvisor = 
            VectorStoreChatMemoryAdvisor.builder(memoryVectorStore)
                .searchRequest(SearchRequest.builder()
                    .topK(10)
                    .similarityThreshold(0.6)
                    .build())
                .build();
        
        return ChatClient.builder(chatModel)
            .defaultAdvisors(memoryAdvisor)
            .build();
    }
}
```

### 实战示例

```java
@Service
public class LongTermMemoryService {
    
    private final ChatClient chatClient;
    
    /**
     * 对话（自动管理长期记忆）
     */
    public String chat(String userId, String message) {
        return chatClient
            .prompt(message)
            .advisors(spec -> spec
                .param("conversationId", userId)
            )
            .call()
            .content();
    }
}
```

---

## 自定义RAG实现

### 基础RAG实现

```java
@Service
public class CustomRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 完整的RAG实现
     */
    public RagResponse ragQuery(String question) {
        // 1. 检索阶段
        List<Document> retrievedDocs = retrieveDocuments(question);
        
        // 2. 增强阶段
        String augmentedPrompt = augmentPrompt(question, retrievedDocs);
        
        // 3. 生成阶段
        String answer = generate(augmentedPrompt);
        
        // 4. 返回结果
        return new RagResponse(
            answer,
            extractSources(retrievedDocs),
            retrievedDocs
        );
    }
    
    private List<Document> retrieveDocuments(String question) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );
    }
    
    private String augmentPrompt(
            String question,
            List<Document> documents) {
        
        String context = documents.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return """
            上下文：
            %s
            
            问题：%s
            
            请基于上下文回答问题。
            """.formatted(context, question);
    }
    
    private String generate(String prompt) {
        return chatClient
            .prompt(prompt)
            .call()
            .content();
    }
    
    private List<String> extractSources(List<Document> documents) {
        return documents.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .distinct()
            .toList();
    }
    
    record RagResponse(
        String answer,
        List<String> sources,
        List<Document> documents
    ) {}
}
```

### 高级RAG实现

```java
@Service
public class AdvancedRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    private final RerankingService rerankingService;
    
    /**
     * 带重排序的RAG
     */
    public RagResponse ragWithReranking(String question) {
        // 1. 初步检索（多返回一些）
        List<Document> candidates = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(20)  // 初步检索更多文档
                .similarityThreshold(0.6)
                .build()
        );
        
        // 2. 重排序（使用更精确的模型）
        List<Document> rerankedDocs = 
            rerankingService.rerank(question, candidates, 5);
        
        // 3. 构建上下文和生成答案
        String context = buildContext(rerankedDocs);
        String answer = generateAnswer(question, context);
        
        return new RagResponse(answer, rerankedDocs);
    }
    
    /**
     * 混合检索（向量+关键词）
     */
    public RagResponse hybridRag(
            String question,
            List<String> keywords) {
        
        // 1. 向量检索
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(10)
                .build()
        );
        
        // 2. 关键词过滤
        List<Document> filtered = vectorResults.stream()
            .filter(doc -> containsKeywords(doc, keywords))
            .limit(5)
            .toList();
        
        // 3. 生成答案
        String context = buildContext(filtered);
        String answer = generateAnswer(question, context);
        
        return new RagResponse(answer, filtered);
    }
    
    /**
     * 迭代式RAG（需要时追加检索）
     */
    public RagResponse iterativeRag(String question) {
        List<Document> allDocs = new ArrayList<>();
        String currentAnswer = "";
        int iteration = 0;
        int maxIterations = 3;
        
        while (iteration < maxIterations) {
            // 检索
            List<Document> newDocs = vectorStore.similaritySearch(
                SearchRequest.builder()
                    .query(question)
                    .topK(5)
                    .build()
            );
            
            allDocs.addAll(newDocs);
            
            // 生成答案
            String context = buildContext(allDocs);
            currentAnswer = generateAnswer(question, context);
            
            // 检查答案是否充分
            if (isAnswerSufficient(currentAnswer)) {
                break;
            }
            
            // 优化查询
            question = refineQuery(question, currentAnswer);
            iteration++;
        }
        
        return new RagResponse(currentAnswer, allDocs);
    }
    
    private boolean containsKeywords(
            Document doc,
            List<String> keywords) {
        String text = doc.getText().toLowerCase();
        return keywords.stream()
            .anyMatch(kw -> text.contains(kw.toLowerCase()));
    }
    
    private String buildContext(List<Document> documents) {
        return documents.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
    }
    
    private String generateAnswer(String question, String context) {
        return chatClient
            .prompt(spec -> spec
                .text("""
                    上下文：{context}
                    问题：{question}
                    """)
                .param("context", context)
                .param("question", question)
            )
            .call()
            .content();
    }
    
    private boolean isAnswerSufficient(String answer) {
        // 检查答案是否包含"不知道"、"信息不足"等
        return !answer.contains("不知道") && 
               !answer.contains("信息不足");
    }
    
    private String refineQuery(String originalQuery, String partialAnswer) {
        // 根据部分答案优化查询
        return originalQuery + " " + extractKeyTerms(partialAnswer);
    }
    
    private String extractKeyTerms(String text) {
        // 提取关键术语
        return ""; // 简化实现
    }
    
    record RagResponse(
        String answer,
        List<Document> documents
    ) {}
}
```

---

## 文档处理

### 文档读取

Spring AI提供多种文档读取器。

```java
@Service
public class DocumentReaderService {
    
    /**
     * PDF文档
     */
    public List<Document> readPdf(Resource pdfResource) {
        PagePdfDocumentReader reader = 
            new PagePdfDocumentReader(pdfResource);
        return reader.get();
    }
    
    /**
     * Word文档
     */
    public List<Document> readWord(Resource wordResource) {
        TikaDocumentReader reader = 
            new TikaDocumentReader(wordResource);
        return reader.get();
    }
    
    /**
     * Markdown文档
     */
    public List<Document> readMarkdown(Resource mdResource) {
        MarkdownDocumentReader reader = 
            new MarkdownDocumentReader(mdResource);
        return reader.get();
    }
    
    /**
     * HTML文档
     */
    public List<Document> readHtml(Resource htmlResource) {
        JsoupDocumentReader reader = 
            new JsoupDocumentReader(htmlResource);
        return reader.get();
    }
    
    /**
     * 文本文件
     */
    public List<Document> readText(Resource textResource) 
            throws IOException {
        String content = textResource.getContentAsString(
            StandardCharsets.UTF_8
        );
        return List.of(new Document(content));
    }
}
```

### 文档分块

```java
@Service
public class DocumentChunkingService {
    
    /**
     * 按Token数分块
     */
    public List<Document> chunkByTokens(
            Document document,
            int maxTokens,
            int overlap) {
        
        TokenTextSplitter splitter = new TokenTextSplitter(
            maxTokens,
            overlap
        );
        
        return splitter.split(document);
    }
    
    /**
     * 按字符数分块
     */
    public List<Document> chunkByCharacters(
            Document document,
            int chunkSize,
            int overlap) {
        
        List<Document> chunks = new ArrayList<>();
        String text = document.getText();
        int start = 0;
        int chunkId = 0;
        
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            String chunk = text.substring(start, end);
            
            Document doc = Document.builder()
                .text(chunk)
                .metadata("sourceId", document.getId())
                .metadata("chunkId", chunkId++)
                .metadata("chunkStart", start)
                .metadata(document.getMetadata())
                .build();
            
            chunks.add(doc);
            start += chunkSize - overlap;
        }
        
        return chunks;
    }
    
    /**
     * 语义分块（保持段落完整）
     */
    public List<Document> semanticChunking(Document document) {
        String[] paragraphs = document.getText().split("\n\n+");
        
        List<Document> chunks = new ArrayList<>();
        StringBuilder currentChunk = new StringBuilder();
        int chunkId = 0;
        int maxChunkSize = 1000;
        
        for (String paragraph : paragraphs) {
            if (currentChunk.length() + paragraph.length() > maxChunkSize) {
                // 保存当前块
                if (currentChunk.length() > 0) {
                    chunks.add(createChunk(
                        currentChunk.toString(),
                        document,
                        chunkId++
                    ));
                    currentChunk = new StringBuilder();
                }
            }
            
            currentChunk.append(paragraph).append("\n\n");
        }
        
        // 保存最后一块
        if (currentChunk.length() > 0) {
            chunks.add(createChunk(
                currentChunk.toString(),
                document,
                chunkId
            ));
        }
        
        return chunks;
    }
    
    private Document createChunk(
            String text,
            Document source,
            int chunkId) {
        
        return Document.builder()
            .text(text.trim())
            .metadata("sourceId", source.getId())
            .metadata("chunkId", chunkId)
            .metadata(source.getMetadata())
            .build();
    }
}
```

### 完整文档处理流程

```java
@Service
public class DocumentIngestionPipeline {
    
    private final DocumentReaderService readerService;
    private final DocumentChunkingService chunkingService;
    private final VectorStore vectorStore;
    
    /**
     * 完整的文档导入流程
     */
    public void ingestDocument(
            Resource resource,
            Map<String, Object> metadata) {
        
        // 1. 读取文档
        List<Document> documents = readDocument(resource);
        logger.info("Read {} documents from {}", 
            documents.size(), 
            resource.getFilename()
        );
        
        // 2. 分块
        List<Document> chunks = documents.stream()
            .flatMap(doc -> chunkingService
                .chunkByTokens(doc, 500, 50)
                .stream()
            )
            .toList();
        logger.info("Split into {} chunks", chunks.size());
        
        // 3. 添加元数据
        chunks.forEach(chunk -> {
            chunk.getMetadata().put("source", resource.getFilename());
            chunk.getMetadata().put("ingestedAt", Instant.now());
            chunk.getMetadata().putAll(metadata);
        });
        
        // 4. 导入向量库
        vectorStore.add(chunks);
        logger.info("Ingested {} chunks to vector store", chunks.size());
    }
    
    /**
     * 批量导入
     */
    public void batchIngest(
            List<Resource> resources,
            Map<String, Object> commonMetadata) {
        
        for (Resource resource : resources) {
            try {
                ingestDocument(resource, commonMetadata);
            } catch (Exception e) {
                logger.error("Failed to ingest: {}", 
                    resource.getFilename(), e);
            }
        }
    }
    
    private List<Document> readDocument(Resource resource) {
        String filename = resource.getFilename();
        
        if (filename == null) {
            throw new IllegalArgumentException("Resource has no filename");
        }
        
        if (filename.endsWith(".pdf")) {
            return readerService.readPdf(resource);
        } else if (filename.endsWith(".docx") || filename.endsWith(".doc")) {
            return readerService.readWord(resource);
        } else if (filename.endsWith(".md")) {
            return readerService.readMarkdown(resource);
        } else if (filename.endsWith(".html")) {
            return readerService.readHtml(resource);
        } else {
            try {
                return readerService.readText(resource);
            } catch (IOException e) {
                throw new RuntimeException("Failed to read document", e);
            }
        }
    }
}
```

---

## 高级RAG模式

### 1. 多跳RAG（Multi-Hop RAG）

用于需要多次检索的复杂问题。

```java
@Service
public class MultiHopRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 多跳RAG
     */
    public String multiHopRag(String complexQuestion) {
        List<Document> allRelevantDocs = new ArrayList<>();
        String currentQuery = complexQuestion;
        int maxHops = 3;
        
        for (int hop = 0; hop < maxHops; hop++) {
            // 检索
            List<Document> docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                    .query(currentQuery)
                    .topK(5)
                    .build()
            );
            
            allRelevantDocs.addAll(docs);
            
            // 判断是否需要继续
            if (hop < maxHops - 1) {
                // 生成下一跳的查询
                currentQuery = generateFollowUpQuery(
                    complexQuestion,
                    docs,
                    hop
                );
            }
        }
        
        // 最终生成答案
        String context = allRelevantDocs.stream()
            .map(Document::getText)
            .distinct()
            .collect(Collectors.joining("\n\n"));
        
        return generateAnswer(complexQuestion, context);
    }
    
    private String generateFollowUpQuery(
            String originalQuestion,
            List<Document> previousDocs,
            int hop) {
        
        String context = previousDocs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n"));
        
        return chatClient
            .prompt(spec -> spec
                .text("""
                    原始问题：{question}
                    已检索内容：{context}
                    当前跳数：{hop}
                    
                    请生成下一个需要检索的子问题。
                    """)
                .param("question", originalQuestion)
                .param("context", context)
                .param("hop", hop)
            )
            .call()
            .content();
    }
    
    private String generateAnswer(String question, String context) {
        return chatClient
            .prompt(spec -> spec
                .text("""
                    问题：{question}
                    上下文：{context}
                    
                    请综合所有信息回答问题。
                    """)
                .param("question", question)
                .param("context", context)
            )
            .call()
            .content();
    }
}
```

### 2. 自我查询RAG（Self-Querying RAG）

AI自动提取查询中的过滤条件。

```java
@Service
public class SelfQueryingRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 自我查询RAG
     */
    public String selfQueryingRag(String userQuery) {
        // 1. 让AI提取查询意图和过滤条件
        QueryAnalysis analysis = analyzeQuery(userQuery);
        
        // 2. 使用提取的条件检索
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(analysis.semanticQuery())
                .topK(5)
                .filterExpression(analysis.filter())
                .build()
        );
        
        // 3. 生成答案
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return generateAnswer(userQuery, context);
    }
    
    private QueryAnalysis analyzeQuery(String userQuery) {
        String analysisPrompt = """
            分析以下用户查询，提取：
            1. 语义查询（semantic_query）：用于向量搜索的核心查询
            2. 过滤条件（filter）：元数据过滤表达式
            
            用户查询：%s
            
            请以JSON格式返回：
            {
              "semantic_query": "...",
              "filter": "..."
            }
            """.formatted(userQuery);
        
        String response = chatClient
            .prompt(analysisPrompt)
            .call()
            .content();
        
        // 解析JSON（简化）
        return parseQueryAnalysis(response);
    }
    
    private QueryAnalysis parseQueryAnalysis(String json) {
        // JSON解析逻辑
        return new QueryAnalysis("", null); // 简化
    }
    
    private String generateAnswer(String question, String context) {
        return chatClient
            .prompt(spec -> spec
                .text("问题：{question}\n上下文：{context}")
                .param("question", question)
                .param("context", context)
            )
            .call()
            .content();
    }
    
    record QueryAnalysis(
        String semanticQuery,
        String filter
    ) {}
}
```

### 3. HyDE (Hypothetical Document Embeddings)

生成假设性文档以改善检索。

```java
@Service
public class HyDERAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * HyDE RAG
     */
    public String hydeRag(String question) {
        // 1. 生成假设性答案
        String hypotheticalAnswer = generateHypotheticalAnswer(question);
        
        // 2. 使用假设性答案检索
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(hypotheticalAnswer)  // 注意：用答案而不是问题
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );
        
        // 3. 基于真实文档生成最终答案
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return generateFinalAnswer(question, context);
    }
    
    private String generateHypotheticalAnswer(String question) {
        return chatClient
            .prompt(spec -> spec
                .text("""
                    请为以下问题生成一个假设性的、详细的答案：
                    
                    {question}
                    
                    注意：这是假设性的，用于检索相关文档。
                    """)
                .param("question", question)
            )
            .call()
            .content();
    }
    
    private String generateFinalAnswer(String question, String context) {
        return chatClient
            .prompt(spec -> spec
                .text("""
                    问题：{question}
                    参考文档：{context}
                    
                    请基于参考文档生成准确的答案。
                    """)
                .param("question", question)
                .param("context", context)
            )
            .call()
            .content();
    }
}
```

---

## 性能优化

### 1. 缓存策略

```java
@Service
@EnableCaching
public class CachedRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 缓存向量搜索结果
     */
    @Cacheable(
        value = "vectorSearchCache",
        key = "#question + '_' + #topK"
    )
    public List<Document> cachedVectorSearch(
            String question,
            int topK) {
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(topK)
                .build()
        );
    }
    
    /**
     * 缓存RAG答案
     */
    @Cacheable(
        value = "ragAnswerCache",
        key = "#question",
        unless = "#result == null || #result.isEmpty()"
    )
    public String cachedRagAnswer(String question) {
        List<Document> docs = cachedVectorSearch(question, 5);
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return chatClient
            .prompt(buildPrompt(question, context))
            .call()
            .content();
    }
    
    /**
     * 清除缓存
     */
    @CacheEvict(value = {"vectorSearchCache", "ragAnswerCache"}, allEntries = true)
    public void clearCache() {
        logger.info("Cleared RAG caches");
    }
    
    private String buildPrompt(String question, String context) {
        return "问题：" + question + "\n上下文：" + context;
    }
}
```

### 2. 异步处理

```java
@Service
public class AsyncRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 异步RAG查询
     */
    @Async
    public CompletableFuture<String> asyncRag(String question) {
        return CompletableFuture.supplyAsync(() -> {
            // 检索
            List<Document> docs = vectorStore.similaritySearch(question);
            
            // 生成答案
            String context = docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n\n"));
            
            return chatClient
                .prompt(buildPrompt(question, context))
                .call()
                .content();
        });
    }
    
    /**
     * 并行处理多个问题
     */
    public List<String> parallelRag(List<String> questions) {
        List<CompletableFuture<String>> futures = questions.stream()
            .map(this::asyncRag)
            .toList();
        
        return futures.stream()
            .map(CompletableFuture::join)
            .toList();
    }
    
    private String buildPrompt(String question, String context) {
        return "问题：" + question + "\n上下文：" + context;
    }
}
```

### 3. 批量优化

```java
@Service
public class BatchRAGService {
    
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    private final ChatClient chatClient;
    
    /**
     * 批量嵌入查询
     */
    public Map<String, String> batchRag(List<String> questions) {
        // 1. 批量生成embedding（一次API调用）
        List<float[]> queryEmbeddings = 
            embeddingModel.embed(questions);
        
        // 2. 批量检索
        Map<String, List<Document>> retrievalResults = new HashMap<>();
        for (int i = 0; i < questions.size(); i++) {
            // 使用预计算的embedding检索
            // （需要VectorStore支持直接向量查询）
            List<Document> docs = vectorStore.similaritySearch(
                questions.get(i)
            );
            retrievalResults.put(questions.get(i), docs);
        }
        
        // 3. 批量生成答案
        Map<String, String> answers = new HashMap<>();
        for (String question : questions) {
            List<Document> docs = retrievalResults.get(question);
            String context = docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n\n"));
            
            String answer = chatClient
                .prompt(buildPrompt(question, context))
                .call()
                .content();
            
            answers.put(question, answer);
        }
        
        return answers;
    }
    
    private String buildPrompt(String question, String context) {
        return "问题：" + question + "\n上下文：" + context;
    }
}
```

---

## 实战案例

### 1. 企业知识库问答系统

```java
@Service
public class EnterpriseKnowledgeBase {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    private final DocumentIngestionPipeline ingestionPipeline;
    
    /**
     * 导入企业文档
     */
    public void ingestCompanyDocuments(
            String department,
            List<Resource> documents) {
        
        Map<String, Object> metadata = Map.of(
            "department", department,
            "type", "internal",
            "ingestedAt", Instant.now()
        );
        
        ingestionPipeline.batchIngest(documents, metadata);
    }
    
    /**
     * 按部门查询
     */
    public Answer askByDepartment(
            String question,
            String department) {
        
        // 限制在特定部门的文档
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .filterExpression("department == '" + department + "'")
                .build()
        );
        
        if (docs.isEmpty()) {
            return new Answer(
                "抱歉，在" + department + "部门的文档中没有找到相关信息。",
                List.of()
            );
        }
        
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        String answer = chatClient
            .prompt(spec -> spec
                .system("""
                    你是{department}部门的智能助手。
                    请基于部门文档回答问题。
                    """)
                .user("""
                    参考资料：
                    {context}
                    
                    问题：{question}
                    """)
                .param("department", department)
                .param("context", context)
                .param("question", question)
            )
            .call()
            .content();
        
        List<String> sources = docs.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .distinct()
            .toList();
        
        return new Answer(answer, sources);
    }
    
    /**
     * 多部门联合查询
     */
    public Answer crossDepartmentQuery(
            String question,
            List<String> departments) {
        
        String filter = departments.stream()
            .map(dept -> "department == '" + dept + "'")
            .collect(Collectors.joining(" || "));
        
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(10)
                .filterExpression(filter)
                .build()
        );
        
        // 按部门分组
        Map<String, List<Document>> byDept = docs.stream()
            .collect(Collectors.groupingBy(doc -> 
                (String) doc.getMetadata().get("department")
            ));
        
        // 构建分部门的上下文
        StringBuilder contextBuilder = new StringBuilder();
        for (Map.Entry<String, List<Document>> entry : byDept.entrySet()) {
            contextBuilder.append("【").append(entry.getKey()).append("部门】\n");
            entry.getValue().forEach(doc -> 
                contextBuilder.append("- ").append(doc.getText()).append("\n")
            );
            contextBuilder.append("\n");
        }
        
        String answer = chatClient
            .prompt(spec -> spec
                .text("""
                    以下是来自不同部门的相关资料：
                    
                    {context}
                    
                    问题：{question}
                    
                    请综合各部门的信息回答问题，并注明信息来源。
                    """)
                .param("context", contextBuilder.toString())
                .param("question", question)
            )
            .call()
            .content();
        
        return new Answer(answer, List.copyOf(byDept.keySet()));
    }
    
    record Answer(String text, List<String> sources) {}
}
```

### 2. 客户服务机器人

```java
@Service
public class CustomerServiceBot {
    
    private final ChatClient chatClient;
    private final VectorStore faqVectorStore;
    private final VectorStore productVectorStore;
    
    /**
     * 智能客服回答
     */
    public ServiceResponse answer(
            String customerId,
            String question) {
        
        // 1. 分类问题类型
        QuestionType type = classifyQuestion(question);
        
        // 2. 根据类型选择知识库
        List<Document> relevantDocs = switch (type) {
            case FAQ -> searchFAQ(question);
            case PRODUCT -> searchProduct(question);
            case TECHNICAL -> searchTechnical(question);
            case GENERAL -> searchGeneral(question);
        };
        
        // 3. 检查是否需要人工介入
        if (requiresHumanAgent(question, relevantDocs)) {
            return new ServiceResponse(
                "您的问题需要专业人员处理，正在为您转接人工客服...",
                ResponseType.ESCALATE,
                List.of()
            );
        }
        
        // 4. 生成回答
        String context = relevantDocs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        String answer = chatClient
            .prompt(spec -> spec
                .system("""
                    你是一个专业的客服助手。
                    - 态度友好、专业
                    - 回答准确、简洁
                    - 基于知识库内容回答
                    - 如果不确定，建议联系人工客服
                    """)
                .user("""
                    客户ID：{customerId}
                    知识库内容：{context}
                    
                    客户问题：{question}
                    """)
                .param("customerId", customerId)
                .param("context", context)
                .param("question", question)
            )
            .call()
            .content();
        
        // 5. 提取相关产品或FAQ链接
        List<String> relatedLinks = relevantDocs.stream()
            .map(doc -> (String) doc.getMetadata().get("link"))
            .filter(Objects::nonNull)
            .toList();
        
        return new ServiceResponse(
            answer,
            ResponseType.AUTO,
            relatedLinks
        );
    }
    
    private QuestionType classifyQuestion(String question) {
        // 使用LLM分类问题
        String classification = chatClient
            .prompt("""
                将以下问题分类为：FAQ、PRODUCT、TECHNICAL、GENERAL
                
                问题：%s
                
                只返回类别名称。
                """.formatted(question))
            .call()
            .content();
        
        return QuestionType.valueOf(classification.trim().toUpperCase());
    }
    
    private List<Document> searchFAQ(String question) {
        return faqVectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(3)
                .similarityThreshold(0.8)
                .build()
        );
    }
    
    private List<Document> searchProduct(String question) {
        return productVectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .build()
        );
    }
    
    private List<Document> searchTechnical(String question) {
        // 搜索技术文档
        return List.of(); // 简化
    }
    
    private List<Document> searchGeneral(String question) {
        // 通用搜索
        return List.of(); // 简化
    }
    
    private boolean requiresHumanAgent(
            String question,
            List<Document> docs) {
        
        // 检查是否需要人工
        return docs.isEmpty() || 
               question.contains("投诉") ||
               question.contains("退款");
    }
    
    enum QuestionType {
        FAQ, PRODUCT, TECHNICAL, GENERAL
    }
    
    enum ResponseType {
        AUTO, ESCALATE
    }
    
    record ServiceResponse(
        String answer,
        ResponseType type,
        List<String> relatedLinks
    ) {}
}
```

### 3. 代码文档助手

```java
@Service
public class CodeDocumentationAssistant {
    
    private final VectorStore codeVectorStore;
    private final ChatClient chatClient;
    
    /**
     * 代码解释
     */
    public String explainCode(String code) {
        // 搜索相似代码和文档
        List<Document> docs = codeVectorStore.similaritySearch(
            SearchRequest.builder()
                .query(code)
                .topK(3)
                .filterExpression("type == 'code_example'")
                .build()
        );
        
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return chatClient
            .prompt(spec -> spec
                .system("""
                    你是一个代码解释专家。
                    - 解释代码的功能和逻辑
                    - 指出潜在问题
                    - 提供最佳实践建议
                    """)
                .user("""
                    参考代码示例：
                    {context}
                    
                    需要解释的代码：
                    ```
                    {code}
                    ```
                    """)
                .param("context", context)
                .param("code", code)
            )
            .call()
            .content();
    }
    
    /**
     * API使用示例
     */
    public String showApiExample(String apiName) {
        List<Document> docs = codeVectorStore.similaritySearch(
            SearchRequest.builder()
                .query(apiName)
                .topK(5)
                .filterExpression(
                    "type == 'api_doc' || type == 'code_example'"
                )
                .build()
        );
        
        // 分离文档和示例
        Map<String, List<Document>> grouped = docs.stream()
            .collect(Collectors.groupingBy(doc -> 
                (String) doc.getMetadata().get("type")
            ));
        
        String apiDoc = grouped.getOrDefault("api_doc", List.of())
            .stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n"));
        
        String examples = grouped.getOrDefault("code_example", List.of())
            .stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        return chatClient
            .prompt(spec -> spec
                .system("你是一个API文档助手")
                .user("""
                    API文档：
                    {apiDoc}
                    
                    代码示例：
                    {examples}
                    
                    请为{apiName}生成一个完整的使用示例，包括：
                    1. 基本用法
                    2. 常见场景
                    3. 注意事项
                    """)
                .param("apiDoc", apiDoc)
                .param("examples", examples)
                .param("apiName", apiName)
            )
            .call()
            .content();
    }
}
```

---

## 最佳实践

### 1. Prompt设计

```java
/**
 * 高质量RAG Prompt模板
 */
public class RAGPromptTemplates {
    
    /**
     * 基础模板
     */
    public static final String BASIC_TEMPLATE = """
        请基于以下上下文回答问题：
        
        上下文：
        {context}
        
        问题：{question}
        
        要求：
        1. 仅基于上下文信息回答
        2. 如果上下文中没有相关信息，请明确说明
        3. 保持回答简洁、准确
        """;
    
    /**
     * 带来源引用的模板
     */
    public static final String WITH_CITATION_TEMPLATE = """
        请基于以下文档回答问题，并引用来源：
        
        {documents}
        
        问题：{question}
        
        回答格式：
        【答案】
        （你的回答）
        
        【来源】
        - 文档1：（相关内容）
        - 文档2：（相关内容）
        """;
    
    /**
     * 结构化输出模板
     */
    public static final String STRUCTURED_TEMPLATE = """
        上下文：{context}
        问题：{question}
        
        请按以下JSON格式回答：
        {
          "answer": "简洁的答案",
          "explanation": "详细解释",
          "confidence": "high/medium/low",
          "sources": ["来源1", "来源2"]
        }
        """;
}
```

### 2. 评估和监控

```java
@Service
public class RAGEvaluationService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 评估RAG答案质量
     */
    public RagQuality evaluateAnswer(
            String question,
            String answer,
            List<Document> retrievedDocs) {
        
        // 1. 相关性评分
        double relevanceScore = evaluateRelevance(
            question,
            retrievedDocs
        );
        
        // 2. 答案准确性评分
        double accuracyScore = evaluateAccuracy(
            answer,
            retrievedDocs
        );
        
        // 3. 完整性评分
        double completenessScore = evaluateCompleteness(
            question,
            answer
        );
        
        return new RagQuality(
            relevanceScore,
            accuracyScore,
            completenessScore,
            (relevanceScore + accuracyScore + completenessScore) / 3
        );
    }
    
    private double evaluateRelevance(
            String question,
            List<Document> docs) {
        
        if (docs.isEmpty()) {
            return 0.0;
        }
        
        // 使用LLM评估相关性
        String prompt = """
            问题：{question}
            
            检索到的文档：
            {docs}
            
            请评估这些文档与问题的相关性（0-1之间的分数）。
            只返回数字。
            """.replace("{question}", question)
               .replace("{docs}", docs.stream()
                   .map(Document::getText)
                   .collect(Collectors.joining("\n\n")));
        
        String scoreStr = chatClient
            .prompt(prompt)
            .call()
            .content()
            .trim();
        
        try {
            return Double.parseDouble(scoreStr);
        } catch (NumberFormatException e) {
            return 0.5; // 默认值
        }
    }
    
    private double evaluateAccuracy(
            String answer,
            List<Document> docs) {
        
        // 检查答案是否与文档一致
        String context = docs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n"));
        
        String prompt = """
            上下文：{context}
            答案：{answer}
            
            请评估答案与上下文的一致性（0-1之间的分数）。
            只返回数字。
            """.replace("{context}", context)
               .replace("{answer}", answer);
        
        String scoreStr = chatClient
            .prompt(prompt)
            .call()
            .content()
            .trim();
        
        try {
            return Double.parseDouble(scoreStr);
        } catch (NumberFormatException e) {
            return 0.5;
        }
    }
    
    private double evaluateCompleteness(
            String question,
            String answer) {
        
        // 评估答案的完整性
        String prompt = """
            问题：{question}
            答案：{answer}
            
            请评估答案的完整性（0-1之间的分数）。
            只返回数字。
            """.replace("{question}", question)
               .replace("{answer}", answer);
        
        String scoreStr = chatClient
            .prompt(prompt)
            .call()
            .content()
            .trim();
        
        try {
            return Double.parseDouble(scoreStr);
        } catch (NumberFormatException e) {
            return 0.5;
        }
    }
    
    record RagQuality(
        double relevance,
        double accuracy,
        double completeness,
        double overall
    ) {}
}
```

### 3. 错误处理

```java
@Service
public class RobustRAGService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 健壮的RAG实现
     */
    public RagResult robustRag(String question) {
        try {
            // 1. 检索文档
            List<Document> docs = safeRetrieve(question);
            
            if (docs.isEmpty()) {
                return new RagResult(
                    "抱歉，我在知识库中没有找到相关信息。",
                    RagStatus.NO_DOCUMENTS,
                    List.of()
                );
            }
            
            // 2. 生成答案
            String context = docs.stream()
                .map(Document::getText)
                .collect(Collectors.joining("\n\n"));
            
            String answer = safeGenerate(question, context);
            
            return new RagResult(
                answer,
                RagStatus.SUCCESS,
                docs
            );
            
        } catch (Exception e) {
            logger.error("RAG failed", e);
            return new RagResult(
                "抱歉，处理您的问题时出现错误，请稍后重试。",
                RagStatus.ERROR,
                List.of()
            );
        }
    }
    
    private List<Document> safeRetrieve(String question) {
        try {
            return vectorStore.similaritySearch(
                SearchRequest.builder()
                    .query(question)
                    .topK(5)
                    .similarityThreshold(0.7)
                    .build()
            );
        } catch (Exception e) {
            logger.error("Retrieval failed", e);
            return List.of();
        }
    }
    
    private String safeGenerate(String question, String context) {
        try {
            return chatClient
                .prompt(buildPrompt(question, context))
                .call()
                .content();
        } catch (Exception e) {
            logger.error("Generation failed", e);
            return "抱歉，生成答案时出现错误。";
        }
    }
    
    private String buildPrompt(String question, String context) {
        return "问题：" + question + "\n上下文：" + context;
    }
    
    enum RagStatus {
        SUCCESS, NO_DOCUMENTS, ERROR
    }
    
    record RagResult(
        String answer,
        RagStatus status,
        List<Document> documents
    ) {}
}
```

---

## 总结

### RAG核心要点

1. **检索（Retrieval）**: 从向量库中找到相关文档
2. **增强（Augmentation）**: 将文档作为上下文增强Prompt
3. **生成（Generation）**: 使用LLM生成基于上下文的答案

### Spring AI的RAG实现

| 方式 | 优势 | 劣势 | 适用场景 |
|------|------|------|---------|
| **QuestionAnswerAdvisor** | 开箱即用、简单 | 灵活性有限 | 标准RAG应用 |
| **自定义实现** | 完全控制、灵活 | 需要更多代码 | 复杂RAG模式 |

### 最佳实践清单

- ✅ 合理分块文档（500-1000 tokens）
- ✅ 添加丰富的元数据
- ✅ 选择合适的Top K值（3-5）
- ✅ 设置合理的相似度阈值（0.7-0.8）
- ✅ 设计高质量的Prompt模板
- ✅ 提供信息来源
- ✅ 实现错误处理
- ✅ 添加缓存优化
- ✅ 监控和评估质量

### RAG高级模式

- **Multi-Hop RAG**: 多次检索的复杂问题
- **Self-Querying RAG**: AI自动提取过滤条件
- **HyDE**: 使用假设性文档改善检索
- **Hybrid Search**: 结合向量和关键词搜索
- **Re-ranking**: 使用更精确的模型重排序

通过掌握RAG，你可以让大语言模型回答基于你自己数据的问题，构建强大的知识库问答、客服机器人等应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

