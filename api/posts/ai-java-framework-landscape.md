> **本文是 Java AI 框架系列的第三篇**。前两篇分别深入对比了 LangChain4j 与 Spring AI 的工程哲学分歧，以及 AgentScope 2.0 的架构原理。本篇旨在提供一张完整的生态地图，帮助你在不同场景下做出最合适的框架选型决策。
>
> 前情阅读：
> - [LangChain4j 与 Spring AI：一场必然的分歧](/2026/07/21/ai-langchain4j-practice/)
> - [AgentScope 2.0 深度解读](/2026/06/26/ai-agentscope-deep-dive/)

---

## 元认知：为什么有这么多框架？

2025 年以前，Java 开发者在 AI 集成上几乎没有选择——要么自己封装 API 调用，要么忍受 Python 框架的 JNI 桥接。到了 2026 年中，情况完全翻转：GitHub 上 Java AI 框架的 Star 总和已超过 **4.3 万**，主流框架超过 10 个。

但选择多了，困惑也多了。常见问题：

> "LangChain4j 和 Spring AI 哪个好？"
> "AgentScope 和 LangChain4j 是什么关系？"
> "Embabel 是什么，为什么 Spring 创始人 Rod Johnson 在做它？"
> "Google 出了 ADK Java，值得用吗？"

这些问题的根源在于一个普遍的误解：**认为所有框架都在解决同一个问题**。事实并非如此。

### 一个框架不能通吃所有

框架解决的不是同一个问题，而是不同层面的问题：

```
应用层（你的业务代码）
  ↓
调用层       ← 解决"怎么调用模型和工具"
  ↓
集成层       ← 解决"怎么装配和配置"
  ↓
编排层       ← 解决"怎么编排多 agent 协作"
  ↓
运行时层     ← 解决"怎么安全持久运行"
  ↓
治理层       ← 解决"怎么治理和控制"
```

- **调用层**：提供统一 API 封装 20+ 模型提供者，让你一行代码切换模型。代表：**LangChain4j**
- **集成层**：深度融入框架生态，自动装配、可观测、AOP 式拦截。代表：**Spring AI**
- **编排层**：处理多 agent 的工作流、规划、子 agent 分配。代表：**Embabel**、**Spring AI Alibaba Graph**
- **运行时层**：提供沙箱、多租户、权限、状态持久化。代表：**AgentScope Java**
- **治理层**：预算控制、审批门禁、RBAC、审计追踪。代表：**SwarmAI**

> **核心洞察**：这些框架不是竞品，是互补品。一个生产级 AI 应用通常需要组合使用多个层次的框架。

### 各框架的生态位

基于 2026 年 7 月的最新数据，主要框架的地位如下：

```
       Star 数    生产就绪    核心定位
LangChain4j     12,600+    ✅ GA     调用层之王
Spring AI        9,200+    ✅ v2.0   集成层标杆
Spring AI Alibaba 10,300+  ⚠️ M1.1  国内增强版
AgentScope Java  4,600+    ✅ v2.0   运行时层
Embabel          3,700+    ⚠️ RC     编排层创新者
Google ADK Java  1,700+    ✅ v1.6   Google 官方
Agents-Flex      1,000+    ✅ v2.2   轻量国产
Semantic Kernel Java 250+  ⚠️ 停滞  微软 Java 入口
SwarmAI             21     ✅ 1.0    治理层新秀
```

> **数据说明**：以上 Star 数为 2026 年 7 月 GitHub 数据，版本号为最新正式/预发布版本。

---

## 搭积木：八大框架各归其位

### 多维度评分体系

以下从 **7 个维度** 对主流框架评分（1-5 分），数据来源：GitHub 统计、官方文档、社区反馈、Frameworks 横向评测报告（Spring I/O 2026、Baeldung、CodeWiz 等）。

| 维度 | 说明 | 数据来源 |
|------|------|---------|
| 社区活跃度 | Stars、贡献者数、发布频率 | GitHub 直采 |
| 生产就绪度 | GA 状态、版本稳定性、企业案例 | 版本发布记录 |
| 多 Agent 能力 | 编排模式、子 agent、A2A 支持 | 官方文档 |
| 工程化能力 | 可观测、测试、运行时、CI/CD | 社区评测 |
| 生态兼容性 | 模型数、向量库、框架支持 | 官方文档 |
| 学习曲线 | 文档质量、API 设计、入门成本 | 开发者体验 |
| 创新性 | 独特架构、差异化决策 | 技术分析 |

