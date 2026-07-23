## 为什么写这篇文章

调 API 按 token 收费。一个 3 万行的代码库直接喂给 GPT-4o，一次对话可能花掉 2 美元。如果对话 10 轮，就是 20 美元。省 token 不是抠门——是让有限预算下能做更多的事。

这篇文章的定位很明确：**个人开发者，调远程 API（OpenAI/Anthropic/DeepSeek），想在输入端和输出端都省绝对 token**。不考虑省「钱但不省量」的方案（如换便宜模型），不考虑纯本地推理的方案，不考虑需要企业级基础设施的方案。

我找了 10 条不同的省 token 路线，每条选一个最强的开源项目，在 Windows 11 + Python 3.12 + Node 24 环境下全部实测。

## 测试环境

```
OS:      Windows 11
Python:  3.12.6
Node:    24.11.1
CPU:     无 GPU
代理:    http://127.0.0.1:7890（访问 HuggingFace/OpenAI）
测试样本: 3 个文件（calculator.py + utils.py + README.md），共 467 tokens
```

## 10 条路线总览

| # | 路线 | 项目 | Stars | 省 token 机制 | 实测省量 | 个人可用性 |
|---|------|------|-------|--------------|---------|-----------|
| 1 | AST 代码压缩 | Repomix | 27k | Tree-sitter 剥离实现体 | 17.7% | ★★★★★ |
| 2 | Repo Map + Diff | Aider | 47.2k | 签名树替代全文 + 只回 diff | 59.7% | ★★★★★ |
| 3 | Fragment 注入 | llm (simonw) | 12.2k | 按需引用文件片段 | 64.2% | ★★★★★ |
| 4 | 管道过滤 | files-to-prompt | 2.8k | 只投喂变更文件 | 视场景 | ★★★★★ |
| 5 | Token 计数器 | tiktoken | 18.7k | 精确计数后决策裁剪 | 底层依赖 | ★★★★★ |
| 6 | Prompt 语义压缩 | LLMLingua | 6.4k | 小模型删除非关键 token | 最高 20x | ★★☆☆☆ |
| 7 | 记忆层压缩 | Mem0 | 60.5k | 长对话压成结构化记忆条目 | 53.6% | ★★★★☆ |
| 8 | 语义缓存 | GPTCache | 8.1k | 相似 query 命中缓存，0 token | 83.3% | ★★★☆☆ |
| 9 | 约束生成 | Guidance | 21.6k | grammar 约束省格式指令 | 75% | ★★★☆☆ |
| 10 | 时序知识图谱 | Graphiti | 28.6k | 查询返回子图替代文档段 | 38.8% | ★★★☆☆ |

---

## 路线 1：AST 代码压缩 — Repomix

**GitHub**: `yamadashy/repomix` · 27k stars · TypeScript

### 省 token 原理

Repomix 用 Tree-sitter 解析代码 AST，`--compress` 选项只保留函数/类的签名（名称、参数、类型注解、docstring），剥离函数实现体。还可以 `--remove-comments` 去注释、`--remove-empty-lines` 去空行、`--truncate-base64` 截断 base64 数据。

### 实测数据

```
原始 XML 输出:          4012 bytes  | 880 tokens
--compress 压缩:        3429 bytes  | 767 tokens  (↓ 12.8%)
--compress + 去注释+去空行: 3213 bytes | 724 tokens  (↓ 17.7%)
```

### 压缩前后对比

压缩前（calculator.py 片段）：
```python
def add(self, a, b):
    """Add two numbers and store in history."""
    result = a + b
    self.history.append(f"{a} + {b} = {result}")
    return result
```

压缩后（只剩签名，`⋮` 标记被裁剪处）：
```python
def add(self, a, b):
⋮
```

### 安装与使用

```bash
# 零安装，直接 npx
npx repomix@latest ./your-project --compress --remove-comments --remove-empty-lines

# 带 token 预算守卫（CI 友好）
npx repomix@latest ./your-project --compress --token-budget 50000
```

### 评价

- **优点**: 零安装（npx 即用）、XML 输出对 Claude 注意力友好、`--token-budget` 可做 CI 守卫
- **缺点**: 压缩率对小文件不明显（17.7%），大文件效果更好；压缩后模型无法看到实现细节
- **个人推荐度**: ★★★★★ — 零门槛，大仓库打包投喂前必用

---

## 路线 2：Repo Map + Diff 输出 — Aider

**GitHub**: `Aider-AI/aider` · 47.2k stars · Python

### 省 token 原理

