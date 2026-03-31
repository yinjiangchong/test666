## 五、性能瓶颈排查与优化（核心亮点）

### 5.1 瓶颈一：图片下载串行 → 并发优化

**问题现象**：单任务处理耗时 8s+，CPU 却只有 20% 利用率，明显 IO 等待

**排查过程**：

```bash
# 用 py-spy 采样，发现大量时间阻塞在 requests.get()
py-spy top --pid <worker_pid>
```

**解决方案**：串行下载改为线程池并发

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

class ConcurrentImageDownloader:
    def __init__(self, max_workers: int = 8):
        self.downloader = ImageDownloader()
        self.executor = ThreadPoolExecutor(max_workers=max_workers)

    def batch_download(self, urls: list[str]) -> dict[str, bytes]:
        """并发下载多张图片"""
        futures = {
            self.executor.submit(self.downloader.download, url): url
            for url in urls
        }
        results = {}
        for future in as_completed(futures, timeout=30):
            url = futures[future]
            try:
                results[url] = future.result()
            except Exception as e:
                results[url] = None   # 失败记录，不阻断整体
                logger.warning(f"Image download failed: {url}, {e}")
        return results
```

| 指标 | 优化前 | 优化后 | 提升幅度 |
|------|--------|--------|---------|
| 图片下载阶段耗时 | 4.2s | 0.8s | **↓ 81%** |

---

### 5.2 瓶颈二：数据库慢查询 → 索引 + 查询优化

**问题现象**：任务状态查询接口 P99 > 3s，PostgreSQL slow query log 频繁报警

**排查过程**：

```sql
-- 执行 EXPLAIN ANALYZE，发现全表扫描
EXPLAIN ANALYZE
SELECT * FROM tasks
WHERE user_id = 123 AND status = 'pending'
ORDER BY created_at DESC LIMIT 20;

-- 结果：Seq Scan on tasks  实际扫描 2,800,000 行
```

**解决方案**：

```sql
-- 1. 添加复合索引（等值查询列在前，排序列在后）
CREATE INDEX CONCURRENTLY idx_tasks_user_status_created
    ON tasks(user_id, status, created_at DESC);

-- 2. 高频状态查询结果缓存到 Redis（TTL 5s）

-- 3. 历史任务归档（> 90 天数据迁移到冷表）
```

| 指标 | 优化前 | 优化后 | 提升幅度 |
|------|--------|--------|---------|
| 查询接口 P99 | 3s | 80ms | **↓ 97%** |
| 接口 QPS | 200 | 1500 | **↑ 650%** |

---

### 5.3 瓶颈三：Redis 热 Key — 模板配置缓存击穿

**问题现象**：大促期间 Redis CPU 飙升到 90%+，部分请求直接打穿到 PostgreSQL

**根因**：热门模板配置使用同一个 Key，瞬时并发请求同时 miss，触发缓存击穿

**解决方案**：互斥锁 + 本地二级缓存

```python
import threading, time, json

class TemplateCache:
    _local_cache = {}         # 进程级本地缓存（一级，极低延迟）

    def get_template(self, template_id: str) -> dict:
        # L1：本地缓存（秒级 TTL）
        local = self._local_cache.get(template_id)
        if local and local['expire'] > time.time():
            return local['data']

        # L2：Redis 缓存
        cached = redis_client.get(f"template:{template_id}")
        if cached:
            data = json.loads(cached)
            self._set_local(template_id, data, ttl=5)
            return data

        # L3：互斥锁防击穿，只允许一个请求回源 DB
        lock_key = f"lock:template:{template_id}"
        with redis_lock(lock_key, timeout=3):
            # 双重检查（拿锁后再次检查 Redis）
            cached = redis_client.get(f"template:{template_id}")
            if cached:
                return json.loads(cached)

            # 回源 PostgreSQL
            data = db.query_template(template_id)
            redis_client.setex(f"template:{template_id}", 300, json.dumps(data))
            self._set_local(template_id, data, ttl=5)
            return data