### 评分总表

| 框架 | 社区活跃度 | 生产就绪度 | 多 Agent 能力 | 工程化 | 生态兼容 | 学习曲线 | 创新性 | **平均分** |
|------|-----------|-----------|---------|--------|---------|---------|-------|----------|
| LangChain4j | 5 | 5 | 3 | 3 | 5 | 4 | 3 | **4.0** |
| Spring AI | 5 | 5 | 2 | 5 | 4 | 4 | 3 | **4.0** |
| Spring AI Alibaba | 5 | 4 | 4 | 4 | 3 | 3 | 4 | **3.9** |
| AgentScope Java | 3 | 4 | 4 | 5 | 3 | 2 | 5 | **3.7** |
| Embabel | 3 | 2 | 5 | 3 | 3 | 3 | 5 | **3.4** |
| Google ADK Java | 2 | 3 | 5 | 3 | 2 | 3 | 4 | **3.1** |
| Agents-Flex | 2 | 3 | 3 | 3 | 2 | 4 | 3 | **2.9** |
| Semantic Kernel Java | 1 | 2 | 2 | 2 | 2 | 2 | 2 | **1.9** |

> **⚠️ 评分说明**：每个维度 1-5 分，按社区相对分位划分：Stars > 10K 或贡献者 > 300 为 5 分，Stars 3-10K 为 3-4 分，Stars < 1K 为 1-2 分。评分反映的是各维度相对水平，**高总分不意味更适合你的场景**。例如 AgentScope 总分 3.7 但运行时层维度独一档——如果你的需求就是运行时层能力，它的实际价值远超 4.0+ 的通用框架。

### 逐个解读

---

#### 1. LangChain4j——调用层之王

| 维度 | 评分 | 说明 |
|------|------|------|
| 社区活跃度 | 5 | 12.6K Stars，440 贡献者，76 个版本，月更 2-3 次 |
| 生产就绪度 | 5 | v1.18.0 GA，Quarkus/Micronaut/Spring 三栖兼容 |
| 生态兼容性 | 5 | 20+ 模型提供者，30+ 向量库，框架无关 |
| 学习曲线 | 4 | AiServices 接口声明式是最 Java 原生的 AI 抽象 |

LangChain4j 是 Java AI 框架中**最成熟、生态最广**的选择。它不做 framework lock-in，Spring Boot 项目能用，Quarkus 项目能用，裸 Java 也能用。

核心优势在于 **AiServices 模式**——定义一个接口，加注解，框架帮你生成实现：

```java
interface CustomerSupportAgent {
    @SystemMessage("你是一个客服助手，用中文回答")
    @UserMessage("用户问题: {{it}}")
    Answer reply(@V("it") String question);
}
// 一行构建
CustomerSupportAgent agent = AiServices
    .create(CustomerSupportAgent.class, model);
```

2026 年 1.18.0 的主要进展：
- `langchain4j-agentic` 模块进入 beta（多 Agent 工作流模式）
- MCP 客户端全面支持（Streamable HTTP 传输）
- Guardrails API（输入/输出护栏）
- A2A 协议支持

**适用场景**：框架中立的项目、Quarkus 生态、需要最大模型兼容性的场景。

> **注意**：LangChain4j 不做运行时层——没有沙箱、没有多租户、没有内置权限系统。这些需要自己实现或配合 AgentScope 使用。

---

#### 2. Spring AI——集成层标杆

| 维度 | 评分 | 说明 |
|------|------|------|
| 社区活跃度 | 5 | 9.2K Stars，410 贡献者，Spring 官方出品 |
| 生产就绪度 | 5 | v2.0.0 GA，Spring Boot 4.0/4.1 |
| 工程化能力 | 5 | Micrometer 原生集成，Actuator 健康检查，Spring 生态 |
| 学习曲线 | 4 | Spring Native 体验，自动装配，Advisors 模式 |

