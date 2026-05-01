# 项目十二：系统架构设计

## 📋 项目概述

系统架构设计是构建可靠、可扩展、高性能软件系统的基础。本项目学习企业级系统架构设计原则、模式和实践方法。

## 🎯 学习目标

- 掌握架构设计核心原则
- 理解常见架构模式
- 学习高可用设计方法
- 实施性能优化策略
- 掌握容量规划方法

## 🏗️ 架构设计原则

### 1. SOLID 原则

| 原则 | 说明 | 示例 |
|------|------|------|
| **S** - 单一职责 | 一个类只负责一项职责 | 分离业务逻辑和数据访问 |
| **O** - 开闭原则 | 对扩展开放，对修改关闭 | 使用接口和抽象类 |
| **L** - 里氏替换 | 子类可以替换父类 | 继承关系合理设计 |
| **I** - 接口隔离 | 使用多个专用接口 | 避免胖接口 |
| **D** - 依赖倒置 | 依赖抽象而非具体 | 依赖注入模式 |

### 2. CAP 定理

```
        Consistency (一致性)
              /   \
             /     \
            /       \
           /         \
          /           \
    CP 系统            AP 系统
 (ZooKeeper)        (Cassandra)
        \           /
         \         /
          \       /
           \     /
            \   /
        Availability (可用性)
              |
              |
        Partition Tolerance (分区容错)
```

### 3. BASE 理论

- **BA** - Basically Available (基本可用)
- **S** - Soft state (软状态)
- **E** - Eventually consistent (最终一致性)

## 📐 常见架构模式

### 1. 分层架构

```
┌─────────────────────────────────┐
│     Presentation Layer          │
│        (UI/API Gateway)         │
└─────────────────────────────────┘
              │
┌─────────────────────────────────┐
│       Business Layer            │
│     (Service/Controller)        │
└─────────────────────────────────┘
              │
┌─────────────────────────────────┐
│       Data Access Layer         │
│        (DAO/Repository)         │
└─────────────────────────────────┘
              │
┌─────────────────────────────────┐
│       Database Layer            │
│      (MySQL/MongoDB/Redis)      │
└─────────────────────────────────┘
```

### 2. 微服务架构

```
┌─────────────────────────────────────────────────────────┐
│                      API Gateway                         │
│                    (Kong/Nginx)                          │
└────┬────────────┬────────────┬────────────┬────────────┘
     │            │            │            │
┌────▼───┐  ┌────▼───┐  ┌────▼───┐  ┌────▼───┐
│ User   │  │ Order  │  │ Product│  │ Payment│
│ Service│  │ Service│  │ Service│  │ Service│
│        │  │        │  │        │  │        │
│ MySQL  │  │ MySQL  │  │ MongoDB│  │ MySQL  │
└────────┘  └────────┘  └────────┘  └────────┘
```

### 3. 事件驱动架构

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Producer │────►│  Event   │────►│ Consumer │
│          │     │  Bus     │     │          │
└──────────┘     │ (Kafka)  │     └──────────┘
                 └──────────┘
                      │
                 ┌────▼────┐
                 │ Consumer│
                 │    #2   │
                 └─────────┘
```

### 4. CQRS 架构

```
┌─────────────────┐         ┌─────────────────┐
│   Write Model   │         │   Read Model    │
│   (Command)     │         │   (Query)       │
│                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │
│  │  Command  │  │         │  │   Query   │  │
│  │  Handler  │  │         │  │  Handler  │  │
│  └─────┬─────┘  │         │  └─────┬─────┘  │
│        │        │         │        │        │
│  ┌─────▼─────┐  │         │  ┌─────▼─────┐  │
│  │  Write    │  │         │  │   Read    │  │
│  │   DB      │  │         │  │    DB     │  │
│  │ (Master)  │  │         │  │  (Slave)  │  │
│  └───────────┘  │         │  └───────────┘  │
└────────┬────────┘         └─────────────────┘
         │
         │ Event
         ▼
┌─────────────────┐
│  Event Handler  │
│  (Sync Read DB) │
└─────────────────┘
```

## 🔧 高可用设计

### 1. 冗余设计

```yaml
# Kubernetes Deployment 多副本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3  # 至少 3 个副本
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-app
            topologyKey: kubernetes.io/hostname
