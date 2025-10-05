# Spring AI 学习文档

> 深入学习Spring AI框架的完整指南

## 📖 关于本文档集

这是一套完整的Spring AI框架学习文档，涵盖从基础概念到高级特性的所有核心内容。每份文档都包含：

- ✅ **源码级解析**：深入剖析核心接口和实现
- ✅ **完整示例**：提供可运行的代码示例
- ✅ **实战案例**：真实业务场景的完整实现
- ✅ **最佳实践**：生产环境的经验总结

---

## 📚 文档列表

### 🎯 核心基础（必读）

#### 1. [ChatModel核心详解](./Spring-AI-ChatModel核心详解.md)
**内容概览**：
- ChatModel核心接口和生命周期
- Prompt、Message、ChatResponse数据结构
- StreamingChatModel流式响应
- ChatOptions配置详解
- OpenAI、Ollama等多模型支持
- 实战：智能客服、代码生成、文档摘要

**适合人群**：所有Spring AI学习者（必读）  
**前置知识**：无  
**学习时长**：2-3小时

---

#### 2. [ChatClient完整指南](./Spring-AI-ChatClient完整指南.md)
**内容概览**：
- Fluent API设计理念
- 请求构建（user/system/options/context）
- 响应处理（content/entity/stream）
- Advisor集成机制
- 实战：REST API、SSE流式、结构化输出

**适合人群**：需要快速开发AI应用的开发者  
**前置知识**：ChatModel基础  
**学习时长**：2-3小时

---

#### 3. [Prompt核心详解](./Spring-AI-Prompt核心详解.md)
**内容概览**：
- Prompt核心概念和架构定位
- Prompt类的6种创造方式和所有方法详解
- PromptTemplate模板系统完整解析
- Message体系（System/User/Assistant/Tool）
- ChatPromptTemplate多角色组合
- augment增强、mutate修改机制
- 实战：代码审查、翻译服务、文档问答、Few-shot学习

**内容特色**：
- 📖 **源码级剖析**：逐个方法详细解释
- 🎯 **6种创建方式**：从简单到复杂的完整对比
- 🔧 **模板系统**：PromptTemplate家族完整介绍
- 💡 **实战案例**：4个完整的生产级应用场景

**适合人群**：所有Spring AI开发者（核心必读）  
**前置知识**：ChatModel基础  
**学习时长**：2-3小时

**推荐阅读时机**：
- ✅ 刚开始学习Spring AI
- ✅ 想理解Prompt的完整机制
- ✅ 需要设计复杂的提示词
- ✅ 要实现动态Prompt构建

---

#### 4. [提示词工程详解](./Spring-AI提示词工程详解.md)
**内容概览**：
- PromptTemplate模板引擎
- ChatPromptTemplate角色组合
- SystemPromptTemplate/UserPromptTemplate
- StTemplateRenderer渲染器
- Prompt与Message的关系
- 实战：动态提示词、多角色对话、Few-shot学习

**适合人群**：需要优化AI交互质量的开发者  
**前置知识**：Prompt核心、ChatModel基础  
**学习时长**：1-2小时

---

#### 5. [大模型调用完整流程详解](./Spring-AI大模型调用完整流程详解.md)
**内容概览**：
- 七个阶段完整流程（请求构建 → API调用 → 响应返回）
- 源码级执行追踪（每一行代码的作用）
- Advisor链的递归调用机制
- ChatModel内部实现原理
- OpenAI API交互细节
- 请求/响应数据转换链
- 流式调用流程对比
- 实战：完整调用追踪、调试技巧

**内容特色**：
- 🔍 **超详细源码解析**：逐行解释关键代码
- 📊 **可视化流程图**：架构图、时序图、执行栈图
- 🎯 **实战追踪**：第1次和第2次对话的完整流程对比
- 💡 **调试指南**：断点位置、日志配置、性能分析

**适合人群**：想深入理解框架内部机制的开发者、需要调试和优化的工程师  
**前置知识**：ChatModel、ChatClient、Advisor基础  
**学习时长**：2-3小时

