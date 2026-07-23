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
