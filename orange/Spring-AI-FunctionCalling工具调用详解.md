# Spring AI Function Calling 工具调用详解

## 📋 目录
- [概述](#概述)
- [核心概念](#核心概念)
- [Tool注解](#tool注解)
- [FunctionToolCallback](#functiontoolcallback)
- [MethodToolCallback](#methodtoolcallback)
- [工具定义](#工具定义)
- [工具上下文](#工具上下文)
- [ChatClient集成](#chatclient集成)
- [高级特性](#高级特性)
- [实战场景](#实战场景)
- [最佳实践](#最佳实践)

---

## 概述

### 什么是Function Calling（工具调用）？

**Function Calling** 是让AI模型能够**调用外部工具**的能力，使AI从"只能对话"变成"可以行动"。

```
传统对话:
用户: "北京今天天气怎么样？"
AI: "对不起，我无法获取实时天气信息。"

Function Calling:
用户: "北京今天天气怎么样？"
AI: 识别需要调用天气工具
  → 调用 getWeather("北京")
  → 获取结果: "晴天，25°C"
AI: "北京今天天气晴朗，温度25°C。"
```

### 为什么需要Function Calling？

#### LLM的局限性

1. **知识截止**: 无法获取实时信息
2. **无法操作**: 不能执行实际操作
3. **数据隔离**: 无法访问私有数据
4. **计算受限**: 复杂计算能力有限

#### Function Calling的优势

1. ✅ **实时数据访问**: 调用API获取最新信息
2. ✅ **执行操作**: 发送邮件、创建订单等
3. ✅ **数据库查询**: 访问企业数据库
4. ✅ **系统集成**: 与现有系统对接
5. ✅ **扩展能力**: 赋予AI更多技能

### 应用场景

- 🌤️ **天气查询**: 获取实时天气
- 📊 **数据查询**: 查询数据库
- 📧 **发送邮件**: 自动发送通知
- 🔍 **搜索引擎**: 联网搜索信息
- 🧮 **计算器**: 执行复杂计算
- 📅 **日程管理**: 创建/查询日程
- 💰 **订单处理**: 创建/查询订单
- 🔧 **系统控制**: 控制IoT设备

---

## 核心概念

### 1. 工具（Tool）

工具是AI可以调用的函数，包含：
- **名称**: 唯一标识
- **描述**: 告诉AI何时使用
- **参数**: 输入定义
- **返回值**: 输出格式

### 2. 工具调用流程

```
用户输入
  ↓
AI判断是否需要工具
  ↓  需要
工具调用请求（JSON）
  ↓
Spring AI执行工具
  ↓
返回结果给AI
  ↓
AI基于结果生成回答
  ↓
返回给用户
```

### 3. 核心接口

#### ToolCallback（工具回调）

```java
/**
 * 工具回调接口
 * 代表一个可被AI模型调用的工具
 */
public interface ToolCallback {
    
    /**
     * 工具定义
     * AI用它判断何时、如何调用工具
     */
    ToolDefinition getToolDefinition();
    
    /**
     * 工具元数据
     * 提供额外的处理信息
     */
    default ToolMetadata getToolMetadata() {
        return ToolMetadata.builder().build();
    }
    
    /**
     * 执行工具
     * @param toolInput JSON格式的输入
     * @return 字符串格式的输出
     */
    String call(String toolInput);
    
    /**
     * 执行工具（带上下文）
     * @param toolInput JSON格式的输入
     * @param toolContext 工具上下文
     * @return 字符串格式的输出
     */
    default String call(String toolInput, @Nullable ToolContext toolContext) {
        return call(toolInput);
    }
}
```

#### ToolDefinition（工具定义）

```java
/**
 * 工具定义接口
 * AI用它理解工具
 */
public interface ToolDefinition {
    
    /**
     * 工具名称
     * 必须唯一
     */
    String name();
    
    /**
     * 工具描述
     * AI用它判断何时使用工具
     */
    String description();
    
    /**
     * 输入参数的JSON Schema
     * 定义工具接受的参数格式
     */
    String inputSchema();
}
```

#### ToolMetadata（工具元数据）

```java
/**
 * 工具元数据
 */
public record ToolMetadata(
    /**
     * 是否直接返回工具结果
     * true: 直接返回工具结果
     * false: 将结果传回AI生成回答
     */
    boolean returnDirect
) {
    public static Builder builder() {
        return new Builder();
    }
    
    public static class Builder {
        private boolean returnDirect = false;
        
        public Builder returnDirect(boolean returnDirect) {
            this.returnDirect = returnDirect;
            return this;
        }
        
        public ToolMetadata build() {
            return new ToolMetadata(returnDirect);
        }
    }
}
```

---

## Tool注解

### @Tool注解定义

```java
/**
 * 标记方法为工具
 * Spring AI会自动将这些方法转换为可调用的工具
 */
@Target({ ElementType.METHOD, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Tool {
    
    /**
     * 工具名称
     * 如果不提供，使用方法名
     * 
     * 建议：只使用字母数字、下划线、连字符、点
     * 推荐: "get_weather", "search-docs", "tool.v1"
     * 避免: "get weather" (空格), "tool()" (括号)
     */
    String name() default "";
    
    /**
     * 工具描述
     * 如果不提供，使用方法名
     */
    String description() default "";
    
    /**
     * 是否直接返回工具结果
     * false: 结果传回AI生成回答（默认）
     * true: 直接返回工具结果
     */
    boolean returnDirect() default false;
    
    /**
     * 结果转换器类
     * 将工具返回值转换为字符串
     */
    Class<? extends ToolCallResultConverter> resultConverter() 
        default DefaultToolCallResultConverter.class;
}
```

### @ToolParam注解

```java
/**
 * 标记工具参数
 */
@Target({ ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ToolParam {
    
    /**
     * 参数是否必需
     */
    boolean required() default true;
    
    /**
     * 参数描述
     */
    String description() default "";
}
```

### 基本用法

```java
@Component
public class WeatherTools {
    
    /**
     * 最简单的工具
     */
    @Tool(description = "获取城市的当前天气")
    public String getWeather(
            @ToolParam(description = "城市名称") String city) {
        
        // 调用天气API
        return "北京: 晴天, 25°C";
    }
    
    /**
     * 带多个参数的工具
     */
    @Tool(
        name = "get_weather_forecast",
        description = "获取城市的天气预报"
    )
    public String getWeatherForecast(
            @ToolParam(description = "城市名称") String city,
            @ToolParam(description = "预报天数", required = false) Integer days) {
        
        if (days == null) {
            days = 3;  // 默认3天
        }
        
        // 调用天气API
        return String.format("北京未来%d天天气预报：...", days);
    }
    
    /**
     * 返回对象的工具
     */
    @Tool(description = "获取详细天气信息")
    public WeatherInfo getDetailedWeather(
            @ToolParam(description = "城市名称") String city) {
        
        return new WeatherInfo(
            city,
            25.0,
            "晴天",
            60,
            5.2
        );
    }
    
    record WeatherInfo(
        String city,
        double temperature,
        String condition,
        int humidity,
        double windSpeed
    ) {}
}
```

### 配置工具

```java
@Configuration
public class ToolConfig {
    
    /**
     * 注册工具Bean
     */
    @Bean
    public MethodToolCallbackProvider weatherToolsProvider(
            WeatherTools weatherTools) {
        
        return new MethodToolCallbackProvider(weatherTools);
    }
    
    /**
     * 或者使用自动扫描
     */
    @Bean
    public MethodToolCallbackProvider autoScanTools(
            ApplicationContext context) {
        
        // 自动发现所有带@Component的工具类
        return new MethodToolCallbackProvider(context);
    }
}
```

---

## FunctionToolCallback

### 概述

`FunctionToolCallback` 允许使用**函数式编程**方式定义工具。

### 基本用法

```java
@Configuration
public class FunctionToolConfig {
    
    /**
     * 使用Function定义工具
     */
    @Bean
    public FunctionToolCallback weatherTool() {
        
        Function<WeatherRequest, WeatherResponse> weatherFunction = 
            request -> {
                // 调用天气API
                return new WeatherResponse(
                    request.city(),
                    25.0,
                    "晴天"
                );
            };
        
        return FunctionToolCallback.builder("get_weather", weatherFunction)
            .description("获取城市当前天气")
            .inputType(WeatherRequest.class)
            .build();
    }
    
    /**
     * 使用BiFunction（带上下文）
     */
    @Bean
    public FunctionToolCallback contextualTool() {
        
        BiFunction<ToolRequest, ToolContext, ToolResponse> function = 
            (request, context) -> {
                // 使用上下文信息
                String userId = (String) context.getContext().get("user_id");
                
                // 执行操作
                return new ToolResponse("成功");
            };
        
        return FunctionToolCallback.builder("contextual_tool", function)
            .description("需要上下文的工具")
            .inputType(ToolRequest.class)
            .build();
    }
    
    /**
     * 使用Supplier（无参数）
     */
    @Bean
    public FunctionToolCallback currentTimeTool() {
        
        Supplier<String> timeSupplier = () -> 
            LocalDateTime.now().toString();
        
        return FunctionToolCallback.builder("get_current_time", timeSupplier)
            .description("获取当前时间")
            .build();
    }
    
    /**
     * 使用Consumer（无返回值）
     */
    @Bean
    public FunctionToolCallback logTool() {
        
        Consumer<LogRequest> logConsumer = request -> 
            logger.info("Log: {}", request.message());
        
        return FunctionToolCallback.builder("log_message", logConsumer)
            .description("记录日志消息")
            .inputType(LogRequest.class)
            .build();
    }
    
    record WeatherRequest(String city) {}
    record WeatherResponse(String city, double temperature, String condition) {}
    record ToolRequest(String action) {}
    record ToolResponse(String result) {}
    record LogRequest(String message) {}
}
```

### Builder API

```java
public class FunctionToolCallbackBuilder {
    
    /**
     * 从Function创建
     */
    public static <I, O> Builder<I, O> builder(
            String name,
            Function<I, O> function) {
        
        return new Builder<>(name, (input, context) -> function.apply(input));
    }
    
    /**
     * 从BiFunction创建
     */
    public static <I, O> Builder<I, O> builder(
            String name,
            BiFunction<I, ToolContext, O> function) {
        
        return new Builder<>(name, function);
    }
    
    /**
     * 从Supplier创建
     */
    public static <O> Builder<Void, O> builder(
            String name,
            Supplier<O> supplier) {
        
        Function<Void, O> function = input -> supplier.get();
        return builder(name, function).inputType(Void.class);
    }
    
    /**
     * 从Consumer创建
     */
    public static <I> Builder<I, Void> builder(
            String name,
            Consumer<I> consumer) {
        
        Function<I, Void> function = input -> {
            consumer.accept(input);
            return null;
        };
        return builder(name, function);
    }
}
```

---

## MethodToolCallback

### 概述

`MethodToolCallback` 基于反射调用Java方法，是`@Tool`注解的底层实现。

### 实现原理

```java
/**
 * 方法工具回调
 * 通过反射调用Java方法
 */
public class MethodToolCallback implements ToolCallback {
    
    private final ToolDefinition toolDefinition;
    private final ToolMetadata toolMetadata;
    private final Method toolMethod;
    private final Object toolObject;
    private final ToolCallResultConverter toolCallResultConverter;
    
    @Override
    public String call(String toolInput, @Nullable ToolContext toolContext) {
        
        // 1. 验证上下文支持
        validateToolContextSupport(toolContext);
        
        // 2. 解析工具参数
        Map<String, Object> toolArguments = extractToolArguments(toolInput);
        
        // 3. 构建方法参数
        Object[] methodArguments = buildMethodArguments(
            toolArguments,
            toolContext
        );
        
        // 4. 调用方法
        Object result = callMethod(methodArguments);
        
        // 5. 转换结果
        Type returnType = toolMethod.getGenericReturnType();
        return toolCallResultConverter.convert(result, returnType);
    }
    
    private Object callMethod(Object[] methodArguments) {
        try {
            return toolMethod.invoke(toolObject, methodArguments);
        } catch (IllegalAccessException | InvocationTargetException ex) {
            throw new ToolExecutionException(toolDefinition, ex);
        }
    }
    
    private Object[] buildMethodArguments(
            Map<String, Object> toolInputArguments,
            @Nullable ToolContext toolContext) {
        
        return Stream.of(toolMethod.getParameters())
            .map(parameter -> {
                // 如果参数是ToolContext
                if (parameter.getType().isAssignableFrom(ToolContext.class)) {
                    return toolContext;
                }
                
                // 从工具输入中获取参数
                Object rawArgument = toolInputArguments.get(
                    parameter.getName()
                );
                
                return buildTypedArgument(
                    rawArgument,
                    parameter.getParameterizedType()
                );
            })
            .toArray();
    }
}
```

---

## 工具定义

### 手动创建工具定义

```java
/**
 * 手动构建工具定义
 */
public class ManualToolDefinition {
    
    public static ToolDefinition weatherToolDefinition() {
        
        return ToolDefinition.builder()
            .name("get_weather")
            .description("获取指定城市的当前天气信息")
            .inputSchema("""
                {
                    "type": "object",
                    "properties": {
                        "city": {
                            "type": "string",
                            "description": "城市名称，如：北京、上海"
                        },
                        "unit": {
                            "type": "string",
                            "enum": ["celsius", "fahrenheit"],
                            "description": "温度单位"
                        }
                    },
                    "required": ["city"]
                }
                """)
            .build();
    }
}
```

### 从类自动生成

```java
/**
 * 从类自动生成工具定义
 */
public class AutoToolDefinition {
    
    public static ToolDefinition fromClass(Class<?> clazz) {
        
        return ToolDefinition.builder()
            .name(clazz.getSimpleName())
            .description("Tool for " + clazz.getSimpleName())
            .inputSchema(generateJsonSchema(clazz))
            .build();
    }
    
    private static String generateJsonSchema(Class<?> clazz) {
        // 使用Jackson或其他JSON Schema生成器
        return "{}";  // 简化
    }
}
```

---

## 工具上下文

### ToolContext定义

```java
/**
 * 工具上下文
 * 在工具执行期间传递额外信息
 */
public class ToolContext {
    
    /**
     * 对话历史的键
     */
    public static final String TOOL_CALL_HISTORY = "tool_call_history";
    
    private final Map<String, Object> context;
    
    public ToolContext(Map<String, Object> context) {
        this.context = context;
    }
    
    public Map<String, Object> getContext() {
        return context;
    }
    
    public Object get(String key) {
        return context.get(key);
    }
}
```

### 使用工具上下文

```java
@Component
public class ContextualTools {
    
    /**
     * 使用上下文的工具
     */
    @Tool(description = "发送个性化消息")
    public String sendMessage(
            @ToolParam(description = "消息内容") String message,
            ToolContext context) {
        
        // 从上下文获取用户信息
        String userId = (String) context.get("user_id");
        String userName = (String) context.get("user_name");
        
        // 使用用户信息
        String personalizedMessage = String.format(
            "亲爱的%s，%s",
            userName,
            message
        );
        
        // 发送消息
        sendToUser(userId, personalizedMessage);
        
        return "消息已发送";
    }
    
    /**
     * 访问对话历史
     */
    @Tool(description = "根据历史提供建议")
    public String provideRecommendation(
            @ToolParam(description = "类别") String category,
            ToolContext context) {
        
        // 获取对话历史
        @SuppressWarnings("unchecked")
        List<Message> history = (List<Message>) context.get(
            ToolContext.TOOL_CALL_HISTORY
        );
        
        // 基于历史生成建议
        return generateRecommendation(category, history);
    }
    
    private void sendToUser(String userId, String message) {
        // 实现发送逻辑
    }
    
    private String generateRecommendation(
            String category,
            List<Message> history) {
        // 实现推荐逻辑
        return "推荐内容";
    }
}
```

---

## ChatClient集成

### 基本集成

```java
@Service
public class ChatClientWithTools {
    
    private final ChatClient chatClient;
    
    public ChatClientWithTools(
            ChatModel chatModel,
            WeatherTools weatherTools) {
        
        this.chatClient = ChatClient.builder(chatModel)
            // 全局注册工具
            .defaultTools(
                new MethodToolCallbackProvider(weatherTools)
                    .getToolCallbacks()
            )
            .build();
    }
    
    /**
     * 使用全局工具
     */
    public String chat(String message) {
        return chatClient
            .prompt(message)
            .call()
            .content();
    }
    
    /**
     * 使用请求级工具
     */
    public String chatWithSpecificTools(String message) {
        return chatClient
            .prompt(message)
            .tools("get_weather", "get_weather_forecast")  // 指定工具
            .call()
            .content();
    }
    
    /**
     * 带工具上下文
     */
    public String chatWithContext(String message, String userId) {
        return chatClient
            .prompt(message)
            .toolContext(Map.of(
                "user_id", userId,
                "user_name", getUserName(userId)
            ))
            .call()
            .content();
    }
    
    private String getUserName(String userId) {
        // 查询用户名
        return "用户" + userId;
    }
}
```

### 完整示例

```java
@Service
public class CompleteFunctionCallingExample {
    
    private final ChatClient chatClient;
    
    @Autowired
    public CompleteFunctionCallingExample(
            ChatModel chatModel,
            WeatherTools weatherTools,
            DatabaseTools databaseTools,
            EmailTools emailTools) {
        
        // 创建工具提供者
        MethodToolCallbackProvider weatherProvider = 
            new MethodToolCallbackProvider(weatherTools);
        
        MethodToolCallbackProvider databaseProvider = 
            new MethodToolCallbackProvider(databaseTools);
        
        MethodToolCallbackProvider emailProvider = 
            new MethodToolCallbackProvider(emailTools);
        
        // 合并所有工具
        List<ToolCallback> allTools = Stream.of(
            weatherProvider.getToolCallbacks(),
            databaseProvider.getToolCallbacks(),
            emailProvider.getToolCallbacks()
        )
        .flatMap(Arrays::stream)
        .toList();
        
        // 创建ChatClient
        this.chatClient = ChatClient.builder(chatModel)
            .defaultTools(allTools.toArray(ToolCallback[]::new))
            .build();
    }
    
    /**
     * 智能助手
     */
    public String assist(String userMessage, Map<String, Object> context) {
        
        return chatClient
            .prompt(spec -> spec
                .system("""
                    你是一个智能助手。
                    你可以使用工具来：
                    - 查询天气
                    - 查询数据库
                    - 发送邮件
                    
                    根据用户需求选择合适的工具。
                    """)
                .user(userMessage)
                .toolContext(context)
            )
            .call()
            .content();
    }
}
```

---

## 高级特性

### 1. 条件工具选择

```java
@Service
public class ConditionalToolService {
    
    private final ChatClient chatClient;
    
    /**
     * 根据条件选择工具
     */
    public String chatWithConditionalTools(
            String message,
            UserRole role) {
        
        // 根据角色选择可用工具
        List<String> availableTools = getToolsForRole(role);
        
        return chatClient
            .prompt(message)
            .tools(availableTools.toArray(String[]::new))
            .call()
            .content();
    }
    
    private List<String> getToolsForRole(UserRole role) {
        return switch (role) {
            case ADMIN -> List.of(
                "get_weather",
                "query_database",
                "send_email",
                "manage_users"
            );
            case USER -> List.of(
                "get_weather",
                "query_own_data"
            );
            case GUEST -> List.of(
                "get_weather"
            );
        };
    }
    
    enum UserRole {
        ADMIN, USER, GUEST
    }
}
```

### 2. 工具链

```java
@Component
public class ChainedTools {
    
    /**
     * 工具1：搜索产品
     */
    @Tool(description = "搜索产品")
    public List<Product> searchProducts(
            @ToolParam(description = "搜索关键词") String keyword) {
        
        // 搜索逻辑
        return List.of(
            new Product("P001", "产品A", 99.99),
            new Product("P002", "产品B", 149.99)
        );
    }
    
    /**
     * 工具2：获取产品详情
     */
    @Tool(description = "获取产品详细信息")
    public ProductDetail getProductDetail(
            @ToolParam(description = "产品ID") String productId) {
        
        // 查询详情
        return new ProductDetail(
            productId,
            "详细描述",
            99.99,
            100
        );
    }
    
    /**
     * 工具3：创建订单
     */
    @Tool(description = "创建订单")
    public Order createOrder(
            @ToolParam(description = "产品ID") String productId,
            @ToolParam(description = "数量") int quantity,
            ToolContext context) {
        
        String userId = (String) context.get("user_id");
        
        // 创建订单逻辑
        return new Order(
            "O" + System.currentTimeMillis(),
            userId,
            productId,
            quantity
        );
    }
    
    record Product(String id, String name, double price) {}
    record ProductDetail(String id, String description, double price, int stock) {}
    record Order(String orderId, String userId, String productId, int quantity) {}
}
```

### 3. 异步工具

```java
@Component
public class AsyncTools {
    
    /**
     * 异步工具
     */
    @Tool(description = "异步处理大任务")
    public CompletableFuture<String> processLargeTask(
            @ToolParam(description = "任务ID") String taskId) {
        
        return CompletableFuture.supplyAsync(() -> {
            // 模拟长时间处理
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            
            return "任务 " + taskId + " 处理完成";
        });
    }
    
    /**
     * 批量异步工具
     */
    @Tool(description = "批量处理")
    public List<CompletableFuture<String>> batchProcess(
            @ToolParam(description = "任务列表") List<String> tasks) {
        
        return tasks.stream()
            .map(task -> CompletableFuture.supplyAsync(() -> 
                process(task)
            ))
            .toList();
    }
    
    private String process(String task) {
        // 处理逻辑
        return "处理完成: " + task;
    }
}
```

### 4. 错误处理

```java
@Component
public class RobustTools {
    
    /**
     * 带错误处理的工具
     */
    @Tool(description = "安全的数据库查询")
    public String safeQuery(
            @ToolParam(description = "SQL查询") String query) {
        
        try {
            // 验证查询
            if (!isValidQuery(query)) {
                return "错误：无效的查询语句";
            }
            
            // 执行查询
            return executeQuery(query);
            
        } catch (SQLException e) {
            logger.error("查询失败", e);
            return "错误：查询执行失败 - " + e.getMessage();
        } catch (Exception e) {
            logger.error("意外错误", e);
            return "错误：系统异常";
        }
    }
    
    /**
     * 带重试的工具
     */
    @Tool(description = "可重试的API调用")
    public String reliableApiCall(
            @ToolParam(description = "API端点") String endpoint) {
        
        int maxRetries = 3;
        int attempt = 0;
        
        while (attempt < maxRetries) {
            try {
                return callExternalApi(endpoint);
            } catch (Exception e) {
                attempt++;
                if (attempt >= maxRetries) {
                    return "错误：API调用失败，已重试" + maxRetries + "次";
                }
                
                // 等待后重试
                try {
                    Thread.sleep(1000 * attempt);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
        
        return "错误：未知错误";
    }
    
    private boolean isValidQuery(String query) {
        // 验证逻辑
        return !query.toLowerCase().contains("drop");
    }
    
    private String executeQuery(String query) throws SQLException {
        // 执行逻辑
        return "查询结果";
    }
    
    private String callExternalApi(String endpoint) {
        // API调用逻辑
        return "API响应";
    }
}
```

---

## 实战场景

### 1. 智能客服系统

```java
@Component
public class CustomerServiceTools {
    
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final TicketService ticketService;
    
    /**
     * 查询订单状态
     */
    @Tool(description = "查询订单状态")
    public OrderStatus queryOrderStatus(
            @ToolParam(description = "订单号") String orderId,
            ToolContext context) {
        
        String userId = (String) context.get("user_id");
        
        // 验证权限
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ToolExecutionException(
                "订单不存在：" + orderId
            ));
        
        if (!order.getUserId().equals(userId)) {
            throw new ToolExecutionException("无权限查询此订单");
        }
        
        return new OrderStatus(
            orderId,
            order.getStatus(),
            order.getTrackingNumber(),
            order.getEstimatedDelivery()
        );
    }
    
    /**
     * 创建工单
     */
    @Tool(description = "创建客服工单")
    public Ticket createTicket(
            @ToolParam(description = "问题类型") String issueType,
            @ToolParam(description = "问题描述") String description,
            ToolContext context) {
        
        String userId = (String) context.get("user_id");
        
        Ticket ticket = ticketService.create(
            userId,
            issueType,
            description
        );
        
        return ticket;
    }
    
    /**
     * 申请退款
     */
    @Tool(description = "申请订单退款")
    public RefundApplication requestRefund(
            @ToolParam(description = "订单号") String orderId,
            @ToolParam(description = "退款原因") String reason,
            ToolContext context) {
        
        String userId = (String) context.get("user_id");
        
        // 验证订单
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ToolExecutionException("订单不存在"));
        
        if (!order.getUserId().equals(userId)) {
            throw new ToolExecutionException("无权限操作此订单");
        }
        
        // 检查是否可退款
        if (!order.isRefundable()) {
            throw new ToolExecutionException("订单不符合退款条件");
        }
        
        // 创建退款申请
        return refundService.createApplication(orderId, reason);
    }
    
    record OrderStatus(
        String orderId,
        String status,
        String trackingNumber,
        LocalDateTime estimatedDelivery
    ) {}
}

@Service
public class CustomerServiceChatbot {
    
    private final ChatClient chatClient;
    
    public String chat(String userMessage, String userId) {
        
        return chatClient
            .prompt(spec -> spec
                .system("""
                    你是一个专业的客服助手。
                    你可以帮助用户：
                    - 查询订单状态
                    - 创建客服工单
                    - 申请退款
                    
                    请礼貌、专业地回答用户问题。
                    """)
                .user(userMessage)
                .toolContext(Map.of("user_id", userId))
            )
            .call()
            .content();
    }
}
```

### 2. 数据分析助手

```java
@Component
public class DataAnalysisTools {
    
    private final JdbcTemplate jdbcTemplate;
    
    /**
     * 执行SQL查询
     */
    @Tool(description = "执行SQL查询获取数据")
    public QueryResult executeQuery(
            @ToolParam(description = "SQL查询语句") String sql) {
        
        // 安全检查
        if (!isSafeQuery(sql)) {
            throw new ToolExecutionException("不允许的SQL操作");
        }
        
        try {
            List<Map<String, Object>> rows = jdbcTemplate.queryForList(sql);
            
            return new QueryResult(
                rows,
                rows.size(),
                "success"
            );
            
        } catch (Exception e) {
            return new QueryResult(
                List.of(),
                0,
                "错误: " + e.getMessage()
            );
        }
    }
    
    /**
     * 生成统计报告
     */
    @Tool(description = "生成数据统计报告")
    public StatisticsReport generateReport(
            @ToolParam(description = "报告类型") String reportType,
            @ToolParam(description = "开始日期") String startDate,
            @ToolParam(description = "结束日期") String endDate) {
        
        String sql = buildReportQuery(reportType, startDate, endDate);
        
        List<Map<String, Object>> data = jdbcTemplate.queryForList(sql);
        
        return new StatisticsReport(
            reportType,
            startDate,
            endDate,
            calculateStatistics(data)
        );
    }
    
    /**
     * 数据可视化建议
     */
    @Tool(description = "建议合适的数据可视化图表")
    public VisualizationSuggestion suggestVisualization(
            @ToolParam(description = "数据类型") String dataType,
            @ToolParam(description = "数据维度") int dimensions) {
        
        String chartType = switch (dataType.toLowerCase()) {
            case "time_series" -> "折线图";
            case "category" -> dimensions <= 5 ? "饼图" : "柱状图";
            case "comparison" -> "柱状图";
            case "distribution" -> "直方图";
            default -> "表格";
        };
        
        return new VisualizationSuggestion(
            chartType,
            "建议使用" + chartType + "展示数据"
        );
    }
    
    private boolean isSafeQuery(String sql) {
        String lowerSql = sql.toLowerCase();
        return lowerSql.startsWith("select") &&
               !lowerSql.contains("drop") &&
               !lowerSql.contains("delete") &&
               !lowerSql.contains("update") &&
               !lowerSql.contains("insert");
    }
    
    private String buildReportQuery(
            String reportType,
            String startDate,
            String endDate) {
        // 构建查询
        return "SELECT * FROM reports WHERE type = '" + reportType + "'";
    }
    
    private Map<String, Object> calculateStatistics(
            List<Map<String, Object>> data) {
        // 计算统计信息
        return Map.of(
            "count", data.size(),
            "mean", 0.0,
            "median", 0.0
        );
    }
    
    record QueryResult(
        List<Map<String, Object>> data,
        int rowCount,
        String status
    ) {}
    
    record StatisticsReport(
        String reportType,
        String startDate,
        String endDate,
        Map<String, Object> statistics
    ) {}
    
    record VisualizationSuggestion(
        String chartType,
        String reason
    ) {}
}
```

### 3. 自动化运维助手

```java
@Component
public class DevOpsTools {
    
    /**
     * 检查服务状态
     */
    @Tool(description = "检查服务健康状态")
    public ServiceStatus checkServiceStatus(
            @ToolParam(description = "服务名称") String serviceName) {
        
        try {
            HttpResponse response = httpClient.get(
                "http://" + serviceName + "/actuator/health"
            );
            
            return new ServiceStatus(
                serviceName,
                response.getStatusCode() == 200 ? "UP" : "DOWN",
                response.getBody()
            );
            
        } catch (Exception e) {
            return new ServiceStatus(
                serviceName,
                "ERROR",
                e.getMessage()
            );
        }
    }
    
    /**
     * 重启服务
     */
    @Tool(description = "重启指定服务")
    public RestartResult restartService(
            @ToolParam(description = "服务名称") String serviceName,
            ToolContext context) {
        
        // 权限检查
        String userId = (String) context.get("user_id");
        if (!hasRestartPermission(userId)) {
            throw new ToolExecutionException("无权限重启服务");
        }
        
        try {
            // 执行重启
            kubernetesClient.restartDeployment(serviceName);
            
            // 等待服务启动
            Thread.sleep(5000);
            
            // 检查状态
            ServiceStatus status = checkServiceStatus(serviceName);
            
            return new RestartResult(
                serviceName,
                status.status().equals("UP"),
                "服务已重启"
            );
            
        } catch (Exception e) {
            return new RestartResult(
                serviceName,
                false,
                "重启失败: " + e.getMessage()
            );
        }
    }
    
    /**
     * 查询日志
     */
    @Tool(description = "查询服务日志")
    public LogQueryResult queryLogs(
            @ToolParam(description = "服务名称") String serviceName,
            @ToolParam(description = "关键词") String keyword,
            @ToolParam(description = "最近N分钟", required = false) Integer minutes) {
        
        if (minutes == null) {
            minutes = 10;
        }
        
        try {
            List<String> logs = kubernetesClient.getLogs(
                serviceName,
                minutes,
                keyword
            );
            
            return new LogQueryResult(
                serviceName,
                logs,
                logs.size()
            );
            
        } catch (Exception e) {
            return new LogQueryResult(
                serviceName,
                List.of("错误: " + e.getMessage()),
                0
            );
        }
    }
    
    private boolean hasRestartPermission(String userId) {
        // 检查权限
        return true;  // 简化
    }
    
    record ServiceStatus(String serviceName, String status, String details) {}
    record RestartResult(String serviceName, boolean success, String message) {}
    record LogQueryResult(String serviceName, List<String> logs, int count) {}
}
```

---

## 最佳实践

### 1. 工具命名规范

```java
/**
 * 好的工具名：
 * - get_weather
 * - search_products
 * - create_order
 * - user.get_profile
 * 
 * 避免的名称：
 * - "get weather" (包含空格)
 * - "search()" (包含括号)
 * - "产品搜索" (中文，可能不兼容)
 */
```

### 2. 工具描述最佳实践

```java
@Component
public class WellDescribedTools {
    
    /**
     * ❌ 不好的描述
     */
    @Tool(description = "天气")
    public String badWeatherTool(String city) {
        return "晴天";
    }
    
    /**
     * ✅ 好的描述
     */
    @Tool(description = """
        获取指定城市的当前天气信息，包括：
        - 温度（摄氏度）
        - 天气状况（晴/雨/雪等）
        - 湿度
        - 风速
        
        支持中国主要城市。
        """)
    public WeatherInfo goodWeatherTool(
            @ToolParam(description = "城市名称，如：北京、上海、深圳") 
            String city) {
        
        return new WeatherInfo(city, 25.0, "晴天", 60, 5.2);
    }
}
```

### 3. 参数验证

```java
@Component
public class ValidatedTools {
    
    @Tool(description = "创建用户")
    public User createUser(
            @ToolParam(description = "用户名（3-20字符）") String username,
            @ToolParam(description = "邮箱地址") String email,
            @ToolParam(description = "年龄（18-120）") int age) {
        
        // 参数验证
        if (username == null || username.length() < 3 || username.length() > 20) {
            throw new ToolExecutionException("用户名长度必须在3-20之间");
        }
        
        if (!isValidEmail(email)) {
            throw new ToolExecutionException("邮箱格式无效");
        }
        
        if (age < 18 || age > 120) {
            throw new ToolExecutionException("年龄必须在18-120之间");
        }
        
        // 创建用户
        return userService.create(username, email, age);
    }
    
    private boolean isValidEmail(String email) {
        return email != null && email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }
}
```

### 4. 异常处理

```java
@Component
public class SafeTools {
    
    @Tool(description = "安全的工具调用")
    public ToolResult safeTool(
            @ToolParam(description = "参数") String param) {
        
        try {
            // 执行操作
            String result = riskyOperation(param);
            
            return new ToolResult(true, result, null);
            
        } catch (BusinessException e) {
            // 业务异常 - 返回友好消息
            logger.warn("业务异常", e);
            return new ToolResult(
                false,
                null,
                "操作失败：" + e.getMessage()
            );
            
        } catch (Exception e) {
            // 系统异常 - 记录并返回通用消息
            logger.error("系统异常", e);
            return new ToolResult(
                false,
                null,
                "系统错误，请稍后重试"
            );
        }
    }
    
    private String riskyOperation(String param) throws BusinessException {
        // 可能抛出异常的操作
        return "结果";
    }
    
    record ToolResult(boolean success, String data, String error) {}
}
```

### 5. 性能优化

```java
@Component
public class OptimizedTools {
    
    /**
     * 缓存工具结果
     */
    @Tool(description = "获取配置")
    @Cacheable(value = "config", key = "#key")
    public String getConfig(
            @ToolParam(description = "配置键") String key) {
        
        // 从数据库加载配置（耗时操作）
        return configRepository.findByKey(key);
    }
    
    /**
     * 异步执行
     */
    @Tool(description = "发送通知")
    public String sendNotification(
            @ToolParam(description = "接收人") String recipient,
            @ToolParam(description = "消息") String message) {
        
        // 异步发送
        CompletableFuture.runAsync(() -> {
            notificationService.send(recipient, message);
        });
        
        // 立即返回
        return "通知已提交发送";
    }
    
    /**
     * 批量处理
     */
    @Tool(description = "批量查询用户")
    public List<User> batchGetUsers(
            @ToolParam(description = "用户ID列表") List<String> userIds) {
        
        // 一次查询所有用户（避免N+1问题）
        return userRepository.findAllById(userIds);
    }
}
```

---

## 总结

### Function Calling核心特点

1. **扩展AI能力**: 让AI可以调用外部工具
2. **实时数据**: 访问最新信息
3. **系统集成**: 与现有系统对接
4. **灵活定义**: 多种方式定义工具
5. **安全可控**: 权限和验证机制

### Spring AI Function Calling API

```
定义工具:
- @Tool注解 (声明式)
- FunctionToolCallback (函数式)
- MethodToolCallback (反射式)

集成ChatClient:
- defaultTools() (全局工具)
- tools() (请求级工具)
- toolContext() (传递上下文)

执行流程:
用户输入 → AI判断 → 调用工具 → 返回结果 → AI生成回答
```

### 最佳实践清单

- ✅ 使用清晰的工具名称和描述
- ✅ 提供详细的参数说明
- ✅ 实现参数验证
- ✅ 妥善处理异常
- ✅ 考虑性能优化（缓存、异步）
- ✅ 实现权限控制
- ✅ 记录工具调用日志
- ✅ 编写工具单元测试

通过Spring AI的Function Calling功能，你可以让AI助手具备真正的行动能力，构建智能客服、数据分析、运维自动化等强大应用！

---

**文档版本**: 1.0  
**最后更新**: 2025-10-05  
**Spring AI版本**: 1.1.0-SNAPSHOT