Spring AI 2.0 在 2026 年 6 月正式 GA，是第一个为 Spring Boot 4 设计的 AI 框架。它的定位非常明确：**如果你是 Spring Boot 团队，这就是最省力的 AI 集成方式**。

Advisors 模式是其独有的杀手锏——像 AOP 一样拦截 AI 调用链：

```java
ChatClient.Builder builder = ChatClient.builder(model)
    .defaultAdvisors(
        new QuestionAnswerAdvisor(vectorStore),   // RAG
        new MessageChatMemoryAdvisor(chatMemory), // 记忆
        new SafeGuardAdvisor()                     // 安全护栏
    );
```

但 Spring AI 2.0 **没有多 agent 编排**。它做的是 ChatClient + Tools + Advisors 的抽象，不做多 agent 工作流。如果需要多 agent，需要配合 Spring AI Alibaba 或 Embabel。

**适用场景**：Spring Boot 团队的首选，已有 Spring 基础设施的企业。

---

#### 3. Spring AI Alibaba——国内增强版

| 维度 | 评分 | 说明 |
|------|------|------|
| 社区活跃度 | 5 | 10.3K Stars，250 贡献者，国内最高 |
| 多 Agent 能力 | 4 | Graph 引擎 + 6 种多 Agent 模式 |
| 创新性 | 4 | Graph 作运行时，A2A+Nacos，Admin UI |
| 生态兼容性 | 3 | 阿里云生态绑定较重 |

Spring AI Alibaba 在 Spring AI 基础上增加了三样核心能力：
1. **Graph 工作流引擎**——以有向图定义 agent 协作流程，支持条件路由、嵌套子图、并行执行
2. **多 Agent 模式**——SequentialAgent、ParallelAgent、RoutingAgent、LoopAgent、Supervisor、Subagent、Handoffs
3. **A2A 通信**——基于 Nacos 的 Agent-to-Agent 远程发现

Graph 的核心 API 极具表达力：

```java
StateGraph<AgentState> graph = new StateGraph<>(AgentState.SCHEMA)
    .addNode("analyze", new AnalyzeNode())
    .addNode("search", new SearchNode())
    .addNode("respond", new RespondNode())
    .addConditionalEdges("analyze",
        state -> state.needSearch() ? "search" : "respond"
    );
CompiledGraph<AgentState> app = graph.compile();
```

**适用场景**：国内企业、阿里云用户、需要复杂工作流编排的 Spring 团队。

---

#### 4. AgentScope Java——运行时层独一档

| 维度 | 评分 | 说明 |
|------|------|------|
| 工程化能力 | 5 | 沙箱 + 权限 + 多租户 + Workspace，独一档 |
| 创新性 | 5 | Harness 双层 Agent、ContentBlock、事件总线 |
| 学习曲线 | 2 | Reactor 响应式 + 多层抽象，入门门槛较高 |
| 社区活跃度 | 3 | 4.6K Stars，100 贡献者，发展迅速但还年轻 |

2026 年 7 月 10 日，AgentScope Java 2.0 GA 发布。这是 Java AI 领域中**唯一一个把"运行时"作为一等公民**的框架。

其他框架解决的是"怎么开发 agent"，AgentScope 解决的是"怎么运行 agent"——这在生产环境中才是真正的难题。

核心差异化能力：
- **PermissionEngine**：细粒度工具/资源权限控制（allow/approve/reject 三级）
- **Workspace 抽象**：文件系统隔离，每个 session 独立工作区
- **Sandbox**：本地沙箱 / Docker 沙箱 / E2B 沙箱三层可选
- **Multi-tenancy**：租户级隔离，数据完全不互通
- **AgentStateStore**：支持 Redis/JSON File 持久化，无状态水平扩展
- **HarnessAgent**：在 ReActAgent 上加 Workspace/记忆/压缩/子 agent/Plan Mode/技能

```java
HarnessAgent agent = HarnessAgent.builder()
    .name("assistant")
    .model(modelRegistry.get("qwen-max"))
    .workspace(Paths.get("./workspace"))
    .toolkit(Toolkit.builder()
        .tool(new BashTool())
        .tool(new ReadTool())
        .build())
    .build();
```

