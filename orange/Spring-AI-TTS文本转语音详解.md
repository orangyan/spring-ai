# Spring AI TTS 文本转语音详解

## 📋 目录
- [概述](#概述)
- [核心接口](#核心接口)
- [TextToSpeechModel详解](#texttospeechmodel详解)
- [配置选项](#配置选项)
- [OpenAI TTS实现](#openai-tts实现)
- [ElevenLabs TTS实现](#elevenlabs-tts实现)
- [使用指南](#使用指南)
- [流式语音合成](#流式语音合成)
- [实战场景](#实战场景)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是TTS？

TTS (Text-to-Speech) 即**文本转语音**，是一种将文本内容转换为自然语音的AI技术。

```
文本内容 → TTS模型 → 音频数据
```

### TTS的特点

1. ✅ **自然语音**: 类人的语音合成
2. ✅ **多语言支持**: 支持多种语言和口音
3. ✅ **声音定制**: 选择不同的声音特征
4. ✅ **语速控制**: 调整语音速度
5. ✅ **流式输出**: 支持实时流式合成
6. ✅ **多种格式**: MP3、WAV、FLAC、PCM等

### 应用场景

- 🎧 **有声读物**: 将书籍、文章转为有声版本
- 📱 **语音助手**: 虚拟助手的语音输出
- 🎬 **视频配音**: 自动生成视频旁白
- 📞 **电话语音**: IVR系统、自动外呼
- ♿ **辅助功能**: 为视障人士提供语音输出
- 📚 **教育应用**: 语言学习、在线课程配音
- 🎮 **游戏配音**: NPC对话生成

---

## 核心接口

### 接口层次结构

```
Model<TReq, TRes> (根接口)
    ↑
    │
TextToSpeechModel (文本转语音接口)
    ↑
    │
OpenAiAudioSpeechModel (OpenAI实现)
ElevenLabsTextToSpeechModel (ElevenLabs实现)

StreamingModel<TReq, TRes> (流式根接口)
    ↑
    │
StreamingTextToSpeechModel (流式文本转语音接口)
    ↑
    │
OpenAiAudioSpeechModel (支持流式)
```

### TextToSpeechModel接口

```java
/**
 * 文本转语音模型接口
 * 继承自Model接口
 */
public interface TextToSpeechModel 
    extends Model<TextToSpeechPrompt, TextToSpeechResponse> {
    
    /**
     * 便捷方法：直接将文本转为语音
     * @param text 要转换的文本
     * @return 音频字节数组
     */
    default byte[] call(String text) {
        TextToSpeechPrompt prompt = new TextToSpeechPrompt(text);
        ModelResult<byte[]> result = call(prompt).getResult();
        return (result != null) ? result.getOutput() : new byte[0];
    }
    
    /**
     * 核心方法：文本转语音
     * @param prompt 包含文本和选项的Prompt
     * @return 语音响应
     */
    @Override
    TextToSpeechResponse call(TextToSpeechPrompt prompt);
    
    /**
     * 获取默认选项
     * @return 默认的TTS选项
     */
    default TextToSpeechOptions getDefaultOptions() {
        return TextToSpeechOptions.builder().build();
    }
}
```

### StreamingTextToSpeechModel接口

```java
/**
 * 流式文本转语音接口
 * 支持实时语音流
 */
public interface StreamingTextToSpeechModel 
    extends StreamingModel<TextToSpeechPrompt, TextToSpeechResponse> {
    
    /**
     * 流式转换文本（简化）
     * @param text 要转换的文本
     * @return 音频数据流
     */
    default Flux<byte[]> stream(String text) {
        TextToSpeechPrompt prompt = new TextToSpeechPrompt(text);
        return stream(prompt)
            .map(response -> 
                (response.getResult() == null || 
                 response.getResult().getOutput() == null) ?
                    new byte[0] : 
                    response.getResult().getOutput()
            );
    }
    
    /**
     * 流式转换文本（带选项）
     * @param text 要转换的文本
     * @param options TTS选项
     * @return 音频数据流
     */
    default Flux<byte[]> stream(String text, TextToSpeechOptions options) {
        TextToSpeechPrompt prompt = new TextToSpeechPrompt(text, options);
        return stream(prompt)
            .map(response -> 
                (response.getResult() == null || 
                 response.getResult().getOutput() == null) ?
                    new byte[0] : 
                    response.getResult().getOutput()
            );
    }
    
    /**
     * 流式转换（完整版）
     * @param prompt 包含文本和选项的Prompt
     * @return 响应流
     */
    @Override
    Flux<TextToSpeechResponse> stream(TextToSpeechPrompt prompt);
}
```

---

## TextToSpeechModel详解

### 1. TextToSpeechPrompt（TTS请求）

```java
/**
 * 文本转语音请求
 * 封装文本消息和TTS选项
 */
public class TextToSpeechPrompt 
    implements ModelRequest<TextToSpeechMessage> {
    
    /**
     * 文本消息
     */
    private final TextToSpeechMessage message;
    
    /**
     * TTS选项（可选）
     */
    private TextToSpeechOptions options;
    
    /**
     * 构造方法1：仅文本
     */
    public TextToSpeechPrompt(String text) {
        this(new TextToSpeechMessage(text), 
            TextToSpeechOptions.builder().build());
    }
    
    /**
     * 构造方法2：文本+选项
     */
    public TextToSpeechPrompt(String text, TextToSpeechOptions options) {
        this(new TextToSpeechMessage(text), options);
    }
    
    /**
     * 构造方法3：消息
     */
    public TextToSpeechPrompt(TextToSpeechMessage message) {
        this(message, TextToSpeechOptions.builder().build());
    }
    
    /**
     * 构造方法4：消息+选项（完整版）
     */
    public TextToSpeechPrompt(
            TextToSpeechMessage message,
            TextToSpeechOptions options) {
        this.message = message;
        this.options = options;
    }
    
    @Override
    public TextToSpeechMessage getInstructions() {
        return this.message;
    }
    
    @Override
    public TextToSpeechOptions getOptions() {
        return this.options;
    }
    
    public void setOptions(TextToSpeechOptions options) {
        this.options = options;
    }
}
```

### 2. TextToSpeechMessage（文本消息）

```java
/**
 * 文本转语音消息
 * 包装要转换的文本内容
 */
public class TextToSpeechMessage {
    
    /**
     * 文本内容
     */
    private final String text;
    
    public TextToSpeechMessage(String text) {
        Assert.hasText(text, "Text must not be empty");
        this.text = text;
    }
    
    public String getText() {
        return this.text;
    }
}
```

### 3. TextToSpeechResponse（TTS响应）

```java
/**
 * 文本转语音响应
 */
public class TextToSpeechResponse 
    implements ModelResponse<Speech> {
    
    /**
     * 语音结果列表
     */
    private final List<Speech> speeches;
    
    /**
     * 响应元数据
     */
    private final TextToSpeechResponseMetadata responseMetadata;
    
    /**
     * 获取第一个语音结果
     */
    public Speech getResult() {
        if (CollectionUtils.isEmpty(this.speeches)) {
            return null;
        }
        return this.speeches.get(0);
    }
    
    /**
     * 获取所有语音结果
     */
    @Override
    public List<Speech> getResults() {
        return this.speeches;
    }
    
    /**
     * 获取元数据
     */
    @Override
    public TextToSpeechResponseMetadata getMetadata() {
        return this.responseMetadata;
    }
}
```

### 4. Speech（语音结果）

```java
/**
 * 单个语音结果
 * 包含音频字节数据
 */
public class Speech implements ModelResult<byte[]> {
    
    /**
     * 音频数据
     */
    private final byte[] speech;
    
    public Speech(byte[] speech) {
        this.speech = speech;
    }
    
    /**
     * 获取音频字节数组
     */
    @Override
    public byte[] getOutput() {
        return this.speech;
    }
    
    /**
     * 获取元数据
     */
    @Override
    public ResultMetadata getMetadata() {
        return null;
    }
}
```

---

## 配置选项

### TextToSpeechOptions接口

```java
/**
 * 文本转语音选项接口
 * 定义通用的TTS配置
 */
public interface TextToSpeechOptions extends ModelOptions {
    
    /**
     * 创建Builder
     */
    static TextToSpeechOptions.Builder builder() {
        return new DefaultTextToSpeechOptions.Builder();
    }
    
    /**
     * 获取模型名称
     * @return 模型名称
     */
    @Nullable
    String getModel();
    
    /**
     * 获取声音
     * @return 声音标识符
     */
    @Nullable
    String getVoice();
    
    /**
     * 获取输出格式
     * @return 音频格式（如："mp3", "wav"）
     */
    @Nullable
    String getFormat();
    
    /**
     * 获取语速
     * @return 语速（如：1.0为正常速度）
     */
    @Nullable
    Double getSpeed();
    
    /**
     * 复制选项
     */
    <T extends TextToSpeechOptions> T copy();
    
    /**
     * Builder接口
     */
    interface Builder {
        
        /**
         * 设置模型
         */
        Builder model(String model);
        
        /**
         * 设置声音
         */
        Builder voice(String voice);
        
        /**
         * 设置格式
         */
        Builder format(String format);
        
        /**
         * 设置语速
         */
        Builder speed(Double speed);
        
        /**
         * 构建选项
         */
        TextToSpeechOptions build();
    }
}
```

### DefaultTextToSpeechOptions实现

```java
/**
 * 默认的文本转语音选项实现
 */
public class DefaultTextToSpeechOptions 
    implements TextToSpeechOptions {
    
    private String model;
    private String voice;
    private String format;
    private Double speed;
    
    // Getters
    @Override
    public String getModel() {
        return this.model;
    }
    
    @Override
    public String getVoice() {
        return this.voice;
    }
    
    @Override
    public String getFormat() {
        return this.format;
    }
    
    @Override
    public Double getSpeed() {
        return this.speed;
    }
    
    /**
     * Builder实现
     */
    public static class Builder 
        implements TextToSpeechOptions.Builder {
        
        private String model;
        private String voice;
        private String format;
        private Double speed;
        
        @Override
        public Builder model(String model) {
            this.model = model;
            return this;
        }
        
        @Override
        public Builder voice(String voice) {
            this.voice = voice;
            return this;
        }
        
        @Override
        public Builder format(String format) {
            this.format = format;
            return this;
        }
        
        @Override
        public Builder speed(Double speed) {
            this.speed = speed;
            return this;
        }
        
        @Override
        public TextToSpeechOptions build() {
            DefaultTextToSpeechOptions options = 
                new DefaultTextToSpeechOptions();
            options.model = this.model;
            options.voice = this.voice;
            options.format = this.format;
            options.speed = this.speed;
            return options;
        }
    }
}
```

---

## OpenAI TTS实现

### OpenAI TTS模型

OpenAI提供三个TTS模型：

1. **tts-1**: 标准质量，速度优化
2. **tts-1-hd**: 高清质量
3. **gpt-4o-mini-tts**: GPT-4o mini驱动

### OpenAI声音选项

```java
/**
 * OpenAI支持的声音
 */
public enum Voice {
    
    ALLOY("alloy"),         // 中性、均衡
    ASH("ash"),             // 温暖、自信
    BALLAD("ballad"),       // 平静、舒缓
    CORAL("coral"),         // 友好、温暖
    ECHO("echo"),           // 深沉、回响
    FABLE("fable"),         // 叙事风格
    ONYX("onyx"),           // 深沉、权威
    NOVA("nova"),           // 年轻、活力
    SAGE("sage"),           // 智慧、成熟
    SHIMMER("shimmer"),     // 柔和、闪亮
    VERSE("verse");         // 诗意、韵律
    
    private final String value;
    
    Voice(String value) {
        this.value = value;
    }
    
    public String getValue() {
        return this.value;
    }
}
```

### OpenAiAudioSpeechOptions

```java
/**
 * OpenAI TTS选项
 */
@JsonInclude(JsonInclude.Include.NON_NULL)
public class OpenAiAudioSpeechOptions implements ModelOptions {
    
    /**
     * 模型ID
     * 可选：tts-1, tts-1-hd, gpt-4o-mini-tts
     */
    @JsonProperty("model")
    private String model;
    
    /**
     * 输入文本
     * 最多4096个token
     */
    @JsonProperty("input")
    private String input;
    
    /**
     * 声音
     * 可选：alloy, ash, ballad, coral, echo, fable, 
     *      onyx, nova, sage, shimmer, verse
     */
    @JsonProperty("voice")
    private String voice;
    
    /**
     * 响应格式
     * 可选：mp3, opus, aac, flac, wav, pcm
     * 默认：mp3
     */
    @JsonProperty("response_format")
    private AudioResponseFormat responseFormat;
    
    /**
     * 语速
     * 范围：0.25（最慢）到4.0（最快）
     * 默认：1.0（正常）
     * 注意：gpt-4o-mini-tts不支持
     */
    @JsonProperty("speed")
    private Float speed;
    
    public static Builder builder() {
        return new Builder();
    }
    
    // Getters and Setters
    
    public static class Builder {
        
        private String model;
        private String input;
        private String voice;
        private AudioResponseFormat responseFormat;
        private Float speed;
        
        public Builder model(String model) {
            this.model = model;
            return this;
        }
        
        public Builder input(String input) {
            this.input = input;
            return this;
        }
        
        public Builder voice(String voice) {
            this.voice = voice;
            return this;
        }
        
        public Builder voice(Voice voice) {
            this.voice = voice.getValue();
            return this;
        }
        
        public Builder responseFormat(AudioResponseFormat format) {
            this.responseFormat = format;
            return this;
        }
        
        public Builder speed(Float speed) {
            this.speed = speed;
            return this;
        }
        
        public OpenAiAudioSpeechOptions build() {
            OpenAiAudioSpeechOptions options = 
                new OpenAiAudioSpeechOptions();
            options.setModel(this.model);
            options.setInput(this.input);
            options.setVoice(this.voice);
            options.setResponseFormat(this.responseFormat);
            options.setSpeed(this.speed);
            return options;
        }
    }
}
```

### 音频格式

```java
/**
 * 音频响应格式
 */
public enum AudioResponseFormat {
    
    /**
     * MP3格式（默认）
     * 最通用、兼容性好
     */
    MP3("mp3"),
    
    /**
     * Opus格式
     * 适合流式传输、低延迟
     */
    OPUS("opus"),
    
    /**
     * AAC格式
     * 高质量、适合移动设备
     */
    AAC("aac"),
    
    /**
     * FLAC格式
     * 无损压缩、音质最佳
     */
    FLAC("flac"),
    
    /**
     * WAV格式
     * 无压缩、音质最佳、文件最大
     */
    WAV("wav"),
    
    /**
     * PCM格式
     * 原始音频数据
     */
    PCM("pcm");
    
    private final String value;
    
    AudioResponseFormat(String value) {
        this.value = value;
    }
    
    public String getValue() {
        return this.value;
    }
}
```

### OpenAiAudioSpeechModel

```java
/**
 * OpenAI TTS模型实现
 * 支持同步和流式输出
 */
public class OpenAiAudioSpeechModel 
    implements SpeechModel, StreamingSpeechModel {
    
    private static final Float SPEED = 1.0f;
    
    private final OpenAiAudioSpeechOptions defaultOptions;
    private final RetryTemplate retryTemplate;
    private final OpenAiAudioApi audioApi;
    
    /**
     * 默认构造（使用默认配置）
     */
    public OpenAiAudioSpeechModel(OpenAiAudioApi audioApi) {
        this(audioApi,
            OpenAiAudioSpeechOptions.builder()
                .model(TtsModel.TTS_1.getValue())
                .responseFormat(AudioResponseFormat.MP3)
                .voice(Voice.ALLOY.getValue())
                .speed(SPEED)
                .build());
    }
    
    /**
     * 自定义配置构造
     */
    public OpenAiAudioSpeechModel(
            OpenAiAudioApi audioApi,
            OpenAiAudioSpeechOptions options) {
        this(audioApi, options, RetryUtils.DEFAULT_RETRY_TEMPLATE);
    }
    
    /**
     * 完整构造（包含重试）
     */
    public OpenAiAudioSpeechModel(
            OpenAiAudioApi audioApi,
            OpenAiAudioSpeechOptions options,
            RetryTemplate retryTemplate) {
        
        Assert.notNull(audioApi, "OpenAiAudioApi must not be null");
        Assert.notNull(options, "Options must not be null");
        Assert.notNull(retryTemplate, "RetryTemplate must not be null");
        
        this.audioApi = audioApi;
        this.defaultOptions = options;
        this.retryTemplate = retryTemplate;
    }
    
    /**
     * 同步调用
     */
    @Override
    public SpeechResponse call(SpeechPrompt speechPrompt) {
        
        SpeechRequest request = createRequest(speechPrompt);
        
        ResponseEntity<byte[]> responseEntity = 
            this.retryTemplate.execute(ctx -> 
                this.audioApi.createSpeech(request)
            );
        
        byte[] audioBytes = responseEntity.getBody();
        
        if (audioBytes == null) {
            throw new IllegalStateException(
                "No audio returned from OpenAI TTS API"
            );
        }
        
        SpeechGeneration generation = new SpeechGeneration(audioBytes);
        
        RateLimit rateLimits = 
            OpenAiResponseHeaderExtractor
                .extractAiResponseHeaders(responseEntity);
        
        return new SpeechResponse(
            List.of(generation),
            OpenAiAudioSpeechResponseMetadata.from(responseEntity)
                .withRateLimit(rateLimits)
        );
    }
    
    /**
     * 流式调用
     */
    @Override
    public Flux<SpeechResponse> stream(SpeechPrompt speechPrompt) {
        
        SpeechRequest request = createRequest(speechPrompt);
        
        return this.audioApi.streamSpeech(request)
            .map(audioChunk -> {
                SpeechGeneration generation = 
                    new SpeechGeneration(audioChunk);
                return new SpeechResponse(
                    List.of(generation),
                    OpenAiAudioSpeechResponseMetadata.empty()
                );
            });
    }
    
    private SpeechRequest createRequest(SpeechPrompt speechPrompt) {
        
        // 合并选项
        OpenAiAudioSpeechOptions options = 
            mergeOptions(speechPrompt.getOptions());
        
        String text = speechPrompt.getInstructions();
        
        return SpeechRequest.builder()
            .model(options.getModel())
            .input(text)
            .voice(options.getVoice())
            .responseFormat(options.getResponseFormat())
            .speed(options.getSpeed())
            .build();
    }
    
    private OpenAiAudioSpeechOptions mergeOptions(
            ModelOptions requestOptions) {
        
        if (requestOptions == null) {
            return this.defaultOptions;
        }
        
        // 请求级选项优先
        return (OpenAiAudioSpeechOptions) requestOptions;
    }
}
```

---

## ElevenLabs TTS实现

### ElevenLabs简介

ElevenLabs是专注于语音合成的AI公司，提供：
- 更自然的语音质量
- 声音克隆功能
- 情感控制
- 多语言支持

### ElevenLabsTextToSpeechModel

```java
/**
 * ElevenLabs TTS实现
 */
public class ElevenLabsTextToSpeechModel 
    implements TextToSpeechModel, StreamingTextToSpeechModel {
    
    private final ElevenLabsApi api;
    private final ElevenLabsTextToSpeechOptions defaultOptions;
    
    @Override
    public TextToSpeechResponse call(TextToSpeechPrompt prompt) {
        
        String text = prompt.getInstructions().getText();
        ElevenLabsTextToSpeechOptions options = 
            (ElevenLabsTextToSpeechOptions) prompt.getOptions();
        
        // 调用ElevenLabs API
        byte[] audioBytes = api.textToSpeech(
            text,
            options.getVoiceId(),
            options.getModelId(),
            options.getVoiceSettings()
        );
        
        Speech speech = new Speech(audioBytes);
        
        return new TextToSpeechResponse(
            List.of(speech),
            TextToSpeechResponseMetadata.empty()
        );
    }
    
    @Override
    public Flux<TextToSpeechResponse> stream(
            TextToSpeechPrompt prompt) {
        
        String text = prompt.getInstructions().getText();
        ElevenLabsTextToSpeechOptions options = 
            (ElevenLabsTextToSpeechOptions) prompt.getOptions();
        
        // 流式调用
        return api.streamTextToSpeech(
            text,
            options.getVoiceId(),
            options.getModelId(),
            options.getVoiceSettings()
        )
        .map(audioChunk -> {
            Speech speech = new Speech(audioChunk);
            return new TextToSpeechResponse(
                List.of(speech),
                TextToSpeechResponseMetadata.empty()
            );
        });
    }
}
```

---

## 使用指南

### 1. 添加依赖

```xml
<!-- OpenAI TTS -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- 或 ElevenLabs TTS -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-elevenlabs-spring-boot-starter</artifactId>
</dependency>
```

### 2. 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      audio:
        speech:
          options:
            model: tts-1
            voice: alloy
            response-format: mp3
            speed: 1.0
```

### 3. 基本使用

```java
@Service
public class TextToSpeechService {
    
    private final TextToSpeechModel ttsModel;
    
    public TextToSpeechService(TextToSpeechModel ttsModel) {
        this.ttsModel = ttsModel;
    }
    
    /**
     * 最简单的TTS
     */
    public byte[] textToSpeech(String text) {
        return ttsModel.call(text);
    }
    
    /**
     * 保存为文件
     */
    public void textToSpeechFile(String text, String outputPath) 
            throws IOException {
        
        byte[] audioBytes = ttsModel.call(text);
        
        Files.write(
            Path.of(outputPath),
            audioBytes,
            StandardOpenOption.CREATE
        );
    }
    
    /**
     * 使用配置选项
     */
    public byte[] textToSpeechWithOptions(String text) {
        
        TextToSpeechOptions options = 
            TextToSpeechOptions.builder()
                .model("tts-1-hd")  // 高清质量
                .voice("nova")      // Nova声音
                .format("mp3")
                .speed(1.2)         // 1.2倍速
                .build();
        
        TextToSpeechPrompt prompt = 
            new TextToSpeechPrompt(text, options);
        
        return ttsModel.call(prompt)
            .getResult()
            .getOutput();
    }
}
```

### 4. REST API示例

```java
@RestController
@RequestMapping("/api/tts")
public class TTSController {
    
    private final TextToSpeechModel ttsModel;
    
    /**
     * 文本转语音（返回音频）
     */
    @PostMapping(
        value = "/synthesize",
        produces = MediaType.APPLICATION_OCTET_STREAM_VALUE
    )
    public ResponseEntity<byte[]> synthesize(
            @RequestBody TTSRequest request) {
        
        try {
            OpenAiAudioSpeechOptions options = 
                OpenAiAudioSpeechOptions.builder()
                    .model(request.model())
                    .voice(request.voice())
                    .responseFormat(
                        AudioResponseFormat.valueOf(
                            request.format().toUpperCase()
                        )
                    )
                    .speed(request.speed())
                    .build();
            
            TextToSpeechPrompt prompt = 
                new TextToSpeechPrompt(request.text(), options);
            
            byte[] audioBytes = ttsModel.call(prompt)
                .getResult()
                .getOutput();
            
            // 设置Content-Type
            String contentType = switch (request.format()) {
                case "mp3" -> "audio/mpeg";
                case "wav" -> "audio/wav";
                case "flac" -> "audio/flac";
                default -> "application/octet-stream";
            };
            
            return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_TYPE, contentType)
                .header(
                    HttpHeaders.CONTENT_DISPOSITION,
                    "attachment; filename=\"speech.\" + request.format()"
                )
                .body(audioBytes);
                
        } catch (Exception e) {
            return ResponseEntity.status(500).build();
        }
    }
    
    /**
     * 文本转语音（保存到服务器）
     */
    @PostMapping("/synthesize-and-save")
    public ResponseEntity<TTSResponse> synthesizeAndSave(
            @RequestBody TTSRequest request) {
        
        try {
            byte[] audioBytes = ttsModel.call(request.text());
            
            // 生成文件名
            String filename = "speech_" + 
                System.currentTimeMillis() + ".mp3";
            String filepath = "/uploads/audio/" + filename;
            
            // 保存文件
            Files.write(
                Path.of(filepath),
                audioBytes,
                StandardOpenOption.CREATE
            );
            
            return ResponseEntity.ok(
                new TTSResponse(
                    "success",
                    filepath,
                    audioBytes.length
                )
            );
            
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(new TTSResponse(
                    "error: " + e.getMessage(),
                    null,
                    0
                ));
        }
    }
    
    record TTSRequest(
        String text,
        String model,
        String voice,
        String format,
        Float speed
    ) {}
    
    record TTSResponse(
        String status,
        String filepath,
        long fileSize
    ) {}
}
```

---

## 流式语音合成

### 为什么需要流式？

- ⚡ **低延迟**: 边生成边播放，不需等待全部生成
- 💾 **省内存**: 不需要一次性加载整个音频
- 🎯 **实时性**: 适合实时对话场景

### 流式使用示例

```java
@Service
public class StreamingTTSService {
    
    private final StreamingTextToSpeechModel streamingTtsModel;
    
    /**
     * 流式生成语音
     */
    public Flux<byte[]> streamTextToSpeech(String text) {
        return streamingTtsModel.stream(text);
    }
    
    /**
     * 流式生成并保存
     */
    public Mono<Void> streamToFile(String text, String outputPath) {
        
        return Flux.create(sink -> {
            try (FileOutputStream fos = 
                    new FileOutputStream(outputPath)) {
                
                streamingTtsModel.stream(text)
                    .doOnNext(audioChunk -> {
                        try {
                            fos.write(audioChunk);
                            sink.next(audioChunk);
                        } catch (IOException e) {
                            sink.error(e);
                        }
                    })
                    .doOnComplete(sink::complete)
                    .doOnError(sink::error)
                    .subscribe();
                    
            } catch (IOException e) {
                sink.error(e);
            }
        })
        .then();
    }
    
    /**
     * 流式生成并实时播放
     */
    public Flux<byte[]> streamForPlayback(String text) {
        
        TextToSpeechOptions options = 
            TextToSpeechOptions.builder()
                .model("tts-1")       // 使用速度优化模型
                .format("opus")       // 使用适合流式的格式
                .build();
        
        return streamingTtsModel.stream(text, options);
    }
}
```

### 流式REST API

```java
@RestController
@RequestMapping("/api/tts")
public class StreamingTTSController {
    
    private final StreamingTextToSpeechModel streamingTtsModel;
    
    /**
     * 服务器发送事件（SSE）
     */
    @GetMapping(
        value = "/stream",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<ServerSentEvent<String>> streamTTS(
            @RequestParam String text) {
        
        return streamingTtsModel.stream(text)
            .map(audioChunk -> 
                ServerSentEvent.<String>builder()
                    .event("audio-chunk")
                    .data(Base64.getEncoder()
                        .encodeToString(audioChunk))
                    .build()
            );
    }
    
    /**
     * WebFlux响应流
     */
    @PostMapping(
        value = "/stream-audio",
        produces = MediaType.APPLICATION_OCTET_STREAM_VALUE
    )
    public Flux<byte[]> streamAudio(
            @RequestBody String text) {
        
        return streamingTtsModel.stream(text);
    }
}
```

---

## 实战场景

### 1. 有声书生成器

```java
@Service
public class AudioBookService {
    
    private final TextToSpeechModel ttsModel;
    
    /**
     * 将长文本转为有声书
     */
    public void generateAudioBook(
            String bookContent,
            String outputDir) throws IOException {
        
        // 1. 分章节
        List<Chapter> chapters = splitIntoChapters(bookContent);
        
        // 2. 为每章生成音频
        for (int i = 0; i < chapters.size(); i++) {
            Chapter chapter = chapters.get(i);
            
            String audioFilename = String.format(
                "%s/chapter_%02d.mp3",
                outputDir,
                i + 1
            );
            
            generateChapterAudio(chapter, audioFilename);
        }
    }
    
    private void generateChapterAudio(
            Chapter chapter,
            String outputPath) throws IOException {
        
        // 配置：使用高质量模型和适合叙事的声音
        TextToSpeechOptions options = 
            TextToSpeechOptions.builder()
                .model("tts-1-hd")
                .voice("fable")  // 叙事风格
                .format("mp3")
                .speed(1.0)
                .build();
        
        // 分段处理（避免单次文本过长）
        List<String> segments = splitIntoSegments(
            chapter.content(),
            4000  // 每段约4000字符
        );
        
        ByteArrayOutputStream audioStream = 
            new ByteArrayOutputStream();
        
        for (String segment : segments) {
            TextToSpeechPrompt prompt = 
                new TextToSpeechPrompt(segment, options);
            
            byte[] audioBytes = ttsModel.call(prompt)
                .getResult()
                .getOutput();
            
            audioStream.write(audioBytes);
        }
        
        // 保存完整章节音频
        Files.write(
            Path.of(outputPath),
            audioStream.toByteArray()
        );
    }
    
    private List<Chapter> splitIntoChapters(String content) {
        // 按章节标题分割
        return List.of(); // 简化
    }
    
    private List<String> splitIntoSegments(String text, int maxLength) {
        List<String> segments = new ArrayList<>();
        int start = 0;
        
        while (start < text.length()) {
            int end = Math.min(start + maxLength, text.length());
            
            // 在句号处断开
            if (end < text.length()) {
                int lastPeriod = text.lastIndexOf('。', end);
                if (lastPeriod > start) {
                    end = lastPeriod + 1;
                }
            }
            
            segments.add(text.substring(start, end));
            start = end;
        }
        
        return segments;
    }
    
    record Chapter(String title, String content) {}
}
```

### 2. 多语言播客生成

```java
@Service
public class PodcastGeneratorService {
    
    private final TextToSpeechModel ttsModel;
    private final ChatClient chatClient;
    
    /**
     * 生成播客
     */
    public void generatePodcast(
            String topic,
            String language,
            String outputPath) throws IOException {
        
        // 1. 生成播客脚本
        String script = generateScript(topic, language);
        
        // 2. 根据语言选择声音
        String voice = selectVoiceForLanguage(language);
        
        // 3. 生成音频
        TextToSpeechOptions options = 
            TextToSpeechOptions.builder()
                .model("tts-1-hd")
                .voice(voice)
                .format("mp3")
                .speed(1.0)
                .build();
        
        TextToSpeechPrompt prompt = 
            new TextToSpeechPrompt(script, options);
        
        byte[] audioBytes = ttsModel.call(prompt)
            .getResult()
            .getOutput();
        
        // 4. 保存
        Files.write(Path.of(outputPath), audioBytes);
    }
    
    private String generateScript(String topic, String language) {
        return chatClient
            .prompt("""
                为以下主题创建一个3分钟的播客脚本：
                
                主题：{topic}
                语言：{language}
                
                脚本应该：
                1. 引人入胜的开场白
                2. 清晰的主体内容
                3. 有力的结尾
                """)
            .param("topic", topic)
            .param("language", language)
            .call()
            .content();
    }
    
    private String selectVoiceForLanguage(String language) {
        return switch (language.toLowerCase()) {
            case "chinese", "中文" -> "shimmer";
            case "english" -> "nova";
            case "japanese" -> "alloy";
            default -> "nova";
        };
    }
}
```

### 3. 实时语音助手

```java
@Service
public class VoiceAssistantService {
    
    private final ChatClient chatClient;
    private final StreamingTextToSpeechModel streamingTtsModel;
    
    /**
     * 对话并返回语音
     */
    public Flux<byte[]> chat(String userMessage) {
        
        // 1. 获取AI回复
        Flux<String> textStream = chatClient
            .prompt(userMessage)
            .stream()
            .content();
        
        // 2. 累积文本到句子级别
        Flux<String> sentenceStream = accumulateToSentences(textStream);
        
        // 3. 将每个句子转为语音
        return sentenceStream.flatMap(sentence -> 
            streamingTtsModel.stream(sentence)
        );
    }
    
    /**
     * 累积文本到句子级别
     */
    private Flux<String> accumulateToSentences(Flux<String> textStream) {
        
        return textStream.window(
            textStream.filter(text -> 
                text.endsWith("。") || 
                text.endsWith("！") || 
                text.endsWith("？") ||
                text.endsWith(".") ||
                text.endsWith("!") ||
                text.endsWith("?")
            )
        )
        .flatMap(window -> window.reduce("", String::concat));
    }
}
```

### 4. 视频配音系统

```java
@Service
public class VideoDubbingService {
    
    private final TextToSpeechModel ttsModel;
    private final ChatClient chatClient;
    
    /**
     * 为视频生成配音
     */
    public void generateDubbing(
            VideoScript script,
            String outputDir) throws IOException {
        
        // 为每个场景生成配音
        for (int i = 0; i < script.scenes().size(); i++) {
            VideoScene scene = script.scenes().get(i);
            
            String audioFile = String.format(
                "%s/scene_%02d.mp3",
                outputDir,
                i + 1
            );
            
            generateSceneAudio(scene, audioFile);
        }
    }
    
    private void generateSceneAudio(
            VideoScene scene,
            String outputPath) throws IOException {
        
        // 选择合适的声音
        String voice = selectVoiceForCharacter(scene.character());
        
        // 配置
        TextToSpeechOptions options = 
            TextToSpeechOptions.builder()
                .model("tts-1-hd")
                .voice(voice)
                .format("mp3")
                .speed(scene.speedMultiplier())
                .build();
        
        // 生成音频
        TextToSpeechPrompt prompt = 
            new TextToSpeechPrompt(scene.dialogue(), options);
        
        byte[] audioBytes = ttsModel.call(prompt)
            .getResult()
            .getOutput();
        
        // 保存
        Files.write(Path.of(outputPath), audioBytes);
    }
    
    private String selectVoiceForCharacter(String character) {
        return switch (character.toLowerCase()) {
            case "narrator" -> "fable";
            case "young_male" -> "echo";
            case "young_female" -> "nova";
            case "mature_male" -> "onyx";
            case "mature_female" -> "shimmer";
            default -> "alloy";
        };
    }
    
    record VideoScript(List<VideoScene> scenes) {}
    
    record VideoScene(
        String character,
        String dialogue,
        Float speedMultiplier
    ) {}
}
```

---

## 最佳实践

### 1. 文本预处理

```java
@Service
public class TextPreprocessingService {
    
    /**
     * 清理和优化文本
     */
    public String preprocessText(String text) {
        
        // 1. 移除HTML标签
        text = text.replaceAll("<[^>]*>", "");
        
        // 2. 标准化空白字符
        text = text.replaceAll("\\s+", " ");
        
        // 3. 处理特殊字符
        text = text.replaceAll("[\\x00-\\x1F\\x7F]", "");
        
        // 4. 添加适当的停顿
        text = text.replaceAll("([。！？])", "$1\n");
        
        return text.trim();
    }
    
    /**
     * 检查文本长度
     */
    public boolean isTextTooLong(String text) {
        // OpenAI限制：4096 tokens（约3000-3500字）
        return text.length() > 3000;
    }
    
    /**
     * 分割长文本
     */
    public List<String> splitLongText(String text, int maxLength) {
        List<String> chunks = new ArrayList<>();
        
        // 按段落分割
        String[] paragraphs = text.split("\n\n");
        
        StringBuilder currentChunk = new StringBuilder();
        
        for (String paragraph : paragraphs) {
            if (currentChunk.length() + paragraph.length() > maxLength) {
                if (currentChunk.length() > 0) {
                    chunks.add(currentChunk.toString());
                    currentChunk = new StringBuilder();
                }
            }
            currentChunk.append(paragraph).append("\n\n");
        }
        
        if (currentChunk.length() > 0) {
            chunks.add(currentChunk.toString());
        }
        
        return chunks;
    }
}
```

### 2. 缓存策略

```java
@Service
public class CachedTTSService {
    
    private final TextToSpeechModel ttsModel;
    
    /**
     * 缓存TTS结果
     * 相同文本+选项 → 返回缓存的音频
     */
    @Cacheable(
        value = "tts-cache",
        key = "#text + '_' + #options"
    )
    public byte[] textToSpeechWithCache(
            String text,
            TextToSpeechOptions options) {
        
        TextToSpeechPrompt prompt = 
            new TextToSpeechPrompt(text, options);
        
        return ttsModel.call(prompt)
            .getResult()
            .getOutput();
    }
    
    /**
     * 清除缓存
     */
    @CacheEvict(value = "tts-cache", allEntries = true)
    public void clearCache() {
        // 缓存会自动清除
    }
}
```

### 3. 错误处理

```java
@Service
public class RobustTTSService {
    
    private final TextToSpeechModel ttsModel;
    private final RetryTemplate retryTemplate;
    
    /**
     * 带重试的TTS
     */
    public byte[] textToSpeechWithRetry(String text) {
        
        return retryTemplate.execute(context -> {
            try {
                return ttsModel.call(text);
            } catch (Exception e) {
                logger.error("TTS failed, attempt: {}", 
                    context.getRetryCount(), e);
                throw e;
            }
        });
    }
    
    /**
     * 带降级的TTS
     */
    public byte[] textToSpeechWithFallback(
            String text,
            byte[] defaultAudio) {
        
        try {
            return ttsModel.call(text);
        } catch (Exception e) {
            logger.error("TTS failed, using fallback", e);
            return defaultAudio;
        }
    }
}
```

### 4. 性能优化

```java
@Service
public class OptimizedTTSService {
    
    private final TextToSpeechModel ttsModel;
    
    /**
     * 异步TTS
     */
    @Async
    public CompletableFuture<byte[]> textToSpeechAsync(String text) {
        return CompletableFuture.supplyAsync(() -> 
            ttsModel.call(text)
        );
    }
    
    /**
     * 批量处理
     */
    public List<byte[]> textToSpeechBatch(List<String> texts) {
        
        return texts.parallelStream()
            .map(text -> {
                try {
                    return ttsModel.call(text);
                } catch (Exception e) {
                    logger.error("Failed to convert text: {}", text, e);
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .toList();
    }
    
    /**
     * 流式处理长文本
     */
    public Flux<byte[]> streamLongText(String longText) {
        
        // 分段
        List<String> segments = splitText(longText, 1000);
        
        // 流式生成每段
        return Flux.fromIterable(segments)
            .flatMap(segment -> 
                Mono.fromCallable(() -> ttsModel.call(segment))
            );
    }
    
    private List<String> splitText(String text, int chunkSize) {
        List<String> chunks = new ArrayList<>();
        int start = 0;
        
        while (start < text.length()) {
            int end = Math.min(start + chunkSize, text.length());
            chunks.add(text.substring(start, end));
            start = end;
        }
        
        return chunks;
    }
}
```

---

## 总结

### TTS核心特点

1. **多模型支持**: OpenAI TTS-1/HD/GPT-4o-mini, ElevenLabs
2. **多种声音**: 11种OpenAI声音，适合不同场景
3. **灵活输出**: MP3、WAV、FLAC、PCM等格式
4. **语速控制**: 0.25x到4.0x
5. **流式输出**: 支持实时流式合成

### Spring AI TTS API

```
TextToSpeechModel (接口)
    ↓
call(String) → byte[]
    ↓
call(TextToSpeechPrompt) → TextToSpeechResponse

StreamingTextToSpeechModel (流式接口)
    ↓
stream(String) → Flux<byte[]>
    ↓
stream(TextToSpeechPrompt) → Flux<TextToSpeechResponse>
```

### 配置要点

- **model**: tts-1（速度）或tts-1-hd（质量）
- **voice**: 选择合适的声音特征
- **format**: 根据用途选择格式（MP3通用，OPUS流式）
- **speed**: 调整语速（1.0为正常）

### 最佳实践清单

- ✅ 长文本分段处理（<4000字符）
- ✅ 使用缓存避免重复转换
- ✅ 实时场景使用流式输出
- ✅ 选择合适的音频格式
- ✅ 文本预处理去除特殊字符
- ✅ 实现错误处理和重试
- ✅ 异步处理提升性能
- ✅ 根据场景选择模型和声音

通过Spring AI的TTS集成，你可以轻松实现文本转语音功能，构建有声书、播客、语音助手、视频配音等丰富应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-02  
**Spring AI版本**: 1.1.0-SNAPSHOT

