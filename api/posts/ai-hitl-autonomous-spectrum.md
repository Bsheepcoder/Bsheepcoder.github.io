## 元认知：人类参与不是"要不要"，而是"在哪里"

当我们讨论 AI Agent 是否应该"自主运行"时，问题的本质不是"要不要人类参与"，而是"在什么环节、以什么程度参与"。

这个问题的答案，决定了 AI 产品的架构设计、用户体验和安全边界。

### 三种模式的定义

| 模式 | 英文 | 定义 | 人类角色 |
|------|------|------|----------|
| **人在环中** | Human-in-the-Loop (HITL) | 每一步都需要人类审批才能继续 | 审批者 |
| **人在环上** | Human-on-the-Loop (HOTL) | AI 自主运行，人类监控并可随时干预 | 监控者 |
| **人在环外** | Human-out-of-the-Loop (HOFL) | 完全自主，人类只看最终结果 | 验收者 |

这三种模式不是"好"与"坏"的区分，而是**不同场景下的最优选择**。

---

## 搭积木：从 Anthropic 的架构说起

Anthropic 在《Building Effective Agents》中提出了一个清晰的架构框架：

### Workflow vs Agent

> **Workflow**：LLM 和工具通过预定义代码路径编排
> **Agent**：LLM 动态指导自己的流程和工具使用

这个区分至关重要：
- **Workflow** 适合确定性任务（如数据处理流水线）
- **Agent** 适合开放性任务（如代码修复、问题解答）

### Anthropic 的五种工作流模式

Anthropic 总结了五种常见的工作流模式，每种模式对人类参与的要求不同：

| 模式 | 人类参与度 | 适用场景 |
|------|------------|----------|
| **Prompt Chaining** | 高（每步检查） | 任务可分解为固定子任务 |
| **Routing** | 中（分类后自动） | 输入有明确分类 |
| **Parallelization** | 中（聚合时审查） | 子任务可并行 |
| **Orchestrator-Workers** | 低（仅验证结果） | 子任务动态确定 |
| **Evaluator-Optimizer** | 低（迭代自优化） | 有明确评估标准 |

### Agent 的人类参与设计

对于真正的 Agent（自主运行），Anthropic 的设计是：

> "Agents can then pause for human feedback at checkpoints or when encountering blockers."

关键设计：
1. **检查点暂停**：在关键节点等待人类反馈
2. **阻塞求助**：遇到无法解决的问题时主动求助
3. **停止条件**：设置最大迭代次数等安全边界

---

## 案例即原理：OpenAI Codex 的实践

OpenAI 在 2025 年 5 月发布的 Codex，是 HOTL 模式的典型实践。

### Codex 的架构

```
用户提出任务
    ↓
Codex 在云端沙箱自主运行
    ↓
用户可实时监控进度
    ↓
Codex 完成后提交变更
    ↓
用户审查并验证结果
    ↓
用户决定是否集成
```

### 人类参与的三个层面

**1. 任务定义层**

> "Users can assign tasks by typing a prompt and clicking 'Code'."

人类定义任务目标，AI 规划执行路径。

**2. 执行监控层**

> "You can monitor Codex's progress in real time."

人类可以实时查看 AI 的执行状态，但不干预。

**3. 结果验证层**

> "It still remains essential for users to manually review and validate all agent-generated code before integration and execution."

人类必须验证最终结果，这是**不可省略**的环节。

### 可验证性设计

OpenAI 的核心洞察是：**HOFL 需要可验证性**。

Codex 提供了三种验证机制：
1. **Citations**：引用文件路径和行号
2. **Terminal Logs**：展示命令执行过程
3. **Test Results**：展示测试通过情况

> "When uncertain or faced with test failures, the Codex agent explicitly communicates these issues, enabling users to make informed decisions about how to proceed."

---

## 缺陷与批判：三种模式的陷阱

### HITL 的陷阱：效率瓶颈

HITL 的问题是**人类成为瓶颈**：
- 每一步都需要人类审批
- 人类的工作时间成为限制
- 人类的认知负荷成为上限

**案例**：早期的 AI 辅助诊断系统，每张 X 光片都需要医生确认，导致系统无法规模化。

### HOTL 的陷阱：监控疲劳

HOTL 的问题是**人类监控疲劳**：
- 长时间监控导致注意力下降
- 异常事件被忽略
- "狼来了"效应

**案例**：自动驾驶系统要求人类随时接管，但人类在长时间无干预后反应速度下降。

### HOFL 的陷阱：错误累积

HOFL 的问题是**错误累积**：
- 没有人类干预，错误会传播
- 小错误累积成大问题
- 发现时已经造成不可逆影响

**案例**：高频交易算法在没有人类监控的情况下，因小错误累积导致巨额亏损。

---

## 前沿方向：渐进式自治

### Anthropic 的"角色训练"思路

Anthropic 在《Claude's Character》中提出了一个深刻的洞察：

> "We want people to know that they're interacting with an imperfect entity with its own biases and with a disposition towards some opinions more than others."

这意味着：
- AI 应该知道自己的局限性
- AI 应该在不确定时主动求助
- AI 应该透明地展示自己的推理过程

**品格训练**的三个核心特质：
1. **诚实**：不隐瞒自己的局限性
2. **谦逊**：不假装自己是客观真理
3. **好奇**：对不同观点保持开放

### OpenAI 的"渐进式部署"策略

OpenAI 在《Practices for Governing Agentic AI Systems》中提出了渐进式策略：

> "Agentic AI systems—AI systems that can pursue complex goals with limited direct supervision—are likely to be broadly useful if we can integrate them responsibly into our society."

**三个阶段**：
1. **第一阶段**：HITL（每步审批）→ 建立信任
2. **第二阶段**：HOTL（监控+可干预）→ 扩大规模
3. **第三阶段**：HOFL（自主运行+人类验证）→ 最大效率

### 自适应自治度

最前沿的观点是：**AI 应该根据场景自动选择合适的自治度**。

| 场景 | 自治度 | 理由 |
|------|--------|------|
| 低风险、明确任务 | HOFL | 效率优先 |
| 中等风险、复杂任务 | HOTL | 平衡效率与安全 |
| 高风险、关键任务 | HITL | 安全优先 |

**实现路径**：
1. **风险评估**：AI 评估任务的风险等级
2. **动态调整**：根据风险等级调整自治度
3. **人类确认**：高风险任务自动切换到 HITL

---

## 总结：选择合适的模式

| 问题 | 答案 |
|------|------|
| HITL 的反面是什么？ | Human-out-of-the-Loop (HOFL) |
| 目前主流是什么？ | Human-on-the-Loop (HOTL) |
| OpenAI 的立场？ | HOTL + 人类必须验证 |
| Anthropic 的立场？ | 检查点式 HITL + 沙箱测试 |
| 前沿方向是什么？ | 渐进式自治 + 自适应自治度 |

**核心结论**：

HITL、HOTL、HOFL 不是"好"与"坏"的区分，而是**不同场景下的最优选择**。当前的共识是 **HOTL**——Agent 自主执行，人类监控并验证。未来的方向是**自适应自治度**——AI 根据场景自动选择合适的自治级别。

正如 OpenAI 所说：

> "We imagine a future where developers drive the work they want to own and delegate the rest to agents—moving faster and being more productive with AI."

这不是"AI 替代人类"，而是"AI 与人类协作"的新范式。

---

*参考资料：*
- *Anthropic, "Building Effective Agents", 2024*
- *Anthropic, "Claude's Character", 2024*
- *OpenAI, "Introducing Codex", 2025*
- *OpenAI, "Practices for Governing Agentic AI Systems", 2023*