**适用场景**：生产级多租户部署、安全敏感的金融/政务场景、需要持久化会话的长期运行 agent、与 LangChain4j/Spring AI 组合使用的运行时层。

> 关于 AgentScope 2.0 的架构深度解读，见本站[前文](/2026/06/26/ai-agentscope-deep-dive/)。

---

#### 5. Embabel——编排层创新者

| 维度 | 评分 | 说明 |
|------|------|------|
| 创新性 | 5 | GOAP 规划替代 ReAct 循环，游戏 AI 启发 |
| 多 Agent 能力 | 5 | 子 agent、A2A、MCP、GOAP 规划 |
| 生产就绪度 | 2 | v1.0.0-RC1，尚未 GA |
| 社区活跃度 | 3 | 3.7K Stars，60 贡献者（Spring 创始人效应） |

Embabel 是 Spring 创始人 **Rod Johnson** 的新项目，可能是 2026 年 Java AI 领域**最有思想深度**的框架。

它的核心创新：**抛弃 ReAct 循环，改用 GOAP（Goal-Oriented Action Planning）**。ReAct（Reason-Act 循环）是所有主流框架的默认路线——模型自己决定下一步做什么。Embabel 认为这不可靠、不可审计、不可测试，转而采用游戏 AI 领域的经典规划算法 A* 搜索 + 效用函数。

```kotlin
@Agent
class NutritionPlanner {
    @Stage
    fun fetchUserProfile(): UserProfile = // ...

    @Stage(goal = "Generate healthy weekly meal plan")
    @DependsOn(FetchUserProfile::class, FetchSeasonalIngredients::class)
    fun createPlan(profile: UserProfile, ingredients: List<Ingredient>): MealPlan = // ...

    @Stage(goal = "Validate nutritional balance")
    @DependsOn(CreatePlan::class)
    fun validate(plan: MealPlan): ValidationResult = // ...
}
```

规划器在运行时自动构建依赖图，A* 搜索最短执行路径。**编排是确定性算法，不是 LLM 推理**——这是 Embabel 的根本不同。

**适用场景**：复杂工作流需要确定性编排、可审计/可测试场景、愿意承担非 GA 风险的早期采用者。

---

#### 6. Google ADK Java——Google 官方入场

| 维度 | 评分 | 说明 |
|------|------|------|
| 多 Agent 能力 | 5 | Agent-only 架构，A2A 原生，seq/parallel/loop |
| 创新性 | 4 | 一切皆 agent，Dev UI，评估工具 |
| 生态兼容性 | 2 | GCP 强绑定，Gemini 优先 |
| 生产就绪度 | 3 | v1.6.0 已 GA，但 GCP 锁定风险 |

Google ADK（Agent Development Kit）Java 1.0 在 2026 年初 GA，标志着 Google 正式将 Java 视为 AI agent 的一等语言。

ADK 的架构哲学极端而纯粹：**一切皆 agent**。没有普通 generation 接口，没有 flow 抽象，只有 agent——LLM agent、workflow agent（sequential/parallel/loop）、custom agent。

A2A（Agent-to-Agent）协议是其骨架——所有 agent 通过统一协议发现和调用彼此，而不是通过框架私有的编排机制。这意味着理论上 ADK 的 agent 可以被任何实现 A2A 协议的框架调用（LangChain4j 和 Embabel 也支持 A2A）。

**适用场景**：Google Cloud / Vertex AI 客户、Gemini 优先的团队、需要官方生产 SLA 的企业。

---

#### 7. Agents-Flex——轻量国产选择

| 维度 | 评分 | 说明 |
|------|------|------|
| 学习曲线 | 4 | Java 8+ 兼容，Spring Boot 一键集成 |
| 生态兼容性 | 2 | 以国内模型为主，国际社区较小 |
| 生产就绪度 | 3 | v2.2.2 GA，功能全面但企业案例较少 |

Agents-Flex 定位为"轻量级 AI Agent 开发框架"，对 Java 版本要求极低（JDK 8+），模块划分清晰（Core/Chat/Tool/MCP/Skills/Text2SQL 等模块）。国内开发者可以快速上手。

