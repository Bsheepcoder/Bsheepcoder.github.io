## 元认知：Java 生态面对 LLM 的三个根本矛盾

2023 年初，当 ChatGPT 引爆新一轮 AI 浪潮时，Java 社区面临一个尴尬的局面：Python 开发者有 LangChain、LlamaIndex、Haystack 等琳琅满目的框架，而 Java 开发者只能手写 HTTP 请求调用 OpenAI API，自己处理 JSON 解析、Token 计数、对话记忆管理。这种落差不是偶然的——它暴露了 Java 企业生态与 LLM 快速迭代之间的深层鸿沟。

要理解后来出现的 LangChain4j 和 Spring AI 为何选择了不同的道路，需要先看清 Java 生态面对 LLM 集成时遇到的三个根本矛盾。

### 类型安全 vs 非确定输出

Java 的核心竞争力之一是类型系统。接口、泛型、编译时检查构成了企业级应用的信任基石。但 LLM 的本质是非确定性的——同一个 prompt 两次调用可能得到不同的输出，输出格式可能偏离预期，工具调用的参数可能 JSON 解析失败。

这不是一个可以"加个 try-catch"就解决的问题。非确定性向上传染：如果一个方法返回 `String`，但底层调用了 LLM，那么调用方无法从类型签名中得知这个方法的可靠性边界。Spring AI 和 LangChain4j 都试图用结构化输出（`ChatClient.entity()`、`AiServices` 接口返回值）来驯服这种非确定性，但它们的策略截然不同。

### 声明式编程 vs 链式调用

Java 开发者习惯声明式编程：写一个接口，加几个注解，框架替你实现细节。Spring Data JPA 的 `@Repository`、Feign 的 `@FeignClient` 都是这个模式的经典代表。但 LLM 的工作流本质上是链式的：先嵌入查询 → 检索向量库 → 重排序 → 构建 prompt → 调用模型 → 解析输出 → 可能还要调用工具 → 再调用模型。

两个框架都试图弥合这个差距，但切入角度不同。Spring AI 把链式调用封装进 `Advisor` 模式，让开发者感觉不到链的存在；LangChain4j 则直接把链暴露为 `AiServices` 接口声明，让开发者显式编排组件。这不是一个"谁更好"的问题，而是**对"抽象应该隐藏什么"的不同判断**。

### 框架绑定 vs 框架中立

Spring 是 Java 企业生态的事实标准，但不是全部。Quarkus 在云原生领域快速崛起，Micronaut 在微服务场景有独特优势，还有大量遗留系统运行在 Jakarta EE 上。一个 LLM 集成框架是否与 Spring 绑定，直接决定了它的适用半径。

Spring AI 的答案是"绑定即优势"——与 Spring Boot 的自动配置、Actuator 的可观测性、Spring Security 的安全体系深度整合，让 Spring 开发者零成本上手。LangChain4j 的答案是"中立即优势"——提供与 Spring Boot、Quarkus、Micronaut、Helidon 乃至纯 Java SE 的集成，让团队不被框架锁定。

这两种选择没有对错，它们反映了对"Java 生态的未来"的不同预判。

### 两套架构的哲学岔路口

基于以上三个矛盾的解法差异，两个框架走上了不同的道路：

```
                ┌─────────────────────────────────┐
                │    Java LLM 集成框架            │
                │                                 │
   ┌────────────┤    根本问题：如何驯服          ├────────────┐
   │            │    LLM 的非确定性？             │            │
   │            └─────────────────────────────────┘            │
   │                                                           │
   ▼                                                           ▼
┌──────────────────────┐                          ┌──────────────────────┐
│  Spring AI           │                          │  LangChain4j         │
│  平台学派            │                          │  工具箱学派          │
├──────────────────────┤                          ├──────────────────────┤
│  ChatClient 中心     │                          │  AiServices 接口驱动 │
│  Advisor 切面模式    │                          │  组件模块化拼装      │
│  自动装配 + 配置驱动 │                          │  Builder 显式构造    │
│  Spring 生态深度绑定  │                          │  多框架中立          │
│  Micrometer 原生可观测│                          │  30+ 向量库/20+ 模型 │
└──────────────────────┘                          └──────────────────────┘
```

---

## 搭积木：核心抽象的模式差异

理解了两个框架的设计哲学，就可以逐一审视它们在核心抽象层面上的差异。这些差异不是偶然的——每一种选择都源于前文所述的三个根本矛盾的侧重点不同。