**推荐阅读时机**：
- ✅ 已经能熟练使用ChatClient
- ✅ 想了解"黑盒"内部发生了什么
- ✅ 需要进行性能优化或问题排查
- ✅ 计划开发自定义Advisor或ChatModel

---

### 🔧 能力扩展

#### 6. [Embedding向量嵌入详解](./Spring-AI-Embedding向量嵌入详解.md)
**内容概览**：
- 向量嵌入的原理和应用
- EmbeddingModel接口
- EmbeddingRequest/EmbeddingResponse
- 批处理和Token管理
- OpenAI、Ollama、Azure等模型支持
- 实战：文档相似度、文本聚类、语义搜索

**适合人群**：需要实现语义搜索的开发者  
**前置知识**：基础向量概念  
**学习时长**：2小时

---

#### 7. [VectorStore向量数据库详解](./Spring-AI-VectorStore向量数据库详解.md)
**内容概览**：
- VectorStore核心接口
- SearchRequest相似度搜索
- Filter元数据过滤（三种创建方式）
- SimpleVectorStore实现
- 20+种向量数据库支持
- 实战：知识库问答、文档去重、智能推荐

**适合人群**：需要持久化向量数据的开发者  
**前置知识**：Embedding基础  
**学习时长**：2-3小时

---

#### 8. [Function Calling工具调用详解](./Spring-AI-FunctionCalling工具调用详解.md)
**内容概览**：
- @Tool注解使用
- ToolCallback接口
- FunctionToolCallback函数式工具
- MethodToolCallback方法式工具
- ToolDefinition和JSON Schema
- ChatClient集成
- 实战：数据查询、天气查询、订单处理

**适合人群**：需要让AI调用外部工具的开发者  
**前置知识**：ChatClient基础  
**学习时长**：2-3小时

---

#### 9. [Whisper语音转文本详解](./Spring-AI-Whisper语音转文本详解.md)
**内容概览**：
- TranscriptionModel接口
- AudioTranscriptionPrompt/Response
- 音频格式支持（MP3、WAV、M4A等）
- 多语言识别和翻译
- OpenAI Whisper实现
- 实战：会议纪要、语音助手、字幕生成

**适合人群**：需要语音识别功能的开发者  
**前置知识**：无  
**学习时长**：1-2小时

---

#### 10. [TTS文本转语音详解](./Spring-AI-TTS文本转语音详解.md)
**内容概览**：
- TextToSpeechModel接口
- StreamingTextToSpeechModel流式
- 语音选择（alloy、echo、nova等）
- 音频格式（MP3、Opus、AAC等）
- OpenAI TTS、ElevenLabs实现
- 实战：有声读物、语音提醒、多语言播报

**适合人群**：需要语音合成功能的开发者  
**前置知识**：无  
**学习时长**：1-2小时

---

### 🚀 高级特性

#### 11. [Advisor机制详解](./Spring-AI-Advisor机制详解.md)
**内容概览**：
- Advisor核心接口（CallAdvisor/StreamAdvisor）
- BaseAdvisor基类
- AdvisorChain执行链
- 内置Advisor（Memory、RAG、Logger等）
- 自定义Advisor开发
- 实战：日志记录、安全检查、重试机制

**适合人群**：需要扩展ChatClient行为的高级开发者  
**前置知识**：ChatClient基础  
**学习时长**：2-3小时

---

#### 12. [RAG检索增强生成详解](./Spring-AI-RAG检索增强生成详解.md)
**内容概览**：
- RAG核心概念和架构
- RetrievalAugmentationAdvisor
- Query转换和扩展
- DocumentRetriever检索器
- DocumentPostProcessor后处理
- QueryAugmenter增强器
- 实战：企业知识库、技术文档助手、智能客服

**适合人群**：需要构建知识库问答系统的开发者  
**前置知识**：Embedding、VectorStore、Advisor基础  
**学习时长**：3-4小时

