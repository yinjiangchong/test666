# Elasticsearch 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、搜索引擎方向面试  
> 覆盖范围：核心原理 · 索引与映射 · 查询优化 · 集群高可用 · 实战场景

---

## 目录

1. [基础概念与核心架构](#一基础概念与核心架构)
2. [索引原理与倒排索引](#二索引原理与倒排索引)
3. [映射（Mapping）与数据类型](#三映射mapping与数据类型)
4. [查询 DSL 与搜索优化](#四查询-dsl-与搜索优化)
5. [聚合分析](#五聚合分析)
6. [写入原理与性能调优](#六写入原理与性能调优)
7. [集群架构与高可用](#七集群架构与高可用)
8. [Python 实战（elasticsearch-py）](#八python-实战elasticsearch-py)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、基础概念与核心架构

### 1.1 核心概念对比（ES vs MySQL）

| Elasticsearch | MySQL | 说明 |
|---|---|---|
| Index | Database/Table | 索引，存储同类文档的集合 |
| Document | Row | 文档，JSON 格式的数据单元 |
| Field | Column | 字段 |
| Mapping | Schema | 字段类型定义 |
| Shard | Partition | 分片，索引的物理分布单元 |
| Replica | Slave | 副本，提供高可用和读扩展 |
| Node | Server | 集群中的单个实例 |
| Cluster | Cluster | 多个节点组成的集群 |

> ⚠️ **注意**：ES 7.0+ 移除了 Type 概念，一个 Index 只有一个隐式 Type `_doc`。

### 1.2 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                   Elasticsearch Cluster                   │
│                                                          │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐            │
│  │  Node 1  │    │  Node 2  │    │  Node 3  │            │
│  │ (Master) │◄──►│  (Data)  │◄──►│  (Data)  │            │
│  │ P0 | R1  │    │ P1 | R2  │    │ P2 | R0  │            │
│  │ P3 | R4  │    │ P4 | R3  │    │    | R4  │            │
│  └──────────┘    └──────────┘    └──────────┘            │
│                                                          │
│  P = Primary Shard    R = Replica Shard                  │
└──────────────────────────────────────────────────────────┘
```

### 1.3 节点类型

| 节点类型 | 职责 | 生产建议 |
|---|---|---|
| **Master Node** | 集群元数据管理（索引创建/删除、节点加入/离开） | 专用 3 个，奇数防脑裂 |
| **Data Node** | 存储分片数据，执行搜索/聚合 | 按存储容量水平扩展 |
| **Coordinating Node** | 请求路由、结果聚合（不存数据） | 大集群独立部署 |
| **Ingest Node** | 写入前数据预处理（pipeline） | 按需部署 |
| **ML Node** | 机器学习任务 | 按需部署 |

### 1.4 读写请求流程

**写入流程**：
```
Client
  ──► Coordinating Node
        │ 路由: shard = hash(_id) % num_primary_shards
        ▼
      Primary Shard（写入）
        │── 并行同步 ──► Replica Shard × N
        │ 等待副本确认（wait_for_active_shards）
        ▼
      返回 Client
```

**搜索流程（两阶段）**：
```
Phase 1 - Query:
  Coordinating Node 广播到所有相关 Shard
  每个 Shard 返回 top N 的 DocID + 评分

Phase 2 - Fetch:
  Coordinating Node 合并排序，取最终 top N
  根据 DocID 回查对应 Shard 获取完整文档
```

---
## 二、索引原理与倒排索引

### 2.1 倒排索引结构

```
原始文档：
  Doc1: "苹果手机 华为手机"
  Doc2: "苹果电脑 笔记本"
  Doc3: "华为手机 5G"

倒排索引：
  Term      │ Posting List（DocID: 位置）
  ──────────┼──────────────────────────────
  苹果      │ Doc1(pos:0), Doc2(pos:0)
  手机      │ Doc1(pos:1), Doc1(pos:3), Doc3(pos:1)
  华为      │ Doc1(pos:2), Doc3(pos:0)
  电脑      │ Doc2(pos:1)
  5G        │ Doc3(pos:2)
```

- **Term Dictionary**：所有 Term 排序后的字典（B-Tree/FST 结构）
- **Posting List**：每个 Term 对应的 DocID 列表（支持位置、词频等信息）
- **Term Index**：Term Dictionary 的索引（前缀树 FST，加速查找，常驻内存）

### 2.2 分析器（Analyzer）工作流程

```
原始文本
  │── Character Filter（字符过滤：去 HTML 标签、字符替换）
  ▼
  Tokenizer（分词：按规则切分成 Token 流）
  │── Token Filter（词元过滤：小写、去停用词、同义词扩展）
  ▼
最终 Term 列表（写入倒排索引）
```

**常用分析器**：

| 分析器 | 适用语言 | 说明 |
|---|---|---|
| `standard` | 英文（默认） | Unicode 分词，小写，去标点 |
| `ik_max_word` | 中文 | 最细粒度切分（需安装 IK 插件） |
| `ik_smart` | 中文 | 最粗粒度切分 |
| `pinyin` | 中文拼音 | 支持拼音搜索（需安装插件） |
| `keyword` | 任意 | 不分词，整体作为一个 Term |

```json
// 测试分析器效果
GET /_analyze
{
  "analyzer": "ik_max_word",
  "text": "我是中国人"
}
// 结果：["我", "是", "中国人", "中国", "国人", "人"]
```

### 2.3 Segment 写入机制

```
写入流程（内存 → 磁盘）：

  Write ──► In-Memory Buffer
              │ refresh（默认 1s）
              ▼
           Segment（不可变，已可搜索）
              │ flush（translog 达阈值 / 30min）
              ▼
           持久化到磁盘
              │ merge（后台定期合并小 Segment）
              ▼
           大 Segment（旧 Segment 删除）
```

- **Segment 不可变**：更新 = 标记旧文档删除 + 写入新文档
- **近实时搜索**：refresh 后新文档即可被搜索（默认 1s 延迟）
- **Translog**：类似 WAL，防止 flush 前宕机丢数据

---

## 三、映射（Mapping）与数据类型

### 3.1 常用数据类型

| 类型 | 说明 | 示例场景 |
|---|---|---|
| `text` | 全文检索，会分词 | 文章标题、内容 |
| `keyword` | 精确匹配，不分词，可聚合排序 | 状态、标签、ID |
| `integer / long / float / double` | 数值 | 价格、数量 |
| `date` | 日期 | `"2024-01-01"` / 时间戳 |
| `boolean` | 布尔 | — |
| `object` | JSON 对象（扁平化存储） | 嵌套普通对象 |
| `nested` | 嵌套对象数组（独立文档） | 订单中的商品列表 |
| `geo_point` | 地理坐标 | 经纬度 |
| `dense_vector` | 稠密向量 | 语义搜索、向量相似度 |

### 3.2 text vs keyword 核心区别

```json
// 字段同时支持全文检索和聚合/排序（multi-field）
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      }
    }
  }
}
// 全文检索：title
// 精确匹配 / 聚合 / 排序：title.keyword
```

### 3.3 Dynamic Mapping 陷阱

```json
// 生产建议：显式定义 Mapping，关闭 dynamic
PUT /my_index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "id":    { "type": "keyword" },
      "title": { "type": "text", "analyzer": "ik_max_word" },
      "price": { "type": "double" },
      "tags":  { "type": "keyword" },
      "create_time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      }
    }
  }
}
```
> ⚠️ **陷阱**：Mapping 一旦确定不可修改（只能新增字段）。需要改字段类型时，必须新建索引 + Reindex。

### 3.4 nested 与 object 的区别

```
object 类型（扁平化存储，跨字段关联失效）：
  文档：{"comments": [{"user":"A","msg":"好"}, {"user":"B","msg":"差"}]}
  存储：{"comments.user": ["A","B"], "comments.msg": ["好","差"]}
  查询 user=A AND msg=差 → 会错误命中！（关联关系丢失）

nested 类型（每个元素独立存储为隐式文档）：
  每个 comment 作为独立隐式文档存储
  查询 user=A AND msg=差 → 正确不命中（关联关系保留）
```

```json
// nested 查询语法
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "term": { "comments.user": "A" } },
            { "term": { "comments.msg": "好" } }
          ]
        }
      }
    }
  }
}
```

---

## 四、查询 DSL 与搜索优化

### 4.1 Query vs Filter

| | Query | Filter |
|---|---|---|
| 作用 | 相关性评分（_score） | 是/否匹配，不计算评分 |
| 缓存 | 不缓存 | 缓存（bitset），性能更高 |
| 适用 | 全文检索 | 精确条件过滤（状态、范围、标签） |

```json
// 推荐：filter 包裹精确条件，must 只用于全文检索
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "手机" } }
      ],
      "filter": [
        { "term":  { "status": "on_sale" } },
        { "range": { "price": { "gte": 100, "lte": 5000 } } }
      ]
    }
  }
}
```

### 4.2 常用查询类型

```json
// match：全文检索（分词后匹配）
{ "match": { "title": "苹果手机" } }

