# 分类：计算机基础

共 3 篇文章

---

# Argon2id 密码哈希：memory-hard 的工程实践与前沿
Date: 2026-07-01 | Tags: 计算机基础, 密码学, Argon2, 安全 | URL: https://bsheepcoder.github.io/2026/07/01/cs-argon2id-explained/

## 一、元认知：为什么 CPU 慢还不够

[上一篇文章](/2026/07/01/cs-bcrypt-explained/)讲了 BCrypt 的核心贡献：让哈希变慢，且慢的程度可调（cost factor）。这在 1999 年是突破性的——但到了 2015 年，"慢"的维度不够用了。

### GPU 和 ASIC 改变了攻击经济学

BCrypt 的 cost factor 让 CPU 计算变慢，但攻击者不只用 CPU。一块 RTX 4090 有 16384 个 CUDA 核心，能并行跑 16384 个 bcrypt 实例。虽然 bcrypt 的 cache-hard 特性让 GPU 每个核心的效率低于 CPU（没有 L1 cache），但**并行数量弥补了单核效率**。

更极端的是 ASIC——定制芯片。攻击者花几百万美元流片一块专用芯片，把 bcrypt 的 Eksblowfish 算法烧进硬件，单芯片每秒算几百万次。bcrypt 的 cost factor 对 ASIC 无效，因为 cost 只增加计算轮数，不增加硬件需求。

### 核心矛盾

攻击者的成本 = 计算时间 × 硬件单价。BCrypt 提高了"计算时间"，但没有提高"硬件单价"——攻击者用便宜的 GPU/ASIC 就能并行。

> 如果能让哈希计算**既慢又费内存**，攻击者就不能只用 GPU 并行——每个并行实例都需要独立的内存，内存是昂贵的、不可压缩的硬件资源。

这就是 **memory-hard function（MHF）** 的出发点：把攻击者成本从"买更多 GPU"变成"买更多 GPU + 买更多内存"。

## 二、搭积木：Argon2 怎么工作的

### PHC 竞赛：24 个提案的淘汰赛

2013-2015 年，密码学界举办了一次 Password Hashing Competition（PHC），收到 24 个提案。评选标准：

- 抗 GPU/ASIC 并行（memory-hardness）
- 抗侧信道攻击（side-channel resistance）
- 可调参数（时间、内存、并行度）
- 简洁可分析（不依赖冷门数学假设）

2015 年 7 月，**Argon2** 获胜。设计者是卢森堡大学的 Alex Biryukov、Daniel Dinu、Dmitry Khovratovich。2021 年 9 月成为 RFC 9106，许可 CC0（公有领域）。

### 三个变体：一个工程权衡

Argon2 不是一个算法，是三个——区别在于**内存访问模式**：

| 变体 | 内存访问 | 抗侧信道 | 抗 GPU/ASIC | 适用场景 |
|------|---------|---------|------------|---------|
| **Argon2d** | 依赖密码（data-dependent） | 弱 | 强 | 加密货币、不怕侧信道 |
| **Argon2i** | 独立于密码（data-independent） | 强 | 中 | 密码哈希、抗侧信道 |
| **Argon2id** | 混合（前半 i，后半 d） | 中 | 强 | **RFC 9106 推荐默认** |

这个权衡的核心是 **data-dependent vs data-independent**：

- **Argon2d** 的内存访问地址由上一步的计算结果决定——攻击者无法预知访问模式，所以无法优化内存布局。但如果攻击者能测量缓存访问时间（侧信道），就能推断密码信息。
- **Argon2i** 的内存访问地址是固定的（不依赖密码）——攻击者无法从侧信道获取信息，但因为访问模式可预测，攻击者能用 time-memory tradeoff（TMTO）攻击降低内存需求。
- **Argon2id** 的折中：前半趟用 data-independent（建立抗侧信道的内存填充），后半趟用 data-dependent（利用随机访问增加 TMTO 攻击成本）。

RFC 9106 和 OWASP 都推荐 **Argon2id** 作为默认——它在两种攻击之间取得了最好的平衡。

### 三个参数：m、t、p

Argon2id 的三个可调参数：

| 参数 | 含义 | OWASP 推荐 | 影响 |
|------|------|-----------|------|
| **m** | 内存（KiB） | 19456（19 MiB） | 每个并行实例占用的内存 |
| **t** | 时间（迭代次数） | 2 | 遍历内存的趟数 |
| **p** | 并行度（线程数） | 1 | 并行计算线程数 |

三个参数的工程含义：

**m（内存）** 是最关键的参数。它直接决定攻击者复制攻击的内存成本。19 MiB 意味着攻击者用 GPU 并行 1000 个实例需要 19 GB 显存——一块 RTX 4090 只有 24 GB，只能并行约 1200 个实例。如果调到 64 MiB，同一块 GPU 只能并行约 375 个。

**t（时间）** 是次要参数。第一趟已经填满了内存，后续趟是"再搅拌"——增加时间但增加的内存成本为零（内存已分配）。所以 t 主要增加 CPU 时间，不增加攻击者的硬件成本。通常 t=1-3，不建议超过 4。

**p（并行度）** 允许多线程并行计算。对防御者来说，p=4 能用 4 个核心加速 4 倍。对攻击者来说，每个并行实例仍需独立内存——p 不降低 memory-hardness。但 p>1 意味着防御者也要分配 p×m 的内存，且验证时需要 p 个核心。

### 计算流程

Argon2id 的计算过程：

```
1. 初始化
   输入：密码 P、盐 S、参数 (m, t, p)
   分配：m KiB 内存块（分成 p 个并行 lane，每 lane m/p KiB）

2. 第一趟（Argon2i 模式，data-independent）
   按 H(i) 确定的固定地址，填充内存块
   → 建立抗侧信道的基础填充

3. 后续趟（Argon2d 模式，data-dependent）
   按上一步计算结果确定的动态地址，重新搅拌内存块
   → 增加 TMTO 攻击成本

4. 最终化
   合并所有 lane 的最后一个块 → 输出哈希
```

关键点：内存块在计算全程保持分配。攻击者不能用"边算边释放"的流式方式降低内存峰值——必须同时持有全部 m KiB。这是 memory-hard 的本质。

### 输出格式

一个 Argon2id 哈希长这样：

```
$argon2id$v=19$m=19456,t=2,p=1$nKj8Vp3T5r2YwE8mF1qK2O5xQ9zL8aB3cD4eF5gH6iJ7kL8mN9oP0$5xQ9zL8aB3cD4eF5gH6iJ7kL8mN9oP0qRsTuVwXyZ
```

拆解：

| 段 | 含义 |
|----|------|
| `$argon2id$` | 算法标识 |
| `v=19` | 版本号（1.3 = v19） |
| `m=19456,t=2,p=1` | 三个参数 |
| `nKj8Vp3...` | Base64 编码盐 |
| `5xQ9zL8...` | Base64 编码哈希 |

和 bcrypt 一样，盐和参数编码在同一字符串里——一个字符串包含验证密码所需的全部信息。

## 三、案例即原理：多语言实战集成

### Python：argon2-cffi

```python
from argon2 import PasswordHasher, exceptions

ph = PasswordHasher(
    time_cost=2,        # t
    memory_cost=19456,  # m (KiB)
    parallelism=1,      # p
    hash_len=32,        # 输出长度
    salt_len=16,        # 盐长度
)

# 哈希密码
hashed = ph.hash("my_password")
# $argon2id$v=19$m=19456,t=2,p=1$...

# 验证密码
try:
    ph.verify(hashed, "my_password")  # 正确 → True
    ph.verify(hashed, "wrong")        # 错误 → 抛 VerifyMismatchError
except exceptions.VerifyMismatchError:
    print("密码错误")

# 检查是否需要重新哈希（参数过时时升级）
if ph.check_needs_rehash(hashed):
    new_hashed = ph.hash("my_password")  # 用新参数重新哈希
```