---

#### 13. [Memory对话记忆详解](./Spring-AI-Memory对话记忆详解.md)
**内容概览**：
- ChatMemory vs ChatHistory
- MessageWindowChatMemory实现
- ChatMemoryRepository存储（6种）
- MessageChatMemoryAdvisor消息列表模式
- PromptChatMemoryAdvisor文本模式
- VectorStoreChatMemoryAdvisor向量检索模式
- 实战：智能客服、长期知识库、混合策略

**适合人群**：需要实现多轮对话的开发者  
**前置知识**：ChatClient、Advisor基础  
**学习时长**：2-3小时

---

## 🎓 学习路径推荐

### 路径一：快速上手（适合新手）

```
第1天：ChatModel核心详解 → Prompt核心详解
       ↓
第2天：ChatClient完整指南 → Memory对话记忆详解
       ↓
第3天：实战项目：构建一个简单的聊天机器人

可选进阶：
  - 提示词工程详解（优化Prompt设计）
  - 大模型调用完整流程详解（了解内部机制）
```

**学习目标**：能够快速搭建一个具备多轮对话能力的AI应用

---

### 路径二：RAG知识库（适合企业应用）

```
第1周：基础准备
  ├─ ChatModel核心详解
  ├─ ChatClient完整指南
  └─ Advisor机制详解

第2周：向量技术栈
  ├─ Embedding向量嵌入详解
  ├─ VectorStore向量数据库详解
  └─ 实践：文档向量化和存储

第3周：RAG实战
  ├─ RAG检索增强生成详解
  ├─ Memory对话记忆详解
  └─ 实战项目：企业知识库问答系统
```

**学习目标**：掌握构建企业级RAG知识库系统

---

### 路径三：多模态AI（适合全栈应用）

```
第1-2周：核心基础
  ├─ ChatModel核心详解
  ├─ ChatClient完整指南
  ├─ Advisor机制详解
  └─ Memory对话记忆详解

第3周：多模态能力
  ├─ Whisper语音转文本详解
  ├─ TTS文本转语音详解
  └─ 实践：语音对话功能

第4周：能力整合
  ├─ Function Calling工具调用详解
  ├─ Embedding + VectorStore
  └─ 实战项目：多模态智能助手
```

**学习目标**：构建支持语音交互的全功能AI助手

---

### 路径四：架构师进阶（适合高级开发者）

```
第1周：核心抽象
  ├─ ChatModel核心详解（重点：接口设计）
  ├─ Prompt核心详解（重点：请求构建）⭐ 必读
  ├─ ChatClient完整指南（重点：Fluent API）
  ├─ 大模型调用完整流程详解（重点：执行流程）⭐ 核心
  └─ Advisor机制详解（重点：AOP设计）

第2周：存储架构
  ├─ VectorStore向量数据库详解（重点：存储抽象）
  ├─ Memory对话记忆详解（重点：Repository模式）
  └─ 源码阅读：多种Repository实现

第3周：高级特性
  ├─ RAG检索增强生成详解（重点：Modular RAG）
  ├─ Function Calling工具调用详解（重点：工具编排）
  └─ 实践：自定义Advisor和工具

第4周：生产实践
  ├─ 监控和可观测性
  ├─ 性能优化和成本控制
  └─ 分布式部署和扩展
```

**学习目标**：深入理解Spring AI架构，具备设计大规模AI应用的能力

---

## 📊 文档功能对照表

