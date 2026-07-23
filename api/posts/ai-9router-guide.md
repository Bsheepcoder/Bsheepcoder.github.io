## 元认知：一个现象，而非一个产品

先不解释 9Router 是什么。先问：为什么会有这样一个项目？

我们正在经历一个 AI 编码工具爆炸的阶段。[Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) 用 Messages API 走 OAuth，[Codex](https://codex.so) 用 Chat Completions API 走订阅，Cursor 封闭在 IDE 里，[Gemini CLI](https://github.com/google-gemini/gemini-cli) 已经关停，[Kiro](https://github.com/kirodotdev/kiro) 有免费 credits，[OpenCode](https://github.com/anomalco/opencode) 有免验证的 passthrough，还有 GLM、MiniMax、Kimi、DeepSeek……每个提供商有各自的 API 格式、计费单位、配额周期、重置时间。开发者要记住的不只是一个 API key，而是一整套碎片化的访问规则。

9Router 就是对这种碎片化的自组织响应。它不做任何"AI"——不训练模型，不写推理逻辑，不做代理池，不搞 IDE 集成。它只做一件事：**路由器**。把请求从工具那里接过来，决定走哪条路径发出去，把结果翻译回工具能理解的格式。

这个项目在 [GitHub](https://github.com/decolua/9router) 上已经有 22K Stars，npm 周下载量 48.7K，190 个贡献者，6 个月内发布了 73 个版本。增长速度本身就是信号：这个需求不是被创造出来的，是**已经被感受到的**。

```plaintext
你的 CLI 工具                         9Router                          AI 提供商
(Claude Code / Cursor           ┌─────────────────────┐       ┌─ Claude Code (订阅)
 / Codex / Cline / Copilot)     │  三层回退 + RTK     │       ├─ GLM ($0.6/1M)
         │                      │  + 格式翻译          │       ├─ Kiro (免费)
         └── http://localhost:20128/v1                  │       └─ ...
                                └─────────────────────┘
```

技术上讲，9Router 是一个运行在本地的 OpenAI 兼容代理。但更有趣的是它的**存在本身**——它不属于任何大厂的战略规划，没有运营团队，没有商业化压力，就是一个开源社区对"AI 工具太多太乱"的自发回应。它在这个生态里扮演的角色，和 DNS 解析器在互联网里的角色有结构上的相似：不产生内容，但让内容可以被找到、被路由到正确的地方。

## 碎片化即冗余，冗余即可靠

9Router 最核心的贡献是三层回退架构。

当开发者向 9Router 发送一个请求时，它按优先级依次尝试：

- **第一层：订阅**。Claude Code Pro/Max、Codex Plus/Pro、GitHub Copilot、Cursor IDE。你已经付了月费，每多一次调用都是"免费"的。
- **第二层：廉价 API**。GLM-5.1 约 $0.6/1M tokens，MiniMax M2.7 约 $0.2/1M。日重置或 5 小时滑动窗口。
- **第三层：免费层**。Kiro AI 约 50 credits/月、OpenCode Free 免验证、Vertex AI 新用户 $300 额度。

当第一层配额耗尽或出现错误，自动降级到第二层，按需继续到第三层。

```plaintext
Combo: "my-coding-stack"
  1. cc/claude-opus-4-6             # 订阅, 5小时重置
  2. glm/glm-5.1                    # 廉价, $0.6/1M, 日重置
  3. kr/claude-sonnet-4.5           # 免费, Kiro credits
```

这个设计的技术实现并不复杂——检查响应状态码、统计 token 用量、比较阈值。真正有意思的是它背后的逻辑：**碎片化不是缺陷，碎片化是冗余的来源**。

单一提供商的问题是单点故障——配额用完了就是完了，不能提前充值，不能临时扩容。但 40+ 个提供商，每个的配额周期不同（有些 5 小时、有些日重置、有些月重置）、定价策略不同（有些按量、有些月费固定、有些靠免费引流），组合起来就有了套利空间。9Router 本质上是一套**配额套利系统**：在某一个时间点，总有一个提供商的配额是充足的。

这个逻辑有一个朴素但有力的洞察：当你的资源池足够多样化，单点的不确定性会收敛为系统的确定性。每个提供商各自设定配额和定价时，并未考虑彼此——但恰好是这种独立性，让它们组合后变得可靠。

## 格式翻译：被低估的复杂度

跨提供商路由不只是一个成本问题，它还是一个兼容性问题。

OpenAI 的 Chat Completions API 用 `role` 区分 user/assistant/system/tool，用 `tool_calls` 表达工具调度，`content` 可以是字符串或数组。Anthropic 的 Messages API 用 `role` 加 `content` 块，image 用 `source` 包裹 base64 和 media_type，thinking 块有独立的 `type: thinking`。Gemini 的结构又不同——`contents` 列表、`parts` 数组、`inlineData` 加 `mimeType`。

这三家的 API 格式不是"风格差异"——是结构差异。system prompt 的放置位置不同、工具调用参数的 schema 不同、多模态内容的编码方式不同、流式响应的事件类型不同。一个代理从 OpenAI 格式进来，走 Claude 出去，需要做的不只是改字段名，而是重新编排消息结构。

```plaintext
# OpenAI 格式
{"role": "system", "content": "..."}
{"role": "user", "content": [{"type": "text", "text": "..."}, {"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}]}
{"role": "assistant", "tool_calls": [{"id": "...", "function": {"name": "read_file", "arguments": "{\"path\": \"...\"}"}}]}

# Claude 格式
{"role": "user", "content": [{"type": "text", "text": "..."}, {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": "..."}}]}
{"role": "assistant", "content": [{"type": "tool_use", "id": "...", "name": "read_file", "input": {"path": "..."}}]}
```

9Router 的处理方式是：工具端统一收 OpenAI 格式，内部做一次规范化，再根据目标提供商的格式要求做一次翻译。这套映射是 9Router 真正的技术门槛——它覆盖的 40+ 提供商不是各有一个适配器，而是共享一套格式抽象。

这个抽象边界选得准确：它不试图统一 AI 模型的能力差异（有些支持并行工具调用有些不支持，有些有 thinking 块有些没有——这些差异暴露给上层，不做抹平），只统一传输层的格式差异。

## Token 治理：你为噪音付费

比路由更深一层的是 Token 优化。9Router 内置了 RTK 集成，这不是一个"nice to have"的功能，而是对整个 AI 编码工作流的一个重要的观察。

AI 编码助手（Claude Code、Cursor、Cline 这类 agentic coding tool）的大部分 token 消耗，不是来自"推理"，而是来自**读取工具输出**。每次 agent 调用 `git diff`、`ls -la`、`grep -r`、`find .`、`cat file`，返回的都是未经压缩的原始 shell 输出。一个典型的 `git diff` 可以轻松达到数千行，包含大量上下文行、文件头、diff 标记——agent 真正需要的是"哪些文件变了，改了什么"，而不是完整的 diff 正文。

这就是 [RTK](https://github.com/rtk-ai/rtk) 的切入点。它是一个用 Rust 编写的 CLI 代理，零依赖，<10ms 开销，在 shell 命令和 LLM 之间插入一层 filter，将常见命令的输出压缩到极致：

```plaintext
# git status（典型 15 行） → rtk git status（1 行）
On branch main
Your branch is up to date with 'origin/main'.
Changes not staged for commit:
  modified:   src/agent.rs
  modified:   src/tool.rs
  Untracked files: tests/

# cargo test（失败时 200+ 行） → rtk cargo test（约 20 行）
FAILED: 2/15 tests
  test_parse: assertion failed at src/parser.rs:42
  test_overflow: panic at src/tool.rs:18
```

压缩策略分四类：

- **智能过滤**——移除注释、空白行、样板信息
- **分组聚合**——按目录归类文件、按规则归类 lint 错误
- **截断**——保留上下文，砍掉冗余
- **去重**——用计数代替重复日志行

这看起来像一个简单的"文本处理工具"，但它的存在说了一个事实：**AI 编码助手的 token 消耗结构，和传统 API 调用的 token 消耗结构根本不同。**传统场景下，用户输入是 token 消耗的主因素，工具输出很少被传给 LLM。但在 agentic coding 场景中，agent 每轮推理都会调用多次工具，每次调用返回大量 shell 输出，这些输出占 prompt 的 30-50%。一个 $20/月的订阅，可能有 $6-10 花在了让 LLM 阅读 `ls` 输出上。

[RTK 的 GitHub](https://github.com/rtk-ai/rtk) 上写着 72.7K Stars，这比大部分 AI 框架本身的 Star 数还高。这个数字本身就是一个启示：**在 AI 编码普及之后，第一个大规模爆发的不是新的 AI 能力，而是 Token 治理。**优化既有的浪费，比发明新功能更能立刻产生价值。

### Caveman 和 Ponytail：输出端的镜像

RTK 做输入压缩，Caveman（52K ⭐）和 Ponytail 做输出压缩。

[Caveman](https://github.com/JuliusBrussee/caveman) 注入一句"原始人说话"风格的系统提示——要求 LLM 用最简短的语言回答，去掉客套、去掉解释、去掉不必要的形式化措辞。[Ponytail](https://github.com/DietrichGebert/ponytail) 注入"资深懒程序员"的思维模式——倾向于删除代码而非添加，倾向于标准库而非新依赖，倾向于一行代码而非抽象。

这三者构成了一个完整的 token 治理体系：
```plaintext
输入                   处理                    输出
工具输出               推理                    LLM 回复
  │                     │                       │
  │ RTK 压缩            │                       │
  │ (60-90% 减少)       │                       ├── Caveman (输出简短)
  │                     │                       └── Ponytail (代码最少)
  ↓                     ↓                       ↓
[压缩后的工具输出] → [LLM 推理] → [被优化的回复]
```

有意思的是，这个治理体系是事后加上去的——它不是 AI 编码工具原生设计的一部分，而是在使用过程中发现"太多 token 浪费在噪音上"之后，社区自发补充的优化层。

## 代价与边界

9Router 不是没有代价。

第一，它是一个本地运行的 Node.js 服务。进程在，路由在；进程崩，编码停。虽然部署方式可选 Docker 或 VPS，但默认情况下，它绑定在 localhost:20128，每一个 AI 编码请求都经过这个进程。这意味着多了一层故障点、多了一层延迟（虽然可以忽略不计）、多了一层内存占用。

第二，免费层的动态变化存在不确定性。[iFlow](https://github.com/iFlow-ai/iFlow-cli) 免费服务已停，[Qwen Code](https://github.com/QwenLM/qwen-code) 在 2026-04-15 停用了免费 OAuth 层，[Gemini CLI](https://github.com/google-gemini/gemini-cli) 在 2026-06-18 全量关停，[Kiro](https://github.com/kirodotdev/kiro) 从不限量免费转为约 50 credits/月的受限模式。依赖免费层的路由策略，本质上是将自身可靠性建立在他人未承诺的善意之上。这不是 9Router 的问题，是所有"免费"服务的固有时限性。

第三，安全边界需要关注。9Router 作为一个本地进程，持有所有提供商 API key 和 OAuth token。在单机开发环境中，这通常是可接受的。但在零信任网络、安全审计严格的环境中，一个持有 40+ 提供商凭证的进程会成为攻击面。9Router 不会上传你的 key，但它运行在你的机器上，和你的终端一样——谁控制了你的机器，就控制了你的路由。

第四，9Router 的迭代速度极快——73 releases 在 6 个月，平均每 2.5 天一个版本。高频迭代意味着功能快速演进，也意味着 API 不稳定、配置变更频繁、文档可能滞后。RTK 采用了 Rust + 零依赖 + <10ms 开销的极致工程，而 9Router 本身是 Next.js 应用加上 Node.js 代理层——这个技术栈差异暗示了一个有趣的取舍：路由器本身不需要极致性能（因为网络延迟才是瓶颈），但压缩器需要（因为它在关键路径上）。

## 何时用

9Router 适合的场景：

- 你同时使用多个 AI 编码工具（Claude Code + Cursor + Codex），厌倦了为每个工具单独配置 provider
- 你的月订阅配额用不完或用不够——想让 $20 覆盖更多编码时间
- 你想让编码过程不因配额耗尽而中断——死线压力下，工具不可用的成本高于工具的订阅成本
- 你想了解自己的 token 消耗分布——dashboard 实时追踪每个 provider、每个 model 的使用情况
- RTK 适合所有人：无感安装，基准开销可以忽略，对编码体验无副作用

不适合的场景：

- 安全敏感环境——代码和凭证必须严格隔离的合规场景
- 需要确定性 provider 回应的生产 CI——你的构建流水线不应该依赖"如果 A 超限就试 B"的逻辑
- 纯 API 调用场景——如果你只是调 API 做推理，不需要 agent 路由，直接调就好

## 这件事有趣在哪里

9Router 作为一个项目，它告诉了我们几件事。

> **1. AI 编码工具进入基础设施化阶段**  
> 提供商超过 40、工具超过 15、格式超过 6 种之后，开发者自然会搭建中间层管理这种复杂。这和 Web 发展史如出一辙——服务器和浏览器数量爆炸时，CDN、反向代理、DNS 负载均衡应运而生。AI 编码正在经历类似阶段，区别在于从「需要路由」到「出现路由器」的间隔以月为单位，而非年。
>
> **2. Token 经济催生新的工程范式**  
> 传统 API 成本优化关注请求数量；AI 编码助手的成本结构完全不同——一次请求含数十次工具调用，每次返回数千行输出。优化「每请求成本」无效，需优化「每 token 价值」。RTK、Caveman、Ponytail 皆从 token 价值出发：压缩低价值工具输出和模型回复，把 token 预算留给推理。
>
> **3. 开源自组织从「建新」转向「连接」**  
> 9Router 的 190 位贡献者几乎不写 AI 代码，不写 LLM、agent 逻辑、IDE 插件。他们写胶水——把已有的 API、工具、协议、计费系统粘在一起。2026 年最大的工程瓶颈或许不是「能不能造更好的 AI 模型」，而是「能不能让已有 AI 能力以可负担的成本可靠地抵达用户」。9Router 只做后者，因此获得 22K Stars。
>
> **4. 项目的成功取决于「不做什么」的精确度**  
> 9Router 不做代理池、模型租赁、UI IDE、训练平台、评测基准。README 第一句是「Never stop coding」，但代码库里没有一个文件处理 coding——它只处理路由。选择边界，在边界之内做到极致，比功能大而全但每个功能都平庸的项目更持久。

## 怎么用

9Router 的安装和使用已经简化到极限。以下是最快路径。

安装：

```bash
npm install -g 9router
9router
```

启动后 dashboard 在 `http://localhost:20128`。OpenAI 兼容端点在 `http://localhost:20128/v1`。

在 Dashboard → Providers 中连接一个免费 provider（Kiro AI、OpenCode Free），复制 API key。

然后在你的编码工具中配置：
```plaintext
Claude Code / Cursor / Codex / Cline 设置:
  Endpoint: http://localhost:20128/v1
  API Key: <从 dashboard 复制>
  Model: kr/claude-sonnet-4.5
```

RTK 集成默认开启。如果希望获得更大的 Token 节省，可以额外安装 Caveman 或 Ponytail，在 Dashboard → Endpoint 中启用。

对于生产环境，建议通过 Docker 部署：

```bash
docker run -d \
  -p 20128:20128 \
  -v 9router-data:/app/data \
  decolua/9router
```

---

这不是 9Router 的文档，也不是教程。这是一个项目现象的分析，一个关于"当 AI 工具太多时会发生什么"的样本观察。你可以不同意其中的任何判断，但你不能否认：22K Stars 是一个信号。这个信号说的是，AI 编码的下一个瓶颈不是模型能力，而是基础设施。
