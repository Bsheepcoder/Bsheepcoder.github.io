## 代码库理解的工程悖论

让 AI 读懂代码库，看似是一个"把代码塞给 LLM"的问题，实则是一个**信息密度与计算预算的约束优化问题**。

约束条件是确定的：context window 有限，注意力会随序列长度衰减，token 有成本。目标也是确定的：让 LLM 在回答代码问题时，用最少的 token 获取最精确的结构信息。困难在于，代码库的信息总量远超 context 容量，且代码的"有用信息"高度依赖查询上下文--回答"谁调用了 X"需要调用链，回答"X 和 Y 的关系"需要依赖路径，回答"这段代码做了什么"需要语义。

传统工程给出的答案分两类。一类是**全量打包**（Repomix、Aider repo map），把代码拼成文本塞进 prompt，简单但 token 消耗是 O(仓库大小)，大型仓库不可持续。另一类是**向量检索**（RAG 系），把代码切片做 embedding，查询时召回相似片段，语义模糊查询强但精确结构查询弱，且依赖外部 embedding 服务。

这两类方案共享同一个缺陷：**它们都不理解代码的结构**。打包派把代码当文本，向量派把代码当文档。而代码的本质是图--函数调用函数、类继承类、模块导入模块、路由处理请求。这些关系是确定性的、可精确解析的，不需要"语义相似"来近似。

codebase-memory-mcp 的核心思想，是把代码库当作一个**可查询的属性图数据库**来对待。它用 tree-sitter 解析 AST 提取确定性结构关系，用自研的 Cypher 引擎让 LLM 按需查询子图，用 SQLite 持久化索引实现 O(1) 的查询延迟。查询返回的是几 KB 的精炼结果而非几 MB 的源文件--5 次结构查询消耗约 3,400 tokens，对比文件遍历的 412,000 tokens，约 120 倍缩减。

这个"120 倍"不是魔法，是一系列工程取舍的结果。本文从源码层面拆解这些取舍。

## 把代码库变成图的七层架构

理解 codebase-memory-mcp 的关键，是看它的源码如何分层。`src/` 下 15 个目录，每个有清晰的单一职责，共同构成从源文件到查询结果的管道。

```
源文件 → discover（发现）→ pipeline（多趟解析）→ graph_buffer（内存图）→ store（SQLite 持久化）→ cypher/mcp（查询）
                                                              ↑
                                        semantic + simhash（语义增强，后处理）
```

### 第一层：文件发现与忽略

`discover/` 负责扫描文件，三层忽略策略：硬编码模式（`.git`、`node_modules`）→ `.gitignore` 层级合并 → `.cbmignore`（项目特定，gitignore 语法）。符号链接一律跳过。这看似简单，却是性能的第一道关卡--跳过不该解析的文件，比高效解析它更重要。

### 第二层：多趟解析管道

`pipeline/` 是整个项目最复杂的目录，37 个文件。核心是一个**顺序+并行混合的多趟管道**，`pipeline.c:767` 定义了顺序趟：

```c
static const struct { seq_pass_fn fn; const char *name; bool ignore_err; } seq_passes[] = {
    {cbm_pipeline_pass_definitions, "definitions", false},
    {cbm_pipeline_pass_k8s,         "k8s",         true},
    {seq_pass_lsp_cross_dispatch,   "lsp_cross",   true},
    {cbm_pipeline_pass_calls,        "calls",       false},
    {cbm_pipeline_pass_usages,       "usages",      false},
    {cbm_pipeline_pass_semantic,     "semantic",    false},
};
```

趟的顺序有设计：先提取定义（函数/类/方法），构建跨文件符号注册表；再用注册表解析调用关系（`calls` 趟依赖 `definitions` 趟的结果）；最后跑语义增强。`lsp_cross` 是 Hybrid LSP 的跨文件类型解析趟，能容错（`ignore_err: true`），因为它是对 tree-sitter 结果的增强而非基础。

这是一个关键取舍：**不追求单趟完美，而是多趟渐进精化**。第一趟给出 80% 可用的图，后续趟逐步精化。对 LLM 而言，80% 的调用链已经足够回答大多数问题。

### 第三层：内存图缓冲

`graph_buffer/graph_buffer.h` 定义了索引期的内存数据结构。所有节点和边先在 RAM 中构建，最后一次性 dump 到 SQLite。这个设计的核心注释道出了原因：

