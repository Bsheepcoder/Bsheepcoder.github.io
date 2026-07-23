## 一、元认知：Schema 演化的根本矛盾

应用代码出 bug，修一行代码重新部署就行。数据库 schema 出 bug--一个列名写错、一个索引漏建、一个类型选窄了--代价完全不同：生产数据已经在那张表里了，你不能 `git checkout` 回去。

这就是数据库 schema 演化的核心矛盾：**DDL 不是代码，没有 git 天然管理；但 DDL 出错的代价远高于代码**。更棘手的是，schema 变更必须跨环境（开发/测试/生产）可复现，还得多人协作不冲突。

传统做法有三种，各有致命缺陷：

- **手动执行**：DBA 在生产跑 SQL，没人记录跑没跑过，环境间必然漂移
- **SQL 文件丢仓库**：文件有了，但谁跑了谁没跑、跑了哪个版本、跑成功了没有--全凭记忆
- **ORM 自动建表**（Hibernate `ddl-auto: update`）：开发方便，生产是灾难--它只增量加表加列，绝不删列改类型，且不记录变更历史，schema 状态完全不可审计

Flyway 要解决的根本问题不是"执行 SQL"（这谁都会），而是**把 schema 变更降维成有序、不可变、可校验的迁移脚本，用一张账本表做唯一事实源**。

> Flyway 之于数据库 schema，近似 Git 之于源码--但带一条"只前进不后退"的强约束。这条约束是 Flyway 最有争议也最核心的设计决策。

## 二、搭积木：Flyway 怎么工作

### 账本表：唯一事实源

Flyway 在每个目标数据库中维护一张表，旧版叫 `schema_version`，Flyway 5.0 起更名为 `flyway_schema_history`。这张表就是"账本"--记录了哪些迁移脚本已经应用过、何时应用、耗时多久、是否成功。

核心字段：

| 字段 | 含义 |
|------|------|
| `installed_rank` | 应用顺序（1, 2, 3...） |
| `version` | 脚本版本号，如 `1`、`2.1.3` |
| `description` | 从文件名解析的描述 |
| `type` | `SQL`、`JDBC`、`SPRING_JDBC` 等 |
| `script` | 脚本文件名 |
| `checksum` | 文件内容的 CRC32 校验和 |
| `installed_by` | 执行的数据库用户 |
| `installed_on` | 执行时间戳 |
| `execution_time` | 执行耗时（毫秒） |
| `success` | 是否成功 |

这张表是整个系统的基石。Flyway 每次启动都读它，和本地脚本目录对比，决定"还差哪些没跑"。

### 命名契约：文件名即协议

Flyway 不靠注册中心、不靠配置文件，靠**文件名约定**区分迁移类型：

```
V1__create_user_table.sql          ← 版本迁移，执行一次
V1.1__add_email_column.sql         ← 版本 1.1，在 V1 之后 V2 之前
V2__create_order_table.sql         ← 版本 2
R__refresh_views.sql               ← 可重复迁移，checksum 变则重跑
```

两条规则：

- **`V{version}__{description}.sql`**：版本迁移（Versioned）。`version` 是点分数字（`1`、`1.1`、`2.1.3`），**按数值排序而非字典序**--`V9` 在 `V10` 之前（字典序会把 `V10` 排到 `V2` 前面，Flyway 不会犯这个错）。两个下划线 `__` 分隔版本和描述，描述中用单下划线或驼峰替代空格。每个版本号**只执行一次**。
- **`R__{description}.sql`**：可重复迁移（Repeatable）。没有版本号，在所有 V 迁移之后执行，且每当文件 checksum 变化就重新执行。适合重建视图、存储过程、种子数据。

> 命名契约就是协议。你不需要在任何地方注册"我加了 V3"--Flyway 扫描目录，按规则解析，自动纳入。这是约定优于配置的纯粹实践。

### 执行流程

一次 `migrate` 调用的完整流程：

