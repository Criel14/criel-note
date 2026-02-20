# LangChain4j

学习自黑马的[LangChain4j课程](https://www.bilibili.com/video/BV1sDMqzpEQ3/?p=6&share_source=copy_web&vd_source=184246d521185707999f94e18a91519f)和LangChain4J的[官方中文文档](https://docs.langchain4j.info/)



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

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai</artifactId>
    <version>1.10.0-beta18</version>
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

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>1.10.0-beta18</version>
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



## 声明式服务

> 上文提到的为基本使用（白学），该部分才更贴近SpringBoot项目的开发思想；

需要引入新的依赖：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
    <version>1.10.0-beta18</version>
</dependency>
```

### AiService

在自定义**接口**上加上`@AiService`注解（将其视为标准的 Spring Boot `@Service`，但具有 AI 功能），然后**注入**这个Bean并调用其中的方法即可：

```java
// 接口名和方法名都随意
@AiService
public interface Assistant {
    String chat(String userMessage);  
}
```

当有多个AI服务时，需使用显式装配模式（`@AiService(wiringMode = EXPLICIT)`）指定要使用的组件：

```java
@AiService(wiringMode = EXPLICIT, chatModel = "openAiChatModel")
interface OpenAiAssistant {
    String chat(String userMessage);
}

@AiService(wiringMode = EXPLICIT, chatModel = "ollamaChatModel")
interface OllamaAssistant {
    String chat(String userMessage);
}
```

```properties
# OpenAI
langchain4j.open-ai.chat-model.api-key=${OPENAI_API_KEY}
langchain4j.open-ai.chat-model.model-name=gpt-4o-mini
...

# Ollama
langchain4j.ollama.chat-model.base-url=http://localhost:11434
langchain4j.ollama.chat-model.model-name=llama3.1
...
```



### 流式输出

流式输出分为`Flux`和`TokenStream`：

- `TokenStream` 是 LangChain4j AI Service 风格，回调式，配合SSE很简单；
- `Flux` 是 Reactor 响应式流风格；

> 二者其实都可以在 **阻塞式** 的SpringBoot 项目里使用；

首先需要配置一个流式传输的模型：

```yaml
# 大模型配置，这里用deepseek
langchain4j:
  open-ai:
    streaming-chat-model: # 流式传输模型
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      model-name: deepseek-chat
      log-requests: true
      log-responses: true
    chat-model: # 阻塞式模型
      api-key: ${DEEPSEEK_API_KEY}
      base-url: https://api.deepseek.com
      model-name: deepseek-chat
      log-requests: true
      log-responses: true
```

然后在`@AiService`里指定：

```java
@AiService(
        streamingChatModel = "openAiStreamingChatModel",
)
```



#### TokenStream 方式

将`AiService`的方法的返回值改成`LangChain4j`框架带的`TokenStream`；

```java
public interface Assistant {
    @SystemMessage("你是一个快递驿站的智能助手")
    TokenStream chat(@UserMessage String message);
}
```

在`Service`层创建一个返回值是`SseEmitter`的方法；

> SseEmitter是Spring提供的一个用于SSE响应的类，可以通过`.send()`回调发送数据，通过`.complete()`结束连接；

```java
public SseEmitter adminChat(String message) {
    SseEmitter emitter = new SseEmitter(0L); // 参数是timeout，为0表示永不超时
    TokenStream tokenStream = assistant.chat(message);

    // 写好tokenStream的逻辑后启动
    tokenStream
        .onPartialResponse( /* 每个分片token到达时触发的逻辑*/ )
        .onCompleteResponse( /* 模型完整结束时触发的逻辑 */ )
        .onError(emitter::completeWithError)
        .start();

    return emitter;
}
```

`TokenStream`有很多方法，具体看https://docs.langchain4j.dev/tutorials/ai-services#streaming，但必要的方法就是上面代码中的四个；

具体示例：

```java
MediaType UTF8_TEXT = new MediaType("text", "plain", StandardCharsets.UTF_8); // UTF8防中文乱码

tokenStream
    .onPartialResponse(partial -> { // 每个分片 token 到达时触发
        try {
            emitter.send(
                SseEmitter.event() // 创建一个SSE事件
                .name("token") // 事件名称
                .data(partial, UTF8_TEXT) // 数据是模型当前响应的token
            );
        } catch (Exception e) {
            emitter.completeWithError(e);
        }
    })
    .onCompleteResponse(resp -> { // 模型完整结束时触发
        try {
            String fullText = resp.aiMessage().text(); // 完整回复
            emitter.send(
                SseEmitter.event()
                .name("done")
                .data(fullText, UTF8_TEXT) // 数据是模型的完整内容
            );
        } catch (Exception ignored) {
        } finally {
            emitter.complete();
        }
    })
    .onError(emitter::completeWithError)
    .start();
```

最后在Controller层调用`SseEmitter`，返回，需要加上`text/event-stream`，表明这是个**SSE响应类型**（`springframework`有给出常量类）

```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter chat(String message) {
    return assistantService.chat(message);
}
```

在浏览器调试，可以看到前端接收到类似下面这样的数据；可以看到一个个`event`，一开始是很多`token`，最后是一个`done`，和代码里一致；

```
event:token
data:你好

event:token
data:！

event:token
data:我是

event:token
data:快递

event:token
data:驿站

# 这里省略掉一些内容...

event:done
data:你好！我是快递驿站管理后台助手，可以帮你处理门店管理、货架管理、包裹查询、包裹出入库等系统内业务。请告诉我你需要办理的具体任务。
```



#### Flux 方式

> 这里就大概说说。

需要引入依赖：

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-reactor</artifactId>
    <version>1.0.0-beta3</version>
</dependency>
```

用`Flux`代替`String`：

```java
public interface Assistant {
    @SystemMessage("你是一个快递驿站的智能助手")
    Flux<String> chat(@UserMessage String message);
}
```

Controller层的方法的返回值也改成`Flux<String>`，还需要加上`text/event-stream`，表明这是个**SSE响应类型**（`springframework`有给出常量类），HTTP长连接，服务端不断推送数据；

```java
@GetMapping(value = "/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chat(String message) {
    return assistant.chat(message);
}
```



### 消息注解

通过`@UserMessage`定义用户消息，可以用`@V`注解定义参数，并用双花括号`{{}}`引用参数：

```java
// 不加任何注解，则参数认为是用户消息
String chat(String userMessage);
// 与上面等同
String chat(@UserMessage String userMessage);
// 但有多个参数时，需要用注解指定
String chat(@MemoryId String memoryId, @UserMessage String userMessage);

// 定义参数
@UserMessage("写一首关于 {{theme}} 的诗，并提到 {{name}}")
String createPoem(@V("theme") String theme, @V("name") String name);

// 方法无参
@UserMessage("0.8 和 0.11 哪个大？")
String chat();
```

通过`@SystemMessage`定义系统消息，同样可以用`@V`注解定义参数，并用双花括号`{{}}`引用参数：

```java
@SystemMessage("你是一个可爱的猫娘")
String chat(String message);

@SystemMessage("描述一个国家, {{answerInstructions}}")
@UserMessage("{{country}}")
String chat(@V("answerInstructions") String answerInstructions, @V("country") String country);
```

上面2个注解都可以通过`fromResource`参数指定prompt文件，路径是`/resource`目录开始：

```java
@SystemMessage(fromResource = "my-prompt.txt")
String askSomething(String input);

@UserMessage(fromResource = "my-prompt.txt")
String askSomething(String input);
```



### 工具/函数调用

定义一个`Spring Bean`，然后用`@Tool`注解声明方法，给定方法的**描述信息**，通过`@P`注解描述方法的参数：

```java
@Component // 需要注册为Bean
public class MyTool {

    @Tool("对给定的两个数字做求积")
    public double times(@P("第一个数字") double a, @P("第二个数字") double b) {
        return a * b;
    }

    @Tool("查询天气")
    public String weather(@P("日期") LocalDateTime date) {
        return "晴转多云";
    }

}
```

在`@AiService`注解中指定`Bean`（Spring中的Bean默认为**类名首字母小写**）

```java
@AiService(
        tools = {
                "myTool"
        }
)
public interface assistant {
    String chat(String message);
}
```

然后正常使用即可，AI会知道它可以使用哪些工具方法；

在调试中可以看到，对大模型API的请求参数里，多了`tools`字段：

```json
"tools" : [ {
    "type" : "function",
    "function" : {
        "name" : "weather",
        "description" : "查询天气",
        "parameters" : {
            "type" : "object",
            "properties" : {
                "date" : {
                    "type" : "object",
                    "description" : "日期",
                    "properties" : {
                        "date" : {
                            "type" : "object",
                            "properties" : {
                                "year" : {
                                    "type" : "integer"
                                },
                                "month" : {
                                    "type" : "integer"
                                },
                                "day" : {
                                    "type" : "integer"
                                }
                            },
                            "required" : [ "year", "month", "day" ]
                        },
                        "time" : {
                            "type" : "object",
                            "properties" : {
                                "hour" : {
                                    "type" : "integer"
                                },
                                "minute" : {
                                    "type" : "integer"
                                },
                                "second" : {
                                    "type" : "integer"
                                },
                                "nano" : {
                                    "type" : "integer"
                                }
                            },
                            "required" : [ "hour", "minute", "second", "nano" ]
                        }
                    },
                    "required" : [ "date", "time" ]
                }
            },
            "required" : [ "date" ]
        }
    }
}, {
    "type" : "function",
    "function" : {
        "name" : "times",
        "description" : "对给定的两个数字做求积",
        "parameters" : {
            "type" : "object",
            "properties" : {
                "a" : {
                    "type" : "number",
                    "description" : "第一个数字"
                },
                "b" : {
                    "type" : "number",
                    "description" : "第二个数字"
                }
            },
            "required" : [ "a", "b" ]
        }
    }
} ]
```



## 会话记忆与隔离

框架会通过维护一些列**对象**来标识，并有`memoryId → ChatMemory`的映射，将之前的对话存储在内存中；

`memoryId`可以是任意类型的对象，没有限制，只要类正确覆盖了 `equals()` 和 `hashCode()`，就可以作为 memoryId 用来区分不同会话；

一般的做法：**前端**可以传递给后端一个 **会话 ID / 用户 ID / Token**，作为 `@MemoryId`；

> 如果配置了记忆，但没有配置`memoryId`做会话隔离，则所有对话都储存在`"default"`这个`memoryId`中，会有不同用户串记忆的问题；

具体操作如下：

创建一个 `ChatMemoryProvider`，重写其方法：

```java
@Configuration
public class AssistantConfig {

    /**
     * 创建一个 ChatMemoryProvider
     */
    @Bean
    public ChatMemoryProvider chatMemoryProvider() {
        return new ChatMemoryProvider() {

            /**
             * 创建一个 ChatMemory
             * @param memoryId 用于标记聊天的对象
             */
            @Override
            public ChatMemory get(Object memoryId) {
                // 这里返回的 MessageWindowChatMemory 是基于【消息窗口】记忆和淘汰数据的
                return MessageWindowChatMemory.builder()
                        .maxMessages(10)
                        .id(memoryId)
                        .build();
            }
        };
    }

}
```

然后添加`@AiService`的参数，指定`Bean`名称，并且方法的参数加上一个`memoryId`，并用`@MemoryId`注解修饰：

```java
@AiService(
        chatMemoryProvider = "chatMemoryProvider"
)
public interface Assistant {
    String chat(@MemoryId String memeryId, @UserMessage String message);
}
```



## 其他

还有RAG、消息持久化等，就看官方文档吧