```

### 2. 故障转移

```yaml
# Keepalived 配置
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    virtual_ipaddress {
        192.168.1.100
    }
    
    track_script {
        chk_http_port
    }
}
```

### 3. 限流降级

```java
// Sentinel 限流配置
@SentinelResource(value = "getUser", 
    blockHandler = "handleBlock", 
    fallback = "handleFallback")
public User getUser(Long userId) {
    return userService.findById(userId);
}

public User handleBlock(Long userId, BlockException ex) {
    // 限流降级逻辑
    return new User("default", "服务繁忙");
}
```

### 4. 熔断器模式

```python
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
def call_external_service():
    response = requests.get('http://external-service/api')
    response.raise_for_status()
    return response.json()

# 使用示例
try:
    data = call_external_service()
except CircuitBreakerError:
    # 熔断器打开，使用缓存或默认值
    data = get_cached_data()
```

## 📊 性能优化

### 1. 缓存策略

```python
# 多级缓存设计
import redis
from functools import wraps
import hashlib

class MultiLevelCache:
    def __init__(self):
        self.local_cache = {}  # L1: 本地缓存
        self.redis_client = redis.Redis(host='localhost', port=6379)  # L2: Redis
        self.db = get_database()  # L3: 数据库
    
    def get(self, key):
        # L1 缓存
        if key in self.local_cache:
            return self.local_cache[key]
        
        # L2 缓存
        value = self.redis_client.get(key)
        if value:
            self.local_cache[key] = value
            return value
        
        # L3 数据库
        value = self.db.query(key)
        if value:
            self.redis_client.setex(key, 300, value)  # 5 分钟过期
            self.local_cache[key] = value
        return value
    
    def set(self, key, value, ttl=300):
        self.local_cache[key] = value
        self.redis_client.setex(key, ttl, value)
        self.db.save(key, value)
```

### 2. 数据库优化

```sql
-- 1. 索引优化
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_status_created ON orders(status, created_at);

-- 2. 查询优化
-- ❌ 避免
SELECT * FROM orders WHERE YEAR(created_at) = 2026;

-- ✅ 推荐
SELECT * FROM orders 
WHERE created_at >= '2026-01-01' 
  AND created_at < '2027-01-01';

-- 3. 分库分表
-- 按用户 ID 分片
users_0: user_id % 10 = 0
users_1: user_id % 10 = 1
...
users_9: user_id % 10 = 9
```

### 3. 异步处理

```python
# Celery 异步任务
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task(bind=True, max_retries=3)
def send_email(self, user_id, template):
    try:
        user = User.objects.get(id=user_id)
        email_service.send(user.email, template)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=60)

# 使用
send_email.delay(user_id=123, template='welcome')
```

## 📈 容量规划

### 1. 流量估算

```
日活用户 (DAU): 100 万
峰值并发：DAU × 10% = 10 万
QPS: 10 万 / 86400 × 峰值系数 (5) ≈ 6000

单实例 QPS: 1000
所需实例数：6000 / 1000 = 6 个
冗余备份：6 × 1.5 = 9 个
```

### 2. 存储规划

```
单用户数据：10MB
日增数据：100 万 × 10MB = 10TB
年增数据：10TB × 365 = 3.65PB

存储方案：
- 热数据 (30 天): SSD 存储
- 温数据 (90 天): HDD 存储
- 冷数据 (归档): 对象存储
```

### 3. 带宽规划

```
平均响应大小：50KB
峰值 QPS: 6000
带宽需求：6000 × 50KB × 8 = 2.4Gbps

建议带宽：3Gbps (预留 25%)
```

## 📚 架构设计文档模板

```markdown
# 系统架构设计文档

## 1. 概述
- 项目背景
- 设计目标
- 约束条件

## 2. 架构视图
- 逻辑架构
- 物理架构
- 部署架构

## 3. 核心设计
- 技术选型
- 数据模型
- 接口设计

## 4. 非功能性需求
- 性能指标
- 可用性要求
- 安全策略

## 5. 风险评估
- 技术风险
- 运维风险
- 应对措施
```

## 📚 参考资料

- 《企业应用架构模式》
- 《微服务架构设计模式》
- 《数据密集型应用系统设计》

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
