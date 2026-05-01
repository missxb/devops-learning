# 项目二：RabbitMQ 3.8 镜像队列集群

## 📋 项目概述

RabbitMQ 是一个广泛使用的消息队列中间件，支持多种消息协议。本项目实现 RabbitMQ 3.8 版本的多节点镜像队列集群部署，确保高可用性和消息不丢失。

## 🎯 学习目标

- 理解 RabbitMQ 架构和核心概念
- 掌握镜像队列（Mirrored Queues）配置
- 实现 RabbitMQ 集群部署
- 配置负载均衡和高可用
- 实施监控和告警

## 🏗️ 架构设计

```
                    ┌────────────────────┐
                    │   Load Balancer    │
                    │   (Nginx/HAProxy)  │
                    └──────────┬─────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
     ┌────────▼────────┐ ┌─────▼─────┐ ┌───────▼───────┐
     │  RabbitMQ Node1 │ │Node2      │ │    Node3      │
     │  - Management   │ │- Management│ │ - Management  │
     │  - Queue Master │ │- Mirrors  │ │ - Mirrors     │
     │  - Messages     │ │           │ │               │
     └─────────────────┘ └───────────┘ └───────────────┘
              │                │                │
              └────────────────┼────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Client Apps       │
                    │   (Producers/Consumers) │
                    └─────────────────────┘
```

## 📦 核心概念

| 概念 | 说明 |
|------|------|
| **Exchange** | 消息交换机，接收生产者消息并路由到队列 |
| **Queue** | 消息队列，存储实际的消息数据 |
| **Binding** | 绑定关系，连接 Exchange 和 Queue |
| **Virtual Host (vhost)** | 虚拟主机，逻辑隔离环境 |
| **Mirror Queue** | 镜像队列，多个节点同步副本 |
| **Shovel** | 跨集群消息传递工具 |
| **Federation** | 联邦，多个 Broker 互联 |

## 🔄 镜像队列工作原理

```
Primary Node:          Mirror Node 1:        Mirror Node 2:
[Message In] ──────► [Sync Copy]         [Sync Copy]
     │
     ▼
[Queue Consumer]
```

## 🔧 部署步骤

### 1. 前置要求

- 至少 3 台服务器
- CentOS 7.x / Ubuntu 18.04+
- Docker 19.03+ 或手动安装 Erlang + RabbitMQ

### 2. Erlang 环境配置

```bash
# 安装 Erlang
yum install epel-release -y
yum install erlang -y

# 验证版本
erl -eval 'erlang:display(erlang:system_info(otp_release)),halt().' -noshell
```

### 3. RabbitMQ 安装

```bash
# 添加 RabbitMQ 仓库
cat <<EOF > /etc/yum.repos.d/rabbitmq_rabbit-server.repo
[rabbit_server]
name=Rabbit Server
baseurl=https://rpm.rabbitmq.com/rabbitmq/rpm/el/8/x86_64
gpgcheck=1
enabled=1
gpgkey=https://raw.githubusercontent.com/rabbitmq/signing-keys-master/master/rabbitmq-packaging@rabbitmq.com.pub
EOF

# 安装 RabbitMQ
yum install rabbitmq-server -y
```

### 4. RabbitMQ 配置模板

```ini
# /etc/rabbitmq/rabbitmq.conf
# Cluster configuration
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
cluster_formation.classic_config.nodes.1 = rabbit@node1
cluster_formation.classic_config.nodes.2 = rabbit@node2
cluster_formation.classic_config.nodes.3 = rabbit@node3

# Mirror queue policy
default_queue_type = classic
cluster_partition_handling = autoheal
queue_master_locator = min-masters

# User management
loopback_users.guest = false

# Listeners
listeners.tcp.default = 5672
management.tcp.port = 15672

# Clustering
cluster_formation.election_strategy = least-leaders

# Logging
log.file.level = info
```

### 5. 启用插件

```bash
# 启动 RabbitMQ 服务
systemctl start rabbitmq-server
systemctl enable rabbitmq-server

# 查看状态
rabbitmqctl status

# 启用管理控制台
rabbitmq-plugins enable rabbitmq_management
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_federation
```

### 6. 创建镜像队列策略

```bash
# 查看所有 vhost
rabbitmqctl list_vhosts

# 创建镜像策略：所有队列在所有节点镜像
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}' --priority 1 --apply-to queues

# 创建基于主节点的镜像策略
rabbitmqctl set_policy ha-master "^" '{"ha-mode":"masters","ha-sync-mode":"automatic"}' --priority 2 --apply-to queues

# 查看策略
rabbitmqctl list_policies
```

### 7. 集群节点加入命令