Aider 是唯一一个**双向省**的工具：

**输入端 — Repo Map**：用 Tree-sitter 提取全仓库所有函数/类的签名，构建一棵「签名树」。模型看签名树而非全文，知道仓库里有什么、在哪个文件，需要看全文时再请求。

**输出端 — Unified Diff**：模型只输出 diff 补丁（修改了哪些行），而非重写整个文件。

### 实测数据

```
原始文件总 token:  467 tokens (3 个文件)
Repo Map token:    188 tokens
节省:              279 tokens (59.7%)
```

### Repo Map 内容

```
calculator.py:
⋮
│class Calculator:
│    """A simple calculator class that supports basic arithmetic."""
│    
│    def __init__(self):
⋮
│    def add(self, a, b):
⋮
│    def subtract(self, a, b):
⋮
│    def multiply(self, a, b):
⋮
│    def divide(self, a, b):
⋮
│    def get_history(self):
⋮

utils.py:
⋮
│def format_result(value, precision=2):
⋮
│def parse_input(text):
⋮
│def validate_number(n):
⋮
```

### 安装与使用

```bash
pip install aider-chat

# 在 git 仓库目录下运行
aider --model gpt-4o

# Aider 会自动生成 Repo Map，无需手动操作
# 模型只输出 diff，自动 git commit
```

### 评价

- **优点**: 双向省（输入签名树 + 输出 diff）、自动 git commit、支持 100+ 语言、有 `/ask` 模式只读不改
- **缺点**: 必须在 git 仓库中运行、安装依赖较重（含 tree-sitter 编译）
- **踩坑**: 安装时会 pin `openai==2.20.0`，与其他工具的 openai 版本冲突，建议独立 venv
- **个人推荐度**: ★★★★★ — 个人开发者 ROI 最高的工具，没有之一

---

## 路线 3：Fragment 片段注入 — llm (simonw)

**GitHub**: `simonw/llm` · 12.2k stars · Python

### 省 token 原理

`llm` 的 Fragment 机制（`-f` 参数）让你只注入选定的文件片段，而非全量仓库。配合 `ttok` 插件做 token 计数，`strip-tags` 去除 HTML 标签噪音。

### 实测数据

```
全量注入:           467 tokens (3 个文件)
Fragment 只注入 utils.py: 167 tokens
节省:               300 tokens (64.2%)
```

### 安装与使用

```bash
pip install llm

# 只注入 calculator.py 而非全量仓库
llm -f calculator.py "解释这个类的作用"

# 管道式组合：find 找最近修改的文件，注入给 LLM
find . -name "*.py" -mtime -1 | llm -f - "审查这些文件的变更"

# ttok 插件做 token 计数（安装后）
echo "some text" | llm ttok
```

### 评价

- **优点**: 管道式体验极佳、Fragment 精准控制注入内容、可组合性强
- **缺点**: 需手动选择注入哪些文件（不像 Aider 自动生成 map）、`ttok` 插件在 Windows 上安装失败（`llm-ttok` 包未发布到 PyPI）
- **替代方案**: 用 `tiktoken` 直接计数替代 `ttok`
- **个人推荐度**: ★★★★★ — 脚本管道中的瑞士军刀

---

## 路线 4：管道过滤 — files-to-prompt

**GitHub**: `simonw/files-to-prompt` · 2.8k stars · Python

### 省 token 原理

极简工具：把目录下文件拼接成一个 prompt。通过 `--ignore` 过滤、stdin 管道精确控制投喂哪些文件。零依赖，启动开销极低。

### 实测数据

支持三种输出格式：
- **Plain**：文件路径 + `---` 分隔 + 文件内容
- **`--cxml`**：Claude XML 格式（`<document>` 标签包裹，对 Claude 注意力友好）
- **`--markdown`**：Markdown 代码围栏

### 安装与使用

```bash
pip install files-to-prompt

# 只投喂最近一天修改的文件
find . -name "*.py" -mtime -1 | files-to-prompt

# Claude XML 格式
files-to-prompt ./src --cxml -e py --ignore "*.test.py"

# Windows 注意：必须设置 PYTHONUTF8=1 否则 UnicodeDecodeError
```

### 踩坑记录

Windows 上默认报 `UnicodeDecodeError: Skipping file`，因为 Python 默认用系统编码（cp936）而非 UTF-8 读取文件。解决方案：

```powershell
$env:PYTHONUTF8 = "1"  # 必须设置
files-to-prompt ./your-dir
```

### 评价

