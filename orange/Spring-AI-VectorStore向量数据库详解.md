# Spring AI VectorStore 向量数据库详解

## 📋 目录
- [概述](#概述)
- [核心接口](#核心接口)
- [VectorStore接口详解](#vectorstore接口详解)
- [SearchRequest搜索请求](#searchrequest搜索请求)
- [Filter过滤表达式](#filter过滤表达式)
- [SimpleVectorStore实现](#simplevectorstore实现)
- [支持的向量数据库](#支持的向量数据库)
- [使用指南](#使用指南)
- [高级特性](#高级特性)
- [实战场景](#实战场景)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是Vector Store（向量数据库）？

**VectorStore** 是专门用于存储和检索**向量（embeddings）**的数据库，支持基于**向量相似度**的快速搜索。

```
文档 → Embedding → 向量数据库存储
                       ↓
查询 → Embedding → 相似度搜索 → 返回相关文档
```

### 为什么需要Vector Store？

1. ✅ **语义搜索**: 基于含义而非关键词查找
2. ✅ **高效检索**: 支持大规模向量的快速相似度搜索
3. ✅ **RAG应用**: 检索增强生成的基础设施
4. ✅ **智能推荐**: 基于向量相似度的推荐系统
5. ✅ **去重检测**: 识别语义相似的重复内容
6. ✅ **知识管理**: 企业知识库的核心存储

### Vector Store的特点

| 特点 | 说明 |
|------|------|
| **向量索引** | 使用特殊索引（HNSW、IVF等）加速搜索 |
| **相似度算法** | 余弦相似度、欧氏距离、点积等 |
| **元数据过滤** | 支持基于元数据的过滤查询 |
| **可扩展性** | 支持数百万到数十亿向量 |
| **持久化** | 向量和元数据持久化存储 |

### 应用场景

- 📚 **知识库问答**: 根据问题检索相关文档
- 🔍 **语义搜索**: 理解用户意图的搜索
- 💬 **对话上下文**: 检索历史对话
- 📄 **文档管理**: 企业文档的智能检索
- 🎯 **推荐系统**: 基于内容的推荐
- 🔎 **重复检测**: 识别相似内容

---

## 核心接口

### 接口层次结构

```
DocumentWriter (写入接口)
    ↑
    │
VectorStoreRetriever (只读检索接口)
    ↑
    │
VectorStore (完整接口)
    ↑
    │
├── SimpleVectorStore (简单实现)
├── PgVectorStore (PostgreSQL)
├── ChromaVectorStore (Chroma)
├── PineconeVectorStore (Pinecone)
└── ... (20+种实现)
```

### VectorStoreRetriever（只读接口）

```java
/**
 * 向量存储只读检索接口
 * 遵循最小权限原则，仅暴露检索功能
 */
@FunctionalInterface
public interface VectorStoreRetriever {
    
    /**
     * 相似度搜索（完整版）
     * @param request 搜索请求
     * @return 相似文档列表
     */
    List<Document> similaritySearch(SearchRequest request);
    
    /**
     * 相似度搜索（简化版）
     * @param query 查询文本
     * @return 相似文档列表
     */
    default List<Document> similaritySearch(String query) {
        return this.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .build()
        );
    }
}
```

---

## VectorStore接口详解

### 完整接口定义

```java
/**
 * 向量存储完整接口
 * 扩展DocumentWriter和VectorStoreRetriever
 */
public interface VectorStore 
    extends DocumentWriter, VectorStoreRetriever {
    
    /**
     * 获取向量存储名称
     */
    default String getName() {
        return this.getClass().getSimpleName();
    }
    
    /**
     * 添加文档
     * @param documents 文档列表
     * @throws IllegalStateException 如果有重复ID
     */
    void add(List<Document> documents);
    
    /**
     * 实现DocumentWriter接口
     */
    @Override
    default void accept(List<Document> documents) {
        add(documents);
    }
    
    /**
     * 按ID删除文档
     * @param idList 文档ID列表
     */
    void delete(List<String> idList);
    
    /**
     * 按过滤条件删除文档
     * @param filterExpression 过滤表达式
     */
    void delete(Filter.Expression filterExpression);
    
    /**
     * 按过滤条件删除文档（字符串版本）
     * @param filterExpression 过滤表达式字符串
     */
    default void delete(String filterExpression) {
        SearchRequest searchRequest = SearchRequest.builder()
            .filterExpression(filterExpression)
            .build();
        
        Filter.Expression textExpression = 
            searchRequest.getFilterExpression();
        
        Assert.notNull(textExpression, 
            "Filter expression must not be null");
        
        this.delete(textExpression);
    }
    
    /**
     * 获取原生客户端（可选）
     * @return 原生客户端的Optional
     */
    default <T> Optional<T> getNativeClient() {
        return Optional.empty();
    }
}
```

### 核心方法说明

| 方法 | 功能 | 适用场景 |
|------|------|----------|
| `add(List<Document>)` | 添加文档到向量存储 | 批量导入文档 |
| `similaritySearch(SearchRequest)` | 相似度搜索 | 查找相关文档 |
| `similaritySearch(String)` | 简化搜索 | 快速查询 |
| `delete(List<String>)` | 按ID删除 | 精确删除 |
| `delete(Filter.Expression)` | 按条件删除 | 批量删除 |
| `accept(List<Document>)` | DocumentWriter实现 | 流式处理 |

---

## SearchRequest搜索请求

### 完整定义

```java
/**
 * 相似度搜索请求
 */
public class SearchRequest {
    
    /**
     * 查询文本
     */
    private String query = "";
    
    /**
     * 返回Top K个结果
     * 默认：4
     */
    private int topK = DEFAULT_TOP_K;
    
    /**
     * 相似度阈值（0.0 - 1.0）
     * 0.0 = 接受所有
     * 1.0 = 要求完全匹配
     */
    private double similarityThreshold = SIMILARITY_THRESHOLD_ACCEPT_ALL;
    
    /**
     * 元数据过滤表达式
     */
    @Nullable
    private Filter.Expression filterExpression;
    
    // Getters
    public String getQuery() { return query; }
    public int getTopK() { return topK; }
    public double getSimilarityThreshold() { return similarityThreshold; }
    public Filter.Expression getFilterExpression() { return filterExpression; }
    
    /**
     * 是否有过滤表达式
     */
    public boolean hasFilterExpression() {
        return filterExpression != null;
    }
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    /**
     * 从现有请求复制
     */
    public static Builder from(SearchRequest original) {
        return builder()
            .query(original.getQuery())
            .topK(original.getTopK())
            .similarityThreshold(original.getSimilarityThreshold())
            .filterExpression(original.getFilterExpression());
    }
}
```

### Builder使用

```java
/**
 * SearchRequest Builder
 */
public static class Builder {
    
    /**
     * 设置查询文本
     */
    public Builder query(String query) {
        Assert.notNull(query, "Query can not be null.");
        this.searchRequest.query = query;
        return this;
    }
    
    /**
     * 设置Top K
     */
    public Builder topK(int topK) {
        Assert.isTrue(topK >= 0, "TopK should be positive.");
        this.searchRequest.topK = topK;
        return this;
    }
    
    /**
     * 设置相似度阈值
     * @param threshold 0.0-1.0之间
     */
    public Builder similarityThreshold(double threshold) {
        Assert.isTrue(threshold >= 0 && threshold <= 1,
            "Similarity threshold must be in [0,1] range.");
        this.searchRequest.similarityThreshold = threshold;
        return this;
    }
    
    /**
     * 禁用相似度阈值（接受所有）
     */
    public Builder similarityThresholdAll() {
        this.searchRequest.similarityThreshold = 0.0;
        return this;
    }
    
    /**
     * 设置过滤表达式
     */
    public Builder filterExpression(Filter.Expression expression) {
        this.searchRequest.filterExpression = expression;
        return this;
    }
    
    /**
     * 设置过滤表达式（字符串版本）
     */
    public Builder filterExpression(String textExpression) {
        this.searchRequest.filterExpression = (textExpression != null) ?
            new FilterExpressionTextParser().parse(textExpression) :
            null;
        return this;
    }
    
    public SearchRequest build() {
        return this.searchRequest;
    }
}
```

### 使用示例

```java
// 1. 最简单的搜索
SearchRequest request1 = SearchRequest.builder()
    .query("Spring AI是什么？")
    .build();

// 2. 带Top K的搜索
SearchRequest request2 = SearchRequest.builder()
    .query("如何使用ChatClient？")
    .topK(10)
    .build();

// 3. 带相似度阈值的搜索
SearchRequest request3 = SearchRequest.builder()
    .query("RAG应用示例")
    .topK(5)
    .similarityThreshold(0.75)  // 只返回相似度>0.75的结果
    .build();

// 4. 带元数据过滤的搜索
SearchRequest request4 = SearchRequest.builder()
    .query("技术文档")
    .topK(10)
    .filterExpression("category == 'tutorial' && year >= 2024")
    .build();

// 5. 复杂过滤
SearchRequest request5 = SearchRequest.builder()
    .query("Spring Boot教程")
    .topK(20)
    .similarityThreshold(0.6)
    .filterExpression(
        "type == 'article' && " +
        "(tag IN ['java', 'spring'] || author == 'admin')"
    )
    .build();
```

---

## Filter过滤表达式

### Filter表达式概述

Spring AI提供了**跨向量数据库通用**的过滤表达式语法，类似SQL WHERE子句。

### 1. 表达式类型

```java
/**
 * 过滤表达式类型
 */
public enum ExpressionType {
    
    // 比较操作
    EQ,   // 等于 (==)
    NE,   // 不等于 (!=)
    GT,   // 大于 (>)
    GTE,  // 大于等于 (>=)
    LT,   // 小于 (<)
    LTE,  // 小于等于 (<=)
    
    // 集合操作
    IN,   // 在集合中
    NIN,  // 不在集合中
    
    // 逻辑操作
    AND,  // 与
    OR,   // 或
    NOT   // 非
}
```

### 2. 表达式组件

```java
/**
 * 过滤表达式的组成部分
 */

// 键（字段名）
public record Key(String key) implements Operand {}

// 值（常量）
public record Value(Object value) implements Operand {}

// 表达式
public record Expression(
    ExpressionType type,
    Operand left,
    Operand right
) implements Operand {}

// 分组（括号）
public record Group(Expression content) implements Operand {}
```

### 3. 三种创建方式

#### 方式1：手动构建

```java
// country == "BG"
var exp1 = new Filter.Expression(
    ExpressionType.EQ,
    new Filter.Key("country"),
    new Filter.Value("BG")
);

// genre == "drama" AND year >= 2020
var exp2 = new Filter.Expression(
    ExpressionType.AND,
    new Filter.Expression(
        ExpressionType.EQ,
        new Filter.Key("genre"),
        new Filter.Value("drama")
    ),
    new Filter.Expression(
        ExpressionType.GTE,
        new Filter.Key("year"),
        new Filter.Value(2020)
    )
);

// genre IN ["comedy", "documentary", "drama"]
var exp3 = new Filter.Expression(
    ExpressionType.IN,
    new Filter.Key("genre"),
    new Filter.Value(List.of("comedy", "documentary", "drama"))
);
```

#### 方式2：使用Builder DSL

```java
var b = new FilterExpressionBuilder();

// 1. country == "BG"
var exp1 = b.eq("country", "BG");

// 2. genre == "drama" AND year >= 2020
var exp2 = b.and(
    b.eq("genre", "drama"),
    b.gte("year", 2020)
);

// 3. genre IN ["comedy", "documentary", "drama"]
var exp3 = b.in("genre", "comedy", "documentary", "drama");

// 4. year >= 2020 OR country == "BG" AND city != "Sofia"
var exp4 = b.and(
    b.or(b.gte("year", 2020), b.eq("country", "BG")),
    b.ne("city", "Sofia")
);

// 5. (year >= 2020 OR country == "BG") AND city NIN ["Sofia", "Plovdiv"]
var exp5 = b.and(
    b.group(b.or(b.gte("year", 2020), b.eq("country", "BG"))),
    b.nin("city", "Sofia", "Plovdiv")
);

// 6. isOpen == true AND year >= 2020 AND country IN ["BG", "NL", "US"]
var exp6 = b.and(
    b.and(b.eq("isOpen", true), b.gte("year", 2020)),
    b.in("country", "BG", "NL", "US")
);
```

#### 方式3：使用文本解析器

```java
var parser = new FilterExpressionTextParser();

// 简单比较
var exp1 = parser.parse("country == 'BG'");

// 逻辑组合
var exp2 = parser.parse("genre == 'drama' && year >= 2020");

// IN操作
var exp3 = parser.parse("genre IN ['comedy', 'documentary', 'drama']");

// 复杂表达式
var exp4 = parser.parse(
    "year >= 2020 || (country == 'BG' && city != 'Sofia')"
);

// 带括号分组
var exp5 = parser.parse(
    "(year >= 2020 || country == 'BG') && city NOT IN ['Sofia', 'Plovdiv']"
);

// 多条件组合
var exp6 = parser.parse(
    "isOpen == true && year >= 2020 && country IN ['BG', 'NL', 'US']"
);
```

### 4. 支持的操作符

| 操作类型 | 操作符 | 示例 |
|---------|-------|------|
| **相等** | `==` | `country == 'UK'` |
| **不等** | `!=` | `status != 'deleted'` |
| **大于** | `>` | `price > 100` |
| **大于等于** | `>=` | `year >= 2020` |
| **小于** | `<` | `age < 18` |
| **小于等于** | `<=` | `score <= 0.5` |
| **在集合中** | `IN` | `tag IN ['java', 'spring']` |
| **不在集合中** | `NOT IN` | `city NOT IN ['Sofia']` |
| **逻辑与** | `&&` 或 `AND` | `a == 1 && b == 2` |
| **逻辑或** | `\|\|` 或 `OR` | `a == 1 \|\| b == 2` |
| **非** | `NOT` | `NOT (a == 1)` |

### 5. 数据类型支持

```java
// 字符串
"country == 'China'"

// 数字
"year >= 2020"
"price < 99.99"

// 布尔值
"isActive == true"
"isDeleted != false"

// 数组
"tags IN ['java', 'spring', 'ai']"
"colors NOT IN ['red', 'blue']"
```

---

## SimpleVectorStore实现

### 概述

`SimpleVectorStore` 是Spring AI提供的**内存向量存储**实现，适合开发和测试。

### 核心实现

```java
/**
 * 简单内存向量存储实现
 */
public class SimpleVectorStore extends AbstractObservationVectorStore {
    
    /**
     * 内存存储
     * Key: 文档ID
     * Value: 文档内容+向量
     */
    protected Map<String, SimpleVectorStoreContent> store = 
        new ConcurrentHashMap<>();
    
    /**
     * 嵌入模型
     */
    private final EmbeddingModel embeddingModel;
    
    /**
     * 创建Builder
     */
    public static SimpleVectorStoreBuilder builder(
            EmbeddingModel embeddingModel) {
        return new SimpleVectorStoreBuilder(embeddingModel);
    }
    
    /**
     * 添加文档
     */
    @Override
    public void doAdd(List<Document> documents) {
        Objects.requireNonNull(documents, "Documents list cannot be null");
        
        if (documents.isEmpty()) {
            throw new IllegalArgumentException(
                "Documents list cannot be empty"
            );
        }
        
        for (Document document : documents) {
            logger.info("Calling EmbeddingModel for document id = {}", 
                document.getId());
            
            // 嵌入文档
            float[] embedding = embeddingModel.embed(document);
            
            // 存储
            SimpleVectorStoreContent storeContent = 
                new SimpleVectorStoreContent(
                    document.getId(),
                    document.getText(),
                    document.getMetadata(),
                    embedding
                );
            
            store.put(document.getId(), storeContent);
        }
    }
    
    /**
     * 删除文档
     */
    @Override
    public void doDelete(List<String> idList) {
        for (String id : idList) {
            store.remove(id);
        }
    }
    
    /**
     * 相似度搜索
     */
    @Override
    public List<Document> doSimilaritySearch(SearchRequest request) {
        
        // 1. 嵌入查询
        float[] queryEmbedding = embeddingModel.embed(request.getQuery());
        
        // 2. 计算所有文档的相似度
        List<ScoredDocument> scoredDocs = store.values()
            .stream()
            .map(content -> {
                float similarity = cosineSimilarity(
                    queryEmbedding,
                    content.embedding()
                );
                
                Document doc = new Document(
                    content.id(),
                    content.text(),
                    content.metadata()
                );
                doc.setScore(similarity);
                
                return new ScoredDocument(doc, similarity);
            })
            .toList();
        
        // 3. 应用元数据过滤
        if (request.hasFilterExpression()) {
            scoredDocs = filterByMetadata(
                scoredDocs,
                request.getFilterExpression()
            );
        }
        
        // 4. 应用相似度阈值
        scoredDocs = scoredDocs.stream()
            .filter(sd -> sd.score() >= request.getSimilarityThreshold())
            .toList();
        
        // 5. 排序并返回Top K
        return scoredDocs.stream()
            .sorted((a, b) -> Float.compare(b.score(), a.score()))
            .limit(request.getTopK())
            .map(ScoredDocument::document)
            .toList();
    }
    
    /**
     * 余弦相似度计算
     */
    private float cosineSimilarity(float[] vec1, float[] vec2) {
        float dot = 0.0f;
        float norm1 = 0.0f;
        float norm2 = 0.0f;
        
        for (int i = 0; i < vec1.length; i++) {
            dot += vec1[i] * vec2[i];
            norm1 += vec1[i] * vec1[i];
            norm2 += vec2[i] * vec2[i];
        }
        
        return dot / (float) (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
    
    /**
     * 元数据过滤
     */
    private List<ScoredDocument> filterByMetadata(
            List<ScoredDocument> docs,
            Filter.Expression filterExpression) {
        
        // 转换为SpEL表达式并过滤
        // ...实现细节
        
        return filteredDocs;
    }
    
    /**
     * 保存到文件
     */
    public void save(File file) throws IOException {
        objectMapper.writeValue(file, store);
    }
    
    /**
     * 从文件加载
     */
    public void load(File file) throws IOException {
        Map<String, SimpleVectorStoreContent> loadedStore = 
            objectMapper.readValue(file, 
                new TypeReference<Map<String, SimpleVectorStoreContent>>() {});
        
        store.clear();
        store.putAll(loadedStore);
    }
    
    record ScoredDocument(Document document, float score) {}
}
```

### SimpleVectorStoreContent

```java
/**
 * 简单向量存储内容
 */
public record SimpleVectorStoreContent(
    String id,
    String text,
    Map<String, Object> metadata,
    float[] embedding
) {}
```

### 使用示例

```java
@Configuration
public class SimpleVectorStoreConfig {
    
    @Bean
    public VectorStore simpleVectorStore(
            EmbeddingModel embeddingModel) {
        
        return SimpleVectorStore.builder(embeddingModel)
            .build();
    }
}

@Service
public class SimpleVectorStoreService {
    
    private final VectorStore vectorStore;
    
    /**
     * 添加文档
     */
    public void addDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }
    
    /**
     * 搜索
     */
    public List<Document> search(String query) {
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(5)
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    /**
     * 持久化
     */
    public void saveToFile(String filepath) throws IOException {
        if (vectorStore instanceof SimpleVectorStore simpleStore) {
            simpleStore.save(new File(filepath));
        }
    }
    
    /**
     * 加载
     */
    public void loadFromFile(String filepath) throws IOException {
        if (vectorStore instanceof SimpleVectorStore simpleStore) {
            simpleStore.load(new File(filepath));
        }
    }
}
```

---

## 支持的向量数据库

### 1. PostgreSQL (PgVector)

```java
@Configuration
public class PgVectorStoreConfig {
    
    @Bean
    public VectorStore pgVectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        
        return new PgVectorStore(
            jdbcTemplate,
            embeddingModel,
            PgVectorStoreOptions.builder()
                .tableName("vector_store")
                .schemaName("public")
                .indexType(PgIndexType.HNSW)
                .dimensions(1536)
                .build()
        );
    }
}
```

### 2. Chroma

```java
@Configuration
public class ChromaVectorStoreConfig {
    
    @Bean
    public VectorStore chromaVectorStore(
            ChromaApi chromaApi,
            EmbeddingModel embeddingModel) {
        
        return new ChromaVectorStore(
            embeddingModel,
            chromaApi,
            "spring_ai_collection",
            true // 初始化schema
        );
    }
}
```

### 3. Pinecone

```java
@Configuration
public class PineconeVectorStoreConfig {
    
    @Bean
    public VectorStore pineconeVectorStore(
            PineconeApi pineconeApi,
            EmbeddingModel embeddingModel) {
        
        return new PineconeVectorStore(
            pineconeApi,
            embeddingModel,
            PineconeVectorStoreOptions.builder()
                .namespace("default")
                .indexName("spring-ai-index")
                .build()
        );
    }
}
```

### 4. Redis

```java
@Configuration
public class RedisVectorStoreConfig {
    
    @Bean
    public VectorStore redisVectorStore(
            RedisVectorStoreConfig config,
            EmbeddingModel embeddingModel) {
        
        return new RedisVectorStore(
            config,
            embeddingModel,
            RedisVectorStoreOptions.builder()
                .indexName("spring-ai-index")
                .prefix("doc:")
                .build()
        );
    }
}
```

### 向量数据库对比

| 数据库 | 类型 | 优势 | 适用场景 |
|--------|------|------|----------|
| **PgVector** | 扩展 | PostgreSQL生态，SQL查询 | 已有PG的项目 |
| **Chroma** | 专用 | 开源，易用，内嵌模式 | 原型开发 |
| **Pinecone** | 云服务 | 托管服务，高性能 | 生产环境 |
| **Milvus** | 专用 | 开源，高性能，大规模 | 企业级应用 |
| **Qdrant** | 专用 | 高性能，Rust实现 | 高并发场景 |
| **Redis** | 缓存+ | 内存速度，Redis生态 | 实时检索 |
| **Weaviate** | 专用 | GraphQL，多租户 | 复杂查询 |
| **MongoDB Atlas** | 文档+ | MongoDB生态 | 已有Mongo的项目 |

---

## 使用指南

### 1. 添加依赖

```xml
<!-- Simple Vector Store (开发/测试) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vector-store</artifactId>
</dependency>

<!-- PgVector (PostgreSQL) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>

<!-- Chroma -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-chroma-store-spring-boot-starter</artifactId>
</dependency>

<!-- Redis -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-redis-store-spring-boot-starter</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  ai:
    vectorstore:
      pgvector:
        url: jdbc:postgresql://localhost:5432/vectordb
        username: postgres
        password: postgres
        table-name: vector_store
        dimensions: 1536
        index-type: HNSW
      
      chroma:
        url: http://localhost:8000
        collection-name: spring_ai_collection
      
      redis:
        url: redis://localhost:6379
        index-name: spring-ai-index
        prefix: "doc:"
```

### 3. 基本使用

```java
@Service
public class VectorStoreService {
    
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    /**
     * 添加单个文档
     */
    public void addDocument(String text, Map<String, Object> metadata) {
        Document doc = new Document(text, metadata);
        vectorStore.add(List.of(doc));
    }
    
    /**
     * 批量添加文档
     */
    public void addDocuments(List<String> texts) {
        List<Document> documents = texts.stream()
            .map(text -> new Document(
                text,
                Map.of("source", "import")
            ))
            .toList();
        
        vectorStore.add(documents);
    }
    
    /**
     * 简单搜索
     */
    public List<Document> search(String query) {
        return vectorStore.similaritySearch(query);
    }
    
    /**
     * 高级搜索
     */
    public List<Document> advancedSearch(
            String query,
            int topK,
            double threshold,
            String filterExpr) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .similarityThreshold(threshold)
            .filterExpression(filterExpr)
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    /**
     * 删除文档
     */
    public void deleteDocument(String id) {
        vectorStore.delete(List.of(id));
    }
    
    /**
     * 批量删除
     */
    public void deleteByFilter(String filterExpr) {
        vectorStore.delete(filterExpr);
    }
}
```

### 4. REST API示例

```java
@RestController
@RequestMapping("/api/vectorstore")
public class VectorStoreController {
    
    private final VectorStore vectorStore;
    
    /**
     * 添加文档
     */
    @PostMapping("/documents")
    public ResponseEntity<AddResult> addDocuments(
            @RequestBody AddRequest request) {
        
        try {
            List<Document> documents = request.texts().stream()
                .map(text -> new Document(
                    text,
                    request.metadata()
                ))
                .toList();
            
            vectorStore.add(documents);
            
            return ResponseEntity.ok(
                new AddResult(documents.size(), "success")
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new AddResult(0, "Error: " + e.getMessage()));
        }
    }
    
    /**
     * 搜索文档
     */
    @PostMapping("/search")
    public ResponseEntity<SearchResult> search(
            @RequestBody SearchRequestDto request) {
        
        try {
            SearchRequest searchRequest = SearchRequest.builder()
                .query(request.query())
                .topK(request.topK())
                .similarityThreshold(request.threshold())
                .filterExpression(request.filter())
                .build();
            
            List<Document> documents = 
                vectorStore.similaritySearch(searchRequest);
            
            List<DocumentDto> results = documents.stream()
                .map(doc -> new DocumentDto(
                    doc.getId(),
                    doc.getText(),
                    doc.getMetadata(),
                    doc.getScore()
                ))
                .toList();
            
            return ResponseEntity.ok(
                new SearchResult(results, "success")
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new SearchResult(
                    List.of(),
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    /**
     * 删除文档
     */
    @DeleteMapping("/documents/{id}")
    public ResponseEntity<DeleteResult> deleteDocument(
            @PathVariable String id) {
        
        try {
            vectorStore.delete(List.of(id));
            return ResponseEntity.ok(
                new DeleteResult(1, "success")
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new DeleteResult(
                    0,
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    record AddRequest(
        List<String> texts,
        Map<String, Object> metadata
    ) {}
    
    record AddResult(int count, String status) {}
    
    record SearchRequestDto(
        String query,
        int topK,
        double threshold,
        String filter
    ) {}
    
    record DocumentDto(
        String id,
        String text,
        Map<String, Object> metadata,
        Double score
    ) {}
    
    record SearchResult(
        List<DocumentDto> documents,
        String status
    ) {}
    
    record DeleteResult(int count, String status) {}
}
```

---

## 高级特性

### 1. 批量导入优化

```java
@Service
public class BatchImportService {
    
    private final VectorStore vectorStore;
    
    /**
     * 大批量导入（分批处理）
     */
    public void importLargeDataset(List<String> texts) {
        int batchSize = 100;
        
        for (int i = 0; i < texts.size(); i += batchSize) {
            int end = Math.min(i + batchSize, texts.size());
            List<String> batch = texts.subList(i, end);
            
            List<Document> documents = batch.stream()
                .map(text -> new Document(
                    text,
                    Map.of(
                        "batch", i / batchSize,
                        "imported_at", LocalDateTime.now().toString()
                    )
                ))
                .toList();
            
            vectorStore.add(documents);
            
            logger.info("Imported batch {}/{}", 
                i / batchSize + 1,
                (texts.size() + batchSize - 1) / batchSize
            );
        }
    }
}
```

### 2. 异步导入

```java
@Service
public class AsyncImportService {
    
    private final VectorStore vectorStore;
    
    /**
     * 异步批量导入
     */
    @Async
    public CompletableFuture<ImportResult> importAsync(
            List<String> texts) {
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                List<Document> documents = texts.stream()
                    .map(text -> new Document(text, Map.of()))
                    .toList();
                
                vectorStore.add(documents);
                
                return new ImportResult(documents.size(), "success");
                
            } catch (Exception e) {
                logger.error("Async import failed", e);
                return new ImportResult(0, "failed: " + e.getMessage());
            }
        });
    }
    
    record ImportResult(int count, String status) {}
}
```

### 3. 元数据丰富化

```java
@Service
public class MetadataEnrichmentService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 自动生成元数据
     */
    public void addDocumentWithEnrichedMetadata(String text) {
        
        // 使用AI生成元数据
        String metadataJson = chatClient
            .prompt("""
                分析以下文本，生成JSON格式的元数据：
                - category: 文本类别
                - tags: 关键词标签（数组）
                - summary: 简短摘要
                
                文本：
                {text}
                """)
            .param("text", text)
            .call()
            .content();
        
        // 解析JSON
        Map<String, Object> metadata = parseMetadata(metadataJson);
        metadata.put("created_at", LocalDateTime.now().toString());
        metadata.put("source", "enriched");
        
        // 添加文档
        Document doc = new Document(text, metadata);
        vectorStore.add(List.of(doc));
    }
    
    private Map<String, Object> parseMetadata(String json) {
        // JSON解析逻辑
        return Map.of(); // 简化
    }
}
```

### 4. 增量更新

```java
@Service
public class IncrementalUpdateService {
    
    private final VectorStore vectorStore;
    
    /**
     * 更新文档（删除+重新添加）
     */
    public void updateDocument(String id, String newText) {
        
        // 1. 删除旧文档
        vectorStore.delete(List.of(id));
        
        // 2. 添加新文档
        Document newDoc = new Document(
            id,
            newText,
            Map.of("updated_at", LocalDateTime.now().toString())
        );
        
        vectorStore.add(List.of(newDoc));
    }
    
    /**
     * 批量更新
     */
    public void updateDocuments(Map<String, String> updates) {
        
        // 1. 删除所有旧文档
        vectorStore.delete(new ArrayList<>(updates.keySet()));
        
        // 2. 添加所有新文档
        List<Document> newDocs = updates.entrySet()
            .stream()
            .map(entry -> new Document(
                entry.getKey(),
                entry.getValue(),
                Map.of("updated_at", LocalDateTime.now().toString())
            ))
            .toList();
        
        vectorStore.add(newDocs);
    }
}
```

---

## 实战场景

### 1. 知识库问答系统

```java
@Service
public class KnowledgeBaseService {
    
    private final VectorStore vectorStore;
    private final ChatClient chatClient;
    
    /**
     * 导入知识库文档
     */
    public void importKnowledgeBase(List<KBDocument> kbDocs) {
        
        List<Document> documents = kbDocs.stream()
            .map(kbDoc -> new Document(
                kbDoc.content(),
                Map.of(
                    "title", kbDoc.title(),
                    "category", kbDoc.category(),
                    "author", kbDoc.author(),
                    "date", kbDoc.date().toString(),
                    "tags", kbDoc.tags()
                )
            ))
            .toList();
        
        vectorStore.add(documents);
    }
    
    /**
     * 问答
     */
    public String ask(String question) {
        
        // 1. 检索相关文档
        SearchRequest searchRequest = SearchRequest.builder()
            .query(question)
            .topK(3)
            .similarityThreshold(0.7)
            .build();
        
        List<Document> relevantDocs = 
            vectorStore.similaritySearch(searchRequest);
        
        if (relevantDocs.isEmpty()) {
            return "抱歉，我在知识库中没有找到相关信息。";
        }
        
        // 2. 构建上下文
        String context = relevantDocs.stream()
            .map(doc -> "【文档】" + doc.getText())
            .collect(Collectors.joining("\n\n"));
        
        // 3. 生成答案
        return chatClient
            .prompt("""
                基于以下知识库文档回答问题：
                
                问题：{question}
                
                知识库：
                {context}
                
                请基于知识库内容回答，如果知识库中没有相关信息，请明确说明。
                """)
            .param("question", question)
            .param("context", context)
            .call()
            .content();
    }
    
    /**
     * 按分类检索
     */
    public List<Document> searchByCategory(
            String query,
            String category) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(10)
            .filterExpression("category == '" + category + "'")
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    record KBDocument(
        String title,
        String content,
        String category,
        String author,
        LocalDate date,
        List<String> tags
    ) {}
}
```

### 2. 文档去重系统

```java
@Service
public class DocumentDeduplicationService {
    
    private final VectorStore vectorStore;
    
    /**
     * 检查文档是否重复
     */
    public DuplicationCheckResult checkDuplication(
            String text,
            double threshold) {
        
        // 搜索相似文档
        SearchRequest request = SearchRequest.builder()
            .query(text)
            .topK(5)
            .similarityThreshold(threshold)
            .build();
        
        List<Document> similarDocs = 
            vectorStore.similaritySearch(request);
        
        if (similarDocs.isEmpty()) {
            return new DuplicationCheckResult(
                false,
                null,
                "No duplicate found"
            );
        }
        
        Document mostSimilar = similarDocs.get(0);
        
        return new DuplicationCheckResult(
            true,
            mostSimilar.getId(),
            "Similarity: " + mostSimilar.getScore()
        );
    }
    
    /**
     * 添加文档（带去重）
     */
    public AddResult addDocumentWithDeduplication(
            String text,
            Map<String, Object> metadata,
            double threshold) {
        
        // 检查重复
        DuplicationCheckResult dupCheck = 
            checkDuplication(text, threshold);
        
        if (dupCheck.isDuplicate()) {
            return new AddResult(
                false,
                null,
                "Duplicate of document: " + dupCheck.duplicateId()
            );
        }
        
        // 添加文档
        Document doc = new Document(text, metadata);
        vectorStore.add(List.of(doc));
        
        return new AddResult(
            true,
            doc.getId(),
            "Document added successfully"
        );
    }
    
    record DuplicationCheckResult(
        boolean isDuplicate,
        String duplicateId,
        String message
    ) {}
    
    record AddResult(
        boolean success,
        String documentId,
        String message
    ) {}
}
```

### 3. 智能文档推荐

```java
@Service
public class DocumentRecommendationService {
    
    private final VectorStore vectorStore;
    
    /**
     * 基于文档推荐相似文档
     */
    public List<Document> recommendSimilarDocuments(
            String documentId,
            int count) {
        
        // 1. 获取文档内容
        // 注意：实际实现可能需要单独的方法获取文档
        // 这里假设我们有文档文本
        String documentText = getDocumentText(documentId);
        
        // 2. 搜索相似文档
        SearchRequest request = SearchRequest.builder()
            .query(documentText)
            .topK(count + 1)  // +1因为会包含自己
            .build();
        
        List<Document> similar = 
            vectorStore.similaritySearch(request);
        
        // 3. 过滤掉自己
        return similar.stream()
            .filter(doc -> !doc.getId().equals(documentId))
            .limit(count)
            .toList();
    }
    
    /**
     * 基于用户历史推荐
     */
    public List<Document> recommendByUserHistory(
            String userId,
            int count) {
        
        // 1. 获取用户阅读历史
        List<String> history = getUserHistory(userId);
        
        // 2. 基于历史推荐
        // 为每个历史文档找相似的，然后合并去重
        Set<String> recommended = new HashSet<>();
        List<Document> results = new ArrayList<>();
        
        for (String historicalDocId : history) {
            String docText = getDocumentText(historicalDocId);
            
            SearchRequest request = SearchRequest.builder()
                .query(docText)
                .topK(5)
                .build();
            
            List<Document> similar = 
                vectorStore.similaritySearch(request);
            
            for (Document doc : similar) {
                if (!recommended.contains(doc.getId()) &&
                    !history.contains(doc.getId())) {
                    
                    recommended.add(doc.getId());
                    results.add(doc);
                    
                    if (results.size() >= count) {
                        return results;
                    }
                }
            }
        }
        
        return results;
    }
    
    private String getDocumentText(String documentId) {
        // 实现获取文档文本的逻辑
        return "";
    }
    
    private List<String> getUserHistory(String userId) {
        // 实现获取用户历史的逻辑
        return List.of();
    }
}
```

---

## 最佳实践

### 1. 向量数据库选择

```java
@Configuration
public class VectorStoreSelectionConfig {
    
    /**
     * 开发环境：SimpleVectorStore
     */
    @Bean
    @Profile("dev")
    public VectorStore devVectorStore(EmbeddingModel embeddingModel) {
        return SimpleVectorStore.builder(embeddingModel).build();
    }
    
    /**
     * 测试环境：Chroma（轻量级）
     */
    @Bean
    @Profile("test")
    public VectorStore testVectorStore(
            ChromaApi chromaApi,
            EmbeddingModel embeddingModel) {
        return new ChromaVectorStore(embeddingModel, chromaApi);
    }
    
    /**
     * 生产环境：PgVector（企业级）
     */
    @Bean
    @Profile("prod")
    public VectorStore prodVectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        return new PgVectorStore(jdbcTemplate, embeddingModel);
    }
}
```

### 2. 错误处理

```java
@Service
public class RobustVectorStoreService {
    
    private final VectorStore vectorStore;
    private final RetryTemplate retryTemplate;
    
    /**
     * 带重试的添加
     */
    public void addWithRetry(List<Document> documents) {
        retryTemplate.execute(context -> {
            try {
                vectorStore.add(documents);
                return null;
            } catch (Exception e) {
                logger.error("Add failed, attempt: {}", 
                    context.getRetryCount(), e);
                throw e;
            }
        });
    }
    
    /**
     * 带降级的搜索
     */
    public List<Document> searchWithFallback(String query) {
        try {
            return vectorStore.similaritySearch(query);
        } catch (Exception e) {
            logger.error("Search failed, returning empty list", e);
            return List.of();
        }
    }
}
```

### 3. 性能监控

```java
@Service
public class MonitoredVectorStoreService {
    
    private final VectorStore vectorStore;
    private final MeterRegistry meterRegistry;
    
    /**
     * 带监控的搜索
     */
    public List<Document> searchWithMetrics(String query) {
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            List<Document> results = vectorStore.similaritySearch(query);
            
            // 记录成功
            sample.stop(Timer.builder("vectorstore.search")
                .tag("status", "success")
                .register(meterRegistry));
            
            // 记录结果数量
            meterRegistry.counter("vectorstore.search.results")
                .increment(results.size());
            
            return results;
            
        } catch (Exception e) {
            // 记录失败
            sample.stop(Timer.builder("vectorstore.search")
                .tag("status", "failure")
                .register(meterRegistry));
            
            throw e;
        }
    }
}
```

---

## 总结

### VectorStore核心特点

1. **语义存储**: 基于向量的文档存储
2. **相似度检索**: 快速找到相关文档
3. **元数据过滤**: 灵活的过滤表达式
4. **多数据库支持**: 20+种向量数据库
5. **统一API**: 跨数据库通用接口

### Spring AI VectorStore API

```
VectorStore (接口)
    ↓
add(List<Document>)  // 添加文档
    ↓
similaritySearch(SearchRequest)  // 相似度搜索
    ↓
delete(List<String>)  // 删除文档
```

### 核心配置

- **topK**: 返回结果数量
- **similarityThreshold**: 相似度阈值
- **filterExpression**: 元数据过滤
- **indexType**: 索引类型（HNSW、IVF等）

### 最佳实践清单

- ✅ 根据场景选择合适的向量数据库
- ✅ 使用元数据过滤提高检索精度
- ✅ 批量处理大规模数据导入
- ✅ 实现错误处理和重试机制
- ✅ 监控性能和成本
- ✅ 定期清理无用数据
- ✅ 使用适当的相似度阈值

通过Spring AI的VectorStore功能，你可以轻松构建强大的语义搜索、RAG应用、智能推荐等系统！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-05  
**Spring AI版本**: 1.1.0-SNAPSHOT