> Holds all nodes and edges in RAM during indexing, then dumps to SQLite. Provides O(1) node lookup by qualified name and edge dedup by key.

为什么不用 SQLite 直接写？因为索引期有海量的 upsert（同一个函数被多个趟引用）和 edge 去重，SQLite 的逐行事务开销会成为瓶颈。RAM-first 让去重变成哈希表查找，O(1)。只在最后 dump 时才做一次批量写入，此时索引已完成，写的是只增不删的最终结果。

`graph_buffer.h:53` 还有一个细节：`cbm_gbuf_new_shared_ids` 接受一个 `_Atomic int64_t *id_source`，用于**并行提取时多 gbuf 共享唯一 ID 源**。每个 worker 拉取一个 atomic ID，零竞争。这是 lock-free work-stealing 模式的体现（`worker_pool.h` 注释）：

> Each worker pulls from a shared counter - zero contention, natural load balancing across heterogeneous cores (P/E on Apple Silicon).

### 第四层：SQLite 持久化与查询

`store/store.h` 是查询期的核心。SQLite 作为图数据库使用，节点表存属性（label/name/qualified_name/file_path/行号/properties_json），边表存关系（source_id/target_id/type/properties_json）。

值得注意的优化在 `store.h:244`：

> Tune pragmas for bulk write throughput (synchronous=OFF, large cache). WAL journal mode is preserved throughout for crash safety.

批量写入时关闭同步（`synchronous=OFF`）换取吞吐，但保留 WAL（Write-Ahead Logging）保证崩溃安全。这是性能与安全的平衡--牺牲 fsync 的持久性（崩溃时可能丢最后几个事务），但保证不损坏数据库。`store.h:262` 还处理了 mmap 的边界情况：

> Negative values clamp to 0 (which disables mmap and reverts to read()/pread() I/O - recoverable SQLITE_IOERR instead of SIGBUS when concurrent processes truncate the DB file under live mappings).

当并发进程截断 DB 文件时，mmap 会触发 SIGBUS（不可恢复），而 read() 返回 SQLITE_IOERR（可恢复）。这个细节体现了对生产环境的深刻理解。

### 第五层：Cypher 查询引擎

`cypher/cypher.h` 实现了一个**自研的 openCypher 读子集引擎**：手写 lexer → parser → AST → executor。支持 MATCH、WHERE、RETURN、ORDER BY、聚合、CASE、UNION、变量长度路径 `[*1..3]`、EXISTS 子查询等。

为什么不用 SQL 直接查？因为图查询的模式匹配（`MATCH (n:Function)-[:CALLS*1..3]->(m)`）用 SQL 表达是递归 CTE，可读性和性能都差。Cypher 是图查询的领域特定语言，让 LLM 能用一句话表达"找到所有调用链深度 1-3 的 callers"。

关键取舍：**只实现读子集**。写操作（CREATE/MERGE/DELETE）明确拒绝并返回 `unsupported` 错误，而非静默失败。这是安全边界--查询引擎不可能意外修改图数据。

### 第六层：语义增强（灵魂所在）

`semantic/` 和 `simhash/` 是这个项目区别于"纯 AST 图"的核心。这部分后面详述，是本文的重点。

### 第七层：MCP 工具暴露

`mcp/` 暴露 14 个 MCP tool，分索引类（`index_repository`、`index_status`）和查询类（`search_graph`、`trace_path`、`query_graph`、`get_architecture`、`detect_changes` 等）。LLM 通过这些 tool 按需查询，而非一次性获取整个图。

## 确定性结构：tree-sitter 与 Hybrid LSP 的两层架构

### tree-sitter：语法层的确定性

tree-sitter 是增量解析器，生成 CST（具体语法树）。codebase-memory-mcp 把 158 种语言的 grammar 编译进二进制，解析是纯确定性的--同一段代码每次解析结果一致，零幻觉。

但 tree-sitter 有根本局限：**它只懂语法，不懂语义**。README 的 Hybrid LSP 章节一针见血地指出：

> Tree-sitter alone gives a syntactic AST. That handles naming, structure, and call sites well, but it can't tell you that `user.profile.display_name()` resolves to `Profile.display_name` declared three modules away - tree-sitter doesn't track imports, generics, inheritance, or stdlib types.