| 文档 | 核心接口 | 主要特性 | 典型应用 | 难度 |
|------|----------|----------|----------|------|
| ChatModel | `ChatModel` | 同步/流式调用 | 所有AI应用 | ⭐ |
| ChatClient | `ChatClient` | Fluent API | 快速开发 | ⭐ |
| **Prompt** | **Prompt/PromptTemplate** | **请求构建/模板** | **所有场景** | **⭐** |
| 提示词工程 | `PromptTemplate` | 高级模板 | 提示词优化 | ⭐⭐ |
| **调用流程** | **全流程** | **源码追踪** | **调试优化** | **⭐⭐⭐** |
| Embedding | `EmbeddingModel` | 向量生成 | 语义搜索 | ⭐⭐ |
| VectorStore | `VectorStore` | 向量存储 | 知识库 | ⭐⭐ |
| Function Calling | `@Tool` | 工具调用 | 智能代理 | ⭐⭐ |
| Whisper | `TranscriptionModel` | 语音识别 | 语音助手 | ⭐ |
| TTS | `TextToSpeechModel` | 语音合成 | 语音播报 | ⭐ |
| Advisor | `BaseAdvisor` | 拦截器 | 扩展增强 | ⭐⭐⭐ |
| RAG | `RetrievalAugmentationAdvisor` | 检索增强 | 知识问答 | ⭐⭐⭐ |
| Memory | `ChatMemory` | 对话记忆 | 多轮对话 | ⭐⭐ |

**难度说明**：
- ⭐ 基础：可快速上手
- ⭐⭐ 中级：需要一定理解
- ⭐⭐⭐ 高级：需要深入学习

---

## 🔗 文档关联关系

```
                    ChatModel (核心抽象)
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ChatClient  ⭐ Prompt ⭐   ChatOptions
      (使用)        (输入)         (配置)
                     │
              ┌──────┴──────┐
              │             │
        PromptTemplate   Message
         (模板化)       (消息体)
          │
          ├─────────── Advisor (扩展机制)
          │                │
          │        ┌───────┼───────┬───────────┐
          │        │       │       │           │
          │    Memory    RAG   Function   Logger
          │                │    Calling
          │                │
          │          ┌─────┴─────┐
          │          │           │
          │     Embedding   VectorStore
          │                      │
          │                ┌─────┴─────┬─────────┐
          │                │           │         │
          │            PgVector    Chroma    Pinecone
          │
          └─── 多模态能力
                  │
            ┌─────┼─────┐
            │           │
        Whisper       TTS
      (语音转文本)  (文本转语音)
```

**理解关联关系有助于**：
- 明确学习顺序
- 理解组件协作
- 设计系统架构

---

## 💡 如何使用这些文档

### 1. 针对性学习

**场景驱动学习法**：

```java
// 场景1: 我想做一个简单的聊天机器人
→ 阅读：ChatModel + ChatClient + Memory

// 场景2: 我想做一个企业知识库问答系统
→ 阅读：Embedding + VectorStore + RAG + Memory

// 场景3: 我想做一个语音助手
→ 阅读：Whisper + TTS + ChatClient + Memory

// 场景4: 我想让AI能调用外部工具
→ 阅读：Function Calling + ChatClient
```

### 2. 渐进式学习

```
阶段1：理解概念
  └─ 阅读"核心概念"部分

阶段2：源码研读
  └─ 阅读"源码解析"部分

阶段3：动手实践
  └─ 运行"使用示例"代码

阶段4：实战应用
  └─ 实现"完整实战案例"

阶段5：优化提升
  └─ 参考"最佳实践"
```

### 3. 查阅式使用

每份文档都包含详细目录，可作为API参考手册使用：

```
遇到问题 → 查找对应文档 → 定位章节 → 查看示例 → 解决问题
```

---

## 🛠️ 实战项目建议

### 初级项目

#### 1. 智能聊天机器人
**涉及文档**：ChatModel、ChatClient、Memory  
**功能点**：
- 多轮对话
- 上下文记忆
- 个性化回复

---

#### 2. 文档问答助手
**涉及文档**：Embedding、VectorStore、RAG  
**功能点**：
- 文档上传和向量化
- 语义搜索
- 基于文档的问答

---

### 中级项目

#### 3. 语音交互助手
**涉及文档**：Whisper、TTS、ChatClient、Memory  
**功能点**：
- 语音输入识别
- 智能对话
- 语音输出合成

---