### 模型调用：ChatClient vs ChatModel

Spring AI 的核心抽象是 `ChatClient`，它是一个围绕模型调用构建的 fluent API：

```java
// Spring AI
ChatResponse response = chatClient.prompt()
    .user("你好，请介绍一下自己")
    .call()
    .chatResponse();
```

`ChatClient` 不仅封装了模型调用，还集成了 Advisor 链、工具调用、记忆管理。在 Spring AI 2.0 中，工具调用循环也被移入 Advisor 链，使得 `ChatClient` 成为整个系统的唯一入口。

LangChain4j 则将底层 API（`ChatModel`）与高层 API（`AiServices`）明确分离：

```java
// LangChain4j - 底层 API
ChatModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4o-mini")
    .build();
String answer = model.generate("你好，请介绍一下自己");

// LangChain4j - 高层 API
Assistant assistant = AiServices.builder(Assistant.class)
    .chatModel(model)
    .build();
String answer = assistant.chat("你好，请介绍一下自己");
```

这里的差异本质上是**抽象粒度的选择**：Spring AI 用一个 `ChatClient` 解决所有问题（统一入口），LangChain4j 提供从底层到高层的多个抽象层级（灵活选择）。前者适合标准化的企业场景，后者适合需要精细控制的工作流。

### 声明式 AI 服务：AiServices vs Advisor 链

这是两个框架最具辨识度的差异。

LangChain4j 的 `AiServices` 采用 Java 开发者最熟悉的模式——定义接口，加注解，框架自动生成实现：

```java
interface CustomerSupportAgent {

    @SystemMessage("你是客户支持助手，回答问题时基于提供的上下文信息")
    String chat(@UserMessage @V("query") String query);
}

CustomerSupportAgent agent = AiServices.builder(CustomerSupportAgent.class)
    .chatModel(model)
    .contentRetriever(retriever)
    .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
    .tools(new OrderLookupTool())
    .build();
```

这种模式与 Spring Data JPA、OpenFeign 如出一辙。接口即契约，注解即配置，Builder 即装配。团队的 Java 经验可以直接迁移。

Spring AI 则采用不同的模式——Advisor 链：

```java
ChatClient chatClient = ChatClient.builder(model)
    .defaultAdvisors(
        new QuestionAnswerAdvisor(vectorStore),
        new MessageChatMemoryAdvisor(chatMemory)
    )
    .defaultTools(new OrderLookupTool())
    .build();

String answer = chatClient.prompt()
    .user("我的订单什么时候发货？")
    .call()
    .content();
```

Advisor 模式本质上是一种责任链（Chain of Responsibility）。每个 Advisor 可以拦截请求、增强上下文、处理响应。RAG 是 Advisor，记忆是 Advisor，工具调用也是 Advisor。它的优势在于可组合性和可插拔——你可以像搭积木一样叠加功能，且每个 Advisor 可以独立测试。

两种模式的本质区别：

| 维度 | AiServices（LangChain4j） | Advisor 链（Spring AI） |
|------|--------------------------|------------------------|
| 心智模型 | 接口即服务 | 过滤器链 |
| 声明方式 | 接口注解 | 链式装配 |
| 单元测试 | Mock 接口即可 | 需 Mock Advisor 链 |
| 复杂场景 | 多接口分工 | 单链 + 分支 |
| 框架耦合 | 无（纯注解） | 与 Spring 紧耦合 |

这个差异再次反映了两者的哲学分歧：LangChain4j 选择的是 Java 开发者最熟悉的"接口声明"模式，Spring AI 选择的是 Spring 生态最熟悉的"过滤器链"模式。

### 工具调用：@Tool vs @Bean

工具（函数调用）是两个框架都视为核心能力的特性。

LangChain4j 使用 `@Tool` 注解标记方法：

```java
class OrderLookupTool {

    private final OrderRepository orderRepository;

    @Tool("根据订单号查询订单状态")
    String lookupOrder(
        @P("订单号") String orderId
    ) {
        return orderRepository.findStatus(orderId);
    }
}
```

Spring AI 的模式则看似相似但实质不同——任何 Spring Bean 的方法都可以作为工具：

```java
@Component
class OrderService {

    @Tool("根据订单号查询订单状态")
    public String lookupOrder(String orderId) {
        return orderRepository.findStatus(orderId);
    }
}
```

