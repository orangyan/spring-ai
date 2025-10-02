# Spring AI VectorStore 向量数据库详解

## 📋 目录
- [概述](#概述)
- [核心概念](#核心概念)
- [接口体系](#接口体系)
- [Document文档模型](#document文档模型)
- [Embedding嵌入向量](#embedding嵌入向量)
- [SearchRequest检索请求](#searchrequest检索请求)
- [Filter过滤表达式](#filter过滤表达式)
- [向量数据库实现](#向量数据库实现)
- [使用指南](#使用指南)
- [高级特性](#高级特性)
- [最佳实践](#最佳实践)
- [实战示例](#实战示例)

---

## 概述

### 什么是向量数据库？

向量数据库是一种专门用于AI应用的特殊数据库，它存储数据的**向量表示（embeddings）**而不是原始数据。向量数据库的核心特点是执行**相似性搜索**而非传统的精确匹配。

```
传统数据库: WHERE name = '张三'        (精确匹配)
向量数据库: WHERE similar(embedding, query_embedding) > 0.7  (相似性搜索)
```

### 工作原理

```
1. 文本内容 → 2. Embedding模型 → 3. 向量表示 → 4. 存储到向量数据库

查询:
1. 查询文本 → 2. Embedding模型 → 3. 查询向量 → 4. 相似性搜索 → 5. 返回相似文档
```

### 为什么需要向量数据库？

1. **语义搜索**: 理解查询的含义，而不仅仅是关键词匹配
2. **RAG应用**: 为大语言模型提供相关上下文
3. **推荐系统**: 找到相似的商品、文章、用户等
4. **异常检测**: 识别与正常模式不同的数据
5. **多模态搜索**: 跨文本、图片、音频等多种模态搜索

### Spring AI的设计理念

Spring AI通过统一的`VectorStore`接口抽象了各种向量数据库的实现，让开发者可以：

- ✅ **无缝切换**向量数据库（PgVector、Chroma、Pinecone等）
- ✅ **统一API**，学习一次处处使用
- ✅ **可移植过滤表达式**，跨数据库工作
- ✅ **Spring生态集成**，自动配置和依赖注入
- ✅ **可观测性**，集成Micrometer监控

---

## 核心概念

### 1. 向量（Vector/Embedding）

向量是数据的数值表示，通常是一个浮点数数组。

```java
// 文本 "Spring Boot" 的向量表示（简化示例）
float[] embedding = new float[]{
    0.123, -0.456, 0.789, ..., 0.321  // 1536维（OpenAI）
};
```

**特点**:
- **维度**: 通常是768、1536、3072等
- **范围**: 通常在[-1, 1]或[0, 1]之间
- **语义**: 相似的内容有相似的向量

### 2. 相似度度量

衡量两个向量的相似程度。

#### 余弦相似度（Cosine Similarity）

```java
// 最常用，取值范围 [-1, 1]
// 1: 完全相同
// 0: 正交（无关）
// -1: 完全相反

cosine_similarity = (A · B) / (||A|| * ||B||)
```

#### 欧几里得距离（Euclidean Distance）

```java
// L2距离，值越小越相似
euclidean_distance = sqrt(Σ(A[i] - B[i])²)
```

#### 内积（Inner Product）

```java
// 点积，值越大越相似（归一化向量）
inner_product = A · B = Σ(A[i] * B[i])
```

### 3. 向量索引

为了高效检索，向量数据库使用特殊的索引结构。

#### HNSW (Hierarchical Navigable Small World)

```
特点:
- 高效的近似最近邻搜索
- 查询速度快
- 内存占用较大
- 构建索引慢

适用场景:
- 实时查询要求高
- 数据集不频繁更新
```

#### IVFFlat (Inverted File with Flat Compression)

```
特点:
- 将向量分组到聚类中
- 构建索引快
- 内存占用小
- 查询速度中等

适用场景:
- 数据集频繁更新
- 内存受限
```

#### Flat（暴力搜索）

```
特点:
- 精确搜索，100%准确
- 无索引开销
- 查询慢（O(n)）

适用场景:
- 小数据集
- 需要精确结果
```

---

## 接口体系

### 1. VectorStore接口（完整功能）

```java
/**
 * 向量数据库完整接口
 * 提供读写操作
 */
public interface VectorStore 
    extends DocumentWriter, VectorStoreRetriever {
    
    /**
     * 获取向量库名称
     */
    default String getName() {
        return this.getClass().getSimpleName();
    }
    
    /**
     * 添加文档
     * @param documents 文档列表
     */
    void add(List<Document> documents);
    
    /**
     * 删除文档（按ID）
     * @param idList 文档ID列表
     */
    void delete(List<String> idList);
    
    /**
     * 删除文档（按过滤条件）
     * @param filterExpression 过滤表达式
     */
    void delete(Filter.Expression filterExpression);
    
    /**
     * 删除文档（按字符串过滤表达式）
     * @param filterExpression SQL样式的过滤表达式
     */
    default void delete(String filterExpression) {
        SearchRequest request = SearchRequest.builder()
            .filterExpression(filterExpression)
            .build();
        delete(request.getFilterExpression());
    }
    
    /**
     * 相似性搜索
     * @param request 搜索请求
     * @return 相似文档列表
     */
    List<Document> similaritySearch(SearchRequest request);
    
    /**
     * 相似性搜索（简化版）
     * @param query 查询文本
     * @return 相似文档列表
     */
    default List<Document> similaritySearch(String query) {
        return similaritySearch(
            SearchRequest.builder().query(query).build()
        );
    }
    
    /**
     * 获取原生客户端（可选）
     */
    default <T> Optional<T> getNativeClient() {
        return Optional.empty();
    }
}
```

### 2. VectorStoreRetriever接口（只读）

```java
/**
 * 只读向量检索接口
 * 遵循最小权限原则，只暴露检索功能
 */
@FunctionalInterface
public interface VectorStoreRetriever {
    
    /**
     * 相似性搜索
     */
    List<Document> similaritySearch(SearchRequest request);
    
    /**
     * 简化的相似性搜索
     */
    default List<Document> similaritySearch(String query) {
        return similaritySearch(
            SearchRequest.builder().query(query).build()
        );
    }
}
```

### 3. 接口层次结构

```
DocumentWriter (写入接口)
    ↑
    │
VectorStore ←────── VectorStoreRetriever (读取接口)
    │                       ↑
    │                       │ (只读访问)
    ├─ add()               │
    ├─ delete()            └─ similaritySearch()
    └─ similaritySearch()

使用场景:
- VectorStore: 完整功能，用于数据管理
- VectorStoreRetriever: 只读访问，用于检索（如RAG）
```

### 4. Builder接口

```java
/**
 * VectorStore构建器
 */
interface Builder<T extends Builder<T>> {
    
    /**
     * 设置观测注册表
     */
    T observationRegistry(ObservationRegistry registry);
    
    /**
     * 设置自定义观测约定
     */
    T customObservationConvention(
        VectorStoreObservationConvention convention
    );
    
    /**
     * 设置批处理策略
     */
    T batchingStrategy(BatchingStrategy strategy);
    
    /**
     * 构建VectorStore实例
     */
    VectorStore build();
}
```

---

## Document文档模型

### Document类

```java
/**
 * 文档是向量库的基本存储单元
 * 包含文本内容、元数据和可选的嵌入向量
 */
public class Document implements Serializable {
    
    /**
     * 文档唯一标识符
     */
    private String id;
    
    /**
     * 文档文本内容
     */
    private String text;
    
    /**
     * 文档元数据（可搜索）
     */
    private Map<String, Object> metadata;
    
    /**
     * 嵌入向量（可选）
     */
    @Nullable
    private float[] embedding;
    
    /**
     * 媒体数据（可选，用于多模态）
     */
    @Nullable
    private List<Media> media;
    
    // 构造方法
    public Document(String text) {
        this(UUID.randomUUID().toString(), text, Map.of());
    }
    
    public Document(String text, Map<String, Object> metadata) {
        this(UUID.randomUUID().toString(), text, metadata);
    }
    
    public Document(String id, String text, 
                   Map<String, Object> metadata) {
        this.id = id;
        this.text = text;
        this.metadata = new HashMap<>(metadata);
    }
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private String id = UUID.randomUUID().toString();
        private String text;
        private Map<String, Object> metadata = new HashMap<>();
        private float[] embedding;
        
        public Builder id(String id) {
            this.id = id;
            return this;
        }
        
        public Builder text(String text) {
            this.text = text;
            return this;
        }
        
        public Builder metadata(String key, Object value) {
            this.metadata.put(key, value);
            return this;
        }
        
        public Builder metadata(Map<String, Object> metadata) {
            this.metadata.putAll(metadata);
            return this;
        }
        
        public Builder embedding(float[] embedding) {
            this.embedding = embedding;
            return this;
        }
        
        public Document build() {
            Document doc = new Document(id, text, metadata);
            if (embedding != null) {
                doc.setEmbedding(embedding);
            }
            return doc;
        }
    }
}
```

### 创建Document示例

```java
// 1. 最简单的方式
Document doc1 = new Document("Spring Boot是一个快速开发框架");

// 2. 带元数据
Document doc2 = new Document(
    "Spring Boot是一个快速开发框架",
    Map.of(
        "category", "技术",
        "language", "Java",
        "year", 2014,
        "author", "Pivotal"
    )
);

// 3. 使用Builder
Document doc3 = Document.builder()
    .id("doc-001")
    .text("Spring Boot是一个快速开发框架")
    .metadata("category", "技术")
    .metadata("language", "Java")
    .metadata("year", 2014)
    .metadata("tags", List.of("framework", "microservice"))
    .build();

// 4. 带预计算的嵌入向量
Document doc4 = Document.builder()
    .text("Spring Boot是一个快速开发框架")
    .metadata("source", "documentation")
    .embedding(precomputedEmbedding)  // float[]
    .build();
```

### 元数据最佳实践

```java
// 推荐的元数据结构
Document doc = Document.builder()
    .text(content)
    
    // 来源信息
    .metadata("source", "document.pdf")
    .metadata("sourceType", "pdf")
    .metadata("page", 5)
    
    // 分类信息
    .metadata("category", "技术文档")
    .metadata("subcategory", "架构设计")
    .metadata("tags", List.of("微服务", "Spring"))
    
    // 时间信息
    .metadata("createdAt", Instant.now())
    .metadata("updatedAt", Instant.now())
    .metadata("year", 2024)
    
    // 访问控制
    .metadata("tenantId", "company-001")
    .metadata("department", "研发部")
    .metadata("accessLevel", "internal")
    
    // 内容特征
    .metadata("language", "zh-CN")
    .metadata("wordCount", 500)
    .metadata("complexity", "intermediate")
    
    .build();
```

---

## Embedding嵌入向量

### EmbeddingModel接口

```java
/**
 * 嵌入模型接口
 * 将文本转换为向量表示
 */
public interface EmbeddingModel 
    extends Model<EmbeddingRequest, EmbeddingResponse> {
    
    /**
     * 嵌入单个文本
     * @param text 要嵌入的文本
     * @return 嵌入向量
     */
    default float[] embed(String text) {
        List<float[]> embeddings = embed(List.of(text));
        return embeddings.get(0);
    }
    
    /**
     * 嵌入文档
     * @param document 要嵌入的文档
     * @return 嵌入向量
     */
    float[] embed(Document document);
    
    /**
     * 批量嵌入文本
     * @param texts 文本列表
     * @return 嵌入向量列表
     */
    default List<float[]> embed(List<String> texts) {
        return call(new EmbeddingRequest(texts, null))
            .getResults()
            .stream()
            .map(Embedding::getOutput)
            .toList();
    }
    
    /**
     * 批量嵌入文档（带批处理策略）
     */
    default List<float[]> embed(
            List<Document> documents,
            EmbeddingOptions options,
            BatchingStrategy batchingStrategy) {
        // 实现批处理逻辑
    }
}
```

### 常见的Embedding模型

```java
// 1. OpenAI Embeddings
OpenAiEmbeddingModel embeddingModel = OpenAiEmbeddingModel.builder()
    .apiKey(apiKey)
    .modelName("text-embedding-3-small")  // 1536维
    // .modelName("text-embedding-3-large")  // 3072维
    .build();

// 2. Azure OpenAI
AzureOpenAiEmbeddingModel embeddingModel = 
    AzureOpenAiEmbeddingModel.builder()
        .endpoint(endpoint)
        .apiKey(apiKey)
        .deploymentName(deploymentName)
        .build();

// 3. Ollama (本地)
OllamaEmbeddingModel embeddingModel = OllamaEmbeddingModel.builder()
    .baseUrl("http://localhost:11434")
    .modelName("nomic-embed-text")  // 768维
    .build();

// 4. Transformers (本地)
TransformersEmbeddingModel embeddingModel = 
    new TransformersEmbeddingModel();

// 5. Google VertexAI
VertexAiEmbeddingModel embeddingModel = 
    VertexAiEmbeddingModel.builder()
        .projectId(projectId)
        .location("us-central1")
        .modelName("textembedding-gecko@003")
        .build();
```

### 使用示例

```java
@Service
public class EmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    public float[] embedText(String text) {
        return embeddingModel.embed(text);
    }
    
    public List<float[]> embedBatch(List<String> texts) {
        return embeddingModel.embed(texts);
    }
    
    public Document embedDocument(Document document) {
        float[] embedding = embeddingModel.embed(document);
        document.setEmbedding(embedding);
        return document;
    }
    
    public List<Document> embedDocuments(List<Document> documents) {
        return documents.stream()
            .map(this::embedDocument)
            .toList();
    }
    
    // 计算相似度
    public double cosineSimilarity(float[] vec1, float[] vec2) {
        double dotProduct = 0.0;
        double norm1 = 0.0;
        double norm2 = 0.0;
        
        for (int i = 0; i < vec1.length; i++) {
            dotProduct += vec1[i] * vec2[i];
            norm1 += vec1[i] * vec1[i];
            norm2 += vec2[i] * vec2[i];
        }
        
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
}
```

---

## SearchRequest检索请求

### SearchRequest类

```java
/**
 * 相似性搜索请求配置
 */
public class SearchRequest {
    
    /**
     * 接受所有相似度的阈值
     */
    public static final double SIMILARITY_THRESHOLD_ACCEPT_ALL = 0.0;
    
    /**
     * 默认返回Top K结果数
     */
    public static final int DEFAULT_TOP_K = 4;
    
    /**
     * 查询文本
     */
    private String query = "";
    
    /**
     * 返回Top K个最相似结果
     */
    private int topK = DEFAULT_TOP_K;
    
    /**
     * 相似度阈值 (0.0 - 1.0)
     */
    private double similarityThreshold = SIMILARITY_THRESHOLD_ACCEPT_ALL;
    
    /**
     * 元数据过滤表达式
     */
    @Nullable
    private Filter.Expression filterExpression;
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        
        /**
         * 设置查询文本
         */
        public Builder query(String query) {
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
         * @param threshold 0.0 - 1.0
         */
        public Builder similarityThreshold(double threshold) {
            Assert.isTrue(threshold >= 0 && threshold <= 1, 
                "Similarity threshold must be in [0,1] range.");
            this.searchRequest.similarityThreshold = threshold;
            return this;
        }
        
        /**
         * 接受所有相似度（阈值=0.0）
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
         * 设置过滤表达式（字符串）
         */
        public Builder filterExpression(String expression) {
            if (StringUtils.hasText(expression)) {
                FilterExpressionTextParser parser = 
                    new FilterExpressionTextParser();
                this.searchRequest.filterExpression = 
                    parser.parse(expression);
            }
            return this;
        }
        
        public SearchRequest build() {
            return this.searchRequest;
        }
    }
}
```

### 使用示例

```java
@Service
public class VectorSearchService {
    
    private final VectorStore vectorStore;
    
    // 1. 最简单的搜索
    public List<Document> simpleSearch(String query) {
        return vectorStore.similaritySearch(query);
    }
    
    // 2. 限制返回数量
    public List<Document> searchTopK(String query, int topK) {
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    // 3. 设置相似度阈值
    public List<Document> searchWithThreshold(
            String query, 
            double threshold) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(10)
            .similarityThreshold(threshold)  // 0.7表示70%以上相似
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    // 4. 带过滤条件
    public List<Document> searchWithFilter(
            String query,
            String category) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(5)
            .similarityThreshold(0.7)
            .filterExpression("category == '" + category + "'")
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    // 5. 复杂搜索
    public List<Document> advancedSearch(
            String query,
            int topK,
            double threshold,
            String department,
            int minYear) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .similarityThreshold(threshold)
            .filterExpression(String.format(
                "department == '%s' && year >= %d",
                department, minYear
            ))
            .build();
        
        return vectorStore.similaritySearch(request);
    }
}
```

---

## Filter过滤表达式

### Filter.Expression（表达式树）

```java
/**
 * 过滤表达式
 * 跨所有向量数据库可移植
 */
public interface Filter {
    
    /**
     * 表达式节点
     */
    record Expression(
        ExpressionType type,
        Expression left,
        Expression right,
        Key key,
        Value value
    ) {}
    
    /**
     * 表达式类型
     */
    enum ExpressionType {
        // 逻辑运算符
        AND, OR, NOT,
        
        // 比较运算符
        EQ,   // ==
        NE,   // !=
        LT,   // <
        LTE,  // <=
        GT,   // >
        GTE,  // >=
        
        // 特殊运算符
        IN,   // in
        NIN,  // not in
    }
    
    /**
     * 元数据键
     */
    record Key(String key) {}
    
    /**
     * 值
     */
    record Value(Object value) {}
}
```

### 创建过滤表达式的三种方式

#### 1. 程序化构建

```java
// 构建: country == 'China' && year >= 2020
Filter.Expression expression = new Filter.Expression(
    AND,
    new Filter.Expression(
        EQ,
        new Filter.Key("country"),
        new Filter.Value("China")
    ),
    new Filter.Expression(
        GTE,
        new Filter.Key("year"),
        new Filter.Value(2020)
    )
);
```

#### 2. 使用FilterExpressionBuilder (DSL)

```java
FilterExpressionBuilder builder = new FilterExpressionBuilder();

// 简单条件
Filter.Expression exp1 = builder.eq("country", "China");

// 复合条件
Filter.Expression exp2 = builder.and(
    builder.eq("country", "China"),
    builder.gte("year", 2020),
    builder.eq("isActive", true)
);

// 复杂嵌套
Filter.Expression exp3 = builder.and(
    builder.or(
        builder.eq("country", "China"),
        builder.eq("country", "USA")
    ),
    builder.and(
        builder.gte("year", 2020"),
        builder.lte("year", 2024)
    ),
    builder.in("category", List.of("Tech", "Science"))
);
```

#### 3. 文本解析（最推荐）

```java
FilterExpressionTextParser parser = 
    new FilterExpressionTextParser();

// 简单表达式
Filter.Expression exp1 = parser.parse("country == 'China'");

// 复合表达式
Filter.Expression exp2 = parser.parse(
    "country == 'China' && year >= 2020"
);

// 复杂表达式
Filter.Expression exp3 = parser.parse("""
    (country == 'China' || country == 'USA') &&
    year >= 2020 && year <= 2024 &&
    category in ['Tech', 'Science'] &&
    isActive == true &&
    price > 100.0
    """
);
```

### 过滤表达式语法

```sql
-- 比较运算符
key == value       -- 等于
key != value       -- 不等于
key > value        -- 大于
key >= value       -- 大于等于
key < value        -- 小于
key <= value       -- 小于等于

-- 逻辑运算符
expr1 && expr2     -- 与
expr1 || expr2     -- 或
!expr              -- 非

-- IN运算符
key in [val1, val2, val3]      -- 包含
key nin [val1, val2, val3]     -- 不包含

-- 值类型
'string'           -- 字符串
123                -- 整数
123.45             -- 小数
true/false         -- 布尔值
[val1, val2]       -- 数组
```

### 实战示例

```java
@Service
public class FilteredSearchService {
    
    private final VectorStore vectorStore;
    private final FilterExpressionTextParser parser = 
        new FilterExpressionTextParser();
    
    // 1. 按类别过滤
    public List<Document> searchByCategory(
            String query,
            String category) {
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)
                .filterExpression("category == '" + category + "'")
                .build()
        );
    }
    
    // 2. 按时间范围过滤
    public List<Document> searchByDateRange(
            String query,
            int startYear,
            int endYear) {
        
        String filter = String.format(
            "year >= %d && year <= %d",
            startYear, endYear
        );
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .build()
        );
    }
    
    // 3. 多条件组合
    public List<Document> advancedFilter(
            String query,
            String department,
            List<String> tags,
            boolean isActive) {
        
        String filter = String.format(
            "department == '%s' && " +
            "tags in %s && " +
            "isActive == %s",
            department,
            tags.toString().replace("[", "['").replace("]", "']")
                .replace(", ", "', '"),
            isActive
        );
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .build()
        );
    }
    
    // 4. 租户隔离
    public List<Document> searchByTenant(
            String query,
            String tenantId,
            String userId) {
        
        String filter = String.format(
            "tenantId == '%s' && " +
            "(accessLevel == 'public' || ownerId == '%s')",
            tenantId, userId
        );
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .build()
        );
    }
    
    // 5. 动态构建过滤器
    public List<Document> dynamicSearch(
            String query,
            Map<String, Object> filters) {
        
        List<String> conditions = new ArrayList<>();
        
        if (filters.containsKey("category")) {
            conditions.add(
                "category == '" + filters.get("category") + "'"
            );
        }
        
        if (filters.containsKey("minYear")) {
            conditions.add(
                "year >= " + filters.get("minYear")
            );
        }
        
        if (filters.containsKey("tags")) {
            @SuppressWarnings("unchecked")
            List<String> tags = (List<String>) filters.get("tags");
            String tagsStr = tags.stream()
                .map(t -> "'" + t + "'")
                .collect(Collectors.joining(", "));
            conditions.add("tags in [" + tagsStr + "]");
        }
        
        String filter = String.join(" && ", conditions);
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .build()
        );
    }
}
```

---

## 向量数据库实现

Spring AI支持20+种向量数据库，每种都有其特点和适用场景。

### 1. PgVector (PostgreSQL)

**特点**:
- ✅ 基于PostgreSQL，稳定可靠
- ✅ ACID事务支持
- ✅ 丰富的SQL生态
- ✅ 免费开源

**配置**:

```java
@Configuration
public class PgVectorConfig {
    
    @Bean
    public VectorStore pgVectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
            .schemaName("public")
            .vectorTableName("vector_store")
            .dimensions(1536)  // OpenAI embedding维度
            .distanceType(PgDistanceType.COSINE_DISTANCE)
            .indexType(PgIndexType.HNSW)
            .initializeSchema(true)  // 自动创建表
            .build();
    }
}
```

**使用**:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: password
  ai:
    vectorstore:
      pgvector:
        initialize-schema: true
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
```

### 2. Chroma

**特点**:
- ✅ 专为AI应用设计
- ✅ 简单易用
- ✅ 开发友好
- ✅ 支持Docker部署

**配置**:

```java
@Configuration
public class ChromaConfig {
    
    @Bean
    public VectorStore chromaVectorStore(
            ChromaApi chromaApi,
            EmbeddingModel embeddingModel) {
        
        return ChromaVectorStore.builder(chromaApi, embeddingModel)
            .collectionName("my_collection")
            .initializeSchema(true)
            .build();
    }
    
    @Bean
    public ChromaApi chromaApi() {
        return ChromaApi.create("http://localhost:8000");
    }
}
```

### 3. Pinecone

**特点**:
- ✅ 完全托管的云服务
- ✅ 性能卓越
- ✅ 自动扩展
- ❌ 收费服务

**配置**:

```java
@Configuration
public class PineconeConfig {
    
    @Bean
    public VectorStore pineconeVectorStore(
            PineconeApi pineconeApi,
            EmbeddingModel embeddingModel) {
        
        return PineconeVectorStore.builder(pineconeApi, embeddingModel)
            .indexName("my-index")
            .namespace("default")
            .contentFieldName("text")
            .build();
    }
}
```

### 4. Redis

**特点**:
- ✅ 超快的内存数据库
- ✅ 支持持久化
- ✅ 丰富的数据结构
- ✅ 适合实时场景

**配置**:

```java
@Configuration
public class RedisVectorConfig {
    
    @Bean
    public VectorStore redisVectorStore(
            JedisPooled jedis,
            EmbeddingModel embeddingModel) {
        
        return RedisVectorStore.builder(jedis, embeddingModel)
            .indexName("spring-ai-index")
            .prefix("embedding:")
            .vectorAlgorithm(Algorithm.HNSW)
            .build();
    }
}
```

### 5. MongoDB Atlas

**特点**:
- ✅ 文档数据库
- ✅ 托管服务
- ✅ 全局分布
- ✅ 丰富的查询能力

**配置**:

```java
@Configuration
public class MongoDBAtlasConfig {
    
    @Bean
    public VectorStore mongoDBAtlasVectorStore(
            MongoTemplate mongoTemplate,
            EmbeddingModel embeddingModel) {
        
        return MongoDBAtlasVectorStore.builder(
                mongoTemplate, 
                embeddingModel
            )
            .collectionName("vector_store")
            .pathName("embedding")
            .indexName("vector_index")
            .build();
    }
}
```

### 6. SimpleVectorStore（内存）

**特点**:
- ✅ 无需外部依赖
- ✅ 开发测试方便
- ✅ 可持久化到文件
- ❌ 不适合生产环境

**配置**:

```java
@Configuration
public class SimpleVectorStoreConfig {
    
    @Bean
    public VectorStore simpleVectorStore(
            EmbeddingModel embeddingModel) {
        
        SimpleVectorStore vectorStore = 
            SimpleVectorStore.builder(embeddingModel)
                .build();
        
        // 可选：从文件加载
        File storeFile = new File("vector-store.json");
        if (storeFile.exists()) {
            vectorStore.load(storeFile);
        }
        
        return vectorStore;
    }
}
```

### 向量数据库选择指南

| 数据库 | 适用场景 | 优势 | 劣势 |
|--------|---------|------|------|
| **PgVector** | 生产环境、需要ACID | 稳定、SQL生态 | 性能中等 |
| **Chroma** | 开发、原型、小规模 | 简单、免费 | 功能较少 |
| **Pinecone** | 高并发、大规模 | 性能好、托管 | 收费较贵 |
| **Redis** | 实时应用、缓存 | 超快速度 | 内存占用大 |
| **MongoDB** | 混合查询、文档存储 | 灵活查询 | 成本较高 |
| **Milvus** | 大规模、高性能 | 专业向量库 | 部署复杂 |
| **Qdrant** | Rust性能、开源 | 快速、特性丰富 | 社区较小 |
| **Weaviate** | 知识图谱、多模态 | 功能丰富 | 学习曲线陡 |
| **SimpleVectorStore** | 开发测试 | 零依赖 | 不可用于生产 |

---

## 使用指南

### 1. 添加依赖

```xml
<!-- 核心依赖 -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vector-store</artifactId>
</dependency>

<!-- 选择向量数据库（以PgVector为例） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store</artifactId>
</dependency>

<!-- Embedding模型（以OpenAI为例） -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

### 2. 配置向量数据库

```java
@Configuration
public class VectorStoreConfig {
    
    @Bean
    public VectorStore vectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
            .schemaName("public")
            .vectorTableName("vector_store")
            .dimensions(1536)
            .distanceType(PgDistanceType.COSINE_DISTANCE)
            .indexType(PgIndexType.HNSW)
            .initializeSchema(true)
            .build();
    }
}
```

### 3. 导入文档

```java
@Service
public class DocumentIngestionService {
    
    private final VectorStore vectorStore;
    private final DocumentReader documentReader;
    
    /**
     * 导入单个文档
     */
    public void ingestDocument(String text, Map<String, Object> metadata) {
        Document document = new Document(text, metadata);
        vectorStore.add(List.of(document));
    }
    
    /**
     * 导入多个文档
     */
    public void ingestDocuments(List<Document> documents) {
        vectorStore.add(documents);
    }
    
    /**
     * 从文件导入
     */
    public void ingestFromFile(Resource resource) {
        // 读取文档
        List<Document> documents = documentReader.read(resource);
        
        // 添加元数据
        documents.forEach(doc -> 
            doc.getMetadata().put("source", resource.getFilename())
        );
        
        // 导入向量库
        vectorStore.add(documents);
    }
    
    /**
     * 批量导入（大文件）
     */
    public void batchIngest(List<Document> documents, int batchSize) {
        for (int i = 0; i < documents.size(); i += batchSize) {
            int end = Math.min(i + batchSize, documents.size());
            List<Document> batch = documents.subList(i, end);
            
            vectorStore.add(batch);
            
            logger.info("Ingested batch {}/{}", 
                (i / batchSize) + 1,
                (documents.size() + batchSize - 1) / batchSize
            );
        }
    }
}
```

### 4. 搜索文档

```java
@Service
public class DocumentSearchService {
    
    private final VectorStore vectorStore;
    
    /**
     * 基本搜索
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
            String filter) {
        
        SearchRequest request = SearchRequest.builder()
            .query(query)
            .topK(topK)
            .similarityThreshold(threshold)
            .filterExpression(filter)
            .build();
        
        return vectorStore.similaritySearch(request);
    }
    
    /**
     * 搜索并格式化结果
     */
    public List<SearchResult> searchWithScore(String query) {
        List<Document> documents = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(5)
                .build()
        );
        
        return documents.stream()
            .map(doc -> new SearchResult(
                doc.getId(),
                doc.getText(),
                doc.getMetadata(),
                // 相似度分数通常存在元数据中
                (Double) doc.getMetadata().get("distance")
            ))
            .toList();
    }
    
    record SearchResult(
        String id,
        String text,
        Map<String, Object> metadata,
        Double similarity
    ) {}
}
```

### 5. 更新和删除

```java
@Service
public class DocumentManagementService {
    
    private final VectorStore vectorStore;
    
    /**
     * 更新文档（先删除再添加）
     */
    public void updateDocument(Document document) {
        // 删除旧文档
        vectorStore.delete(List.of(document.getId()));
        
        // 添加新文档
        vectorStore.add(List.of(document));
    }
    
    /**
     * 批量更新
     */
    public void updateDocuments(List<Document> documents) {
        // 提取ID
        List<String> ids = documents.stream()
            .map(Document::getId)
            .toList();
        
        // 删除
        vectorStore.delete(ids);
        
        // 重新添加
        vectorStore.add(documents);
    }
    
    /**
     * 按ID删除
     */
    public void deleteById(String id) {
        vectorStore.delete(List.of(id));
    }
    
    /**
     * 按过滤条件删除
     */
    public void deleteByFilter(String filterExpression) {
        vectorStore.delete(filterExpression);
    }
    
    /**
     * 清空所有文档（危险操作）
     */
    public void deleteAll() {
        // 根据具体实现可能需要不同的方法
        // 某些向量库支持删除整个集合
    }
}
```

---

## 高级特性

### 1. 批处理策略

```java
/**
 * 批处理策略
 * 用于优化大量文档的embedding生成
 */
@Configuration
public class BatchingConfig {
    
    @Bean
    public VectorStore vectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel) {
        
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
            .batchingStrategy(new TokenCountBatchingStrategy())
            // 或者
            // .batchingStrategy(new MaxBatchSizeBatchingStrategy(100))
            .build();
    }
}

// 自定义批处理策略
public class CustomBatchingStrategy implements BatchingStrategy {
    
    @Override
    public List<List<Document>> batch(List<Document> documents) {
        // 自定义批处理逻辑
        // 例如：按文档大小、元数据类型等分组
        return groupBy SomeLogic(documents);
    }
}
```

### 2. 可观测性（Observation）

```java
@Configuration
public class ObservableVectorStoreConfig {
    
    @Bean
    public VectorStore vectorStore(
            JdbcTemplate jdbcTemplate,
            EmbeddingModel embeddingModel,
            ObservationRegistry observationRegistry) {
        
        return PgVectorStore.builder(jdbcTemplate, embeddingModel)
            .observationRegistry(observationRegistry)
            .customObservationConvention(
                new CustomVectorStoreObservationConvention()
            )
            .build();
    }
}

// 监控指标
// - vectorstore.query.duration: 查询耗时
// - vectorstore.query.documents: 返回文档数
// - vectorstore.add.duration: 添加耗时
// - vectorstore.add.documents: 添加文档数
```

### 3. 多租户支持

```java
@Service
public class MultiTenantVectorStore {
    
    private final VectorStore vectorStore;
    
    /**
     * 添加文档（自动添加租户ID）
     */
    public void addDocument(String tenantId, Document document) {
        document.getMetadata().put("tenantId", tenantId);
        vectorStore.add(List.of(document));
    }
    
    /**
     * 搜索（自动过滤租户）
     */
    public List<Document> search(String tenantId, String query) {
        String filter = "tenantId == '" + tenantId + "'";
        
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .filterExpression(filter)
                .build()
        );
    }
    
    /**
     * 删除租户所有数据
     */
    public void deleteTenant(String tenantId) {
        String filter = "tenantId == '" + tenantId + "'";
        vectorStore.delete(filter);
    }
}
```

### 4. 混合搜索（向量+关键词）

```java
@Service
public class HybridSearchService {
    
    private final VectorStore vectorStore;
    
    /**
     * 混合搜索：向量相似度 + 关键词匹配
     */
    public List<Document> hybridSearch(
            String query,
            List<String> keywords) {
        
        // 1. 向量搜索
        List<Document> vectorResults = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(20)  // 多返回一些
                .similarityThreshold(0.6)
                .build()
        );
        
        // 2. 关键词过滤
        return vectorResults.stream()
            .filter(doc -> {
                String text = doc.getText().toLowerCase();
                return keywords.stream()
                    .anyMatch(keyword -> 
                        text.contains(keyword.toLowerCase())
                    );
            })
            .limit(5)  // 最终返回5个
            .toList();
    }
    
    /**
     * 重排序（Reranking）
     */
    public List<Document> rerankResults(
            String query,
            List<Document> documents) {
        
        // 使用更精确的模型重新排序
        // 例如：CrossEncoder模型
        return documents; // 简化示例
    }
}
```

---

## 最佳实践

### 1. 文档分块策略

```java
@Service
public class DocumentChunkingService {
    
    /**
     * 按字符数分块
     */
    public List<Document> chunkByCharacters(
            String text,
            int chunkSize,
            int chunkOverlap) {
        
        List<Document> chunks = new ArrayList<>();
        int start = 0;
        int docId = 1;
        
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            String chunk = text.substring(start, end);
            
            Document doc = Document.builder()
                .text(chunk)
                .metadata("chunkId", docId++)
                .metadata("chunkStart", start)
                .metadata("chunkEnd", end)
                .build();
            
            chunks.add(doc);
            
            start += chunkSize - chunkOverlap;
        }
        
        return chunks;
    }
    
    /**
     * 按段落分块
     */
    public List<Document> chunkByParagraphs(String text) {
        String[] paragraphs = text.split("\n\n+");
        
        return Arrays.stream(paragraphs)
            .filter(p -> !p.trim().isEmpty())
            .map(p -> new Document(p.trim()))
            .toList();
    }
    
    /**
     * 智能分块（保持语义完整）
     */
    public List<Document> semanticChunking(String text) {
        // 使用NLP技术分句
        // 保持句子完整性
        // 控制块大小在合理范围
        return List.of(); // 简化示例
    }
}
```

### 2. 元数据设计

```java
/**
 * 元数据最佳实践
 */
public class MetadataDesign {
    
    public Document createWellStructuredDocument(String content) {
        return Document.builder()
            .text(content)
            
            // 1. 来源追踪
            .metadata("source", "user_manual.pdf")
            .metadata("sourceType", "PDF")
            .metadata("page", 42)
            .metadata("section", "5.2")
            
            // 2. 时间信息
            .metadata("createdAt", Instant.now().toString())
            .metadata("updatedAt", Instant.now().toString())
            .metadata("publishedYear", 2024)
            
            // 3. 分类标签
            .metadata("category", "技术文档")
            .metadata("subcategory", "API参考")
            .metadata("tags", List.of("REST", "JSON", "API"))
            
            // 4. 访问控制
            .metadata("tenantId", "company-001")
            .metadata("department", "Engineering")
            .metadata("accessLevel", "internal")
            .metadata("owner", "user@example.com")
            
            // 5. 内容特征
            .metadata("language", "zh-CN")
            .metadata("wordCount", 250)
            .metadata("complexity", "intermediate")
            .metadata("version", "2.1")
            
            // 6. 业务相关
            .metadata("productId", "prod-123")
            .metadata("customerId", "cust-456")
            .metadata("priority", "high")
            
            .build();
    }
}
```

### 3. 性能优化

```java
@Service
public class VectorStoreOptimization {
    
    private final VectorStore vectorStore;
    private final EmbeddingModel embeddingModel;
    
    /**
     * 批量操作优化
     */
    public void efficientBatchAdd(List<Document> documents) {
        // 1. 批量生成embedding
        List<float[]> embeddings = embeddingModel.embed(
            documents.stream()
                .map(Document::getText)
                .toList()
        );
        
        // 2. 设置embedding
        for (int i = 0; i < documents.size(); i++) {
            documents.get(i).setEmbedding(embeddings.get(i));
        }
        
        // 3. 批量添加
        vectorStore.add(documents);
    }
    
    /**
     * 缓存搜索结果
     */
    @Cacheable(value = "vectorSearchCache", key = "#query")
    public List<Document> cachedSearch(String query) {
        return vectorStore.similaritySearch(query);
    }
    
    /**
     * 异步导入
     */
    @Async
    public CompletableFuture<Void> asyncIngest(
            List<Document> documents) {
        
        vectorStore.add(documents);
        return CompletableFuture.completedFuture(null);
    }
}
```

### 4. 错误处理

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
                logger.error("Failed to add documents, attempt: {}", 
                    context.getRetryCount(), e);
                throw e;
            }
        });
    }
    
    /**
     * 容错搜索
     */
    public List<Document> robustSearch(String query) {
        try {
            return vectorStore.similaritySearch(query);
        } catch (Exception e) {
            logger.error("Search failed", e);
            // 返回空结果或降级结果
            return List.of();
        }
    }
    
    /**
     * 部分失败处理
     */
    public void resilientBatchAdd(List<Document> documents) {
        List<Document> failed = new ArrayList<>();
        
        for (Document doc : documents) {
            try {
                vectorStore.add(List.of(doc));
            } catch (Exception e) {
                logger.error("Failed to add document: {}", 
                    doc.getId(), e);
                failed.add(doc);
            }
        }
        
        if (!failed.isEmpty()) {
            logger.warn("Failed to add {} documents", failed.size());
            // 重试或保存到DLQ
        }
    }
}
```

---

## 实战示例

### 1. 文档问答系统

```java
@Service
public class DocumentQASystem {
    
    private final VectorStore vectorStore;
    private final DocumentReader pdfReader;
    private final TextSplitter textSplitter;
    
    /**
     * 导入知识库
     */
    public void ingestKnowledgeBase(List<Resource> pdfs) {
        for (Resource pdf : pdfs) {
            // 1. 读取PDF
            List<Document> documents = pdfReader.read(pdf);
            
            // 2. 分块
            List<Document> chunks = documents.stream()
                .flatMap(doc -> textSplitter.split(doc).stream())
                .toList();
            
            // 3. 添加元数据
            chunks.forEach(chunk -> 
                chunk.getMetadata().put("source", pdf.getFilename())
            );
            
            // 4. 导入向量库
            vectorStore.add(chunks);
            
            logger.info("Ingested: {}", pdf.getFilename());
        }
    }
    
    /**
     * 回答问题
     */
    public Answer answerQuestion(String question) {
        // 1. 检索相关文档
        List<Document> relevantDocs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        );
        
        if (relevantDocs.isEmpty()) {
            return new Answer(
                "抱歉，我在知识库中没有找到相关信息。",
                List.of()
            );
        }
        
        // 2. 构建上下文
        String context = relevantDocs.stream()
            .map(Document::getText)
            .collect(Collectors.joining("\n\n"));
        
        // 3. 生成答案（使用ChatClient，见RAG文档）
        String answer = generateAnswer(question, context);
        
        // 4. 提取来源
        List<String> sources = relevantDocs.stream()
            .map(doc -> (String) doc.getMetadata().get("source"))
            .distinct()
            .toList();
        
        return new Answer(answer, sources);
    }
    
    private String generateAnswer(String question, String context) {
        // 使用ChatClient生成答案
        // 见RAG文档
        return "";
    }
    
    record Answer(String text, List<String> sources) {}
}
```

### 2. 语义搜索引擎

```java
@Service
public class SemanticSearchEngine {
    
    private final VectorStore vectorStore;
    
    /**
     * 导入产品目录
     */
    public void indexProducts(List<Product> products) {
        List<Document> documents = products.stream()
            .map(product -> Document.builder()
                .id(product.id())
                .text(product.name() + " " + product.description())
                .metadata("productId", product.id())
                .metadata("name", product.name())
                .metadata("category", product.category())
                .metadata("price", product.price())
                .metadata("inStock", product.inStock())
                .build()
            )
            .toList();
        
        vectorStore.add(documents);
    }
    
    /**
     * 语义搜索产品
     */
    public List<ProductResult> searchProducts(
            String query,
            String category,
            Double maxPrice,
            Boolean inStock) {
        
        // 构建过滤表达式
        List<String> filters = new ArrayList<>();
        
        if (category != null) {
            filters.add("category == '" + category + "'");
        }
        
        if (maxPrice != null) {
            filters.add("price <= " + maxPrice);
        }
        
        if (inStock != null) {
            filters.add("inStock == " + inStock);
        }
        
        String filterExpression = filters.isEmpty() ? 
            null : String.join(" && ", filters);
        
        // 执行搜索
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(10)
                .similarityThreshold(0.6)
                .filterExpression(filterExpression)
                .build()
        );
        
        // 转换结果
        return docs.stream()
            .map(doc -> new ProductResult(
                (String) doc.getMetadata().get("productId"),
                (String) doc.getMetadata().get("name"),
                doc.getText(),
                (Double) doc.getMetadata().get("price")
            ))
            .toList();
    }
    
    record Product(
        String id,
        String name,
        String description,
        String category,
        Double price,
        Boolean inStock
    ) {}
    
    record ProductResult(
        String id,
        String name,
        String description,
        Double price
    ) {}
}
```

### 3. 智能推荐系统

```java
@Service
public class RecommendationService {
    
    private final VectorStore vectorStore;
    
    /**
     * 基于内容的推荐
     */
    public List<Article> recommendSimilarArticles(
            String articleId,
            int count) {
        
        // 1. 获取当前文章
        List<Document> current = vectorStore.similaritySearch(
            SearchRequest.builder()
                .filterExpression("articleId == '" + articleId + "'")
                .topK(1)
                .build()
        );
        
        if (current.isEmpty()) {
            return List.of();
        }
        
        // 2. 查找相似文章
        Document currentDoc = current.get(0);
        List<Document> similar = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(currentDoc.getText())
                .topK(count + 1)  // +1因为会包含自己
                .similarityThreshold(0.5)
                .filterExpression("articleId != '" + articleId + "'")
                .build()
        );
        
        // 3. 转换结果
        return similar.stream()
            .map(doc -> new Article(
                (String) doc.getMetadata().get("articleId"),
                (String) doc.getMetadata().get("title"),
                doc.getText()
            ))
            .toList();
    }
    
    /**
     * 基于用户兴趣的推荐
     */
    public List<Article> recommendForUser(
            String userId,
            List<String> userInterests) {
        
        // 构建用户兴趣查询
        String query = String.join(" ", userInterests);
        
        // 搜索相关文章
        List<Document> docs = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(query)
                .topK(10)
                .similarityThreshold(0.6)
                .build()
        );
        
        return docs.stream()
            .map(doc -> new Article(
                (String) doc.getMetadata().get("articleId"),
                (String) doc.getMetadata().get("title"),
                doc.getText()
            ))
            .toList();
    }
    
    record Article(String id, String title, String content) {}
}
```

---

## 总结

### 核心要点

1. **向量数据库**基于相似性搜索，而非精确匹配
2. **VectorStore接口**统一了不同向量数据库的操作
3. **Document**是基本存储单元，包含文本、元数据和向量
4. **Embedding**将文本转换为向量表示
5. **SearchRequest**配置搜索参数（topK、相似度阈值、过滤器）
6. **Filter表达式**支持跨数据库的元数据过滤

### 选择向量数据库

| 需求 | 推荐 |
|------|------|
| 生产环境、稳定性 | **PgVector** |
| 开发测试、快速上手 | **Chroma**, **SimpleVectorStore** |
| 高性能、大规模 | **Pinecone**, **Milvus** |
| 实时应用 | **Redis** |
| 文档数据库用户 | **MongoDB Atlas** |

### 最佳实践清单

- ✅ 合理设计文档分块策略
- ✅ 添加丰富的元数据便于过滤
- ✅ 选择合适的相似度阈值
- ✅ 使用批处理提升性能
- ✅ 实现错误处理和重试
- ✅ 添加监控和日志
- ✅ 考虑多租户隔离
- ✅ 定期维护和清理数据

### 下一步

- 学习**RAG（检索增强生成）**模式
- 探索**混合搜索**技术
- 了解**向量索引**优化
- 实践**多模态**向量搜索

通过掌握VectorStore，你可以构建强大的语义搜索、智能推荐、文档问答等AI应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