#### 4. 智能客服系统
**涉及文档**：全部核心文档  
**功能点**：
- 多渠道接入（文字、语音）
- 知识库检索
- 工单创建（Function Calling）
- 对话历史管理

---

### 高级项目

#### 5. 企业级RAG知识库
**涉及文档**：全部文档 + 架构设计  
**功能点**：
- 分布式向量存储
- 多模型支持
- 查询优化和缓存
- 监控和日志
- 权限控制

---

#### 6. AI智能代理平台
**涉及文档**：Function Calling、Advisor、RAG  
**功能点**：
- 工具编排
- 多Agent协作
- 任务规划和执行
- 可观测性

---

## 📈 学习进度追踪

### 建议的学习检查清单

#### 核心基础
- [ ] 理解ChatModel的核心接口
- [ ] 掌握Prompt的6种创建方式
- [ ] 掌握ChatClient的Fluent API
- [ ] 理解PromptTemplate模板系统
- [ ] 实现一个简单的聊天应用

#### 能力扩展
- [ ] 理解向量嵌入的原理
- [ ] 掌握VectorStore的使用
- [ ] 实现Function Calling工具
- [ ] 集成语音识别和合成

#### 高级特性
- [ ] 理解Advisor的执行机制
- [ ] 掌握RAG的完整流程
- [ ] 实现自定义Advisor
- [ ] 构建企业级RAG系统

#### 实战能力
- [ ] 完成至少2个完整项目
- [ ] 掌握性能优化技巧
- [ ] 了解生产环境部署
- [ ] 能够进行架构设计

---

## 🆘 常见问题

### Q1: 我应该从哪份文档开始？
**A**: 建议从`ChatModel核心详解`开始，这是所有功能的基础。

### Q2: 文档之间有依赖关系吗？
**A**: 有。建议按照"核心基础 → 能力扩展 → 高级特性"的顺序学习。

### Q3: 需要什么前置知识？
**A**: 
- Java基础（必需）
- Spring Boot基础（推荐）
- 基本的AI概念（可选）

### Q4: 文档中的代码可以直接运行吗？
**A**: 大部分示例代码可以直接运行，部分需要配置API密钥和依赖。

### Q5: 如何获取实战项目的完整代码？
**A**: 文档中的实战案例提供了核心代码，可以作为起点进行扩展。

---

## 📝 文档更新记录

### 2025年10月5日
- ✅ 创建13份核心文档
- ✅ 覆盖Spring AI所有主要特性
- ✅ 提供完整的学习路径
- ✅ 新增1：大模型调用完整流程详解（源码级深度解析）
- ✅ 新增2：Prompt核心详解（6种创建方式+完整方法详解+4个实战案例）

### 未来计划
- [ ] Image Generation（图像生成）
- [ ] Document Readers（文档读取器）
- [ ] MCP（Model Context Protocol）
- [ ] Observability（可观测性）
- [ ] Moderation（内容审核）

---

## 🎯 学习建议

### 1. 动手实践
**理论 + 实践 = 真正掌握**

每学完一份文档，务必：
- 运行示例代码
- 修改代码参数观察效果
- 实现一个小项目

### 2. 源码阅读
**理解设计思想**

文档中的源码解析是精简版，建议：
- 对照Spring AI源码阅读
- 理解接口设计理念
- 学习最佳实践

### 3. 社区交流
**分享和讨论**

- 参与Spring AI社区讨论
- 分享自己的实践经验
- 提出问题和建议

### 4. 持续学习
**AI技术日新月异**

- 关注Spring AI更新
- 学习最新的AI技术
- 扩展到其他AI框架

---

## 🌟 致谢

感谢Spring AI团队提供了如此优秀的框架！

---

## 📞 联系方式

如果你在学习过程中遇到问题或有改进建议，欢迎：
- 提交Issue
- 参与讨论
- 贡献代码

---

**祝你学习愉快！🚀**

> "The best way to learn is by doing."  
> 最好的学习方式就是动手实践。

---

**最后更新时间**：2025年10月5日

