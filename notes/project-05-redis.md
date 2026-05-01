# 项目五：Redis 5.0 集群部署

## 📋 项目概述

Redis 是一个高性能的内存数据存储系统，常用作缓存、消息队列和实时分析引擎。本项目实现 Redis 5.0 版本的 Cluster 集群模式部署，支持自动分片和故障转移。

## 🎯 学习目标

- 理解 Redis Cluster 架构原理
- 掌握集群模式部署流程
- 配置客户端连接和高可用
- 实施持久化和性能优化
- 学习数据迁移和管理操作

## 🏗️ 架构设计

```
                    ┌──────────────────────┐
                    │     Clients           │
                    │ (Jedis / Lettuce)     │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   Proxy/Route        │
                    │  (Client-side routing)│
                    └──┬───┬───┬───┬───┬──┘
                       │   │   │   │   │
          ┌────────────┼───┼───┼───┼───┼────────────┐
          ▼            ▼   ▼   ▼   ▼   ▼            ▼
    ┌──────────┐  ┌──────── ┌──────┐ ──────┐  ┌──────────┐
    │ Master 1 │  │Master 2│ │Master│ │Master│  │ Master N │
    │ (port 7000)| (port 7001)│(7002)│ │(7003)│  │ (port N) │
    └────┬─────┘  └───┬────┘ └──┬───┘ └──┬───┘  └────┬─────┘
         │            │        │        │            │
    ┌────▼─────┐  ┌───▼────┐ ┌─┴───┐ ┌─┴───┐  ┌───▼────┐
    │Slave 1  │  │Slave 2 │ │Slave│ │Slave│  │ Slave N │
    │(port 7000)│ │(port 7001)│(7002)│ │(7003)│  │ (port N) │
    └──────────┘  └────────┘ └──────┘ └──────┘  └──────────
```

## 📦 核心概念

| 概念 | 说明 |
|------|------|
| **Slot** | 哈希槽，共 16384 个（0-16383） |
| **Hash Slot** | key → CRC16(key) % 16384 → 分配到 slot |
| **Master** | 主节点，处理读写请求 |
| **Slave** | 从节点，复制主节点数据 |
| **Failover** | 故障转移，从节点提升为主节点 |
| **Gossip** | 节点间通信协议，交换集群信息 |
| **Epoch** | 版本号，用于冲突解决 |
| **Cluster Node** | 集群中的一个独立实例 |

## 🔧 快速部署

### 1. 环境要求

- CentOS 7.x / Ubuntu 18.04+
- 至少 6 台服务器（3 主 3 从）
- Docker 19.03+ 或手动编译安装

### 2. 手动编译安装

```bash
# 下载 Redis 5.0
cd /opt
wget http://download.redis.io/releases/redis-5.0.14.tar.gz
tar -xzf redis-5.0.14.tar.gz
cd redis-5.0.14

# 编译安装
make && make install PREFIX=/usr/local/redis

# 创建配置文件目录
mkdir -p /etc/redis
mkdir -p /var/lib/redis/data
mkdir -p /var/log/redis
```

### 3. 单节点配置文件

```ini
# /etc/redis/redis-7000.conf

# 基本配置
port 7000
bind 0.0.0.0
daemonize no
pidfile /var/run/redis-7000.pid
loglevel notice
logfile /var/log/redis/redis-7000.log

# 集群配置
cluster-enabled yes
cluster-config-file nodes-7000.conf
cluster-node-timeout 15000
cluster-require-full-coverage no

# 持久化
save 900 1
save 300 10
save 60 10000
rdbcompression yes
dbfilename dump.rdb
dir /var/lib/redis/data
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

# 内存管理
maxmemory 2gb
maxmemory-policy allkeys-lru

# 安全
requirepass mypassword
masterauth mypassword

# 网络
tcp-backlog 511
timeout 0
tcp-keepalive 300

# 其他
databases 1
```

### 4. 生成多节点配置

```bash
# 创建 6 个端口的配置文件
for port in 7000 7001 7002 7003 7004 7005; do
    cp /etc/redis/redis-7000.conf /etc/redis/redis-${port}.conf
    sed -i "s/port 7000/port ${port}/g" /etc/redis/redis-${port}.conf
    sed -i "s/redis-7000.log/redis-${port}.log/g" /etc/redis/redis-${port}.conf
    sed -i "s/nodes-7000.conf/nodes-${port}.conf/g" /etc/redis/redis-${port}.conf
done

# 启动所有节点
for port in 7000 7001 7002 7003 7004 7005; do
    redis-server /etc/redis/redis-${port}.conf &
    sleep 1
done

# 验证进程
ps aux | grep redis-server
```

### 5. 创建集群