`check_needs_rehash` 是密码哈希升级的关键——用户登录时验证旧哈希，如果参数过时（比如从 m=8192 升级到 m=19456），在验证成功后用新参数重新哈希，用户无感知地完成升级。

### Node.js：argon2

```javascript
const argon2 = require("argon2");

// 哈希密码
const hash = await argon2.hash("my_password", {
  type: argon2.argon2id,
  memoryCost: 19456,  // m (KiB)
  timeCost: 2,        // t
  parallelism: 1,     // p
  hashLength: 32,
  saltLength: 16,
});

// 验证密码
const valid = await argon2.verify(hash, "my_password");  // true
```

### Java：Bouncy Castle

```java
import org.bouncycastle.crypto.generators.Argon2BytesGenerator;
import org.bouncycastle.crypto.params.Argon2Parameters;

Argon2Parameters params = new Argon2Parameters.Builder(Argon2Parameters.ARGON2_id)
    .withVersion(Argon2Parameters.ARGON2_VERSION_13)
    .withIterations(2)           // t
    .withMemoryAsKB(19456)       // m
    .withParallelism(1)          // p
    .withSalt(saltBytes)
    .build();

Argon2BytesGenerator gen = new Argon2BytesGenerator();
gen.init(params);

byte[] output = new byte[32];
gen.generateBytes(output, passwordChars, passwordChars.length);
```

### Go：golang.org/x/crypto/argon2

```go
import "golang.org/x/crypto/argon2"

// key := argon2.IDKey(password, salt, t, m, p, keyLen)
hash := argon2.IDKey([]byte("my_password"), salt, 2, 19456, 1, 32)
```

Go 标准库的 `argon2.IDKey` 返回原始字节，不生成编码字符串。需要自己拼 `$argon2id$v=19$m=...,t=...,p=1$salt$hash` 格式（或用第三方库 `github.com/alexedwards/argon2id`）。

### 参数调优实战

OWASP 推荐的 `m=19456, t=2, p=1`（19 MiB）是 2024 年的基准。但实际选型应基于**目标验证时间**：

```python
# 调优脚本：找到目标耗时对应的参数
import time
from argon2 import PasswordHasher, Type

def benchmark(m, t, p):
    ph = PasswordHasher(time_cost=t, memory_cost=m, parallelism=p, type=Type.ID)
    start = time.perf_counter()
    ph.hash("benchmark_password_123!")
    elapsed = (time.perf_counter() - start) * 1000
    return elapsed

# 目标：250ms ± 50ms
for m in [12288, 19456, 32768, 65536]:
    for t in [1, 2, 3]:
        ms = benchmark(m, t, 1)
        flag = " ✓" if 200 <= ms <= 300 else ""
        print(f"m={m:6d} ({"{:.1f}".format(m/1024)} MiB)  t={t}  →  {ms:.0f} ms{flag}")
```

典型输出（M2 Pro 笔记本）：

```
m= 12288 (12.0 MiB)  t=1  →  85 ms
m= 12288 (12.0 MiB)  t=2  →  150 ms
m= 19456 (19.0 MiB)  t=2  →  230 ms ✓
m= 32768 (32.0 MiB)  t=1  →  195 ms
m= 32768 (32.0 MiB)  t=2  →  350 ms
m= 65536 (64.0 MiB)  t=1  →  380 ms
```

选 `m=19456, t=2`（230ms）或 `m=32768, t=1`（195ms）——两者都在目标范围内，但前者内存更高（攻击者成本更高），后者时间更高（攻击者 CPU 成本更高但内存成本低）。**优先选内存更高的组合**。

> **调优原则**：先固定目标时间（通常 200-500ms），然后在这个约束下最大化 m（内存），最后调 t。内存是攻击者不可压缩的成本，时间是可被并行压缩的。

## 四、缺陷与边界：Argon2 不是银弹

### 1. cache-hard 之争：bcrypt 在短运行时可能更优

2024 年 11 月，Soatok 发表《Beyond Bcrypt》，PHC 的多名参与者事后反思：**过度强调了 memory-hardness。**

论点：在 <1 秒的交互式登录场景下，Argon2id 能用的内存有限（几十 MB）。攻击者用高端 GPU 仍能并行多个实例——因为 GPU 有足够显存。而 bcrypt 的 cache-hard 特性（~4KB L1 cache 访问）在这个时间预算下，让 GPU 的每核心效率远低于 CPU。

bcrypt wiki 甚至声称：运行时间 <1 秒时 bcrypt 比 Argon2 更强（标注 citation needed，但有逻辑支撑）。

这不是"Argon2 不安全"，而是"在特定时间预算下，cache-hard 可能比 memory-hard 更划算"。Pufferfish2（cache-hard 方案，只在 L2 缓存内操作）是这一方向的代表演化。

### 2. 内存消耗是真实成本

Argon2id 的 memory-hardness 对防御者也是成本。m=19456（19 MiB）意味着**每次密码验证都分配 19 MiB 内存**。

- 单用户登录：无感
- 100 并发登录：1.9 GB 内存——中等服务器扛得住
- 10000 并发登录（DDoS 场景）：190 GB 内存——任何服务器都会 OOM

这是 Argon2id 的固有弱点：**它让攻击者费内存，但也让防御者费内存**。bcrypt 没有这个问题（4KB cache，几乎不占内存）。高并发登录场景需要 rate limiting 或用 bcrypt 兜底。

### 3. 侧信道的现实威胁

Argon2id 的前半趟用 Argon2i 模式（data-independent）来抗侧信道。但侧信道攻击需要攻击者**能在目标机器上运行代码**（如云环境的邻居租户）。对大多数自托管应用，侧信道不是现实威胁——Argon2d（纯 data-dependent，更强抗 GPU）可能更合适。

OWASP 仍推荐 Argon2id 作为默认，因为"侧信道不是现实威胁"这个假设在某些部署场景下不成立。但如果你的应用确定不受侧信道威胁（如本地部署、专用服务器），Argon2d 的抗 GPU 能力更强。

### 4. FIPS 合规性

Argon2 不在 FIPS 140-2 认证算法列表中。美国联邦信息系统必须用 FIPS 认证算法——这意味着只能用 PBKDF2-HMAC-SHA256（≥600,000 迭代）。如果你的项目需要 FIPS 合规，Argon2id 不可用。

### 5. 实现成熟度

Argon2 的参考实现是 C 语言，各语言绑定通过 FFI 调用。相比 bcrypt（20+ 年实战），Argon2 的生产环境使用历史较短（2015 年竞赛冠军，2021 年 RFC）。虽然没有已知安全漏洞，但实现层面的 bug 比 bcrypt 更可能出现——这也是"用经过审计的库，不要自己实现"的原因。

## 五、总结：Argon2id 的本质

回到根本问题：Argon2id 是什么？

**Argon2id 是第一个在"计算时间"和"内存用量"两个维度同时可调的密码哈希算法。** 在它之前，bcrypt 调时间（cost），scrypt 调内存（N），但没有一个算法能独立控制两者并兼顾抗侧信道。

### 三代 memory-hard 的演化

| 代 | 算法 | 年份 | 硬件成本维度 | 抗侧信道 |
|----|------|------|-------------|---------|
| 1 | bcrypt | 1999 | cache（~4KB） | 不考虑 |
| 2 | scrypt | 2009 | 内存（可调，N×r×128 字节） | 不考虑 |
| 3 | Argon2id | 2015 | 内存（可调 m）+ 时间（可调 t）+ 并行（可调 p） | 前半趟抗侧信道 |

