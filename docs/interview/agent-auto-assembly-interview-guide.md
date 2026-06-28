# Agent 自动装配面试讲解

## 1. 这部分解决的是什么问题

在传统 Java 项目里，如果要接入一个新的智能体，常见做法是：

- 手写一个 Service
- 手写一个 Prompt
- 手写工具注册逻辑
- 手写执行入口
- 手写多智能体协作编排

这种方式的主要问题是：

- 新增一个 Agent 需要改很多 Java 代码，扩展成本高
- 单个 Agent 和多 Agent Workflow 混在一起，维护复杂
- 工具、模型、工作流、入口耦合在代码里，不利于配置化管理
- 难以做成脚手架或平台化能力

这个项目的自动装配模块，本质上是在解决一个很典型的 Agent 工程化问题：

`如何把 YAML 中声明式配置的 Agent、Workflow、Model、MCP、Skills、Plugin，在应用启动时自动装配成一个可运行的 Agent Runner。`

面试中可以这样描述：

> 我们没有把 Agent 写死在 Java 代码里，而是做成了配置驱动的自动装配模式。启动时系统会读取 YAML 配置，依次构造模型能力、基础 Agent、Workflow Agent 和最终 Runner，再注册到 Spring 容器，运行时只需要通过 agentId 获取即可。

这个表达是一个很好的项目亮点，因为它体现了：

- 配置驱动
- 插件化装配
- 运行时解耦
- 面向平台能力设计

---

## 2. 需求场景是什么

这套自动装配能力适合下面几类场景：

### 场景 1：快速新增一个智能体

例如业务要新增：

- Java 学习规划智能体
- SQL 优化智能体
- 面试问答智能体

如果每次都改 Java 代码，交付效率会很低。配置驱动的好处是：

- 通过 YAML 新增一份配置即可
- 只在必要时新增工具或插件
- 重启后自动生效

### 场景 2：从单 Agent 升级到多 Agent 协作

例如：

- 一个 Agent 负责搜索
- 一个 Agent 负责总结
- 一个 Agent 负责评估结果

这时基础 Agent 已经不够，需要支持：

- `loop`
- `parallel`
- `sequential`

也就是把多个基础 Agent 编排成一个工作流 Agent。

### 场景 3：做 Agent 脚手架或平台底座

这个项目的名字本身就体现出脚手架特征。脚手架的核心诉求不是做一个写死的功能点，而是提供可扩展的通用底座：

- 业务方只关心配置
- 平台负责启动装配
- 运行时通过统一入口调用

这就是自动装配存在的最现实原因。

---

## 3. 为什么选择“自动装配 + 配置驱动”

从方案选择角度，通常有 3 种做法。

### 方案 A：硬编码式 Agent 装配

做法：

- 每个 Agent 都写一个 Bean
- 每个 Workflow 都写一套 Java 编排代码
- 工具注册写死在配置类里

优点：

- 逻辑直观
- 对简单项目上手快

缺点：

- 扩展成本高
- 难平台化
- 代码重复严重

### 方案 B：数据库驱动 Agent 平台

做法：

- Agent 元数据、Prompt、Workflow、工具配置都存数据库
- 后台页面维护

优点：

- 适合完整平台化
- 支持动态运营

缺点：

- 实现复杂
- 对脚手架阶段来说过重
- 数据模型、后台管理、校验机制成本高

### 方案 C：YAML 配置驱动 + 启动时自动装配

做法：

- 把 Agent 结构描述在 YAML 中
- 应用启动时将配置绑定到对象模型
- 通过装配链构建最终运行对象

优点：

- 非常适合脚手架
- 低成本扩展
- 结构清晰
- 比写死代码灵活，比数据库平台轻量

缺点：

- 热更新能力弱
- 配置错误通常在启动期或运行期暴露
- 大规模动态运营能力不如平台方案

这个项目明显选择了方案 C，因为它兼顾了：

- 脚手架轻量性
- DDD 分层清晰度
- Agent 编排扩展能力

面试中推荐这样说：

> 我们选择 YAML 配置驱动而不是硬编码，是为了降低新增 Agent 和 Workflow 的成本；同时又没有上数据库平台，是因为这个项目定位是脚手架，优先考虑落地速度、结构清晰和可扩展性。

---

## 4. 技术栈与设计关键词

这部分很适合面试时先抛关键词，建立技术密度。

### 核心技术栈