核心能力：MCP 客户端、Skills 技能系统、Text2SQL 自然语言数据查询、WebSearch 搜索引擎、LLM Wiki 分层知识库。

**适用场景**：需要兼容老旧 JDK 的项目、国内模型（通义千问/DeepSeek）为主的团队、轻量级集成已有 Spring Boot 服务。

---

#### 8. Semantic Kernel Java——微软的 Java 入口

| 维度 | 评分 | 说明 |
|------|------|------|
| 社区活跃度 | 1 | 257 Stars，30 贡献者，最后版本 2025 年 5 月 |
| 生产就绪度 | 2 | Java 1.4.4-RC1（2025 年 5 月），已停滞 |
| 多 Agent 能力 | 2 | Planner 功能有限，无多 agent |

必须坦白地说：**Semantic Kernel Java 已经事实停滞**。主仓库 microsoft/semantic-kernel-java 的最后一次发布是 2025 年 5 月（超过 14 个月前），最后 commit 是 2026 年 3 月（维护性更新）。对比母项目 semantic-kernel（27.7K Stars，C#/Python 版本仍活跃更新），Java 分支处于严重维护不足状态。

**不建议新项目选用**。

---

### 框架定位全景图

```
治理层          SwarmAI
                 ↑
运行时层       AgentScope Java
                 ↑
编排层          Embabel · Spring AI Alibaba Graph · Google ADK Java
                 ↑
集成层          Spring AI · Spring AI Alibaba
                 ↑
调用层          LangChain4j · Agents-Flex · 裸 SDK
                 ↑
应用层          (你的业务代码)
```

---

## 案例即原理：三套组合架构方案

以一个**企业客服 Agent** 为例，展示不同技术栈的选型路径。

### 需求描述

一个面向 SaaS 平台的企业客服 AI：
- **多租户**：不同公司的客服数据完全隔离
- **工具调用**：查订单、退款操作、知识库检索
- **人工审批**：超过 ¥5000 的退款需审批
- **持续性**：会话跨越数天，用户可随时回来继续
- **合规审计**：所有 agent 行为可追溯

### 方案 A：Spring AI + AgentScope（推荐/Spring 团队）

```
[Spring Boot 4 应用]
  │
  ├─ Spring AI (集成层)
  │   ├─ ChatClient         ← 模型调用
  │   ├─ Advisors            ← RAG + 记忆 + 安全护栏
  │   └─ @Tool beans         ← 工具声明
  │
  └─ AgentScope (运行时层)
      ├─ HarnessAgent        ← 长运行 Agent
      ├─ PermissionEngine    ← 退款审批（人工审批）
      ├─ Multi-tenancy       ← 租户隔离
      └─ AgentStateStore     ← 会话持久化（Redis）
```

```java
@SpringBootApplication
public class CustomerSupportApp {

    @Bean
    ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            .defaultAdvisors(
                new QuestionAnswerAdvisor(vectorStore),
                new MessageChatMemoryAdvisor(chatMemory))
            .build();
    }

    @Bean
    HarnessAgent supportAgent(ModelRegistry models) {
        return HarnessAgent.builder()
            .name("support")
            .model(models.get("qwen-max"))
            .toolkit(Toolkit.builder()
                .tool(new OrderQueryTool())
                .tool(new RefundTool())       // 需要 PermissionEngine 审批
                .build())
            .build();
    }
}
```

一次用户请求在两个框架间的数据流：

```
用户输入 "查订单 #12345，如果超时未发货就退款"
  ↓
Spring AI ChatClient 接收 → Advisor 链（RAG+记忆+护栏）
  ↓
模型决策 → 调用 OrderQueryTool（Spring @Bean）
  ↓ 返回订单状态 "已超时，可退款"
  ↓
模型决策 → 调用 RefundTool
  ↓
AgentScope PermissionEngine 拦截 → 检查：
  ├─ 此用户有 refund 权限？ ✓
  └─ 金额 > ¥5000？ → 触发 human-in-the-loop 审批
  ↓
审批通过 → RefundTool 执行
  ↓ 返回 "退款成功"
  ↓
AgentScope AgentStateStore 持久化会话 → Redis
  ↓
Spring AI ChatClient 返回响应给用户
```