一个 `user.profile.display_name()` 调用，tree-sitter 只知道"这是一个方法调用，对象是 `user.profile`，方法名是 `display_name`"。但它不知道 `user` 的类型是 `User`，`profile` 属性的类型是 `Profile`，`display_name` 定义在 `Profile` 类中三个模块之外。这些是**类型解析**的职责，属于语义层。

### Hybrid LSP：类型层的增强

传统解决方案是跑一个语言服务器（tsserver、pyright、gopls、rust-analyzer），但每个语言服务器都是独立进程，需要项目配置，启动慢，资源重。codebase-memory-mcp 的取舍是：

> A lightweight C implementation of language type-resolution algorithms, structurally inspired by and compatible with major language servers, embedded directly into the static binary. No language server process, no per-project setup, no API key.

它不运行语言服务器，而是用 C **重新实现了类型解析算法的核心逻辑**，编译进二进制。支持 9 种语言（Python、TS/JS/JSX/TSX、PHP、C#、Go、C/C++、Java、Kotlin、Rust），每种处理该语言最常见的类型推断模式。

两层架构的精妙在于**优雅降级**：

> Languages without a Hybrid LSP pass yet fall back to textual resolution, so you always get some answer.

没有 Hybrid LSP 的语言（158 种中的 149 种），调用解析回退到文本匹配（符号名匹配）。质量下降，但不中断--你总是能得到一个图，只是调用链可能不精确。这是工程上的务实：先覆盖广度（158 语言能解析），再逐步深化（9 语言精确解析）。

源码中 `pass_lsp_cross.c` 和 `registry.c` 实现了这个逻辑。`registry.c` 维护一个跨文件定义注册表，`calls` 趟通过注册表把 `display_name()` 这样的调用解析到 `Profile.display_name` 的定义。当多个候选时，用**导入可达性**（`is_import_reachable`）和**导入距离**（`best_by_import_distance`）来排序候选--只有被 import 的定义才可能是真实目标。

## 语义相似度：11 信号融合的零依赖方案

这是整个项目最精巧的部分。`semantic/semantic.h` 的头注释列出了 11 个信号：

```
1. TF-IDF on metadata tokens (vocabulary overlap)
2. Random Indexing with co-occurrence (within-codebase synonym bridging)
3. MinHash structural (existing, decoded from "fp" property)
4. API Signature vectors (same callees -> related)
5. Type Signature vectors (same param/return types -> related)
6. Module Proximity (same directory -> boost)
7. Decorator Pattern vectors (same annotations -> related)
8. AST Structural Profile (control flow shape, expression types)
9. Approximate Data Flow (params->return, params->condition)
10. Graph Diffusion (transitive closure via neighbor blending)
11. Halstead-Lite (operator/operand complexity profile)
```

关键声明在注释开头：

> Combines 11 signals into a unified similarity score without external models or dependencies.

**零外部依赖、零 LLM 调用**。这 11 个信号全部从 AST 和图缓冲区的元数据计算得出。对比向量 RAG 方案需要 embedding API，这是根本性的架构差异--语义相似度不依赖任何模型，是纯算法的。

### 信号融合的加权公式

`semantic.c:40` 定义了默认权重，总和约 1.0：

```c
#define CBM_SEM_W_TFIDF          0.20F
#define CBM_SEM_W_RI             0.25F
#define CBM_SEM_W_MINHASH        0.10F
#define CBM_SEM_W_API            0.15F
#define CBM_SEM_W_TYPE           0.10F
#define CBM_SEM_W_DECORATOR      0.05F
#define CBM_SEM_W_STRUCT_PROFILE 0.10F
#define CBM_SEM_W_DATAFLOW       0.05F
```

权重分配反映了各信号的**信息密度**。Random Indexing（0.25）权重最高，因为它捕获跨文件的词汇共现--两个函数如果在相似上下文中出现相同 token，大概率是同义词或相关概念，这是"代码库内 synonym bridging"的核心。TF-IDF（0.20）次之，是词汇重叠的经典度量。MinHash（0.10）处理结构近克隆。

`cbm_sem_combined_score`（`semantic.c:1601`）是融合函数，有一个精妙的短路逻辑：

