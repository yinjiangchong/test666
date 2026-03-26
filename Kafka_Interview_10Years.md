# Kafka 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、消息队列方向面试  
> 覆盖范围：核心原理 · 存储机制 · 高可用 · 消费者组 · 性能调优 · 实战场景

---

## 目录

1. [基础概念与核心架构](#一基础概念与核心架构)
2. [存储原理与日志机制](#二存储原理与日志机制)
3. [生产者（Producer）原理](#三生产者producer原理)
4. [消费者（Consumer）与消费者组](#四消费者consumer与消费者组)
5. [Broker 与副本机制](#五broker-与副本机制)
6. [事务与精确一次语义](#六事务与精确一次语义)
7. [性能调优](#七性能调优)
8. [与 RabbitMQ 对比](#八与-rabbitmq-对比)
9. [Python 实战（kafka-python / confluent-kafka）](#九python-实战)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、基础概念与核心架构

### 1.1 核心概念

| 概念 | 说明 |
|---|---|
| **Broker** | Kafka 服务节点，负责消息存储与转发 |
| **Topic** | 消息的逻辑分类（类似数据库的表） |
| **Partition** | Topic 的物理分片，有序、不可变的消息序列 |
| **Offset** | 消息在 Partition 中的唯一位置编号（单调递增） |
| **Producer** | 消息生产者，向 Topic 写入消息 |
| **Consumer** | 消息消费者，从 Topic 读取消息 |
| **Consumer Group** | 消费者组，同一组内每个 Partition 只被一个 Consumer 消费 |
| **ZooKeeper / KRaft** | 元数据管理（2.8+ 支持 KRaft 模式，无需 ZooKeeper） |
| **ISR** | In-Sync Replicas，与 Leader 保持同步的副本集合 |
| **LEO** | Log End Offset，副本末尾偏移量 |
| **HW** | High Watermark，消费者可见的最大偏移量 |

### 1.2 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        Kafka Cluster                          │
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐               │
│  │ Broker 1 │    │ Broker 2 │    │ Broker 3 │               │
│  │          │    │          │    │          │               │
│  │ Topic-A  │    │ Topic-A  │    │ Topic-A  │               │
│  │  P0(L)   │    │  P1(L)   │    │  P2(L)   │               │
│  │  P1(F)   │    │  P2(F)   │    │  P0(F)   │               │
│  └──────────┘    └──────────┘    └──────────┘               │
│                                                              │
│  L = Leader Partition    F = Follower Partition              │
└──────────────────────────────────────────────────────────────┘
         ▲                                    ▼
    Producer                            Consumer Group
    （写 Leader）                    （读 Leader，按 Partition 分配）
```

### 1.3 Topic 与 Partition

```
Topic: order-events
  ├── Partition 0: [msg0, msg1, msg2, msg3 ...]  offset: 0,1,2,3...
  ├── Partition 1: [msg0, msg1, msg2 ...]
  └── Partition 2: [msg0, msg1 ...]

特性：
  ✅ 同一 Partition 内消息有序
  ❌ 跨 Partition 不保证全局有序
  ✅ 同一 key 的消息路由到同一 Partition（保证局部有序）
```

---

## 二、存储原理与日志机制

### 2.1 日志文件结构

```
/kafka-logs/order-events-0/          ← Partition 目录
  ├── 00000000000000000000.log        ← 消息数据文件（segment）
  ├── 00000000000000000000.index      ← 稀疏索引（offset → 物理位置）
  ├── 00000000000000000000.timeindex  ← 时间索引（timestamp → offset）
  ├── 00000000000001000000.log        ← 下一个 segment（起始 offset=1000000）
  ├── 00000000000001000000.index
  └── leader-epoch-checkpoint
```

- **Segment**：每个 Partition 由多个 Segment 文件组成，默认 1GB 或 7 天滚动
- **稀疏索引**：不是每条消息都建索引，每隔 `index.interval.bytes`（默认 4KB）建一条
- **查找消息流程**：
  ```
  offset → 二分查找 .index 文件定位 segment → 顺序扫描 .log 文件找到消息
  ```

### 2.2 消息格式（Record Batch）

```
RecordBatch:
  baseOffset        ← 批次起始 offset
  batchLength
  magic             ← 版本号（v2 = Kafka 0.11+）
  attributes        ← 压缩类型、事务标记等
  lastOffsetDelta
  firstTimestamp
  maxTimestamp
  producerId        ← 幂等/事务支持
  producerEpoch
  baseSequence
  records[]:        ← 实际消息列表
    length
    attributes
    timestampDelta
    offsetDelta
    key
    value
    headers[]
```
### 2.3 日志清理策略

| 策略 | 配置 | 说明 |
|---|---|---|
| **delete**（默认） | `log.cleanup.policy=delete` | 超过保留时间/大小则删除旧 Segment |
| **compact** | `log.cleanup.policy=compact` | 相同 key 只保留最新消息（适合状态类 Topic） |
| **delete + compact** | 两者结合 | 先 compact 去重，再按时间删除 |

```
# 常用保留策略配置
log.retention.hours=168          # 保留 7 天（默认）
log.retention.bytes=1073741824   # 每个 Partition 最大 1GB
log.segment.bytes=1073741824     # 单个 Segment 文件大小
log.segment.ms=604800000         # Segment 最长存活时间
```

### 2.4 零拷贝（Zero-Copy）

```
传统文件传输（4次拷贝）：
  磁盘 → 内核缓冲区 → 用户空间 → Socket 缓冲区 → 网卡

Kafka Zero-Copy（sendfile，2次拷贝）：
  磁盘 → 内核缓冲区 → 网卡（跳过用户空间）

结论：Kafka 消费时直接通过 sendfile 发送，极大提升吞吐量
```

---

## 三、生产者（Producer）原理

### 3.1 消息发送流程

```
Producer
  │
  ├─ 序列化（Serializer）
  ├─ 分区路由（Partitioner）
  │     ├── 指定 key → hash(key) % num_partitions
  │     ├── 未指定 key → Sticky Partitioner（粘性分区，批量发送）
  │     └── 自定义 Partitioner
  ├─ 写入 RecordAccumulator（内存缓冲区，按 Partition 分 Deque<ProducerBatch>）
  └─ Sender 线程（后台）
        ├── 从 Accumulator 取 Batch
        ├── 请求合并（同一 Broker 的多个 Partition Batch 合并发送）
        └── 发送到 Broker Leader
```

### 3.2 重要生产者配置

| 参数 | 默认值 | 说明 |
|---|---|---|
| `acks` | `1` | `0`=不等待确认；`1`=Leader 确认；`-1`/`all`=ISR 全部确认 |
| `retries` | `2147483647` | 失败重试次数（配合幂等使用） |
| `retry.backoff.ms` | `100` | 重试间隔 |
| `linger.ms` | `0` | 等待时间（> 0 可提高批次填充率，提升吞吐） |
| `batch.size` | `16384`（16KB） | 批次大小（增大可提升吞吐） |
| `buffer.memory` | `33554432`（32MB） | 生产者缓冲区总大小 |
| `compression.type` | `none` | 压缩算法：`gzip`/`snappy`/`lz4`/`zstd` |
| `max.in.flight.requests.per.connection` | `5` | 未收到响应的最大请求数（幂等时需 ≤ 5） |
| `enable.idempotence` | `true`（3.0+） | 开启幂等，防止重试导致重复消息 |

### 3.3 acks 详解

```
acks=0：
  Producer 发出即认为成功，不等待 Broker 响应
  最高吞吐，数据可能丢失（网络故障/Broker 宕机）

acks=1：
  Leader 写入日志后响应
  Leader 宕机时，若消息未同步到 Follower，则丢失

acks=-1（all）：
  Leader + ISR 中所有副本都写入后才响应
  最强可靠性，结合 min.insync.replicas 使用

# 推荐生产配置（不丢消息）：
acks=all
min.insync.replicas=2      # Broker 端配置
enable.idempotence=true
retries=Integer.MAX_VALUE
```

### 3.4 幂等生产者（Idempotent Producer）

```
原理：
  每个 Producer 分配唯一 PID（Producer ID）
  每条消息携带 <PID, Partition, SequenceNumber>
  Broker 记录每个 <PID, Partition> 的最大 SequenceNumber
  重复消息（相同 SequenceNumber）→ Broker 直接丢弃

作用范围：
  ✅ 单个 Partition 内幂等（防止重试导致重复）
  ❌ 不跨 Partition，不跨会话（Producer 重启后 PID 变化）
  → 跨 Partition / 跨会话幂等需使用「事务」
```

---

## 四、消费者（Consumer）与消费者组

### 4.1 消费者组工作原理

```
Topic: order-events（3个 Partition）

Consumer Group A（3个消费者）：
  Consumer-1 → Partition 0
  Consumer-2 → Partition 1
  Consumer-3 → Partition 2

Consumer Group B（2个消费者）：
  Consumer-1 → Partition 0 + Partition 1
  Consumer-2 → Partition 2

规则：
  ✅ 同一 Group 内，每个 Partition 只能被一个 Consumer 消费
  ✅ 不同 Group 之间相互独立，各自维护自己的 Offset
  ⚠️ Consumer 数量 > Partition 数量时，多余的 Consumer 空闲
```
### 4.2 Rebalance 机制

**触发条件**：
- Consumer 加入或离开 Consumer Group
- Topic 的 Partition 数量变化
- Consumer 心跳超时（`session.timeout.ms`，默认 10s）

**Rebalance 过程**：
```
1. Group Coordinator（某个 Broker）检测到成员变化
2. 向所有 Consumer 发送 rebalance 通知
3. 所有 Consumer 停止消费，提交 Offset
4. Group Leader（第一个加入的 Consumer）执行分区分配
5. Coordinator 下发分配结果
6. 所有 Consumer 按新分配结果恢复消费

⚠️ Rebalance 期间整个 Consumer Group 停止消费（Stop-The-World）
```

**分区分配策略**：

| 策略 | 说明 | 适用场景 |
|---|---|---|
| `RangeAssignor` | 按范围分配（默认，可能不均衡） | 简单场景 |
| `RoundRobinAssignor` | 轮询分配（均衡） | 一般场景 |
| `StickyAssignor` | 尽量保持原分配，减少移动（Rebalance 影响小） | 推荐 |
| `CooperativeStickyAssignor` | 增量式 Rebalance（不停止全部消费） | 推荐（Kafka 2.4+） |

### 4.3 Offset 管理

```
Offset 存储位置（Kafka 0.9+）：
  内部 Topic：__consumer_offsets（50个 Partition）

提交方式：
  自动提交（enable.auto.commit=true，默认）：
    每隔 auto.commit.interval.ms（默认 5s）提交一次
    ⚠️ 风险：拉取后未处理完就提交 → 消费者宕机 → 消息丢失

  手动提交（推荐）：
    同步提交：consumer.commitSync()   → 阻塞，确保提交成功
    异步提交：consumer.commitAsync()  → 非阻塞，失败不重试
    推荐：处理完后 commitSync，或批量处理 + 定期 commitSync
```

### 4.4 消费者重要配置

| 参数 | 默认值 | 说明 |
|---|---|---|
| `group.id` | — | 消费者组 ID（必填） |
| `auto.offset.reset` | `latest` | 无初始 Offset 时：`earliest`（从头）/`latest`（从末尾）/`none`（报错） |
| `enable.auto.commit` | `true` | 是否自动提交 Offset |
| `max.poll.records` | `500` | 每次 poll 最多拉取条数 |
| `max.poll.interval.ms` | `300000`（5min） | 两次 poll 最大间隔，超时则认为 Consumer 死亡触发 Rebalance |
| `session.timeout.ms` | `10000`（10s） | 心跳超时时间 |
| `heartbeat.interval.ms` | `3000`（3s） | 心跳发送间隔（建议 session.timeout.ms 的 1/3） |
| `fetch.min.bytes` | `1` | 每次 fetch 最少返回字节数 |
| `fetch.max.wait.ms` | `500` | fetch 最大等待时间 |

### 4.5 消费语义

| 语义 | 说明 | 实现方式 |
|---|---|---|
| **At Most Once** | 最多一次，可能丢消息 | 拉取后立即提交 Offset，再处理 |
| **At Least Once** | 至少一次，可能重复消费 | 处理完后提交 Offset（默认推荐） |
| **Exactly Once** | 精确一次，不丢不重 | Kafka 事务 + 幂等消费（或业务幂等） |

---

## 五、Broker 与副本机制

### 5.1 副本（Replica）机制

```
Topic: order-events，3个 Partition，副本数=3

Partition 0：
  Leader  → Broker 1（负责读写）
  Follower → Broker 2（同步 Leader）
  Follower → Broker 3（同步 Leader）

ISR（In-Sync Replicas）：
  与 Leader 日志差距在 replica.lag.time.max.ms（默认 30s）内的副本集合
  ISR 收缩：Follower 落后太多 → 移出 ISR → 进入 OSR（Out-Sync Replicas）
  ISR 扩张：Follower 追上 Leader → 重新加入 ISR
```

### 5.2 LEO 与 HW

```
LEO（Log End Offset）：每个副本自己的最新 Offset + 1
HW（High Watermark）：ISR 中所有副本 LEO 的最小值

Consumer 只能消费 HW 之前的消息（保证消费到的消息已在所有 ISR 中存在）

Broker 1 (Leader): LEO=10, HW=8
Broker 2 (ISR):    LEO=9,  HW=8
Broker 3 (ISR):    LEO=8,  HW=8

→ 消费者最多消费到 offset=7
```

### 5.3 Leader 选举

```
触发条件：Leader 所在 Broker 宕机

选举策略（unclean.leader.election.enable）：
  false（默认，推荐）：只从 ISR 中选举 → 不丢数据，但若 ISR 为空则不可用
  true：允许 OSR 副本成为 Leader → 可用性高，但可能丢数据

选举过程（KRaft 模式）：
  Controller 检测到 Leader 宕机
  从 ISR 列表中选择第一个可用副本作为新 Leader
  更新 Partition 元数据，通知所有 Broker 和 Producer/Consumer
```

### 5.4 Broker 重要配置

```properties
# 副本同步
num.replica.fetchers=1                     # 副本拉取线程数
replica.lag.time.max.ms=30000              # ISR 最大滞后时间
replica.fetch.max.bytes=1048576            # 副本每次 fetch 最大字节

# 可靠性
default.replication.factor=3              # 默认副本数
min.insync.replicas=2                      # 最少同步副本数（与 acks=all 配合）
unclean.leader.election.enable=false       # 禁止 OSR 副本成为 Leader

# 性能
num.io.threads=8                           # I/O 线程数
num.network.threads=3                      # 网络线程数
log.flush.interval.messages=10000         # 强制 flush 前的消息数
```

---
## 六、事务与精确一次语义

### 6.1 事务 Producer

```python
# 事务配置
producer = KafkaProducer(
    transactional_id="my-transactional-id"  # 唯一事务 ID
)

producer.init_transactions()

try:
    producer.begin_transaction()
    producer.send("topic-a", key=b"k1", value=b"v1")
    producer.send("topic-b", key=b"k2", value=b"v2")
    # 同时提交 Consumer Offset（读-处理-写 原子性）
    producer.send_offsets_to_transaction(
        offsets={TopicPartition("input-topic", 0): OffsetAndMetadata(100, "")},
        group_id="my-consumer-group"
    )
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

### 6.2 Exactly Once 实现原理

```
端到端精确一次（EOS）需要：

1. 幂等 Producer（防止 Producer 重试导致重复）
   enable.idempotence=true

2. 事务（防止跨 Partition / 跨会话重复）
   transactional.id=唯一ID

3. 消费端隔离级别（防止读到未提交事务的消息）
   isolation.level=read_committed（默认 read_uncommitted）

适用场景：
  Kafka Streams 内部状态处理（天然支持 EOS）
  读-处理-写（consume-transform-produce）模式
```

---

## 七、性能调优

### 7.1 吞吐量优化

**Producer 端**：
```properties
# 增大批次，减少请求次数
batch.size=65536              # 64KB（默认 16KB）
linger.ms=20                  # 等待 20ms 积累更多消息
compression.type=lz4          # 压缩（lz4 速度最快，zstd 压缩率最高）
buffer.memory=67108864        # 64MB

# 增大请求大小
max.request.size=5242880      # 5MB
```

**Consumer 端**：
```properties
fetch.min.bytes=65536         # 64KB，减少 fetch 请求次数
fetch.max.wait.ms=500
max.poll.records=1000         # 每次拉取更多消息
```

**Broker 端**：
```properties
num.io.threads=16
num.network.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

### 7.2 延迟优化

```properties
# Producer
linger.ms=0                   # 不等待，立即发送
batch.size=1                  # 极端低延迟（吞吐量大幅下降）
acks=1                        # 减少等待副本确认时间

# Consumer
fetch.min.bytes=1             # 有消息即返回
fetch.max.wait.ms=0
```

### 7.3 Partition 数量与吞吐量

```
经验公式：
  Partition 数 = max(
    目标吞吐量 / 单 Partition Producer 吞吐量,
    目标吞吐量 / 单 Partition Consumer 吞吐量
  )

注意事项：
  ✅ Partition 越多，并行度越高，吞吐量越大
  ❌ Partition 过多的代价：
    - Leader 选举时间增长
    - 元数据增大，ZooKeeper/Controller 压力增加
    - 文件句柄增多（每个 Partition 若干文件）
    - Rebalance 时间增长

建议：
  - 初始设置适中（如 Topic 的消费 QPS / 单Consumer QPS）
  - 可以增加 Partition 数（不会重新分配现有消息）
  - 无法减少 Partition 数
```

### 7.4 消息积压处理

```
原因排查：
  ① Consumer 处理速度 < Producer 发送速度
  ② Consumer 频繁 Rebalance（心跳超时、处理超时）
  ③ Consumer 数量不足（少于 Partition 数）

处理方案：
  ① 扩容 Consumer 实例（增加消费者数量至 Partition 数）
  ② 优化消费逻辑（异步处理、批量写库）
  ③ 增加 Partition 数（需同步增加 Consumer）
  ④ 临时方案：新建 Topic（更多 Partition），将积压消息搬移过去
```

---
## 八、与 RabbitMQ 对比

| 维度 | Kafka | RabbitMQ |
|---|---|---|
| **设计目标** | 高吞吐日志流、事件流 | 灵活消息路由、任务队列 |
| **吞吐量** | 极高（百万级/s） | 中等（万~十万级/s） |
| **消息模型** | 发布订阅（Pull） | 队列 / 发布订阅（Push） |
| **消息保留** | 按时间/大小保留（可重复消费） | 消费后删除（默认） |
| **消息顺序** | Partition 内有序 | 单队列有序 |
| **消费方式** | Consumer 主动 Pull | Broker 主动 Push |
| **路由能力** | 简单（按 key hash） | 强大（Exchange: direct/topic/fanout/headers） |
| **事务** | 支持（Kafka 事务） | 支持（AMQP 事务 / Publisher Confirms） |
| **优先级队列** | 不支持 | 支持 |
| **死信队列** | 需手动实现 | 原生支持 DLX |
| **延迟消息** | 需插件或手动实现 | 支持（TTL + DLX / 插件） |
| **适用场景** | 日志收集、实时流处理、事件溯源、大数据管道 | 业务解耦、任务分发、RPC、延迟任务 |

---

## 九、Python 实战

### 9.1 使用 kafka-python

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.errors import KafkaError
import json

# ── Producer ──────────────────────────────────────────────
producer = KafkaProducer(
    bootstrap_servers=["localhost:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    key_serializer=lambda k: k.encode("utf-8") if k else None,
    acks="all",
    retries=3,
    enable_idempotence=True,
    compression_type="lz4",
    linger_ms=10,
    batch_size=32768,
)

def send_message(topic: str, key: str, value: dict):
    future = producer.send(topic, key=key, value=value)
    try:
        record_metadata = future.get(timeout=10)
        print(f"发送成功: topic={record_metadata.topic}, "
              f"partition={record_metadata.partition}, "
              f"offset={record_metadata.offset}")
    except KafkaError as e:
        print(f"发送失败: {e}")
        raise

# 批量发送
def batch_send(topic: str, messages: list[dict]):
    for msg in messages:
        producer.send(topic, key=msg.get("id"), value=msg)
    producer.flush()  # 确保所有消息发送完毕


# ── Consumer ──────────────────────────────────────────────
consumer = KafkaConsumer(
    "order-events",
    bootstrap_servers=["localhost:9092"],
    group_id="order-processor",
    auto_offset_reset="earliest",
    enable_auto_commit=False,           # 手动提交
    value_deserializer=lambda v: json.loads(v.decode("utf-8")),
    max_poll_records=100,
    session_timeout_ms=10000,
    heartbeat_interval_ms=3000,
)

def consume_messages():
    try:
        for message in consumer:
            try:
                process(message.value)
                consumer.commit()       # 处理成功后手动提交
            except Exception as e:
                print(f"处理失败: {e}")
                # 根据业务决定：跳过 / 重试 / 发送死信
    finally:
        consumer.close()

def process(msg: dict):
    print(f"处理消息: {msg}")
```

### 9.2 使用 confluent-kafka（高性能，推荐生产使用）

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json

# ── Producer ──────────────────────────────────────────────
producer = Producer({
    "bootstrap.servers": "localhost:9092",
    "acks": "all",
    "enable.idempotence": True,
    "compression.type": "lz4",
    "linger.ms": 10,
    "batch.size": 65536,
})

def delivery_report(err, msg):
    if err is not None:
        print(f"发送失败: {err}")
    else:
        print(f"发送成功: {msg.topic()} [{msg.partition()}] @{msg.offset()}")

def send(topic: str, key: str, value: dict):
    producer.produce(
        topic,
        key=key.encode("utf-8"),
        value=json.dumps(value).encode("utf-8"),
        callback=delivery_report
    )
    producer.poll(0)  # 触发回调

producer.flush()  # 等待所有消息发送完毕


# ── Consumer ──────────────────────────────────────────────
consumer = Consumer({
    "bootstrap.servers": "localhost:9092",
    "group.id": "order-processor",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False,
})

consumer.subscribe(["order-events"])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            if msg.error().code() == KafkaError._PARTITION_EOF:
                continue
            print(f"Consumer 错误: {msg.error()}")
            break

        value = json.loads(msg.value().decode("utf-8"))
        try:
            process(value)
            consumer.commit(asynchronous=False)
        except Exception as e:
            print(f"处理失败: {e}")
finally:
    consumer.close()
```

### 9.3 AdminClient（Topic 管理）

```python
from confluent_kafka.admin import AdminClient, NewTopic

admin = AdminClient({"bootstrap.servers": "localhost:9092"})

# 创建 Topic
new_topics = [
    NewTopic("order-events", num_partitions=6, replication_factor=3)
]
result = admin.create_topics(new_topics)
for topic, future in result.items():
    try:
        future.result()
        print(f"Topic {topic} 创建成功")
    except Exception as e:
        print(f"Topic {topic} 创建失败: {e}")

# 查看 Topic 元数据
metadata = admin.list_topics(timeout=10)
for topic in metadata.topics.values():
    print(f"Topic: {topic.topic}, Partitions: {len(topic.partitions)}")
```

---
## 十、高频追问与陷阱题

### Q1：Kafka 如何保证消息不丢失？

> **三端配置缺一不可**：
>
> **Producer 端**：
> - `acks=all`：等待 ISR 全部确认
> - `enable.idempotence=true`：防止重试导致重复（幂等）
> - `retries` 足够大：确保临时故障后重试
>
> **Broker 端**：
> - `replication.factor >= 3`：至少 3 副本
> - `min.insync.replicas=2`：ISR 中至少 2 个副本
> - `unclean.leader.election.enable=false`：禁止非 ISR 副本成为 Leader
>
> **Consumer 端**：
> - `enable.auto.commit=false`：手动提交 Offset
> - 处理完成后再 commitSync

---

### Q2：Kafka 消费者如何实现幂等消费？

> Kafka 只能保证 **At Least Once**（acks=all + 手动提交），消费端可能重复。  
> 业务层实现幂等的方案：
>
> 1. **唯一键去重**：消息携带 `message_id`，消费前查 Redis/DB 判断是否已处理
> 2. **乐观锁**：数据库使用版本号，重复处理时 update 影响行数为 0
> 3. **状态机**：业务状态只能单向流转，重复消费同一状态无副作用
> 4. **数据库唯一索引**：直接 INSERT，重复时捕获唯一键冲突异常

---

### Q3：Kafka 为什么吞吐量这么高？

> 1. **顺序写磁盘**：Kafka 日志文件追加写，顺序 I/O 比随机 I/O 快 100 倍以上
> 2. **零拷贝（sendfile）**：消费时跳过用户空间，减少 2 次数据拷贝
> 3. **批量压缩发送**：Producer 批量积累消息，压缩后发送
> 4. **Page Cache**：利用操作系统 Page Cache，写磁盘实际上先写内存
> 5. **Partition 并行**：多 Partition 多线程并行读写
> 6. **Pull 模式**：Consumer 主动拉取，Broker 无需管理推送速率

---

### Q4：消息积压了怎么办？

> 排查 + 处理步骤：
> 1. **查监控**：Consumer Lag（消费延迟）急速增大
> 2. **定位瓶颈**：Consumer 处理慢？还是 Consumer 数量不足？
> 3. **扩容 Consumer**：增加消费者实例至 Partition 数量上限
> 4. **优化消费逻辑**：并发处理、批量写库
> 5. **临时扩分区**：增加 Partition + Consumer（注意 key 路由变化）
> 6. **降级方案**：若积压严重，可临时跳过历史数据（修改 Offset）后恢复

---

### Q5：Kafka 和 ZooKeeper 的关系？KRaft 是什么？

> **历史上 ZooKeeper 的作用**：
> - 存储 Broker 元数据、Topic 配置、Partition 状态
> - Controller 选举（哪个 Broker 是 Controller）
> - Consumer Offset 存储（旧版本，0.9+ 已迁移到 `__consumer_offsets`）
>
> **问题**：ZooKeeper 成为性能瓶颈，运维复杂度高
>
> **KRaft 模式（Kafka 2.8+ 预览，3.3+ 生产可用）**：
> - 使用 Raft 共识算法替代 ZooKeeper
> - Kafka 节点内置 Controller（部分节点）
> - 简化部署，支持更多 Partition（百万级）
> - Kafka 4.0 将完全移除 ZooKeeper

---

### Q6：Rebalance 期间会发生什么？如何减少影响？

> **Rebalance 期间**：整个 Consumer Group 停止消费（Stop-The-World），影响实时性。
>
> **减少 Rebalance 的方法**：
> 1. **增大 `session.timeout.ms`**：减少因网络抖动误判 Consumer 死亡
> 2. **增大 `max.poll.interval.ms`**：处理耗时任务时不触发超时
> 3. **使用 `CooperativeStickyAssignor`**：增量式 Rebalance，只迁移必要的 Partition
> 4. **静态组成员（`group.instance.id`）**：Consumer 重启不触发 Rebalance（Kafka 2.3+）

---

### Q7：Kafka 消息顺序如何保证？

> **Partition 内有序，跨 Partition 无序**
>
> 保证顺序的方案：
> 1. **同 key 路由到同 Partition**：相同 key（如订单 ID）的消息路由到同一 Partition
> 2. **单 Partition**：极端情况下只用 1 个 Partition（牺牲并行度）
> 3. **Producer 端**：`max.in.flight.requests.per.connection=1`（禁止乱序，但配合幂等使用时可为 5）
>
> **⚠️ 注意**：Consumer 端多线程并发处理会打乱顺序，需要在消费端串行处理同一 key 的消息

---

## 附录：面试前必看清单

- [ ] 能描述 Kafka 的整体架构（Broker / Topic / Partition / Consumer Group）
- [ ] 理解 ISR / HW / LEO 的含义及其关系
- [ ] 熟悉 acks 三种模式及适用场景
- [ ] 能解释幂等 Producer 的实现原理（PID + SequenceNumber）
- [ ] 理解 Rebalance 的触发条件和过程，以及如何减少影响
- [ ] 掌握三种消费语义（At Most Once / At Least Once / Exactly Once）
- [ ] 了解 Kafka 高吞吐的原因（顺序写 / 零拷贝 / 批量 / Page Cache）
- [ ] 能说出消息积压的排查和处理方法
- [ ] 了解 KRaft 模式及其与 ZooKeeper 的区别
- [ ] 能区分 Kafka 与 RabbitMQ 的适用场景

---

*文档版本：Kafka 3.x 适用 ｜ 最后更新：2025 年*
