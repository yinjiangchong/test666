# Redis 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师方向面试  
> 覆盖范围：原理深度 · 数据结构 · 持久化 · 高可用 · 分布式实战

---

## 目录

1. [基础原理与数据结构](#一基础原理与数据结构)
2. [持久化机制](#二持久化机制)
3. [过期与淘汰策略](#三过期与淘汰策略)
4. [主从复制与哨兵](#四主从复制与哨兵)
5. [Redis Cluster 集群](#五redis-cluster-集群)
6. [事务与 Lua 脚本](#六事务与-lua-脚本)
7. [常见应用场景](#七常见应用场景)
8. [性能调优与运维](#八性能调优与运维)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、基础原理与数据结构

### 1.1 Redis 为什么这么快？

```
1. 纯内存操作：数据全量在内存，读写延迟 < 100μs
2. 单线程模型（命令处理）：避免锁竞争，无上下文切换开销
   （Redis 6.0+ 网络 I/O 多线程，命令执行仍单线程）
3. I/O 多路复用：epoll/kqueue，单线程处理海量连接
4. 高效数据结构：SDS、跳表、压缩列表等专为内存优化
```

### 1.2 核心数据类型及底层实现

| 类型 | 常用命令 | 底层结构 | 适用场景 |
|---|---|---|---|
| **String** | SET/GET/INCR/SETNX | SDS（简单动态字符串） | 缓存、计数器、分布式锁 |
| **Hash** | HSET/HGET/HGETALL | ziplist（小）/ hashtable（大） | 对象存储、购物车 |
| **List** | LPUSH/RPOP/LRANGE | quicklist（ziplist 链表） | 消息队列、时间线 |
| **Set** | SADD/SMEMBERS/SINTER | intset（纯整数）/ hashtable | 标签、好友关系、去重 |
| **ZSet** | ZADD/ZRANGE/ZRANGEBYSCORE | ziplist（小）/ skiplist+hashtable | 排行榜、延迟队列 |
| **Stream** | XADD/XREAD/XGROUP | rax 树 | 消息队列（可持久化） |
| **Bitmap** | SETBIT/GETBIT/BITCOUNT | SDS | 签到、布隆过滤器 |
| **HyperLogLog** | PFADD/PFCOUNT | 特殊字符串 | UV 统计（误差 0.81%） |
| **Geo** | GEOADD/GEODIST/GEORADIUS | ZSet | 地理位置、附近的人 |

### 1.3 SDS（简单动态字符串）

```c
struct sdshdr {
    int len;     // 已用长度
    int free;    // 剩余空间
    char buf[];  // 字节数组
};
```

**优于 C 字符串的地方**：
- O(1) 获取长度（存储 len）
- 杜绝缓冲区溢出（追加前检查 free）
- 空间预分配 + 惰性释放，减少内存重分配
- 二进制安全（不以 `\0` 判断结尾）

### 1.4 跳表（Skip List）

ZSet 大数据量时的底层结构，支持 O(log N) 查找、范围查询。

```
Level 3:  1 ──────────────────── 9
Level 2:  1 ──── 4 ──────── 8 ── 9
Level 1:  1 ── 3 ─ 4 ── 6 ─ 8 ── 9
```

**为什么不用红黑树？**
- 范围查询更高效（链表遍历）
- 实现简单，易于调试
- 内存占用与红黑树相当

---

## 二、持久化机制

### 2.1 RDB（快照持久化）

**原理**：fork 子进程，将内存数据序列化为二进制文件（`.rdb`）

```
触发方式：
  手动：SAVE（阻塞）/ BGSAVE（后台 fork 子进程，推荐）
  自动：save 900 1     # 900s 内至少 1 次修改
        save 300 10    # 300s 内至少 10 次修改
        save 60 10000  # 60s 内至少 10000 次修改
```

| 优点 | 缺点 |
|---|---|
| 文件紧凑，恢复速度快 | 两次快照间的数据会丢失 |
| fork 后父进程不阻塞 | fork 大内存时 COW 开销大 |
| 适合灾备、全量备份 | 不适合对数据丢失敏感的场景 |

### 2.2 AOF（追加日志持久化）

**原理**：将每条写命令追加到 `.aof` 文件

```ini
appendonly yes
appendfsync everysec   # always（最安全）/ everysec（推荐）/ no（最快）
auto-aof-rewrite-percentage 100  # AOF 文件增长 100% 时触发重写
auto-aof-rewrite-min-size 64mb
```
| appendfsync | 说明 | 数据安全性 | 性能 |
|---|---|---|---|
| `always` | 每次写命令后 fsync | 最安全，最多丢 1 条 | 最低 |
| `everysec` | 每秒 fsync（默认推荐） | 最多丢 1s 数据 | 较好 |
| `no` | 由 OS 决定 fsync | 可能丢较多数据 | 最高 |

**AOF 重写**：BGREWRITEAOF，将内存数据重新生成最小命令集合，缩小 AOF 文件。

### 2.3 RDB + AOF 混合持久化（推荐）

Redis 4.0+ 支持，AOF 重写时将 RDB 快照写入 AOF 文件头部，后面追加增量 AOF 命令。

```ini
aof-use-rdb-preamble yes
```

**恢复速度快（RDB）+ 数据完整性好（AOF）**，生产环境首选。

### 2.4 持久化策略选型

| 场景 | 推荐方案 |
|---|---|
| 缓存，允许丢失 | 关闭持久化 |
| 可容忍分钟级数据丢失 | 仅 RDB |
| 要求秒级数据安全 | AOF（everysec）|
| 生产环境（推荐） | RDB + AOF 混合 |

---

## 三、过期与淘汰策略

### 3.1 Key 过期删除策略

| 策略 | 说明 | 特点 |
|---|---|---|
| **惰性删除** | 访问时检查是否过期，过期则删除 | 节省 CPU，内存不友好 |
| **定期删除** | 每隔 100ms 随机扫描部分 key，删除过期的 | 折中方案，Redis 默认 |

> Redis 同时使用**惰性删除 + 定期删除**，两者互补。

### 3.2 内存淘汰策略（maxmemory-policy）

| 策略 | 说明 |
|---|---|
| `noeviction` | 内存满时拒绝写入（默认） |
| `allkeys-lru` | 从全部 key 中淘汰最近最少使用的 |
| `volatile-lru` | 从设置了过期时间的 key 中淘汰 LRU |
| `allkeys-lfu` | 从全部 key 中淘汰最不频繁使用的（4.0+） |
| `volatile-lfu` | 从设置了过期时间的 key 中淘汰 LFU |
| `allkeys-random` | 随机淘汰全部 key |
| `volatile-random` | 随机淘汰有过期时间的 key |
| `volatile-ttl` | 淘汰剩余 TTL 最小的 key |

> **生产推荐**：缓存场景用 `allkeys-lru` 或 `allkeys-lfu`；有冷热数据区分用 `volatile-lru`。

### 3.3 LRU vs LFU

| 对比 | LRU | LFU |
|---|---|---|
| 淘汰依据 | 最近访问时间 | 访问频率 |
| 缺点 | 突发冷数据会污染热数据 | 新数据频率低容易被淘汰 |
| 适用 | 访问均匀的场景 | 热点数据明显的场景 |

---

## 四、主从复制与哨兵

### 4.1 主从复制原理

```
Slave 启动 → 发送 PSYNC → Master 执行 BGSAVE → 发送 RDB
                                                      ↓
                                          Slave 加载 RDB（全量同步）
                                                      ↓
                                          Master 发送 repl_backlog（增量）
                                                      ↓
                                          后续命令实时同步（命令传播）
```

**全量复制 vs 增量复制**：
- 首次连接或 offset 超出 `repl-backlog-size`：全量复制
- 断线重连且 offset 在 backlog 内：增量复制（PSYNC2，Redis 4.0+）

### 4.2 哨兵模式（Sentinel）

```
┌──────────┐    监控    ┌──────────┐
│ Sentinel │ ─────────► │  Master  │
│    1     │            └──────────┘
└──────────┘                 │ 同步
┌──────────┐    监控    ┌──────────┐
│ Sentinel │ ─────────► │  Slave   │
│    2     │            └──────────┘
└──────────┘
┌──────────┐    客观下线（多数哨兵同意）→ 故障转移
│ Sentinel │
│    3     │
└──────────┘
```

**故障转移流程**：
1. 哨兵 A 检测 Master **主观下线**（`is-master-down-by-addr`）
2. 超过 `quorum` 数量哨兵确认 → **客观下线**
3. 哨兵之间选举 **Leader**（Raft 算法）
4. Leader 从 Slave 中选新 Master（优先级 > 复制偏移量 > runid）
5. 通知其他 Slave 指向新 Master，通知客户端

### 4.3 主从复制常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| 复制延迟大 | 主库写入量大，网络延迟 | 升级带宽，减少大 key |
| 频繁全量同步 | backlog 太小，断线后 offset 超出 | 调大 `repl-backlog-size` |
| 脑裂 | 网络分区，主从都认为自己是 Master | `min-replicas-to-write` 限制 |

---
## 五、Redis Cluster 集群

### 5.1 数据分片原理

Redis Cluster 使用 **Hash Slot（哈希槽）** 分片，共 **16384** 个槽。

```
slot = CRC16(key) % 16384

示例（3 主节点）：
  节点 A：0    ~ 5460
  节点 B：5461 ~ 10922
  节点 C：10923 ~ 16383
```

**为什么是 16384 个槽？**
- 心跳包携带槽位图，16384 slots = 2KB，足够小
- 节点数不超过 1000 时，16384 个槽分配粒度合理

### 5.2 集群架构

```
Client
  │
  ├──► Node A (Master) ◄──► Node A' (Slave)
  ├──► Node B (Master) ◄──► Node B' (Slave)
  └──► Node C (Master) ◄──► Node C' (Slave)

每个主节点负责部分槽，主从之间异步复制
```

### 5.3 集群常见问题

| 问题 | 说明 |
|---|---|
| MOVED 重定向 | 客户端请求槽不在当前节点，返回正确节点地址，客户端重新请求 |
| ASK 重定向 | 槽迁移中，临时跳转到目标节点 |
| 跨槽操作 | 多 key 命令（MSET/MGET）要求 key 在同一槽，用 `{}` Hash Tag 解决 |
| 脑裂 | 网络分区时可能出现，通过 `cluster-require-full-coverage` 控制 |

```bash
# Hash Tag：强制多个 key 落在同一槽
MSET {user:1}:name "Alice" {user:1}:age 25
```

---

## 六、事务与 Lua 脚本

### 6.1 Redis 事务

```bash
MULTI          # 开启事务
SET k1 v1
SET k2 v2
INCR counter
EXEC           # 执行（原子提交）
# DISCARD    # 取消事务
```

**Redis 事务特点**：
- **不支持回滚**：命令执行错误不影响其他命令
- **原子性有限**：命令入队语法错误则全部不执行；运行时错误则其他命令继续
- `WATCH key`：乐观锁，被监视的 key 变更后 EXEC 返回 nil（事务失败）

### 6.2 Lua 脚本（推荐替代事务）

```python
# Python 示例
lua_script = """
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) > 0 then
    redis.call('DECR', KEYS[1])
    return 1
end
return 0
"""
result = redis_client.eval(lua_script, 1, 'stock:product:1')
```

**Lua 优于 MULTI/EXEC 的原因**：
- 真正的原子性（脚本执行期间不会被中断）
- 支持条件判断和逻辑控制
- 减少网络往返次数

---
## 七、常见应用场景

### 7.1 缓存三大问题

| 问题 | 描述 | 解决方案 |
|---|---|---|
| **缓存穿透** | 查询不存在的数据，缓存无效，每次打到 DB | 缓存空值 + 布隆过滤器 |
| **缓存击穿** | 热点 key 过期，大量并发直接打到 DB | 互斥锁重建 + 逻辑过期（不设 TTL） |
| **缓存雪崩** | 大量 key 同时过期或 Redis 宕机 | TTL 加随机抖动 + 集群高可用 + 限流降级 |

```python
# 缓存穿透：布隆过滤器（python 示例）
from redis import Redis
from redisbloom.client import Client

bf = Client()
bf.bfCreate('user_filter', 0.001, 1000000)  # 误判率 0.1%，容量 100 万
bf.bfAdd('user_filter', user_id)
if not bf.bfExists('user_filter', user_id):
    return None  # 直接拦截，不查 DB
```

### 7.2 分布式锁

```python
import redis
import uuid

client = redis.Redis()

def acquire_lock(lock_key: str, expire: int = 10) -> str | None:
    token = str(uuid.uuid4())
    # SET key value NX PX milliseconds：原子操作
    ok = client.set(lock_key, token, nx=True, px=expire * 1000)
    return token if ok else None

def release_lock(lock_key: str, token: str) -> bool:
    # Lua 保证原子性：判断 + 删除
    lua = """
    if redis.call('GET', KEYS[1]) == ARGV[1] then
        return redis.call('DEL', KEYS[1])
    else
        return 0
    end
    """
    return bool(client.eval(lua, 1, lock_key, token))
```

> **Redlock 算法**：向 N（≥5）个独立 Redis 节点请求加锁，超过半数成功且耗时 < 锁有效期，则认为加锁成功。

### 7.3 排行榜

```python
# ZSet 实现实时排行榜
redis_client.zadd('rank:game', {'player_1': 1500, 'player_2': 2300})
redis_client.zincrby('rank:game', 100, 'player_1')  # 加分

# 取 Top 10（从高到低）
top10 = redis_client.zrevrange('rank:game', 0, 9, withscores=True)
```

### 7.4 延迟队列

```python
import time

# 生产者：score 为执行时间戳
def push_delay_task(task_id: str, delay_seconds: int):
    execute_at = time.time() + delay_seconds
    redis_client.zadd('delay_queue', {task_id: execute_at})

# 消费者：轮询到期任务
def consume_delay_tasks():
    now = time.time()
    tasks = redis_client.zrangebyscore('delay_queue', 0, now)
    for task in tasks:
        # 原子抢占任务
        if redis_client.zrem('delay_queue', task):
            process(task)
```

### 7.5 Session 共享 / 限流

```python
# 滑动窗口限流（每用户每分钟最多 100 次）
def rate_limit(user_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f'rate:{user_id}'
    pipe = redis_client.pipeline()
    now = time.time()
    pipe.zremrangebyscore(key, 0, now - window)   # 清除过期记录
    pipe.zadd(key, {str(now): now})               # 记录本次请求
    pipe.zcard(key)                                # 统计窗口内请求数
    pipe.expire(key, window)
    results = pipe.execute()
    return results[2] <= limit
```
---

## 八、性能调优与运维

### 8.1 关键配置参数

```ini
# 内存
maxmemory 8gb
maxmemory-policy allkeys-lru

# 网络
tcp-backlog 511
timeout 300
tcp-keepalive 300

# 持久化
save ""                        # 关闭 RDB（纯缓存场景）
appendonly yes
appendfsync everysec

# 慢日志
slowlog-log-slower-than 10000  # 单位 μs，超过 10ms 记录
slowlog-max-len 128

# 懒删除（异步删除大 key，避免阻塞）
lazyfree-lazy-eviction yes
lazyfree-lazy-expire yes
lazyfree-lazy-server-del yes
```

### 8.2 大 Key 问题

**定义**：String > 10KB；Hash/List/Set/ZSet 元素数 > 5000

**危害**：
- 阻塞单线程命令执行（DEL 大 key 耗时长）
- 网络传输慢，带宽占用高
- 集群数据倾斜

**排查与解决**：

```bash
# 扫描大 key（非阻塞）
redis-cli --bigkeys

# 异步删除
UNLINK key           # 异步删除，替代 DEL

# Python：分批删除 Hash 大 key
cursor = 0
while True:
    cursor, fields = redis_client.hscan('big_hash', cursor, count=100)
    if fields:
        redis_client.hdel('big_hash', *fields.keys())
    if cursor == 0:
        break
```

### 8.3 热 Key 问题

**解决方案**：
1. 本地缓存（`cachetools`）+ Redis 二级缓存
2. 读写分离，热 key 读从节点
3. Key 分片：`hot_key:0`～`hot_key:N`，随机读取

### 8.4 常用监控命令

```bash
# 实时监控命令（生产慎用，影响性能）
redis-cli MONITOR

# 查看统计信息
redis-cli INFO all
redis-cli INFO memory     # 内存使用
redis-cli INFO replication # 主从状态

# 慢日志
redis-cli SLOWLOG GET 10

# 客户端连接
redis-cli CLIENT LIST
```

---

## 九、实战场景题

### 场景 1：如何用 Redis 实现消息队列？各方案对比？

| 方案 | 实现 | 优点 | 缺点 |
|---|---|---|---|
| List | LPUSH + BRPOP | 简单，天然 FIFO | 消息无法持久化确认，无消费组 |
| Pub/Sub | PUBLISH/SUBSCRIBE | 广播，实时性好 | 不持久化，消费者离线丢消息 |
| ZSet | ZADD + ZRANGEBYSCORE | 支持延迟队列 | 需轮询，消息无确认机制 |
| **Stream** | XADD + XREADGROUP | 持久化、消费组、ACK 确认 | 实现复杂，Redis 5.0+ |

```python
# Stream 消费组示例
redis_client.xgroup_create('stream:orders', 'order-group', id='0', mkstream=True)

# 消费
messages = redis_client.xreadgroup('order-group', 'consumer-1', {'stream:orders': '>'}, count=10)
for stream, msgs in messages:
    for msg_id, data in msgs:
        process(data)
        redis_client.xack('stream:orders', 'order-group', msg_id)
```

### 场景 2：Redis 与 DB 双写一致性如何保证？

```
策略对比：

1. Cache Aside（旁路缓存）[推荐]
   读：先读缓存 → 未命中读 DB → 写入缓存
   写：先更新 DB → 再删除缓存（不更新缓存）
   问题：更新 DB 后删缓存失败 → 引入消息队列重试 / Canal 监听 binlog

2. Write Through（同步写穿）
   写：同时写 DB 和缓存（强一致，性能差）

3. Write Behind（异步回写）
   写：只写缓存，异步批量刷 DB（高性能，可能丢数据）
```

### 场景 3：Redis 集群节点宕机，数据会丢失吗？

```
情况分析：
1. 主节点宕机，从节点未完成同步（异步复制）→ 可能丢失部分数据
2. 通过 min-replicas-to-write 和 min-replicas-max-lag 降低丢失窗口
3. 极端情况（脑裂）：旧主仍接受写入，切换后数据丢失
4. 对数据完整性要求极高 → 使用 Redis 同步复制（wait 命令）或换用 Kafka
```

---

## 十、高频追问与陷阱题

### Q1：Redis 单线程为什么还这么快？

> 1. 纯内存操作，绝大部分命令时间复杂度 O(1)
> 2. 单线程避免锁竞争和线程切换开销
> 3. I/O 多路复用（epoll）处理大量连接
> 4. 数据结构专门优化（如 SDS、跳表、压缩列表）  
> 单线程瓶颈在 CPU 计算，而 Redis 瓶颈通常在内存和网络，单线程足够。

---

### Q2：SETNX + EXPIRE 实现分布式锁有什么问题？

> 两条命令不是原子操作：SETNX 成功后进程崩溃，EXPIRE 未执行 → 死锁。  
> **正确做法**：`SET key value NX PX milliseconds`（单条原子命令）

---

### Q3：Redis 过期 key 是立即删除的吗？

> 不是立即删除。Redis 使用**惰性删除 + 定期删除**策略：  
> - 惰性：访问时发现过期才删
> - 定期：每 100ms 随机抽取部分有过期时间的 key 检查删除  
> 因此过期 key 在内存中可能短暂残留，`DBSIZE` 统计可能包含已过期 key。

---

### Q4：Redis 持久化会阻塞主进程吗？

> - **BGSAVE / BGREWRITEAOF**：fork 子进程执行，主进程不阻塞（fork 本身瞬间阻塞）
> - **SAVE**：同步执行，主进程阻塞，生产禁用
> - **AOF everysec**：后台线程每秒 fsync，主进程不阻塞
> - **COW（写时复制）**：fork 后父子进程共享内存页，写入时才复制，大内存 + 高写入时开销大

---

### Q5：Cluster 模式为什么最多 1000 个节点？

> 每个节点通过 Gossip 协议互相通信，节点数越多，心跳包数量 = O(N²)，  
> 网络开销和 CPU 消耗随节点数平方增长，官方建议不超过 1000 个节点。

---

### Q6：热 key 导致单节点 QPS 打满怎么办？

> 1. **本地缓存**（进程内 `functools.lru_cache` 或 `cachetools`）+ 短 TTL
> 2. **Key 打散**：`hot_key:{random 0-N}` 分散到多个槽
> 3. **读从节点**：Cluster 开启 `READONLY` 命令，从节点分担读流量
> 4. **Redis 6.0 多线程网络 I/O**：提升单节点吞吐

---

*文档版本：Redis 7.x 适用 ｜ 最后更新：2025 年*
