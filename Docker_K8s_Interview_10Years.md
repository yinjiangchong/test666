# Docker & Kubernetes 高级面试文档（10年经验）

> 适用场景：P7/P8 级别后端开发、架构师、云原生方向面试  
> 覆盖范围：Docker 原理 · 镜像优化 · K8s 核心架构 · 调度机制 · 网络存储 · 生产实践

---

## 目录

1. [Docker 核心原理](#一docker-核心原理)
2. [Dockerfile 与镜像优化](#二dockerfile-与镜像优化)
3. [Docker 网络与存储](#三docker-网络与存储)
4. [Kubernetes 核心架构](#四kubernetes-核心架构)
5. [Pod 与工作负载](#五pod-与工作负载)
6. [Service 与网络](#六service-与网络)
7. [存储（PV/PVC/StorageClass）](#七存储pvpvcstorageclass)
8. [调度、资源管理与自动伸缩](#八调度资源管理与自动伸缩)
9. [生产实践与运维](#九生产实践与运维)
10. [高频追问与陷阱题](#十高频追问与陷阱题)

---

## 一、Docker 核心原理

### 1.1 容器 vs 虚拟机

```
┌─────────────────────────┐    ┌─────────────────────────┐
│       虚拟机（VM）        │    │       容器（Container）   │
├─────────────────────────┤    ├─────────────────────────┤
│  App A  │  App B        │    │  App A  │  App B        │
│─────────│───────        │    │─────────│───────        │
│ Guest OS│ Guest OS      │    │ 容器运行时（Docker）       │
│─────────────────────────│    │─────────────────────────│
│       Hypervisor        │    │       Host OS Kernel    │
│─────────────────────────│    │─────────────────────────│
│       Host OS           │    │       Hardware          │
│─────────────────────────│    └─────────────────────────┘
│       Hardware          │
└─────────────────────────┘

VM：完整的操作系统隔离，启动慢（分钟级），资源开销大
容器：共享宿主机内核，启动快（秒级），轻量，隔离弱于 VM
```

| 特性 | 容器 | 虚拟机 |
|---|---|---|
| 启动时间 | 秒级 | 分钟级 |
| 磁盘占用 | MB 级 | GB 级 |
| 内核隔离 | 共享宿主机内核 | 独立内核 |
| 安全隔离 | 弱（Namespace 隔离） | 强（硬件虚拟化） |
| 性能损耗 | 接近原生 | 5~20% |

### 1.2 Linux 核心技术

```
Docker 容器的底层依赖：

Namespace（隔离）：
  pid    → 进程隔离（容器有独立的 PID 1）
  net    → 网络隔离（独立网卡、IP、路由）
  mnt    → 文件系统挂载点隔离
  uts    → 主机名和域名隔离
  ipc    → 进程间通信隔离
  user   → 用户和组 ID 隔离

Cgroups（资源限制）：
  cpu    → CPU 使用率限制
  memory → 内存使用限制
  blkio  → 磁盘 I/O 限制
  net_cls→ 网络带宽限制

UnionFS（联合文件系统）：
  OverlayFS（默认）→ 分层存储，镜像层只读，容器层可写
```

### 1.3 镜像分层与 OverlayFS

```
镜像分层结构：

  Layer 4: 应用代码       ← 可写层（容器层）
  ─────────────────────
  Layer 3: pip install   ← 只读层
  Layer 2: Python 3.11   ← 只读层
  Layer 1: Ubuntu 22.04  ← 只读层（Base Image）

OverlayFS 工作原理：
  lowerdir = 镜像只读层（叠加）
  upperdir = 容器可写层
  workdir  = 临时工作目录
  merged   = 用户看到的合并视图

写时复制（CoW）：
  读文件 → 直接读只读层
  写文件 → 从只读层复制到 upperdir，再修改
  删文件 → 在 upperdir 创建 .wh.（whiteout）标记文件
```

### 1.4 容器生命周期

```
docker create → created
docker start  → running
docker pause  → paused
docker stop   → exited（SIGTERM → 等待 → SIGKILL）
docker kill   → exited（立即 SIGKILL）
docker rm     → 删除

# 查看容器状态
docker ps -a
docker inspect <container>
docker stats   # 实时资源监控
docker logs -f <container>
docker exec -it <container> /bin/bash
```

---

## 二、Dockerfile 与镜像优化

### 2.1 Dockerfile 最佳实践

```dockerfile
# ── 多阶段构建（Multi-stage Build）──────────────────────────
# 阶段1：构建环境（包含编译工具，体积大）
FROM python:3.11-slim AS builder

WORKDIR /app

# ✅ 先复制依赖文件，利用层缓存
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# 阶段2：运行环境（仅包含运行时，体积小）
FROM python:3.11-slim AS runtime

# ✅ 使用非 root 用户运行（安全）
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# 从构建阶段复制已安装的包
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

USER appuser

# ✅ 明确暴露端口（文档化）
EXPOSE 8000

# ✅ 使用 ENTRYPOINT + CMD 组合（可被覆盖）
ENTRYPOINT ["python", "-m"]
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 2.2 镜像体积优化

```dockerfile
# ❌ 不优化版本（镜像大）
FROM ubuntu:22.04
RUN apt-get update
RUN apt-get install -y python3 pip
RUN pip install flask

# ✅ 优化版本
FROM python:3.11-slim        # 使用 slim/alpine 基础镜像

# 合并 RUN 指令（减少层数）
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*    # 清理包缓存

# pip 安装优化
RUN pip install --no-cache-dir flask  # 不缓存 pip 包

# .dockerignore（排除不必要文件）
# .git
# __pycache__
# *.pyc
# .env
# node_modules
# tests/
```
**镜像大小对比**：

| 基础镜像 | 大小 |
|---|---|
| `ubuntu:22.04` | ~77 MB |
| `python:3.11` | ~900 MB |
| `python:3.11-slim` | ~130 MB |
| `python:3.11-alpine` | ~50 MB |
| `scratch`（空镜像） | 0 MB |

### 2.3 构建缓存优化

```dockerfile
# 利用层缓存：变化少的放前面，变化多的放后面

# ✅ 正确顺序：依赖文件变化少，代码变化多
COPY requirements.txt .
RUN pip install -r requirements.txt   # 依赖不变则缓存命中
COPY . .                               # 代码变化不影响依赖层

# ❌ 错误顺序：代码变化导致依赖重新安装
COPY . .
RUN pip install -r requirements.txt   # 每次代码变化都要重装依赖
```

### 2.4 常用 Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime          # 多阶段构建目标
    image: myapp:latest
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy   # 等待健康检查通过
      redis:
        condition: service_started
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## 三、Docker 网络与存储

### 3.1 Docker 网络模式

| 网络模式 | 说明 | 适用场景 |
|---|---|---|
| `bridge`（默认） | 容器通过 docker0 网桥通信，NAT 访问外网 | 单机多容器通信 |
| `host` | 共享宿主机网络栈，性能最高 | 高性能网络场景 |
| `none` | 无网络，完全隔离 | 安全隔离场景 |
| `overlay` | 跨主机容器通信（Swarm/K8s） | 多主机集群 |
| `macvlan` | 容器分配 MAC 地址，直连物理网络 | 需要直接访问物理网络 |

```bash
# 自定义网络（推荐，支持 DNS 解析容器名）
docker network create mynet
docker run --network mynet --name app myapp
docker run --network mynet --name db postgres
# app 容器内可直接通过 "db" 域名访问数据库
```

### 3.2 Docker 存储

```bash
# 三种挂载方式
# 1. Volume（推荐，Docker 管理）
docker run -v myvolume:/app/data myapp

# 2. Bind Mount（挂载宿主机目录）
docker run -v /host/path:/container/path myapp

# 3. tmpfs（内存文件系统，不持久化）
docker run --tmpfs /tmp myapp

# Volume 操作
docker volume create myvolume
docker volume ls
docker volume inspect myvolume
docker volume rm myvolume
docker volume prune           # 清理未使用的 volume
```

---
## 四、Kubernetes 核心架构

### 4.1 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                        │
│                                                              │
│  ┌─────────────────────────────────────┐                     │
│  │           Control Plane             │                     │
│  │                                     │                     │
│  │  kube-apiserver  ← 集群唯一入口      │                     │
│  │  etcd            ← 集群状态存储      │                     │
│  │  kube-scheduler  ← Pod 调度         │                     │
│  │  controller-manager ← 控制循环      │                     │
│  └─────────────────────────────────────┘                     │
│                         │                                    │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Node 1   │  │   Node 2   │  │   Node 3   │            │
│  │            │  │            │  │            │            │
│  │  kubelet   │  │  kubelet   │  │  kubelet   │            │
│  │  kube-proxy│  │  kube-proxy│  │  kube-proxy│            │
│  │  容器运行时 │  │  容器运行时 │  │  容器运行时 │            │
│  │  Pod(s)    │  │  Pod(s)    │  │  Pod(s)    │            │
│  └────────────┘  └────────────┘  └────────────┘            │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Control Plane 组件

| 组件 | 职责 |
|---|---|
| **kube-apiserver** | 集群的 REST API 入口，所有组件通过它通信，水平扩展 |
| **etcd** | 分布式 KV 存储，保存集群所有状态，强一致性（Raft） |
| **kube-scheduler** | 监听未调度的 Pod，根据策略选择合适的 Node |
| **controller-manager** | 运行各种控制器（Deployment/ReplicaSet/Node 控制器等） |
| **cloud-controller-manager** | 与云厂商 API 交互（LB、存储、节点） |

### 4.3 Node 组件

| 组件 | 职责 |
|---|---|
| **kubelet** | 节点代理，负责 Pod 生命周期管理，与 apiserver 通信 |
| **kube-proxy** | 维护 iptables/IPVS 规则，实现 Service 的负载均衡 |
| **容器运行时** | 拉取镜像、创建/删除容器（containerd、CRI-O） |

### 4.4 声明式 API 与控制循环

```
声明式 API（Desired State）：
  用户声明期望状态（YAML）→ 提交到 apiserver → 存入 etcd

控制循环（Reconcile Loop）：
  Controller 持续观察实际状态
  实际状态 ≠ 期望状态 → 执行操作使其一致
  实际状态 = 期望状态 → 无操作

示例（Deployment Controller）：
  期望：3 个副本
  实际：2 个副本（1 个 Pod 崩溃）
  操作：创建 1 个新 Pod → 实际变为 3
```

---

## 五、Pod 与工作负载

### 5.1 Pod 基础

```yaml
# Pod 完整示例
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: v1
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8000
    resources:
      requests:                  # 调度时的最低保障
        cpu: "100m"              # 0.1 核
        memory: "128Mi"
      limits:                    # 硬上限（超出则 OOMKilled/Throttle）
        cpu: "500m"
        memory: "512Mi"
    env:
    - name: DB_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url
    livenessProbe:               # 存活探针：失败则重启容器
      httpGet:
        path: /health
        port: 8000
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:              # 就绪探针：失败则从 Service 摘除
      httpGet:
        path: /ready
        port: 8000
      initialDelaySeconds: 5
      periodSeconds: 5
    startupProbe:                # 启动探针：启动慢的应用
      httpGet:
        path: /health
        port: 8000
      failureThreshold: 30
      periodSeconds: 10
  initContainers:                # 初始化容器（按顺序执行，成功后才启动主容器）
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
  restartPolicy: Always          # Always / OnFailure / Never
```
### 5.2 工作负载类型

| 类型 | 用途 | 特点 |
|---|---|---|
| **Deployment** | 无状态应用（Web/API） | 支持滚动更新、回滚、扩缩容 |
| **StatefulSet** | 有状态应用（DB/MQ） | 稳定网络标识、有序部署/删除、持久存储 |
| **DaemonSet** | 节点级守护进程（日志/监控） | 每个节点运行一个 Pod |
| **Job** | 一次性批处理任务 | 运行到完成，支持并行 |
| **CronJob** | 定时任务 | 基于 cron 表达式调度 Job |
| **ReplicaSet** | Pod 副本管理 | 通常由 Deployment 管理，不直接使用 |

### 5.3 Deployment 滚动更新

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # 滚动更新时最多多出 2 个 Pod
      maxUnavailable: 1    # 滚动更新时最多不可用 1 个 Pod
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:2.0   # 更新镜像版本触发滚动更新
```

```bash
# 常用操作
kubectl apply -f deployment.yaml
kubectl rollout status deployment/myapp      # 查看滚动更新状态
kubectl rollout history deployment/myapp     # 查看更新历史
kubectl rollout undo deployment/myapp        # 回滚到上一版本
kubectl rollout undo deployment/myapp --to-revision=2  # 回滚到指定版本
kubectl scale deployment myapp --replicas=10            # 手动扩缩容
```

### 5.4 StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless   # 必须关联 Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:          # 每个 Pod 自动创建独立 PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

# StatefulSet Pod 命名规则：mysql-0, mysql-1, mysql-2
# DNS 解析：mysql-0.mysql-headless.default.svc.cluster.local
```

---

## 六、Service 与网络

### 6.1 Service 类型

| 类型 | 说明 | 访问方式 |
|---|---|---|
| **ClusterIP**（默认） | 集群内部虚拟 IP，只在集群内可访问 | `<service-name>.<namespace>.svc.cluster.local` |
| **NodePort** | 在每个 Node 上开放端口（30000~32767） | `<NodeIP>:<NodePort>` |
| **LoadBalancer** | 云厂商创建外部负载均衡器 | 外部 IP |
| **ExternalName** | 将 Service 映射到外部 DNS 名称 | CNAME 解析 |
| **Headless** | `clusterIP: None`，直接返回 Pod IP | DNS 解析到 Pod IP |

```yaml
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp            # 匹配 Pod 的 Label
  ports:
  - port: 80             # Service 端口
    targetPort: 8000     # Pod 端口
  type: ClusterIP

---
# NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8000
    nodePort: 30080      # 指定 NodePort（不指定则随机）
  type: NodePort
```

### 6.2 Ingress

```yaml
# Ingress：HTTP/HTTPS 路由（需要 Ingress Controller，如 nginx-ingress）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
```
### 6.3 DNS 解析规则

```
Pod DNS 格式：
  <pod-ip>.<namespace>.pod.cluster.local
  例：10-244-1-5.default.pod.cluster.local

Service DNS 格式：
  <service-name>.<namespace>.svc.cluster.local
  例：myapp-service.default.svc.cluster.local

同命名空间内：
  可直接用 service 名访问：myapp-service

跨命名空间：
  需加命名空间：myapp-service.production
```

---

## 七、存储（PV/PVC/StorageClass）

### 7.1 存储体系

```
StorageClass（存储类）
  ↓ 动态供给
PersistentVolume（PV）── 集群级别，管理员创建或动态供给
  ↓ 绑定
PersistentVolumeClaim（PVC）── 命名空间级别，用户申请
  ↓ 挂载
Pod
```

### 7.2 PV/PVC 示例

```yaml
# PersistentVolume（管理员创建，或 StorageClass 动态供给）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce       # RWO：单节点读写
    # - ReadOnlyMany      # ROX：多节点只读
    # - ReadWriteMany     # RWX：多节点读写（NFS/CephFS）
  persistentVolumeReclaimPolicy: Retain  # Retain/Delete/Recycle
  storageClassName: standard
  hostPath:               # 生产用 NFS/Ceph/云盘等
    path: /data/my-pv

---
# PersistentVolumeClaim（用户申请存储）
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard

---
# Pod 使用 PVC
spec:
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc
  containers:
  - name: app
    volumeMounts:
    - name: data-volume
      mountPath: /app/data
```

### 7.3 StorageClass 动态供给

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # 设为默认
provisioner: kubernetes.io/aws-ebs        # 云厂商或存储插件
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete                      # PVC 删除时自动删除 PV
volumeBindingMode: WaitForFirstConsumer    # 延迟绑定（优化调度）
allowVolumeExpansion: true                 # 允许扩容
```

---

## 八、调度、资源管理与自动伸缩

### 8.1 调度流程

```
Pod 创建 → kube-scheduler 监听到未调度的 Pod

调度分两阶段：

1. Filtering（过滤）：排除不满足条件的节点
   - NodeSelector / NodeAffinity（节点标签）
   - Taints / Tolerations（污点/容忍）
   - 资源是否充足（requests）
   - PodAffinity / PodAntiAffinity（Pod 亲和性）

2. Scoring（打分）：对剩余节点打分，选最高分
   - 资源均衡（LeastAllocated）
   - 镜像已存在（ImageLocality）
   - 亲和性规则
```
### 8.2 节点亲和性与污点

```yaml
# NodeAffinity（节点亲和性）
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬要求
        nodeSelectorTerms:
        - matchExpressions:
          - key: disk-type
            operator: In
            values: ["ssd"]
      preferredDuringSchedulingIgnoredDuringExecution: # 软偏好
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]

# PodAntiAffinity（Pod 反亲和，分散部署）
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname  # 同一节点不能有相同 app 的 Pod
```

```bash
# Taint（污点）：阻止 Pod 调度到节点
kubectl taint nodes node1 key=value:NoSchedule   # 不允许调度
kubectl taint nodes node1 key=value:PreferNoSchedule  # 尽量不调度
kubectl taint nodes node1 key=value:NoExecute    # 不允许调度且驱逐现有 Pod

# Toleration（容忍）：允许 Pod 调度到有污点的节点
spec:
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

### 8.3 自动伸缩

```yaml
# HPA（Horizontal Pod Autoscaler）：水平扩缩容
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # CPU 使用率超 70% 则扩容
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 400Mi
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # 扩容冷却 60s
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容冷却 300s
```

```
VPA（Vertical Pod Autoscaler）：垂直扩缩（调整 requests/limits）
  适用：单 Pod 资源需求波动大的场景
  ⚠️ 注意：VPA 调整需要重启 Pod

CA（Cluster Autoscaler）：节点自动扩缩
  节点资源不足，Pod Pending → 自动添加节点
  节点利用率过低 → 自动缩减节点
```
### 8.4 资源配额

```yaml
# ResourceQuota：限制命名空间资源总量
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    services: "20"

---
# LimitRange：设置 Pod/Container 默认资源限制
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    default:           # 默认 limits
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:    # 默认 requests
      cpu: "100m"
      memory: "128Mi"
    max:               # 最大值
      cpu: "2"
      memory: "2Gi"
```

---

## 九、生产实践与运维

### 9.1 ConfigMap 与 Secret

```yaml
# ConfigMap：非敏感配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8000
    database:
      pool_size: 10

---
# Secret：敏感信息（Base64 编码，推荐配合 Vault/Sealed Secrets）
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=   # echo -n "password123" | base64
stringData:                     # 明文（k8s 自动 base64）
  url: "postgresql://user:pass@db:5432/mydb"

# Pod 中使用
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
volumeMounts:
- name: config
  mountPath: /app/config
volumes:
- name: config
  configMap:
    name: app-config
```

### 9.2 健康检查与优雅停机

```yaml
# 完整的生产级健康检查配置
containers:
- name: app
  lifecycle:
    preStop:                           # 优雅停机：先等待 30s
      exec:
        command: ["/bin/sh", "-c", "sleep 30"]
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8000
    initialDelaySeconds: 30            # 启动后 30s 才开始检查
    periodSeconds: 15
    timeoutSeconds: 5
    failureThreshold: 3                # 连续 3 次失败才重启
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8000
    initialDelaySeconds: 5
    periodSeconds: 5
    successThreshold: 1
    failureThreshold: 3
  terminationGracePeriodSeconds: 60   # 给 60s 时间优雅关闭
```
### 9.3 常用运维命令

```bash
# 查看资源
kubectl get pods -n production -o wide    # 查看 Pod 及所在节点
kubectl get pods --all-namespaces
kubectl describe pod myapp-xxx            # 查看详细信息（Events）
kubectl top pods                          # 查看资源使用（需 metrics-server）
kubectl top nodes

# 日志
kubectl logs myapp-xxx -c app             # 指定容器
kubectl logs myapp-xxx --previous         # 查看上次崩溃的日志
kubectl logs -l app=myapp --tail=100 -f  # 按 Label 查看多 Pod 日志

# 调试
kubectl exec -it myapp-xxx -- /bin/bash
kubectl port-forward pod/myapp-xxx 8080:8000     # 端口转发
kubectl cp myapp-xxx:/app/logs ./local-logs      # 文件拷贝

# 排查 Pod 问题
kubectl get events --sort-by='.lastTimestamp' -n production
kubectl describe node node1    # 查看节点状态和资源

# 资源清理
kubectl delete pod myapp-xxx --grace-period=0 --force  # 强制删除
kubectl rollout restart deployment/myapp               # 滚动重启

# 快速生成 YAML
kubectl create deployment myapp --image=myapp:1.0 --dry-run=client -o yaml
kubectl expose deployment myapp --port=80 --dry-run=client -o yaml
```

### 9.4 Pod 状态排查

| Pod 状态 | 原因 | 排查方法 |
|---|---|---|
| `Pending` | 无可用节点（资源不足/亲和性） | `describe pod` 看 Events |
| `CrashLoopBackOff` | 容器反复崩溃 | `logs --previous` 看上次日志 |
| `OOMKilled` | 内存超出 limits | 增大 memory limits |
| `ImagePullBackOff` | 镜像拉取失败 | 检查镜像名/仓库权限 |
| `Evicted` | 节点资源不足被驱逐 | 检查节点磁盘/内存 |
| `Terminating` 卡住 | finalizer 未处理 | 手动 patch 删除 finalizer |

---

## 十、高频追问与陷阱题

### Q1：Docker 容器与宿主机如何隔离？

> Docker 使用 Linux 内核的两个关键特性：
>
> 1. **Namespace（隔离）**：为每个容器创建独立的进程树（pid）、网络（net）、文件系统（mnt）、主机名（uts）等视图，容器内进程感知不到其他容器
>
> 2. **Cgroups（资源限制）**：限制容器能使用的 CPU、内存、I/O 等资源上限
>
> 3. **OverlayFS（文件系统）**：镜像只读层 + 容器可写层，写时复制
>
> ⚠️ **与 VM 的区别**：容器共享宿主机内核，不是完全隔离，容器内进程本质上是宿主机进程（不同 Namespace）。

---

### Q2：K8s 中 Deployment、StatefulSet、DaemonSet 如何选择？

> - **Deployment**：无状态应用（Web 服务、API、Worker）
>   - Pod 之间完全等价，可随时替换
>   - 支持滚动更新、自动扩缩容
>
> - **StatefulSet**：有状态应用（数据库、消息队列、ZooKeeper）
>   - 每个 Pod 有稳定网络标识（mysql-0、mysql-1）
>   - 有序部署和删除
>   - 每个 Pod 绑定独立持久存储
>
> - **DaemonSet**：节点级守护进程
>   - 每个节点恰好运行一个 Pod（日志收集/监控 Agent/网络插件）
>   - 节点加入集群时自动调度

---

### Q3：Pod 的 requests 和 limits 有什么区别？

> - **requests**：调度时保证分配的资源量（Scheduler 根据 requests 选择 Node）
>   - CPU requests：保证最低 CPU 时间片
>   - Memory requests：保证最低内存分配
>
> - **limits**：运行时的硬上限
>   - CPU limits：超出则被 Throttle（降速，不会被杀死）
>   - Memory limits：超出则被 OOMKill（容器重启）
>
> ```
> 建议：
>   requests < limits（留有弹性空间）
>   生产环境必须设置 limits（防止单 Pod 吃完节点资源）
>   limits/requests 比值不要过大（建议 ≤ 2）
> ```

---
### Q4：K8s Service 的负载均衡是如何实现的？

> **kube-proxy** 在每个节点维护网络规则，有三种模式：
>
> 1. **iptables 模式**（默认）：
>    - 通过 iptables DNAT 规则将 ClusterIP:Port → PodIP:Port
>    - 随机轮询，规则数量随 Service/Pod 增多线性增长
>    - 缺点：规则更新时锁定 iptables，延迟高
>
> 2. **IPVS 模式**（推荐，大规模集群）：
>    - 使用 Linux 内核 IPVS（IP Virtual Server）
>    - 支持更多负载均衡算法（RR/LC/SH 等）
>    - O(1) 规则查找，万级 Service 时性能显著优于 iptables
>
> 3. **userspace 模式**（已废弃）

---

### Q5：如何实现 K8s 应用的零停机发布？

> **关键配置组合**：
>
> 1. **滚动更新策略**：`maxUnavailable=0`（始终保持全量 Pod 可用）
> 2. **就绪探针**：新 Pod 就绪后才摘除旧 Pod
> 3. **PreStop Hook + terminationGracePeriodSeconds**：给旧 Pod 足够时间处理完请求
> 4. **PodDisruptionBudget（PDB）**：限制同时中断的 Pod 数量
>
> ```yaml
> # PodDisruptionBudget：至少保持 3 个可用
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: myapp-pdb
> spec:
>   minAvailable: 3    # 或 maxUnavailable: 1
>   selector:
>     matchLabels:
>       app: myapp
> ```

---

### Q6：etcd 在 K8s 中的作用？为什么用 etcd？

> **etcd 的作用**：
> - 存储 K8s 集群的所有状态（Pod、Service、Deployment 等所有资源的定义）
> - 提供 Watch 机制（组件监听资源变化，实现控制循环）
> - 分布式协调（Leader 选举）
>
> **为什么选 etcd**：
> - 强一致性（基于 Raft 共识算法，不丢数据）
> - 支持 Watch（事件通知机制，高效实现控制循环）
> - 高可用（奇数节点部署，容忍 (n-1)/2 个节点故障）
>
> **生产建议**：
> - 独立部署（不与业务节点混用）
> - 至少 3 节点（生产推荐 5 节点）
> - 定期备份 etcd 数据（`etcdctl snapshot save`）
> - 使用 SSD 存储（etcd 对磁盘延迟敏感）

---

### Q7：K8s 网络插件（CNI）有哪些？如何选择？

> | 插件 | 特点 | 适用场景 |
> |---|---|---|
> | **Flannel** | 简单、性能一般，VXLAN/host-gw | 测试/小规模集群 |
> | **Calico** | 高性能 BGP 路由，支持网络策略 | 生产推荐 |
> | **Cilium** | eBPF 实现，高性能，强可观测性 | 新一代云原生推荐 |
> | **Weave** | 易部署，支持加密 | 安全要求高 |
>
> **选型建议**：
> - 中小规模：Flannel（简单）或 Calico（需要 NetworkPolicy）
> - 大规模生产：Calico（BGP 性能好）或 Cilium（eBPF，观测性强）
> - 云厂商：优先使用厂商 CNI（AWS VPC CNI / GKE CNI）

---

## 附录：面试前必看清单

- [ ] 能解释 Docker 容器的隔离原理（Namespace / Cgroups / OverlayFS）
- [ ] 能写出多阶段 Dockerfile 并说明镜像优化方法
- [ ] 熟悉 K8s 控制平面各组件职责（apiserver/etcd/scheduler/controller-manager）
- [ ] 能说出 Deployment / StatefulSet / DaemonSet 的使用场景
- [ ] 理解 Pod 的三种探针（liveness/readiness/startup）及作用
- [ ] 熟悉 requests 与 limits 的区别及 OOMKilled 处理
- [ ] 了解 Service 的四种类型及 kube-proxy 的 iptables/IPVS 实现
- [ ] 能描述滚动更新 + 就绪探针 + PreStop 实现零停机发布
- [ ] 了解 HPA / VPA / CA 三种自动伸缩方案
- [ ] 熟悉常用运维命令（logs/describe/exec/top/rollout）

---

*文档版本：Docker 24.x / Kubernetes 1.29+ 适用 ｜ 最后更新：2025 年*