- Java 17
- Spring Boot 3
- Spring `@ConfigurationProperties`
- Google ADK
- Spring AI
- RxJava `Flowable`
- 自定义策略路由链
- Spring 容器动态注册 Bean

### 核心设计关键词

- 配置驱动
- 自动装配
- 策略路由链
- Workflow 编排
- 运行时注册
- 动态上下文
- Agent 平台化思维

---

## 5. 自动装配整体架构

自动装配主链路可以拆成 5 层。

### 第 1 层：配置绑定

入口类是 [AiAgentAutoConfigProperties](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/valobj/properties/AiAgentAutoConfigProperties.java)。

它通过：

- `@ConfigurationProperties(prefix = "ai.agent.config")`

把 YAML 中的：

- `tables`

绑定成：

- `Map<String, AiAgentConfigTableVO>`

每个 `table` 就是一套完整的 Agent 配置单元。

配置结构模型定义在 [AiAgentConfigTableVO](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/valobj/AiAgentConfigTableVO.java)。

### 第 2 层：启动触发

入口类是 [AiAgentAutoConfig](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-app/src/main/java/cn/bugstack/ai/config/AiAgentAutoConfig.java)。

它监听应用启动完成事件：

- `ApplicationReadyEvent`

启动后会执行：

- `armoryService.acceptArmoryAgents(...)`

这一步的含义是：

- 所有 Agent 不是在第一次请求时懒加载
- 而是在系统启动完成后统一完成装配

这样做的优点是：

- 启动后即可直接提供服务
- 错误更早暴露
- 运行时不用再反复组装

### 第 3 层：装配入口

装配服务在 [ArmoryService](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/ArmoryService.java)。

它会遍历每一个 `table`：

- 每个 `table` 代表一个独立的 Agent 应用配置
- 每次装配都会创建一个新的 `DynamicContext`

这是一个非常重要的设计点。

面试中可以强调：

> 每个 table 的装配上下文都是隔离的，防止不同 Agent 配置之间相互污染，比如模型配置、workflow 中间态、agentGroup 等都不会串。

### 第 4 层：节点式装配链

装配链入口在 [DefaultArmoryFactory](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/factory/DefaultArmoryFactory.java)。

它返回的根节点是：

- [RootNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/RootNode.java)

整体装配顺序是：

1. `RootNode`
2. `AiApiNode`
3. `ChatModelNode`
4. `AgentNode`
5. `AgentWorkflowNode`
6. `RunnerNode`

这条链路体现的是典型“分步构造对象”的思想。

### 第 5 层：最终注册运行对象

最终由 [RunnerNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/RunnerNode.java) 创建：

- `InMemoryRunner`
- `AiAgentRegisterVO`

然后按 `agentId` 注册到 Spring 容器中。

这样运行时就能通过：

- `agentId -> AiAgentRegisterVO -> runner`

直接找到执行入口。

---

## 6. 为什么 `AiAgentConfigTableVO` 要分成 `appName / agent / module`

这是一个非常值得面试讲清楚的设计点。

相关模型在 [AiAgentConfigTableVO](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/model/valobj/AiAgentConfigTableVO.java)。

### `appName`

它是运行级别的标识，更偏内部运行上下文。

用途：

- session 创建时作为应用名
- runner 构造时作为运行标识

它不是前端展示字段，而是运行系统内部要用的名字。

### `agent`

它是对外暴露的智能体身份信息，包括：

- `agentId`
- `agentName`
- `agentDesc`

用途：

- 前端查询可用 Agent 列表
- 运行时按 `agentId` 获取注册对象

这部分可以理解成“智能体对外身份层”。

### `module`

它是“内部实现层”，定义这个 Agent 是怎么运行起来的。

里面包括：

- `aiApi`
- `chatModel`
- `agents`
- `agentWorkflows`
- `runner`

这是整个配置最核心的部分。

面试中可以这么回答：

> 之所以分成 `appName / agent / module`，本质上是为了隔离运行标识、对外身份和内部实现三个关注点。如果全部揉在一起，配置既难理解，也不利于平台化扩展。

---

## 7. Workflow 是怎么被自动装配出来的

### 先装基础 Agent

在 [AgentNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/AgentNode.java) 中，系统会先把 `module.agents` 全部装配成基础 `LlmAgent`。

每个基础 Agent 都会放入：

- `dynamicContext.agentGroup`