区别在于工具的来源和生命周期管理。LangChain4j 的工具是你显式传入 `AiServices` 的，工具的实例化和管理由你控制。Spring AI 的工具是 Spring 容器中的 Bean，由 DI 容器管理。前者更灵活（工具可以是 POJO、远程代理、Mock），后者与 Spring 生态无缝整合（事务传播、AOP 切面、作用域代理）。

LangChain4j 还支持更高级的特性：工具搜索（Tool Search，适用于数十个工具的自动发现）、立即返回（`returnBehavior = IMMEDIATE`，跳过 LLM 重新处理）、补偿操作（`@CompensateFor`，多工具执行失败时自动回滚）。这些特性在 Spring AI 2.0 中正在追赶，但 LangChain4j 仍领先一个迭代周期。

### RAG 管线：模块化 vs 自动化

RAG 可能是功能差异最明显的领域。

LangChain4j 提供三层 RAG：Easy RAG（一键启动）、Naive RAG（检索 + 注入）、Advanced RAG（完整管线：查询变换 → 路由 → 检索 → 重排序 → 聚合 → 注入）。

```java
// LangChain4j Advanced RAG
RetrievalAugmentor augmentor = DefaultRetrievalAugmentor.builder()
    .queryTransformer(new CompressingQueryTransformer(chatModel))
    .contentRetriever(EmbeddingStoreContentRetriever.builder()
        .embeddingStore(embeddingStore)
        .embeddingModel(embeddingModel)
        .maxResults(5)
        .minScore(0.75)
        .build())
    .contentAggregator(new DefaultContentAggregator())
    .contentInjector(DefaultContentInjector.builder()
        .prefix("请基于以下信息回答问题：\n")
        .build())
    .build();
```

每个环节都是可替换的接口——你可以自定义查询变换策略、替换检索实现、注入自己的重排序逻辑。

Spring AI 的 RAG 则通过 Advisor 模式实现：

```java
// Spring AI RAG through Advisor
ChatClient chatClient = ChatClient.builder(model)
    .defaultAdvisors(
        RetrievalAugmentationAdvisor.builder()
            .queryAugmenter(ContextualQueryAugmenter.builder()
                .allowEmptyContext(true)
                .build())
            .documentRetriever(VectorStoreDocumentRetriever.builder()
                .vectorStore(vectorStore)
                .similarityThreshold(0.75)
                .build())
            .build()
    )
    .build();
```

Spring AI 2.0 的 ETL 框架提供了 `DocumentReader` → `DocumentTransformer` → `DocumentWriter` 的标准管线，配合 EasyRAG 风格可以极速启动，但深度定制时需要自己实现 Advisor。

这里的模式差异反映了一个重要工程决策：**LangChain4j 把 RAG 的每个环节都暴露为可替换接口，让你精确控制每一步；Spring AI 把 RAG 封装为 Advisor，让你快速组装但定制路径受限。** 对大多数企业应用来说，"受限但快速"可能更优；但对需要精细调优检索质量的应用，"可控但复杂"才是正确选择。

### MCP 支持：两个框架的协议层战略

Model Context Protocol（MCP）是 2026 年 Java AI 生态的关键词。它不是一个新的框架特性，而是一个标准化的工具发现和调用协议。

Spring AI 2.0 内置了完整的 MCP 客户端和服务端支持，支持 Streamable HTTP（现代标准）、SSE（向后兼容）、stdio（本地进程）三种传输方式。这意味着一个 Spring Boot 应用可以通过 MCP 协议暴露自己的业务逻辑，供任何 MCP 兼容的 Agent 调用。

LangChain4j 在 1.17+ 版本中也加入了 MCP 客户端支持，工具声明自动转换为 MCP-compliant 格式。但与 Spring AI 不同的是，LangChain4j 的 MCP 支持是作为一个独立模块存在的（`langchain4j-mcp`），需要显式引入。

两者对 MCP 的支持代表了各自对"协议层"的态度：Spring AI 把 MCP 视为 Spring 生态的自然延伸，LangChain4j 视为可选的高级特性。

---

## 案例即原理：同一个需求，两套解法

理论与模式的讨论需要落到具体场景中才有意义。以下用同一个需求——构建一个能够查询内部文档和实时订单状态的客服 Agent——在两个框架中的实现，揭示工程模式差异如何影响最终的代码结构和架构决策。

### 需求定义

一个在线零售平台的客服 Agent，需要：
1. 检索内部知识库（退货政策、发货流程等 FAQ 文档）
2. 查询实时订单状态（通过 REST API）
3. 维护多轮对话上下文
4. 在工具执行失败时提供补偿处理