// match_phrase：短语匹配（词序固定）
{ "match_phrase": { "title": "苹果手机" } }

// term：精确匹配（不分词，适合 keyword）
{ "term": { "status": "active" } }

// terms：多值精确匹配（类似 IN）
{ "terms": { "status": ["active", "pending"] } }

// range：范围查询
{ "range": { "price": { "gte": 100, "lte": 500 } } }

// wildcard：通配符（慎用，性能差）
{ "wildcard": { "email": "*@gmail.com" } }

// fuzzy：模糊查询（纠错）
{ "fuzzy": { "title": { "value": "苹果", "fuzziness": "AUTO" } } }

// exists：字段存在性
{ "exists": { "field": "description" } }
```

### 4.3 bool 复合查询

```json
{
  "query": {
    "bool": {
      "must":     [...],   // 必须匹配，影响评分（AND）
      "should":   [...],   // 可以匹配，影响评分（OR）
      "must_not": [...],   // 必须不匹配，不影响评分（NOT）
      "filter":   [...]    // 必须匹配，不影响评分，有缓存（AND）
    }
  }
}
```

### 4.4 分页方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|---|---|---|---|---|
| `from + size` | 每个 Shard 取 top(from+size)，协调节点合并 | 简单 | 深分页性能差，最多 10000 条 | 浅分页（< 1000） |
| `search_after` | 基于上一页最后记录的排序值翻页 | 性能稳定 | 只能下一页，不能跳页 | 深翻页、滚动加载 |
| `scroll` | 快照查询，每次取一批 | 适合大批量导出 | 有状态，占内存，不适合实时 | 全量数据导出 |

```json
// search_after 示例
GET /orders/_search
{
  "size": 10,
  "sort": [
    { "create_time": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2024-01-15T10:30:00", "order_1234"]
}
```

### 4.5 相关性评分（BM25）

ES 默认使用 **BM25** 算法（ES 5.0+ 替代 TF-IDF）：

```
score(D, Q) = Σ IDF(qi) * TF(qi, D) * (k1 + 1) / (TF(qi,D) + k1*(1 - b + b*|D|/avgdl))

IDF：逆文档频率（包含该词的文档越少，IDF 越高）
TF ：词频（BM25 对高词频有饱和处理，避免过度权重）
k1 ：词频饱和参数（默认 1.2）
b  ：字段长度归一化（默认 0.75）
```

---
## 五、聚合分析

### 5.1 三大聚合类型

| 类型 | 说明 | 示例 |
|---|---|---|
| **Bucket** | 分桶（按条件分组） | terms、range、date_histogram、geo_distance |
| **Metric** | 指标计算（数值统计） | avg、sum、min、max、cardinality、percentiles |
| **Pipeline** | 基于其他聚合结果再聚合 | moving_avg、bucket_sort、derivative |

### 5.2 常用聚合示例

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    // 按城市分桶
    "by_city": {
      "terms": { "field": "city", "size": 10 },
      "aggs": {
        // 每个城市的平均订单金额
        "avg_amount": { "avg": { "field": "amount" } },
        // 每个城市的订单总额
        "total_amount": { "sum": { "field": "amount" } }
      }
    },
    // 按时间分桶（按月统计）
    "by_month": {
      "date_histogram": {
        "field": "create_time",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      },
      "aggs": {
        "order_count": { "value_count": { "field": "_id" } }
      }
    },
    // 去重计数（类似 COUNT DISTINCT）
    "unique_users": {
      "cardinality": { "field": "user_id" }
    }
  }
}
```

### 5.3 聚合性能注意事项

- **`cardinality` 是近似值**：基于 HyperLogLog 算法，误差约 0.5%~5%
- **大基数 terms 聚合**：`size` 越大越慢；生产环境避免 `size: 0`（无限制）
- **`fielddata`**：text 字段聚合需开启 fielddata（非常耗内存），应改用 `keyword` 字段
- **Index Sorting**：建索引时设置排序，可加速 date_histogram 等聚合

---

## 六、写入原理与性能调优

### 6.1 写入流程详解

```
1. 客户端请求 → Coordinating Node
2. 路由到对应 Primary Shard
3. Primary Shard：
   a. 写入 In-Memory Buffer（同时写入 Translog）
   b. refresh（默认 1s）→ 生成 Segment，数据可搜索
   c. flush（定期）→ Segment 持久化磁盘，清空 Translog
4. 同步到 Replica Shard（并行）
5. 返回写入成功
```

### 6.2 写入性能调优

```json
// 批量写入（Bulk API，生产必用）
POST /_bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "title": "iPhone 15", "price": 5999 }
{ "index": { "_index": "products", "_id": "2" } }
{ "title": "华为 Mate60", "price": 6999 }
```

**关键优化参数**：

```json
// 索引级别设置（写入期间临时调整）
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "-1",          // 关闭自动 refresh（批量写完后手动 refresh）
    "number_of_replicas": 0,           // 批量写入期间关闭副本
    "translog.durability": "async",    // 异步 translog（牺牲少量可靠性换性能）
    "translog.flush_threshold_size": "1gb"
  }
}

// 写完后恢复
PUT /my_index/_settings
{
  "index": {
    "refresh_interval": "1s",
    "number_of_replicas": 1
  }
}
```

**Bulk 写入建议**：
- 单次 Bulk 请求大小：**5~15 MB**（不是条数）
- 线程数与 Bulk 大小需测试调整
- 使用 `_id` 为顺序递增值可提升写入性能（减少随机寻址）

### 6.3 Merge 调优

```json
// 强制合并（将所有 Segment 合并为 1 个，适合只读索引）
POST /my_index/_forcemerge?max_num_segments=1
// ⚠️ 生产活跃索引不要执行，会占用大量 I/O
```

---
## 七、集群架构与高可用

### 7.1 分片规划原则

| 原则 | 说明 |
|---|---|
| 主分片数量不可更改 | 创建索引后 primary shard 数固定（副本可动态调整） |
| 单个分片大小 | 建议 **10~50 GB**，日志类可到 50GB，搜索类建议 20GB 以内 |
| 分片总数 | 不超过节点数的 20 倍 |
| 避免过多小分片 | 每个分片都有 Lucene 开销，过多小分片浪费资源 |

```json
// 创建索引时指定分片数
PUT /my_index
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}
```

### 7.2 集群健康状态

| 状态 | 含义 | 处理 |
|---|---|---|
| 🟢 **green** | 所有主分片和副本分片正常 | — |
| 🟡 **yellow** | 主分片正常，部分副本未分配 | 检查节点数量是否不足 |
| 🔴 **red** | 部分主分片不可用 | 紧急处理，数据可能丢失 |

```bash
# 查看集群健康
GET /_cluster/health

# 查看未分配分片原因
GET /_cluster/allocation/explain
```

### 7.3 脑裂问题与 minimum_master_nodes

```yaml
# ES 7.0+ 使用新的选举机制（zen2），自动处理脑裂
# ES 6.x 及以下需手动配置：
discovery.zen.minimum_master_nodes: (master_eligible_nodes / 2) + 1
# 3 个 Master 候选节点：minimum_master_nodes = 2
```

### 7.4 索引生命周期管理（ILM）

```json
// 创建 ILM 策略（适合日志类索引）
PUT /_ilm/policy/log_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": { "max_size": "50gb", "max_age": "1d" }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": { "max_num_segments": 1 },
          "shrink": { "number_of_shards": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": { "freeze": {} }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

---

## 八、Python 实战（elasticsearch-py）

### 8.1 连接与基础操作

```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk

es = Elasticsearch(
    hosts=["http://localhost:9200"],
    http_auth=("user", "password"),
    timeout=30,
    max_retries=3,
    retry_on_timeout=True,
)

# 创建索引
es.indices.create(
    index="products",
    body={
        "settings": {
            "number_of_shards": 3,
            "number_of_replicas": 1,
            "analysis": {
                "analyzer": {
                    "ik_analyzer": {
                        "type": "custom",
                        "tokenizer": "ik_max_word"
                    }
                }
            }
        },
        "mappings": {
            "dynamic": "strict",
            "properties": {
                "id":    {"type": "keyword"},
                "title": {"type": "text", "analyzer": "ik_analyzer",
                          "fields": {"keyword": {"type": "keyword"}}},
                "price": {"type": "double"},
                "tags":  {"type": "keyword"},
                "create_time": {"type": "date", "format": "yyyy-MM-dd HH:mm:ss"}
            }
        }
    },
    ignore=400  # 忽略索引已存在错误
)
```

### 8.2 批量写入

```python
def bulk_index(docs: list[dict], index: str):
    actions = [
        {
            "_index": index,
            "_id": doc["id"],
            "_source": doc
        }
        for doc in docs
    ]
    success, errors = bulk(es, actions, chunk_size=500, request_timeout=60)
    print(f"成功: {success}, 失败: {len(errors)}")
    return errors
```
### 8.3 复杂搜索查询

```python
def search_products(keyword: str, min_price: float = None,
                    max_price: float = None, page: int = 1, size: int = 10):
    must = []
    filters = []

    if keyword:
        must.append({"match": {"title": keyword}})

    if min_price is not None or max_price is not None:
        price_range = {}
        if min_price is not None:
            price_range["gte"] = min_price
        if max_price is not None:
            price_range["lte"] = max_price
        filters.append({"range": {"price": price_range}})

    body = {
        "query": {
            "bool": {
                "must": must or [{"match_all": {}}],
                "filter": filters
            }
        },
        "from": (page - 1) * size,
        "size": size,
        "sort": [{"_score": "desc"}, {"create_time": "desc"}],
        "highlight": {
            "fields": {
                "title": {
                    "pre_tags": ["<em>"],
                    "post_tags": ["</em>"]
                }
            }
        }
    }

    result = es.search(index="products", body=body)
    hits = result["hits"]
    return {
        "total": hits["total"]["value"],
        "items": [
            {**h["_source"], "highlight": h.get("highlight", {})}
            for h in hits["hits"]
        ]
    }
```

### 8.4 聚合查询

```python
def sales_stats():
    body = {
        "size": 0,
        "aggs": {
            "by_category": {
                "terms": {"field": "category", "size": 20},
                "aggs": {
                    "total_sales": {"sum": {"field": "amount"}},
                    "avg_price":   {"avg": {"field": "price"}}
                }
            },
            "monthly_trend": {
                "date_histogram": {
                    "field": "create_time",
                    "calendar_interval": "month",
                    "format": "yyyy-MM"
                },
                "aggs": {
                    "revenue": {"sum": {"field": "amount"}}
                }
            }
        }
    }
    return es.search(index="orders", body=body)
```

### 8.5 Reindex（索引迁移）

```python
from elasticsearch.helpers import reindex

# 新建目标索引（新 Mapping）
es.indices.create(index="products_v2", body={...})

# Reindex
reindex(es, source_index="products", target_index="products_v2")

# 切换别名（零停机）
es.indices.update_aliases(body={
    "actions": [
        {"remove": {"index": "products",    "alias": "products_alias"}},
        {"add":    {"index": "products_v2", "alias": "products_alias"}}
    ]
})
```

---

## 九、实战场景题

### 场景 1：电商商品搜索设计

```
需求：支持关键词搜索、分类过滤、价格排序、高亮显示

Mapping 设计：
  title    → text（ik_max_word）+ keyword（聚合/排序）
  category → keyword
  price    → double
  stock    → integer
  tags     → keyword（数组）
  status   → keyword

查询策略：
  ① must: match title（全文检索，计算评分）
  ② filter: term category / range price / term status=on_sale（精确过滤，走缓存）
  ③ sort: _score desc, price asc（评分优先，同分按价格升序）
  ④ highlight: title 高亮

性能优化：
  ① 合理分片（商品索引 3~5 片）
  ② 冷热数据分离（新品/热销 → hot node）
  ③ 查询缓存：filter 条件命中率高，自动缓存
```

### 场景 2：MySQL 数据同步到 ES

```
方案一：双写（代码层）
  优点：实时、简单
  缺点：耦合业务代码，MySQL 写成功 ES 写失败时数据不一致

方案二：Canal + MQ（推荐）
  Canal 监听 MySQL Binlog
    → 发送到 MQ（Kafka/RabbitMQ）
    → ES 消费者消费并写入 ES
  优点：业务无感知，最终一致性
  缺点：存在同步延迟（秒级）

方案三：Logstash / Debezium
  适合存量数据迁移或简单场景
```

### 场景 3：ES 查询慢如何排查？

```
排查步骤：
  ① 使用 Profile API 分析查询耗时分布
     GET /my_index/_search
     { "profile": true, "query": {...} }

  ② 检查是否深分页（from 过大）→ 改用 search_after

  ③ 检查是否对 text 字段做 term 精确查询（应用 keyword 子字段）

  ④ 检查 wildcard / script 查询（全表扫描，性能极差）

  ⑤ 检查 fielddata 使用（text 字段聚合）

  ⑥ 检查集群状态：节点 CPU/内存/GC / 分片分布是否均衡

  ⑦ 查看慢日志：
     PUT /my_index/_settings
     { "index.search.slowlog.threshold.query.warn": "5s" }
```

---
## 十、高频追问与陷阱题

### Q1：ES 为什么是近实时搜索而不是实时？

> 写入的数据需要经过 **refresh**（默认 1s）生成 Segment 后才能被搜索。  
> 如果需要写入即可见，可以：  
> ① 写入时指定 `refresh=true`（同步 refresh，性能影响大，慎用）  
> ② 调小 `refresh_interval`（如 `100ms`，但会产生更多小 Segment）  
> ③ 业务上接受 1s 延迟（大多数场景可以接受）

---

### Q2：ES 如何保证数据不丢失？

> **Translog 机制**：  
> 数据写入 In-Memory Buffer 的同时写入 Translog（默认每次写入 fsync，`durability=request`）。  
> 节点宕机恢复时，重放 Translog 中未 flush 的操作，确保数据不丢失。  
>
> **副本机制**：  
> 写入成功 = Primary 写入成功 + 等待指定数量副本确认（`wait_for_active_shards`）。  
> 生产建议至少 1 个副本。

---

### Q3：分片数设置多少合适？

> - **无法事后修改**（primary shard 数创建后固定），需提前规划
> - **经验公式**：分片数 ≈ 索引总大小（GB）/ 单分片目标大小（30GB）
> - **节点数限制**：分片数不超过数据节点数 × 3
> - **过多分片的代价**：每个分片都有 Lucene 索引开销，Master 需维护更多元数据
> - **小数据量**：1~2 个分片即可，不要过度设计

---

### Q4：ES 与关系型数据库如何选型？

| 需求 | 选 ES | 选 MySQL |
|---|---|---|
| 全文搜索、模糊搜索 | ✅ | ❌（LIKE 效率极低） |
| 复杂联表查询、事务 | ❌ | ✅ |
| 实时写入后查询 | ❌（1s 延迟） | ✅ |
| 大数据量聚合分析 | ✅ | ❌（性能差） |
| 地理位置搜索 | ✅ | ❌ |
| 数据一致性要求高 | ❌ | ✅ |

> **最佳实践**：MySQL 作为主数据库保证一致性，ES 作为搜索层，通过 Canal/Binlog 同步，读写分离。

---

### Q5：update 和 delete 文档后，旧数据何时真正删除？

> ES 中 Segment **不可变**：
> - **delete**：在 `.del` 文件中标记该 DocID 为删除，查询时过滤，Segment 合并时物理删除
> - **update**：旧文档标记删除 + 新文档写入新 Segment，Merge 后旧 Segment 消失
>
> 因此，频繁更新/删除的索引会积累大量"软删除"文档，需要定期 `_forcemerge` 清理（只读索引）。

---

### Q6：如何实现搜索词高亮？

```json
GET /articles/_search
{
  "query": { "match": { "content": "Elasticsearch" } },
  "highlight": {
    "pre_tags":  ["<span class='highlight'>"],
    "post_tags": ["</span>"],
    "fields": {
      "content": {
        "fragment_size": 150,
        "number_of_fragments": 3
      }
    }
  }
}
```

---

### Q7：如何实现搜索建议（Suggest）？

```json
// Completion Suggester（最快，前缀匹配）
// Mapping 中定义 suggest 字段
"suggest_field": { "type": "completion", "analyzer": "ik_smart" }

// 搜索建议请求
GET /products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "苹果",
      "completion": {
        "field": "suggest_field",
        "size": 5,
        "skip_duplicates": true
      }
    }
  }
}
```

---

## 附录：面试前必看清单

- [ ] 能描述倒排索引的结构（Term Dictionary / Posting List / Term Index）
- [ ] 能解释 Segment 写入流程（Buffer → refresh → Segment → flush → 磁盘）
- [ ] 熟悉 text 和 keyword 的区别及 multi-field 设计
- [ ] 能写出 bool 查询（must / filter / should / must_not）
- [ ] 了解三种分页方案及适用场景（from+size / search_after / scroll）
- [ ] 能解释 nested 与 object 的本质区别
- [ ] 熟悉集群健康状态（green/yellow/red）及分片规划原则
- [ ] 了解写入性能调优手段（refresh_interval / bulk / 副本数）
- [ ] 能描述 ES + MySQL 数据同步方案（Canal + MQ）
- [ ] 了解 ILM 索引生命周期管理

---

*文档版本：Elasticsearch 8.x 适用 ｜ 最后更新：2025 年*