```bash
# 使用 redis-cli 创建集群
redis-cli --cluster create \
    192.168.1.10:7000 \
    192.168.1.10:7001 \
    192.168.1.10:7002 \
    192.168.1.10:7003 \
    192.168.1.10:7004 \
    192.168.1.10:7005 \
    --cluster-replicas 1 \
    --cluster-password mypassword

# 查看集群状态
redis-cli -a mypassword cluster info
redis-cli -a mypassword cluster nodes

# 检查 slots 分配
redis-cli -a mypassword cluster slots
```

### 6. 集群拓扑结构

```
Node 1 (192.168.1.10:7000) - Master
├── Slots: 0-5460
└── Slave: Node 4 (192.168.1.10:7003)

Node 2 (192.168.1.10:7001) - Master
├── Slots: 5461-10922
└── Slave: Node 5 (192.168.1.10:7004)

Node 3 (192.168.1.10:7002) - Master
├── Slots: 10923-16383
└── Slave: Node 6 (192.168.1.10:7005)
```

## ✅ 客户端测试

### Python 客户端示例

```python
import redis
from redis.cluster import RedisCluster, ClusterNode

class RedisClusterClient:
    def __init__(self, host='192.168.1.10', port=7000):
        self.startup_nodes = [
            ClusterNode(host, 7000),
            ClusterNode(host, 7001),
            ClusterNode(host, 7002),
        ]
        
        try:
            self.client = RedisCluster(
                startup_nodes=self.startup_nodes,
                password='mypassword',
                decode_responses=True,
                socket_connect_timeout=5,
                socket_timeout=5,
                retry_on_timeout=True
            )
            print("Connected to Redis Cluster successfully!")
        except Exception as e:
            print(f"Connection failed: {e}")
            raise
    
    def set_and_get(self, key, value):
        """设置和获取键值对"""
        try:
            self.client.set(key, value)
            result = self.client.get(key)
            return result
        except redis.cluster.ClusterError as e:
            print(f"Operation failed: {e}")
            return None
    
    def batch_operations(self, items, ttl=3600):
        """批量设置键值对并设置过期时间"""
        pipe = self.client.pipeline(transaction=False)
        for key, value in items.items():
            pipe.set(key, value, ex=ttl)
        
        try:
            results = pipe.execute()
            print(f"Batch operation completed: {len(results)} keys")
            return results
        except redis.cluster.ClusterError as e:
            print(f"Batch operation failed: {e}")
            return []
    
    def delete_keys(self, pattern):
        """删除匹配模式的键"""
        try:
            cursor = 0
            deleted_count = 0
            
            while True:
                cursor, keys = self.client.scan(cursor=cursor, match=pattern, count=100)
                if keys:
                    self.client.delete(*keys)
                    deleted_count += len(keys)
                
                if cursor == 0:
                    break
            
            print(f"Deleted {deleted_count} keys matching '{pattern}'")
            return deleted_count
        except redis.cluster.ClusterError as e:
            print(f"Delete failed: {e}")
            return 0
    
    def get_cluster_info(self):
        """获取集群信息"""
        try:
            info = self.client.info(section='cluster')
            keyspace = self.client.info(section='keyspace')
            
            print("=== Cluster Info ===")
            print(f"Cluster Enabled: {info.get('cluster_enabled')}")
            print(f"Cluster State: {info.get('cluster_state')}")
            print(f"Cluster Slots Assigned: {info.get('cluster_slots_assigned')}")
            print(f"Cluster Slots OK: {info.get('cluster_slots_ok')}")
            print(f"Cluster Slots Pfail: {info.get('cluster_slots_pfail')}")
            print(f"Cluster Slots Fail: {info.get('cluster_slots_fail')}")
            
            print("\n=== Keyspace Info ===")
            for db, stats in keyspace.items():
                print(f"{db}: {stats}")
            
            return info, keyspace
        except redis.cluster.ClusterError as e:
            print(f"Info retrieval failed: {e}")
            return None, None
    
    def close(self):
        """关闭连接"""
        self.client.close()

# 使用示例
if __name__ == '__main__':
    client = RedisClusterClient()
    
    # 测试基本操作
    client.set_and_get('user:1001:name', '张三')
    client.set_and_get('user:1001:email', 'zhangsan@example.com')
    client.set_and_get('user:1002:name', '李四')
    
    # 批量操作
    batch_data = {
        f'cache:page:{i}': f'Page Content {i}' 
        for i in range(100)
    }
    client.batch_operations(batch_data, ttl=7200)
    
    # 获取集群信息
    client.get_cluster_info()
    
    # 清理数据
    client.delete_keys('cache:page:*')
    
    client.close()
```

### Java Jedis 客户端