```c
/* Short-circuit: if MinHash Jaccard is already above the SIMILAR_TO threshold,
 * the pass_similarity pipeline already emitted a SIMILAR_TO edge for this pair.
 * Returning 0 here avoids flooding top-k with cross-service copy-paste boilerplate
 * (logging_middleware, shared push/pull handlers) that SIMILAR_TO already covers,
 * freeing the edge budget for true semantic leaps. */
if (a->has_minhash && b->has_minhash) {
    double early_j = cbm_minhash_jaccard(...);
    if (early_j >= CBM_MINHASH_JACCARD_THRESHOLD) {
        return 0.0F;
    }
}
```

如果两个函数已经是近克隆（MinHash Jaccard ≥ 0.95），就不发 `SEMANTICALLY_RELATED` 边。原因是跨服务的复制粘贴样板代码（日志中间件、共享 push/pull handler）会"淹没"语义边，挤占边预算。这个短路让语义边的配额留给真正的"语义跃迁"--词汇桥接的、非结构相似的关系。

**模块邻近度**是乘法而非加法：

```c
score *= cbm_sem_proximity(a->file_path, b->file_path);
if (score > CBM_SEM_UNIT_POS) score = CBM_SEM_UNIT_POS;
```

同目录的函数获得最高 1.10 倍的 boost，最终 clamp 到 [0, 1]。乘法而非加法，是因为邻近度是"先验概率调整"而非"独立证据"--同目录的函数本来就更可能相关，这是一个先验，不是新信息。

### 阈值校准的诚实

`semantic.h:54` 的阈值注释体现了工程诚实：

```c
/* Default score threshold for SEMANTICALLY_RELATED edge emission.
 * 0.75 balances recall with precision: validated ~95% precision on
 * Linux kernel (0.80 = 100% but only 90 edges, 0.70 = 2047 edges
 * but ~80% precision). */
#define CBM_SEM_EDGE_THRESHOLD 0.75
```

阈值不是拍脑袋定的，是在 Linux kernel 上校准的：0.80 给 100% 精度但只有 90 条边（召回太低），0.70 给 2047 条边但精度降到 80%。0.75 是 precision-recall 的平衡点。这种"附上原始数据让读者自己判断"的注释风格，是成熟工程项目的标志。

## RaBitQ 量化：把 3KB 向量压到 0.5KB

### 问题：内存爆炸

11 个信号中，4 个是 768 维稠密向量（RI、API、Type、Decorator）。`semantic.h:36`：

```c
/* 768 = nomic-embed-code embedding dimension. Matches PRETRAINED_DIM. */
enum { CBM_SEM_DIM = 768 };
```

768 维 float 是 3 KB。Linux kernel 有约 100 万个函数，4 个向量 × 3 KB × 100 万 = **12 GB**。这在内存中是不可持续的。注释直接点出：

> 4 x 3 KB resident floats per function were ~9.4 GB on the linux kernel; the codes are ~0.5 KB each.

### 解法：RaBitQ 风格的 4-bit 量化

`rotsq.h` 实现了 Extended RaBitQ（Gao et al., SIGMOD 2024/2025）的核心，但**从论文重写**而非引用参考库（注释明确声明 "written from the papers"）。它命名为 `rotsq` 而非 `rabitq`，刻意不夸大保真度。

原理分三步：

**第一步：随机旋转**。用随机 ±1 对角矩阵（XXH3 种子，可复现）乘向量，再做 Fast Walsh-Hadamard Transform。旋转的目的是**把向量的质量均匀分布到所有坐标**--旋转后，单位向量的每个坐标接近高斯分布，这使得朴素的标量量化接近最优。注释点出这是 RaBitQ 的核心观察：

> The rotation spreads a vector's mass evenly across coordinates (rotated coords of a unit vector are near-Gaussian), which makes plain scalar quantization behave near-optimally - that observation IS RaBitQ.

**第二步：每向量标量量化**。旋转后的每个坐标量化到 4 bit（15 个等级）。关键是用**每向量的 scale 和 offset**，而非全局固定范围。`cbm_rsq_encode` 找到该向量的 [lo, hi]，映射到 [0, 15]。

**第三步：精确的码展开内积估计**。这是最精妙的部分。`cbm_rsq_ip`（`rotsq.c:100`）不解码回 float 再点积，而是直接从 4-bit 码用展开公式估计内积：

```
x_i ≈ ox + sx·cx_i  ⇒  ⟨x,y⟩ ≈ D·ox·oy + ox·sy·Σcy + oy·sx·Σcx + sx·sy·Σ(cx_i·cy_i)
```