每个框架只负责自己擅长的那一层——Spring AI 管模型对话和 RAG，AgentScope 管权限、审批和持久化。两者通过 Spring 的依赖注入容器组合，不互相侵入。

**优点**：
- Spring AI 做集成层（自动装配、可观测性、Advisors）
- AgentScope 做运行时层（多租户、权限、沙箱、持久化）
- 两者通过 Spring Bean 组合，不冲突

**缺点**：需要同时学习两个框架。

### 方案 B：LangChain4j + AgentScope（非 Spring 团队）

```
[Quarkus/Micronaut/Spring 应用]
  │
  ├─ LangChain4j (调用层)
  │   ├─ AiServices          ← 声明式 Agent
  │   ├─ guardrails          ← 输入/输出护栏
  │   └─ Tool 注解            ← 工具定义
  │
  └─ AgentScope (运行时层)
      ├─ Permissions          ← 退款审批
      ├─ Multi-tenancy        ← 租户隔离
      └─ Sandbox              ← 工具隔离执行
```

```java
// LangChain4j 层
interface CustomerSupportAiService {
    @SystemMessage("你是一个客服助手")
    @ToolAnnotation(OrderQueryTool.class)
    String reply(String question);
}

// AgentScope 层做运行时
var agent = HarnessAgent.builder()
    .name("support")
    .model(openAiModel)
    .permissions(config.getPermissionPolicy())
    .build();
```

### 方案 C：Google ADK Java（GCP 原生）

```
[Google Cloud 环境]
  │
  └─ ADK Java (全栈 Agent)
      ├─ LLM Agent              ← Gemini
      ├─ Sequential Workflow    ← 标准客服流程
      ├─ A2A 协议                ← 与 Python agent 互通
      ├─ Session Management     ← Vertex AI 管理
      └─ Dev UI                 ← 开发调试
```

**适合**：已锁定 GCP 的团队。ADK 提供了一个从开发到部署到监控的完整闭环，但代价是平台锁定。

---

## 缺陷与批判

没有完美的框架。每个框架的设计选择背后都有权衡。

> **治理层说明**：本文五层模型中的治理层以 SwarmAI 为代表，但 SwarmAI 目前处于极早期（21 Stars），治理能力尚未成熟。实际项目中可在集成层（Spring AI Actuator + Micrometer 告警）或运行时层（AgentScope PermissionEngine 的审批日志）实现部分治理需求。治理层整体处于探索阶段，2026 下半年值得关注。

#### LangChain4j

- **没有运行时层**：这是它最明显的空白。多租户、沙箱、权限——这些都要自己实现。进入生产后这些问题的优先级会急剧上升
- **可观测性要靠外部**：不像 Spring AI 有 Micrometer 原生集成，LangChain4j 需要手动配置 OpenTelemetry 或依赖 Quarkus CDI
- **广度牺牲新颖性**：抽象 40+ 提供者意味着新能力（如 Gemini 的 grounding API）需要几周才能在框架中暴露

#### Spring AI

- **没有多 agent 编排**：v2.0 仍然没有解决这个问题。如果你需要多 agent 协作，必须依赖其他框架（Spring AI Alibaba、Embabel）
- **Spring 强绑定**：非 Spring 项目无法使用。对于 Quarkus 团队或裸 Java 项目，这不是选项
- **抽象不保证语义一致**：不同模型的 prompt 表现差异不会被框架抹平。换模型几乎总要调 prompt

#### Spring AI Alibaba

- **阿里云锁定**：Graph engine 虽然可以在任何基础设施运行，但最佳体验依赖 Nacos 和阿里云
- **版本追赶压力**：v2.0.0-M1.1 基于 Spring AI 2.0.0-M1，但 Spring AI 2.0.0 GA 已发布。版本差距可能需要额外注意
- **文档中文优先**：国际团队可能面临文档语言壁垒

#### AgentScope Java