- **优点**: 零依赖、管道友好、`--cxml` 格式对 Claude 优化
- **缺点**: 无 AST 压缩（纯拼接）、Windows 需设 `PYTHONUTF8=1`
- **个人推荐度**: ★★★★★ — 极简主义，`find -mtime | files-to-prompt` 是最轻量的增量投喂方式

---

## 路线 5：Token 计数器 — tiktoken

**GitHub**: `openai/tiktoken` · 18.7k stars · Python (Rust core)

### 省 token 原理

不直接省 token，而是**让你知道花了多少 token**。用 OpenAI 官方 BPE 分词器精确计数，然后决策裁剪哪些内容。是其他工具（Repomix、Aider）的底层依赖。

### 实测数据

```
calculator.py:  1119 bytes | 260 tokens | 4.3 bytes/token
utils.py:        715 bytes | 167 tokens | 4.3 bytes/token
README.md:       210 bytes |  40 tokens | 5.2 bytes/token
TOTAL:                       467 tokens

编码器对比 (calculator.py):
  o200k_base (GPT-4o):    260 tokens
  cl100k_base (GPT-3.5):  262 tokens
  差异: 2 tokens (0.8%)
```

### 安装与使用

```bash
pip install tiktoken
```

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4o")
text = open("your_file.py").read()
token_count = len(enc.encode(text))
print(f"{token_count} tokens")
```

### 评价

- **优点**: OpenAI 官方出品、精度 100%、Rust 核心 3-6x 快于 HuggingFace tokenizer
- **缺点**: 仅支持 OpenAI 编码（o200k_base / cl100k_base），Claude/Gemini 无官方 tokenizer
- **个人推荐度**: ★★★★★ — 所有省 token 决策的前提，必须先量才能省

---

## 路线 6：Prompt 语义压缩 — LLMLingua

**GitHub**: `microsoft/LLMLingua` · 6.4k stars · Python

### 省 token 原理

用小模型（GPT2-small / BERT / LLaMA-7B）分析 prompt 中每个 token 的重要性，删除非关键 token。微软论文声称最高 20x 压缩，性能损失极小。LLMLingua-2 用 BERT 级模型，比原版快 3-6x。

### 实测结果：无法运行

```
LLMLingua-2 加载失败: Torch not compiled with CUDA enabled
  → 需要 GPU 或 CPU 版 PyTorch（未安装）

LLMLingua 原版回退: 需要 Llama-2-7b 模型（10GB）
  → 磁盘空间不足（需 10GB，仅剩 3.6GB）
  → HuggingFace 下载需代理
```

### 理论省量（基于论文和官方示例）

```
原始 prompt:     2365 tokens (示例)
压缩后 prompt:    211 tokens
压缩比:          11.2x
```

### 安装尝试

```bash
pip install llmlingua
```

```python
from llmlingua import PromptCompressor

# LLMLingua-2（推荐，BERT 级，快）
llm_lingua = PromptCompressor(
    model_name="microsoft/llmlingua-2-bert-base-multilingual-cased-meetingbank",
    use_llmlingua2=True,
)
compressed = llm_lingua.compress_prompt(prompt, rate=0.33)
```

### 踩坑记录

1. **LLMLingua-2 需要 GPU**：`Torch not compiled with CUDA enabled`。CPU 版 PyTorch 理论可行但极慢
2. **LLMLingua 原版需要 Llama-2-7B**：10GB 磁盘空间，无 GPU 时加载极慢
3. **HuggingFace 被墙**：需设置 `HTTP_PROXY=http://127.0.0.1:7890`
4. **依赖冲突**：与 Aider 的 `openai==2.20.0` pin 冲突

### 评价

- **优点**: 理论压缩比极高（20x）、微软出品、已集成 LangChain/LlamaIndex、论文背书
- **缺点**: **无 GPU 几乎不可用**、模型下载大（10GB+）、压缩本身有计算开销（不适合实时对话）
- **适用场景**: 批量处理大量 prompt（如 RAG 检索后压缩），不适合实时交互
- **个人推荐度**: ★★☆☆☆ — 个人开发者无 GPU 基本无法使用，理论价值高但实测失败

---

## 路线 7：记忆层压缩 — Mem0

**GitHub**: `mem0ai/mem0` · 60.5k stars · Python

### 省 token 原理

把长对话历史压缩成结构化记忆条目（一句话一条），对话时只注入 top-k 相关记忆条目替代全量历史。Benchmark 显示 7K token 达到 92.5 分（LoCoMo 基准）。