### LangChain4j 实现

```java
// 1. 定义 AI Service 接口（即契约）
interface CustomerSupportAgent {

    @SystemMessage("""
        你是零售平台的客户支持助手。
        当用户询问订单状态时，查询订单系统。
        当用户询问政策问题时，使用检索到的知识库信息回答。
        不要编造信息。
        """)
    String chat(@UserMessage @V("query") String query);
}

// 2. 实现订单查询工具
class OrderTool {

    private final OrderApiClient orderClient;

    OrderTool(OrderApiClient orderClient) {
        this.orderClient = orderClient;
    }

    @Tool("根据订单号查询实时订单状态，包括配送进度")
    OrderStatus queryOrder(@P("订单号") String orderId) {
        return orderClient.getStatus(orderId);
    }

    @CompensateFor("queryOrder")
    void notifyOrderQueryFailed(String orderId) {
        // 记录失败日志，发送通知
    }
}

// 3. 构建 Agent
CustomerSupportAgent agent = AiServices.builder(CustomerSupportAgent.class)
    .chatModel(OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o-mini")
        .build())
    .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
    .contentRetriever(EmbeddingStoreContentRetriever.builder()
        .embeddingStore(pgvectorStore)
        .embeddingModel(OpenAiEmbeddingModel.builder()
            .apiKey(System.getenv("OPENAI_API_KEY"))
            .build())
        .maxResults(3)
        .minScore(0.7)
        .build())
    .tools(new OrderTool(new OrderApiClient()))
    .build();
```

LangChain4j 的代码结构呈现了清晰的**分层**：接口定义契约、POJO 实现能力、Builder 装配系统。每个组件都是独立的、可测试的、可替换的。工具类 `OrderTool` 是一个普通的 Java 类，不依赖任何框架注解（除 `@Tool` 外）。

### Spring AI 实现

```java
@Configuration
class SupportAgentConfig {

    @Bean
    ChatClient supportChatClient(
            ChatClient.Builder builder,
            VectorStore vectorStore,
            OrderService orderService) {

        return builder
            .defaultSystem("""
                你是零售平台的客户支持助手。
                当用户询问订单状态时，查询订单系统。
                当用户询问政策问题时，使用检索到的知识库信息回答。
                不要编造信息。
                """)
            .defaultAdvisors(
                new QuestionAnswerAdvisor(vectorStore),
                new MessageChatMemoryAdvisor(
                    InMemoryChatMemoryRepository.builder()
                        .maxMessages(20)
                        .build())
            )
            .defaultTools(orderService)
            .build();
    }
}

@Component
class OrderService {

    private final OrderApiClient orderClient;

    OrderService(OrderApiClient orderClient) {
        this.orderClient = orderClient;
    }

    @Tool("根据订单号查询实时订单状态，包括配送进度")
    OrderStatus queryOrder(String orderId) {
        return orderClient.getStatus(orderId);
    }
}

@RestController
class ChatController {

    private final ChatClient supportChatClient;

    ChatController(@Qualifier("supportChatClient") ChatClient supportChatClient) {
        this.supportChatClient = supportChatClient;
    }

    @PostMapping("/chat")
    String chat(@RequestBody String message) {
        return supportChatClient.prompt()
            .user(message)
            .call()
            .content();
    }
}
```

Spring AI 的代码结构呈现了 Spring 生态的标准模式：`@Configuration` 装配系统、`@Bean` 声明组件、`@Component` 注册服务、`@RestController` 暴露端点。所有东西都在 Spring 容器的管理下。

### 模式对比

| 维度 | LangChain4j | Spring AI |
|------|-------------|-----------|
| 入口声明 | `AiServices.builder(Interface.class)` | `ChatClient.builder(model)` |
| 工具注册 | `tools(new OrderTool(...))` 显式传参 | `.defaultTools(orderService)` DI 注入 |
| 记忆管理 | `ChatMemory` 可配置策略 | `MessageChatMemoryAdvisor` 作为 Advisor |
| RAG 集成 | `ContentRetriever` 组件式 | `QuestionAnswerAdvisor` 切面式 |
| 错误补偿 | `@CompensateFor` 原生支持 | 需自定义 Advisor |
| 可测试性 | Mock 接口即可 | 需构建完整 Advisor 链 |

两种实现都达到了同样的功能目标，但架构风格截然不同。LangChain4j 的代码更像是"组合式"的——你选择组件并拼装它们，系统是你的设计产物。Spring AI 的代码更像是"配置式"的——你声明需要什么，系统替你装配，平台是你的运行环境。