```
1. 扫描 locations（默认 classpath:db/migration）所有脚本
2. 解析文件名，分类为 V / R，按版本号排序
3. 读取 flyway_schema_history，获取已应用脚本列表
4. validate：对比已应用脚本的 checksum 与当前文件 checksum
   ↳ 不一致 → 抛错，阻塞启动（除非配置 ignoreMigrationPatterns）
5. 计算待执行：未应用的 V 脚本 + checksum 变化的 R 脚本
6. 按顺序逐个执行，每个脚本在独立事务中（若数据库支持 DDL 事务）
7. 成功 → 写入 flyway_schema_history（含 checksum）
   失败 → 记录 success=0，抛异常，阻塞后续
```

### 六大命令

| 命令 | 作用 | 风险 |
|------|------|------|
| `migrate` | 应用待执行的迁移 | 低 |
| `info` | 显示当前状态（已应用/待执行/忽略） | 无 |
| `validate` | 校验已应用脚本 checksum 未被篡改 | 无 |
| `baseline` | 对已有库建立基线，标记 `baselineVersion` 之前的版本视为已应用 | 中 |
| `repair` | 重置 checksum、移除失败记录 | **高**--掩盖真实不一致 |
| `clean` | 清空所有表（含 schema_history） | **极高**--生产绝对禁用 |

### Spring Boot 集成

Spring Boot 自动检测 classpath 中的 Flyway，应用启动时自动执行 `migrate`。核心配置：

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true       # 非空库首次启用时建立基线
    baseline-version: 0             # 基线版本号，低于此的脚本不执行
    out-of-order: false             # 是否允许乱序补跑
    validate-on-migrate: true       # 迁移前校验 checksum
    table: flyway_schema_history    # 账本表名
```

多环境适配用 `{vendor}` 占位符：

```yaml
spring:
  flyway:
    locations: classpath:db/migration/{vendor}
    # 会匹配 db/migration/mysql、db/migration/postgresql 等
```

Maven 插件用于命令行操作：

```xml
<plugin>
  <groupId>org.flywaydb</groupId>
  <artifactId>flyway-maven-plugin</artifactId>
  <version>10.x</version>
  <configuration>
    <url>jdbc:mysql://localhost:3306/mydb</url>
    <user>root</user>
    <password>secret</password>
  </configuration>
</plugin>
```

```bash
mvn flyway:info    # 查看迁移状态
mvn flyway:migrate # 执行迁移
mvn flyway:validate # 校验
```

### SQL 迁移 vs Java 迁移

大多数场景用 SQL 迁移（`.sql` 文件）。但有些变更 SQL 做不了--比如数据回填需要复杂逻辑、调用外部 API、条件分支。Flyway 支持 Java 迁移：实现 `JdbcMigration` 接口，类名遵循 `V2__Backfill_user_data.java`，编译后在 classpath 中被扫描。

Java 迁移是逃生通道，不是日常手段。一旦用它，SQL 文件无法审查迁移逻辑，code review 成本飙升。

## 三、案例即原理

### 案例：版本冲突如何暴露

开发者 A 写了 `V3__create_audit_table.sql`，同时开发者 B 在另一分支也写了 `V3__add_audit_index.sql`。两人各自本地测试通过，合并到主干后部署：

```
flyway:validate
→ 检测到两个 V3
→ 报错：Found more than one migration with version 3
→ 阻塞部署
```

这不是 Flyway 的 bug，是它的**核心价值**：用版本号唯一性强制团队协调。Git 解决不了这个问题--两个文件名不同、内容不同的脚本可以共存于仓库，但 Flyway 拒绝执行，逼你把其中一个改成 V4。

### 案例：接入遗留库

一个运行了三年的生产库，schema 已经被手动改过无数次，没有迁移脚本。直接上 Flyway：

```
flyway:migrate
→ flyway_schema_history 不存在
→ 执行 V1__init.sql（试图建所有表）
→ 表已存在 → 报错，阻塞
```

解法是 `baseline`：对当前库打一个基线，告诉 Flyway"当前状态等于 V1，之前的脚本不要跑了"：

```yaml
spring:
  flyway:
    baseline-on-migrate: true
    baseline-version: 1    # V1 及以下视为已应用