其中 `Σcx` 预存为 `code_sum`，所以**每对向量的点积只需一次整数点积（u8×u8）加四次乘法**。这是确定性的--纯码的函数，不引入随机误差。

结果：768 维 float（3 KB）压缩到 512 字节码 + 12 字节元数据 = **524 字节**，约 6 倍压缩。内存从 9.4 GB 降到约 1.5 GB。

注释诚实指出这个量化是估计而非精确：

> Named rotsq, not rabitq: this is the FAMILY core ..., not the papers' exact codebook construction - the name should not overclaim fidelity.

## MinHash + LSH：近克隆检测的工程化

`simhash/minhash.h` 实现了 AST 级的近克隆检测。原理值得拆解，因为它体现了"如何把理论算法工程化"的思路。

### 第一步：AST 叶子 token 归一化

`minhash.c:142` 的 `collect_ast_tokens` 只收集**叶子节点**（实际源 token），跳过内部语法节点。`normalise_node_type`（`minhash.c:98`）把叶子归一化：

- 标识符 → `"I"`
- 字符串 → `"S"`
- 数字 → `"N"`
- 类型注解 → `"T"`
- 其他 → 原始类型名

为什么归一化？因为 `count` 和 `cnt`、`str1` 和 `str2` 的结构是相同的，只是名字不同。归一化让"结构形状"浮现，忽略具体标识符。注释点出这是 BigCloneBench 的标准：

> 30 leaf tokens ≈ BigCloneBench standard of 50 raw source tokens.

### 第二步：trigram 加权 MinHash

收集叶子 token 后，构建 trigram（三元组）。`trigram_structural_weight` 给每个 trigram 计算权重：非归一化 token 越多，权重越高。权重 0 的 trigram（全是 I/S/N/T）是"纯数据操作噪声"，跳过；权重 3 的 trigram（三个都是控制流关键字）是"丰富的控制流信号"。

加权 MinHash 的实现是对权重为 w 的 trigram **哈希 w 次**，等效于按权重抽样。这不是标准 MinHash，而是**加权 MinHash** 的变体。

### 第三步：LSH 候选生成

MinHash 给出 Jaccard 相似度的估计，但两两比较是 O(n²)，百万函数不可行。`minhash.h:38` 定义了 LSH 参数：

```c
#define CBM_LSH_BANDS 32
#define CBM_LSH_ROWS  2
```

32 个 band × 2 行，阈值 ≈ (1/32)^(1/2) ≈ 0.177。只有至少一个 band 完全相同的 pair 才成为候选，把 O(n²) 降到近 O(n)。`cbm_lsh_query_into` 是线程安全的变体，写入调用者提供的缓冲区。

### 阈值的校准

`CBM_MINHASH_JACCARD_THRESHOLD = 0.95`，`CBM_MINHASH_MIN_NODES = 30`。注释解释：

> 64 gives ±0.12 standard error on Jaccard - sufficient for a 0.95 threshold.

K=64 的 MinHash 给 ±0.12 的标准误差，对 0.95 阈值足够（即使误差最大，实际 0.83 的对也能被召回）。这是统计学与工程的结合--参数不是任意的，是根据误差边界反推的。

## RAM-first 管道：为什么 28M 行代码 3 分钟能索引完

### 性能数据的来源

BENCHMARK.md 和 README 给出的硬数据：

| 操作 | 耗时 | 规模 |
|------|------|------|
| Linux kernel 全索引 | 3 min | 28M LOC, 75K 文件, 4.81M 节点, 7.72M 边 |
| Linux kernel 快速索引 | 1m 12s | 1.88M 节点 |
| Django 全索引 | ~6s | 49K 节点, 196K 边 |
| Cypher 查询 | <1ms | 关系遍历 |
| 调用路径追踪（深度 5） | <10ms | BFS |

这些数字的背后是四层优化叠加。

### 优化一：LZ4 HC 压缩读取

`pipeline.c:7` 的注释：

> Bulk load sources (read + LZ4 HC compress)

源文件读取时用 LZ4 HC（高压缩率模式）压缩后存内存，减少内存占用和后续趟的读取开销。LZ4 的解压速度接近 memcpy，压缩比虽不如 zstd，但解压零成本。

### 优化二：内存图缓冲 + 单次 dump

如前所述，所有趟在 RAM 中操作 graph_buffer，只有最后一次性 `cbm_gbuf_dump_to_sqlite`。这避免了 SQLite 逐行 upsert 的开销。dump 时分配最终 ID、重映射边引用，是一次批量操作。