这个 `agentGroup` 定义在 [DefaultArmoryFactory.DynamicContext](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/factory/DefaultArmoryFactory.java#L53-L108)。

可以把它理解成：

- 当前这一次装配过程中的 Agent 注册表

### 再按顺序处理 `agentWorkflows`

在 [AgentWorkflowNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/AgentWorkflowNode.java) 中，会按顺序取出每个 workflow：

- 通过 `currentStepIndex`
- 取 `agentWorkflows.get(currentStepIndex)`
- 放到 `currentAgentWorkflow`
- 再根据类型路由

这里的顺序很重要：

- 前面的 workflow 可以被后面的 workflow 引用
- 后面的 workflow 不能反向引用前面的未来结果

### 再由具体节点组装 Workflow Agent

三种 workflow 对应三个节点：

- [LoopAgentNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/workflow/LoopAgentNode.java)
- [ParallelAgentNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/workflow/ParallelAgentNode.java)
- [SequentialAgentNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/workflow/SequentialAgentNode.java)

这些节点的共同逻辑是：

1. 先拿到当前 workflow 的 `subAgents` 名称列表
2. 去 `dynamicContext.agentGroup` 中按名字查出真正的 `BaseAgent`
3. 构造成新的 workflow agent
4. 再把这个 workflow agent 放回 `agentGroup`

这就是为什么：

- 后续 workflow 可以把前一个 workflow 当成自己的子 Agent

这是项目中非常亮眼的一个设计点。

面试亮点表达：

> 我们把基础 Agent 和 Workflow Agent 都统一抽象成 `BaseAgent` 放进 `agentGroup`，这样后续 workflow 既可以引用基础 Agent，也可以引用前面装出来的 workflow agent，本质上形成了一棵可递归扩展的执行树。

---

## 8. 为什么最终 `RunnerNode` 只取一个 `agentName`

这是面试里非常容易被追问的点。

相关代码在 [RunnerNode](file:///d:/java/scaffold/ai-agent-suke-scaffold/ai-agent-suke-scaffold-domain/src/main/java/cn/bugstack/ai/domain/agent/service/armory/node/RunnerNode.java#L61-L86)。

很多人会困惑：

- 前面明明装了很多 Agent
- 为什么到最后只取一个 `agentName`

原因是：

- `RunnerNode` 的职责不是“继续批量装配”
- 而是“从已经装好的 Agent 集合里选择一个最终入口”

这个最终入口可以是：

- 单个基础 `LlmAgent`
- `ParallelAgent`
- `SequentialAgent`
- `LoopAgent`

也就是说，虽然它只 `get` 一次：

- `dynamicContext.getAgentGroup().get(agentName)`

但取出来的这个对象可能本身就是一棵完整的 Agent 执行树。

面试中建议你这样回答：

> 前面 workflow 阶段做的是“组装树”，RunnerNode 做的是“选根节点”。所以它只需要一个 `agentName`，因为真正运行时只能有一个统一入口，但这个入口内部完全可以是多 Agent 组合体。

---

## 9. 一个完整的示例链路

假设配置如下：

- 基础 Agent
  - `JavaResearcher`
  - `SpringResearcher`
  - `SummaryAgent`

- Workflow
  - `ParallelResearch = JavaResearcher + SpringResearcher`
  - `StudyPipeline = ParallelResearch + SummaryAgent`

- Runner
  - `runner.agentName = StudyPipeline`

那么自动装配过程是：

### 第一步

`AgentNode` 创建：

- `JavaResearcher`
- `SpringResearcher`
- `SummaryAgent`

此时 `agentGroup` 为：

- `JavaResearcher`
- `SpringResearcher`
- `SummaryAgent`

### 第二步

`ParallelAgentNode` 创建：

- `ParallelResearch`

它的 `subAgents` 来源是：

- `JavaResearcher`
- `SpringResearcher`

此时 `agentGroup` 为：

- `JavaResearcher`
- `SpringResearcher`
- `SummaryAgent`
- `ParallelResearch`

### 第三步

`SequentialAgentNode` 创建：

- `StudyPipeline`

它的 `subAgents` 来源是：

- `ParallelResearch`
- `SummaryAgent`

此时 `agentGroup` 为：

- `JavaResearcher`
- `SpringResearcher`
- `SummaryAgent`
- `ParallelResearch`
- `StudyPipeline`

### 第四步

`RunnerNode` 读取：

- `runner.agentName = StudyPipeline`

最终入口就是：

- `StudyPipeline`

然后创建：

- `InMemoryRunner(baseAgent = StudyPipeline, appName, plugins)`

这就是自动装配闭环。

---

## 10. 面试重点怎么讲

这一部分是你真正拿去面试最有价值的内容。

### 面试官常问 1：为什么要做自动装配

建议回答：

> 我们希望 Agent 能像业务组件一样通过配置扩展，而不是每次新增都改大量 Java 代码。自动装配的价值在于把模型、工具、基础 Agent、Workflow 和最终执行入口解耦，让脚手架具备平台化能力。

### 面试官常问 2：为什么不用数据库配置

建议回答：

> 数据库驱动更适合完整平台，但当前项目定位是脚手架，目标是低成本落地和源码清晰度。YAML 配置比硬编码灵活，比数据库平台轻量，是当前阶段的平衡方案。

### 面试官常问 3：Workflow 如何支持递归组合

建议回答：

> 关键是统一抽象成 `BaseAgent`，不管是基础 Agent 还是 workflow agent，最终都放入 `agentGroup`。后续 workflow 只按名称查询子 Agent，因此天然支持 workflow 引用 workflow。

### 面试官常问 4：如何保证不同配置之间不串数据

建议回答：

> 每个 `table` 装配时都会创建新的 `DynamicContext`，上下文中的模型、workflow 当前状态、agentGroup 都是隔离的，因此不会相互覆盖污染。

### 面试官常问 5：Runner 为什么只选择一个入口

建议回答：

> Runner 运行的不是单个普通 Agent，而是整个执行树的根节点。前面的工作流装配已经把多 Agent 协作封装在根节点内部了，所以运行阶段只需要选择一个统一入口。

---

## 11. 这个模块的项目亮点

如果你想把这部分讲得更像“有工程价值”，建议重点提下面几点。

### 亮点 1：配置驱动的 Agent 平台化思维

不是写死某个 Agent，而是让 Agent 成为一种可配置、可组合、可装配的对象。

### 亮点 2：装配链清晰，职责拆分明确

按节点拆为：

- API 装配
- Model 装配
- 基础 Agent 装配
- Workflow 装配
- Runner 封装

这体现了很好的分层能力。

### 亮点 3：统一抽象 `BaseAgent`

基础 Agent 和 Workflow Agent 都能进入同一个容器 `agentGroup`，让递归编排成为可能。

### 亮点 4：启动期完成装配，运行期快速执行

启动时一次性完成所有 Agent 构建，运行时只需按 `agentId` 获取 `runner`，降低请求路径复杂度。

### 亮点 5：可扩展工具链

自动装配过程里还接入了：

- MCP
- Skills
- Plugin

说明它不仅能装 Agent，还能装能力边界更广的生态组件。

---

## 12. 风险点与优化建议

这部分是面试加分项，体现你不仅能看懂代码，还会做架构评估。

### 风险点 1：配置校验不够严格

问题：

- 如果 `runner.agentName` 配错
- 如果 workflow 引用不存在的子 Agent
- 启动期可能晚些时候才暴露问题

优化：

- 增加配置预校验
- 启动时检查所有引用关系是否闭合

### 风险点 2：Bean 注册依赖 `agentId` 唯一

问题：

- 如果不同配置重复使用同一个 `agentId`
- 可能会覆盖已注册 Bean

优化：

- 启动阶段增加唯一性校验

### 风险点 3：YAML 复杂度升高后可读性下降

问题：

- Agent 多了以后配置文件会很长

优化：

- 支持拆分多文件
- 支持环境化配置
- 支持配置模板

### 风险点 4：缺少热更新能力

问题：

- 配置改动通常需要重启

优化：

- 后续可演进为数据库驱动或配置中心驱动

---

## 13. 面试收口表达

如果面试官让你用 1 分钟总结自动装配模块，你可以这样说：

> 这个项目的自动装配模块，本质上是在做 Agent 的配置驱动构建。系统启动时先把 YAML 绑定成配置对象，再通过一条装配链依次构建 LLM API、ChatModel、基础 Agent、Workflow Agent 和最终 Runner，最后按 agentId 注册到 Spring 容器。这样运行时只需要通过 agentId 获取 Runner，就能快速执行。这个设计的核心价值是把 Agent 的定义、编排和执行入口解耦，降低扩展成本，并为多智能体工作流提供统一装配能力。

---

## 14. 这篇内容你应该重点记住什么

如果你时间不够，最少记住这 5 句话：

1. 自动装配的目标是把 YAML 配置转换成可运行的 `runner`
2. `module.agents` 先装基础 Agent，`module.agentWorkflows` 再装组合 Agent
3. 所有已装配 Agent 都统一进入 `agentGroup`
4. `runner.agentName` 选择的是最终入口根节点
5. 运行时通过 `agentId -> AiAgentRegisterVO -> runner` 直接执行

---

## 15. 高频面试题补充

### 问题 1：你们的自动装配到底装配了什么

参考回答：

> 自动装配并不是只装配一个 Bean，而是按配置分阶段装配整条 Agent 运行链路。先装配 `OpenAiApi` 和 `ChatModel`，再装配基础 `LlmAgent`，然后按 workflow 配置装配 `LoopAgent`、`ParallelAgent`、`SequentialAgent`，最后根据 `runner.agentName` 选择根节点，封装成 `InMemoryRunner` 和 `AiAgentRegisterVO` 注册到 Spring 容器。

### 问题 2：为什么不用 `@Bean` 把所有 Agent 直接写死

参考回答：

> 如果直接写死成 `@Bean`，每新增一个 Agent 或 Workflow 都需要改 Java 代码，维护成本很高。当前方案把 Agent 定义从代码里抽离到 YAML，适合脚手架和平台底座场景，能显著降低扩展成本。

### 问题 3：`AiAgentConfigTableVO` 为什么设计得这么深

参考回答：

> 因为它本质上是在映射一份结构化 DSL。YAML 里天然是嵌套结构，所以 Java 对象也按运行语义拆成了 `appName / agent / module` 三层，分别承担运行标识、对外身份、内部实现三个职责，避免配置语义混乱。

### 问题 4：`DynamicContext` 的作用是什么

参考回答：

> `DynamicContext` 是单次装配过程的临时上下文，承载中间态数据，比如 `openAiApi`、`chatModel`、`agentGroup`、`currentStepIndex`、`currentAgentWorkflow`。它不是最终结果存储，而是给节点链在装配过程中传递数据用的。

### 问题 5：为什么每个 `table` 都要 new 一个新的 `DynamicContext`

参考回答：

> 因为每个 `table` 都代表一套独立的 Agent 配置，如果复用上下文，前一个 Agent 应用的模型、workflow 状态、agentGroup 都可能污染后一个配置。新建上下文是为了保证装配隔离性。

### 问题 6：`agentGroup` 为什么这么关键

参考回答：

> `agentGroup` 是当前装配流程里的 Agent 注册表，里面统一存放基础 Agent 和 Workflow Agent。这样 workflow 节点只需要根据 `subAgents` 的名字去查，就能支持基础 Agent 引用，也能支持 workflow 引用 workflow。

### 问题 7：`AgentWorkflowNode` 为什么最终会进入 `RunnerNode`

参考回答：

> 因为 `AgentWorkflowNode` 的职责只是逐个处理 `agentWorkflows` 列表。每次处理一个 workflow，就把 `currentStepIndex` 往后推进。当索引达到 `agentWorkflows.size()` 时，会把 `currentAgentWorkflow` 设为 `null`，此时路由逻辑会直接切到 `RunnerNode`，表示 workflow 装配结束，进入最终运行器封装阶段。

### 问题 8：为什么 `RunnerNode` 只取一个 `agentName`

参考回答：

> 因为 Runner 需要的是“最终执行入口”，而不是一堆未组织的 Agent。前面的节点已经把多个 Agent 组装成了一棵执行树，`runner.agentName` 选择的就是这棵树的根节点。这个根既可以是基础 Agent，也可以是 Workflow Agent。

### 问题 9：如果 workflow 的 `subAgents` 写错了怎么办

参考回答：

> 当前实现会在查询 `agentGroup` 时拿不到对应对象，最终导致 workflow 组装不完整或运行异常。更稳妥的做法是在启动期增加配置校验，确保 `subAgents` 的名称引用都能在 `agents` 或已创建 workflow 中被解析。

### 问题 10：这个自动装配方案最大的工程价值是什么

参考回答：

> 最大价值是把 Agent 的定义、编排和执行入口彻底解耦，让系统具备配置化扩展能力。对脚手架和平台底座来说，这比单纯写一个能跑的 Agent 更有复用价值，也更适合后续接入多业务场景。