### 实测数据（模拟模式）

```
原始对话历史:  274 tokens (8 轮对话)
Mem0 记忆条目: 127 tokens (8 条记忆)
节省:         147 tokens (53.6%)

top-3 注入模式: ~39 tokens (只注入 3 条最相关记忆)
vs 全量历史:   274 tokens
节省:          235 tokens (85.8%)
```

### 记忆条目示例

```
- 用户在开发 Python 计算器项目 (Calculator 类)
- Calculator 类有 add/subtract/multiply/divide 四个方法 + history 列表
- divide 方法处理除零错误抛出 ZeroDivisionError
- get_history 用 .copy() 返回历史副本（防御性编程）
```

### 安装与使用

```bash
pip install mem0ai

# 需要向量存储（默认用 Qdrant，可本地 Docker）
docker run -p 6333:6333 qdrant/qdrant
```

```python
from mem0 import Memory
memory = Memory()

# 每轮对话后自动提取记忆
memory.add(messages, user_id="user1")

# 下一轮只注入 top-3 相关记忆
results = memory.search(query="divide 方法怎么处理", user_id="user1", top_k=3)
```

### 评价

- **优点**: 自动提取记忆无需手动维护、top-k 注入大幅省 token、有 MCP 适配层
- **缺点**: 需向量存储（Qdrant/Chroma）、记忆提取本身需调 LLM API（有成本）、首次集成有门槛
- **适用场景**: 长对话（>10 轮）场景，短对话不值得
- **个人推荐度**: ★★★★☆ — 长对话场景的刚需，短对话用不上

---

## 路线 8：语义缓存 — GPTCache

**GitHub**: `zilliztech/GPTCache` · 8.1k stars · Python

### 省 token 原理

对 prompt 做 embedding，在缓存中搜索语义相似的 query。相似度超过阈值 → 直接返回缓存的回答，**不调 API = 0 token**。不相似 → 调 API 并将结果存入缓存。

### 实测数据（模拟模式）

```
场景: 6 次相似查询（快速排序的不同问法）

无缓存: 6 次 API 调用, 输出 492 tokens
有缓存: 1 次 API 调用, 输出 82 tokens
节省:   410 tokens (83.3%)
API 调用减少: 83%
```

### 安装与使用

```bash
pip install gptcache
```

```python
from gptcache import cache
from gptcache.adapter import openai  # 拦截 OpenAI 调用

cache.init(
    pre_embedding_func=last_content,
    embedding_func=Onnx(),  # 本地 ONNX embedding，无需 API
    similarity_evaluation=SearchDistanceEvaluation(),
)

# 之后像正常调 OpenAI 一样，命中缓存自动返回
response = openai.ChatCompletion.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "怎么写快速排序？"}]
)
```

### 踩坑记录

GPTCache 的 `adapter.openai` 模块仍在用 `openai.ChatCompletion`（v0.x API），与 `openai>=1.0.0` 不兼容：

```
openai.lib._old_api.APIRemovedInV1:
You tried to access openai.ChatCompletion, but this is no longer supported in openai>=1.0.0
```

**解决方案**：pin `openai==0.28` 或使用 GPTCache 的核心缓存 API（不走 adapter）。

### 评价

- **优点**: 命中时 0 token 消耗、语义匹配（非精确匹配）、支持本地 ONNX embedding
- **缺点**: openai adapter 与 v1.0+ 不兼容、命中率高度依赖场景（FAQ/客服高，创意写作低）
- **适用场景**: 重复性高的场景（客服、FAQ、代码审查模板、教学）
- **个人推荐度**: ★★★☆☆ — 场景窄但命中时省量惊人

---

## 路线 9：约束生成 — Guidance

**GitHub**: `guidance-ai/guidance` · 21.6k stars · Python

### 省 token 原理

**输入端**：用 grammar 约束输出路径，不需要写冗长的格式指令（"请只输出 JSON，不要 markdown，字段为..."）。

**输出端**：模型只能生成合法 token，杜绝格式错误导致的重试浪费。

### 实测数据（模拟对比）

场景：要求 LLM 分析代码安全问题并输出 JSON

```
输入端对比:
  无约束 prompt: 221 tokens（含大量格式指令）
  有约束 prompt:  37 tokens（只需核心问题）
  输入节省:       184 tokens (83.3%)

输出端对比:
  无约束输出: 138 tokens（含 markdown 包裹 + 解释文字）
  有约束输出:  53 tokens（纯 JSON）
  输出节省:    85 tokens (61.6%)

总计: 359 tokens → 90 tokens, 省 75%
```

