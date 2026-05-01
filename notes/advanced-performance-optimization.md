# 性能优化实战指南

## 📋 概述

性能优化是运维的核心能力之一，不是调参数，而是系统性解决问题。我的方法论：**定位瓶颈 → 分析根因 → 验证效果**。

---

## 🧭 性能优化方法论

### 三步走流程

```
┌─────────────────────────────────────────────────────────────┐
│                   性能优化三步走                              │
│                                                             │
│  Step 1: 定位瓶颈                                           │
│  ├── 监控数据收集                                           │
│  ├── 性能基准测试                                           │
│  ├── 瓶颈识别 (CPU/内存/磁盘/网络)                          │
│  └── 定位具体组件                                           │
│                                                             │
│  Step 2: 分析根因                                           │
│  ├── 日志分析                                               │
│  ├── 源码审查                                               │
│  ├── 配置检查                                               │
│  └── 架构审视                                               │
│                                                             │
│  Step 3: 验证效果                                           │
│  ├── 实施优化                                               │
│  ├── 性能对比                                               │
│  ├── 监控验证                                               │
│  └── 文档记录                                               │
└─────────────────────────────────────────────────────────────┘
```

### 性能指标金字塔

```
┌─────────────────────────────────────────────────────────────┐
│                    性能指标层次                              │
│                                                             │
│         ┌─────────────────────────────────────┐            │
│         │      业务指标 (最顶层)               │            │
│         │  订单量/响应时间/用户满意度           │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │      应用指标                        │            │
│         │  QPS/TPS/延迟/错误率                 │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │      系统指标                        │            │
│         │  CPU/内存/磁盘/网络                  │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │      基础设施指标                    │            │
│         │  容器/Pod/节点                       │            │
│         └─────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Linux 系统优化

### CPU 优化

```bash
# CPU 性能分析工具

# 1. top - 实时监控
top -H -p <pid>  # 查看线程级 CPU 使用

# 2. mpstat - 多核 CPU 统计
mpstat -P ALL 1 10  # 每核 CPU 使用率

# 3. pidstat - 进程 CPU 统计
pidstat -p <pid> 1 10  # 进程 CPU 使用历史

# 4. perf - 性能分析神器
perf top -g -p <pid>  # 实时函数热点
perf record -g -p <pid> -- sleep 60  # 录制 60 秒
perf report  # 分析报告

# 5. 火焰图生成
perf script | stackcollapse-perf.pl | flamegraph.pl > cpu.svg
```

**CPU 优化策略：**

```
场景 1：CPU 使用率高
├── 用户态高：优化代码逻辑
├── 内核态高：减少系统调用
├── IO 等待高：优化磁盘/网络
└── 软中断高：优化网络收发

场景 2：CPU 使用率低但性能差
├── 锁竞争：减少锁粒度
├── 线程阻塞：增加并发
├── 进程切换：减少进程数
└── 上下文切换：优化调度
```

### 内存优化

```bash
# 内存性能分析工具

# 1. free - 内存概况
free -h

# 2. vmstat - 内存统计
vmstat 1 10  # 每秒统计

# 3. pidstat - 进程内存
pidstat -r -p <pid> 1 10

# 4. smem - 内存排序
smem -t -k -s rss | head -10  # RSS 内存排序

# 5. pmap - 进程内存映射
pmap -x <pid>  # 详细内存映射

# 6. /proc/meminfo - 内核内存信息
cat /proc/meminfo | grep -E "MemFree|MemAvailable|Buffers|Cached"
```

**内存优化策略：**

```ini
# 内核参数优化 /etc/sysctl.conf

# 内存分配策略
vm.swappiness = 10           # 降低 swap 使用
vm.dirty_ratio = 15          #脏页比例
vm.dirty_background_ratio = 3

# 大页内存 (Huge Pages)
vm.nr_hugepages = 1024       # 大页数量 (数据库场景)

# OOM killer 策略
vm.panic_on_oom = 1          # OOM 时内核 panic
```

### 磁盘 I/O 优化

```bash
# 磁盘性能分析工具

# 1. iostat - IO 统计
iostat -x -k 1 10  # 扩展统计

# 2. pidstat - 进程 IO
pidstat -d -p <pid> 1 10

# 3. iotop - 实时 IO 监控
iotop -oP  # 只显示有 IO 的进程

# 4. blktrace - 块设备追踪
blktrace -d /dev/sda -o - | blkparse -i -

# 5. fio - 性能测试工具
fio --name=randwrite --ioengine=libaio --iodepth=16 --rw=randwrite --bs=4k --direct=1 --size=1G --numjobs=4 --filename=/tmp/test
```

**磁盘优化策略：**

```bash
# 1. 调度算法优化
echo deadline > /sys/block/sda/queue/scheduler  # 数据库用 deadline
echo noop > /sys/block/sda/queue/scheduler      # SSD 用 noop

# 2. 预读优化
blockdev --setra 8192 /dev/sda  # 预读 4MB

# 3. 文件系统挂载优化
mount -o noatime,nodiratime,data=writeback /dev/sda1 /data

# 4. SSD 优化
echo 1 > /sys/block/sda/queue/nomerges  # 禁止合并请求
echo 0 > /sys/block/sda/queue/rotational  # 标记为非旋转设备
```

### 网络优化

```bash
# 网络性能分析工具