- **学习曲线陡峭**：Project Reactor 的非阻塞模型 + 多层抽象（Agent/Harness/Middleware/Workspace/Event）对初学者不够友好
- **生态尚小**：模型适配器只有主流几家（OpenAI、Anthropic、DashScope、Gemini、Ollama），和 LangChain4j 的 40+ 覆盖差距明显
- **Python 优先于 Java**：AgentScope 的发源是 Python 版，Java 版虽然独立发展但仍受 Python 版设计影响。部分文档和示例先出 Python 版
- **Spring 集成未官方化**：虽然框架设计是框架无关，但缺少 Spring Boot Starter 官方支持

#### Embabel

- **非 GA**：v1.0.0-RC1，社区版协议，没有生产 SLA。用于生产需要更强的风险承受能力
- **Kotlin 优先**：虽然 Java 互操作良好，但生态工具和社区示例以 Kotlin 为主
- **GOAP 的不确定性**：A* 规划虽然在确定性上优于 LLM 推理，但规划路径仍可能与预期不符。调试工具还不够成熟
- **社区年轻**：60 个贡献者、15 个版本——遇到问题时可依靠的社区资源有限

#### Google ADK Java

- **GCP 强绑定**：ADK 的完整能力（评估、监控、部署、SLA）都依赖 Vertex AI 和 Google Cloud。离开 GCP 等于放弃一半功能
- **Agent-only 架构**：没有普通的 generation 或 flow 抽象。如果你只需要简单的 "prompt → response"，ADK 太重了
- **Python 设计余味**：API 风格明显受 Python SDK 影响，对 Java 开发者来说不够自然

#### Semantic Kernel Java

- **事实停滞**：超过 14 个月没有正式发布，社区贡献接近零。作为长期依赖项风险极高
- **Azure 强绑定**：即使活跃，它的最佳集成体验也面向 Azure OpenAI 和 Azure Cognitive Search
- **Reactor 负担**：Mono/Flux 引入的复杂性在你只需要同步调用时显得多余

---

## 总结事物本身：决策树与迁移路径

### 决策树

```
你的项目用什么框架？
│
├─ Spring Boot
│   ├─ 只需要简单 AI 集成？
│   │   └─ ✅ Spring AI（最省力）
│   ├─ 需要多 agent 工作流？
│   │   └─ ✅ Spring AI + Spring AI Alibaba
│   ├─ 需要生产运行时（多租户/沙箱/权限）？
│   │   └─ ✅ Spring AI + AgentScope
│   └─ 已经在阿里云？
│       └─ ✅ Spring AI Alibaba（一站式）
│
├─ Quarkus / Micronaut / 框架无关
│   ├─ 需要最大模型兼容性？
│   │   └─ ✅ LangChain4j
│   ├─ 需要多 agent 编排？
│   │   └─ ✅ LangChain4j + Embabel / LangChain4j + AgentEnsemble（LangChain4j 生态的多 agent 编排层）
│   └─ 需要运行时层？
│       └─ ✅ LangChain4j + AgentScope
│
└─ Google Cloud
    ├─ 全栈 GCP 用户？
    │   └─ ✅ Google ADK Java
    └─ 多云策略？
        └─ ✅ 参考 Spring 方案，适配 Gemini 模型
```

### 三套推荐组合

#### 组合一：Spring 团队的标准方案

```
Spring AI + AgentScope Java
```
**最适合**：大多数企业 Spring Boot 项目
**能力覆盖**：模型调用 → 自动装配 → 可观测 → 多租户 → 权限 → 沙箱 → 持久化
**缺失**：复杂多 agent 编排（需要时可加 Spring AI Alibaba Graph）

#### 组合二：非 Spring 团队的灵活方案

```
LangChain4j + AgentScope Java
```
**最适合**：Quarkus/Micronaut 项目、框架中立团队
**能力覆盖**：最大生态兼容 → 运行时层 → 权限 → 沙箱 → 持久化
**缺失**：自动装配（手动配置）、多 agent 编排（需要时可加 Embabel/AgentEnsemble）

#### 组合三：阿里云原生方案

```
Spring AI Alibaba + AgentScope Java
```
**最适合**：深度绑定阿里云的国内企业
**能力覆盖**：Graph 编排 → 多 agent → 运行时层 → 阿里云服务集成
**注意**：依赖 Nacos 和阿里云基础设施

### 迁移路径

> **一个务实的策略**：不追求一步到位，而是沿着能力需求逐步演进。