每一代都在上一代的基础上增加了一个维度：bcrypt 增加了"cache 硬度"，scrypt 增加了"内存硬度"，Argon2id 增加了"抗侧信道"和"三参数独立可调"。

### 选型决策

| 场景 | 推荐 | 理由 |
|------|------|------|
| 新项目，无合规限制 | **Argon2id** (m=19456, t=2, p=1) | OWASP 首选 |
| 遗留系统用 bcrypt | **保持 bcrypt，cost≥12** | 迁移成本 > 安全收益 |
| FIPS 合规 | **PBKDF2-SHA256, ≥600k** | 唯一合规选项 |
| 高并发登录（DDoS 风险） | **bcrypt cost=12** 或 Argon2id + rate limit | Argon2id 的内存消耗是双刃剑 |
| 确定无侧信道威胁 | **Argon2d**（比 id 更强抗 GPU） | data-dependent 访问更难并行 |
| <1s 极端安全需求 | **bcrypt cost=14+** 或 Pufferfish2 | cache-hard 在短运行时可能更优 |

### 调优心法

1. **先定目标时间**（200-500ms），再选参数——不要盲目照搬 OWASP 默认值
2. **内存优先于时间**——攻击者不可压缩内存，但可并行压缩时间
3. **p 通常保持 1**——多线程降低用户体验且增加防御者内存（p×m）
4. **定期 check_needs_rehash**——硬件升级后调高参数，用户无感知升级

### 一句话

Argon2id 的本质不是"比 bcrypt 更安全"，而是**在攻击者成本的二维空间（时间×内存）里，给了防御者更多的自由度**。bcrypt 把你锁在"调时间"一个维度，Argon2id 让你同时调时间、内存、并行度——但自由度也是责任，调错了（比如 m 太低）可能不如 bcrypt 安全。

---

## 参考资料

