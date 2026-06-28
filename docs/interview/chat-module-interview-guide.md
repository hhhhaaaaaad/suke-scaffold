# Chat 模块面试讲解

## 1. Chat 模块在整个项目里是干什么的

如果让我用一句话定义这个项目里的 Chat 模块，我会这样说：

> Chat 模块是整个 Agent 系统的运行期对话入口，它负责根据 `agentId` 找到已经自动装配好的 Runner，创建会话、发送消息、执行智能体，并以同步或流式的方式返回结果。

很多人第一次看这个项目时，会误以为 `chat` 只是一个“聊天接口”模块。  
其实它的真实职责比普通 Controller 要深得多，它连接了：

- 前端请求
- 运行时会话
- Agent Runner
- 模型返回事件流
- 同步/流式输出

所以从架构层面说，Chat 模块更像：

- 运行调度层
- 会话接入层
- Agent 执行入口层

---

## 2. 这个模块要解决什么实际问题

一个 Agent 系统启动装配完成后，运行期还需要解决 3 类问题：

### 问题 1：前端怎么知道系统里有哪些 Agent

系统里可能配置了多个 Agent：

- 学习规划 Agent
- 搜索总结 Agent
- 多智能体研究 Agent

前端需要先查询可用列表，再决定调用哪个 `agentId`。

### 问题 2：用户的会话上下文如何保持

大模型对话不是一次请求就结束，通常需要多轮上下文。

因此系统需要处理：

- 创建会话
- 复用会话
- 后续消息绑定到同一个 session

### 问题 3：模型响应如何返回给前端

Agent 的输出有两种典型方式：

- 同步返回：等全部生成完再返回
- 流式返回：边生成边返回

Chat 模块就统一承接了这三类问题。

---

## 3. 为什么 Chat 模块单独存在

从架构设计上，项目把：

- `armory`
- `chat`

拆开是非常合理的。

### `armory` 的职责

负责：

- 读取配置
- 装配模型
- 构建基础 Agent
- 组装 Workflow
- 创建 Runner
- 注册 `AiAgentRegisterVO`

### `chat` 的职责

负责：

- 接收用户请求
- 根据 `agentId` 找到 Runner
- 创建 Session
- 发送用户消息
- 返回执行结果

也就是说：

- `armory` 解决“怎么造”
- `chat` 解决“怎么用”

面试中推荐这样总结：

> 我们把启动期装配和运行期执行解耦了。装配由 armory 完成，chat 只关心运行时通过 `agentId` 获取已装配好的 Runner 并发起调用。这种拆分让职责非常清晰，也更适合后续扩展统一网关或多种触发方式。

---

## 4. Chat 模块的核心技术栈

这部分很适合面试时快速抛关键词。

### 核心技术栈

- Spring MVC
- Spring `@RestController`
- Google ADK `InMemoryRunner`
- Google ADK `Session`
- RxJava `Flowable<Event>`
- Spring MVC `ResponseBodyEmitter`
- `ConcurrentHashMap`
- 统一异常封装

### 核心设计点

- 运行时通过 `agentId` 获取注册对象
- 会话与消息执行分离
- 同步和流式共用底层 Runner
- 事件流驱动输出
- 多模态输入扩展能力

---

## 5. Chat 模块包含哪些关键类

### 1. 对外接口契约

[IAgentService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-api/src/main/java/cn/bugstack/ai/api/IAgentService.java)

它定义了 4 个核心能力：

- 查询智能体列表
- 创建会话
- 同步聊天
- 流式聊天

### 2. Controller 层

[AgentServiceController](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-trigger/src/main/java/cn/bugstack/ai/trigger/http/AgentServiceController.java)

职责：

- 提供 HTTP 接口
- 接收 DTO
- 调用领域服务
- 包装响应

### 3. 领域服务接口

[IChatService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/IChatService.java)

职责：

- 抽象 chat 行为能力

### 4. 领域服务实现

[ChatService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java)

这是 Chat 模块真正的核心。

### 5. 多模态命令模型

[ChatCommandEntity](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/entity/ChatCommandEntity.java)

职责：

- 封装文本、文件、字节流等多模态输入

### 6. 运行时注册对象

[AiAgentRegisterVO](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/valobj/AiAgentRegisterVO.java)

职责：

- 封装 `appName`
- 封装 `agentId`
- 封装 `runner`

它是 Chat 模块获取运行能力的桥梁。

---

## 6. Chat 模块的完整执行链路

这部分是面试中最重要的一段。

假设前端发起一个请求：

```json
{
  "agentId": "100001",
  "userId": "u100",
  "sessionId": "optional",
  "message": "帮我生成一个 Java 学习路线"
}
```