```

| 指标 | 优化前 | 优化后 | 提升幅度 |
|------|--------|--------|---------|
| Redis CPU | 90% | 35% | **↓ 61%** |
| 缓存击穿 | 频繁 | 彻底消除 | ✅ |

---
### 5.4 瓶颈四：AI 服务调用超时 — 熔断器模式

**问题现象**：第三方 AI 服务偶发超时（尤其视频任务），导致 Worker 线程被大量占用，形成雪崩

**解决方案**：实现熔断器（Circuit Breaker）

```python
from enum import Enum
import time

class CircuitState(Enum):
    CLOSED    = "closed"     # 正常，请求通过
    OPEN      = "open"       # 熔断，快速失败
    HALF_OPEN = "half_open"  # 探测恢复中

class CircuitBreaker:
    def __init__(self, failure_threshold=5, recovery_timeout=60):
        self.failure_threshold = failure_threshold   # 连续失败阈值
        self.recovery_timeout  = recovery_timeout    # 熔断恢复时间(s)
        self.failure_count     = 0
        self.last_failure_time = None
        self.state             = CircuitState.CLOSED

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError("AI service circuit breaker OPEN, fast fail")
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e

    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.error(f"Circuit breaker OPEN! failures={self.failure_count}")
```

| 指标 | 优化前 | 优化后 | 提升幅度 |
|------|--------|--------|---------|
| AI 故障时 Worker 可用率 | 10% | 95% | **↑ 850%** |
| 雪崩发生次数 | 频繁 | 0 | ✅ |

---

## 六、面试高频问题 & 标准回答

### Q1：为什么用 RabbitMQ 而不是 Kafka？

> 我们的业务是任务型场景，每条消息对应一个 AI 生成任务，核心诉求是**消息不丢失、处理确认可靠**，而不是超高吞吐。RabbitMQ 的手动 ACK、死信队列、消息优先级都天然满足这些需求。Kafka 的 offset 机制在任务失败重试场景处理相对复杂，且我们 QPS 峰值在千级，完全不需要 Kafka 百万级吞吐能力。后期如果有日志或埋点场景再引入 Kafka。

---

### Q2：视频任务耗时 180s，怎么保证用户体验？

> 三层策略：
> 1. **异步 + 轮询**：接口立即返回 `task_id`，客户端通过 WebSocket 或轮询查询进度，Redis 存储任务状态（`pending / processing / success / failed`）
> 2. **进度推送**：AI 服务每完成一帧回调进度，Worker 更新 Redis，前端实时展示百分比
> 3. **超时兜底**：视频任务设置 300s 最大超时，超时后进入死信队列，触发告警和自动重试，最多重试 2 次

---

### Q3：数据接入层如何保证数据稳定性？

> 四个维度：
> 1. **下载层**：连接池复用 + 3 次指数退避重试 + magic bytes 完整性校验
> 2. **处理层**：数据标准化统一模型，字段映射表兼容多数据源差异，异常字段用默认值兜底不抛出
> 3. **消息层**：RabbitMQ publisher confirm + 手动 ACK，处理成功才确认，失败进死信队列不丢失
> 4. **任务层**：幂等设计，task_id 全局唯一，Redis 记录处理状态，防止重复消费

---
### Q4：如何设计 Prompt 才能保证生成质量？

> 三个层面：
> 1. **模板化**：针对不同品类（服装/食品/电子）维护不同 Prompt 模板，经 A/B 测试迭代优化
> 2. **动态注入**：商品名、品类、描述等关键信息动态填充，增强 Prompt 相关性
> 3. **防护处理**：对商品文本做清洗（去除特殊字符/长度截断），避免影响 Prompt 结构，同时接入内容安全审核 API 过滤违规内容
>
> 另外我们还维护了一个 **Negative Prompt 库**（反向提示词），显著降低了生成废片率。

---

### Q5：系统最大的技术挑战是什么？

> 最大挑战是**异构任务的资源隔离与调度**。图片任务（3~12s）和视频任务（60~180s）资源消耗差异巨大，如果混用 Worker 池，视频任务会把所有线程占满，图片任务饿死。
>
> 解法是：**按任务类型独立 Queue + 独立 Worker 进程池**，通过 Docker 独立部署，可以针对不同队列独立扩缩容。同时对视频任务实施 `prefetch=1`（每个 Worker 同时只处理 1 个视频任务），防止单 Worker 资源过载。
>
> 这个设计让图片任务 P99 响应时间即使在大促期间也能稳定在 **1.5s 以内**。

---

### Q6：Jenkins + Docker 的 CI/CD 流程怎么设计的？

> 标准四阶段流水线：
> 1. **Build**：代码提交触发 Jenkins，运行单元测试（pytest），测试通过后 `docker build` 镜像
> 2. **Push**：镜像推送到私有 Harbor 仓库，Tag 使用 `{branch}-{commit_hash}`
> 3. **Deploy**：先更新 Staging 环境验证，手动确认后灰度发布 Production（先更新 20% 节点，观察 5 分钟错误率）
> 4. **Monitor**：发布后自动运行冒烟测试，失败自动回滚到上一个镜像版本
>
> Worker 扩容只需修改 docker-compose replicas 配置，5 分钟内完成横向扩容。

---

### Q7：如何处理 AI 服务的多供应商切换？

> 我们在 AI 调用层做了**供应商抽象**，定义统一的接口协议，各供应商实现各自的 Adapter。当某个供应商出现故障或成本超出预算时，只需修改路由配置即可切换，业务层完全无感知。同时维护了一个供应商健康评分系统，根据响应时间、成功率、成本动态调整流量权重，实现最优成本下的稳定性保障。

---

## 七、项目亮点总结（面试收尾话术）

> 💡 建议在面试结束前主动提炼，给面试官留下深刻印象：

---

**"这个项目我认为有三个核心亮点：**

**第一，多模态 AI 任务的工程化落地**

将文生图、图生图、文生视频、图生视频四种耗时差异巨大的 AI 能力统一纳入同一平台，通过独立队列、独立 Worker 池、熔断降级等机制保障了整体服务稳定性，四类任务互不干扰，各自可独立扩缩容。

**第二，数据接入层的高可靠设计**

从图片下载的连接池重试、到数据标准化的多源兼容、再到消息投递的三重可靠性保障，每个环节都有明确的失败兜底策略，线上数据丢失率控制在 0.01% 以内。
-+-+-------------------------------
**第三，有效的性能优化方法论**

不是拍脑袋优化，而是通过 py-spy 采样、slow query log、Redis 监控等手段精准定位瓶颈，优化前后有明确量化指标对比。每次优化都产生了可见的 QPS 提升或延迟降低：DB 查询 P99 从 3s 降到 80ms，图片下载并发优化节省 81% 时间，熔断器让 AI 故障时 Worker 可用率从 10% 恢复到 95%。"

---

## 八、关键量化指标汇总

| 优化项 | 优化前 | 优化后 | 提升幅度 |
|--------|--------|--------|---------|
| 服务整体 P99 响应时间 | > 5s | < 1.5s | ↓ 70% |
| DB 查询接口 P99 | 3s | 80ms | ↓ 97% |
| 接口 QPS | 200 | 1500 | ↑ 650% |
| 图片下载阶段耗时 | 4.2s | 0.8s | ↓ 81% |
| Redis CPU（大促期间） | 90% | 35% | ↓ 61% |
| AI 故障时 Worker 可用率 | 10% | 95% | ↑ 850% |
| 设计人力成本 | 基准 | 降低 70%+ | ✅ |
| 服务可用性 | — | 99.9%+ | ✅ |
| 数据丢失率 | — | < 0.01% | ✅ |

---