```java
import redis.clients.jedis.*;
import java.util.*;

public class RedisClusterExample {
    
    // 连接池配置
    private static final JedisPoolConfig POOL_CONFIG = new JedisPoolConfig.Builder()
        .setMaxTotal(100)
        .setMaxIdle(20)
        .setMinIdle(10)
        .setTestOnBorrow(true)
        .setTestOnReturn(false)
        .build();
    
    // 集群连接配置
    private static final JedisCluster CLUSTER_CONNECTION;
    
    static {
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort("192.168.1.10", 7000));
        nodes.add(new HostAndPort("192.168.1.10", 7001));
        nodes.add(new HostAndPort("192.168.1.10", 7002));
        
        CLUSTER_CONNECTION = new JedisCluster(nodes, 5000, 5000, 5, 
                                           "mypassword", POOL_CONFIG);
    }
    
    public static void main(String[] args) {
        // 设置字符串值
        CLUSTER_CONNECTION.set("key1", "value1");
        String value = CLUSTER_CONNECTION.get("key1");
        System.out.println("Get key1: " + value);
        
        // 设置 Hash 数据结构
        Map<String, String> user = new HashMap<>();
        user.put("name", "John Doe");
        user.put("age", "30");
        user.put("city", "Shanghai");
        CLUSTER_CONNECTION.hset("user:1001", user);
        
        Map<String, String> userData = CLUSTER_CONNECTION.hgetAll("user:1001");
        System.out.println("User data: " + userData);
        
        // 设置 List 数据结构
        CLUSTER_CONNECTION.lpush("tasks", "task1", "task2", "task3");
        List<String> tasks = CLUSTER_CONNECTION.lrange("tasks", 0, -1);
        System.out.println("Tasks: " + tasks);
        
        // 设置过期时间
        CLUSTER_CONNECTION.setex("session:abc123", 3600, "active");
        
        // 关闭连接
        CLUSTER_CONNECTION.close();
    }
}
```

## 🔄 集群管理操作

### 1. 添加节点

```bash
# 添加新 Master 节点
redis-server /etc/redis/redis-7006.conf

# 将新节点加入集群
redis-cli -a mypassword --cluster add-node \
    192.168.1.10:7006 \
    192.168.1.10:7000

# 为新 Master 分配 hash slots
redis-cli -a mypassword --cluster reshard \
    192.168.1.10:7000 \
    --cluster-from $(redis-cli -a mypassword cluster myid) \
    --cluster-to $(redis-cli -a mypassword -p 7006 cluster myid) \
    --cluster-slots 1024 \
    --cluster-yes
```

### 2. 移除节点

```bash
# 从集群中移除节点
redis-cli -a mypassword --cluster del-node \
    192.168.1.10:7000 \
    $(redis-cli -a mypassword -p 7000 cluster myid)
```

### 3. 故障转移

```bash
# 手动触发故障转移
redis-cli -a mypassword -p 7003 cluster failover

# 检查主从关系
redis-cli -a mypassword -p 7000 cluster nodes | grep master
redis-cli -a mypassword -p 7003 cluster nodes | grep master
```

### 4. 数据迁移

```bash
# 在线迁移集群数据到新集群
redis-cli --cluster migrate-hosts \
    --cluster-from old-cluster \
    --cluster-to new-cluster

# 或者使用 redis-shake 工具
redis-shake conf/redis-shake.conf
```

## 📊 监控指标

### Prometheus Exporter 配置

```yaml
# docker-compose.monitor.yml
version: '3.8'
services:
  redis-exporter:
    image: oliver006/redis_exporter:v1.41.0
    container_name: redis_exporter
    ports:
      - "9121:9121"
    environment:
      REDIS_ADDR: "redis://192.168.1.10:7000"
      REDIS_PASSWORD: mypassword
      REDIS_EXPORTER_CHECK_KEYS: "my-key-pattern-*"
      REDIS_EXPORTER_CHECK_SINGLE_KEYS: "my-single-key"
      REDIS_EXPORTER_SCRIPT: "/scripts/redis-custom-script.lua"
    volumes:
      - ./scripts:/scripts
    restart: unless-stopped
```

### 关键监控指标

```promql
# Redis 运行时间（秒）
redis_uptime_in_seconds

# 内存使用情况
redis_memory_used_bytes{type="rss"}
redis_memory_used_bytes{type="peak"}
redis_memory_available_bytes

# 命中率统计
redis_keyspace_hits_total
redis_keyspace_misses_total
(redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)) * 100

# 每秒操作数
redis_commands_processed_total[1m]

# 客户端连接数
redis_connected_clients

# 阻塞客户端数量
redis_blocked_clients

# AOF 重写状态
redis_aof_rewrite_in_progress

# RDB 保存状态
redis_rdb_last_bgsave_status

# 复制状态
redis_master_link_up{role="slave"}
redis_master_repl_offset{role="slave"}

# 集群节点数量
redis_cluster_size{type="cluster_size"}
redis_cluster_stats_calls_total{cmd_type="call"}
```