### 优化三：并行提取 + lock-free work-stealing

`worker_pool.h` 用 pthreads（8MB 栈）+ 原子 work-stealing 索引。每个 worker 从共享原子计数器拉取下一个文件索引，零竞争。注释点出这在异构核（Apple Silicon 的 P/E 核）上天然负载均衡--快核拉得多，慢核拉得少，无需调度器干预。

`graph_buffer.h:53` 的 `cbm_gbuf_new_shared_ids` 让多个 worker gbuf 共享原子 ID 源，merge 时无 ID 冲突。

### 优化四：Aho-Corasick 融合模式匹配

README 提到 "fused Aho-Corasick pattern matching"。在 HTTP 路由检测、配置链接等场景，需要在一个文件中同时匹配多个模式（路由前缀、配置键）。Aho-Corasick 是多模式匹配的经典算法，一次扫描匹配所有模式，时间复杂度 O(文本长度 + 匹配数)。"fused" 指它与 AST 遍历融合--不单独扫一遍文件，而是在 AST 遍历时同步做模式匹配。

### 增量索引的诚实

`pipeline_incremental.c` 的增量索引基于 mtime+size 而非内容哈希：

```c
if (stat_mtime_ns(&st) != h->mtime_ns || st.st_size != h->size) {
    changed[i] = true;
}
```

这个选择有取舍。mtime+size 快（一次 stat），但**编辑后改回原样但 mtime 变了会误判为变更**。内容哈希（SHA-256）精确但要读全文件。对"快速增量"的目标，mtime+size 是合理默认。注释没有回避这个取舍。

## 自研 Cypher 引擎：为什么要重造轮子

### 图查询的领域需求

`cypher/cypher.h` 手写了一个 Cypher 子集引擎。为什么不用 SQL？因为图查询的模式匹配用 SQL 表达是递归 CTE：

```sql
WITH RECURSIVE call_chain AS (
    SELECT target_id, 1 AS depth FROM edges WHERE source_id = ? AND type = 'CALLS'
    UNION ALL
    SELECT e.target_id, c.depth + 1 FROM edges e JOIN call_chain c ON e.source_id = c.target_id
    WHERE e.type = 'CALLS' AND c.depth < 3
)
SELECT * FROM call_chain;
```

而 Cypher 是：

```cypher
MATCH (n:Function {name: "ProcessOrder"})-[:CALLS*1..3]->(m:Function) RETURN m.name
```

对 LLM 而言，Cypher 更紧凑、更可预测、更省 token。自研引擎让 LLM 用自然的方式表达图查询意图。

### 支持的范围与边界

`cypher.h` 支持：MATCH（含 OPTIONAL MATCH、多 MATCH）、WHERE（含 AND/OR/XOR/NOT、IN、CONTAINS、STARTS/ENDS WITH、IS NULL、regex `=~`、EXISTS 子查询）、RETURN（含聚合 count/sum/avg/min/max/collect、DISTINCT、CASE）、ORDER BY、SKIP/LIMIT、UNION/UNION ALL、UNWIND。

明确拒绝的：CREATE/MERGE/DELETE/SET 等写操作、CALL 子句、列表/字典字面量、路径函数、参数。拒绝方式是返回明确的 `unsupported …` 错误，而非静默返回空结果。这是**防御性设计**--LLM 不会在不知情的情况下得到误导性的空结果。

### 正则预过滤优化

`store.h:606` 的 `cbm_extract_like_hints` 从正则中提取字面子串（≥3 字符）做 SQL LIKE 预过滤。先用廉价的 LIKE 缩小候选集，再用昂贵的正则精确匹配。这是查询计划优化的经典手法--把昂贵的操作推迟到小集合上。

## 没有完美的工程

### Hybrid LSP 的覆盖鸿沟

9 种语言有 Hybrid LSP，158 种中剩余 149 种回退到文本解析。Python/TS/Go 的调用链精确，但 OCaml（72%）、Haskell（62%）的 benchmark 明显低。这不是 bug，是**优先级排序**的结果--先覆盖主流语言，小众语言靠 tree-sitter 的文本回退兜底。

但这个鸿沟对多语言仓库是实际风险。一个 C++ + Python + Rust 的混合仓库，C++ 和 Python 的调用链精确，但两者之间的跨语言调用（如 Python 通过 CFFI 调用 Rust）无法解析。tree-sitter 是按语言隔离的，没有跨语言符号解析。