```
原型期 (1-2 周)
  │  选择一个调用层或集成层框架
  │  Spring AI 或 LangChain4j
  │  目标：快速验证 AI 能力
  ▼
生产期 (1-2 月)
  │  加入运行时层
  │  + AgentScope Java
  │  目标：多租户、权限、持久化、沙箱
  ▼
企业级 (3-6 月)
  │  加入编排层和治理层
  │  + Spring AI Alibaba Graph / Embabel
  │  + SwarmAI（预算/审批/审计）
  │  目标：复杂编排、成本控制、合规
```

**关键原则**：每层独立演进。你可以在原型期只用 Spring AI（集成层），生产期不改变任何业务代码的情况下引入 AgentScope（运行时层），因为两者的抽象层次不同——它们操作的是不同层面的关注点。

### 展望：下半年值得关注的三个演变

截至 2026 年 7 月，Java AI 框架生态仍处于快速演进中，以下三个趋势值得持续关注：

1. **A2A 协议的标准化进程**：LangChain4j、Embabel、ADK Java 均已支持 A2A，但具体实现互有差异。如果 A2A 走向事实标准，框架之间的互操作性将根本改变当前的选型逻辑——运行时层可以用 AgentScope、编排层可以用 Embabel、调用层可以用 LangChain4j，三者通过 A2A 通信，真正实现"分层专业化 + 协议统一"
2. **Embabel 的 GA 时间线**：v1.0.0-RC1 是重要里程碑。如果 2026 下半年 Embabel 正式 GA（Rod Johnson 的 Spring 生态背景为其提供了天然的企业信任基础），Java AI 编排层可能出现一个不依赖 ReAct 的独立路线，挑战当前 LangChain4j 和 Spring AI Alibaba 的编排方案
3. **AgentScope Java 的 Spring 集成进度**：AgentScope 2.0 GA 后，如果提供官方 Spring Boot Starter，将大幅降低其与 Spring AI 的组合门槛——但目前仍缺失这一环节。社区在等待官方动态

### 写在最后每个框架清晰地解决特定层面的问题，而不是试图成为"一个框架通吃所有"的银弹。

对 Java 开发者来说，最好的消息是：你不用在框架之间痛苦地做单选题。**理解每个框架的生态位，在架构上做好分层决策，然后组合使用**——这本就是 Java 生态多年来的工程传统（Spring + Hibernate + Redis + Kafka），AI 框架也不例外。

如果你只记住一件事：

> 选择框架时，先问自己"我当前最痛的问题是哪个层次的"——是调不通模型（调用层）、配不好环境（集成层）、跑不顺多 agent（编排层）、还是上不了生产（运行时层）？**回答了这个问题的框架，才是你现在需要的框架。**

### 参考

- [LangChain4j GitHub](https://github.com/langchain4j/langchain4j) — v1.18.0
- [Spring AI GitHub](https://github.com/spring-projects/spring-ai) — v2.0.0
- [Spring AI Alibaba GitHub](https://github.com/alibaba/spring-ai-alibaba) — v2.0.0-M1.1
- [AgentScope Java GitHub](https://github.com/agentscope-ai/agentscope-java) — v2.0.0
- [Embabel Agent Framework](https://github.com/embabel/embabel-agent) — v1.0.0-RC1
- [Google ADK Java](https://github.com/google/adk-java) — v1.6.0
- [Agents-Flex](https://github.com/agents-flex/agents-flex) — v2.2.2
- [Semantic Kernel Java](https://github.com/microsoft/semantic-kernel-java) — java-1.4.4-RC1
- [SwarmAI](https://github.com/intelliswarm-ai/swarm-ai) — v1.0.26
- Spring I/O 2026: Comparing Agentic AI Frameworks for Java — Timo Salm, Sandra Ahlgrimm
- CodeWiz: Java AI Agent Frameworks in 2026 — May 2026
- Baeldung: Creating an AI Agent in Java Using Embabel — Jul 2025
- [本站: LangChain4j 与 Spring AI 深度对比](/2026/07/21/ai-langchain4j-practice/)
- [本站: AgentScope 2.0 深度解读](/2026/06/26/ai-agentscope-deep-dive/)