## 🔧 性能调优

### 1. 内存策略选择

```bash
# 最大内存限制
maxmemory 4gb

# 淘汰策略（根据业务需求选择）
allkeys-lru       # 所有键使用 LRU 淘汰（推荐用于缓存）
volatile-lru      # 只有设置了过期时间的键使用 LRU
allkeys-lfu       # 所有键使用 LFU 淘汰
volatile-lfu      # 只有设置了过期时间的键使用 LFU
volatile-ttl      # 优先淘汰即将过期的键
noeviction        # 不淘汰，写入时返回错误（推荐用于数据库）
```

### 2. 持久化优化

```ini
# RDB 快照策略优化
save ""                          # 禁用 RDB（仅用 AOF）
save 900 1                       # 每 15 分钟如果有至少 1 个键修改就保存
save 300 10                      # 每 5 分钟如果有至少 10 个键修改就保存  
save 60 10000                    # 每分钟如果有至少 10000 个键修改就保存

# AOF 重写策略
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# AOF 同步策略
appendfsync always               # 最安全，性能最低
appendfsync everysec             # 平衡性能和安全性（推荐）
appendfsync no                   # 操作系统控制，性能最高
```

### 3. 网络优化

```ini
# 增加 TCP  backlog
tcp-backlog 4096

# 启用 TCP KeepAlive
tcp-keepalive 60

# 禁用慢查询日志
slowlog-log-slower-than -1
slowlog-max-len 128

# 调整工作线程（Linux）
io-threads 4
io-threads-do-reads yes
```

### 4. 集群参数优化

```ini
# 集群超时时间
cluster-node-timeout 15000

# 不等待全量覆盖
cluster-require-full-coverage no

# Gossip 协议配置
cluster-migration-barrier 1

# 集群内部总线端口
cluster-port 17000
```

## 🚨 故障排查

### 常见问题与解决方案

#### 1. 集群状态异常

```bash
# 检查集群状态
redis-cli -a mypassword cluster info

# 查看节点详细信息
redis-cli -a mypassword cluster nodes

# 检查是否有节点未连接
redis-cli -a mypassword cluster info | grep cluster_known_down_since

# 检查槽位是否全部覆盖
redis-cli -a mypassword cluster info | grep cluster_slots_not_covered
```

#### 2. 连接超时

```bash
# 调整客户端超时时间
redis-cli --cluster create \
    192.168.1.10:7000 \
    ... \
    --cluster-timeout 30000

# 检查防火墙规则
iptables -L -n | grep 7000
```

#### 3. 数据不一致

```bash
# 修复主从复制问题
redis-cli -a mypassword -p 7003 slaveof no one
redis-cli -a mypassword -p 7003 slaveof 192.168.1.10 7000

# 检查复制偏移量
redis-cli -a mypassword info replication
```

#### 4. 内存溢出

```bash
# 检查内存使用情况
redis-cli -a mypassword info memory

# 查找大键
redis-cli --bigkeys

# 删除大键（谨慎操作）
redis-cli -a mypassword del large_key
```

## 📈 最佳实践

### 1. Key 设计规范

```python
# 推荐的命名约定
namespace:id:field

# 示例
user:1001:name
user:1001:email
session:abc123:token
order:20260501:001:status
product:catalog:categories

# 避免的模式
long_descriptive_names_with_underscores_too_long
mixed_Case_names_without_consistency
random_numbers_like_12345_as_keys
```

### 2. 集群规模建议

| 场景 | Master 数量 | Slave 数量 | 总节点 | 适用场景 |
|------|------------|------------|--------|----------|
| 小型 | 3 | 3 | 6 | 开发/测试环境 |
| 中型 | 5 | 5 | 10 | 生产环境中小型系统 |
| 大型 | 10 | 10 | 20 | 生产环境中等流量 |
| 超大型 | 20+ | 20+ | 40+ | 高并发大规模系统 |

### 3. 数据安全备份

```bash
# 定期 RDB 快照备份
crontab -e
0 2 * * * /usr/local/bin/redis-cli -a mypassword BGSAVE > /dev/null 2>&1

# 远程备份到对象存储
rsync -av /var/lib/redis/data/dump.rdb backup@server:/backups/redis/

# AOF 文件加密备份
gzip -k appendonly.aof
openssl enc -aes-256-cbc -in appendonly.aof.gz -out backup.aof.enc -k password
```

## 📚 参考资料

- [Redis 官方文档](https://redis.io/documentation)
- [Redis Cluster 规范](https://github.com/redis/redis-doc/blob/master/docs/cluster/spec.md)
- [Redis 性能调优指南](https://redis.io/docs/latest/operate/rc/clusters/optimization/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