没有绝对的优劣。LangChain4j 的组件的独立性和可测试性在复杂 Agent 场景有优势；Spring AI 的配置式开发在标准化企业应用场景更高效。问题在于你的团队处于哪种工程文化中。

---

## 缺陷与批判：从原理层面审视两框架的不足

任何诚实的深度分析都必须指出不足。以下不是"罗列 Bug"，而是从原理层面分析两个框架当前面临的深层问题。

### 共同的困境

**Agent 模式仍在婴儿期。** 截至 2026 年 7 月，两个框架在 Agent 维度上的"真正"智能仍远落后于 Python 生态。LangChain4j 在 1.16-1.18 版本中密集引入了 Debate（辩论）、Blackboard（黑板）、BDI（信念-愿望-意图）等多种 Agent 模式，Spring AI 2.0 也在 Advisor 链中加入了工具调用循环——但这些都只是"模式"而非"能力"。Agent 的核心挑战——长期规划、错误恢复、记忆压缩、多 Agent 协作——没有一个框架给出了令人满意的生产级方案。

原因不在框架本身，而在于 LLM 能力的边界：模型的上下文窗口有限，工具调用中的幻觉无法根治，Agent 的决策路径缺乏可解释性。框架可以封装这些挑战，但不能消除它们。

**RAG 质量的可观测性严重不足。** 两个框架都提供了 RAG 管线，但没有一个能回答生产环境中最关键的问题："这条 RAG 管线的检索准确率是多少？" Spring AI 的 Micrometer 指标只能告诉你调用了多少次 LLM、用了多少 Token，但无法衡量检索的相关性。LangChain4j 可以通过定制 `ContentRetriever` 接入评估框架，但这需要额外的工作量。

这意味着大多数生产部署中的 RAG 系统是用"感觉"来判断检索质量的，而非数据驱动。

**语义缓存的缺失是一个昂贵的空洞。** 对重复查询的语义缓存（非精确匹配，而是语义相似度匹配）可以将 LLM API 成本降低 70% 以上。两个框架都没有内置这个能力。LangChain4j 的模块化设计使得基于 `EmbeddingStore` 实现语义缓存相对简单，但这不是框架提供的能力——需要你自己写。Spring AI 则需要在 Advisor 层拦截并实现缓存，复杂度稍高。

在 API 成本仍然高昂的 2026 年，语义缓存的缺失意味着大部分生产部署都在为重复内容支付全价。

**MCP 协议的承诺与现实之间存在差距。** MCP 被宣传为"工具调用的 USB 标准"，但 2026 年中的现实是生态仍然碎片化。Spring AI 和 LangChain4j 都实现了 MCP 客户端，但互操作性测试不足，不同提供商的 MCP 服务端行为不一致。标准有了，但生态统一还需要时间。

### Spring AI 特有的问题

**Spring Boot 版本锁定的代价。** Spring AI 2.0 要求 Spring Boot 4.0+ 和 Spring Framework 7+。对于大量仍运行在 Spring Boot 3.x 上的企业应用，升级成本可能超过 AI 集成本身的收益。Spring AI 1.1.x 维护线存在，但新特性不会回迁，这意味着选择 Spring AI 就等于选择了跟随 Spring 的大版本发布节奏。

**Advisor 模式的调试复杂度。** Advisor 链本质上是一个责任链，每个 Advisor 可能修改请求、增强上下文、甚至短路处理。当出现问题时，你需要逐层检查每个 Advisor 的行为——这比 LangChain4j 的显式组件组装更难调试。Spring AI 2.0 改进了这一点，但 Advisor 链的本质复杂性没有消失。

**高级定制的隐形成本。** 当标准 Advisors 不够用时，你需要自己实现 Advisor——这意味着理解 Spring AI 的内部架构、扩展点协议、以及如何与现有的 Advisor 链协作。LangChain4j 的组件替换通常是实现一个接口（如 `ContentRetriever`、`QueryTransformer`），边界更清晰。

### LangChain4j 特有的问题

**可观测性需要自行搭建。** LangChain4j 没有内置 Micrometer 集成。你需要手动接入事件监听器或通过 Quarkus 扩展来获取调用指标。对于已经在使用 Spring Boot Actuator 的团队，这意味着额外的运维成本。

**模块分裂增加决策成本。** LangChain4j 有 60+ 个集成模块，BOM 中区分稳定版和 beta 版。选择正确的主版本和子模块组合需要仔细阅读兼容性表。虽然这带来了灵活性，但也增加了学习曲线和升级复杂度。