系统内部会按下面的顺序执行。

### 第 1 步：请求进入 Controller

入口在 [AgentServiceController#chat](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-trigger/src/main/java/cn/bugstack/ai/trigger/http/AgentServiceController.java#L107-L140)。

它会先检查：

- 是否传入 `sessionId`

如果没传，就会先创建会话：

```java
String sessionId = requestDTO.getSessionId();
if (sessionId == null || sessionId.isEmpty()) {
    sessionId = chatService.createSession(requestDTO.getAgentId(), requestDTO.getUserId());
}
```

然后调用：

- `chatService.handleMessage(...)`

### 第 2 步：ChatService 通过 `agentId` 找 Runner

在 [ChatService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L89-L95) 中：

```java
AiAgentRegisterVO aiAgentRegisterVO = defaultArmoryFactory.getAiAgentRegisterVO(agentId);
InMemoryRunner runner = aiAgentRegisterVO.getRunner();
```

这里体现了运行期与装配期的解耦：

- 启动时 `armory` 已经把 Runner 准备好了
- 运行时 `chat` 只负责查出来用

### 第 3 步：把用户消息包装成 ADK 的 `Content`

在 [ChatService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L97-L98) 中：

```java
Content userMsg = Content.fromParts(Part.fromText(message));
Flowable<Event> events = runner.runAsync(userId, sessionId, userMsg);
```

这里说明：

- 对外是普通字符串消息
- 对内会转换成 Agent 运行模型能理解的 `Content`

### 第 4 步：Runner 开始执行

`runner.runAsync(...)` 的输入是：

- `userId`
- `sessionId`
- `userMsg`

输出是：

- `Flowable<Event>`

这代表：

- 结果不是一次性对象
- 而是一条异步事件流

### 第 5 步：同步模式或流式模式返回

#### 同步模式

在 [ChatService#handleMessage](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L99-L103) 中：

```java
List<String> outputs = new ArrayList<>();
events.blockingForEach(event -> outputs.add(event.stringifyContent()));
```

含义：

- 先把所有事件收集完
- 再统一返回

#### 流式模式

在 [ChatService#handleMessageStream](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L106-L118) 中直接返回：

- `Flowable<Event>`

然后 Controller 用 [ResponseBodyEmitter](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-trigger/src/main/java/cn/bugstack/ai/trigger/http/AgentServiceController.java#L145-L165) 逐条向前端发送。

---

## 7. 创建会话为什么一定要用 `runner`

这是一个高频面试追问点。

相关代码在 [ChatService#createSession](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L54-L70)：

```java
return userSessions.computeIfAbsent(userId, uid -> {
    Session session = runner.sessionService().createSession(appName, uid)
            .blockingGet();
    return session.id();
});
```

很多人第一眼会觉得：

- 既然最后只返回一个 `sessionId`
- 为什么不直接自己生成一个字符串

答案是：

- 这个 `sessionId` 不是普通字符串标识
- 它必须先由 `runner.sessionService()` 创建成一个真实的会话对象
- 然后再把这个会话对象的 `id` 暴露给外部

也就是说，真实链路是：

- 先创建 `Session`
- 再返回 `session.id()`

不是：

- 先随便生成一个 `UUID`
- 再告诉系统这就是会话

### 为什么必须通过 Runner 创建

因为：

- Runner 才是最终执行主体
- 它内部绑定了：
  - 根 Agent
  - appName
  - 插件
  - 会话服务

所以会话必须依附于当前 Runner 的运行上下文。

面试建议回答：

> 会话并不是一个普通 ID，而是 Runner 运行上下文中的真实对象。只有通过 `runner.sessionService().createSession(...)` 创建，后续 `runAsync(...)` 才能基于这个 session 继续维护上下文。

---

## 8. 当前的会话管理策略是什么

会话缓存定义在 [ChatService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L36-L36)：

```java
private final Map<String, String> userSessions = new ConcurrentHashMap<>();
```

含义是：

- key：`userId`
- value：`sessionId`

也就是说，当前实现里：

- 同一个用户第一次请求时会创建一个 session
- 后续会复用这个 sessionId

这是通过：

- `computeIfAbsent(userId, ...)`

实现的。

### 这个设计有什么优点

- 简单
- 易于理解
- 单机脚手架场景够用
- 可以快速打通多轮对话能力

### 有什么局限

- 是内存态缓存
- 服务重启后会话丢失
- 只按 `userId` 维度，不按 `userId + agentId`
- 不支持更复杂的多会话管理

这部分是一个非常好的面试加分点。

你可以这样评价：

> 当前实现是一个脚手架级别的轻量会话策略，目标是快速打通 Agent 主链路。它的优点是简单，但在生产环境里需要进一步演进为 Redis 或数据库持久化，并且通常会按 `userId + agentId` 或 `conversationId` 做更细粒度管理。

---

## 9. `runAsync(...)` 到底做了什么

相关代码在 [ChatService#handleMessageStream](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L117-L118)：

```java
Content userMsg = Content.fromParts(Part.fromText(message));
return runner.runAsync(userId, sessionId, userMsg);
```

从当前项目能明确推断出的职责是：

### 第 1 层：拿到根执行入口

这个 `runner` 是启动时装配好的 `InMemoryRunner`，内部已经绑定：

- 最终入口 `BaseAgent`
- 插件
- appName

### 第 2 层：把这条用户消息送入执行图

不管最终入口是：

- 基础 Agent
- `ParallelAgent`
- `SequentialAgent`
- `LoopAgent`

都由 `runner.runAsync(...)` 启动执行。

### 第 3 层：按事件流返回结果

返回的是：

- `Flowable<Event>`

说明它不是把整个执行过程包成一个最终对象，而是：

- 每有一个事件就往外发一次

### 为什么这很适合 Agent 系统

因为 Agent 执行过程中天然存在“分阶段事件”：

- 模型增量输出
- 工具调用前后
- 子 Agent 执行阶段
- Workflow 中间结果

所以用 `Event` 流比一次性返回更贴近 Agent 运行本质。

---

## 10. 为什么可以流式输出

这个问题面试里非常常见。

### 根本原因

不是 Controller 自己把字符串拆成片段，而是底层本来返回的就是：

- `Flowable<Event>`

### Controller 是如何输出的

在 [AgentServiceController#chatStream](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-trigger/src/main/java/cn/bugstack/ai/trigger/http/AgentServiceController.java#L145-L165) 中：

```java
chatService.handleMessageStream(...)
        .subscribe(
                event -> {
                    try {
                        emitter.send(event.stringifyContent());
                    } catch (Exception e) {
                        log.error("流式对话发送失败", e);
                        emitter.completeWithError(e);
                    }
                },
                emitter::completeWithError,
                emitter::complete
        );
```

这段逻辑含义是：

- 事件流里每来一个 `event`
- 就执行一次回调
- 回调里调用 `emitter.send(...)`
- `ResponseBodyEmitter` 立即把这段内容写到 HTTP 响应流

所以整条链路是：

`runAsync(...) -> Flowable<Event> -> subscribe(...) -> emitter.send(...) -> 前端持续收到结果`

### 同步返回为什么不是这样

对比 [ChatService#handleMessage](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L99-L103)：

```java
List<String> outputs = new ArrayList<>();
events.blockingForEach(event -> outputs.add(event.stringifyContent()));
```

同步模式是：

- 把所有事件先收集完
- 再一次性返回

流式模式是：

- 每来一个事件就立即发送

这就是本项目里同步与流式的本质区别。

---

## 11. Chat 模块支持哪些输入方式

普通 HTTP 接口当前主要是：

- 文本消息

请求对象在 [ChatRequestDTO](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-api/src/main/java/cn/bugstack/ai/api/dto/ChatRequestDTO.java)：

- `agentId`
- `userId`
- `sessionId`
- `message`

但项目实际上已经预留了更强的输入能力。

相关模型是 [ChatCommandEntity](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/entity/ChatCommandEntity.java)。

支持：

- `texts`
- `files`
- `inlineDatas`

然后在 [ChatService#handleMessage(ChatCommandEntity)](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/chat/ChatService.java#L120-L162) 中转换为：

- `Part.fromText(...)`
- `Part.fromUri(...)`
- `Part.fromBytes(...)`

这说明：

- 这个项目的 Chat 模块不只是简单的文本聊天层
- 它已经具备多模态输入扩展能力

这也是一个很不错的面试亮点。

---

## 12. Chat 模块和自动装配模块是如何衔接的

这是整套项目必须讲清楚的核心。

### 启动期

自动装配模块会创建：

- `AiAgentRegisterVO`

并注册到 Spring 容器。

相关逻辑在 [RunnerNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/RunnerNode.java#L47-L58)。

### 运行期

Chat 模块通过 [DefaultArmoryFactory#getAiAgentRegisterVO](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/factory/DefaultArmoryFactory.java#L42-L44) 获取：

```java
public AiAgentRegisterVO getAiAgentRegisterVO(String agentId) {
    return applicationContext.getBean(agentId, AiAgentRegisterVO.class);
}
```

所以完整桥梁是：

- `agentId`
  -> `AiAgentRegisterVO`
  -> `InMemoryRunner`
  -> `runAsync(...)`

面试中建议这样说：

> Chat 模块本身并不关心 Agent 是怎么装出来的，它只依赖 `AiAgentRegisterVO` 这个注册结果。启动期由自动装配模块把 Agent 和 Runner 注册进 Spring，运行期 Chat 再按 `agentId` 获取，这样就实现了启动装配和运行执行解耦。

---

## 13. 面试重点怎么讲

### 面试官常问 1：为什么 Chat 层不直接 new 一个模型对象去调用

建议回答：

> 因为这个系统不是单纯的模型调用，而是多 Agent、多 Workflow、可扩展工具的运行体系。真正可运行的不是单个模型，而是装配好的 Runner，所以 Chat 层必须通过 `agentId` 获取对应 Runner，而不是自己临时组装。

### 面试官常问 2：为什么要有 session

建议回答：

> 因为 Agent 对话通常是多轮的，session 的作用是把同一个用户的多轮消息绑定到同一个上下文。如果没有 session，每次对话都像第一次请求，无法延续历史状态。

### 面试官常问 3：为什么要支持流式输出

建议回答：

> 大模型和 Agent 的结果通常不是一次性生成完毕的，而是逐步产生。流式返回可以降低用户等待感知，也更适合展示中间内容和多 Agent 工作流的阶段结果。

### 面试官常问 4：为什么同步和流式能共存

建议回答：

> 因为底层统一是 `Flowable<Event>`。同步模式只是把事件流收集后再返回，流式模式则是每个事件到来就立即发送，所以两种模式共享同一套执行主链路。

### 面试官常问 5：当前会话实现有什么问题

建议回答：

> 当前实现是脚手架级别的简化版本，只按 `userId` 缓存 session，适合快速验证链路，但生产环境一般会升级为 Redis 持久化，并引入 `conversationId` 或 `userId + agentId` 粒度的管理策略。

---

## 14. Chat 模块的项目亮点

### 亮点 1：运行期统一入口

不管底层是单 Agent 还是多 Agent Workflow，Chat 层都只需要：

- `agentId -> runner`

这说明接口层对运行模型是高度抽象的。

### 亮点 2：同步与流式共享一套执行主链路

底层统一使用：

- `runner.runAsync(...)`

同步与流式只是消费方式不同。

### 亮点 3：会话机制与执行机制分离

Chat 层先处理会话，再执行消息，逻辑上非常清晰。

### 亮点 4：多模态输入预留

`ChatCommandEntity` 已经支持：

- 文本
- 文件 URI
- 二进制字节

说明系统具备进一步向多模态 Agent 演进的基础。

### 亮点 5：与自动装配模块天然解耦

启动期装配，运行期执行，职责分离明确，非常适合后续扩展更多触发器：

- HTTP
- Job
- MQ
- WebSocket

---

## 15. 风险点与优化建议

### 风险点 1：会话缓存是内存态

问题：

- 服务重启丢失
- 不适合多实例部署

优化：

- 改为 Redis 或数据库持久化

### 风险点 2：会话粒度过粗

问题：

- 目前是 `userId -> sessionId`
- 同用户多个 Agent 对话时不够精细

优化：

- 改为 `userId + agentId`
- 或直接引入 `conversationId`

### 风险点 3：流式输出协议较简单

问题：

- 目前直接 `emitter.send(event.stringifyContent())`
- 对前端事件类型区分不够清晰

优化：

- 可改造成 SSE 标准事件格式
- 增加事件类型字段，如：
  - token
  - tool_start
  - tool_end
  - workflow_step
  - done

### 风险点 4：异常处理粒度还可以更细

问题：

- 当前主要区分业务异常和未知异常

优化：

- 增加会话异常、Runner 异常、流式中断异常的专项处理

---

## 16. 面试收口表达

如果面试官让你总结 Chat 模块，你可以这样说：

> Chat 模块是整个 Agent 系统的运行期入口。它不负责组装 Agent，而是依赖启动期注册好的 `AiAgentRegisterVO` 和 `InMemoryRunner`。用户请求进来后，系统先根据 `agentId` 找到 Runner，再按需要创建 session，把用户消息包装成 ADK 的 `Content`，最终通过 `runAsync(...)` 发起执行。同步模式会把事件流收集后返回，流式模式则通过 `ResponseBodyEmitter` 把每个事件逐条推给前端。这种设计把装配和运行解耦，也兼顾了多轮会话和流式体验。 

---

## 17. 你应该重点记住的 6 句话

1. Chat 模块不是“普通聊天接口”，而是运行期统一执行入口
2. 它通过 `agentId` 去 Spring 容器里找到已装配好的 `AiAgentRegisterVO`
3. 真正执行消息的是 `runner.runAsync(...)`
4. `sessionId` 不是随便生成的，而是通过 `runner.sessionService()` 创建的真实会话
5. 同步与流式共用同一个底层事件流，只是消费方式不同
6. 当前会话实现是脚手架级轻量方案，生产环境需要持久化和更细粒度管理

---

## 18. 高频面试题补充

### 问题 1：Chat 模块和普通聊天 Controller 有什么本质区别

参考回答：

> 普通聊天 Controller 往往只是把请求转发给模型接口，而这个项目里的 Chat 模块承担的是运行期统一入口角色。它不仅要处理 HTTP 请求，还要根据 `agentId` 找到 Runner、创建或复用 session、执行 Agent，并支持同步和流式两种返回方式。

### 问题 2：为什么 Chat 层不是直接 new 一个模型去调用

参考回答：

> 因为运行时真正执行的不是一个裸模型，而是启动时装配好的 `InMemoryRunner`。这个 Runner 内部已经绑定了根 Agent、workflow、插件和会话能力。Chat 层只负责按 `agentId` 找到对应 Runner，而不是自己重新组装执行链。

### 问题 3：为什么要通过 `agentId` 获取 `AiAgentRegisterVO`

参考回答：

> 因为 `AiAgentRegisterVO` 是启动期自动装配的最终结果，它封装了这个智能体的运行信息，包括 `appName`、`agentId` 和 `runner`。运行期通过 `agentId` 去容器里查，是装配和执行解耦的关键桥梁。

### 问题 4：`createSession()` 为什么需要 `runner`

参考回答：

> 因为 session 不是普通字符串 ID，而是 Runner 会话体系中的真实对象。必须由 `runner.sessionService().createSession(...)` 创建，后续 `runAsync(...)` 才能在这个 session 上继续维护对话上下文。最后返回给前端的只是 `session.id()`。

### 问题 5：当前实现是不是一个用户只对应一个会话

参考回答：

> 在当前实现里，可以近似这么理解。因为本地缓存是 `Map<userId, sessionId>`，并通过 `computeIfAbsent(userId, ...)` 保证同一个用户默认复用同一个会话。但这是脚手架级策略，不是最终生产级方案。

### 问题 6：这个会话管理方案有什么不足

参考回答：

> 主要有三个不足：第一，内存态缓存，重启会丢；第二，只按 `userId` 维度，不够精细；第三，不适合多实例部署。生产环境一般会升级为 Redis 或数据库持久化，并引入更明确的 `conversationId`。

### 问题 7：为什么 `runAsync(...)` 能实现流式返回

参考回答：

> 因为底层返回的是 `Flowable<Event>`，它本身就是响应式事件流。Controller 只需要订阅这个流，然后每收到一个 `event` 就通过 `ResponseBodyEmitter` 发送给前端，所以形成了边生成边返回的流式效果。

### 问题 8：同步返回和流式返回底层有什么区别

参考回答：

> 底层没有本质区别，都是调用 `runner.runAsync(...)`。区别只在消费方式：同步接口用 `blockingForEach` 把所有 `Event` 收集完后统一返回；流式接口则在 `subscribe(...)` 中每来一个事件就立即发送。

### 问题 9：为什么流式输出对 Agent 系统更重要

参考回答：

> 因为 Agent 不只是简单文本生成，它可能包含工具调用、workflow 步骤执行、中间结果产出等阶段性信息。流式输出可以让用户更早看到反馈，也更适合展示复杂执行过程。

### 问题 10：`ChatCommandEntity` 的意义是什么

参考回答：

> 它说明 Chat 模块并不局限于文本输入，而是已经预留了多模态扩展能力。除了文本，还支持文件 URI 和内联字节数据，这为后续接图片、文档、附件类输入打下了基础。

### 问题 11：如果 `agentId` 不存在会怎么样

参考回答：

> `ChatService` 会先通过 `defaultArmoryFactory.getAiAgentRegisterVO(agentId)` 获取注册对象，如果不存在就抛出 `AppException(ResponseCode.E0001)`。然后 Controller 统一捕获并包装成标准响应，保证接口层行为一致。

### 问题 12：这个 Chat 模块最值得讲的亮点是什么

参考回答：

> 我认为有三个亮点。第一，运行期统一入口，屏蔽底层单 Agent 和多 Agent 的差异；第二，同步和流式共享同一套 Runner 执行链；第三，会话、消息执行、结果输出职责清晰，为后续扩展 WebSocket、MQ、Job 等触发方式提供了良好基础。