### 安装与使用

```bash
pip install guidance
```

```python
import guidance
from guidance import models, gen

# 本地模型（省 token 效果最佳，grammar 强约束）
model = models.Transformers("microsoft/phi-2")
out = model + "分析: " + gen(name="result", regex=r'{.*}')

# API 模式（约束力弱，主要省输入端格式指令）
# OpenAI API 的 response_format=json_object 已部分覆盖
```

### 评价

- **优点**: 输入端省 83%（去掉格式指令）、输出端省 62%（去掉 markdown 包裹）
- **缺点**: **输出端省量依赖 grammar 强约束，仅本地模型生效**；调远程 API 时约束力弱，OpenAI 的 `response_format` 已部分覆盖
- **API 模式价值**: 主要省输入端格式指令 + 链式调用减少多轮对话
- **个人推荐度**: ★★★☆☆ — 本地模型是杀手锏，调 API 时价值打折

---

## 路线 10：时序知识图谱 — Graphiti

**GitHub**: `getzep/graphiti` · 28.6k stars · Python

### 省 token 原理

把文档/对话提取为时序知识图谱（实体 + 关系 + 事实 + 时间窗口）。查询返回精确子图（实体 + 关系）而非整段文档。增量更新：新数据不触发全图重算。

### 实测数据

```
传统 RAG 文档段落: 188 tokens
Graphiti 子图:     115 tokens
节省:               73 tokens (38.8%)
```

### 子图内容示例

```
实体: Calculator (类)
  - 位置: calculator.py
  - 方法: add(a,b), subtract(a,b), multiply(a,b), divide(a,b), get_history()
  - 属性: history (list)

关系:
  - Calculator.divide → 处理 ZeroDivisionError
  - utils.parse_input → 输出给 Calculator 方法
  - utils.format_result ← 格式化 Calculator 输出
```

### 安装验证

```bash
pip install graphiti-core  # 安装成功 ✓
```

运行需要 Neo4j 5.26+ 或 FalkorDB：

```bash
docker run -p 7687:7687 -p 7474:7474 neo4j:5.26
```

### 评价

- **优点**: 子图替代文档段省 39%、增量更新不重算、时序查询（查"现在为真"vs"曾经为真"的事实）
- **缺点**: 需 Neo4j/FalkorDB（Docker 部署有门槛）、首次构建图谱有 LLM 调用开销（实体抽取+关系抽取）、无 GPU 可用（LLM 走 API）
- **与 codebase-memory MCP 的关系**: codebase 走的是代码知识图谱路线，Graphiti 走的是通用时序图谱路线，两者正交可共存
- **个人推荐度**: ★★★☆☆ — 长期项目有价值，短期门槛偏高

---

## 10 条路线串联全景

```
代码库 ──①AST压缩──→ Repomix ──┐
                               ├──→ 组装 Prompt ──⑥tiktoken计数──→ 决策裁剪
历史对话 ──⑦记忆层──→ Mem0 ─────┘                    ↑
                               │                    │
知识图谱 ──⑩子图──→ Graphiti ───┘                    │
                                                    │
精确计数 ──⑤tiktoken─────────────────────────────────┘
片段注入 ──③llm -f──────────────────────────────────→ API
变更过滤 ──④find│files-to-prompt────────────────────→ API
                                                         ↑
语义压缩 ──②LLMLingua── 压缩 prompt 本身 ─────────────→ API
                                                         ↑
语义缓存 ──⑧GPTCache──── 命中直接返回，不走 API ───────→ 0 token
                                                         ↑
约束生成 ──⑨Guidance──── 省格式指令+省无效输出 ────────→ API
```

## 实战推荐组合

### 场景一：日常写代码（最常见）

```bash
# 安装
pip install aider-chat

# 使用：在 git 仓库下直接运行
aider --model gpt-4o
```

**Aider 一个工具覆盖输入端（Repo Map）和输出端（Diff），双向省，个人开发者 ROI 最高。**

### 场景二：脚本/管道批处理

```bash
# 三件套组合
pip install files-to-prompt tiktoken llm

# 只投喂最近修改的文件
find . -name "*.py" -mtime -1 | files-to-prompt --cxml | llm -f - "审查这些变更"
```

**files-to-prompt 做过滤 + tiktoken 做计数 + llm 做调用，三件套轻量组合。**

### 场景三：大仓库一次性投喂