```bash
# 在 node2 上执行
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app

# 在 node3 上执行
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@node1
rabbitmqctl start_app
```

### 8. 验证集群状态

```bash
# 查看集群成员
rabbitmqctl cluster_status

# 查看队列分布
rabbitmqctl list_queues name state messages consumers

# 检查镜像队列状态
rabbitmqctl list_queues name owner pid sync
```

## ✅ 客户端测试

### 使用 Python 客户端

```python
import pika

# 连接到集群（通过负载均衡器或指定多节点）
parameters = pika.ConnectionParameters(
    host='loadbalancer.example.com',
    port=5672,
    virtual_host='/',
    credentials=pika.PlainCredentials('admin', 'admin')
)

connection = pika.BlockingConnection(parameters)
channel = connection.channel()

# 声明队列（会触发镜像同步）
channel.queue_declare(queue='test_queue', durable=True)

# 发布消息
for i in range(10):
    channel.basic_publish(exchange='',
                          routing_key='test_queue',
                          body=f'Message {i}')
    print(f"Sent message {i}")

connection.close()
```

## 🔐 安全配置

### 1. 防火墙设置

```bash
# 开放必要端口
firewall-cmd --permanent --add-port={5672,15672,4369,25672}/tcp
firewall-cmd --reload

# Erlang RPC 端口
firewall-cmd --permanent --add-port=9100-9200/tcp
```

### 2. SSL/TLS 配置

```ini
# /etc/rabbitmq/rabbitmq.ssl.conf
listeners.ssl.default = 5671
ssl_options.certfile = /etc/rabbitmq/certs/server.crt
ssl_options.keyfile = /etc/rabbitmq/certs/server.key
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = true
```

### 3. 密码强度策略

```bash
# 修改密码
rabbitmqctl change_password guest newpassword

# 创建新用户
rabbitmqctl add_user admin securePassword123!
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```

## 📊 监控指标

### Prometheus Exporter

```yaml
# docker-compose.monitor.yml
version: '3.8'
services:
  prometheus-rabbitmq-exporter:
    image: oliver006/rabbitmq_exporter
    ports:
      - "9090:9090"
    environment:
      RABBITMQ_USER: admin
      RABBITMQ_PASS: admin
      RABBITMQ_API_URL: http://localhost:15672/api
    depends_on:
      - rabbitmq
  
  alertmanager:
    # Prometheus 告警配置
```

### Grafana Dashboard

导入 ID: `000000176` (RabbitMQ 官方仪表板)

## 🚨 故障排查

### 常见问题

1. **集群节点无法加入**
   ```bash
   # 检查 ERTS 版本一致性
   erl -eval 'io:fwrite("Erlang ~s\\n", [erlang:system_info(otp_release)]), halt().' -noshell
   
   # 清除节点信息后重新加入
   rabbitmqctl forget_cluster_node rabbit@node1
   ```

2. **镜像不同步**
   ```bash
   # 手动同步镜像队列
   rabbitmqctl sync_queue -p / my_queue
   ```

3. **磁盘空间不足**
   ```bash
   # 检查磁盘限制
   rabbitmq-disk-free-limit
   
   # 删除过期消息
   rabbitmq-lanes purge old-messages
   ```

4. **内存过高**
   ```bash
   # 调整内存阈值
   rabbitmqctl set_vm_memory_high_watermark.relative 0.5
   ```

## 📈 生产优化建议

### 1. 队列类型选择

```bash
# Quorum Queues（推荐，RabbitMQ 3.8+）
rabbitmqctl set_parameter ha-mode quorum

# 参数化队列创建
channel.queue_declare(queue='quorum_queue', 
                      arguments={'x-queue-type': 'quorum'})
```

### 2. 持久化配置

```python
# 确保消息和队列都持久化
channel.queue_declare(queue='my_queue', 
                      durable=True,
                      arguments={
                          'x-message-ttl': 86400000,  # 24 小时 TTL
                          'x-max-priority': 10
                      })
```

### 3. 发布确认机制

```python
# 开启 confirm mode
channel.confirm_select()

def on_ack(frame):
    if frame.method.delivery_tag in pending_messages:
        del pending_messages[frame.method.delivery_tag]
        
channel.add_ack_listener(on_ack)
```

## 📚 参考资料

- [RabbitMQ 官方文档](https://www.rabbitmq.com/documentation.html)
- [RabbitMQ Clustering Guide](https://www.rabbitmq.com/clustering.html)
- [Mirror Queues Documentation](https://www.rabbitmq.com/ha.html)
- [RabbitMQ Metrics](https://www.rabbitmq.com/monitoring.html)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
