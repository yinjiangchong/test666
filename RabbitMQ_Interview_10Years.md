# RabbitMQ 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、消息中间件方向面试  
> 覆盖范围：核心原理 · 消息可靠性 · 高可用集群 · 性能调优 · 实战场景

---

## 目录

1. [基础概念与核心架构](#一基础概念与核心架构)
2. [Exchange 交换机详解](#二exchange-交换机详解)
3. [消息可靠性保障](#三消息可靠性保障)
4. [消息顺序性与幂等性](#四消息顺序性与幂等性)
5. [死信队列与延迟队列](#五死信队列与延迟队列)
6. [高可用集群架构](#六高可用集群架构)
7. [性能调优与最佳实践](#七性能调优与最佳实践)
8. [Python 实战（Pika / aio-pika）](#八python-实战pika--aio-pika)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、基础概念与核心架构

### 1.1 RabbitMQ 核心组件

| 组件 | 说明 |
|---|---|
| **Producer** | 消息生产者，发送消息到 Exchange |
| **Exchange** | 交换机，按路由规则将消息分发到 Queue |
| **Queue** | 消息队列，存储待消费的消息 |
| **Consumer** | 消息消费者，从 Queue 拉取/推送消息 |
| **Binding** | Exchange 与 Queue 之间的绑定关系 + Routing Key |
| **Channel** | 信道，TCP 连接上的轻量虚拟连接，复用连接 |
| **Connection** | 与 Broker 的 TCP 物理连接 |
| **VHost** | 虚拟主机，资源隔离单元（类似数据库的 schema） |
| **Broker** | RabbitMQ 服务实例 |

### 1.2 整体架构与消息流转

```
┌──────────────────────────────────────────────────────────────┐
│                        RabbitMQ Broker                       │
│                                                              │
│   Producer                                    Consumer       │
│      │                                            ▲          │
│      │ publish(msg, routing_key)                  │ consume  │
│      ▼                                            │          │
│  ┌──────────┐    binding     ┌─────────────┐      │          │
│  │ Exchange │ ─────────────► │    Queue    │ ─────┘          │
│  │ (fanout/ │  routing_key   │  (durable/  │                 │
│  │  direct/ │                │  transient) │                 │
│  │  topic/  │                └─────────────┘                 │
│  │ headers) │                                                │
│  └──────────┘                                                │
│                                                              │
│  Connection → Channel → Exchange/Queue（逻辑复用）           │
└──────────────────────────────────────────────────────────────┘
```

### 1.3 AMQP 协议核心概念

- **AMQP 0-9-1**：RabbitMQ 默认协议，二进制、面向连接、支持事务
- **Channel 多路复用**：一个 TCP 连接创建多个 Channel，避免频繁建立 TCP 连接的开销
- **Frame 类型**：Method Frame（命令）/ Header Frame（属性）/ Body Frame（消息体）/ Heartbeat Frame

```
TCP Connection
    ├── Channel 1  →  Queue A 生产
    ├── Channel 2  →  Queue B 消费
    └── Channel 3  →  管理操作
```

> **面试追问**：为什么不直接每次操作都建新连接？  
> → TCP 三次握手 + TLS 握手成本极高，Channel 是轻量级虚拟信道，创建/销毁代价极小。

---

## 二、Exchange 交换机详解

### 2.1 四种 Exchange 类型

| 类型 | 路由规则 | 典型场景 |
|---|---|---|
| **direct** | Routing Key 完全匹配 | 点对点，任务分发 |
| **fanout** | 忽略 Routing Key，广播到所有绑定队列 | 日志广播、事件通知 |
| **topic** | Routing Key 通配符匹配（`*` 一个词，`#` 零或多个词） | 多维度订阅 |
| **headers** | 基于消息 headers 属性匹配，忽略 Routing Key | 复杂条件路由 |

### 2.2 Topic Exchange 通配符规则

```
Routing Key 示例：order.created.cn

绑定 Pattern          是否匹配
order.created.cn   →  ✅ 完全匹配
order.*.cn         →  ✅ * 匹配单个词 created
order.#            →  ✅ # 匹配 created.cn（零或多个词）
#.cn               →  ✅ # 匹配 order.created
order.created.*    →  ✅ * 匹配 cn
*.created.cn       →  ✅ * 匹配 order
order.created      →  ❌ 缺少 .cn
```

### 2.3 默认 Exchange

- 名称为空字符串 `""`，类型为 direct
- 每个 Queue 自动绑定到默认 Exchange，Routing Key = Queue 名
- 生产者可直接发送到队列名，无需显式声明 Exchange

```python
# 直接发到队列（使用默认 Exchange）
channel.basic_publish(
    exchange='',
    routing_key='my_queue',
    body='Hello'
)
```

---
## 三、消息可靠性保障

> 可靠性需从三个维度保障：**生产者 → Broker → 消费者**

### 3.1 生产者可靠性：Confirm 机制

```
生产者                    Broker
   │── publish(msg) ──────►│
   │◄── ack/nack ──────────│  （异步确认）
```

**两种模式**：

| 模式 | 说明 | 性能 |
|---|---|---|
| 单条确认 | 发一条等一条 ack | 最低 |
| 批量确认 | 发 N 条后等批量 ack | 中等 |
| **异步确认（推荐）** | 回调函数处理 ack/nack | 最高 |

```python
import pika

def on_confirm(frame):
    print(f"Confirmed: {frame.method.delivery_tag}")

channel.confirm_delivery()
channel.add_on_return_callback(on_confirm)
```

### 3.2 生产者可靠性：Mandatory + Return 机制

- `mandatory=True`：消息无法路由到任何队列时，Broker 将消息**退回**给生产者
- 若 `mandatory=False`（默认），无法路由的消息直接**丢弃**

```python
channel.basic_publish(
    exchange='my_exchange',
    routing_key='non_existent',
    body='msg',
    mandatory=True,          # 无法路由时退回
    properties=pika.BasicProperties(delivery_mode=2)  # 持久化
)
```

### 3.3 Broker 可靠性：持久化

**三层持久化，缺一不可**：

```python
# 1. Exchange 持久化
channel.exchange_declare(exchange='my_ex', durable=True)

# 2. Queue 持久化
channel.queue_declare(queue='my_queue', durable=True)

# 3. 消息持久化（delivery_mode=2）
props = pika.BasicProperties(delivery_mode=pika.spec.PERSISTENT_DELIVERY_MODE)
channel.basic_publish(exchange='my_ex', routing_key='key', body='msg', properties=props)
```

> ⚠️ **注意**：持久化不等于 100% 不丢失，消息写入磁盘前 Broker 宕机仍会丢失。  
> 需配合**镜像队列/仲裁队列**实现真正高可用持久化。

### 3.4 消费者可靠性：ACK 机制

| ACK 模式 | 说明 | 风险 |
|---|---|---|
| `auto_ack=True` | 消息投递即确认 | 消费失败仍丢消息 |
| **`auto_ack=False`（推荐）** | 手动 ack/nack | 需业务代码显式确认 |

```python
def callback(ch, method, properties, body):
    try:
        # 处理消息
        process(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)       # 确认消费
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag,
                      requeue=True)                           # 重新入队

channel.basic_consume(queue='my_queue', on_message_callback=callback, auto_ack=False)
```

**nack 的 requeue 参数**：
- `requeue=True`：消息重新入队（注意避免无限循环）
- `requeue=False`：消息丢弃或进入死信队列

### 3.5 消息不丢失完整方案总结

```
生产者                    Broker                    消费者
   │                         │                         │
   ├─ Confirm 机制 ──────────►│                         │
   ├─ Mandatory 退回 ─────────│                         │
   │                         ├─ Exchange 持久化         │
   │                         ├─ Queue 持久化            │
   │                         ├─ 消息 delivery_mode=2   │
   │                         │─────────────────────────►│
   │                         │           手动 ACK ◄─────┤
   │                         │           失败 NACK ◄────┤
   │                         │           → 死信队列     │
```

---

## 四、消息顺序性与幂等性

### 4.1 顺序消费保障

**问题根源**：一个队列多个消费者并发消费，消息处理顺序无法保证

**方案一：单消费者**
```
Queue → Consumer（单实例）  # 吞吐量受限
```

**方案二：Sharding（分区队列）**
```
按业务 Key（如 order_id）Hash 到不同队列
Queue-0 → Consumer-0   （order_id 尾数 0,3,6）
Queue-1 → Consumer-1   （order_id 尾数 1,4,7）
Queue-2 → Consumer-2   （order_id 尾数 2,5,8,9）
```

**方案三：业务层排序**
- 消息携带序列号，消费端缓存乱序消息，按序号顺序处理
### 4.2 幂等性处理

```python
import redis

r = redis.Redis()

def callback(ch, method, properties, body):
    msg_id = properties.message_id          # 消息唯一ID（生产者生成）

    # 幂等检查：已消费则直接 ACK
    if r.setnx(f"consumed:{msg_id}", 1):
        r.expire(f"consumed:{msg_id}", 86400)
        try:
            process(body)
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception:
            r.delete(f"consumed:{msg_id}")  # 回滚标记
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
    else:
        # 已消费，直接 ACK 跳过
        ch.basic_ack(delivery_tag=method.delivery_tag)
```

---

## 五、死信队列与延迟队列

### 5.1 死信（Dead Letter）产生条件

消息成为死信的三种情况：
1. 消息被 `basic_nack` / `basic_reject` 且 `requeue=False`
2. 消息在队列中存活时间超过 **TTL**（Time To Live）
3. 队列达到最大长度（`x-max-length`），队首消息被挤出

### 5.2 死信队列配置

```python
# 1. 声明死信交换机和死信队列
channel.exchange_declare(exchange='dlx_exchange', exchange_type='direct')
channel.queue_declare(queue='dlx_queue', durable=True)
channel.queue_bind(queue='dlx_queue', exchange='dlx_exchange', routing_key='dlx_key')

# 2. 声明业务队列，绑定死信交换机
args = {
    'x-dead-letter-exchange': 'dlx_exchange',   # 死信转发到此 Exchange
    'x-dead-letter-routing-key': 'dlx_key',     # 死信路由 Key
    'x-message-ttl': 30000,                     # 队列消息 TTL（ms）
    'x-max-length': 1000,                       # 队列最大长度
}
channel.queue_declare(queue='business_queue', durable=True, arguments=args)
```

### 5.3 延迟队列实现方案

**方案一：TTL + 死信队列（原生支持，无需插件）**

```
Producer → [延迟队列（无消费者，设 TTL）] → TTL 到期 → DLX → [实际处理队列] → Consumer
```

```python
# 延迟队列（不挂 consumer，靠 TTL 驱动）
channel.queue_declare(
    queue='delay_30s',
    arguments={
        'x-message-ttl': 30000,                   # 延迟 30 秒
        'x-dead-letter-exchange': 'real_exchange', # 到期后转发
        'x-dead-letter-routing-key': 'real_key',
    }
)
```

⚠️ **缺陷**：每种延迟时长需独立队列；队列头部消息未过期会阻塞后面消息（即使后面消息 TTL 更短）。

**方案二：rabbitmq-delayed-message-exchange 插件（推荐）**

```python
# 声明延迟 Exchange（需安装插件）
channel.exchange_declare(
    exchange='delayed_exchange',
    exchange_type='x-delayed-message',
    arguments={'x-delayed-type': 'direct'}
)

# 发送时指定延迟时间（ms）
channel.basic_publish(
    exchange='delayed_exchange',
    routing_key='my_key',
    body='delayed message',
    properties=pika.BasicProperties(
        headers={'x-delay': 60000}   # 延迟 60 秒
    )
)
```

---
## 六、高可用集群架构

### 6.1 三种集群模式对比

| 模式 | 数据同步 | 可用性 | 一致性 | 适用场景 |
|---|---|---|---|---|
| **普通集群** | 仅元数据同步，消息只在一个节点 | 低（节点宕机消息丢失） | 弱 | 开发/测试 |
| **镜像队列** | 消息同步到所有镜像节点 | 高 | 强 | 生产（旧方案） |
| **仲裁队列（Quorum Queue）** | Raft 协议多数派写入 | 高 | 强 | **生产推荐（3.8+）** |

### 6.2 仲裁队列（Quorum Queue）

- 基于 **Raft 一致性协议**，多数派（`(N/2)+1`）确认后才算写入成功
- 自动故障转移，Leader 宕机后从 Follower 选举新 Leader
- **推荐节点数**：3 或 5（奇数，防止脑裂）

```python
# 声明仲裁队列
channel.queue_declare(
    queue='quorum_queue',
    durable=True,
    arguments={'x-queue-type': 'quorum'}
)
```

### 6.3 集群架构图

```
                    ┌─────────────────────┐
    Producer ──────►│    Load Balancer     │◄────── Consumer
                    │  (HAProxy/Nginx)     │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │  Node 1  │    │  Node 2  │    │  Node 3  │
        │ (Leader) │◄──►│(Follower)│◄──►│(Follower)│
        │  Raft    │    │  Raft    │    │  Raft    │
        └──────────┘    └──────────┘    └──────────┘
              ▲                ▲                ▲
              └────────────────┴────────────────┘
                         Raft 同步
```

### 6.4 镜像队列配置（旧版兼容）

```bash
# 通过 Policy 设置镜像（所有以 ha. 开头的队列同步到所有节点）
rabbitmqctl set_policy ha-all "^ha\." \
  '{"ha-mode":"all","ha-sync-mode":"automatic"}'

# 同步到指定数量节点
rabbitmqctl set_policy ha-two "^two\." \
  '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

---

## 七、性能调优与最佳实践

### 7.1 消费者 prefetch（预取）调优

```python
# prefetch_count：消费者最多同时持有未 ACK 的消息数
# 设为 1：最公平但吞吐量低
# 设为合适值（建议 10~50）：提升并发处理能力
channel.basic_qos(prefetch_count=20)
```

| prefetch_count | 效果 |
|---|---|
| `1` | 完全公平分发，吞吐量最低 |
| `0` | 不限制，消费者一次取走所有消息，负载不均衡 |
| `10~50` | 推荐，平衡吞吐和公平性 |

### 7.2 连接与 Channel 最佳实践

```python
# ✅ 推荐：一个连接，多个 Channel（生产/消费分离）
connection = pika.BlockingConnection(params)
producer_channel = connection.channel()   # 专用生产 Channel
consumer_channel = connection.channel()   # 专用消费 Channel

# ❌ 避免：每次发消息都新建连接（高延迟 + 连接数爆炸）
# ❌ 避免：多线程共享同一个 Channel（Channel 非线程安全）
```
### 7.3 关键参数调优

```ini
# rabbitmq.conf

# 内存水位线（超过后触发 flow control，停止接收新消息）
vm_memory_high_watermark.relative = 0.4   # 40% 内存

# 磁盘空间预警（低于此值触发告警）
disk_free_limit.absolute = 2GB

# 每个连接最大 Channel 数
channel_max = 2047

# 心跳间隔（秒），0 表示关闭，建议 60
heartbeat = 60

# 消费者取消通知
consumer_timeout = 900000   # 15 分钟无 ACK 则断开
```

### 7.4 消息积压处理

```
消息积压原因：
  1. 消费者处理速度 < 生产速度
  2. 消费者宕机或异常
  3. 消费逻辑阻塞（如依赖慢速第三方接口）

处理方案：
  ① 临时扩容消费者实例数（水平扩展）
  ② 新建临时队列，将积压消息转移后并行消费
  ③ 检查消费者是否有死锁/慢处理，修复后重启
  ④ 生产侧限流（令牌桶/漏桶），避免持续积压
  ⑤ 紧急情况：批量丢弃过期消息（业务允许时）
```

---

## 八、Python 实战（Pika / aio-pika）

### 8.1 同步生产者（Pika）

```python
import pika
import uuid

def get_connection():
    credentials = pika.PlainCredentials('user', 'password')
    params = pika.ConnectionParameters(
        host='localhost',
        port=5672,
        virtual_host='/',
        credentials=credentials,
        heartbeat=60,
        blocked_connection_timeout=300,
    )
    return pika.BlockingConnection(params)

def publish(exchange: str, routing_key: str, body: str):
    conn = get_connection()
    channel = conn.channel()
    channel.confirm_delivery()          # 开启 Confirm 模式

    props = pika.BasicProperties(
        delivery_mode=2,                # 消息持久化
        message_id=str(uuid.uuid4()),   # 唯一消息 ID（幂等用）
        content_type='application/json',
    )
    try:
        channel.basic_publish(
            exchange=exchange,
            routing_key=routing_key,
            body=body,
            properties=props,
            mandatory=True,
        )
        print("Message confirmed")
    except pika.exceptions.UnroutableError:
        print("Message returned: unroutable")
    finally:
        conn.close()
```

### 8.2 同步消费者（Pika）

```python
def start_consumer(queue: str):
    conn = get_connection()
    channel = conn.channel()
    channel.basic_qos(prefetch_count=20)

    def callback(ch, method, properties, body):
        try:
            handle(body.decode())
            ch.basic_ack(delivery_tag=method.delivery_tag)
        except Exception as e:
            print(f"Error: {e}")
            # 重试次数超过阈值则不再重新入队
            retry = (properties.headers or {}).get('x-retry-count', 0)
            ch.basic_nack(
                delivery_tag=method.delivery_tag,
                requeue=(retry < 3)
            )

    channel.basic_consume(queue=queue, on_message_callback=callback)
    print("Waiting for messages...")
    channel.start_consuming()
```

### 8.3 异步消费者（aio-pika，推荐）

```python
import asyncio
import aio_pika

async def main():
    connection = await aio_pika.connect_robust(
        "amqp://user:password@localhost/",
        heartbeat=60,
    )

    async with connection:
        channel = await connection.channel()
        await channel.set_qos(prefetch_count=20)

        queue = await channel.declare_queue(
            "async_queue", durable=True
        )

        async with queue.iterator() as queue_iter:
            async for message in queue_iter:
                async with message.process(requeue=True):
                    try:
                        await handle_async(message.body.decode())
                    except Exception:
                        raise  # 触发 nack + requeue

asyncio.run(main())
```

---
## 九、实战场景题

### 场景 1：订单超时取消（延迟队列）

```
需求：用户下单后 30 分钟未支付，自动取消订单

方案（TTL + DLX）：
  ① 下单时发消息到 delay_queue（TTL=30min，无消费者）
  ② 30 分钟后消息 TTL 到期 → 转入 order_cancel_queue
  ③ 消费者监听 order_cancel_queue，检查订单状态
     - 已支付：忽略
     - 未支付：取消订单，释放库存

注意：
  - 消费者需要幂等处理（重复消费时不重复取消）
  - 延迟时间精度受队列头部阻塞影响，精度要求高时改用插件方案
```

### 场景 2：如何保证消息不丢失？

```
完整可靠链路：

生产端：
  ① channel.confirm_delivery() 开启 Confirm
  ② 收到 nack 或超时时，从本地重试表重新发送
  ③ mandatory=True 防止消息无法路由时静默丢弃

Broker 端：
  ① Exchange / Queue / 消息 三层 durable=True / delivery_mode=2
  ② 使用仲裁队列（Quorum Queue）防止单节点宕机丢消息

消费端：
  ① auto_ack=False，手动 ack
  ② 业务处理成功后再 ack
  ③ 异常时 nack + requeue，超过重试次数转入死信队列
  ④ 死信队列单独告警监控
```

### 场景 3：如何防止消息重复消费？

```
重复消费的根本原因：
  - 消费者处理完成但 ACK 发出前 Broker 宕机
  - 网络问题导致 ACK 丢失，Broker 重发消息

解决方案（幂等设计）：
  ① 生产者为每条消息生成全局唯一 message_id（UUID）
  ② 消费者处理前查询 Redis：setnx consumed:{message_id} 1
     - 返回 1：首次消费，正常处理
     - 返回 0：已消费，直接 ACK 跳过
  ③ 数据库操作使用唯一约束兜底（insert ignore / ON CONFLICT）
```

### 场景 4：消息积压百万条怎么办？

```
紧急处理步骤：
  ① 排查消费者：是否宕机？处理逻辑是否阻塞？
  ② 快速扩容：临时增加消费者实例（同一队列多消费者）
  ③ 消息迁移（极端积压）：
     a. 新建 N 个临时队列
     b. 临时消费者从原队列取消息，按 Hash 分发到 N 个临时队列
     c. N 倍消费者并行消费临时队列
     d. 处理完毕后下线临时资源
  ④ 限制生产速率：在上游服务加令牌桶限流
  ⑤ 监控告警：设置队列深度阈值告警（建议 > 1万 触发告警）
```

---

## 十、高频追问与陷阱题

### Q1：RabbitMQ 如何实现事务消息？Confirm 和事务有何区别？

> **AMQP 事务**：`tx_select()` → 操作 → `tx_commit()`  
> 同步阻塞，性能极差（比普通发送慢 250 倍），**生产不推荐**。  
>
> **Confirm 机制**：异步确认，Broker 收到消息后回调 ack/nack，**不阻塞生产者**，性能好。  
>
> 两者本质区别：事务是同步阻塞保证原子性；Confirm 是异步通知保证送达性，各有适用场景。  
> **生产环境一律使用 Confirm 机制**。

---

### Q2：消息被 nack 后一直重试导致死循环怎么办？

> 问题：消息处理逻辑有 Bug，nack + requeue 导致消息无限重入队列，CPU 飙高。
>
> **解决方案**：  
> ① 在消息 headers 中维护 `x-retry-count`，超过阈值（如 3 次）后改为 `requeue=False`，将消息转入死信队列  
> ② 消费端捕获不可恢复异常（如数据格式错误），直接 nack + `requeue=False`  
> ③ 死信队列配置监控告警，人工介入排查

---

### Q3：RabbitMQ 与 Kafka 如何选型？

| 维度 | RabbitMQ | Kafka |
|---|---|---|
| 消息模型 | 推送（Push），消息消费后删除 | 拉取（Pull），消息持久化可回溯 |
| 吞吐量 | 万级 QPS | 百万级 QPS |
| 延迟 | 微秒级（极低延迟） | 毫秒级 |
| 消息路由 | 丰富（4种Exchange） | 简单（Topic/Partition） |
| 顺序消费 | 单队列单消费者保证 | Partition 内有序 |
| 消息回溯 | ❌ 不支持 | ✅ 支持（按 offset 回溯） |
| 适用场景 | 任务队列、RPC、复杂路由 | 日志流、大数据、事件溯源 |

> **选型建议**：  
> - 需要**复杂路由、低延迟、任务队列**：选 RabbitMQ  
> - 需要**高吞吐、消息回溯、流处理**：选 Kafka

---

### Q4：Virtual Host 的作用是什么？

> VHost 是 RabbitMQ 的**逻辑隔离单元**，每个 VHost 有独立的：Exchange、Queue、Binding、权限体系。  
> 类似 MySQL 的 database，不同业务使用不同 VHost，互不影响。  
>
> ```bash
> # 创建 VHost
> rabbitmqctl add_vhost /order_service
> # 授权用户
> rabbitmqctl set_permissions -p /order_service app_user ".*" ".*" ".*"
> ```

---

### Q5：如何监控 RabbitMQ 健康状态？

```bash
# Management UI（默认 15672 端口）
http://localhost:15672

# HTTP API 查询队列状态
curl -u user:pass http://localhost:15672/api/queues/%2F/my_queue

# 关键监控指标
队列深度（messages_ready）      → 积压告警
消费者数量（consumers）         → 掉零告警
消息入队速率（publish_rate）
消息消费速率（deliver_rate）
连接数 / Channel 数
节点内存使用率                  → 超过 watermark 触发 flow control
磁盘空间                        → 低于 disk_free_limit 触发告警
```

---

### Q6：Prefetch 设置为 1 和设置为 0 分别有什么问题？

> - **`prefetch=1`**：消费者每次只持有 1 条未 ACK 消息，完全公平但吞吐量极低，消费者处理速度慢时消息大量积压在队列  
> - **`prefetch=0`**（无限制）：Broker 将队列中所有消息推给消费者，单消费者内存暴涨，且负载无法均衡到其他消费者  
> - **推荐值**：根据消费者处理耗时调整，通常 `10~50`；I/O 密集型任务可适当增大

---

### Q7：RabbitMQ 节点重启后消息会丢失吗？

> 取决于持久化配置：
>
> | 配置组合 | 重启后是否丢失 |
> |---|---|
> | Queue 非 durable | ✅ 丢失（队列本身消失） |
> | Queue durable + 消息非持久化 | ✅ 丢失（队列恢复，消息丢失） |
> | Queue durable + 消息 delivery_mode=2 | ❌ 不丢失（写入磁盘） |
> | 仲裁队列 + 多数节点存活 | ❌ 不丢失（Raft 多副本） |
>
> ⚠️ **注意**：即使消息持久化，写磁盘是**异步**的（先写 OS page cache），极端情况下断电仍可能丢少量消息。生产环境需配合仲裁队列使用。

---

## 附录：面试前必看清单

- [ ] 能描述 AMQP 消息从生产到消费的完整流转链路
- [ ] 能说清楚 Confirm 机制与 AMQP 事务的区别和适用场景
- [ ] 能设计消息不丢失的完整方案（三层持久化 + Confirm + 手动 ACK）
- [ ] 能解释死信队列的产生条件及延迟队列两种实现方案
- [ ] 熟悉仲裁队列与镜像队列的区别，能说明 Raft 多数派原则
- [ ] 能写出 Python 幂等消费的核心代码逻辑
- [ ] 能对比 RabbitMQ 与 Kafka 的选型依据
- [ ] 了解消息积压的排查思路和紧急处理方案

---

*文档版本：RabbitMQ 3.12+ 适用 ｜ 最后更新：2025 年*