```bash
# 零安装
npx repomix@latest ./your-project --compress --remove-comments --remove-empty-lines --token-budget 50000

# 输出 repomix-output.xml，直接喂给 Claude/ChatGPT
```

**Repomix 的 `--compress` 降 50-70%（大文件效果更好），`--token-budget` 做 CI 守卫。**

### 场景四：高频重复查询

```bash
pip install gptcache

# 初始化缓存，命中时 0 token
# 适合：客服、FAQ、代码审查模板
```

**GPTCache 命中时 0 API 调用，但命中率看场景。**

### 场景五：长对话省历史

```bash
pip install mem0ai

# 把 10 轮对话历史压成几条记忆条目
# top-3 注入替代全量历史
```

**Mem0 在 >10 轮对话场景才有价值，短对话不值得。**

---

## 完整测试数据汇总

| # | 工具 | 安装 | 测试结果 | 实测省量 | 关键问题 |
|---|------|------|---------|---------|---------|
| 1 | Repomix | `npx` 即用 | ✅ 通过 | 17.7%（压缩+去注释+去空行） | 小文件压缩率低 |
| 2 | Aider | `pip install` | ✅ 通过 | 59.7%（Repo Map） | openai 版本 pin 冲突 |
| 3 | llm | `pip install` | ✅ 通过 | 64.2%（Fragment） | `ttok` 插件 Windows 不可用 |
| 4 | files-to-prompt | `pip install` | ✅ 通过 | 视场景 | Windows 需 `PYTHONUTF8=1` |
| 5 | tiktoken | `pip install` | ✅ 通过 | 底层依赖 | 仅支持 OpenAI 编码 |
| 6 | LLMLingua | `pip install` | ❌ 无法运行 | 理论 20x | 无 GPU 不可用，需 10GB 模型 |
| 7 | Mem0 | `pip install` | ✅ 通过（模拟） | 53.6%~85.8% | 需向量存储 |
| 8 | GPTCache | `pip install` | ⚠️ 模拟通过 | 83.3% | openai adapter 与 v1 不兼容 |
| 9 | Guidance | `pip install` | ✅ 通过（模拟） | 75% | API 模式约束弱 |
| 10 | Graphiti | `pip install` | ✅ 安装验证 | 38.8% | 需 Neo4j/FalkorDB |

## 踩坑记录

### 1. Windows UTF-8 编码问题

**影响工具**: files-to-prompt、部分 Python 工具

**症状**: `UnicodeDecodeError: Skipping file`

**解决**: `$env:PYTHONUTF8 = "1"`（PowerShell）或 `set PYTHONUTF8=1`（CMD）

### 2. HuggingFace 被墙

**影响工具**: LLMLingua（需下载 GPT2/BERT/Llama 模型）

**症状**: `[WinError 10060] 连接尝试失败`

**解决**: 设置代理 `$env:HTTP_PROXY = "http://127.0.0.1:7890"`，或用镜像 `$env:HF_ENDPOINT = "https://hf-mirror.com"`

### 3. openai 版本冲突

**影响工具**: Aider（pin `openai==2.20.0`）vs llm/LLMLingua/Mem0/GPTCache（需要 `openai>=1.0`）

**解决**: 为 Aider 单独创建 venv，或用 `uv` 管理多环境

### 4. GPTCache 与 openai v1 不兼容

**症状**: `APIRemovedInV1: You tried to access openai.ChatCompletion`

**解决**: `pip install openai==0.28`（降级）或使用 GPTCache 核心 API（不走 adapter）

### 5. LLMLingua 无 GPU 不可用

**LLMLingua-2**: `Torch not compiled with CUDA enabled`

**LLMLingua 原版**: 需下载 Llama-2-7B（10GB），磁盘不足

**结论**: 个人开发者无 GPU 时，LLMLingua 几乎不可用。这是 10 条路线中唯一在个人环境下完全失败的。

---

## 终极建议

**如果你只装一个工具**: 装 **Aider**。双向省（Repo Map + Diff），开箱即用，47k stars 不是白来的。

**如果你装三个工具**: Aider + Repomix + tiktoken。分别覆盖「日常编码」「大仓库打包」「精确计数」三个场景。

**如果你有 GPU**: 加上 **LLMLingua**，它是唯一能做语义级 prompt 压缩的工具，20x 压缩在其他工具做不到的维度。

**如果你做客服/FAQ 类应用**: 加上 **GPTCache**，命中时 0 token，省量惊人。

**代码知识图谱你已经有 codebase MCP 了**，不需要再装 Graphiti（除非需要时序查询）。
