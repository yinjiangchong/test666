# MySQL 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、DBA 方向面试  
> 覆盖范围：原理深度 · 性能调优 · 高可用架构 · 分布式实战

---

## 目录

1. [基础原理与存储引擎](#一基础原理与存储引擎)
2. [索引原理与优化](#二索引原理与优化)
3. [事务与锁机制](#三事务与锁机制)
4. [SQL 执行计划与查询优化](#四sql-执行计划与查询优化)
5. [日志系统](#五日志系统)
6. [主从复制与高可用](#六主从复制与高可用)
7. [分库分表与分布式](#七分库分表与分布式)
8. [连接池与运维调优](#八连接池与运维调优)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、基础原理与存储引擎

### 1.1 InnoDB vs MyISAM 核心差异

| 对比项 | InnoDB | MyISAM |
|---|---|---|
| 事务支持 | ✅ 支持 ACID | ❌ 不支持 |
| 行级锁 | ✅ 行锁 + 意向锁 | ❌ 表锁 |
| 外键约束 | ✅ 支持 | ❌ 不支持 |
| 崩溃恢复 | ✅ redo log 自动恢复 | ❌ 需修复 |
| 全文索引 | ✅ 5.6+ 支持 | ✅ 支持 |
| 存储结构 | 聚簇索引（B+Tree） | 非聚簇索引 |
| count(*) | 需全表扫描 | 直接读取元数据 |
| 适用场景 | 高并发读写、事务场景 | 只读/分析型场景 |

### 1.2 InnoDB 整体架构

```
┌─────────────────────────────────────────────────┐
│                  MySQL Server                   │
│  连接层 → 解析器 → 优化器 → 执行器              │
└───────────────────┬─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│               InnoDB 存储引擎                    │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │            Buffer Pool（缓冲池）          │  │
│  │  数据页 / 索引页 / undo页 / change buffer │  │
│  └──────────────────────────────────────────┘  │
│  ┌───────────┐  ┌──────────┐  ┌─────────────┐  │
│  │ redo log  │  │ undo log │  │ binlog(服务)│  │
│  │ (WAL机制) │  │(MVCC/回滚)│  │(主从/恢复) │  │
│  └───────────┘  └──────────┘  └─────────────┘  │
└─────────────────────────────────────────────────┘
```

### 1.3 Buffer Pool 详解

- **作用**：缓存磁盘数据页，减少 I/O，默认大小 `128MB`，生产建议设为物理内存的 **60%~80%**
- **组成**：
  - `Free List`：空闲页链表
  - `LRU List`：冷热数据分区，5/8 为热区，防止全表扫描污染
  - `Flush List`：脏页链表，后台定期刷盘
- **Change Buffer**：针对非唯一二级索引的写缓存，合并后批量写入，减少随机 I/O

```sql
-- 查看 Buffer Pool 状态
SHOW ENGINE INNODB STATUS\G
-- 查看命中率（建议 > 99%）
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_%';
```

> **面试追问**：Buffer Pool 命中率低怎么办？  
> → 增大 `innodb_buffer_pool_size`；检查是否存在大查询污染 LRU；开启 `innodb_buffer_pool_dump_at_shutdown` 预热。

---

## 二、索引原理与优化

### 2.1 B+Tree 索引结构

```
                    [30 | 60]           ← 根节点（只存 key）
                   /    |    \
          [10|20]    [40|50]    [70|80]  ← 非叶节点
         /  |  \      ...        ...
[1-9] [10-19] [20-29] ...               ← 叶节点（存 key+data，双向链表）
```

**为什么选 B+Tree 而非 B-Tree、红黑树？**

| 结构 | 问题 |
|---|---|
| 红黑树 | 树高 O(log n)，数据量大时 I/O 次数多 |
| B-Tree | 非叶节点存数据，单页存储 key 数量少，树更高 |
| B+Tree | 非叶节点只存 key，矮胖结构；叶节点链表支持范围查询 |
| Hash 索引 | 不支持范围/排序查询，无法利用前缀匹配 |

### 2.2 聚簇索引 vs 非聚簇索引

- **聚簇索引**：叶节点存储完整行数据，InnoDB 默认以主键建立，物理存储顺序与索引顺序一致
- **非聚簇索引（二级索引）**：叶节点存储主键值，查询非覆盖列时需**回表**
- **覆盖索引**：查询列全部在索引中，无需回表，`EXPLAIN` 中 `Extra: Using index`

```sql
-- 覆盖索引示例：idx(age, name)
SELECT age, name FROM user WHERE age > 18;          -- ✅ 覆盖索引，不回表
SELECT age, name, phone FROM user WHERE age > 18;   -- ❌ 需回表
```

### 2.3 索引失效场景（必背）

```sql
-- ❌ 1. 最左前缀原则被破坏（联合索引 idx(a,b,c)）
SELECT * FROM t WHERE b = 1 AND c = 2;  -- a 未用，索引失效

-- ❌ 2. 索引列参与运算或函数
SELECT * FROM t WHERE YEAR(create_time) = 2024;
SELECT * FROM t WHERE id + 1 = 10;

-- ❌ 3. 隐式类型转换（字段 varchar，传入 int）
SELECT * FROM t WHERE phone = 13812345678;  -- phone 是 varchar

-- ❌ 4. 前导模糊查询
SELECT * FROM t WHERE name LIKE '%张%';   -- 失效
SELECT * FROM t WHERE name LIKE '张%';    -- ✅ 有效

-- ❌ 5. NOT IN / NOT EXISTS / != / <>
-- ❌ 6. OR 条件一侧无索引
SELECT * FROM t WHERE id = 1 OR name = '张三';  -- name 无索引则全表

-- ❌ 7. is null / is not null（视数据分布，优化器可能放弃）
```

### 2.4 索引设计原则

1. **选择性高**的列建索引（`cardinality / total_rows > 10%`）
2. **频繁作为 WHERE、JOIN、ORDER BY、GROUP BY** 的列
3. 联合索引遵循**最左前缀**，将区分度高的列放左边
4. 控制单表索引数量（建议 **≤ 5个**），写性能随索引数增加而下降
5. 长字符串使用**前缀索引**：`ALTER TABLE t ADD INDEX idx_email(email(20))`
6. 避免冗余索引：`(a)` 和 `(a,b)` 共存时，单列 `(a)` 多余

---

## 三、事务与锁机制

### 3.1 事务四大特性（ACID）

| 特性 | 说明 | 实现机制 |
|---|---|---|
| 原子性 Atomicity | 全成功或全失败 | undo log 回滚 |
| 一致性 Consistency | 事务前后数据合法 | 其他三者共同保障 |
| 隔离性 Isolation | 并发事务互不干扰 | MVCC + 锁 |
| 持久性 Durability | 提交后永久保存 | redo log + fsync |

### 3.2 事务隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
|---|---|---|---|---|
| READ UNCOMMITTED | ✅ | ✅ | ✅ | 读未提交 |
| READ COMMITTED | ❌ | ✅ | ✅ | 读已提交（Oracle 默认） |
| **REPEATABLE READ** | ❌ | ❌ | ⚠️ | **MySQL InnoDB 默认** |
| SERIALIZABLE | ❌ | ❌ | ❌ | 串行化，性能最差 |

> **InnoDB RR 级别如何解决幻读？**  
> 快照读：通过 **MVCC** 避免幻读（读旧版本快照）  
> 当前读：通过 **Gap Lock（间隙锁）+ Next-Key Lock** 锁定范围，防止插入

### 3.3 MVCC 实现原理

每行数据有两个隐藏列：
- `DB_TRX_ID`：最近修改的事务 ID
- `DB_ROLL_PTR`：指向 undo log 的回滚指针

**ReadView（读视图）** 快照读时生成，包含：
- `m_ids`：当前活跃（未提交）事务 ID 列表
- `min_trx_id`：活跃事务最小 ID
- `max_trx_id`：下一个将分配的事务 ID
- `creator_trx_id`：创建该 ReadView 的事务 ID

**可见性判断规则**：

```
if trx_id == creator_trx_id → 可见（自己修改的）
if trx_id <  min_trx_id     → 可见（已提交的旧版本）
if trx_id >= max_trx_id     → 不可见（未来事务）
if trx_id in m_ids          → 不可见（活跃未提交）
else                        → 可见
```

**RC vs RR 的区别**：
- RC：每次 SELECT 生成新的 ReadView → 能读到别人已提交的数据
- RR：事务开始第一次 SELECT 时生成 ReadView，全程复用 → 可重复读
### 3.4 锁的分类

```
MySQL 锁
├── 按粒度
│   ├── 表锁（MyISAM / InnoDB DDL）
│   ├── 行锁（InnoDB DML）
│   └── 页锁（BDB，已废弃）
│
├── 按性质
│   ├── 共享锁 S Lock（读锁）：SELECT ... LOCK IN SHARE MODE
│   └── 排他锁 X Lock（写锁）：SELECT ... FOR UPDATE / INSERT / UPDATE / DELETE
│
└── InnoDB 行锁类型
    ├── Record Lock：锁定单行记录
    ├── Gap Lock：锁定索引间隙（防幻读）
    ├── Next-Key Lock：Record Lock + Gap Lock（默认）
    └── Insert Intention Lock：插入意向锁（Gap Lock 的一种）
```

### 3.5 死锁

**产生条件**：互持对方需要的锁，并互相等待

**死锁示例**：

```sql
-- 事务 A
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 1;  -- 锁住 id=1
UPDATE account SET balance = balance + 100 WHERE id = 2;  -- 等待 id=2

-- 事务 B（同时执行）
BEGIN;
UPDATE account SET balance = balance - 100 WHERE id = 2;  -- 锁住 id=2
UPDATE account SET balance = balance + 100 WHERE id = 1;  -- 等待 id=1 → 死锁
```

**解决策略**：
1. 统一加锁顺序（始终按 id 升序操作）
2. 减小事务粒度，缩短事务持有锁时间
3. `innodb_lock_wait_timeout`（默认 50s）+ 死锁检测自动回滚代价小的事务
4. `SHOW ENGINE INNODB STATUS\G` 查看最近死锁信息

---

## 四、SQL 执行计划与查询优化

### 4.1 EXPLAIN 字段详解

```sql
EXPLAIN SELECT * FROM `order` o JOIN user u ON o.user_id = u.id WHERE u.age > 18;
```

| 字段 | 说明 | 重点值 |
|---|---|---|
| **type** | 访问类型 | `system > const > eq_ref > ref > range > index > ALL` |
| **key** | 实际使用的索引 | NULL 表示未使用索引 |
| **rows** | 预估扫描行数 | 越小越好 |
| **Extra** | 附加信息 | `Using index`（覆盖）/ `Using filesort`（文件排序）/ `Using temporary`（临时表） |
| possible_keys | 可能使用的索引 | — |
| key_len | 索引使用字节数 | 越长说明利用越充分 |
| ref | 与索引比较的列 | — |
| filtered | 按条件过滤后的行比例 | — |

**type 级别说明**：

| type | 说明 |
|---|---|
| `const` | 通过主键/唯一索引匹配单行，最优 |
| `eq_ref` | JOIN 时被驱动表用主键/唯一索引，一对一 |
| `ref` | 非唯一索引等值查询 |
| `range` | 索引范围扫描（`>`、`<`、`BETWEEN`、`IN`） |
| `index` | 全索引扫描（比 ALL 稍好，避免回表） |
| `ALL` | 全表扫描，需优化 |

### 4.2 慢查询分析流程

```
1. 开启慢查询日志
   slow_query_log      = ON
   slow_query_log_file = /var/log/mysql/slow.log
   long_query_time     = 1   （秒，建议生产设 0.5）

2. 用 mysqldumpslow / pt-query-digest 分析慢日志
   mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

3. EXPLAIN 分析执行计划

4. 针对性优化：
   - 添加/修改索引
   - 改写 SQL（子查询改 JOIN、避免 SELECT *）
   - 拆分大查询
   - 分页优化
```

### 4.3 常见 SQL 优化技巧

```sql
-- 1. 深分页优化（OFFSET 大时极慢）
-- ❌ 慢
SELECT * FROM `order` ORDER BY id LIMIT 1000000, 10;
-- ✅ 延迟关联
SELECT o.* FROM `order` o
JOIN (SELECT id FROM `order` ORDER BY id LIMIT 1000000, 10) t ON o.id = t.id;

-- 2. JOIN 优化：小表驱动大表，确保被驱动表的 JOIN 字段有索引

-- 3. 避免 SELECT *：减少网络传输，防止覆盖索引失效

-- 4. IN 子查询改 EXISTS（子查询结果集大时）
-- ❌
SELECT * FROM user WHERE id IN (SELECT user_id FROM `order` WHERE amount > 100);
-- ✅
SELECT * FROM user u WHERE EXISTS (
    SELECT 1 FROM `order` o WHERE o.user_id = u.id AND o.amount > 100
);

-- 5. OR 改 UNION ALL（各条件有独立索引时）
-- ❌
SELECT * FROM t WHERE a = 1 OR b = 2;
-- ✅
SELECT * FROM t WHERE a = 1
UNION ALL
SELECT * FROM t WHERE b = 2 AND a != 1;
```

---

## 五、日志系统

### 5.1 三大核心日志对比

| | redo log | undo log | binlog |
|---|---|---|---|
| 所属层 | InnoDB 引擎层 | InnoDB 引擎层 | MySQL Server 层 |
| 作用 | 崩溃恢复（持久性） | 事务回滚 + MVCC | 主从复制 + 数据恢复 |
| 日志类型 | 物理日志（页修改） | 逻辑日志（反向操作） | 逻辑日志（SQL/行变更） |
| 写入时机 | 事务执行中实时写 | 事务执行中实时写 | 事务提交时写 |
| 大小限制 | 固定大小循环写 | 按需分配 | 可追加，归档 |

### 5.2 两阶段提交（2PC）保证一致性

```
事务提交流程：
  ① 执行 SQL，写 undo log
  ② 写 redo log（prepare 状态）
  ③ 写 binlog
  ④ 提交 redo log（commit 状态）

崩溃恢复逻辑：
  redo log prepare + binlog 完整   → 提交事务
  redo log prepare + binlog 不完整 → 回滚事务
  redo log commit                  → 直接提交
```

> **为什么需要两阶段提交？**  
> 若先写 redo log 再写 binlog，崩溃后 binlog 缺失，主从数据不一致。  
> 两阶段提交以 binlog 是否完整作为事务是否提交的最终依据。
### 5.3 redo log WAL 机制

- **WAL（Write-Ahead Logging）**：先写日志（顺序 I/O），再写磁盘数据页（随机 I/O），大幅提升写性能
- `innodb_flush_log_at_trx_commit` 三档配置：

| 值 | 行为 | 安全性 | 性能 |
|---|---|---|---|
| `0` | 每秒刷盘 | 宕机丢 1s 数据 | 最好 |
| `1` | 每次提交刷盘（默认） | 最安全 | 较低 |
| `2` | 提交写 OS 缓存，每秒刷盘 | 折中 | 折中 |

### 5.4 binlog 三种格式

| 格式 | 说明 | 优缺点 |
|---|---|---|
| STATEMENT | 记录 SQL 语句 | 日志小，但含函数（NOW/RAND）时主从不一致 |
| ROW | 记录行变更（before/after） | 精确，日志大，**生产推荐** |
| MIXED | 自动选择 | 折中方案 |

---

## 六、主从复制与高可用

### 6.1 主从复制原理

```
Master                              Slave
  │                                   │
  ├─ 写操作 → binlog ─────────────►  IO Thread → relay log
  │                                   │
  │                                   ├─ SQL Thread 回放 relay log
  │                                   │
  │                                   └─ 数据与 Master 同步
```

**三个关键线程**：

| 线程 | 位置 | 作用 |
|---|---|---|
| binlog dump thread | Master | 响应 Slave 请求，发送 binlog |
| IO thread | Slave | 接收 binlog 写入 relay log |
| SQL thread | Slave | 回放 relay log，应用到本地数据 |

### 6.2 主从延迟原因与解决

**原因**：
1. 主库并发写入，从库单线程回放（MySQL 5.6 前）
2. 大事务（如 ALTER TABLE、大批量 DELETE）
3. 从库机器性能差
4. 网络延迟

**解决方案**：

| 方案 | 说明 |
|---|---|
| 并行复制 | MySQL 5.7 `slave_parallel_type = LOGICAL_CLOCK` |
| 拆分大事务 | 批量操作分批提交 |
| 升级硬件 | 从库使用 SSD |
| 半同步复制 | `rpl_semi_sync_master_enabled = ON`，至少一个从库 ACK 后才返回客户端 |

### 6.3 高可用架构方案

| 方案 | 说明 | 特点 |
|---|---|---|
| **MHA** | Master High Availability | 故障检测 30s，自动切换，业界成熟 |
| **MGR** | MySQL Group Replication | 官方组复制，强一致，支持多主 |
| **Orchestrator** | 拓扑管理 + 自动切换 | 可视化，GitHub 开源 |
| **PXC** | Percona XtraDB Cluster | 同步复制，多主，适合强一致场景 |
| **读写分离** | ProxySQL / MyCat | 减轻主库压力 |

---

## 七、分库分表与分布式

### 7.1 何时分库分表

| 指标 | 建议动作 |
|---|---|
| 单表数据量 > 500 万～1000 万 | 考虑分表 |
| 单库 QPS > 1 万 | 考虑分库 |
| 单库存储 > 200 GB | 考虑分库 |

### 7.2 分片策略

| 策略 | 说明 | 优点 | 缺点 |
|---|---|---|---|
| **Range 分片** | 按 ID/时间范围 | 范围查询友好 | 数据热点，分布不均 |
| **Hash 分片** | `id % N` | 分布均匀 | 范围查询需全片 |
| **一致性哈希** | 虚拟节点环 | 扩容影响小 | 实现复杂 |
| **地理/业务分片** | 按地区/租户 | 业务隔离好 | 跨片查询难 |

### 7.3 分库分表带来的问题与解决

**① 分布式 ID**

```
❌ 自增主键：各库独立自增，ID 冲突
✅ 推荐方案：
   - Snowflake（雪花算法）：64bit = 时间戳(41) + 机器ID(10) + 序列号(12)
   - 号段模式（美团 Leaf）：批量申请 ID 段，减少数据库请求
   - Redis INCR：简单，但依赖 Redis 可用性
   - UUID：全局唯一，但无序导致 B+Tree 页分裂，不推荐作主键
```

**② 跨库 JOIN**

```
解决方案：
  A. 冗余字段：将常用字段冗余到关联表，避免 JOIN
  B. 全局表（字典表）：在每个分库冗余一份小型基础数据表
  C. 应用层聚合：分别查询各库，在业务层合并
  D. OLAP 方案：同步到数据仓库（ClickHouse/Hive）做分析查询
```

**③ 分布式事务**

| 方案 | 原理 | 适用场景 |
|---|---|---|
| 2PC（XA） | 强一致，协调者 prepare + commit | 低并发，强一致 |
| TCC | Try-Confirm-Cancel 业务补偿 | 高并发，最终一致 |
| Saga | 长事务拆分，补偿回滚 | 长流程业务 |
| 本地消息表 | 事务 + 消息队列，最终一致 | 异步场景 |
| Seata AT | 自动生成回滚 SQL | 业务侵入少 |

**④ 分页查询**

```
方案：将 ORDER BY + LIMIT 下推各分片，各取 Top N，在应用层归并排序
缺点：OFFSET 越大代价越高
优化：禁止深分页，改为"下一页"模式（记录上次最大 ID）
```

---
## 八、连接池与运维调优

### 8.1 关键参数调优

```ini
# 连接相关
max_connections        = 1000   # 最大连接数，按内存和业务量调整
wait_timeout           = 600    # 空闲连接超时（秒）
interactive_timeout    = 600

# InnoDB 核心
innodb_buffer_pool_size      = 12G  # 建议物理内存 60%~80%
innodb_buffer_pool_instances = 8    # 并发访问减少锁竞争
innodb_log_file_size         = 1G   # redo log 大小，越大恢复越慢，写性能更好
innodb_flush_log_at_trx_commit = 1
innodb_io_capacity           = 2000 # SSD 可设 10000~20000
innodb_read_io_threads       = 8
innodb_write_io_threads      = 8

# 查询缓存（MySQL 8.0 已移除）
query_cache_type = 0  # 8.0 前建议关闭，并发下锁竞争严重

# 排序 / 临时表
sort_buffer_size       = 4M
join_buffer_size       = 4M
tmp_table_size         = 256M
max_heap_table_size    = 256M
```

### 8.2 连接池配置建议（以 HikariCP 为例）

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20       # CPU 核数 × 2 + 磁盘数，经验值
      minimum-idle: 5
      connection-timeout: 3000    # 获取连接超时（ms）
      idle-timeout: 600000        # 空闲连接存活时间（ms）
      max-lifetime: 1800000       # 连接最大存活时间（需 < wait_timeout × 1000）
      keepalive-time: 60000       # 防止连接被防火墙断开
```

### 8.3 常用监控 SQL

```sql
-- 查看当前连接列表
SHOW PROCESSLIST;

-- 查看 InnoDB 锁等待
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看各表数据量与索引大小
SELECT table_name,
       ROUND(data_length  / 1024 / 1024, 2) AS data_MB,
       ROUND(index_length / 1024 / 1024, 2) AS index_MB
FROM information_schema.tables
WHERE table_schema = 'your_db'
ORDER BY data_length DESC;

-- 查看未使用的索引
SELECT * FROM sys.schema_unused_indexes;

-- 查看冗余索引
SELECT * FROM sys.schema_redundant_indexes;

-- 查看慢查询 Top 10
SELECT * FROM sys.statement_analysis ORDER BY total_latency DESC LIMIT 10;
```

---

## 九、实战场景题

### 场景 1：亿级订单表如何设计？

```
1. 分库分表
   按 user_id Hash 分 16 库 × 32 表（共 512 张）

2. 分布式 ID
   Snowflake 算法，订单 ID 编码时间 + 机器 + 序列

3. 冷热分离
   3 个月以上订单归档到历史库（可用 TiDB / OSS）

4. 索引设计
   主键：order_id（Snowflake）
   idx(user_id, create_time)   → 用户订单列表
   idx(status, create_time)    → 运营查询未支付订单

5. 禁止跨库 JOIN
   展示订单时商品信息冗余存储在订单表

6. 读写分离
   查询走从库，下单走主库
```
### 场景 2：如何处理超卖问题？

```sql
-- 方案 1：数据库悲观锁
BEGIN;
SELECT stock FROM product WHERE id = 1 FOR UPDATE;
-- 业务判断 stock > 0
UPDATE product SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- 方案 2：乐观锁（CAS）
UPDATE product
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = #{version} AND stock > 0;
-- affected_rows = 0 时重试或返回失败

-- 方案 3：Redis 原子操作预扣减（推荐高并发场景）
-- DECR stock:product:1
-- 成功后异步落库
```

### 场景 3：大表 DDL 如何不锁表？

```bash
# MySQL 5.6+ Online DDL（支持 INPLACE 时）
ALTER TABLE big_table
  ADD COLUMN remark VARCHAR(255),
  ALGORITHM = INPLACE,
  LOCK = NONE;

# 不支持 INPLACE 时，使用 pt-online-schema-change
pt-online-schema-change \
  --alter "ADD COLUMN remark VARCHAR(255)" \
  --execute D=your_db,t=big_table
```

**pt-osc 原理**：
1. 创建与原表结构相同的影子表（`_new`）
2. 在原表加触发器（INSERT / UPDATE / DELETE 同步到影子表）
3. 分批拷贝原表数据到影子表
4. 原子重命名：`big_table → big_table_old`，`_new → big_table`
5. 删除旧表和触发器

### 场景 4：MySQL 突然变慢排查思路

```
Step 1  检查系统资源
        top / iostat -x 1 / vmstat 1
        → CPU 飙高？I/O util 100%？内存不足？

Step 2  查看 MySQL 当前状态
        SHOW PROCESSLIST;            -- 是否大量 Locked / Waiting
        SHOW ENGINE INNODB STATUS\G  -- 锁信息、死锁、Buffer Pool

Step 3  分析慢查询日志
        pt-query-digest /var/log/mysql/slow.log | head -100

Step 4  检查是否有 DDL 阻塞
        SELECT * FROM information_schema.INNODB_TRX;

Step 5  检查连接数
        SHOW GLOBAL STATUS LIKE 'Threads_%';

Step 6  检查 Buffer Pool 命中率
        SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';

Step 7  排查网络 / DNS 解析
        在 my.cnf 中加入 skip-name-resolve = ON
```

---

## 十、高频追问与陷阱题

### Q1：MySQL 默认隔离级别是什么？为什么不用 RC？

> MySQL InnoDB 默认 **RR（可重复读）**。  
> 历史原因：MySQL 5.0 之前 binlog 只有 STATEMENT 格式，RC 下 binlog 无法保证主从一致性（RC 需要 ROW 格式 binlog）。  
> 现代应用使用 ROW 格式 binlog，RC 性能更好（锁范围小，减少间隙锁导致的死锁），很多互联网公司已改用 RC。

---

### Q2：幻读在 InnoDB RR 下完全解决了吗？

> **没有完全解决**。  
> - 快照读（普通 SELECT）：通过 MVCC 解决  
> - 当前读（`SELECT FOR UPDATE`、`UPDATE`）：同一事务内第一次快照读没有，执行 `FOR UPDATE` 时却发现新行 → 幻读依然存在  
> Next-Key Lock 只在当前读加锁，无法保护两次读之间的间隙。

---

### Q3：count(\*) / count(1) / count(id) 哪个快？

> InnoDB 下性能排序：`count(*)` ≈ `count(1)` > `count(主键)` > `count(非索引列)`
>
> | 写法 | 说明 |
> |---|---|
> | `count(*)` | 优化器自动选最小索引全扫，不读行数据，最快 |
> | `count(1)` | 同上，性能等同 `count(*)` |
> | `count(主键)` | 遍历主键索引，需读取主键值，略慢 |
> | `count(普通列)` | 全表扫描 + 排除 NULL，最慢 |
>
> **注意**：MyISAM 的 `count(*)` 直接读元数据，O(1) 复杂度。

---

### Q4：为什么推荐使用自增主键？

> 1. **写性能**：自增 ID 顺序插入，B+Tree 叶节点从右追加，页分裂少
> 2. **存储效率**：UUID 36 字节 vs BIGINT 8 字节，UUID 索引空间大 3 倍以上
> 3. **二级索引**：叶节点存主键，主键越小，所有二级索引也越小
> 4. **UUID 缺点**：随机写导致大量页分裂和碎片，写性能下降 30%～50%

---

### Q5：一条 UPDATE 语句的完整执行流程？

```
① 客户端发送 SQL → 连接器（鉴权、权限校验）
② 解析器（词法 + 语法解析）→ 生成 AST
③ 优化器（选择索引、确定 JOIN 顺序）
④ 执行器 → 调用 InnoDB 存储引擎接口
⑤ InnoDB 层：
   a. 查 Buffer Pool，未命中则从磁盘加载数据页
   b. 写 undo log（记录旧值，支持回滚 + MVCC）
   c. 修改内存中的数据页（标记为脏页）
   d. 写 redo log（prepare 状态）
⑥ Server 层：写 binlog
⑦ InnoDB：redo log 标记 commit（两阶段提交完成）
⑧ 返回客户端
```

---

### Q6：索引下推（ICP）是什么？

> **Index Condition Pushdown**（MySQL 5.6+）
>
> | | 说明 |
> |---|---|
> | 无 ICP | 存储引擎按索引查到主键 → 全部回表 → Server 层过滤 WHERE 条件 |
> | 有 ICP | WHERE 条件中能用索引判断的部分**下推到存储引擎**，只对满足条件的行回表 |
>
> **示例**：`idx(name, age)`，`WHERE name LIKE '张%' AND age = 25`  
> - 无 ICP：找到所有 `name LIKE '张%'` 的行，全部回表，再过滤 `age = 25`  
> - 有 ICP：在索引中同时判断 `name LIKE '张%' AND age = 25`，只对满足的行回表，回表次数大幅减少

---

### Q7：Online DDL 期间是否完全不锁表？

> **不是完全不锁**，分三阶段：
>
> | 阶段 | MDL 锁类型 | 时长 | 影响 |
> |---|---|---|---|
> | 初始化 | MDL 写锁 | 毫秒级 | 短暂阻塞所有请求 |
> | 拷贝数据 | MDL 读锁 | 分钟～小时 | 允许 DML，row log 记录变更 |
> | 应用 row log + 重命名 | MDL 写锁 | 毫秒～秒级 | DML 短暂等待 |
>
> ⚠️ **高危风险**：若前置存在慢查询持有 MDL 读锁，DDL 的 MDL 写锁请求将阻塞，导致后续所有查询堆积，引发雪崩！  
> **建议**：DDL 前先用 `SHOW PROCESSLIST` 确认无长事务。

---

## 附录：面试前必看清单

- [ ] 能手画 B+Tree 索引结构并解释查询过程
- [ ] 能解释 MVCC ReadView 可见性判断的完整规则
- [ ] 能描述一条 UPDATE SQL 的完整链路（含 redo / undo / binlog）
- [ ] 熟悉 EXPLAIN 各字段含义，能根据执行计划给出优化方案
- [ ] 能设计亿级数据的分库分表方案并讨论各种取舍
- [ ] 了解主从复制延迟产生原因和对应解决方案
- [ ] 能写出死锁的产生场景和预防措施
- [ ] 理解 RR 隔离级别下幻读的残留问题
- [ ] 能解释 Online DDL 的锁阶段与风险点

---

*文档版本：MySQL 8.0 适用 ｜ 最后更新：2025 年*
