# TCP & HTTP 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、网络方向面试  
> 覆盖范围：TCP 核心机制 · HTTP/1.1/2/3 · HTTPS · 网络模型 · 实战场景

---

## 目录

1. [网络分层模型](#一网络分层模型)
2. [TCP 核心机制](#二tcp-核心机制)
3. [TCP 三次握手与四次挥手](#三tcp-三次握手与四次挥手)
4. [TCP 可靠性与流量控制](#四tcp-可靠性与流量控制)
5. [HTTP/1.1 核心特性](#五http11-核心特性)
6. [HTTP/2 与 HTTP/3](#六http2-与-http3)
7. [HTTPS 与 TLS](#七https-与-tls)
8. [HTTP 常见状态码与请求方法](#八http-常见状态码与请求方法)
9. [实战场景题](#九实战场景题)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、网络分层模型

### 1.1 OSI 七层 vs TCP/IP 四层

```
OSI 七层模型              TCP/IP 四层模型         典型协议
─────────────────────────────────────────────────────────
7. 应用层  ┐               应用层                HTTP/HTTPS/FTP/DNS/SMTP
6. 表示层  ├──────────────►                       TLS/SSL
5. 会话层  ┘
─────────────────────────────────────────────────────────
4. 传输层  ─────────────── 传输层                 TCP/UDP
─────────────────────────────────────────────────────────
3. 网络层  ─────────────── 网络层（Internet层）   IP/ICMP/ARP
─────────────────────────────────────────────────────────
2. 数据链路层 ┐             网络接口层             Ethernet/WiFi
1. 物理层     ┘
─────────────────────────────────────────────────────────
```

### 1.2 数据封装与解封装

```
发送方（从上到下封装）：
  应用数据
    ↓ + TCP Header（源端口/目标端口/序号/确认号...）
  TCP Segment
    ↓ + IP Header（源IP/目标IP/TTL...）
  IP Packet
    ↓ + MAC Header（源MAC/目标MAC）
  Ethernet Frame
    ↓ 物理传输（比特流）

接收方（从下到上解封装）：
  Ethernet Frame → IP Packet → TCP Segment → 应用数据
```

### 1.3 TCP vs UDP

| 特性 | TCP | UDP |
|---|---|---|
| 连接 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（确认/重传/排序） | 不可靠 |
| 顺序 | 保证顺序 | 不保证 |
| 速度 | 较慢（控制开销） | 快 |
| 头部大小 | 20~60 字节 | 8 字节 |
| 流量控制 | 有（滑动窗口） | 无 |
| 拥塞控制 | 有 | 无 |
| 适用场景 | HTTP/HTTPS/FTP/SMTP | DNS/视频直播/游戏/QUIC |

---

## 二、TCP 核心机制

### 2.1 TCP 报文头

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────┬───────────────────────────────────────────┤
│    源端口（16位）       │          目标端口（16位）                   │
├───────────────────────────────────────────────────────────────────┤
│                    序列号（Sequence Number，32位）                   │
├───────────────────────────────────────────────────────────────────┤
│                    确认号（Acknowledgment Number，32位）             │
├─────┬──────┬─┬─┬─┬─┬─┬─┬───────────────────────────────────────┤
│数据偏│  保留 │U│A│P│R│S│F│              窗口大小（16位）             │
│移   │      │R│C│S│S│Y│I│                                         │
│     │      │G│K│H│T│N│N│                                         │
├─────┴──────┴─┴─┴─┴─┴─┴─┴───────────────────────────────────────┤
│             校验和（16位）          │     紧急指针（16位）            │
├───────────────────────────────────────────────────────────────────┤
│                         选项（可变长）                               │
└───────────────────────────────────────────────────────────────────┘

关键字段：
  序列号（Seq）：本次发送数据的第一个字节编号
  确认号（Ack）：期望收到对方下一个字节的编号（= 已收到的最大 Seq + 1）
  SYN：建立连接
  FIN：关闭连接
  ACK：确认
  RST：重置连接（异常关闭）
  窗口大小：接收缓冲区剩余空间（流量控制）
```

### 2.2 端口范围

```
0     ~ 1023  ：知名端口（Well-Known），需 root 权限（HTTP:80, HTTPS:443, SSH:22）
1024  ~ 49151 ：注册端口（Registered），如 MySQL:3306, Redis:6379
49152 ~ 65535 ：动态/私有端口（客户端临时端口）
```

---
## 三、TCP 三次握手与四次挥手

### 3.1 三次握手

```
Client                                Server
  │                                     │
  │──── SYN, Seq=x ──────────────────►  │  第一次：Client 发 SYN，进入 SYN_SENT
  │                                     │
  │  ◄── SYN+ACK, Seq=y, Ack=x+1 ────  │  第二次：Server 发 SYN+ACK，进入 SYN_RCVD
  │                                     │
  │──── ACK, Seq=x+1, Ack=y+1 ──────►  │  第三次：Client 发 ACK，进入 ESTABLISHED
  │                                     │  Server 收到后进入 ESTABLISHED
  │         （连接建立，开始传输）          │
```

**为什么是三次，不是两次？**
> - 两次握手无法确认 Client 的接收能力，也无法防止历史连接（旧 SYN）干扰
> - 三次握手确保双方的「发送」和「接收」能力都得到验证
> - 第三次 ACK 让 Server 确认 Client 能正常接收（Client 的接收窗口正常）

**为什么不是四次？**
> Server 可以将 SYN 和 ACK 合并为一个报文，因此三次即可完成。

### 3.2 四次挥手

```
Client                                Server
  │                                     │
  │──── FIN, Seq=u ──────────────────►  │  第一次：Client 主动关闭，进入 FIN_WAIT_1
  │                                     │
  │  ◄── ACK, Ack=u+1 ───────────────  │  第二次：Server 确认，进入 CLOSE_WAIT
  │         （Client 进入 FIN_WAIT_2）   │  （Server 可能还有数据要发）
  │                                     │
  │  ◄── FIN, Seq=v ──────────────────  │  第三次：Server 发完数据后发 FIN，进入 LAST_ACK
  │                                     │
  │──── ACK, Ack=v+1 ────────────────►  │  第四次：Client 发 ACK，进入 TIME_WAIT
  │         （等待 2MSL 后关闭）          │  Server 收到后进入 CLOSED
```

**TIME_WAIT 状态（2MSL）的作用**：
> 1. **确保最后一个 ACK 能到达 Server**：若 ACK 丢失，Server 会重发 FIN，Client 需要能重新响应
> 2. **让本次连接的历史报文在网络中消失**：MSL（Maximum Segment Lifetime）= 2 分钟，等待 2MSL 确保旧报文过期

**为什么挥手是四次，握手是三次？**
> - 握手时 Server 的 SYN 和 ACK 可以合并发送
> - 挥手时 Server 收到 FIN 后可能还有数据没发完，需要先 ACK，等数据发完再 FIN

### 3.3 连接状态机

```
                   ┌────────────────────────────────────────┐
                   │            TCP 状态转换                  │
                   └────────────────────────────────────────┘

CLOSED ──SYN发出──► SYN_SENT ──收到SYN+ACK，发ACK──► ESTABLISHED
                                                         │
CLOSED ◄──收到SYN，发SYN+ACK──► SYN_RCVD ──收到ACK──►  │
                                                         │
                              ESTABLISHED ◄──────────────┘
                                   │
                        主动关闭方（发FIN）
                                   ▼
                              FIN_WAIT_1
                                   │ 收到ACK
                                   ▼
                              FIN_WAIT_2
                                   │ 收到FIN，发ACK
                                   ▼
                              TIME_WAIT ──等待2MSL──► CLOSED

                        被动关闭方（收到FIN）
                              ESTABLISHED
                                   │ 收到FIN，发ACK
                                   ▼
                              CLOSE_WAIT
                                   │ 发FIN
                                   ▼
                              LAST_ACK ──收到ACK──► CLOSED
```

---
## 四、TCP 可靠性与流量控制

### 4.1 可靠传输：序号 + 确认 + 重传

```
发送方维护：
  SND.UNA：已发送但未确认的最小序号
  SND.NXT：下一个要发送的序号

接收方维护：
  RCV.NXT：期望收到的下一个序号

超时重传（RTO）：
  发送报文后启动定时器，超时未收到 ACK 则重传
  RTO 根据 RTT 动态计算（Karn 算法）

快速重传：
  连续收到 3 个重复 ACK → 立即重传，不等超时
  （说明中间某个报文丢失，但后续报文已到达接收方）
```

### 4.2 滑动窗口（流量控制）

```
发送窗口 = min(接收窗口, 拥塞窗口)

接收方通告窗口（rwnd）：
  接收缓冲区剩余空间，放在 TCP 头的窗口字段
  接收方处理慢 → 窗口缩小 → 发送方降速

  Sender                      Receiver
  │  ┌─────────────────────┐    │
  │  │已发已确│已发未确│可发│  │  → 接收方窗口决定"可发"的右边界
  │  └─────────────────────┘    │
  │        ←── rwnd ──►          │

零窗口问题：
  接收方窗口为 0 → 发送方停止发送
  发送方定期发送窗口探测报文（Window Probe），等待窗口恢复
```

### 4.3 拥塞控制

```
拥塞窗口（cwnd）：发送方维护，根据网络状况动态调整

四个阶段：

1. 慢启动（Slow Start）：
   初始 cwnd=1 MSS
   每收到一个 ACK，cwnd += 1 MSS（指数增长）
   cwnd 达到 ssthresh（慢启动阈值）时，进入拥塞避免

2. 拥塞避免（Congestion Avoidance）：
   每个 RTT，cwnd += 1 MSS（线性增长）
   检测到拥塞（超时 or 3个重复ACK）时触发处理

3. 拥塞发生（超时）：
   ssthresh = cwnd / 2
   cwnd = 1（回到慢启动）

4. 快速恢复（Fast Recovery，Reno 算法）：
   收到 3 个重复 ACK（快速重传触发）
   ssthresh = cwnd / 2
   cwnd = ssthresh + 3（不回到1，进入拥塞避免）
```

```
cwnd
 ^
 │                            *
 │                          *
 │         ssthresh ──────*────────────────────
 │                      *  \  线性增长
 │                    *     \
 │                  *        \  3重复ACK → 快速恢复
 │              指数  \
 │          慢启动     \
 │────────────────────────────────────────► time
```

### 4.4 Nagle 算法 与 TCP_NODELAY

```
Nagle 算法（默认开启）：
  目的：合并小包，减少网络中的碎片报文
  规则：只有在以下情况才立即发送：
    ① 数据量达到 MSS（最大报文段）
    ② 或所有已发报文都已 ACK（无未确认数据）
  否则：缓存等待，合并后发送

TCP_NODELAY（禁用 Nagle）：
  适用场景：需要低延迟的应用（SSH/游戏/实时推送）
  代价：可能发送大量小包，增加网络负担

# Python 设置 TCP_NODELAY
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

---
## 五、HTTP/1.1 核心特性

### 5.1 HTTP/1.0 vs HTTP/1.1

| 特性 | HTTP/1.0 | HTTP/1.1 |
|---|---|---|
| 持久连接 | 默认短连接（每次新建 TCP） | 默认长连接（`Connection: keep-alive`） |
| 管道化（Pipelining） | 不支持 | 支持（但有队头阻塞问题） |
| Host 头 | 非必须 | **必须**（支持虚拟主机） |
| 缓存控制 | `Expires` | `Cache-Control`（更灵活） |
| 断点续传 | 不支持 | 支持（`Range` 请求） |
| 分块传输 | 不支持 | 支持（`Transfer-Encoding: chunked`） |

### 5.2 HTTP 请求/响应结构

```
HTTP 请求：
  ┌─────────────────────────────────────────────────────┐
  │ GET /api/users?page=1 HTTP/1.1                       │  请求行
  │ Host: api.example.com                                │
  │ User-Agent: Mozilla/5.0                              │  请求头
  │ Accept: application/json                             │
  │ Authorization: Bearer eyJhbGc...                     │
  │ Content-Type: application/json                       │
  │ Connection: keep-alive                               │
  │                                                     │  空行（CRLF）
  │ {"key": "value"}                                     │  请求体（可选）
  └─────────────────────────────────────────────────────┘

HTTP 响应：
  ┌─────────────────────────────────────────────────────┐
  │ HTTP/1.1 200 OK                                      │  状态行
  │ Content-Type: application/json; charset=utf-8        │
  │ Content-Length: 1234                                 │  响应头
  │ Cache-Control: max-age=3600                          │
  │ Set-Cookie: session=abc; HttpOnly; Secure            │
  │                                                     │  空行
  │ {"users": [...]}                                     │  响应体
  └─────────────────────────────────────────────────────┘
```

### 5.3 HTTP 持久连接与队头阻塞

```
HTTP/1.1 管道化（Pipelining）：
  Client 可以不等响应就发送多个请求
  Server 必须按请求顺序返回响应

队头阻塞（Head-of-Line Blocking）：
  请求1 处理慢 → 请求2/3 必须等待
  ┌──────┐ ┌──────┐ ┌──────┐
  │  req1 │ │ req2 │ │ req3 │  Client 发送
  └──────┘ └──────┘ └──────┘
  ┌──────────────────┐ ┌──┐ ┌──┐
  │     resp1（慢）   │ │r2│ │r3│  Server 响应（r2/r3 被阻塞）
  └──────────────────┘ └──┘ └──┘

解决方案：
  ① 多开 TCP 连接（浏览器默认 6 个并发连接）
  ② 升级到 HTTP/2（多路复用）
```

### 5.4 HTTP 缓存机制

```
强缓存（不请求服务器）：
  Cache-Control: max-age=3600   ← 优先（秒数）
  Expires: Wed, 21 Oct 2025 07:28:00 GMT  ← 旧版（绝对时间）

协商缓存（询问服务器是否过期）：
  Last-Modified / If-Modified-Since：基于修改时间
  ETag / If-None-Match：基于内容指纹（更精确，推荐）

  ① 客户端发送 If-None-Match: "etag-value"
  ② 服务器比较 ETag：
     未变化 → 返回 304 Not Modified（无响应体，节省带宽）
     已变化 → 返回 200 + 新内容

缓存策略选择：
  静态资源（JS/CSS/图片）：强缓存 + 文件名加 hash（内容变则 URL 变）
  HTML：协商缓存（需要感知内容变化）
  API：通常不缓存（no-cache/no-store）
```

### 5.5 Cookie 与 Session

```
Cookie（存在客户端）：
  Set-Cookie: session_id=abc123; 
              Max-Age=3600;       # 有效期
              Path=/;             # 路径
              Domain=.example.com;# 域名（支持子域）
              HttpOnly;           # 禁止 JS 读取（防 XSS）
              Secure;             # 只通过 HTTPS 发送
              SameSite=Strict;    # 防 CSRF

Session（存在服务端）：
  服务端存储用户状态，Cookie 只存 session_id
  问题：分布式系统中 Session 共享（需 Redis 等集中存储）

JWT（JSON Web Token）：
  无状态（服务端无需存储），适合分布式
  Header.Payload.Signature（Base64 + HMAC/RSA 签名）
  问题：无法主动失效（需要黑名单机制）
```

---
## 六、HTTP/2 与 HTTP/3

### 6.1 HTTP/2 核心特性

```
1. 二进制分帧（Binary Framing）：
   HTTP/1.1 是文本协议
   HTTP/2 是二进制协议，更高效解析，更少歧义

2. 多路复用（Multiplexing）：
   一个 TCP 连接上并发多个请求/响应（流 Stream）
   每个流有唯一 Stream ID
   彻底解决 HTTP 层的队头阻塞

   Stream 1: ─── req1 ──────────── resp1 ───►
   Stream 3: ─── req2 ─── resp2 ──────────►
   Stream 5: ── req3 ── resp3 ─────────────►
   （同一 TCP 连接，交错传输）

3. 头部压缩（HPACK）：
   维护动态表，相同头部不重复传输
   大幅减少重复头部（如 Cookie/User-Agent）的带宽

4. 服务器推送（Server Push）：
   Server 主动推送客户端可能需要的资源
   如请求 HTML 时，Server 同时推送 CSS/JS

5. 流优先级：
   客户端可以设置流的权重和依赖关系
```

### 6.2 HTTP/2 的问题

```
TCP 层的队头阻塞：
  HTTP/2 复用在一个 TCP 连接上
  TCP 层的丢包 → 整个 TCP 窗口等待重传
  → 所有 HTTP/2 流都被阻塞（比 HTTP/1.1 多连接更差）

在高丢包率网络下，HTTP/2 性能可能不如 HTTP/1.1
```

### 6.3 HTTP/3 与 QUIC

```
HTTP/3 底层协议：QUIC（Quick UDP Internet Connections）
  基于 UDP 实现可靠传输（不是 TCP）

QUIC 解决的问题：
  1. 消除 TCP 队头阻塞：每个流独立，一个流丢包不影响其他流
  2. 0-RTT / 1-RTT 连接建立：缓存密钥后再次连接无需握手
  3. 连接迁移：基于 Connection ID，切换 IP/端口不断连
     （移动网络从 WiFi 切 4G 时连接不中断）
  4. 内置 TLS 1.3：QUIC 本身就是加密的

HTTP/1.1 vs HTTP/2 vs HTTP/3：

  HTTP/1.1：文本，TCP，队头阻塞，多连接规避
  HTTP/2：  二进制，TCP，多路复用，TCP 层阻塞
  HTTP/3：  二进制，QUIC(UDP)，流独立，0-RTT
```

---
## 七、HTTPS 与 TLS

### 7.1 TLS 握手过程（TLS 1.2）

```
Client                                  Server
  │                                        │
  │── ClientHello ─────────────────────►  │
  │   (支持的TLS版本/加密套件/随机数C)       │
  │                                        │
  │  ◄── ServerHello ─────────────────────│
  │   (选定TLS版本/加密套件/随机数S)         │
  │  ◄── Certificate（服务器证书）──────── │
  │  ◄── ServerKeyExchange（DH参数）────── │
  │  ◄── ServerHelloDone ──────────────── │
  │                                        │
  │── ClientKeyExchange ────────────────► │
  │   (客户端DH公钥 or 预主密钥)             │
  │                                        │
  │  [双方计算出相同的 Master Secret]        │
  │  [派生出对称加密密钥和MAC密钥]            │
  │                                        │
  │── ChangeCipherSpec ─────────────────► │
  │── Finished ─────────────────────────► │
  │                                        │
  │  ◄── ChangeCipherSpec ───────────────  │
  │  ◄── Finished ──────────────────────  │
  │                                        │
  │  ════════════ 加密通信 ════════════════ │

TLS 1.2：2 RTT 握手
TLS 1.3：1 RTT 握手（简化了握手流程）
TLS 1.3 0-RTT：会话恢复时 0 RTT（但有重放攻击风险）
```

### 7.2 证书与 CA 信任链

```
数字证书包含：
  - 域名（Subject）
  - 公钥
  - 颁发机构（CA）
  - 有效期
  - CA 的数字签名

信任链：
  Root CA（根证书，内置在系统）
    └── Intermediate CA（中间证书）
          └── End-Entity Certificate（服务器证书）

验证过程：
  ① 用 Root CA 公钥验证 Intermediate CA 证书签名
  ② 用 Intermediate CA 公钥验证服务器证书签名
  ③ 验证域名匹配、证书未过期、未被吊销（OCSP）
```

### 7.3 对称加密 vs 非对称加密

```
非对称加密（RSA/ECDSA/ECDH）：
  公钥加密，私钥解密
  优点：密钥分发安全
  缺点：计算慢（比对称慢 1000x）
  TLS 中用于：密钥交换 + 数字签名

对称加密（AES-GCM/ChaCha20）：
  同一密钥加密和解密
  优点：极快
  缺点：密钥分发困难
  TLS 中用于：实际数据加密

TLS 结合两者：
  非对称 → 安全协商出对称密钥（会话密钥）
  对称   → 加密后续所有通信数据
```

---

## 八、HTTP 常见状态码与请求方法

### 8.1 状态码

| 状态码 | 含义 | 场景 |
|---|---|---|
| **1xx 信息** | | |
| 100 Continue | 继续发送 | 上传大文件前预检 |
| 101 Switching Protocols | 协议升级 | HTTP → WebSocket |
| **2xx 成功** | | |
| 200 OK | 请求成功 | 通用成功 |
| 201 Created | 创建成功 | POST 创建资源 |
| 204 No Content | 无响应体 | DELETE 成功 |
| 206 Partial Content | 部分内容 | 断点续传 |
| **3xx 重定向** | | |
| 301 Moved Permanently | 永久重定向 | 域名迁移 |
| 302 Found | 临时重定向 | 登录跳转 |
| 304 Not Modified | 协商缓存命中 | 资源未变化 |
| 307 Temporary Redirect | 临时重定向（保持方法） | 与302区别：不改变POST→GET |
| **4xx 客户端错误** | | |
| 400 Bad Request | 请求格式错误 | 参数错误 |
| 401 Unauthorized | 未认证 | 需要登录 |
| 403 Forbidden | 无权限 | 已登录但无权访问 |
| 404 Not Found | 资源不存在 | URL 错误 |
| 405 Method Not Allowed | 方法不允许 | 用 GET 调用只支持 POST 的接口 |
| 409 Conflict | 冲突 | 重复创建 |
| 429 Too Many Requests | 限流 | 触发速率限制 |
| **5xx 服务端错误** | | |
| 500 Internal Server Error | 服务器内部错误 | 未知异常 |
| 502 Bad Gateway | 网关错误 | 上游服务不可用 |
| 503 Service Unavailable | 服务不可用 | 维护/过载 |
| 504 Gateway Timeout | 网关超时 | 上游服务超时 |

### 8.2 HTTP 请求方法

| 方法 | 语义 | 幂等 | 安全 | 有请求体 |
|---|---|---|---|---|
| GET | 获取资源 | ✅ | ✅ | 通常无 |
| POST | 创建资源/提交数据 | ❌ | ❌ | ✅ |
| PUT | 全量更新/替换 | ✅ | ❌ | ✅ |
| PATCH | 局部更新 | ❌ | ❌ | ✅ |
| DELETE | 删除资源 | ✅ | ❌ | 通常无 |
| HEAD | 获取响应头（不含响应体） | ✅ | ✅ | 无 |
| OPTIONS | 预检请求/查询支持的方法 | ✅ | ✅ | 无 |

> **幂等**：多次执行相同操作，结果相同  
> **安全**：不会修改服务器状态
### 8.3 CORS 跨域

```
跨域请求（Origin 不同）：

简单请求（GET/POST + 普通头部）：
  直接发送，浏览器检查响应头
  Access-Control-Allow-Origin: https://client.com

预检请求（复杂请求，如 PUT/DELETE/自定义头）：
  先发 OPTIONS 请求
  Server 响应：
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Methods: GET, POST, PUT, DELETE
    Access-Control-Allow-Headers: Content-Type, Authorization
    Access-Control-Max-Age: 86400   ← 预检结果缓存时间
  通过后再发实际请求

常见配置（FastAPI）：
  from fastapi.middleware.cors import CORSMiddleware
  app.add_middleware(
      CORSMiddleware,
      allow_origins=["https://client.com"],
      allow_methods=["*"],
      allow_headers=["*"],
  )
```

---

## 九、实战场景题

### 场景 1：从输入 URL 到页面显示，发生了什么？

```
1. DNS 解析
   浏览器缓存 → 系统缓存 → 路由器缓存 → ISP DNS → 递归查询
   www.example.com → 203.0.113.1

2. TCP 三次握手（建立连接）
   Client → SYN → Server
   Client ← SYN+ACK ← Server
   Client → ACK → Server

3. TLS 握手（HTTPS）
   TLS 1.3：1 RTT，协商加密套件，交换证书，派生会话密钥

4. HTTP 请求
   GET / HTTP/1.1
   Host: www.example.com

5. 服务器处理 & 响应
   负载均衡 → Web 服务器 → 应用服务器 → 数据库
   返回 HTML（200 OK）

6. 浏览器渲染
   解析 HTML → 构建 DOM
   解析 CSS → 构建 CSSOM
   合并 DOM + CSSOM → Render Tree
   Layout（布局） → Paint（绘制） → Composite（合成层）

7. 加载子资源
   并发请求 CSS/JS/图片
   执行 JS（可能修改 DOM，触发重排/重绘）
```

### 场景 2：HTTP 长轮询、SSE、WebSocket 对比

```
长轮询（Long Polling）：
  Client 发请求 → Server 不立即响应，等待有数据/超时才响应
  → Client 收到响应后立即再发下一个请求
  问题：延迟高，连接频繁建立

SSE（Server-Sent Events）：
  HTTP 持久连接，Server 单向推送事件流
  Content-Type: text/event-stream
  适用：实时日志/通知推送（单向）

WebSocket：
  HTTP Upgrade 升级为 WebSocket 协议（全双工）
  握手：
    GET /ws HTTP/1.1
    Upgrade: websocket
    Connection: Upgrade
    Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  适用：在线聊天/实时游戏/协作编辑（双向）

# Python WebSocket（websockets 库）
import asyncio
import websockets

async def handler(websocket):
    async for message in websocket:
        await websocket.send(f"Echo: {message}")

asyncio.run(websockets.serve(handler, "localhost", 8765))
```

---
## 十、高频追问与陷阱题

### Q1：TCP 为什么需要三次握手，两次不行吗？

> **两次握手的问题**：
> 1. **历史连接（旧 SYN）干扰**：网络中可能存在延迟的旧 SYN 报文，Server 接受后发 SYN+ACK，若 Client 认为这是无效连接会直接丢弃，导致 Server 浪费资源一直等待
> 2. **无法验证双向通信**：两次握手只验证了 Server 能收到 Client 的发送，无法验证 Client 能收到 Server 的发送
>
> **三次握手**：Client 发送 ACK，确认 Client 能收到 Server 的数据，双向通信验证完毕，且能过滤历史 SYN。

---

### Q2：TIME_WAIT 过多怎么办？

> **原因**：主动关闭方（通常是 Client）会进入 TIME_WAIT，等待 2MSL（约 60s）
>
> **影响**：大量 TIME_WAIT 占用端口资源（临时端口耗尽）
>
> **解决方案**：
> ```
> # Linux 内核参数调优
> net.ipv4.tcp_tw_reuse = 1        # 允许将 TIME_WAIT 连接用于新连接（需开启 tcp_timestamps）
> net.ipv4.tcp_timestamps = 1      # 启用时间戳（reuse 的依赖）
> net.ipv4.tcp_fin_timeout = 30    # 缩短 FIN_WAIT_2 超时时间
> net.ipv4.ip_local_port_range = 1024 65535  # 扩大临时端口范围
> ```
> - 服务端主动关闭（如 HTTP Server 读取完 body 后关闭）→ 改为 Client 主动关闭
> - 使用 HTTP 长连接（减少连接关闭次数）
> - 使用连接池

---

### Q3：HTTP 与 HTTPS 的区别？HTTPS 一定安全吗？

> **区别**：
> - HTTP 明文传输，HTTPS = HTTP + TLS，数据加密传输
> - HTTP 默认 80 端口，HTTPS 默认 443 端口
> - HTTPS 有身份验证（证书），防止中间人攻击
>
> **HTTPS 不能保证的**：
> - 服务端程序漏洞（SQL 注入/XSS 仍然可能）
> - 证书被吊销但客户端未检查（OCSP）
> - 客户端信任了伪造的根证书（证书劫持）
> - 仅加密传输，不保证内容本身的合法性

---

### Q4：GET 和 POST 的区别？

> | 维度 | GET | POST |
> |---|---|---|
> | 语义 | 获取数据 | 提交/创建数据 |
> | 参数位置 | URL（Query String） | 请求体（Body） |
> | 长度限制 | URL 有限制（~2KB~8KB，浏览器/服务器决定） | 无限制（服务器可配置） |
> | 幂等性 | ✅ 幂等 | ❌ 非幂等 |
> | 安全性 | URL 参数可见（历史记录/日志） | Body 不可见（但 HTTPS 都加密） |
> | 缓存 | 可缓存 | 通常不缓存 |
>
> ⚠️ **常见误区**：GET 并不比 POST 更不安全，HTTPS 下两者都加密。GET 的"不安全"是指参数在 URL 中可见（日志/浏览器历史），而非传输层安全。

---

### Q5：HTTP 长连接（Keep-Alive）的原理？

> ```
> Connection: keep-alive    ← 请求头
> Keep-Alive: timeout=60, max=100  ← 60s 无活动关闭，最多100个请求
> ```
>
> - HTTP/1.1 默认开启长连接，复用 TCP 连接
> - 避免每次请求都进行 TCP 握手（3 RTT）+ TLS 握手（1~2 RTT）
> - 服务端可配置超时时间，防止连接无限占用
>
> **注意与 HTTP/2 的区别**：
> - HTTP/1.1 长连接：串行复用（一次一个请求，或 pipelining 但有队头阻塞）
> - HTTP/2 多路复用：真正并发（一个连接上同时多个请求/响应）

---

### Q6：为什么 TCP 挥手是四次，握手是三次？

> **握手三次**：Server 可以将 ACK 和 SYN 合并成一个报文（SYN+ACK）
>
> **挥手四次**：
> - Client 发 FIN 表示：Client 没有数据要发了（**但仍可接收**）
> - Server 发 ACK 确认收到 FIN，此时 Server **可能还有数据未发完**
> - Server 等数据发完后，才能发 FIN
> - 因此 Server 的 ACK 和 FIN 不能合并（中间有数据传输的可能）
>
> 如果 Server 没有数据要发，也可以合并 ACK+FIN，变成三次挥手（但实际不常见）。

---

## 附录：面试前必看清单

- [ ] 能画出 TCP 三次握手和四次挥手的状态图
- [ ] 理解 TIME_WAIT 的作用及 2MSL 等待原因
- [ ] 掌握滑动窗口（流量控制）和拥塞控制四阶段
- [ ] 能说出 HTTP/1.1、HTTP/2、HTTP/3 的核心区别
- [ ] 理解 TLS 握手过程（密钥交换 + 对称加密）
- [ ] 熟悉 HTTP 状态码（2xx/3xx/4xx/5xx 各类含义）
- [ ] 能区分 GET/POST 的语义、幂等性差异
- [ ] 了解 HTTP 缓存（强缓存 vs 协商缓存）
- [ ] 理解 CORS 跨域及预检请求机制
- [ ] 能描述「从输入 URL 到页面显示」的完整流程

---

*文档版本：HTTP/3 / TLS 1.3 适用 ｜ 最后更新：2025 年*