**Builder 模式在大型系统中的重复。** 当你在代码中多次构建 `AiServices` 时，Builder 的参数会反复出现。虽然可以通过工厂方法或依赖注入来减少重复，但对比 Spring AI 的自动配置，Builder 模式确实需要更多样板代码。这在纯 Java SE 场景中不是问题，但在大型 Spring Boot 项目中会成为噪音。

**社区治理的潜在风险。** LangChain4j 是社区驱动的项目，虽然获得了 Red Hat 和 Microsoft 的支持，但治理结构不如 Spring AI 明确。对于需要长期承诺的企业级项目，这是一个需要评估的因素。

---

## 总结事物本身：不是框架之争，是工程文化之辩

LangChain4j 和 Spring AI 不是因为功能差异而产生的竞争——它们是对同一个问题的两种回答。

### 框架是工程文化的具象化

LangChain4j 代表了 Java 社区中"工具箱"工程文化：提供丰富的、独立的组件，让开发者按需组合。这种文化认为，最好的系统是那些开发者亲手设计和组装出来的系统。它的根源可以追溯到 Java 的 POJO 传统——不依赖框架、不绑定平台、纯 Java 就能运行。

Spring AI 代表了"平台"工程文化：提供统一的、自动化的抽象，让开发者在约定好的轨道上快速运行。这种文化认为，工程效率来自减少决策——框架替你做了正确的选择，你只需要关注业务逻辑。它的根源是 Spring 生态的成功经验——约定优于配置、自动装配、依赖注入。

两种文化在 Java 社区共存已久。从 ORM 框架（Hibernate vs JPA）、到 Web 框架（Spring MVC vs Jakarta EE）、到构建工具（Maven vs Gradle），同样的哲学分歧一直在重复出现。LLM 集成只是最新的战场。

### 选择的真正含义

选择 LangChain4j 意味着接受以下前提：

- 你希望精确控制 LLM 工作流的每一个环节
- 你的团队有能力理解并管理多个组件的交互
- 你不想被任何一个框架锁定（Spring、Quarkus、或其他）
- 你愿意为灵活性付出装配和可观测性的代价

选择 Spring AI 意味着接受以下前提：

- 你的团队已经深度使用 Spring Boot，不想学习新的编程模型
- 你更关注应用的整体工程质量（可观测性、配置管理、安全）而非 AI 管线的精细控制
- 你愿意接受 Spring 的版本发布节奏来换取生态整合
- 你的大部分 AI 场景是标准化的（RAG + 工具调用 + 记忆管理），不需要极端定制

### 框架之外的真实挑战

将过多的注意力放在框架选择上，可能会让人忽略更重要的东西。2026 年的 Java AI 开发中，真正困难的问题不是选哪个框架，而是：

- 如何衡量 RAG 检索质量并建立持续改进的反馈循环
- 如何在生产环境中控制 LLM API 的成本（Token 监控、语义缓存、模型路由）
- 如何确保 Agent 行为的可预测性和安全性（Guardrails、人机协同、审计日志）
- 如何在多模型策略中平衡质量、成本和延迟
- 如何将 AI 能力融入现有的部署、监控、灾备体系

框架只是解决这些问题的工具。没有一个框架能替你回答"什么样的检索质量是可接受的"或"每个查询花多少 Token 是合理的"——这些是工程和业务决策。

### 面向未来的最实际建议

不要纠结于框架，而是做两件事：

第一，**用两周时间，用两个框架做同一个 POC**。不是对比功能，而是感受哪种模式更符合你团队的思维方式。工程文化的匹配是长远的胜出因素。

第二，**把架构设计成可替换的**。无论选择哪个框架，都通过接口隔离你的业务逻辑与框架调用。这样即使将来需要切换（从 LangChain4j 到 Spring AI，或反向），代价也是可控的。

2026 年的 Java 生态是幸运的——与 2023 年只有手写 HTTP 调用的状况相比，现在有了两个成熟、活跃、设计精良的框架可以选择。它们不是竞争者，而是同一道光谱上的两种颜色。理解它们的差异不是为了选边站队，而是为了看清 LLM 集成这个新问题在 Java 生态中的解决方案空间。

技术会变，需求会变，但工程文化不会轻易改变。与其问"哪个框架更好"，不如问"我的团队更适合哪种工程文化"。这个问题的答案，才是真正有效的长期决策依据。