- [Argon2 - Wikipedia](https://en.wikipedia.org/wiki/Argon2) — PHC 冠军，RFC 9106，三个变体
- [RFC 9106 - Argon2](https://www.rfc-editor.org/rfc/rfc9106) — 官方规范，推荐 Argon2id
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — Argon2id 首选 (m=19456, t=2, p=1)
- [NIST SP 800-63B §5.1.1.2](https://pages.nist.gov/800-63-3/sp800-63b.html) — memory-hard SHOULD be used
- [argon2-cffi (Python)](https://argon2-cffi.readthedocs.io/) — Python 绑定，含 check_needs_rehash
- [Soatok, "Beyond Bcrypt", 2024-11-27](https://soatok.blog/2024/11/27/beyond-bcrypt/) — cache-hard vs memory-hard 争论
- [本站 BCrypt 密码哈希：从 Blowfish 到 cache-hard 的技术演化](/2026/07/01/cs-bcrypt-explained/) — bcrypt 原理、cache-hard 概念、四代哈希演化


---

# BCrypt 密码哈希：从 Blowfish 到 cache-hard 的技术演化
Date: 2026-07-01 | Tags: 计算机基础, 密码学, 安全, BCrypt | URL: https://bsheepcoder.github.io/2026/07/01/cs-bcrypt-explained/

## 一、元认知：哈希为什么不能"快"

存密码这件事，反直觉的地方在于：**哈希越快越危险**。

MD5 每秒能算十几亿次，SHA-256 每秒几亿次。这些"快"在验证文件完整性时是优点——但在存密码时是致命伤。攻击者拿到泄露的哈希库后，用 GPU 每秒尝试几十亿个密码组合，快哈希让暴力破解变得可行。

密码哈希要的不是"快"，是**故意慢**——慢到合法用户登录时多等 200 毫秒无所谓，但攻击者暴力破解 10 亿个密码时要花几百年。

这就是 BCrypt 要解决的根本矛盾：**如何让哈希速度可调，且只能往慢调，不能被攻击者往快调。**

### 彩虹表的威胁

在 BCrypt 之前，密码存储的主要威胁是彩虹表——预先算好的哈希→密码对照表。一个 120GB 的彩虹表能在几秒内破解绝大多数常见密码的 MD5 哈希。

防御彩虹表的标准手段是**加盐**（salt）：每个密码加一段随机字符串再哈希，让预计算表失效。但光加盐不够——如果哈希本身太快（如 MD5+salt），攻击者仍能用 GPU 高速尝试每个盐对应的密码。

BCrypt 的贡献不是"加了盐"（它内置 128-bit 盐），而是**让哈希本身变慢，且慢的程度可调**。

## 二、搭积木：BCrypt 怎么工作的

### 从 Blowfish 到 Eksblowfish

BCrypt 由 Niels Provos 和 David Mazières 在 1999 年 USENIX 会议上提出，论文标题是 **"A Future-Adaptable Password Scheme"**（一个面向未来可适应的密码方案）。名字里的 "B" 来自它所基于的对称加密算法——**Blowfish**，由 Bruce Schneier 于 1993 年设计。

Blowfish 是一个分组密码，核心是**密钥扩展**（key schedule）：用密钥初始化 18 个 32-bit 子密钥（P 数组）和 4 个 8×32 的 S 盒。这个过程有一次性的初始化成本。

Provos 和 Mazières 的关键改造是 **Eksblowfish**（Expensive Key Schedule Blowfish）：

```
标准 Blowfish 密钥扩展：
  初始化 P/S-box → 用密钥执行一次扩展 → 结束

Eksblowfish 密钥扩展：
  初始化 P/S-box → 用盐+密钥执行 N 轮扩展 → 结束（N = 2^cost）
```

区别在于：标准 Blowfish 的密钥扩展只跑一遍，Eksblowfish 跑 **2^cost 遍**。而且第一遍之后的状态被"冻结"——后续的加密过程都用这个昂贵的初始化状态。

这个设计的关键在于：**盐也参与密钥扩展，且扩展轮数可调**。攻击者无法预计算（因为每个盐不同），也无法加速（因为轮数由防御者设定）。

### Cost Factor：自适应的慢

BCrypt 的 cost 参数控制 Eksblowfish 的迭代轮数：

```
cost = 10  →  2^10 = 1,024 轮
cost = 12  →  2^12 = 4,096 轮
cost = 14  →  2^14 = 16,384 轮
```

cost 每加 1，计算时间翻倍。范围是 4-31（但 31 会让单次哈希耗时达到数年，实际没人用）。

**这就是 "Future-Adaptable" 的含义**：1999 年用 cost=6（约 30ms），2025 年用 cost=12（约 300ms）——硬件变快了，就把 cost 调高，让哈希时间始终维持在"用户无感但攻击者痛苦"的水平。这是 BCrypt 最前瞻的设计。

### 输出格式

一个 bcrypt 哈希长这样：

```
$2b$12$nKj8Vp3T5r2YwE8mF1qK2O5xQ9zL8aB3cD4eF5gH6iJ7kL8mN9oP0
```

拆解：

| 段 | 含义 |
|----|------|
| `$2b$` | 算法版本 |
| `12` | cost factor |
| `nKj8Vp3T5r2YwE8mF1qK2O` | 22 字符的 Base64 编码盐（128 bit） |
| `5xQ9zL8aB3cD4eF5gH6iJ7kL8mN9oP0` | 31 字符的 Base64 编码哈希（184 bit） |

盐和哈希编码在同一字符串里——**不需要单独存盐**。这是 bcrypt 的工程友好设计：一个字符串包含验证密码所需的全部信息。

### 72 字节限制

BCrypt 的密码输入上限是 **72 字节**。原因在 P 数组：18 个 32-bit 子密钥 = 72 字节，密码循环填充这 72 字节后，多余部分被忽略。

这个限制的后果：

- **多数实现静默截断**——第 73 字节起的密码被丢弃，用户以为输了个长密码，实际只用了一部分
- **全 4 字节 UTF-8 字符（如 emoji）时，72 字节仅 18 个字符**——密码长度感知严重失真
- **2024 年 Okta 因此出过漏洞**：用户名+密码拼接后过 bcrypt，当用户名超过 52 字符时，密码部分被完全截断丢弃，任意密码都能登录

不同实现对超长密码的处理不一致：

| 实现 | 行为 |
|------|------|
| OpenBSD 原始实现 | 静默截断 |
| Python `bcrypt` | 抛异常 |
| Node.js `bcrypt` | 静默截断 |
| Rails `has_secure_password` | 显式限制 72 字符 |

> **实践建议**：如果用 bcrypt，要么在应用层强制限制密码长度 ≤72 字节，要么先用 SHA-256 预哈希再过 bcrypt（但预哈希引入新的"脚枪"问题——预哈希泄露后彩虹表攻击可行，需加 HMAC）。

## 三、案例即原理：两个 bug 和版本史

BCrypt 的版本号（`$2$` → `$2a$` → `$2b$`）不是功能迭代，而是**两个独立 bug 的修复记录**。这两个 bug 恰好揭示了密码哈希实现中最容易出错的两个维度：字符编码和整数溢出。

### Bug 1：$2a$ 的 8-bit 字符问题（2011）

2011 年 6 月，`crypt_blowfish`（PHP 用的 bcrypt 实现）被发现有一个 bug：当密码中包含**第 8 位为 1 的字符**（即字节值 >127 的非 ASCII 字符）时，处理逻辑有误。

具体来说，`$2a$` 规范要求密码以 UTF-8 编码并包含 null 终止符，但 crypt_blowfish 的实现对高位字节的处理与规范不一致。这导致**同一个密码在不同实现下可能产生不同的哈希**——在安全领域，这叫"哈希不一致"，意味着迁移系统时密码可能失效，或者更糟，某些密码的哈希碰巧变弱。

修复方式不是改 `$2a$` 规范，而是新增两个版本标记：

- `$2x$`：标记"已知有问题的旧哈希"，用于识别已存储的弱哈希
- `$2y$`：标记"修复后的算法"

> **重要**：`$2x$` 和 `$2y$` 是 **crypt_blowfish/PHP 专有**的。OpenBSD（bcrypt 的原始实现）和 Canonical（Ubuntu/glibc）从未采纳这两个版本标记。"Nobody else, including Canonical and OpenBSD, adopted the idea of 2x/2y."

### Bug 2：$2b$ 的 wrap-around（2014）

2014 年 2 月，OpenBSD 发现自己的 bcrypt 实现有一个更严重的 bug：**密码长度用 unsigned 8-bit 值存储**。当密码超过 255 字节时，长度值取 `length mod 256`——比如 260 字节的密码被当成 4 字节处理。

这意味着一个 260 字节的密码和前 4 字节相同的一个短密码，产生完全相同的哈希。虽然 bcrypt 本身截断到 72 字节，但这个 wrap-around bug 在截断之前就发生了，导致长度校验完全失效。

OpenBSD 修复后将版本改为 `$2b$`。这是当前**推荐使用的版本**。

### 两个 bug 的启示

| 维度 | Bug 1（$2x/$2y$） | Bug 2（$2b$） |
|------|-------------------|---------------|
| 出错点 | 字符编码处理 | 整数溢出 |
| 影响范围 | crypt_blowfish 专有 | OpenBSD 原始实现 |
| 发现年份 | 2011 | 2014 |
| 修复方式 | 新增 $2y$ 标记 | 新增 $2b$ 版本 |

密码哈希实现是最不容错的代码——一个 off-by-one 或编码错误就可能让安全性归零。这也是为什么 OWASP 强调"不要自己实现密码哈希，用经过审计的库"。

## 四、衍生：从 CPU-hard 到 memory-hard

### BCrypt 的定位：cache-hard

BCrypt 的 Eksblowfish 在计算时访问约 4KB 的数据（P 数组 + S 盒），这个大小恰好** fits in CPU L1 cache**。攻击者用 GPU 加速时，GPU 的每个核心没有独立的 L1 cache，4KB 的随机访问在 GPU 上变成全局内存访问，慢一个数量级。

这种特性叫 **cache-hard**——利用 CPU 缓存层次结构让 GPU/ASIC 不划算。BCrypt 是最早的 cache-hard 方案，虽然 1999 年提出时还没有这个术语。

### scrypt（2009）：memory-hard 的开端

Colin Percival 在 2009 年为 Tarsnap 在线备份服务设计了 scrypt（RFC 7914）。核心思想不同：**不只用 CPU 时间，还用大量内存**。

scrypt 要求在计算过程中维护一个大的内存缓冲区（默认几 MB 到几十 MB），且访问模式是随机的。这意味着：

- CPU 计算快但内存少的设备（如 ASIC）无法加速——因为没有足够的内存并行计算
- 即使有内存，随机访问模式让内存带宽成为瓶颈

```
bcrypt：  CPU 时间 × 4KB cache → GPU 不划算（cache-hard）
scrypt：  CPU 时间 × N MB 内存 → ASIC 不划算（memory-hard）
```

scrypt 的攻击者成本从"买更多 GPU"变成了"买更多 GPU + 买更多内存"，成本指数级上升。

### Argon2（2015）：PHC 冠军

2013-2015 年的 Password Hashing Competition（PHC）收到了 24 个提案，最终 **Argon2** 胜出（Alex Biryukov、Daniel Dinu、Dmitry Khovratovich，卢森堡大学），2015 年公布，2021 年成为 RFC 9106。

Argon2 有三个变体：

| 变体 | 内存访问模式 | 抗侧信道 | 适用场景 |
|------|------------|---------|---------|
| Argon2d | 依赖密码（data-dependent） | 弱 | 加密货币、不怕侧信道的场景 |
| Argon2i | 独立于密码（data-independent） | 强 | 密码哈希、抗侧信道 |
| Argon2id | 混合（前半趟 i，后半趟 d） | 中 | **RFC 9106 推荐默认** |

Argon2id 是 OWASP 当前推荐的首选密码哈希算法。相比 scrypt，它增加了**并行度参数**（p）和**可调内存/时间**的独立控制，且抗侧信道攻击。

### PBKDF2（2000）：最老但最兼容

PBKDF2（Password-Based Key Derivation Function 2，RFC 2898 / PKCS #5 v2.0）是 2000 年的方案，比 bcrypt 还早一年。它的原理最简单：把密码和盐用 HMAC 重复迭代 N 次。

PBKDF2 的问题在于**纯 CPU-hard，没有内存硬度**。它可以用 GPU 高效并行加速——一块 RTX 4090 每秒能算几百万次 PBKDF2-SHA256。

但 PBKDF2 有一个 bcrypt/Argon2 没有的优势：**FIPS 140-2 合规**。美国联邦信息系统要求 FIPS 认证的算法，Argon2 和 bcrypt 不在 FIPS 列表中，PBKDF2 是唯一合规的密码哈希方案。

### 四代哈希的演化轴

```
第一代：MD5/SHA + salt      →  快，GPU 可暴力破解
第二代：bcrypt (1999)        →  cache-hard，GPU 不划算
第三代：scrypt (2009)        →  memory-hard，ASIC 不划算
第四代：Argon2id (2015)      →  memory-hard + 并行 + 抗侧信道
```

演化的核心轴是**提高攻击者的硬件成本**：从纯 CPU 时间（MD5），到 CPU 缓存（bcrypt），到大内存（scrypt），到可调内存+并行+抗侧信道（Argon2）。

## 五、前沿：cache-hard vs memory-hard 之争

### OWASP 的推荐与现实的落差

OWASP 2024/2025 Password Storage Cheat Sheet 的推荐顺序：

| 优先级 | 算法 | 参数 | 场景 |
|--------|------|------|------|
| 1 | Argon2id | 19 MiB, t=2, p=1 | 新项目首选 |
| 2 | scrypt | N=2^17, r=8, p=1 | Argon2id 不可用时 |
| 3 | bcrypt | cost≥10 | **仅遗留系统** |
| 4 | PBKDF2 | SHA-256, ≥600k 迭代 | FIPS 合规场景 |

注意 bcrypt 已被降级为"legacy only"。但现实是：

| 系统 | 默认算法 | 实际推荐 |
|------|---------|---------|
| PHP `password_hash()` | **bcrypt** (cost=12) | Argon2 需编译启用，非默认 |
| Ruby on Rails | **bcrypt** | 显式限制 72 字符 |
| Django | **PBKDF2-SHA256** | Argon2 需第三方库 |
| WordPress | **PHPass (MD5-based)** | 2024 年仍在讨论迁移 bcrypt |
| Linux shadow | SHA-512 / **yescrypt** | Fedora/Arch 已转 yescrypt |

OWASP 推荐 Argon2id，但现实中大量系统仍用 bcrypt 甚至更弱的算法。迁移有成本——已有密码库要平滑过渡，不能要求所有用户改密码。

### 2024 前沿：cache-hard 被重新评估

2024 年 11 月，密码学家 Soatok 发表《Beyond Bcrypt》，引发了一场重要讨论：**PHC 过度强调了 memory-hardness。**

论点核心：

- **密码认证场景的运行时间通常 <1 秒**。在这个时间预算内，memory-hard 函数（scrypt/Argon2）能用的内存有限（几十 MB），攻击者用高端 GPU 仍能并行多个实例
- **cache-hard 函数**（如 bcrypt）在 <1 秒时，利用 L1 cache 的速度优势，每秒能做更多次有效计算——但攻击者的 GPU 没有 L1 cache，每秒有效计算反而更少
- **bcrypt wiki 甚至声称**：运行时间 <1 秒时 bcrypt 比 Argon2 更强（标注 citation needed，但有逻辑支撑）

讽刺的是：bcrypt 是 cache-hard，而 scrypt 和大多数人用的 Argon2 配置**不是** cache-hard（它们的内存访问模式是随机的，不利用 cache 层次）。

### Pufferfish2 和 bscrypt：cache-hard 的新尝试

沿着 cache-hard 方向的新探索：

- **Pufferfish2**：基于 PHC 决赛入围方案 Pufferfish 改进，只在 CPU 核 L2 缓存内操作（非随机访问 RAM），比 scrypt/Argon2 更难用 GPU/ASIC 实现
- **bscrypt**：Sc00bz 设计的 cache-hard 方案，论证在短运行时下 cache-hard 优于 memory-hard
- **Balloon Hashing**：NIST SP 800-63B 点名推荐，可证明 space-hard，数据独立访问（抗侧信道），但不是 cache-hard

这些方案目前都未成为主流标准，但代表了一个重要的方向：**密码认证场景可能不需要"大内存"，而是需要"高效利用 CPU 缓存层次"**。

### 量子计算的影响

Grover 算法对无结构搜索给出平方加速——2^N 次暴力搜索变成 2^(N/2) 次。对密码哈希的影响：

- **不"破解"哈希**——Grover 不攻击哈希函数本身，只加速暴力搜索
- **缓解方法简单**：约翻倍 work factor（cost +1 即可，因为 bcrypt 的 cost 是 2 的幂）
- **真正的瓶颈是密码熵**——如果用户用 8 位弱密码，不管哈希多慢，搜索空间只有 2^52（考虑字符集），量子计算下 2^26 次搜索就能破

> 量子计算对密码哈希的威胁远小于对公钥密码（RSA/ECC）的威胁。Shor 算法能分解大整数，直接破解 RSA——但 Shor 不适用于哈希函数。密码哈希的量子安全性主要取决于密码熵，而非哈希算法选择。

## 六、总结：密码哈希的本质

回到最初的问题：BCrypt 是什么？

**BCrypt 是第一个"可调速度"的密码哈希方案。** 在它之前，哈希速度由算法固定（MD5 永远那么快）；在它之后，防御者可以随着硬件升级调高 cost，让哈希始终保持"用户无感但攻击者痛苦"的速度。

它的两个核心设计——**Eksblowfish 的可调迭代**和**内置盐**——至今仍是密码哈希的基本范式。scrypt 和 Argon2 改进的是"攻击者的硬件成本"维度（从 CPU 时间到内存），但没有改变这个基本范式。

### 选型决策

| 场景 | 推荐 | 理由 |
|------|------|------|
| 新项目，无合规限制 | **Argon2id** | OWASP 首选，memory-hard + 抗侧信道 |
| 遗留系统用 bcrypt | **保持 bcrypt，cost≥12** | 迁移成本 > 安全收益，调高 cost 即可 |
| FIPS 合规（美国联邦） | **PBKDF2-SHA256, ≥600k 迭代** | 唯一合规选项 |
| 极端安全需求（<1s 预算） | **bcrypt cost=14+ 或 Pufferfish2** | cache-hard 在短运行时下可能更优 |

### 一句话

密码哈希的本质不是"加密"，而是**让暴力破解的成本高到不可行**。bcrypt 用可调的慢做到了这一点，scrypt/Argon2 用内存硬度做到了更多，而 2024 年的 cache-hard 争论提醒我们：**在密码认证这个特定场景下，"慢"不只有一种正确的方式。**

---

## 参考资料

- [Provos & Mazières, "A Future-Adaptable Password Scheme", USENIX 1999](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf) — bcrypt 原始论文
- [bcrypt - Wikipedia](https://en.wikipedia.org/wiki/Bcrypt) — 版本历史、72 字节限制、算法细节
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — Argon2id 首选，bcrypt legacy only
- [NIST SP 800-63B §5.1.1.2](https://pages.nist.gov/800-63-3/sp800-63b.html) — memory-hard SHOULD be used
- [Argon2 - Wikipedia](https://en.wikipedia.org/wiki/Argon2) — PHC 冠军，RFC 9106，三个变体
- [Soatok, "Beyond Bcrypt", 2024-11-27](https://soatok.blog/2024/11/27/beyond-bcrypt/) — cache-hard vs memory-hard 争论
- [RFC 9106 - Argon2](https://www.rfc-editor.org/rfc/rfc9106) — Argon2 官方规范
- [RFC 7914 - scrypt](https://www.rfc-editor.org/rfc/rfc7914) — scrypt 官方规范


---

# 深入浅出 TEE：为什么你的手机里有一块内核都进不去的保险箱
Date: 2026-06-29 | Tags: 计算机基础, TEE, 可信执行环境, 硬件安全 | URL: https://bsheepcoder.github.io/2026/06/29/cs-tee-principles/

## 元认知：所有软件安全都建立在一个谎言上

### 特权层的窥探悖论

通用 CPU 上有一个根本矛盾——**执行代码的层，永远能被更上层的特权层窥探**。

```
应用层      ←  内核能读它的内存
内核层      ←  hypervisor 能读它的内存
hypervisor  ←  ？谁来保证它不被读？
```

这条链条没有尽头。无论你在应用层做了多精妙的加密、签名、沙箱，**只要密钥最终要加载进普通内存被 CPU 执行，那一刻它就是明文，能被更高特权级读到**。

这不是某个操作系统的 bug，是冯·诺依曼架构的固有缺陷：代码和数据共享同一内存空间，"谁特权高谁就能看"。你写的每一行安全代码，都跑在一个你无法控制的特权层之下。

### 软件隔离的信任假设

所有软件安全方案——全盘加密、OS 沙箱、容器隔离、虚拟机——都建立在一个未明说的假设上：

> **内核 / hypervisor 是可信的。**

这个假设在日常场景下成立。但在以下情况会破裂：

- 内核零日漏洞（rootkit 提权后，全盘加密的密钥从内核内存里被掏走）
- 云环境中的恶意 hypervisor（云厂商或被入侵的宿主能读取你的虚拟机内存）
- 供应链攻击（固件被植入后门）

一旦这个假设破裂，上层一切防护归零。**因为软件隔离的本质是"约定不碰"——约定内核不去读应用的内存。但约定可以被打破。**

### TEE 要解决的根本问题

TEE（Trusted Execution Environment，可信执行环境）要解决的是一个极端问题：

> **在"连内核都不可信"的前提下，仍然能保护一段代码和数据的机密性。**

这只能靠硬件。因为软件再聪明，运行时终究要落到 CPU 和内存上，而这层由硬件掌控。软件说"这块内存不能读"，内核可以说"我不管，我直接读物理地址"。但**硬件说"这条总线事务我不放行"，内核没有任何办法绕过**——因为它面对的是晶体管，不是 if 判断。

> TEE 的本质不是"更安全的软件"，而是**让 CPU 自己划出一片"内核也进不去"的区域**。把安全信任的锚点从"可被攻破的操作系统"下沉到"物理上无法篡改的硅片"。

---

## 搭积木：TEE 的六块原理积木

理解了"为什么要靠硬件"，接下来看硬件具体怎么做。TEE 不是一项单一技术，是六块积木拼成的体系。每一块解决一个原理问题。

### 积木一：在总线上画一道硬件的墙

以 ARM TrustZone 为例。它在 CPU 硬件总线上加了一个**位**——`NS`（Non-Secure）bit。

```
每个总线事务都带一个 NS 位：
  NS = 1 → Normal World 请求（普通系统）
  NS = 0 → Secure World 请求（安全世界）
```

这个位由 CPU 当前状态决定，**软件改不了**（除非通过受控的切换指令）。总线上的所有外设——内存控制器、加密引擎、中断控制器、Flash 控制器——都看这个位来决定是否响应。

```
                    ┌─────────────────────┐
                    │     物理内存         │
                    │  ┌───────────────┐  │
                    │  │ Secure 区域    │  │ ← NS=0 才能访问
                    │  ├───────────────┤  │
                    │  │ Normal 区域    │  │ ← NS=1 能访问
                    │  └───────────────┘  │
                    └─────────────────────┘
                          ↑
              内存控制器看 NS bit 决定放行
```

**关键原理**：隔离不是靠软件检查权限，而是靠总线物理上拒绝。Normal World 发出的地址哪怕精确指向 Secure 区域的物理地址，也会被内存控制器直接拒绝——这是晶体管级别的拒绝，不是操作系统的 if 判断。

> 一个类比：普通系统是开放大办公室，TEE 是里面的金库。不是说"员工约定不进金库"，而是金库的门禁是物理的——员工没有门禁卡，门就是打不开，连物业经理（内核）都不行。

Intel SGX 用的是另一种实现——不分割世界，而是在进程地址空间内划出 Enclave，CPU 的内存加密引擎（MKTME）把 Enclave 的内存加密，内核即使读到物理地址也只能看到密文。实现不同，原理相同：**硬件级拒绝，不是软件级约定**。

### 积木二：两个世界之间受控的门

光有墙不行，还得有受控的"门"。否则 Secure World 永远运行不了任何代码，密钥永远用不上。

ARM 用一条专用指令 `SMC`（Secure Monitor Call）做切换：

```
Normal World  ──SMC──>  Monitor Mode  ──>  Secure World
                  ↑
            这是一条硬件陷阱，触发后：
            1. CPU 自动切到 NS=0
            2. 跳转到 Secure World 预设入口
            3. 保存 Normal World 的全部寄存器状态
```

切换的入口地址在 CPU 复位时由硬件固定（或由 ROM 锁定），**软件无法篡改这个入口**。所以 Normal World 不能"随便跳进 Secure World 任意地址"，只能跳进这个固定入口，然后由 Secure World 的代码决定做什么。

> **最小信任接口**是安全设计的核心原则。攻击面只有这一个入口 + 它暴露的服务调用（密钥操作、签名等），而不是整个内核。墙画好了，门只有一个，而且门后面有人查身份。

### 积木三：信任链锚定在硅片上

上面说的"硬件固定的入口"——凭什么相信它本身是干净的？万一 Secure World 的代码被换掉呢？

这靠 **Secure Boot**，本质是一条**信任链**：

```
1. CPU 上电 → 从片内 ROM（厂商烧死、不可改）的固定地址取指
              └─ 这是硬件 root of trust，物理上无法改

2. ROM 里的代码验证 bootloader 的签名（用烧死在 ROM 里的公钥）
              └─ 签名不对就拒绝启动

3. bootloader 验证 Secure World OS 的签名
4. Secure World OS 验证 TA（Trusted Application）签名
```

每一级只信任**上一级已验证过的代码**，链的根是片内 ROM——**硅片本身**。要攻破这条链，必须物理替换芯片或破解非对称签名，软件层面无解。

> 原理核心：信任不能凭空产生，必须有锚点。软件的锚点在硬盘上（可被改写），硬件的锚点在硅片里（出厂烧死，物理不可改）。**Secure Boot 把整个信任体系从"可刷写的固件"下沉到"不可改的硅片"。**

这是为什么 TEE 能扛住 rootkit——内核可以被植入后门，但后门改不了 ROM 里的验证逻辑。

### 积木四：向陌生人证明"我是我"

本地隔离做完了，但还有个问题：你的手机 TEE 里跑着银行 App，银行服务器怎么知道这真的是你的手机 TEE，而不是攻击者伪造的？

这叫**远程证明（Remote Attestation）**，原理是：

```
1. 芯片出厂时，硬件烧死一个唯一私钥 SK_device（对应公钥 PK_device）
   └─ 这个私钥从不离开芯片，从不进入内存明文

2. 厂商 CA 用自己的根私钥给 PK_device 签发证书
   └─ 证书证明"这个公钥属于一块合法 TEE 芯片"

3. 证明时：
   - 服务器发一个随机挑战 nonce
   - TEE 用 SK_device 对 (nonce + 当前运行代码的哈希) 签名
   - 把签名 + 代码哈希 + 证书发给服务器

4. 服务器验证：证书链 → 签名 → 对比代码哈希
   └─ "确实是合法芯片，且里面跑的是预期的代码"
```

> 原理核心：**用硬件私钥做签名，让"我是谁 + 我在跑什么"成为密码学可验证的事实，而非口头声明。** 私钥永不出芯片，所以无法伪造。这是"证明"和"声明"的本质区别——声明可以复制，签名不可伪造。

远程证明是 TEE 区别于"纯本地加密"的关键能力。没有远程证明，你只能自己信自己；有了远程证明，你能向全世界证明"这段代码确实跑在合法 TEE 里，且未被篡改"。

### 积木五：数据不出去的流模型

这是 TEE 设计上最反直觉、也最关键的积木。

你可能想：把密钥读出来用一下再擦掉不行吗？

**不行**。原因是 **TOCTOU 问题**（Time-of-Check to Time-of-Use）的极端化：

```
T0:  密钥在 TEE 里
T1:  把密钥读出 → 拷到普通内存（明文）
T2:  CPU 用密钥加密数据
T3:  擦掉普通内存里的密钥

        ↑
   T1~T3 之间，密钥暴露在普通内存
   内核 / DMA / 中断在这窗口内可以把它偷走
```

这个窗口哪怕只有几微秒，攻击者用 DMA 引擎或内核漏洞也能精确截获。所以 TEE 的设计原则是：

> **敏感数据永远不离开 Secure World 的内存边界。需要使用时，把"待处理的数据"送进去，在里面算完，只把"结果"送出来。**

```
❌ 错误流：密钥出来 → 在外面加密 → 擦掉密钥
           （T1~T3 暴露窗口）

✓ 正确流：密文进去 → TEE 内解密 + 计算 → 结果出来
          （密钥从未离开边界）
```

这本质上是把**数据流**改造成"输入穿越边界，输出穿越边界，中间过程全在墙内"。**密钥永远不出来，只把计算结果送出来。**

> 这就是为什么指纹解锁安全：指纹图像直接送进 TEE，在 TEE 内比对，只返回"通过"或"拒绝"给 Android。指纹原图、模板始终没离开过 TEE。Android 拿到的只是一个 yes/no。

### 积木六：现成硬件的版图

理解了原理，看现实——不是所有设备都有 TEE，不同阵营差异巨大：

| 平台 | TEE 方案 | 状态 |
|------|---------|------|
| ARM 移动端（高通/联发科/苹果/三星/华为） | TrustZone / Apple Secure Enclave | ✅ 全系覆盖，事实标准 |
| Intel 服务器 | SGX（进程级）→ TDX（VM 级） | SGX 限数据中心，TDX 是继任者 |
| AMD 服务器 | SEV-SNP（VM 级） | ✅ 主流云厂商标配 |
| ARM 服务器 | CCA | 🚧 早期 |
| 消费级 x86 PC | 无应用级 TEE | ❌ Intel 11 代后砍掉 SGX |
| Apple Silicon Mac | Secure Enclave | ✅ 消费级 PC 里少数有 TEE 的 |

**一个关键区分：TPM ≠ TEE。** 大部分 PC 有 TPM，但 TPM 只是密钥存储+签名的协处理器，**不能跑任意代码**，能力远弱于 TEE。TPM 是"保险箱的锁"，TEE 是"保险箱本身"。

> 你手机里大概率有 TEE 在跑——指纹、Face ID、支付全靠它。但你桌上的 Windows 笔记本基本没有。这是 Intel 主动放弃 SGX 的结果（后面缺陷部分会讲为什么）。

---

## 案例即原理：用 TEE 造一个数字记忆的根

六块积木单独看是抽象的。用一个真实场景把它们串起来——**Personal Root Device：一个保护个人数字记忆的硬件设备**。

这个案例不是假设。当 AI 越来越懂你，你的"数字记忆"（对话、偏好、工作流、人际关系）就成了比你银行卡密码更敏感的数据。如果记忆存在云端，它属于平台；如果存在本地，内核被入侵就泄露。**需要一块硬件，默认离线，用 TEE 做安全地基，让记忆真正属于你。**

### 为什么 Memory Point 天然适配 TEE

数字记忆的最小单位是 **Memory Point（记忆点）**——任何能够客观描述一个事实的内容：

```
今天认识了张三。
今天完成了项目的部署。
今天决定开始创业。
```

每个记忆点只有最基本的信息：时间、事实、来源、重要程度、标签、关联人物、关联项目。序列化后大约 **200-500 字节**。

算一笔账：

| 量级 | 体积 | TEE 能装下吗 |
|------|------|-------------|
| 每天 20 条记忆点 | ~10 KB | 轻松 |
| 一年 | ~3.6 MB | 进得去 |
| 十年 | ~36 MB | 临界——中等 TEE 能放 |
| 四十年 | ~144 MB | 放不进 TEE，但可加密存外部 |

> 这是整个架构能成立的关键。TEE 的安全内存通常只有几 MB 到几十 MB——这是所有 TEE 方案的硬伤。但数字记忆存的是**意义**（Mean Storage），不是原始数据（Everything Storage）。一条记忆点是"今天认识了张三"这种事实，不是会议录音。**轻量的"意义"可以整体放进 TEE，重量的"附件"加密存外部。**

### 分层架构：意义进 TEE，原始数据加密存外部

```
┌──────────────────────────────────────────────┐
│  TEE 内（Secure World，几 MB 安全内存）          │
│  ┌──────────────────────────────────────┐    │
│  │ ① 主密钥 MK（永不出去）                │    │  ← 积木五：数据不出去
│  │ ② Memory Point 全量（文本，MB 级）     │    │  ← 轻量意义，适配 TEE
│  │ ③ 经验图谱的节点+边（结构化，KB 级）      │    │
│  │ ④ 访问控制策略 + 授权令牌签发逻辑        │    │
│  │ ⑤ Merkle Root（防篡改日志的根哈希）     │    │  ← 积木三：信任链
│  │ ⑥ 远程证明私钥（设备身份）              │    │  ← 积木四：远程证明
│  └──────────────────────────────────────┘    │
├──────────────────────────────────────────────┤
│  普通存储（Flash，GB-TB 级）                   │
│  ┌──────────────────────────────────────┐    │
│  │ ⑦ 附件密文（照片/录音/视频，AES-256）    │    │  ← 用 MK 派生的数据密钥加密
│  │ ⑧ Memory Point 的追加日志（哈希链）      │    │
│  │ ⑨ 程序性记忆 skill 包                    │    │
│  └──────────────────────────────────────┘    │
├──────────────────────────────────────────────┤
│  接口层                                        │
│  USB-C 物理连接（默认离线）                     │
│  临时授权令牌（有时效，到期失效）                 │
└──────────────────────────────────────────────┘
```

这个架构里每一层都对应一块原理积木：

- **主密钥在 TEE，附件密文在 Flash**——对应积木五（数据不出去的流模型）。密钥永不离开 TEE，需要解密附件时，把密文送进 TEE 解密，只把结果送出来。
- **默认离线 + USB-C**——最彻底的安全来自不连接，而不是加密以后再联网。
- **AI 是访客不是拥有者**——AI 申请权限读取 Memory，读取结束权限立即失效，对应积木二（最小信任接口）。

### 不可篡改：Merkle Tree + 远程证明

"防止篡改"是核心需求。原理是 **append-only 日志 + Merkle Tree**：

```
每条 Memory Point 写入时：
  MP_n → hash(MP_n) → 与前一个叶子哈希组合 → 生成新的 Merkle Root

所有记忆点组成一棵哈希树，TEE 持有 Root
任何一条记忆被篡改 → 叶子哈希变化 → Root 变化 → 校验失败
```

**向第三方证明**时（比如法院要求证明"这条记忆确实生成于某天且未被篡改"），只需要提供：

```
1. 这条 Memory Point
2. 从它到 Merkle Root 的哈希路径（O(log n) 个哈希）
3. TEE 用设备私钥对 Root 的签名
```

> 原理串联：Merkle Tree 让"证明某条记忆存在且未篡改"不需要导出全部记忆，只需要对数级的哈希路径。TEE 对 Root 签名，把"整棵树的完整性"绑死在硬件身份上。这是积木三（信任链）和积木四（远程证明）的组合——信任链保证 TEE 代码没被篡改，远程证明保证记忆树没被篡改。

### 双芯片方案：记忆版硬件钱包

落地到具体硬件，推荐双芯片架构：

```
┌─────────────────────────────────┐
│  主 MCU（ESP32-S3 或 STM32）      │  ← 负责 USB-C 通信、文件系统、
│                                  │     附件加密存储、授权 UI
│  Flash：附件密文 + 追加日志        │
├─────────────────────────────────┤
│  Secure Element（NXP SE050）     │  ← 独立安全芯片
│                                  │
│  - 主密钥 MK                     │
│  - 远程证明私钥                   │
│  - 签名 / 解密操作                │
│  - EAL6+ 认证（防物理攻击）        │
└─────────────────────────────────┘
```

> 为什么双芯片而不是单 SoC？独立 Secure Element 的安全认证等级（EAL6+）远高于 SoC 内的 TrustZone。SE050 专门做密钥存储和加解密，物理攻击防护级别更高——防探针、防激光、防功耗分析。主 MCU 做重活，SE 做安全操作，两者通过 I2C 加密通信。

这本质上是硬件钱包（Ledger、Trezor）的架构。**Personal Root Device 就是"记忆版硬件钱包"——存的是记忆密钥而不是比特币私钥，但安全模型完全相同。**

### 设计决策回溯表

| 设计决策 | 对应的积木 | 原理 |
|---------|----------|------|
| 主密钥永不出 TEE | 积木五 | TOCTOU 窗口无法消除，数据必须不出去 |
| Merkle Tree 防篡改 | 积木三 | 信任锚定在硅片，哈希链不可伪造 |
| 远程证明可验证 | 积木四 | 硬件私钥签名，密码学可验证 |
| USB-C 默认离线 | — | 不连接比加密更安全 |
| 双芯片（MCU+SE） | 积木一 | 物理隔离优于软件隔离 |
| Memory Point 进 TEE | 积木五 | 轻量数据适配 TEE 内存限制 |

---

## 缺陷与批判：TEE 管不到的地方

TEE 不是银弹。理解它的边界，比理解它的能力更重要。

### 缺陷一：侧信道是架构级原罪

TEE 的安全承诺是"攻击者看不见内部计算"。但通用 CPU 为性能做了大量推测执行、缓存共享、分支预测——这些都成了侧信道。

```
enclave 内做 if (secret_bit) access A; else access B;
                          ↓
            缓存里留下痕迹 → 攻击者计时测量 → 反推 secret
```

> **这是架构级矛盾**：性能优化依赖共享微架构状态，安全隔离要求状态不共享。要彻底修就得砍掉乱序执行、砍掉共享缓存——CPU 性能倒退 20 年。所以侧信道不是 bug，是通用 CPU 的"原罪"。

SGX 的覆灭史是最佳案例。2018 年起一连串侧信道漏洞把"即使内核被攻破也安全"的承诺戳穿了：

| 漏洞 | 年份 | 后果 |
|------|------|------|
| Foreshadow (L1TF) | 2018 | 推测执行读 L1，能读 enclave 内密钥 |
| SGAxe | 2020 | **从 enclave 里掏出 Intel 远程证明根密钥** |
| ÆPIC Leak | 2022 | 架构级泄露 enclave 内存 |

**致命点是 SGAxe**——它泄露的是 Intel 用来做远程证明的根密钥，意味着攻击者可以伪造"这是合法 SGX"的证明。**远程证明整个体系崩塌**（积木四被攻破）。

每次漏洞 Intel 都得发微码补丁，补丁又拖性能，且总补不全。最终 Intel 砍掉消费端 SGX，把资源押到服务器端的 TDX（VM 级隔离）。**这是"架构级安全 vs 性能"矛盾在通用 CPU 上解不开的直接证据。**

### 缺陷二：TEE 保护设备，不保护 AI 的内存

回到数字记忆的案例。TEE 保护了你的 Personal Root Device——密钥不出设备，记忆不被篡改。但有一个 TEE 管不到的窗口：

```
1. AI 申请读取最近七天的记忆
2. TEE 校验授权令牌，返回明文 Memory Point 给 AI
3. AI 在自己的内存里处理
4. 授权到期，权限失效

        ↑
   步骤 3 中，AI 如果偷偷把 Memory Point 缓存到自己的存储
   权限失效后副本还在——TEE 管不到 AI 的内存
```

> **TEE 保护的是你的设备，不是 AI 的内存。** 这是 Memory Protocol 真正的难点，比"防篡改"难得多。这不是硬件能解决的，必须在协议层设计：返回给 AI 的每条记忆带盲水印（事后可追溯泄露源），或者 AI 侧也跑在 TEE 内（云端 Confidential VM），形成端到端可信链。

### 缺陷三：消费级 PC 的 TEE 缺位

这是最现实的问题。如果你想在自己的 Windows 笔记本上做 TEE 应用开发——做不到。

- **Intel 消费端**：11 代（Tiger Lake）之后取消 SGX，桌面/笔记本没有应用级 TEE
- **AMD 消费端**：Ryzen 系列没有 SEV，SEV 是 EPYC 专属
- **TPM**：大部分 PC 有，但 TPM ≠ TEE，只是密钥存储协处理器，不能跑任意代码

> 为什么 PC 放弃了？因为 SGX 的安全承诺被侧信道连续击穿，补丁补不动、性能还倒退；消费端生态又起不来——开发者难用、普通用户无感知、应用面窄。Intel 干脆砍掉消费端 SGX，把筹码全押到服务器端。**这是"架构级安全 vs 性能"矛盾的工程妥协。**

后果：要做 TEE 应用，要么面向移动端（Android StrongBox / Apple Secure Enclave），要么在云上租 Confidential VM。普通 x86 PC 基本是 TEE 荒漠。

### 缺陷四：信任锚定在芯片厂商

TEE 的整个信任链，根在芯片厂商：

- Secure Boot 的根公钥烧在 ROM 里——厂商决定
- 远程证明的证书链——厂商的 CA 签发
- TEE OS 的代码——通常厂商提供

> 用 NXP、ST、Apple 的芯片，你能蹭到厂商的可信链（这些厂商有成熟的 PKI 和安全审计）。但用白牌芯片，根信任就是未知数。**自研 TEE 基本不可行**——需要数亿流片费用、自己的 PKI 和 Secure Boot 根、安全审计。这是芯片公司干的活，不是个人或小团队能做的。**

信任的锚点在硅片，但硅片是厂商造的。这是 TEE 体系最底层的信任假设——**你必须信任芯片厂商**。这个假设比"信任操作系统"强得多（芯片厂商造假的动机和成本远高于黑客攻破内核），但它仍然是一个假设，不是物理定律。

---

## 总结：TEE 是什么

回到最开始的问题：TEE 到底是什么。

> **TEE 是 CPU 里的一块硬件保险箱，敏感数据进去后就出不来，只把处理结果送出来，连操作系统都碰不到里面。**

三句话概括原理：

1. **靠硬件在总线上画一道线**（NS bit）——让内核特权级在物理上失效。不是软件权限检查，是晶体管拒绝。
2. **靠 Secure Boot 把信任锚定在硅片上**——root of trust 是出厂烧死的 ROM，不是可刷写的固件。
3. **靠"数据不出去，只让计算进出"的流模型**——从根上消除"读出来用一下"的暴露窗口。

软件隔离是"约定不碰"，TEE 是"物理上碰不到"。

但"物理上碰不到"也有边界。侧信道是架构级原罪——性能优化和安全隔离共享微架构状态，这在通用 CPU 上解不开。TEE 保护设备，不保护数据离开设备后的去向。信任锚定在芯片厂商，你必须信任那家公司。

> 理解 TEE 的边界，比理解它的能力更重要。它能挡住内核、挡住 rootkit、挡住物理拆 Flash，但挡不住推测执行的缓存侧信道，挡不住 AI 偷偷缓存读取结果，挡不住芯片厂商自己。**TEE 是目前最硬的隔离手段，但它不是物理定律，是工程妥协。**

在数字记忆、个人 AI、隐私计算这些场景里，TEE 是安全地基的必要条件——没有它，一切软件隔离都建立在"信任操作系统"的沙滩上。但地基之上，还需要协议层（Memory Protocol）、端到端可信链、供应链审计来填补 TEE 管不到的缝隙。

硬件画线，信任锚硅，数据不出去。这是 TEE。但它之上的世界，还需要人去建。


