# Spring AI Whisper 语音转文本详解

## 📋 目录
- [概述](#概述)
- [核心接口](#核心接口)
- [TranscriptionModel详解](#transcriptionmodel详解)
- [配置选项](#配置选项)
- [OpenAI Whisper实现](#openai-whisper实现)
- [使用指南](#使用指南)
- [高级特性](#高级特性)
- [实战场景](#实战场景)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是Whisper？

Whisper是OpenAI开发的通用语音识别模型，也被称为**ASR (Automatic Speech Recognition)** 或 **STT (Speech-to-Text)**。

```
音频文件 → Whisper模型 → 文本内容
```

### Whisper的特点

1. ✅ **多语言支持**: 支持99种语言的语音识别
2. ✅ **多任务能力**: 支持转录、翻译和语言识别
3. ✅ **高准确性**: 训练数据达68万小时
4. ✅ **噪声鲁棒性**: 能处理带背景噪音的音频
5. ✅ **多种输出格式**: JSON、文本、SRT字幕、VTT字幕等

### 应用场景

- 📝 **会议记录**: 自动生成会议纪要
- 🎥 **视频字幕**: 为视频自动生成字幕
- 📞 **客服质检**: 电话录音转文本分析
- 🎙️ **播客转录**: 音频内容转文字
- 🌍 **多语言翻译**: 语音翻译成英文文本
- ♿ **辅助功能**: 为听障人士提供实时字幕

---

## 核心接口

### 接口层次结构

```
Model<TReq, TRes> (根接口)
    ↑
    │
TranscriptionModel (语音转文本接口)
    ↑
    │
OpenAiAudioTranscriptionModel (OpenAI实现)
```

### TranscriptionModel接口

```java
/**
 * 语音转文本模型接口
 * 继承自Model接口
 */
public interface TranscriptionModel 
    extends Model<AudioTranscriptionPrompt, AudioTranscriptionResponse> {
    
    /**
     * 核心方法：转录音频
     * @param transcriptionPrompt 包含音频资源和选项的Prompt
     * @return 转录响应
     */
    AudioTranscriptionResponse call(AudioTranscriptionPrompt transcriptionPrompt);
    
    /**
     * 便捷方法：直接转录音频资源
     * @param resource 音频资源
     * @return 转录的文本
     */
    default String transcribe(Resource resource) {
        AudioTranscriptionPrompt prompt = 
            new AudioTranscriptionPrompt(resource);
        return this.call(prompt)
            .getResult()
            .getOutput();
    }
    
    /**
     * 便捷方法：使用选项转录音频
     * @param resource 音频资源
     * @param options 转录选项
     * @return 转录的文本
     */
    default String transcribe(
            Resource resource, 
            AudioTranscriptionOptions options) {
        
        AudioTranscriptionPrompt prompt = 
            new AudioTranscriptionPrompt(resource, options);
        return this.call(prompt)
            .getResult()
            .getOutput();
    }
}
```

---

## TranscriptionModel详解

### 1. AudioTranscriptionPrompt（转录请求）

```java
/**
 * 音频转录请求
 * 封装音频资源和转录选项
 */
public class AudioTranscriptionPrompt 
    implements ModelRequest<Resource> {
    
    /**
     * 音频资源
     * 支持格式：mp3, mp4, mpeg, mpga, m4a, wav, webm
     */
    private final Resource audioResource;
    
    /**
     * 转录选项（可选）
     */
    private AudioTranscriptionOptions modelOptions;
    
    // 构造方法
    public AudioTranscriptionPrompt(Resource audioResource) {
        this.audioResource = audioResource;
    }
    
    public AudioTranscriptionPrompt(
            Resource audioResource,
            AudioTranscriptionOptions modelOptions) {
        this.audioResource = audioResource;
        this.modelOptions = modelOptions;
    }
    
    @Override
    public Resource getInstructions() {
        return this.audioResource;
    }
    
    @Override
    public AudioTranscriptionOptions getOptions() {
        return this.modelOptions;
    }
}
```

### 2. AudioTranscriptionResponse（转录响应）

```java
/**
 * 音频转录响应
 */
public class AudioTranscriptionResponse 
    implements ModelResponse<AudioTranscription> {
    
    /**
     * 转录结果列表
     */
    private final List<AudioTranscription> transcriptions;
    
    /**
     * 响应元数据
     */
    private final AudioTranscriptionResponseMetadata responseMetadata;
    
    /**
     * 获取第一个转录结果
     */
    public AudioTranscription getResult() {
        if (CollectionUtils.isEmpty(this.transcriptions)) {
            return null;
        }
        return this.transcriptions.get(0);
    }
    
    /**
     * 获取所有转录结果
     */
    @Override
    public List<AudioTranscription> getResults() {
        return this.transcriptions;
    }
    
    /**
     * 获取元数据
     */
    @Override
    public AudioTranscriptionResponseMetadata getMetadata() {
        return this.responseMetadata;
    }
}
```

### 3. AudioTranscription（转录结果）

```java
/**
 * 单个转录结果
 */
public class AudioTranscription implements ModelResult<String> {
    
    /**
     * 转录的文本
     */
    private final String text;
    
    /**
     * 转录元数据
     */
    private AudioTranscriptionMetadata transcriptionMetadata;
    
    /**
     * 获取转录文本
     */
    @Override
    public String getOutput() {
        return this.text;
    }
    
    /**
     * 获取元数据
     */
    @Override
    public AudioTranscriptionMetadata getMetadata() {
        return this.transcriptionMetadata != null ? 
            this.transcriptionMetadata : 
            AudioTranscriptionMetadata.NULL;
    }
}
```

---

## 配置选项

### AudioTranscriptionOptions接口

```java
/**
 * 音频转录选项接口
 */
public interface AudioTranscriptionOptions extends ModelOptions {
    
    /**
     * 获取模型名称
     */
    String getModel();
}
```

### OpenAI Whisper特有选项

```java
/**
 * OpenAI音频转录选项
 */
@JsonInclude(Include.NON_NULL)
public class OpenAiAudioTranscriptionOptions 
    implements AudioTranscriptionOptions {
    
    /**
     * 模型ID
     * 当前只支持 "whisper-1"
     */
    private String model;
    
    /**
     * 输出格式
     * 选项：json, text, srt, verbose_json, vtt
     * 默认：json
     */
    private TranscriptResponseFormat responseFormat;
    
    /**
     * 提示词
     * 用于引导模型的风格或继续前一段音频
     * 应与音频语言匹配
     */
    private String prompt;
    
    /**
     * 音频语言
     * ISO-639-1格式（如：en, zh, ja）
     * 提供语言可提高准确性和速度
     */
    private String language;
    
    /**
     * 采样温度 (0-1)
     * 0: 最确定性
     * 1: 最随机性
     * 默认：0
     */
    private Float temperature;
    
    /**
     * 时间戳粒度
     * 选项：word（单词级）, segment（句子级）
     * 需要responseFormat为verbose_json
     */
    private GranularityType granularityType;
    
    // Builder模式
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        
        public Builder model(String model) { }
        
        public Builder responseFormat(TranscriptResponseFormat format) { }
        
        public Builder prompt(String prompt) { }
        
        public Builder language(String language) { }
        
        public Builder temperature(Float temperature) { }
        
        public Builder granularityType(GranularityType granularityType) { }
        
        public OpenAiAudioTranscriptionOptions build() { }
    }
}
```

### 输出格式

```java
/**
 * 转录输出格式
 */
public enum TranscriptResponseFormat {
    
    /**
     * JSON格式
     * 返回：{"text": "转录文本"}
     */
    JSON,
    
    /**
     * 纯文本格式
     * 返回：转录文本（无格式）
     */
    TEXT,
    
    /**
     * SRT字幕格式
     * SubRip Text格式，包含时间戳
     */
    SRT,
    
    /**
     * 详细JSON格式
     * 包含详细信息（语言、时长、单词级时间戳等）
     */
    VERBOSE_JSON,
    
    /**
     * VTT字幕格式
     * Web Video Text Tracks格式
     */
    VTT
}
```

---

## OpenAI Whisper实现

### OpenAiAudioTranscriptionModel

```java
/**
 * OpenAI Whisper实现
 */
public class OpenAiAudioTranscriptionModel 
    implements TranscriptionModel {
    
    private final OpenAiAudioTranscriptionOptions defaultOptions;
    private final RetryTemplate retryTemplate;
    private final OpenAiAudioApi audioApi;
    
    /**
     * 默认构造（使用默认配置）
     */
    public OpenAiAudioTranscriptionModel(OpenAiAudioApi audioApi) {
        this(audioApi,
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .responseFormat(TranscriptResponseFormat.JSON)
                .temperature(0.7f)
                .build());
    }
    
    /**
     * 自定义配置构造
     */
    public OpenAiAudioTranscriptionModel(
            OpenAiAudioApi audioApi,
            OpenAiAudioTranscriptionOptions options) {
        this(audioApi, options, RetryUtils.DEFAULT_RETRY_TEMPLATE);
    }
    
    /**
     * 完整构造（包含重试）
     */
    public OpenAiAudioTranscriptionModel(
            OpenAiAudioApi audioApi,
            OpenAiAudioTranscriptionOptions options,
            RetryTemplate retryTemplate) {
        
        Assert.notNull(audioApi, "OpenAiAudioApi must not be null");
        Assert.notNull(options, "Options must not be null");
        Assert.notNull(retryTemplate, "RetryTemplate must not be null");
        
        this.audioApi = audioApi;
        this.defaultOptions = options;
        this.retryTemplate = retryTemplate;
    }
    
    @Override
    public AudioTranscriptionResponse call(
            AudioTranscriptionPrompt transcriptionPrompt) {
        
        Resource audioResource = transcriptionPrompt.getInstructions();
        
        // 创建API请求
        TranscriptionRequest request = createRequest(transcriptionPrompt);
        
        // 判断输出格式
        if (request.responseFormat().isJsonType()) {
            // JSON格式响应
            ResponseEntity<StructuredResponse> transcriptionEntity = 
                retryTemplate.execute(ctx -> 
                    audioApi.createTranscription(
                        request, 
                        StructuredResponse.class
                    )
                );
            
            StructuredResponse transcription = transcriptionEntity.getBody();
            
            if (transcription == null) {
                logger.warn("No transcription returned");
                return new AudioTranscriptionResponse(null);
            }
            
            AudioTranscription transcript = 
                new AudioTranscription(transcription.text());
            
            RateLimit rateLimits = 
                OpenAiResponseHeaderExtractor
                    .extractAiResponseHeaders(transcriptionEntity);
            
            return new AudioTranscriptionResponse(
                transcript,
                OpenAiAudioTranscriptionResponseMetadata
                    .from(transcriptionEntity.getBody())
                    .withRateLimit(rateLimits)
            );
        } else {
            // 文本格式响应
            ResponseEntity<String> transcriptionEntity = 
                retryTemplate.execute(ctx -> 
                    audioApi.createTranscription(request, String.class)
                );
            
            String transcription = transcriptionEntity.getBody();
            
            if (transcription == null) {
                logger.warn("No transcription returned");
                return new AudioTranscriptionResponse(null);
            }
            
            AudioTranscription transcript = 
                new AudioTranscription(transcription);
            
            RateLimit rateLimits = 
                OpenAiResponseHeaderExtractor
                    .extractAiResponseHeaders(transcriptionEntity);
            
            return new AudioTranscriptionResponse(
                transcript,
                OpenAiAudioTranscriptionResponseMetadata
                    .from(transcriptionEntity.getBody())
                    .withRateLimit(rateLimits)
            );
        }
    }
    
    private TranscriptionRequest createRequest(
            AudioTranscriptionPrompt transcriptionPrompt) {
        
        // 合并选项（请求级优先）
        OpenAiAudioTranscriptionOptions options = 
            transcriptionPrompt.getOptions() != null ?
                (OpenAiAudioTranscriptionOptions) transcriptionPrompt.getOptions() :
                defaultOptions;
        
        // 读取音频文件
        Resource audioResource = transcriptionPrompt.getInstructions();
        byte[] audioBytes = readAudioBytes(audioResource);
        
        // 构建API请求
        return TranscriptionRequest.builder()
            .file(audioBytes)
            .fileName(audioResource.getFilename())
            .model(options.getModel())
            .responseFormat(options.getResponseFormat())
            .prompt(options.getPrompt())
            .language(options.getLanguage())
            .temperature(options.getTemperature())
            .granularityType(options.getGranularityType())
            .build();
    }
}
```

---

## 使用指南

### 1. 添加依赖

```xml
<!-- Spring AI OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      audio:
        transcription:
          options:
            model: whisper-1
            temperature: 0.0
            language: zh  # 中文
```

### 3. 基本使用

```java
@Service
public class TranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    
    public TranscriptionService(
            TranscriptionModel transcriptionModel) {
        this.transcriptionModel = transcriptionModel;
    }
    
    /**
     * 最简单的转录
     */
    public String transcribe(Resource audioFile) {
        return transcriptionModel.transcribe(audioFile);
    }
    
    /**
     * 使用配置选项转录
     */
    public String transcribeWithOptions(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")  // 中文
                .temperature(0.0f)  // 确定性输出
                .responseFormat(TranscriptResponseFormat.TEXT)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    /**
     * 完整的转录（获取详细信息）
     */
    public AudioTranscriptionResponse transcribeDetailed(
            Resource audioFile) {
        
        AudioTranscriptionPrompt prompt = 
            new AudioTranscriptionPrompt(audioFile);
        
        return transcriptionModel.call(prompt);
    }
}
```

### 4. REST API示例

```java
@RestController
@RequestMapping("/api/transcription")
public class TranscriptionController {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 上传音频文件并转录
     */
    @PostMapping("/upload")
    public ResponseEntity<TranscriptionResult> transcribeAudio(
            @RequestParam("file") MultipartFile file) {
        
        try {
            // 1. 保存临时文件
            File tempFile = File.createTempFile("audio", ".mp3");
            file.transferTo(tempFile);
            Resource audioResource = new FileSystemResource(tempFile);
            
            // 2. 转录
            OpenAiAudioTranscriptionOptions options = 
                OpenAiAudioTranscriptionOptions.builder()
                    .model("whisper-1")
                    .language("zh")
                    .responseFormat(TranscriptResponseFormat.JSON)
                    .build();
            
            AudioTranscriptionPrompt prompt = 
                new AudioTranscriptionPrompt(audioResource, options);
            
            AudioTranscriptionResponse response = 
                transcriptionModel.call(prompt);
            
            // 3. 返回结果
            String text = response.getResult().getOutput();
            
            // 4. 清理临时文件
            tempFile.delete();
            
            return ResponseEntity.ok(
                new TranscriptionResult(text, "success")
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new TranscriptionResult(
                    null, 
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    /**
     * 从URL转录音频
     */
    @PostMapping("/from-url")
    public ResponseEntity<TranscriptionResult> transcribeFromUrl(
            @RequestParam("url") String audioUrl) {
        
        try {
            // 下载音频
            Resource audioResource = new UrlResource(audioUrl);
            
            // 转录
            String text = transcriptionModel.transcribe(audioResource);
            
            return ResponseEntity.ok(
                new TranscriptionResult(text, "success")
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new TranscriptionResult(
                    null, 
                    "Error: " + e.getMessage()
                ));
        }
    }
    
    record TranscriptionResult(String text, String status) {}
}
```

---

## 高级特性

### 1. 多语言支持

```java
@Service
public class MultiLanguageTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 中文转录
     */
    public String transcribeChinese(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    /**
     * 英文转录
     */
    public String transcribeEnglish(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("en")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    /**
     * 日文转录
     */
    public String transcribeJapanese(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("ja")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    /**
     * 自动检测语言
     */
    public String transcribeAuto(Resource audioFile) {
        // 不指定language，让Whisper自动检测
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
}
```

### 2. 生成字幕

```java
@Service
public class SubtitleGenerationService {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 生成SRT字幕
     */
    public String generateSrtSubtitle(Resource videoAudio) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .responseFormat(TranscriptResponseFormat.SRT)
                .language("zh")
                .build();
        
        return transcriptionModel.transcribe(videoAudio, options);
    }
    
    /**
     * 生成VTT字幕
     */
    public String generateVttSubtitle(Resource videoAudio) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .responseFormat(TranscriptResponseFormat.VTT)
                .language("zh")
                .build();
        
        return transcriptionModel.transcribe(videoAudio, options);
    }
    
    /**
     * 保存字幕文件
     */
    public void saveSubtitleFile(
            Resource videoAudio,
            String outputPath) throws IOException {
        
        String subtitle = generateSrtSubtitle(videoAudio);
        
        Files.writeString(
            Path.of(outputPath),
            subtitle,
            StandardCharsets.UTF_8
        );
    }
}
```

### 3. 详细信息（时间戳）

```java
@Service
public class DetailedTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 获取单词级时间戳
     */
    public DetailedTranscription transcribeWithWordTimestamps(
            Resource audioFile) {
        
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .responseFormat(TranscriptResponseFormat.VERBOSE_JSON)
                .granularityType(GranularityType.WORD)
                .build();
        
        AudioTranscriptionPrompt prompt = 
            new AudioTranscriptionPrompt(audioFile, options);
        
        AudioTranscriptionResponse response = 
            transcriptionModel.call(prompt);
        
        // 从元数据中提取详细信息
        AudioTranscriptionMetadata metadata = 
            response.getResult().getMetadata();
        
        return new DetailedTranscription(
            response.getResult().getOutput(),
            metadata
        );
    }
    
    /**
     * 获取句子级时间戳
     */
    public DetailedTranscription transcribeWithSegmentTimestamps(
            Resource audioFile) {
        
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .responseFormat(TranscriptResponseFormat.VERBOSE_JSON)
                .granularityType(GranularityType.SEGMENT)
                .build();
        
        AudioTranscriptionPrompt prompt = 
            new AudioTranscriptionPrompt(audioFile, options);
        
        AudioTranscriptionResponse response = 
            transcriptionModel.call(prompt);
        
        return new DetailedTranscription(
            response.getResult().getOutput(),
            response.getResult().getMetadata()
        );
    }
    
    record DetailedTranscription(
        String text,
        AudioTranscriptionMetadata metadata
    ) {}
}
```

### 4. 提示词优化

```java
@Service
public class PromptGuidedTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 使用提示词引导转录风格
     */
    public String transcribeWithPrompt(
            Resource audioFile,
            String prompt) {
        
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")
                .prompt(prompt)  // 引导性提示词
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    /**
     * 技术术语提示
     */
    public String transcribeTechnicalContent(Resource audioFile) {
        String prompt = """
            这是一段技术讨论，包含专业术语如：
            Spring Boot, Kubernetes, Docker, 
            微服务, 云原生, DevOps
            """;
        
        return transcribeWithPrompt(audioFile, prompt);
    }
    
    /**
     * 医疗术语提示
     */
    public String transcribeMedicalContent(Resource audioFile) {
        String prompt = """
            这是医疗记录，包含医学术语如：
            症状、诊断、治疗方案、药物名称
            """;
        
        return transcribeWithPrompt(audioFile, prompt);
    }
}
```

---

## 实战场景

### 1. 会议记录系统

```java
@Service
public class MeetingTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    private final ChatClient chatClient;
    
    /**
     * 处理会议录音
     */
    public MeetingNotes processMeeting(Resource meetingAudio) {
        // 1. 转录音频
        String transcript = transcribeMeeting(meetingAudio);
        
        // 2. 生成会议纪要
        String summary = generateSummary(transcript);
        
        // 3. 提取行动项
        List<ActionItem> actionItems = extractActionItems(transcript);
        
        // 4. 提取关键决策
        List<String> decisions = extractDecisions(transcript);
        
        return new MeetingNotes(
            transcript,
            summary,
            actionItems,
            decisions
        );
    }
    
    private String transcribeMeeting(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")
                .prompt("这是一场商务会议讨论")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    private String generateSummary(String transcript) {
        return chatClient
            .prompt(spec -> spec
                .system("你是会议记录助手，擅长总结会议内容")
                .user("""
                    请为以下会议记录生成简洁的摘要：
                    
                    {transcript}
                    
                    摘要应包括：
                    1. 会议主题
                    2. 主要讨论点
                    3. 达成的共识
                    """)
                .param("transcript", transcript)
            )
            .call()
            .content();
    }
    
    private List<ActionItem> extractActionItems(String transcript) {
        String response = chatClient
            .prompt(spec -> spec
                .system("提取会议中的行动项")
                .user("""
                    从以下会议记录中提取所有行动项：
                    
                    {transcript}
                    
                    以JSON数组格式返回，每个行动项包含：
                    - task: 任务描述
                    - assignee: 负责人
                    - deadline: 截止日期（如果提到）
                    """)
                .param("transcript", transcript)
            )
            .call()
            .content();
        
        // 解析JSON为ActionItem列表
        return parseActionItems(response);
    }
    
    private List<String> extractDecisions(String transcript) {
        String response = chatClient
            .prompt("从以下会议记录中提取所有重要决策：\n" + transcript)
            .call()
            .content();
        
        return Arrays.asList(response.split("\n"));
    }
    
    private List<ActionItem> parseActionItems(String json) {
        // JSON解析逻辑
        return List.of(); // 简化
    }
    
    record MeetingNotes(
        String transcript,
        String summary,
        List<ActionItem> actionItems,
        List<String> decisions
    ) {}
    
    record ActionItem(
        String task,
        String assignee,
        String deadline
    ) {}
}
```

### 2. 客服质检系统

```java
@Service
public class CustomerServiceQualityService {
    
    private final TranscriptionModel transcriptionModel;
    private final ChatClient chatClient;
    
    /**
     * 分析客服电话
     */
    public QualityReport analyzeCall(Resource callRecording) {
        // 1. 转录电话
        String transcript = transcribeCall(callRecording);
        
        // 2. 情感分析
        String sentiment = analyzeSentiment(transcript);
        
        // 3. 服务质量评分
        int qualityScore = calculateQualityScore(transcript);
        
        // 4. 提取问题
        List<String> issues = extractIssues(transcript);
        
        // 5. 改进建议
        List<String> suggestions = generateSuggestions(transcript);
        
        return new QualityReport(
            transcript,
            sentiment,
            qualityScore,
            issues,
            suggestions
        );
    }
    
    private String transcribeCall(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")
                .prompt("这是客服电话录音，包含客户和客服的对话")
                .temperature(0.0f)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    private String analyzeSentiment(String transcript) {
        return chatClient
            .prompt("""
                分析以下客服对话的情感倾向：
                
                %s
                
                返回：positive（积极）、neutral（中性）或negative（消极）
                """.formatted(transcript))
            .call()
            .content();
    }
    
    private int calculateQualityScore(String transcript) {
        String response = chatClient
            .prompt("""
                评估以下客服对话的服务质量（0-100分）：
                
                评分标准：
                - 礼貌程度（30分）
                - 问题解决能力（40分）
                - 响应速度（15分）
                - 专业性（15分）
                
                对话内容：
                %s
                
                只返回分数（数字）。
                """.formatted(transcript))
            .call()
            .content();
        
        try {
            return Integer.parseInt(response.trim());
        } catch (NumberFormatException e) {
            return 0;
        }
    }
    
    private List<String> extractIssues(String transcript) {
        String response = chatClient
            .prompt("从以下客服对话中提取客户遇到的所有问题：\n" + transcript)
            .call()
            .content();
        
        return Arrays.asList(response.split("\n"));
    }
    
    private List<String> generateSuggestions(String transcript) {
        String response = chatClient
            .prompt("""
                基于以下客服对话，提供改进建议：
                
                %s
                
                建议应该具体、可执行。
                """.formatted(transcript))
            .call()
            .content();
        
        return Arrays.asList(response.split("\n"));
    }
    
    record QualityReport(
        String transcript,
        String sentiment,
        int qualityScore,
        List<String> issues,
        List<String> suggestions
    ) {}
}
```

### 3. 播客转录和发布

```java
@Service
public class PodcastTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    private final ChatClient chatClient;
    
    /**
     * 处理播客
     */
    public PodcastContent processPodcast(
            Resource podcastAudio,
            PodcastMetadata metadata) {
        
        // 1. 转录音频
        String transcript = transcribePodcast(podcastAudio);
        
        // 2. 生成摘要
        String summary = generateSummary(transcript);
        
        // 3. 提取关键词
        List<String> keywords = extractKeywords(transcript);
        
        // 4. 生成标题建议
        List<String> titleSuggestions = generateTitles(transcript);
        
        // 5. 生成章节标记
        List<Chapter> chapters = generateChapters(transcript);
        
        return new PodcastContent(
            transcript,
            summary,
            keywords,
            titleSuggestions,
            chapters,
            metadata
        );
    }
    
    private String transcribePodcast(Resource audioFile) {
        OpenAiAudioTranscriptionOptions options = 
            OpenAiAudioTranscriptionOptions.builder()
                .model("whisper-1")
                .language("zh")
                .temperature(0.0f)
                .responseFormat(TranscriptResponseFormat.TEXT)
                .build();
        
        return transcriptionModel.transcribe(audioFile, options);
    }
    
    private String generateSummary(String transcript) {
        return chatClient
            .prompt("""
                为以下播客内容生成吸引人的摘要（200字以内）：
                
                %s
                """.formatted(transcript))
            .call()
            .content();
    }
    
    private List<String> extractKeywords(String transcript) {
        String response = chatClient
            .prompt("""
                从以下播客内容中提取10个关键词：
                
                %s
                
                只返回关键词，用逗号分隔。
                """.formatted(transcript))
            .call()
            .content();
        
        return Arrays.asList(response.split(","));
    }
    
    private List<String> generateTitles(String transcript) {
        String response = chatClient
            .prompt("""
                为以下播客内容生成5个吸引人的标题：
                
                %s
                """.formatted(transcript))
            .call()
            .content();
        
        return Arrays.asList(response.split("\n"));
    }
    
    private List<Chapter> generateChapters(String transcript) {
        // 使用AI识别章节分界点
        String response = chatClient
            .prompt("""
                将以下播客内容划分为章节，每个章节包含：
                - 章节标题
                - 简要描述
                
                内容：
                %s
                
                以JSON数组格式返回。
                """.formatted(transcript))
            .call()
            .content();
        
        // 解析JSON
        return parseChapters(response);
    }
    
    private List<Chapter> parseChapters(String json) {
        // JSON解析逻辑
        return List.of(); // 简化
    }
    
    record PodcastContent(
        String transcript,
        String summary,
        List<String> keywords,
        List<String> titleSuggestions,
        List<Chapter> chapters,
        PodcastMetadata metadata
    ) {}
    
    record PodcastMetadata(
        String host,
        String date,
        String category
    ) {}
    
    record Chapter(
        String title,
        String description,
        String timestamp
    ) {}
}
```

---

## 最佳实践

### 1. 音频预处理

```java
@Service
public class AudioPreprocessingService {
    
    /**
     * 检查音频格式
     */
    public boolean isSupportedFormat(Resource audioFile) {
        String filename = audioFile.getFilename();
        if (filename == null) {
            return false;
        }
        
        String[] supportedFormats = {
            ".mp3", ".mp4", ".mpeg", ".mpga",
            ".m4a", ".wav", ".webm"
        };
        
        return Arrays.stream(supportedFormats)
            .anyMatch(format -> filename.toLowerCase().endsWith(format));
    }
    
    /**
     * 检查文件大小（Whisper限制25MB）
     */
    public boolean isWithinSizeLimit(Resource audioFile) throws IOException {
        long maxSize = 25 * 1024 * 1024; // 25MB
        return audioFile.contentLength() <= maxSize;
    }
    
    /**
     * 分割大文件
     */
    public List<Resource> splitLargeAudio(Resource audioFile) {
        // 使用FFmpeg等工具分割音频
        // 这里是简化示例
        return List.of(audioFile);
    }
}
```

### 2. 错误处理

```java
@Service
public class RobustTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    private final RetryTemplate retryTemplate;
    
    /**
     * 带重试的转录
     */
    public String transcribeWithRetry(Resource audioFile) {
        return retryTemplate.execute(context -> {
            try {
                return transcriptionModel.transcribe(audioFile);
            } catch (Exception e) {
                logger.error("Transcription failed, attempt: {}", 
                    context.getRetryCount(), e);
                throw e;
            }
        });
    }
    
    /**
     * 带降级的转录
     */
    public String transcribeWithFallback(Resource audioFile) {
        try {
            return transcriptionModel.transcribe(audioFile);
        } catch (Exception e) {
            logger.error("Transcription failed", e);
            return "[转录失败：" + e.getMessage() + "]";
        }
    }
}
```

### 3. 性能优化

```java
@Service
public class OptimizedTranscriptionService {
    
    private final TranscriptionModel transcriptionModel;
    
    /**
     * 异步转录
     */
    @Async
    public CompletableFuture<String> transcribeAsync(Resource audioFile) {
        return CompletableFuture.supplyAsync(() -> 
            transcriptionModel.transcribe(audioFile)
        );
    }
    
    /**
     * 批量转录
     */
    public List<String> transcribeBatch(List<Resource> audioFiles) {
        return audioFiles.parallelStream()
            .map(audioFile -> {
                try {
                    return transcriptionModel.transcribe(audioFile);
                } catch (Exception e) {
                    logger.error("Failed to transcribe: {}", 
                        audioFile.getFilename(), e);
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .toList();
    }
    
    /**
     * 缓存转录结果
     */
    @Cacheable(value = "transcriptions", key = "#audioFile.filename")
    public String transcribeWithCache(Resource audioFile) {
        return transcriptionModel.transcribe(audioFile);
    }
}
```

---

## 总结

### Whisper核心特点

1. **多语言支持**: 99种语言
2. **高准确性**: 68万小时训练数据
3. **多种输出格式**: JSON、文本、SRT、VTT等
4. **时间戳支持**: 单词级和句子级
5. **噪声鲁棒性**: 能处理噪音环境

### Spring AI Whisper API

```
TranscriptionModel (接口)
    ↓
transcribe(Resource) → String
    ↓
call(AudioTranscriptionPrompt) → AudioTranscriptionResponse
```

### 配置要点

- **model**: "whisper-1"（当前唯一模型）
- **language**: 指定语言可提高准确性
- **temperature**: 0.0（确定性）到1.0（随机性）
- **responseFormat**: 选择输出格式
- **prompt**: 引导转录风格和术语

### 最佳实践清单

- ✅ 指定音频语言提高准确性
- ✅ 使用低温度(0.0)获得确定性结果
- ✅ 使用提示词处理专业术语
- ✅ 检查文件格式和大小
- ✅ 实现错误处理和重试
- ✅ 大文件分段处理
- ✅ 使用缓存避免重复转录
- ✅ 异步处理提升性能

通过Spring AI的Whisper集成，你可以轻松实现语音转文本功能，构建会议记录、客服质检、播客转录等强大应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

