**# AI 营销素材自动化生成平台 — 面试完整材料**



\> ***\*适用场景\****：8 年后端工程师技术面试，用于面试前自我梳理、现场介绍及追问应对

\>

\> ***\*技术栈\****：Python · Flask · PostgreSQL · Redis · RabbitMQ · Docker · Jenkins



**---**



**## 📑 大纲目录**



\- [一、项目介绍（STAR 结构）](#一项目介绍star-结构30秒2分钟口述版)

\- [二、技术架构设计](#二技术架构设计)

 \- [2.1 整体架构图](#21-整体架构图)

 \- [2.2 四大业务能力对比](#22-四大业务能力差异对比面试必备)

 \- [2.3 技术选型说明](#23-技术选型说明面试高频追问)

\- [三、数据接入层核心模块设计](#三数据接入层核心模块设计)

 \- [3.1 模块职责划分](#31-模块职责划分)

 \- [3.2 数据稳定性保障核心设计](#32-数据稳定性保障核心设计)

\- [四、任务队列设计与消息可靠性](#四任务队列设计与消息可靠性)

 \- [4.1 四队列独立隔离设计](#41-四队列独立隔离设计)

 \- [4.2 消息可靠性三重保障](#42-消息可靠性三重保障)

\- [五、性能瓶颈排查与优化](#五性能瓶颈排查与优化核心亮点)

 \- [5.1 图片下载串行 → 并发优化](#51-瓶颈一图片下载串行--并发优化)

 \- [5.2 数据库慢查询优化](#52-瓶颈二数据库慢查询--索引--查询优化)

 \- [5.3 Redis 热 Key 缓存击穿](#53-瓶颈三redis-热-key--模板配置缓存击穿)

 \- [5.4 AI 服务熔断器模式](#54-瓶颈四ai-服务调用超时--熔断器模式)

\- [六、面试高频问题 & 标准回答](#六面试高频问题--标准回答)

\- [七、项目亮点总结](#七项目亮点总结面试收尾话术)



**---**



**## 一、项目介绍（STAR 结构，30秒~2分钟口述版）**



**### 【背景 Situation】**



公司电商业务增长迅速，运营团队每日需处理大量商品上架工作，传统人工制作营销图效率低下、风格不统一，设计资源严重不足，无法支撑业务规模化扩张。



**### 【任务 Task】**



参与立项并主导核心后端服务建设，构建一套基于 AI 能力（图像生成 / 图像理解 / 视频生成）的商品营销素材自动化生成平台，实现「商品图片输入 → 高质量营销素材输出」的全链路自动化，支持四大核心能力：



| 能力 | 说明 |

|------|------|

| 🎨 文生图（T2I） | 输入 Prompt 文本，生成商品营销图片 |

| 🖼️ 图生图（I2I） | 输入商品原图，输出风格化营销图片 |

| 🎬 文生视频（T2V） | 输入 Prompt 文本，生成商品宣传视频 |

| 📹 图生视频（I2V） | 输入商品图片，生成动态展示视频 |



**### 【行动 Action】**



\- 参与需求研讨与技术选型，提出并落地核心可行性方案

\- 负责***\*数据接入层\****核心模块开发，保障数据稳定性

\- 主导性能瓶颈排查与优化，提升 QPS 与响应时间

\- 持续迭代优化，处理线上问题与客户反馈



**### 【结果 Result】**



\- 平台支撑日均 ***\*数万张\**** 素材自动生成，设计人力成本降低 ***\*70%+\****

\- 系统 P99 响应时间从 ***\*> 5s\**** 优化至 ***\*< 1.5s\****

\- 服务可用性达 ***\*99.9%+\****，数据丢失率控制在 ***\*0.01%\**** 以内



\> ✅ ***\*面试技巧\****：Result 部分务必有量化数据，请将以上数据替换为真实值。



**---**



**## 二、技术架构设计**



**### 2.1 整体架构图**



\```

┌─────────────────────────────────────────────────────────────────┐

│            客户端 / 运营后台 / OpenAPI          │

└──────────────────────────┬──────────────────────────────────────┘

​              │ HTTP REST API

┌──────────────────────────▼──────────────────────────────────────┐

│           Flask API Gateway 层             │

│       请求校验 · 鉴权 · 限流(Redis) · 任务路由         │

└──────┬────────────────────────────────────────┬─────────────────┘

​    │                     │

┌──────▼──────┐            ┌──────────▼──────────┐

│   Redis   │            │    RabbitMQ     │

│  缓存/限流  │            │  4 类任务独立 Queue  │

│  任务状态轮询│            │  + 死信队列(DLQ)   │

└─────────────┘            └──────────┬───────────┘

​                        │

​       ┌─────────────────────────────────▼──────────────────────────┐

​       │            Worker 消费者集群            │

​       │                                │

​       │  ┌───────────────────────────────────────────────────────┐  │

​       │  │         数据接入层 (DAL)             │  │

​       │  │  · 商品数据标准化 & 多源聚合               │  │

​       │  │  · 图片下载/格式校验/尺寸预处理              │  │

​       │  │  · Prompt 模板引擎(变量注入/风格匹配)           │  │

​       │  └───────────────────┬───────────────────────────────────┘  │

​       │            │                    │

​       │  ┌───────────────────▼───────────────────────────────────┐  │

​       │  │          AI 调度层                │  │

​       │  │                             │  │

​       │  │  ┌─────────────┐  ┌─────────────┐           │  │

​       │  │  │  文生图(T2I) │  │  图生图(I2I) │  Queue: q1 / q2  │  │

​       │  │  └─────────────┘  └─────────────┘           │  │

​       │  │  ┌─────────────┐  ┌─────────────┐           │  │

​       │  │  │  文生视频  │  │  图生视频   │  Queue: q3 / q4  │  │

​       │  │  │  (T2V)   │  │  (I2V)   │           │  │

​       │  │  └─────────────┘  └─────────────┘           │  │

​       │  │     熔断器 · 超时控制 · 重试策略 · 降级        │  │

​       │  └───────────────────┬───────────────────────────────────┘  │

​       └─────────────────────┼──────────────────────────────────────┘

​                  │

​       ┌─────────────────────▼──────────────────────────────────────┐

​       │          PostgreSQL                 │

​       │   任务记录 · 素材元数据 · 模板配置 · 用户配置 · 计费记录    │

​       └─────────────────────────────────────────────────────────────┘



​       ┌─────────────────────────────────────────────────────────────┐

​       │        CI/CD：Jenkins + Docker             │

​       │     自动构建 · 镜像推送 · 分队列灰度扩容          │

​       └─────────────────────────────────────────────────────────────┘

\```

## 2.2 四大业务能力差异对比（面试必备）**



| 能力 | 输入 | 输出 | 典型耗时 | 队列优先级 | 核心难点 |

|------|------|------|---------|-----------|---------|

| ***\*文生图 T2I\**** | Prompt 文本 + 参数 | 图片（PNG/JPG） | 3 ~ 8s | 高 | Prompt 质量控制 |

| ***\*图生图 I2I\**** | 商品原图 + Prompt | 风格化图片 | 5 ~ 12s | 高 | 图片预处理 + 相似度控制 |

| ***\*文生视频 T2V\**** | Prompt 文本 | 视频（MP4） | 30 ~ 120s | 低 | 超长耗时 + 大文件存储 |

| ***\*图生视频 I2V\**** | 商品图片 + 动效描述 | 动态视频 | 60 ~ 180s | 低 | 帧一致性 + 回调超时 |



\> 📌 ***\*架构关键决策\****：图/文生视频耗时是图片任务的 ***\*10 ~ 20 倍\****，必须独立 Queue + 独立 Worker 池，避免视频任务阻塞图片任务。



**---**



**### 2.3 技术选型说明（面试高频追问）**



| 技术 | 选型理由 | 备选方案 | 为什么没选 |

|------|---------|---------|-----------|

| ***\*Flask\**** | 轻量灵活，团队熟悉，适合快速迭代；AI 调用 IO 密集型场景下配合 Gunicorn 多进程足够 | FastAPI | 当时团队 FastAPI 经验不足，且异步需求可通过 MQ 解耦，非强依赖 async |

| ***\*PostgreSQL\**** | 支持 JSONB 存储动态素材配置，事务可靠，适合元数据管理 | MySQL | JSONB 索引能力更强，适合模板配置灵活查询 |

| ***\*Redis\**** | 高性能缓存 + 分布式锁 + 限流计数器，多功能复用 | Memcached | 支持更丰富数据结构，持久化能力满足任务状态需求 |

| ***\*RabbitMQ\**** | AI 生图耗时长（3 ~ 30s），必须异步化；保证任务不丢失，支持死信队列兜底 | Kafka | 任务量级不需要 Kafka 高吞吐，RabbitMQ 消息确认机制更适合任务型场景 |

| ***\*Docker + Jenkins\**** | 环境一致性 + 自动化发布，支持快速横向扩容 Worker | K8s | 初期规模不大，K8s 运维成本过高，Docker Compose + Jenkins 满足需求 |



**---**



**## 三、数据接入层核心模块设计**



**### 3.1 模块职责划分**



\```

dal/

├── ingestion/

│  ├── product_fetcher.py   # 商品数据多源拉取（API/DB/消息）

│  ├── image_downloader.py   # 图片下载（连接池/重试/超时控制）

│  ├── image_validator.py   # 格式/尺寸/内容安全校验

│  └── image_preprocessor.py # 缩放/裁剪/格式转换/背景处理

├── transform/

│  ├── normalizer.py      # 商品数据标准化（字段映射/清洗）

│  └── prompt_builder.py    # Prompt 模板引擎

└── output/

  └── task_publisher.py    # 任务封装 & 发布到 MQ

\```



**---**



**### 3.2 数据稳定性保障核心设计**



**#### ① 图片下载稳定性 — 带重试的连接池**



\```python

import requests

from requests.adapters import HTTPAdapter

from urllib3.util.retry import Retry



class ImageDownloader:

  def __init__(self):

​    self.session = requests.Session()

​    \# 关键：配置重试策略，覆盖网络抖动场景

​    retry = Retry(

​      total=3,

​      backoff_factor=0.5,      # 0.5s → 1s → 2s 指数退避

​      status_forcelist=[500, 502, 503, 504],

​      allowed_methods=["GET"]

​    )

​    adapter = HTTPAdapter(

​      pool_connections=20,

​      pool_maxsize=100,       # 连接池复用，避免频繁建连

​      max_retries=retry

​    )

​    self.session.mount("http://", adapter)

​    self.session.mount("https://", adapter)



  def download(self, url: str, timeout: int = 10) -> bytes:

​    try:

​      resp = self.session.get(url, timeout=timeout, stream=True)

​      resp.raise_for_status()

​      content = resp.content

​      \# 完整性校验：验证图片 magic bytes

​      self._validate_magic_bytes(content)

​      return content

​    except Exception as e:

​      \# 记录失败原因，发送到死信队列而非丢弃

​      raise ImageDownloadError(f"Download failed: {url}, reason: {e}")



  def _validate_magic_bytes(self, data: bytes):

​    """校验文件头，防止下载到 HTML 错误页"""

​    magic = {

​      b'\xff\xd8\xff': 'JPEG',

​      b'\x89PNG': 'PNG',

​      b'RIFF': 'WEBP',

​    }

​    for sig, fmt in magic.items():

​      if data[:len(sig)] == sig:

​        return fmt

​    raise InvalidImageError("Invalid image format, magic bytes mismatch")

\```



**#### ② 数据标准化 — 多源商品数据统一结构**



\```python

from dataclasses import dataclass, field

from typing import Optional



@dataclass

class NormalizedProduct:

  """统一数据模型，屏蔽上游数据源差异"""

  product_id: str

  name: str

  category: str

  image_urls: list[str]

  description: Optional[str] = None

  tags: list[str] = field(default_factory=list)

  extra: dict = field(default_factory=dict)  # 保留原始扩展字段（存 JSONB）



class ProductNormalizer:

  \# 字段映射表：支持多平台数据源差异

  FIELD_MAPPING = {

​    "platform_a": {"id": "sku_id",  "name": "goods_name", "imgs": "image_list"},

​    "platform_b": {"id": "item_id", "name": "title",    "imgs": "pic_urls"},

  }



  def normalize(self, raw: dict, source: str) -> NormalizedProduct:

​    mapping = self.FIELD_MAPPING.get(source, {})

​    return NormalizedProduct(

​      product_id=str(raw.get(mapping.get("id", "id"))),

​      name=self._clean_text(raw.get(mapping.get("name", "name"), "")),

​      category=raw.get("category", "通用"),

​      image_urls=raw.get(mapping.get("imgs", "images"), []),

​      tags=raw.get("tags", []),

​      extra={k: v for k, v in raw.items()}  # 完整原始数据备份

​    )



  def _clean_text(self, text: str) -> str:

​    """去除特殊字符，防止注入 Prompt"""

​    import re

​    return re.sub(r'[^\w\s\u4e00-\u9fff，。！？、]', '', text).strip()

\```

**#### ③ Prompt 模板引擎 — 支持四种任务类型**



\```python

class PromptBuilder:

  """根据任务类型 + 商品信息 + 风格配置动态构建 Prompt"""



  TEMPLATES = {

​    "t2i": "一张{style}风格的{category}商品营销海报，主题：{name}，{desc}，高清，商业摄影",

​    "i2i": "将商品图转换为{style}风格，保留商品主体，背景替换为{scene}，专业商品摄影",

​    "t2v": "一段{duration}秒的{category}商品宣传视频，{name}，{style}风格，流畅动态展示",

​    "i2v": "商品图片动态化，{motion_type}运动效果，{duration}秒，{style}风格，突出{focus}",

  }



  def build(self, task_type: str, product: NormalizedProduct, style_config: dict) -> str:

​    template = self.TEMPLATES[task_type]

​    context = {

​      "name": product.name[:30],     # 截断防止 Prompt 超长

​      "category": product.category,

​      "desc": (product.description or "")[:50],

​      **style_config

​    }

​    prompt = template.format(**context)

​    \# Prompt 长度控制（不同模型 token 限制不同）

​    return prompt[:500] if task_type in ("t2i", "i2i") else prompt[:300]

\```



**---**



**## 四、任务队列设计与消息可靠性**



**### 4.1 四队列独立隔离设计**



\```python

\# RabbitMQ 队列配置

QUEUE_CONFIG = {

  "t2i": {

​    "queue":   "task.text_to_image",

​    "workers":  10,  # 图片任务并发高

​    "prefetch": 5,

​    "priority": 10,

  },

  "i2i": {

​    "queue":   "task.image_to_image",

​    "workers":  10,

​    "prefetch": 5,

​    "priority": 10,

  },

  "t2v": {

​    "queue":   "task.text_to_video",

​    "workers":  3,   # 视频任务耗资源，限并发

​    "prefetch": 1,

​    "priority": 5,

  },

  "i2v": {

​    "queue":   "task.image_to_video",

​    "workers":  3,

​    "prefetch": 1,

​    "priority": 5,

  },

  \# 死信队列：失败任务兜底

  "dlq": {

​    "queue":   "task.dead_letter",

​    "workers":  1,

​    "prefetch": 1,

  },

}

\```



**---**



**### 4.2 消息可靠性三重保障**



\```

消息可靠性保障链路：



发布端          消息队列         消费端

 │            │            │

 ├─ Publisher Confirm ──►│            │

 │  (发布确认)       │◄──── 手动 ACK ───────┤

 │            │    (处理完再确认)   │

 │            │ 消息持久化       ├─ 幂等处理

 │            │ (durable=True)     │  (task_id 去重)

 │            │            │

 │            │            ├─ 失败 → NACK

 │            │            │  → 死信队列(DLQ)

 │            │            │  → 告警 + 人工介入

\```



| 保障层 | 机制 | 作用 |

|--------|------|------|

| ***\*发布端\**** | Publisher Confirm 模式 | 确认消息落入 Broker，失败则重发 |

| ***\*存储层\**** | `durable=True` 持久化 | Broker 重启不丢消息 |

| ***\*消费端\**** | 手动 ACK + 幂等 task_id | 处理成功才确认，重复消费安全 |

| ***\*兜底层\**** | 死信队列 + 告警 | 多次失败任务人工介入，不静默丢弃 |



**---**



**## 五、性能瓶颈排查与优化（核心亮点）**



**### 5.1 瓶颈一：图片下载串行 → 并发优化**



***\*问题现象\****：单任务处理耗时 8s+，CPU 却只有 20% 利用率，明显 IO 等待



***\*排查过程\****：



\```bash

\# 用 py-spy 采样，发现大量时间阻塞在 requests.get()

py-spy top --pid <worker_pid>

\```



***\*解决方案\****：串行下载改为线程池并发



\```python

from concurrent.futures import ThreadPoolExecutor, as_completed



class ConcurrentImageDownloader:

  def __init__(self, max_workers: int = 8):

​    self.downloader = ImageDownloader()

​    self.executor = ThreadPoolExecutor(max_workers=max_workers)



  def batch_download(self, urls: list[str]) -> dict[str, bytes]:

​    """并发下载多张图片"""

​    futures = {

​      self.executor.submit(self.downloader.download, url): url

​      for url in urls

​    }

​    results = {}

​    for future in as_completed(futures, timeout=30):

​      url = futures[future]

​      try:

​        results[url] = future.result()

​      except Exception as e:

​        results[url] = None  # 失败记录，不阻断整体

​        logger.warning(f"Image download failed: {url}, {e}")

​    return results

\```



| 指标 | 优化前 | 优化后 | 提升幅度 |

|------|--------|--------|---------|

| 图片下载阶段耗时 | 4.2s | 0.8s | ***\*↓ 81%\**** |



**---**



**### 5.2 瓶颈二：数据库慢查询 → 索引 + 查询优化**



***\*问题现象\****：任务状态查询接口 P99 > 3s，PostgreSQL slow query log 频繁报警



***\*排查过程\****：



\```sql

-- 执行 EXPLAIN ANALYZE，发现全表扫描

EXPLAIN ANALYZE

SELECT * FROM tasks

WHERE user_id = 123 AND status = 'pending'

ORDER BY created_at DESC LIMIT 20;



-- 结果：Seq Scan on tasks  实际扫描 2,800,000 行

\```



***\*解决方案\****：



\```sql

-- 1. 添加复合索引（等值查询列在前，排序列在后）

CREATE INDEX CONCURRENTLY idx_tasks_user_status_created

  ON tasks(user_id, status, created_at DESC);



-- 2. 高频状态查询结果缓存到 Redis（TTL 5s）



-- 3. 历史任务归档（> 90 天数据迁移到冷表）

\```

| 指标 | 优化前 | 优化后 | 提升幅度 |

|------|--------|--------|---------|

| 查询接口 P99 | 3s | 80ms | ***\*↓ 97%\**** |

| 接口 QPS | 200 | 1500 | ***\*↑ 650%\**** |



**---**



**### 5.3 瓶颈三：Redis 热 Key — 模板配置缓存击穿**



***\*问题现象\****：大促期间 Redis CPU 飙升到 90%+，部分请求直接打穿到 PostgreSQL



***\*根因\****：热门模板配置使用同一个 Key，瞬时并发请求同时 miss，触发缓存击穿



***\*解决方案\****：互斥锁 + 本地二级缓存



\```python

import threading, time, json



class TemplateCache:

  _local_cache = {}     # 进程级本地缓存（一级，极低延迟）



  def get_template(self, template_id: str) -> dict:

​    \# L1：本地缓存（秒级 TTL）

​    local = self._local_cache.get(template_id)

​    if local and local['expire'] > time.time():

​      return local['data']



​    \# L2：Redis 缓存

​    cached = redis_client.get(f"template:{template_id}")

​    if cached:

​      data = json.loads(cached)

​      self._set_local(template_id, data, ttl=5)

​      return data



​    \# L3：互斥锁防击穿，只允许一个请求回源 DB

​    lock_key = f"lock:template:{template_id}"

​    with redis_lock(lock_key, timeout=3):

​      \# 双重检查（拿锁后再次检查 Redis）

​      cached = redis_client.get(f"template:{template_id}")

​      if cached:

​        return json.loads(cached)



​      \# 回源 PostgreSQL

​      data = db.query_template(template_id)

​      redis_client.setex(f"template:{template_id}", 300, json.dumps(data))

​      self._set_local(template_id, data, ttl=5)

​      return data

\```



| 指标 | 优化前 | 优化后 | 提升幅度 |

|------|--------|--------|---------|

| Redis CPU | 90% | 35% | ***\*↓ 61%\**** |

| 缓存击穿 | 频繁 | 彻底消除 | ✅ |



**---**



**### 5.4 瓶颈四：AI 服务调用超时 — 熔断器模式**



***\*问题现象\****：第三方 AI 服务偶发超时（尤其视频任务），导致 Worker 线程被大量占用，形成雪崩



***\*解决方案\****：实现熔断器（Circuit Breaker）



\```python

from enum import Enum

import time



class CircuitState(Enum):

  CLOSED   = "closed"   # 正常，请求通过

  OPEN    = "open"    # 熔断，快速失败

  HALF_OPEN = "half_open"  # 探测恢复中



class CircuitBreaker:

  def __init__(self, failure_threshold=5, recovery_timeout=60):

​    self.failure_threshold = failure_threshold  # 连续失败阈值

​    self.recovery_timeout  = recovery_timeout   # 熔断恢复时间(s)

​    self.failure_count   = 0

​    self.last_failure_time = None

​    self.state       = CircuitState.CLOSED



  def call(self, func, *args, **kwargs):

​    if self.state == CircuitState.OPEN:

​      if time.time() - self.last_failure_time > self.recovery_timeout:

​        self.state = CircuitState.HALF_OPEN

​      else:

​        raise CircuitOpenError("AI service circuit breaker OPEN, fast fail")

​    try:

​      result = func(*args, **kwargs)

​      self._on_success()

​      return result

​    except Exception as e:

​      self._on_failure()

​      raise e



  def _on_success(self):

​    self.failure_count = 0

​    self.state = CircuitState.CLOSED



  def _on_failure(self):

​    self.failure_count += 1

​    self.last_failure_time = time.time()

​    if self.failure_count >= self.failure_threshold:

​      self.state = CircuitState.OPEN

​      logger.error(f"Circuit breaker OPEN! failures={self.failure_count}")

\```



| 指标 | 优化前 | 优化后 | 提升幅度 |

|------|--------|--------|---------|

| AI 故障时 Worker 可用率 | 10% | 95% | ***\*↑ 850%\**** |

| 雪崩发生次数 | 频繁 | 0 | ✅ |



**---**



**## 六、面试高频问题 & 标准回答**



**### Q1：为什么用 RabbitMQ 而不是 Kafka？**



\> 我们的业务是任务型场景，每条消息对应一个 AI 生成任务，核心诉求是***\*消息不丢失、处理确认可靠\****，而不是超高吞吐。RabbitMQ 的手动 ACK、死信队列、消息优先级都天然满足这些需求。Kafka 的 offset 机制在任务失败重试场景处理相对复杂，且我们 QPS 峰值在千级，完全不需要 Kafka 百万级吞吐能力。后期如果有日志或埋点场景再引入 Kafka。



**---**



**### Q2：视频任务耗时 180s，怎么保证用户体验？**



\> 三层策略：

\> 1. ***\*异步 + 轮询\****：接口立即返回 `task_id`，客户端通过 WebSocket 或轮询查询进度，Redis 存储任务状态（`pending / processing / success / failed`）

\> 2. ***\*进度推送\****：AI 服务每完成一帧回调进度，Worker 更新 Redis，前端实时展示百分比

\> 3. ***\*超时兜底\****：视频任务设置 300s 最大超时，超时后进入死信队列，触发告警和自动重试，最多重试 2 次



**---**



**### Q3：数据接入层如何保证数据稳定性？**



\> 四个维度：

\> 1. ***\*下载层\****：连接池复用 + 3 次指数退避重试 + magic bytes 完整性校验

\> 2. ***\*处理层\****：数据标准化统一模型，字段映射表兼容多数据源差异，异常字段用默认值兜底不抛出

\> 3. ***\*消息层\****：RabbitMQ publisher confirm + 手动 ACK，处理成功才确认，失败进死信队列不丢失

\> 4. ***\*任务层\****：幂等设计，task_id 全局唯一，Redis 记录处理状态，防止重复消费



**---**

**### Q4：如何设计 Prompt 才能保证生成质量？**



\> 三个层面：

\> 1. ***\*模板化\****：针对不同品类（服装/食品/电子）维护不同 Prompt 模板，经 A/B 测试迭代优化

\> 2. ***\*动态注入\****：商品名、品类、描述等关键信息动态填充，增强 Prompt 相关性

\> 3. ***\*防护处理\****：对商品文本做清洗（去除特殊字符/长度截断），避免影响 Prompt 结构，同时接入内容安全审核 API 过滤违规内容

\>

\> 另外我们还维护了一个 ***\*Negative Prompt 库\****（反向提示词），显著降低了生成废片率。



**---**



**### Q5：系统最大的技术挑战是什么？**



\> 最大挑战是***\*异构任务的资源隔离与调度\****。图片任务（3~12s）和视频任务（60~180s）资源消耗差异巨大，如果混用 Worker 池，视频任务会把所有线程占满，图片任务饿死。

\>

\> 解法是：***\*按任务类型独立 Queue + 独立 Worker 进程池\****，通过 Docker 独立部署，可以针对不同队列独立扩缩容。同时对视频任务实施 `prefetch=1`（每个 Worker 同时只处理 1 个视频任务），防止单 Worker 资源过载。

\>

\> 这个设计让图片任务 P99 响应时间即使在大促期间也能稳定在 ***\*1.5s 以内\****。



**---**



**### Q6：Jenkins + Docker 的 CI/CD 流程怎么设计的？**



\> 标准四阶段流水线：

\> 1. ***\*Build\****：代码提交触发 Jenkins，运行单元测试（pytest），测试通过后 `docker build` 镜像

\> 2. ***\*Push\****：镜像推送到私有 Harbor 仓库，Tag 使用 `{branch}-{commit_hash}`

\> 3. ***\*Deploy\****：先更新 Staging 环境验证，手动确认后灰度发布 Production（先更新 20% 节点，观察 5 分钟错误率）

\> 4. ***\*Monitor\****：发布后自动运行冒烟测试，失败自动回滚到上一个镜像版本

\>

\> Worker 扩容只需修改 docker-compose replicas 配置，5 分钟内完成横向扩容。



**---**



**### Q7：如何处理 AI 服务的多供应商切换？**



\> 我们在 AI 调用层做了***\*供应商抽象\****，定义统一的接口协议，各供应商实现各自的 Adapter。当某个供应商出现故障或成本超出预算时，只需修改路由配置即可切换，业务层完全无感知。同时维护了一个供应商健康评分系统，根据响应时间、成功率、成本动态调整流量权重，实现最优成本下的稳定性保障。



**---**



**## 七、项目亮点总结（面试收尾话术）**



\> 💡 建议在面试结束前主动提炼，给面试官留下深刻印象：



**---**



***\*"这个项目我认为有三个核心亮点：\****



***\*第一，多模态 AI 任务的工程化落地\****



将文生图、图生图、文生视频、图生视频四种耗时差异巨大的 AI 能力统一纳入同一平台，通过独立队列、独立 Worker 池、熔断降级等机制保障了整体服务稳定性，四类任务互不干扰，各自可独立扩缩容。



***\*第二，数据接入层的高可靠设计\****



从图片下载的连接池重试、到数据标准化的多源兼容、再到消息投递的三重可靠性保障，每个环节都有明确的失败兜底策略，线上数据丢失率控制在 0.01% 以内。



***\*第三，有效的性能优化方法论\****



不是拍脑袋优化，而是通过 py-spy 采样、slow query log、Redis 监控等手段精准定位瓶颈，优化前后有明确量化指标对比。每次优化都产生了可见的 QPS 提升或延迟降低：DB 查询 P99 从 3s 降到 80ms，图片下载并发优化节省 81% 时间，熔断器让 AI 故障时 Worker 可用率从 10% 恢复到 95%。"



**---**



**## 八、关键量化指标汇总**



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



**---**



\> 📌 ***\*最后提示\****：以上所有量化数据请在面试前替换为你项目的***\*真实数据\****，面试官可能会追问数据来源和测量方式，务必做到心中有数。