### 语义边的主观性

`SEMANTICALLY_RELATED` 边的阈值 0.75 是在 Linux kernel 上校准的，但不同代码库的"语义相关"分布不同。一个高度模块化的 Java 企业项目和一个碎片化的 JS 前端项目，最优阈值可能不同。环境变量 `CBM_SEMANTIC_THRESHOLD` 允许覆盖，但需要用户自己调参--把责任推给了用户。

且语义边的**含义模糊**。"语义相关"可能是"做类似的事"、"调用类似的 API"、"有类似的结构"、"在同一目录"。11 个信号融合成一个分数，丢失了"为什么相关"的解释。Graphify 用 `EXTRACTED`/`INFERRED`/`AMBIGUOUS` 标注边的可信度，codebase-memory-mcp 没有等价机制--所有边看起来同样可信。

### 量化引入的误差

RaBitQ 量化是估计而非精确。4-bit 量化在 768 维上的估计误差虽然小，但累积在百万级的两两比较中，可能产生假阳性或假阴性。注释承认：

> this is the FAMILY core (randomized rotation + per-vector scalar quantization + exact code-expansion estimator), not the papers' exact codebook construction - the name should not overclaim fidelity.

工程上这个误差可接受（用于 top-k 召回而非精确排序），但用户不应假设语义分数是完全精确的。

### C 实现的维护门槛

用 C 写高性能代码是性能的保障，却是社区贡献的障碍。修改一个 grammar 的解析逻辑需要懂 C 和 tree-sitter API；添加一个新的语义信号需要理解 RaBitQ 量化和加权融合；修一个内存泄漏要会用 valgrind/ASAN。对比 Graphify 用 Python，社区贡献门槛低一个数量级。

这在项目早期是优势（小团队能控制质量），在项目成长期是风险--如果核心维护者放缓，社区难以接手。SLSA L3 和 70+ 杀软扫描是供应链安全的加分项，但不解决"代码可维护性"问题。

### 增量索引的精度

mtime+size 的增量判断会误判"编辑后改回"的文件。更严重的是，它无法检测**语义不变但语法变化**的情况（如格式化代码）。一次 `gofmt` 会让所有被格式化的文件标记为变更，触发完整的重新解析，即使调用关系完全没变。内容哈希能避免这个问题，但代价是读全文件。

## 代码库即数据库

回到根本，codebase-memory-mcp 的灵魂内核可以浓缩为一句话：**把代码库当作一个属性图数据库来对待，用工程手段把"建库"和"查询"都压到极致性能**。

它的所有设计取舍都围绕这个核心：

- **tree-sitter + Hybrid LSP** 是"建库的数据摄入"--确定性解析保证数据质量，两层架构平衡覆盖广度与解析深度。
- **RAM-first + LZ4 + Aho-Corasick + work-stealing** 是"建库的摄入性能"--把索引时间压到分钟级。
- **SQLite + Cypher 引擎** 是"查询的执行层"--SQL 持久化保证查询延迟，Cypher 让 LLM 用最少 token 表达意图。
- **RaBitQ 量化 + MinHash + LSH + 11 信号融合** 是"查询的语义增强"--在零外部依赖下提供近似的语义检索能力。

这个架构的本质隐喻是**ETL + OLAP**：索引是 ETL（Extract-Transform-Load），把代码提取成结构化数据加载进图；查询是 OLAP，对图做分析查询。LLM 是这个系统的"BI 前端"--它不直接查表，而是通过 MCP tool 表达意图，由查询引擎返回结构化结果。

理解了这个隐喻，就理解了为什么 codebase-memory-mcp 不内嵌 LLM。README 说得直白：

> Why no built-in LLM? Other code graph tools embed an LLM for natural language → graph query translation. This means extra API keys, extra cost, and another model to configure. With MCP, the agent you're already talking to is the query translator.

把"自然语言转查询"的职责交给 LLM，把"查询执行"的职责留在 C 引擎。这是关注点分离--每一层做它最擅长的事。

这种工程的克制，是它真正的灵魂。不是 28.6k stars，不是 158 语言，不是 120x token 缩减，而是每一个注释都在说"这里我们做了什么取舍、为什么、代价是什么"。这种诚实，比任何 benchmark 数字都更有价值。