```

这揭示了一个原理：**Flyway 的账本是"从此刻起的事实源"，不是"从头开始的历史"**。baseline 是一道分割线，之前的历史被冻结，之后的变更才受 Flyway 管控。这是务实的妥协--你不能要求一个跑了三年的库重新执行所有 DDL。

### 案例：乱序迁移

A 的 `V3` 因故晚于 B 的 `V4` 合入主干。V4 已经在生产跑过。默认情况下 Flyway 不会补跑 V3（它只跑版本号大于已应用最大版本的脚本）：

```
已应用：V1, V2, V4
待执行：V3（版本号 < V4）
默认行为：忽略 V3，不报错也不执行
```

开启 `out-of-order: true` 后，Flyway 会补跑 V3。但这里有个隐性风险：V4 如果依赖 V3 创建的表，补跑时 V4 已经成功，V3 却可能因为表已存在而失败。**乱序执行把顺序依赖的风险转嫁给了开发者**--你必须保证每个迁移脚本独立可执行、不依赖其他脚本的副作用。

## 四、缺陷与批判：为什么不行

### 不回滚是根本设计，不是缺陷

Flyway 没有 `rollback` 命令。迁移脚本一旦应用，只能往前加新脚本"修正"，不能撤销。这不是技术实现做不到，而是**刻意的哲学选择**：

- DDL 的事务边界不可控（MySQL 的 DDL 隐式提交，无法回滚）
- 回滚 DDL 比正向执行更危险（`ALTER TABLE ADD COLUMN` 能回滚成 `DROP COLUMN`，但列里的数据没了）
- 自动回滚制造假安全感，开发者以为"反正能撤"就不够谨慎

这个哲学在多数场景合理。但在 P0 故障场景--一次迁移导致生产表锁死、服务全挂--团队被迫手写一个新的 V 脚本来"逆向修复"，而这个修复脚本本身也需要测试、需要时间。**"禁止回滚"在故障面前的奢望**是 Flyway 最常被诟病的地方。Liquibase 的 `rollback` 标签虽然也救不了无事务的 DDL，但至少提供了 undo 标签的语义框架。

> forward-only 不是技术限制，是工程哲学。它把回滚的责任还给了人类，代价是故障时的恢复速度。

### checksum 地狱

已应用的脚本**一旦被修改**（哪怕只改一个注释、一个空格），下次 `validate` 就会失败：

```
Flyway Validate failed: Migration checksum mismatch for migration version 3
→ Applied to database: 1234567890
→ Resolved locally:    9876543210
```

这个机制是必要的--它防止"生产跑的脚本和仓库里的不一致"。但现实中，开发者经常因为"只是修个注释"而不解为什么 Flyway 炸了。解法是 `repair`：

```bash
mvn flyway:repair
```

`repair` 会用当前文件的 checksum 覆盖账本中的记录。但这是**逃生门也是隐患**--它把真实的不一致抹掉了，没人知道生产到底跑的是哪个版本的脚本。在严格审计场景，`repair` 应该被禁用或需要双人审批。

### MySQL DDL 无事务

PostgreSQL 的 DDL 支持事务（`CREATE TABLE` 可以回滚），Flyway 的迁移脚本在一个事务内执行，失败则整体回滚，账本干净。

MySQL 的 DDL 是隐式提交的，无法回滚。一个迁移脚本包含两条 DDL：

```sql
ALTER TABLE users ADD COLUMN status INT;  -- 成功，已提交
ALTER TABLE orders ADD COLUMN user_id INT; -- 失败
```

第二条失败时，第一条已经提交，无法回滚。Flyway 在账本中记录 `success=0`，下次 `migrate` 看到这个失败记录会直接阻塞：

```
Detected failed migration to version 3. Please remove any changes made by this migration.
```

开发者必须手动清理第一条 DDL 的副作用，再 `repair` 移除失败记录。**这是底层数据库能力的局限转嫁给了 Flyway**，而 MySQL 是最大的用户群体，所以这个问题几乎每个团队都会撞上。

### repeatable 的非确定性陷阱

`R__` 脚本在每次 V 脚本之后全量重跑。如果一个可重复迁移重建一个大视图或复杂的存储过程，每次 schema 变更都会触发重编译，迁移时间陡增。

更隐蔽的误用：有人把数据回填写成 `R__seed_data.sql`。每次 checksum 变（改了一行 INSERT），整个脚本重跑--但数据不是幂等的（INSERT 不带 `ON DUPLICATE KEY UPDATE` 会报主键冲突）。**R 脚本的"可重复"语义是针对 schema 对象（视图、存储过程），不是针对数据操作**。

### 不适合数据回填

Flyway 是 schema migration tool，不是 data migration tool。它的设计假设是"每个脚本快速、幂等、只碰结构"。但实际项目中，迁移脚本经常夹带大量数据操作：

```sql
-- V5__migrate_user_names.sql
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name);
-- 1000 万行，跑 10 分钟
```

这种脚本在迁移期间锁表，生产服务阻塞。Flyway 没有分批执行、断点续跑的机制--它就是一次性执行 SQL 文件。**大批量数据搬运应该走专用 ETL（如 DataX、自定义批处理脚本），不要塞进 Flyway 迁移脚本**。这不是 Flyway 的缺陷，而是工具适用边界的误判，但文档对此强调不够。

### 多环境漂移

开发环境跑到 V10，生产停在 V5，中间隔了 5 个未验证的迁移。开发觉得"没问题"，一到生产 `validate` 发现某个 V7 脚本被改过（开发时随手修了注释），checksum 不对，部署阻塞。

根本原因是**环境间的 schema 演进速度不同步**。Flyway 提供了 `ignoreMigrationPatterns` 来跳过某些校验，但这是治标--真正的问题在于团队没有把"生产 schema 状态"当作需要持续对齐的资产。Flyway 只能在发现不一致时报错，不能消除不一致。

### 与 Liquibase 的根本分野

| 维度 | Flyway | Liquibase |
|------|--------|-----------|
| 迁移格式 | 纯 SQL | XML/YAML/JSON changelog（也支持 SQL） |
| 回滚 | 不支持（forward-only） | 支持 undo 标签 |
| 抽象层级 | 命令式（写什么执行什么） | 声明式（描述目标状态，diff 生成） |
| 学习曲线 | 低（会写 SQL 就会用） | 中高（需学 changelog 语法） |
| 数据库无关性 | 低（SQL 绑定具体方言） | 高（changelog 可跨数据库翻译） |
| 透明度 | 高（所见即所执行） | 中（生成的 SQL 需 preview 查看） |

Flyway 的 SQL-first 是优势也是限制：团队完全掌控执行的 SQL，但换数据库引擎（MySQL -> PostgreSQL）时每个脚本都要重写。Liquibase 的声明式 changelog 能跨库翻译，但"翻译出的 SQL 和我预期一致吗"这个不确定性，是很多团队放弃它的原因。

## 五、总结：Flyway 到底是什么

Flyway 本质是**用一张账本表强制的、向前不可逆的 schema 变更契约**。

它真正的价值不在"执行 SQL"（一个 shell 脚本也能执行 SQL），而在于它强制团队接受三条纪律：

1. **schema 历史是不可变的**--已应用的脚本不能改，改了就 checksum 报错
2. **schema 变更是有序的**--版本号是全序，冲突即阻塞
3. **schema 状态是可审计的**--账本表记录了谁、何时、跑了什么、成功与否

forward-only 不是 Flyway 的技术短板，而是它的哲学立场。它认为：**自动回滚是危险的假象，DDL 出错时人类必须介入，手动写一个修正脚本比信任自动 undo 更安全**。这个选择牺牲了故障恢复速度，换来了对"每个变更都是深思熟虑"的强制约束。

适用边界很清晰：如果你的团队 schema 变更频率低、追求简单透明、接受"不回滚"的纪律，Flyway 是最低成本的方案。如果你需要频繁回滚、跨多种数据库引擎、或需要声明式的 schema diff 能力，Liquibase 更合适--代价是学习成本和"生成的 SQL 不完全可控"的不确定性。

Flyway 把 schema 从"运维操作"提升为"工程制品"。它不是最强的工具，但是约束最清晰、侵入性最低的那个。在工程领域，**清晰可预测的约束往往比强大的能力更有价值**--这正是 Flyway 流行的真正原因。
