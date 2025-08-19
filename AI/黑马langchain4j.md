学习AI相关的内容



# LangChain4j

来自黑马的课程：

【黑马程序员LangChain4j从入门到实战项目全套视频课程，涵盖LangChain4j+ollama+RAG，Java传统项目AI智能化升级】 https://www.bilibili.com/video/BV1sDMqzpEQ3/?p=6&share_source=copy_web&vd_source=184246d521185707999f94e18a91519f



## 常见请求参数

请求curl如下（这里以deepseek官方api为例）

```powershell
curl https://api.deepseek.com/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <DeepSeek API Key>" \
  -d '{
        "model": "deepseek-chat",
        "messages": [
          {"role": "system", "content": "You are a helpful assistant."},
          {"role": "user", "content": "Hello!"}
        ],
        "stream": false
      }'
```

**请求体**中的参数如下：

```json
{
    "model": "deepseek-chat", // 指定模型
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."}, // 系统信息，指定角色等
        {"role": "user", "content": "Hello!"}, // 用户的消息
        {"role": "assistant", "content": "Hello! How can I assist you today? 😊"} // 模型的消息，对话历史
    ],
    "stream": false // 流式传输 或 阻塞式传输
}
```

关于**流式传输**：

- 如果不启用，则会阻塞请求，直到所有内容生成完毕，再一次性返回；
- 如果启用，则会不断返回数据，会收到多次响应，每次响应一个分词；可以在Postman中明显看出；

还有很多其他参数，具体看模型的官方文档；

教程里说可以加上下面的参数开启联网功能，但是实测deepseek官方api并不行：

```json
"enable_search": true
```



## 常见响应信息

响应体如下：

```json
{
    "id": "2bebb3ff-ef43-4a5d-b6d2-66ab46eb9005",
    "object": "chat.completion",
    "created": 1755613707,
    "model": "deepseek-chat",
    "choices": [ // 大模型的响应信息数组
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Hello! How can I help you today? 😊"
            },
            "logprobs": null,
            "finish_reason": "stop"
        }
    ],
    "usage": {  // 本地会话的token细节
        "prompt_tokens": 12,
        "completion_tokens": 11,
        "total_tokens": 23,
        "prompt_tokens_details": {
            "cached_tokens": 0
        },
        "prompt_cache_hit_tokens": 0,
        "prompt_cache_miss_tokens": 12
    },
    "system_fingerprint": "fp_baeac5aaa3_prod0623_fp8_kvcache"
}
```



## 原生依赖基本使用

官方文档：https://docs.langchain4j.dev/

### 引入依赖

截止 2025-08-19 最新依赖版本是 `1.3.0`

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <version>1.3.0</version>
</dependency>
```

### 构建OpenAiChatModel

构建 `OpenAiChatModel` 对象后调用 `chat` 方法即可；

```java
OpenAiChatModel model = OpenAiChatModel.builder()
    .baseUrl("https://api.deepseek.com")
    .apiKey(System.getenv("DEEPSEEK_API_KEY"))
    .modelName("deepseek-chat")
    .logRequests(true) // 打印请求日志
    .logResponses(true) // 打印响应日志
    .build();
String responseMessage = model.chat("早上好！"); // 接收模型的回复
```

> 教程里说打印日志需要引入日志依赖，例如 `logback`，但是我没有引入也可以成功打印日志；应该是因为`spring-boot-starter-web`里有日志依赖，而教程一开始演示的是普通maven项目；



## 整合SpringBoot基本使用

### 引入依赖

截止 2025-08-19 最新依赖版本是 `1.3.0-beta9`

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>1.3.0-beta9</version>
</dependency>
```

### 修改配置

也是基本的几个信息

```properties
langchain4j.open-ai.chat-model.base-url=https://api.deepseek.com
langchain4j.open-ai.chat-model.api-key=${DEEPSEEK_API_KEY}
langchain4j.open-ai.chat-model.model-name=deepseek-chat
langchain4j.open-ai.chat-model.log-requests=true
langchain4j.open-ai.chat-model.log-responses=true

logging.level.dev.langchain4j = debug
```

### 注入OpenAiChatModel

直接依赖注入 `OpenAiChatModel`，然后调用即可；

```java
@RestController
public class ChatController {
    @Autowired
    OpenAiChatModel model;

    @PostMapping("/chat")
    public String chat(@RequestBody String message) {
        return model.chat(message);
    }
}
```

请求：

```powershell
curl --location 'http://localhost:8080/chat' \
--header 'Content-Type: text/plain' \
--data '你好呀'
```

