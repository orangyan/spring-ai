# Spring AI Embedding 向量嵌入详解

## 📋 目录
- [概述](#概述)
- [核心概念](#核心概念)
- [EmbeddingModel接口](#embeddingmodel接口)
- [核心数据结构](#核心数据结构)
- [OpenAI Embedding实现](#openai-embedding实现)
- [配置选项](#配置选项)
- [使用指南](#使用指南)
- [多模型支持](#多模型支持)
- [高级特性](#高级特性)
- [实战场景](#实战场景)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是Embedding（向量嵌入）？

**Embedding**是将文本、图像等数据转换为**高维向量**（数字数组）的过程，这些向量能够捕捉数据的**语义信息**。

```
文本 → Embedding模型 → 向量（float[]）

"Spring AI很强大" → [0.123, -0.456, 0.789, ...]
                    (1536维浮点数数组)
```

### 为什么需要Embedding？

1. ✅ **语义搜索**: 基于意义而非关键词匹配
2. ✅ **相似度计算**: 计算文本之间的语义相似度
3. ✅ **知识库构建**: RAG应用的基础
4. ✅ **推荐系统**: 基于向量相似度推荐
5. ✅ **聚类分析**: 将相似内容分组
6. ✅ **异常检测**: 识别语义异常的内容

### Embedding的特点

```
语义相似 → 向量接近
"苹果很甜" ≈ "这个水果真好吃"  (向量距离小)

语义不同 → 向量距离远
"苹果很甜" ≠ "今天下雨了"  (向量距离大)
```

### 应用场景

- 📚 **知识库检索**: 根据问题找到相关文档
- 🔍 **智能搜索**: 语义搜索而非关键词匹配
- 💬 **对话系统**: 找到相似的历史对话
- 📰 **内容推荐**: 推荐相似文章
- 🏷️ **文本分类**: 基于向量相似度分类
- 🔎 **去重检测**: 识别重复或相似内容

---

## 核心概念

### 1. 向量维度

不同模型生成不同维度的向量：

| 模型 | 维度 | 特点 |
|------|------|------|
| text-embedding-ada-002 | 1536 | OpenAI旧模型 |
| text-embedding-3-small | 512/1536 | 可调整维度，高效 |
| text-embedding-3-large | 256/1024/3072 | 最高质量 |
| cohere-embed-multilingual | 768 | 多语言支持 |
| bge-large-zh-v1.5 | 1024 | 中文优化 |

### 2. 相似度计算

常用的相似度计算方法：

#### 余弦相似度（Cosine Similarity）
```java
float cosineSimilarity(float[] vec1, float[] vec2) {
    float dot = 0.0f;
    float norm1 = 0.0f;
    float norm2 = 0.0f;
    
    for (int i = 0; i < vec1.length; i++) {
        dot += vec1[i] * vec2[i];
        norm1 += vec1[i] * vec1[i];
        norm2 += vec2[i] * vec2[i];
    }
    
    return dot / (Math.sqrt(norm1) * Math.sqrt(norm2));
}
```

#### 欧氏距离（Euclidean Distance）
```java
float euclideanDistance(float[] vec1, float[] vec2) {
    float sum = 0.0f;
    for (int i = 0; i < vec1.length; i++) {
        float diff = vec1[i] - vec2[i];
        sum += diff * diff;
    }
    return (float) Math.sqrt(sum);
}
```

### 3. Document类

`Document`是Spring AI中表示文档的核心类：

```java
/**
 * 文档 = 内容 + 元数据 + ID + 向量（可选）
 */
public class Document {
    
    private final String id;              // 唯一标识
    private final String text;            // 文本内容
    private final Media media;            // 媒体内容（可选）
    private final Map<String, Object> metadata;  // 元数据
    private final Double score;           // 相似度分数（可选）
    
    // 构造方法
    public Document(String text, Map<String, Object> metadata) {
        this(new RandomIdGenerator().generateId(), text, metadata);
    }
    
    public Document(String id, String text, Map<String, Object> metadata) {
        this.id = id;
        this.text = text;
        this.metadata = metadata;
        this.score = null;
    }
    
    // Getter方法
    public String getId() { return id; }
    public String getText() { return text; }
    public Map<String, Object> getMetadata() { return metadata; }
    public Double getScore() { return score; }
}
```

**示例：**
```java
// 创建文档
Document doc = new Document(
    "Spring AI是一个强大的AI应用框架",
    Map.of(
        "source", "documentation",
        "category", "技术",
        "author", "Spring团队"
    )
);

// 使用Builder
Document doc = Document.builder()
    .id("doc-001")
    .text("Spring AI支持多种AI模型")
    .metadata("source", "blog")
    .metadata("date", "2025-01-01")
    .build();
```

---

## EmbeddingModel接口

### 接口层次结构

```
Model<TReq, TRes> (根接口)
    ↑
    │
EmbeddingModel (向量嵌入接口)
    ↑
    │
OpenAiEmbeddingModel (OpenAI实现)
OllamaEmbeddingModel (Ollama实现)
...
```

### 接口定义

```java
/**
 * 向量嵌入模型接口
 */
public interface EmbeddingModel 
    extends Model<EmbeddingRequest, EmbeddingResponse> {
    
    /**
     * 核心方法：完整的嵌入请求
     * @param request 嵌入请求
     * @return 嵌入响应
     */
    @Override
    EmbeddingResponse call(EmbeddingRequest request);
    
    /**
     * 便捷方法1：嵌入单个文本
     * @param text 文本内容
     * @return 向量数组
     */
    default float[] embed(String text) {
        Assert.notNull(text, "Text must not be null");
        List<float[]> response = this.embed(List.of(text));
        return response.iterator().next();
    }
    
    /**
     * 便捷方法2：嵌入文档
     * @param document 文档对象
     * @return 向量数组
     */
    float[] embed(Document document);
    
    /**
     * 便捷方法3：批量嵌入文本
     * @param texts 文本列表
     * @return 向量列表
     */
    default List<float[]> embed(List<String> texts) {
        Assert.notNull(texts, "Texts must not be null");
        return this.call(
            new EmbeddingRequest(
                texts, 
                EmbeddingOptionsBuilder.builder().build()
            )
        )
        .getResults()
        .stream()
        .map(Embedding::getOutput)
        .toList();
    }
    
    /**
     * 便捷方法4：批量嵌入文档（带分批策略）
     * @param documents 文档列表
     * @param options 嵌入选项
     * @param batchingStrategy 分批策略
     * @return 向量列表
     */
    default List<float[]> embed(
            List<Document> documents,
            EmbeddingOptions options,
            BatchingStrategy batchingStrategy) {
        
        Assert.notNull(documents, "Documents must not be null");
        
        List<float[]> embeddings = new ArrayList<>(documents.size());
        
        // 按策略分批
        List<List<Document>> batch = batchingStrategy.batch(documents);
        
        for (List<Document> subBatch : batch) {
            // 提取文本
            List<String> texts = subBatch.stream()
                .map(Document::getText)
                .toList();
            
            // 批量嵌入
            EmbeddingRequest request = new EmbeddingRequest(texts, options);
            EmbeddingResponse response = this.call(request);
            
            // 收集结果
            for (int i = 0; i < subBatch.size(); i++) {
                embeddings.add(response.getResults().get(i).getOutput());
            }
        }
        
        return embeddings;
    }
    
    /**
     * 便捷方法5：嵌入并返回完整响应
     * @param texts 文本列表
     * @return 嵌入响应
     */
    default EmbeddingResponse embedForResponse(List<String> texts) {
        Assert.notNull(texts, "Texts must not be null");
        return this.call(
            new EmbeddingRequest(
                texts, 
                EmbeddingOptionsBuilder.builder().build()
            )
        );
    }
    
    /**
     * 获取向量维度
     * @return 维度数
     */
    default int dimensions() {
        return embed("Test String").length;
    }
}
```

### 方法对比

| 方法 | 输入 | 输出 | 适用场景 |
|------|------|------|----------|
| `embed(String)` | 单个文本 | `float[]` | 简单嵌入 |
| `embed(Document)` | 单个文档 | `float[]` | 文档处理 |
| `embed(List<String>)` | 文本列表 | `List<float[]>` | 批量文本 |
| `embed(List<Document>, ...)` | 文档列表+策略 | `List<float[]>` | 大批量处理 |
| `embedForResponse(List)` | 文本列表 | `EmbeddingResponse` | 需要元数据 |
| `call(EmbeddingRequest)` | 完整请求 | `EmbeddingResponse` | 完全控制 |

---

## 核心数据结构

### 1. EmbeddingRequest（嵌入请求）

```java
/**
 * 嵌入请求
 */
public class EmbeddingRequest implements ModelRequest<List<String>> {
    
    /**
     * 要嵌入的文本列表
     */
    private final List<String> inputs;
    
    /**
     * 嵌入选项
     */
    private final EmbeddingOptions options;
    
    // 构造方法
    public EmbeddingRequest(List<String> inputs) {
        this(inputs, EmbeddingOptionsBuilder.builder().build());
    }
    
    public EmbeddingRequest(
            List<String> inputs,
            EmbeddingOptions options) {
        this.inputs = inputs;
        this.options = options;
    }
    
    @Override
    public List<String> getInstructions() {
        return this.inputs;
    }
    
    @Override
    public EmbeddingOptions getOptions() {
        return this.options;
    }
}
```

**使用示例：**
```java
// 简单请求
EmbeddingRequest request = new EmbeddingRequest(
    List.of("文本1", "文本2", "文本3")
);

// 带选项的请求
EmbeddingOptions options = EmbeddingOptionsBuilder.builder()
    .model("text-embedding-3-small")
    .dimensions(512)
    .build();

EmbeddingRequest request = new EmbeddingRequest(
    List.of("文本1", "文本2"),
    options
);
```

### 2. EmbeddingResponse（嵌入响应）

```java
/**
 * 嵌入响应
 */
public class EmbeddingResponse 
    implements ModelResponse<Embedding> {
    
    /**
     * 嵌入结果列表
     */
    private final List<Embedding> embeddings;
    
    /**
     * 响应元数据
     */
    private final EmbeddingResponseMetadata metadata;
    
    // 构造方法
    public EmbeddingResponse(List<Embedding> embeddings) {
        this(embeddings, new EmbeddingResponseMetadata());
    }
    
    public EmbeddingResponse(
            List<Embedding> embeddings,
            EmbeddingResponseMetadata metadata) {
        this.embeddings = embeddings;
        this.metadata = metadata;
    }
    
    /**
     * 获取第一个嵌入结果
     */
    @Override
    public Embedding getResult() {
        Assert.notEmpty(embeddings, "No embedding data available.");
        return embeddings.get(0);
    }
    
    /**
     * 获取所有嵌入结果
     */
    @Override
    public List<Embedding> getResults() {
        return embeddings;
    }
    
    /**
     * 获取元数据
     */
    public EmbeddingResponseMetadata getMetadata() {
        return metadata;
    }
}
```

### 3. Embedding（单个嵌入结果）

```java
/**
 * 单个嵌入向量
 */
public class Embedding implements ModelResult<float[]> {
    
    /**
     * 向量数据
     */
    private final float[] embedding;
    
    /**
     * 索引位置
     */
    private final Integer index;
    
    /**
     * 元数据
     */
    private final EmbeddingResultMetadata metadata;
    
    // 构造方法
    public Embedding(float[] embedding, Integer index) {
        this(embedding, index, EmbeddingResultMetadata.EMPTY);
    }
    
    public Embedding(
            float[] embedding,
            Integer index,
            EmbeddingResultMetadata metadata) {
        this.embedding = embedding;
        this.index = index;
        this.metadata = metadata;
    }
    
    /**
     * 获取向量数组
     */
    @Override
    public float[] getOutput() {
        return embedding;
    }
    
    /**
     * 获取索引
     */
    public Integer getIndex() {
        return index;
    }
    
    /**
     * 获取元数据
     */
    public EmbeddingResultMetadata getMetadata() {
        return metadata;
    }
}
```

### 4. EmbeddingResponseMetadata（响应元数据）

```java
/**
 * 嵌入响应元数据
 */
public class EmbeddingResponseMetadata {
    
    /**
     * 使用的模型名称
     */
    private final String model;
    
    /**
     * Token使用情况
     */
    private final Usage usage;
    
    public EmbeddingResponseMetadata(String model, Usage usage) {
        this.model = model;
        this.usage = usage;
    }
    
    public String getModel() {
        return model;
    }
    
    public Usage getUsage() {
        return usage;
    }
}
```

---

## OpenAI Embedding实现

### OpenAiEmbeddingModel

```java
/**
 * OpenAI嵌入模型实现
 */
public class OpenAiEmbeddingModel extends AbstractEmbeddingModel {
    
    private final OpenAiEmbeddingOptions defaultOptions;
    private final RetryTemplate retryTemplate;
    private final OpenAiApi openAiApi;
    private final MetadataMode metadataMode;
    private final ObservationRegistry observationRegistry;
    
    /**
     * 默认构造
     */
    public OpenAiEmbeddingModel(OpenAiApi openAiApi) {
        this(openAiApi, MetadataMode.EMBED);
    }
    
    /**
     * 带元数据模式的构造
     */
    public OpenAiEmbeddingModel(
            OpenAiApi openAiApi,
            MetadataMode metadataMode) {
        this(
            openAiApi,
            metadataMode,
            OpenAiEmbeddingOptions.builder()
                .model(OpenAiApi.DEFAULT_EMBEDDING_MODEL)
                .build()
        );
    }
    
    /**
     * 完整构造
     */
    public OpenAiEmbeddingModel(
            OpenAiApi openAiApi,
            MetadataMode metadataMode,
            OpenAiEmbeddingOptions options,
            RetryTemplate retryTemplate,
            ObservationRegistry observationRegistry) {
        
        Assert.notNull(openAiApi, "openAiApi must not be null");
        Assert.notNull(metadataMode, "metadataMode must not be null");
        Assert.notNull(options, "options must not be null");
        
        this.openAiApi = openAiApi;
        this.metadataMode = metadataMode;
        this.defaultOptions = options;
        this.retryTemplate = retryTemplate;
        this.observationRegistry = observationRegistry;
    }
    
    /**
     * 嵌入文档
     */
    @Override
    public float[] embed(Document document) {
        Assert.notNull(document, "Document must not be null");
        // 根据元数据模式格式化文档内容
        return this.embed(
            document.getFormattedContent(this.metadataMode)
        );
    }
    
    /**
     * 核心调用方法
     */
    @Override
    public EmbeddingResponse call(EmbeddingRequest request) {
        
        // 1. 合并请求选项和默认选项
        EmbeddingRequest embeddingRequest = 
            buildEmbeddingRequest(request);
        
        // 2. 创建API请求
        OpenAiApi.EmbeddingRequest<List<String>> apiRequest = 
            createRequest(embeddingRequest);
        
        // 3. 创建观测上下文
        var observationContext = EmbeddingModelObservationContext.builder()
            .embeddingRequest(embeddingRequest)
            .provider(OpenAiApiConstants.PROVIDER_NAME)
            .build();
        
        // 4. 执行API调用（带观测）
        return EmbeddingModelObservationDocumentation
            .EMBEDDING_MODEL_OPERATION
            .observation(
                observationConvention,
                DEFAULT_OBSERVATION_CONVENTION,
                () -> observationContext,
                observationRegistry
            )
            .observe(() -> {
                // 4.1 带重试的API调用
                EmbeddingList<OpenAiApi.Embedding> apiResponse = 
                    retryTemplate.execute(ctx -> 
                        openAiApi.embeddings(apiRequest).getBody()
                    );
                
                if (apiResponse == null) {
                    logger.warn("No embeddings returned for request: {}", request);
                    return new EmbeddingResponse(List.of());
                }
                
                // 4.2 提取Usage信息
                OpenAiApi.Usage usage = apiResponse.usage();
                Usage embeddingResponseUsage = usage != null ? 
                    getDefaultUsage(usage) : 
                    new EmptyUsage();
                
                // 4.3 构建元数据
                var metadata = new EmbeddingResponseMetadata(
                    apiResponse.model(),
                    embeddingResponseUsage
                );
                
                // 4.4 转换嵌入结果
                List<Embedding> embeddings = apiResponse.data()
                    .stream()
                    .map(e -> new Embedding(e.embedding(), e.index()))
                    .toList();
                
                // 4.5 构建响应
                EmbeddingResponse embeddingResponse = 
                    new EmbeddingResponse(embeddings, metadata);
                
                // 4.6 更新观测上下文
                observationContext.setResponse(embeddingResponse);
                
                return embeddingResponse;
            });
    }
    
    /**
     * 创建API请求
     */
    private OpenAiApi.EmbeddingRequest<List<String>> createRequest(
            EmbeddingRequest embeddingRequest) {
        
        return new OpenAiApi.EmbeddingRequest<>(
            embeddingRequest.getInstructions(),
            ((OpenAiEmbeddingOptions) embeddingRequest.getOptions())
                .getModel(),
            // ... 其他选项
        );
    }
    
    /**
     * 合并请求和默认选项
     */
    private EmbeddingRequest buildEmbeddingRequest(
            EmbeddingRequest request) {
        
        if (request.getOptions() != null) {
            // 使用请求级选项
            return request;
        }
        
        // 使用默认选项
        return new EmbeddingRequest(
            request.getInstructions(),
            defaultOptions
        );
    }
}
```

### OpenAI支持的模型

```java
/**
 * OpenAI嵌入模型
 */
public enum OpenAiEmbeddingModel {
    
    /**
     * text-embedding-ada-002
     * 维度：1536
     * 旧模型，不推荐
     */
    TEXT_EMBEDDING_ADA_002("text-embedding-ada-002"),
    
    /**
     * text-embedding-3-small
     * 维度：可调整（默认1536，可降至512）
     * 性价比高，推荐
     */
    TEXT_EMBEDDING_3_SMALL("text-embedding-3-small"),
    
    /**
     * text-embedding-3-large
     * 维度：可调整（默认3072，可降至256）
     * 质量最高
     */
    TEXT_EMBEDDING_3_LARGE("text-embedding-3-large");
    
    private final String value;
    
    OpenAiEmbeddingModel(String value) {
        this.value = value;
    }
    
    public String getValue() {
        return value;
    }
}
```

---

## 配置选项

### EmbeddingOptions接口

```java
/**
 * 嵌入选项接口
 */
public interface EmbeddingOptions extends ModelOptions {
    
    /**
     * 获取模型名称
     */
    @Nullable
    String getModel();
    
    /**
     * 获取向量维度
     */
    @Nullable
    Integer getDimensions();
}
```

### OpenAiEmbeddingOptions

```java
/**
 * OpenAI嵌入选项
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class OpenAiEmbeddingOptions implements EmbeddingOptions {
    
    /**
     * 模型ID
     */
    @JsonProperty("model")
    private String model;
    
    /**
     * 编码格式
     * 可选：float, base64
     */
    @JsonProperty("encoding_format")
    private String encodingFormat;
    
    /**
     * 向量维度
     * 仅text-embedding-3及以后支持
     */
    @JsonProperty("dimensions")
    private Integer dimensions;
    
    /**
     * 用户标识
     * 用于监控和滥用检测
     */
    @JsonProperty("user")
    private String user;
    
    // Getters and Setters
    
    @Override
    public String getModel() {
        return model;
    }
    
    public void setModel(String model) {
        this.model = model;
    }
    
    @Override
    public Integer getDimensions() {
        return dimensions;
    }
    
    public void setDimensions(Integer dimensions) {
        this.dimensions = dimensions;
    }
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        
        private String model;
        private String encodingFormat;
        private Integer dimensions;
        private String user;
        
        public Builder model(String model) {
            this.model = model;
            return this;
        }
        
        public Builder encodingFormat(String encodingFormat) {
            this.encodingFormat = encodingFormat;
            return this;
        }
        
        public Builder dimensions(Integer dimensions) {
            this.dimensions = dimensions;
            return this;
        }
        
        public Builder user(String user) {
            this.user = user;
            return this;
        }
        
        public OpenAiEmbeddingOptions build() {
            OpenAiEmbeddingOptions options = 
                new OpenAiEmbeddingOptions();
            options.setModel(model);
            options.setEncodingFormat(encodingFormat);
            options.setDimensions(dimensions);
            options.setUser(user);
            return options;
        }
    }
}
```

---

## 使用指南

### 1. 添加依赖

```xml
<!-- OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- 或 Ollama -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      embedding:
        options:
          model: text-embedding-3-small
          dimensions: 1536
```

### 3. 基本使用

```java
@Service
public class EmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    public EmbeddingService(EmbeddingModel embeddingModel) {
        this.embeddingModel = embeddingModel;
    }
    
    /**
     * 最简单的嵌入
     */
    public float[] embedText(String text) {
        return embeddingModel.embed(text);
    }
    
    /**
     * 批量嵌入
     */
    public List<float[]> embedBatch(List<String> texts) {
        return embeddingModel.embed(texts);
    }
    
    /**
     * 嵌入文档
     */
    public float[] embedDocument(Document document) {
        return embeddingModel.embed(document);
    }
    
    /**
     * 带配置的嵌入
     */
    public float[] embedWithOptions(String text) {
        EmbeddingOptions options = 
            EmbeddingOptionsBuilder.builder()
                .model("text-embedding-3-small")
                .dimensions(512)  // 降低维度
                .build();
        
        EmbeddingRequest request = 
            new EmbeddingRequest(List.of(text), options);
        
        EmbeddingResponse response = embeddingModel.call(request);
        
        return response.getResult().getOutput();
    }
    
    /**
     * 获取完整响应
     */
    public EmbeddingResponse embedForResponse(List<String> texts) {
        return embeddingModel.embedForResponse(texts);
    }
}
```

### 4. REST API示例

```java
@RestController
@RequestMapping("/api/embedding")
public class EmbeddingController {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 嵌入单个文本
     */
    @PostMapping("/embed")
    public ResponseEntity<EmbeddingResult> embedText(
            @RequestBody EmbeddingRequest request) {
        
        try {
            float[] vector = embeddingModel.embed(request.text());
            
            return ResponseEntity.ok(
                new EmbeddingResult(
                    vector,
                    vector.length,
                    "success"
                )
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new EmbeddingResult(
                    null,
                    0,
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    /**
     * 批量嵌入
     */
    @PostMapping("/embed-batch")
    public ResponseEntity<BatchEmbeddingResult> embedBatch(
            @RequestBody BatchEmbeddingRequest request) {
        
        try {
            List<float[]> vectors = 
                embeddingModel.embed(request.texts());
            
            return ResponseEntity.ok(
                new BatchEmbeddingResult(
                    vectors,
                    vectors.size(),
                    vectors.get(0).length,
                    "success"
                )
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new BatchEmbeddingResult(
                    null,
                    0,
                    0,
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    /**
     * 计算文本相似度
     */
    @PostMapping("/similarity")
    public ResponseEntity<SimilarityResult> calculateSimilarity(
            @RequestBody SimilarityRequest request) {
        
        try {
            float[] vec1 = embeddingModel.embed(request.text1());
            float[] vec2 = embeddingModel.embed(request.text2());
            
            float similarity = cosineSimilarity(vec1, vec2);
            
            return ResponseEntity.ok(
                new SimilarityResult(
                    request.text1(),
                    request.text2(),
                    similarity,
                    "success"
                )
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new SimilarityResult(
                    request.text1(),
                    request.text2(),
                    0.0f,
                    "Error: " + e.getMessage()
                ));
        }
    }
    
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
    
    record EmbeddingRequest(String text) {}
    
    record EmbeddingResult(
        float[] vector,
        int dimensions,
        String status
    ) {}
    
    record BatchEmbeddingRequest(List<String> texts) {}
    
    record BatchEmbeddingResult(
        List<float[]> vectors,
        int count,
        int dimensions,
        String status
    ) {}
    
    record SimilarityRequest(String text1, String text2) {}
    
    record SimilarityResult(
        String text1,
        String text2,
        float similarity,
        String status
    ) {}
}
```

---

## 多模型支持

### 1. OpenAI

```java
@Configuration
public class OpenAiEmbeddingConfig {
    
    @Bean
    public EmbeddingModel openAiEmbeddingModel(OpenAiApi openAiApi) {
        
        OpenAiEmbeddingOptions options = 
            OpenAiEmbeddingOptions.builder()
                .model("text-embedding-3-small")
                .dimensions(1536)
                .build();
        
        return new OpenAiEmbeddingModel(
            openAiApi,
            MetadataMode.EMBED,
            options
        );
    }
}
```

### 2. Ollama

```java
@Configuration
public class OllamaEmbeddingConfig {
    
    @Bean
    public EmbeddingModel ollamaEmbeddingModel(OllamaApi ollamaApi) {
        
        OllamaEmbeddingOptions options = 
            OllamaEmbeddingOptions.builder()
                .model("nomic-embed-text")
                .build();
        
        return new OllamaEmbeddingModel(ollamaApi, options);
    }
}
```

### 3. Azure OpenAI

```java
@Configuration
public class AzureOpenAiEmbeddingConfig {
    
    @Bean
    public EmbeddingModel azureEmbeddingModel(
            AzureOpenAiApi azureOpenAiApi) {
        
        return new AzureOpenAiEmbeddingModel(
            azureOpenAiApi,
            MetadataMode.EMBED,
            AzureOpenAiEmbeddingOptions.builder()
                .deploymentName("text-embedding-ada-002")
                .build()
        );
    }
}
```

### 4. Google (Vertex AI)

```java
@Configuration
public class VertexAiEmbeddingConfig {
    
    @Bean
    public EmbeddingModel vertexAiEmbeddingModel(
            VertexAiApi vertexAiApi) {
        
        return new VertexAiEmbeddingModel(vertexAiApi);
    }
}
```

---

## 高级特性

### 1. 批处理策略

```java
@Service
public class BatchEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 使用批处理策略嵌入大量文档
     */
    public List<float[]> embedLargeDocumentBatch(
            List<Document> documents) {
        
        // 创建批处理策略（每批100个文档）
        BatchingStrategy strategy = new TokenCountBatchingStrategy(
            OpenAiApi.DEFAULT_EMBEDDING_MODEL,
            100
        );
        
        EmbeddingOptions options = 
            EmbeddingOptionsBuilder.builder()
                .model("text-embedding-3-small")
                .build();
        
        return embeddingModel.embed(documents, options, strategy);
    }
}
```

### 2. 缓存嵌入结果

```java
@Service
public class CachedEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 缓存嵌入结果
     */
    @Cacheable(value = "embeddings", key = "#text")
    public float[] embedWithCache(String text) {
        return embeddingModel.embed(text);
    }
    
    /**
     * 清除缓存
     */
    @CacheEvict(value = "embeddings", allEntries = true)
    public void clearCache() {
        // 缓存会自动清除
    }
}
```

### 3. 异步嵌入

```java
@Service
public class AsyncEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 异步嵌入
     */
    @Async
    public CompletableFuture<float[]> embedAsync(String text) {
        return CompletableFuture.supplyAsync(() -> 
            embeddingModel.embed(text)
        );
    }
    
    /**
     * 并行批量嵌入
     */
    public List<float[]> embedParallel(List<String> texts) {
        return texts.parallelStream()
            .map(text -> embeddingModel.embed(text))
            .toList();
    }
}
```

### 4. 元数据模式

```java
@Service
public class MetadataAwareEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 使用不同的元数据模式
     */
    public float[] embedWithMetadata(
            Document document,
            MetadataMode mode) {
        
        // MetadataMode.EMBED: 包含元数据
        // MetadataMode.NONE: 仅内容
        
        String content = document.getFormattedContent(mode);
        
        return embeddingModel.embed(content);
    }
}
```

---

## 实战场景

### 1. 文档相似度搜索

```java
@Service
public class DocumentSimilarityService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 找到最相似的文档
     */
    public List<ScoredDocument> findSimilarDocuments(
            String query,
            List<Document> documents,
            int topK) {
        
        // 1. 嵌入查询
        float[] queryVector = embeddingModel.embed(query);
        
        // 2. 嵌入所有文档
        List<String> documentTexts = documents.stream()
            .map(Document::getText)
            .toList();
        
        List<float[]> documentVectors = 
            embeddingModel.embed(documentTexts);
        
        // 3. 计算相似度
        List<ScoredDocument> scoredDocs = new ArrayList<>();
        for (int i = 0; i < documents.size(); i++) {
            float similarity = cosineSimilarity(
                queryVector,
                documentVectors.get(i)
            );
            
            scoredDocs.add(new ScoredDocument(
                documents.get(i),
                similarity
            ));
        }
        
        // 4. 排序并返回Top K
        return scoredDocs.stream()
            .sorted((a, b) -> Float.compare(b.score(), a.score()))
            .limit(topK)
            .toList();
    }
    
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
    
    record ScoredDocument(Document document, float score) {}
}
```

### 2. 文本聚类

```java
@Service
public class TextClusteringService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 使用K-Means聚类
     */
    public Map<Integer, List<String>> clusterTexts(
            List<String> texts,
            int numClusters) {
        
        // 1. 嵌入所有文本
        List<float[]> vectors = embeddingModel.embed(texts);
        
        // 2. K-Means聚类
        List<Integer> assignments = kMeansClustering(
            vectors,
            numClusters
        );
        
        // 3. 按簇分组
        Map<Integer, List<String>> clusters = new HashMap<>();
        for (int i = 0; i < texts.size(); i++) {
            int clusterId = assignments.get(i);
            clusters.computeIfAbsent(clusterId, k -> new ArrayList<>())
                .add(texts.get(i));
        }
        
        return clusters;
    }
    
    private List<Integer> kMeansClustering(
            List<float[]> vectors,
            int k) {
        // K-Means实现（简化）
        // 实际应用中可以使用Apache Commons Math等库
        
        int dimensions = vectors.get(0).length;
        List<float[]> centroids = initializeCentroids(vectors, k);
        
        List<Integer> assignments = new ArrayList<>(vectors.size());
        boolean changed = true;
        
        while (changed) {
            changed = false;
            assignments.clear();
            
            // 分配到最近的质心
            for (float[] vector : vectors) {
                int nearestCentroid = findNearestCentroid(
                    vector,
                    centroids
                );
                assignments.add(nearestCentroid);
            }
            
            // 更新质心
            List<float[]> newCentroids = 
                updateCentroids(vectors, assignments, k, dimensions);
            
            if (!centroidsEqual(centroids, newCentroids)) {
                centroids = newCentroids;
                changed = true;
            }
        }
        
        return assignments;
    }
    
    // 辅助方法（简化实现）
    private List<float[]> initializeCentroids(
            List<float[]> vectors,
            int k) {
        // 随机选择k个向量作为初始质心
        List<float[]> centroids = new ArrayList<>();
        for (int i = 0; i < k; i++) {
            centroids.add(vectors.get(i).clone());
        }
        return centroids;
    }
    
    private int findNearestCentroid(
            float[] vector,
            List<float[]> centroids) {
        
        int nearest = 0;
        float minDistance = Float.MAX_VALUE;
        
        for (int i = 0; i < centroids.size(); i++) {
            float distance = euclideanDistance(vector, centroids.get(i));
            if (distance < minDistance) {
                minDistance = distance;
                nearest = i;
            }
        }
        
        return nearest;
    }
    
    private List<float[]> updateCentroids(
            List<float[]> vectors,
            List<Integer> assignments,
            int k,
            int dimensions) {
        
        List<float[]> newCentroids = new ArrayList<>();
        
        for (int i = 0; i < k; i++) {
            float[] centroid = new float[dimensions];
            int count = 0;
            
            for (int j = 0; j < vectors.size(); j++) {
                if (assignments.get(j) == i) {
                    float[] vector = vectors.get(j);
                    for (int d = 0; d < dimensions; d++) {
                        centroid[d] += vector[d];
                    }
                    count++;
                }
            }
            
            if (count > 0) {
                for (int d = 0; d < dimensions; d++) {
                    centroid[d] /= count;
                }
            }
            
            newCentroids.add(centroid);
        }
        
        return newCentroids;
    }
    
    private boolean centroidsEqual(
            List<float[]> c1,
            List<float[]> c2) {
        
        for (int i = 0; i < c1.size(); i++) {
            if (!Arrays.equals(c1.get(i), c2.get(i))) {
                return false;
            }
        }
        return true;
    }
    
    private float euclideanDistance(float[] vec1, float[] vec2) {
        float sum = 0.0f;
        for (int i = 0; i < vec1.length; i++) {
            float diff = vec1[i] - vec2[i];
            sum += diff * diff;
        }
        return (float) Math.sqrt(sum);
    }
}
```

### 3. 去重检测

```java
@Service
public class DuplicationDetectionService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 检测重复文档
     */
    public List<DuplicateGroup> findDuplicates(
            List<String> texts,
            float threshold) {
        
        // 1. 嵌入所有文本
        List<float[]> vectors = embeddingModel.embed(texts);
        
        // 2. 找到重复组
        List<DuplicateGroup> duplicates = new ArrayList<>();
        boolean[] processed = new boolean[texts.size()];
        
        for (int i = 0; i < texts.size(); i++) {
            if (processed[i]) {
                continue;
            }
            
            List<Integer> group = new ArrayList<>();
            group.add(i);
            
            for (int j = i + 1; j < texts.size(); j++) {
                if (processed[j]) {
                    continue;
                }
                
                float similarity = cosineSimilarity(
                    vectors.get(i),
                    vectors.get(j)
                );
                
                if (similarity >= threshold) {
                    group.add(j);
                    processed[j] = true;
                }
            }
            
            if (group.size() > 1) {
                List<String> duplicateTexts = group.stream()
                    .map(texts::get)
                    .toList();
                
                duplicates.add(new DuplicateGroup(duplicateTexts));
            }
            
            processed[i] = true;
        }
        
        return duplicates;
    }
    
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
    
    record DuplicateGroup(List<String> texts) {}
}
```

### 4. 语义标签推荐

```java
@Service
public class SemanticTagRecommendationService {
    
    private final EmbeddingModel embeddingModel;
    
    private final Map<String, float[]> tagVectors;
    
    public SemanticTagRecommendationService(
            EmbeddingModel embeddingModel,
            List<String> predefinedTags) {
        
        this.embeddingModel = embeddingModel;
        
        // 预先嵌入所有标签
        this.tagVectors = new HashMap<>();
        List<float[]> vectors = embeddingModel.embed(predefinedTags);
        
        for (int i = 0; i < predefinedTags.size(); i++) {
            tagVectors.put(predefinedTags.get(i), vectors.get(i));
        }
    }
    
    /**
     * 为文本推荐标签
     */
    public List<ScoredTag> recommendTags(
            String text,
            int topK) {
        
        // 1. 嵌入文本
        float[] textVector = embeddingModel.embed(text);
        
        // 2. 计算与所有标签的相似度
        List<ScoredTag> scoredTags = tagVectors.entrySet()
            .stream()
            .map(entry -> {
                float similarity = cosineSimilarity(
                    textVector,
                    entry.getValue()
                );
                return new ScoredTag(entry.getKey(), similarity);
            })
            .toList();
        
        // 3. 排序并返回Top K
        return scoredTags.stream()
            .sorted((a, b) -> Float.compare(b.score(), a.score()))
            .limit(topK)
            .toList();
    }
    
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
    
    record ScoredTag(String tag, float score) {}
}
```

---

## 最佳实践

### 1. 模型选择

```java
@Configuration
public class EmbeddingModelSelectionConfig {
    
    /**
     * 根据场景选择模型
     */
    @Bean
    @ConditionalOnProperty(name = "embedding.scenario", havingValue = "cost-effective")
    public EmbeddingModel costEffectiveModel(OpenAiApi openAiApi) {
        // 性价比场景：text-embedding-3-small + 降维
        return new OpenAiEmbeddingModel(
            openAiApi,
            MetadataMode.EMBED,
            OpenAiEmbeddingOptions.builder()
                .model("text-embedding-3-small")
                .dimensions(512)  // 降低维度
                .build()
        );
    }
    
    @Bean
    @ConditionalOnProperty(name = "embedding.scenario", havingValue = "high-quality")
    public EmbeddingModel highQualityModel(OpenAiApi openAiApi) {
        // 高质量场景：text-embedding-3-large
        return new OpenAiEmbeddingModel(
            openAiApi,
            MetadataMode.EMBED,
            OpenAiEmbeddingOptions.builder()
                .model("text-embedding-3-large")
                .dimensions(3072)  // 最高维度
                .build()
        );
    }
    
    @Bean
    @ConditionalOnProperty(name = "embedding.scenario", havingValue = "offline")
    public EmbeddingModel offlineModel(OllamaApi ollamaApi) {
        // 离线场景：Ollama本地模型
        return new OllamaEmbeddingModel(
            ollamaApi,
            OllamaEmbeddingOptions.builder()
                .model("nomic-embed-text")
                .build()
        );
    }
}
```

### 2. 错误处理

```java
@Service
public class RobustEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    private final RetryTemplate retryTemplate;
    
    /**
     * 带重试的嵌入
     */
    public float[] embedWithRetry(String text) {
        return retryTemplate.execute(context -> {
            try {
                return embeddingModel.embed(text);
            } catch (Exception e) {
                logger.error("Embedding failed, attempt: {}", 
                    context.getRetryCount(), e);
                throw e;
            }
        });
    }
    
    /**
     * 带降级的嵌入
     */
    public float[] embedWithFallback(String text) {
        try {
            return embeddingModel.embed(text);
        } catch (Exception e) {
            logger.error("Embedding failed, using zero vector", e);
            // 返回零向量作为降级
            return new float[embeddingModel.dimensions()];
        }
    }
}
```

### 3. 批处理优化

```java
@Service
public class OptimizedBatchEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    
    /**
     * 智能批处理
     */
    public List<float[]> embedBatchOptimized(List<String> texts) {
        
        if (texts.size() <= 10) {
            // 小批量：直接处理
            return embeddingModel.embed(texts);
        } else {
            // 大批量：分批处理
            List<float[]> allEmbeddings = new ArrayList<>();
            
            int batchSize = 100;
            for (int i = 0; i < texts.size(); i += batchSize) {
                int end = Math.min(i + batchSize, texts.size());
                List<String> batch = texts.subList(i, end);
                
                List<float[]> batchEmbeddings = 
                    embeddingModel.embed(batch);
                
                allEmbeddings.addAll(batchEmbeddings);
            }
            
            return allEmbeddings;
        }
    }
}
```

### 4. 性能监控

```java
@Service
public class MonitoredEmbeddingService {
    
    private final EmbeddingModel embeddingModel;
    private final MeterRegistry meterRegistry;
    
    /**
     * 带监控的嵌入
     */
    public float[] embedWithMetrics(String text) {
        
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            float[] result = embeddingModel.embed(text);
            
            // 记录成功
            sample.stop(Timer.builder("embedding.call")
                .tag("status", "success")
                .tag("model", "openai")
                .register(meterRegistry));
            
            return result;
            
        } catch (Exception e) {
            // 记录失败
            sample.stop(Timer.builder("embedding.call")
                .tag("status", "failure")
                .tag("model", "openai")
                .register(meterRegistry));
            
            throw e;
        }
    }
}
```

---

## 总结

### Embedding核心特点

1. **语义理解**: 捕捉文本的语义信息
2. **向量表示**: 将文本转换为数值向量
3. **相似度计算**: 支持语义相似度搜索
4. **多模型支持**: OpenAI、Ollama、Azure等
5. **批处理**: 高效处理大量文本

### Spring AI Embedding API

```
EmbeddingModel (接口)
    ↓
embed(String) → float[]
    ↓
embed(List<String>) → List<float[]>
    ↓
embed(Document) → float[]
    ↓
call(EmbeddingRequest) → EmbeddingResponse
```

### 核心配置

- **model**: 选择合适的模型
- **dimensions**: 调整向量维度（性能vs质量）
- **batchSize**: 批处理大小
- **metadataMode**: 是否包含元数据

### 最佳实践清单

- ✅ 根据场景选择合适的模型
- ✅ 使用批处理提高效率
- ✅ 缓存常用的嵌入结果
- ✅ 实现错误处理和重试
- ✅ 监控性能和成本
- ✅ 选择合适的向量维度
- ✅ 异步处理大批量嵌入

通过Spring AI的Embedding功能，你可以轻松构建语义搜索、推荐系统、文本聚类等强大的AI应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-05  
**Spring AI版本**: 1.1.0-SNAPSHOT