# 1. iftop - 流量监控
iftop -i eth0 -n

# 2. nethogs - 进程流量
nethogs eth0

# 3. tcpdump - 抓包分析
tcpdump -i eth0 -nn -XX 'port 80'

# 4. ss - 连接统计
ss -s  # 连接概览
ss -tan 'sport = :80'  # 端口连接

# 5. ethtool - 网卡统计
ethtool -S eth0 | grep -E "rx|tx"
```

**网络优化策略：**

```ini
# /etc/sysctl.conf 网络优化

# TCP 连接优化
net.core.somaxconn = 32768          # 监听队列长度
net.core.netdev_max_backlog = 16384  # 接收队列长度
net.ipv4.tcp_max_syn_backlog = 8192  # SYN 队列长度

# TCP 缓冲区优化
net.ipv4.tcp_rmem = 4096 87380 16777216  # TCP 读缓冲
net.ipv4.tcp_wmem = 4096 65536 16777216  # TCP 写缓冲
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# TCP 超时优化
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1

# 连接追踪优化
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
```

---

## 🗄️ 数据库优化

### MySQL 性能优化

```sql
-- 慢查询分析

-- 1. 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.1;  # 100ms
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- 2. 查看慢查询
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;

-- 3. 分析慢查询 (mysqldumpslow)
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

-- 4. 查看执行计划
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- 5. 查看表统计信息
SHOW TABLE STATUS LIKE 'orders';
ANALYZE TABLE orders;
```

**MySQL 配置优化：**

```ini
# my.cnf 生产优化

# InnoDB 缓冲池 (系统内存的 70-80%)
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8

# 日志优化
innodb_log_file_size = 1G
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 1

# IO 优化
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT

# 连接优化
max_connections = 500
thread_cache_size = 100
table_open_cache = 4000

# 查询优化
query_cache_type = 0  # 关闭查询缓存 (MySQL 5.7+)
query_cache_size = 0
```

### Redis 性能优化

```bash
# Redis 性能分析

# 1. 查看慢日志
redis-cli SLOWLOG GET 10

# 2. 查看统计信息
redis-cli INFO memory
redis-cli INFO stats

# 3. 查看命令统计
redis-cli INFO commandstats

# 4. 内存分析
redis-cli MEMORY STATS
redis-cli MEMORY USAGE key_name
```

**Redis 配置优化：**

```conf
# redis.conf 生产优化

# 内存优化
maxmemory 8gb
maxmemory-policy allkeys-lru  # 内存满时 LRU 淘汰

# 持久化优化
save ""                        # 关闭 RDB (纯缓存场景)
appendonly yes                 # 开启 AOF
appendfsync everysec           # 每秒同步

# 网络优化
tcp-backlog 511
timeout 300
tcp-keepalive 300

# 客户端优化
maxclients 10000
```

---

## 🌐 Web 服务优化

### Nginx 性能优化

```nginx
# nginx.conf 生产优化

worker_processes auto;               # 自动匹配 CPU 核数
worker_rlimit_nofile 100000;         # 每个进程最大文件数

events {
    worker_connections 4096;         # 每个进程连接数
    use epoll;                       # Linux 使用 epoll
    multi_accept on;                 # 一次接受多个连接
}

http {
    # 连接优化
    keepalive_timeout 30;
    keepalive_requests 1000;
    
    # 缓冲区优化
    client_body_buffer_size 16K;
    client_header_buffer_size 1k;
    client_max_body_size 8m;
    large_client_header_buffers 4 8k;
    
    # 文件缓存
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # Gzip 压缩
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript;
    
    # 上游优化
    upstream backend {
        server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
        server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;  # 保持连接池
    }
    
    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_pass http://backend;
    }
}
```

---

## 📊 性能基准测试

### 常用压测工具

```bash
# HTTP 压测工具

# 1. Apache Bench (ab)
ab -n 10000 -c 100 http://localhost:8080/api

# 2. wrk
wrk -t12 -c400 -d30s http://localhost:8080/api

# 3. hey
hey -n 10000 -c 100 http://localhost:8080/api

# 4. JMeter (复杂场景)
jmeter -n -t test-plan.jmx -l results.jtl

# 数据库压测工具

# MySQL
sysbench oltp_read_write --tables=10 --table-size=1000000 --threads=64 --time=300 run

# Redis
redis-benchmark -h localhost -p 6379 -t set,get -n 100000 -c 50
```

---

## 📋 性能优化检查清单

```markdown
# 性能优化 Checklist

## CPU
- [ ] CPU 使用率 < 80%
- [ ] 上下文切换 < 10000/s
- [ ] 软中断占比 < 30%

## 内存
- [ ] 内存使用率 < 85%
- [ ] Swap 使用率 < 10%
- [ ] 内存泄漏检测

## 磁盘
- [ ] IOPS < 设备上限 80%
- [ ] IO 延迟 < 10ms
- [ ] 磁盘使用率 < 80%

## 网络
- [ ] 带宽使用率 < 80%
- [ ] TCP 连接数 < 上限 80%
- [ ] 网络延迟 < 100ms

## 应用
- [ ] 响应时间 < SLA
- [ ] 错误率 < 0.1%
- [ ] QPS 达到预期
```

---

*性能优化实战指南 | 10 年运维经验沉淀*