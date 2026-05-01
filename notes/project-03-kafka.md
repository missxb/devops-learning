# 项目三：Kafka 集群部署及监控

## 📋 项目概述

Apache Kafka 是一个分布式流处理平台，提供高吞吐量、持久化的消息队列功能。本项目实现 Kafka 多节点集群部署，配置 ZooKeeper 协调服务，并建立完整的监控系统。

## 🎯 学习目标

- 理解 Kafka 核心架构组件
- 掌握 Kafka 集群部署流程
- 配置 Producer/Consumer 最佳实践
- 实施实时监控和告警
- 优化性能和调优参数

## 🏗️ 架构设计

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Producer  │───►│  Broker 1   │◄───│  Broker 2   │
│             │    │ (znode1)    │    │ (znode2)    │
└─────────────┘    │ [Topic A]   │    │ [Topic B]   │
                   └──────┬──────┘    └──────┬──────┘
                          │                  │
          ┌───────────────┼──────────────────┼───────────────┐
          ▼               ▼                  ▼               ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │ Consumer    │  │ Consumer    │  │ Consumer    │  │ Consumer    │
   │ Group 1     │  │ Group 2     │  │ Group 3     │  │ Group 4     │
   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    ZooKeeper Ensemble                           │
│              /opt/zookeeper/data                                │
│         Session: 10s, Tick: 2s                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 📦 核心概念

| 概念 | 说明 |
|------|------|
| **Broker** | Kafka 服务器节点，存储分区数据 |
| **Topic** | 消息分类主题，分为多个 Partition |
| **Partition** | Topic 的物理分片，顺序写入磁盘 |
| **Replica** | 分区副本，包含 Leader 和 Follower |
| **Producer** | 消息生产者，发布到 Topic |
| **Consumer** | 消息消费者，从 Subscription Group 读取 |
| **Consumer Group** | 一组消费者，协作消费分区 |
| **ZooKeeper** | 分布式协调服务，管理集群元数据 |
| **Offset** | 消息在 Partition 中的位置标识 |

## 🔧 快速部署

### 1. 环境要求

- JDK 8+ (推荐 OpenJDK 11)
- 至少 3 台服务器
- CentOS 7.x / Ubuntu 20.04+

### 2. 安装 Java

```bash
# 安装 OpenJDK 11
yum install java-11-openjdk-devel -y

# 验证版本
java -version
```

### 3. 下载 Kafka

```bash
KAFKA_VERSION="3.5.0"
cd /opt
wget https://archive.apache.org/dist/kafka/${KAFKA_VERSION}/kafka_2.13-${KAFKA_VERSION}.tgz
tar -xzf kafka_2.13-${KAFKA_VERSION}.tgz
ln -s kafka_2.13-${KAFKA_VERSION} kafka
```

### 4. ZooKeeper 配置

```properties
# /opt/zookeeper/conf/zoo.cfg
dataDir=/var/lib/zookeeper/data
clientPort=2181
initLimit=5
syncLimit=2
maxClientCnxns=60

# 集群模式配置
server.1=zookeeper1:2888:3888
server.2=zookeeper2:2888:3888
server.3=zookeeper3:2888:3888

# 创建 myid 文件
echo "1" > /var/lib/zookeeper/data/myid
echo "2" >> /var/lib/zookeeper/data/myid
echo "3" >> /var/lib/zookeeper/data/myid
```

### 5. Kafka Broker 配置

```properties
# /opt/kafka/config/server.properties
############################# Server Basics #############################
broker.id=0
num.network.threads=3
num.io.threads=8

############################# Socket Server Settings #############################
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://kafka1.example.com:9092
listener.name.plainsecurity.sasl.enabled.mechanisms=PLAIN
netty.backpressure.maxHeapSizeBytes=134217728

############################# Log Basics #############################
log.dirs=/var/lib/kafka/data
num.partitions=1
num.recovery.threads.per.data.dir=1

############################# Replication #############################
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
min.insync.replicas=2
default.replication.factor=3

############################# Zookeeper #############################
zookeeper.connect=zookeeper1.example.com:2181,zookeeper2.example.com:2181,zookeeper3.example.com:2181
zookeeper.connection.timeout.ms=18000
zookeeper.set.acl=
group.initial.rebalance.delay.ms=0

############################# Log Flush Policy #############################
log.flush.interval.messages=10000
log.flush.interval.ms=1000

############################# Log Retention Policy #############################
log.retention.hours=168
log.retention.check.interval.ms=300000
log.retention.bytes=1073741824
log.segment.bytes=104857600
log.segment.delete.delay.ms=60000

############################# GC Tuning #############################
auto.create.topics.enable=true
delete.topic.enable=true
```

### 6. 启动服务

```bash
# 启动 ZooKeeper
/opt/kafka/bin/zookeeper-server-start.sh -daemon /opt/zookeeper/conf/zoo.cfg

# 等待 ZooKeeper 就绪
sleep 5

# 启动 Kafka Broker
/opt/kafka/bin/kafka-server-start.sh -daemon /opt/kafka/config/server.properties

# 检查进程
jps -l | grep -E 'ZooKeeperServerMain|Kafka'
```

### 7. 创建 Topic

```bash
# 创建测试 topic（3 副本，3 个分区）
./bin/kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic test-topic \
  --replication-factor 3 \
  --partitions 3

# 查看 Topic 详情
./bin/kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic test-topic

# 列出所有 Topic
./bin/kafka-topics.sh --list \
  --bootstrap-server localhost:9092
```

## ✅ 生产环境客户端配置

### Python Producer

```python
from kafka import KafkaProducer
import json
import time
from kafka.admin import KafkaAdminClient, NewTopic

class KafkaProducerClient:
    def __init__(self, brokers, acks='all'):
        self.producer = KafkaProducer(
            bootstrap_servers=brokers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks=acks,  # all, 1, or 0
            retries=3,
            retry_backoff_ms=100,
            max_in_flight_requests_per_connection=5,
            compression_type='lz4',
            linger_ms=10,
            batch_size=16384,
            buffer_memory=33554432
        )
    
    def send_message(self, topic, key, value):
        """发送单条消息"""
        try:
            future = self.producer.send(
                topic, 
                key=key.encode() if key else None, 
                value=value
            )
            record_metadata = future.get(timeout=10)
            return {
                'topic': record_metadata.topic,
                'partition': record_metadata.partition,
                'offset': record_metadata.offset,
                'timestamp': record_metadata.timestamp
            }
        except Exception as e:
            print(f"Send error: {e}")
            raise
    
    def send_batch(self, topic, messages):
        """批量发送消息"""
        futures = []
        for msg in messages:
            future = self.producer.send(topic, value=msg)
            futures.append(future)
        
        results = []
        for i, future in enumerate(futures):
            try:
                record_metadata = future.get(timeout=10)
                results.append({
                    'index': i,
                    'offset': record_metadata.offset
                })
            except Exception as e:
                print(f"Batch send error: {e}")
        
        return results

# 使用示例
producer = KafkaProducerClient('kafka1:9092,kafka2:9092,kafka3:9092')
for i in range(100):
    producer.send_message('orders', f'order_{i}', {'order_id': i})
```

### Python Consumer with Auto Commit

```python
from kafka import KafkaConsumer
import json
import time
from kafka.admin import ConsumerGroupDescribeResult

class KafkaConsumerClient:
    def __init__(self, brokers, group_id, auto_offset_reset='earliest'):
        self.consumer = KafkaConsumer(
            bootstrap_servers=brokers,
            group_id=group_id,
            auto_offset_reset=auto_offset_reset,
            enable_auto_commit=True,
            auto_commit_interval_ms=1000,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            client_id=f'consumer-{group_id}'
        )
    
    def subscribe_topics(self, topics):
        """订阅主题"""
        self.consumer.subscribe(topics=topics)
    
    def consume_messages(self, timeout_seconds=30):
        """消费消息"""
        messages_consumed = []
        start_time = time.time()
        
        while time.time() - start_time < timeout_seconds:
            records = self.consumer.poll(timeout_ms=1000)
            for topic, partitions in records.items():
                for partition, msgs in partitions.items():
                    for msg in msgs:
                        messages_consumed.append({
                            'topic': msg.topic,
                            'partition': msg.partition,
                            'offset': msg.offset,
                            'key': msg.key,
                            'value': msg.value,
                            'timestamp': msg.timestamp
                        })
                        
                        # 手动提交 offset（如果需要）
                        # self.consumer.commit()
        
        return messages_consumed
    
    def get_current_offsets(self):
        """获取当前消费的偏移量"""
        assignments = self.consumer.position().items()
        return dict(assignments)
    
    def seek_to_beginning(self):
        """重置到起始位置"""
        topics = self.consumer.subscription()
        offsets = self.consumer.beginning_offsets(topics)
        self.consumer.assign([
            topic_partition
            for topic, part_offsets in offsets.items()
            for topic_partition in part_offsets
        ])
        self.consumer.seek_to_beginning()
    
    def close(self):
        """关闭消费者"""
        self.consumer.close()

# 使用示例
consumer = KafkaConsumerClient(
    brokers='kafka1:9092,kafka2:9092,kafka3:9092',
    group_id='analytics-group'
)
consumer.subscribe_topics(['orders'])
messages = consumer.consume_messages(timeout_seconds=60)
consumer.close()
```

## 📊 完整监控方案

### 1. Prometheus + Kafka Exporter

```yaml
# docker-compose.monitor.yml
version: '3.8'
services:
  kafka-exporter:
    image: danielqsj/kafka_exporter
    container_name: kafka_exporter
    command:
      - '--kafka.server=kafka1:9092'
      - '--kafka.server=kafka2:9092'
      - '--kafka.server=kafka3:9092'
    ports:
      - "9308:9308"
    restart: unless-stopped
  
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
  
  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

### 2. Prometheus 配置文件

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:

scrape_configs:
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']
    metrics_path: /kafka

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 3. Grafana Dashboard

导入 ID: `715` (Kafka Cluster Metrics)

### 4. 关键监控指标

```promql
# Broker Active Count
count(kafka_server_replicamanager_leadercount)

# Under Replicated Partitions
sum(kafka_server_replicamanager_underreplicatedpartitions)

# Controller Count
sum(kafka_controller_kafkacontroller_activecontrollercount)

# Offline Partitions
sum(kafka_controller_kafkacontroller_offlinepartitionscount)

# Bytes In Per Second
rate(kafka_network_requestmetrics_bytesin_total[5m])

# Bytes Out Per Second
rate(kafka_network_requestmetrics_bytessent_total[5m])

# Request Queue Time
histogram_quantile(0.99, rate(kafka_request_queue_time_milliseconds_bucket[5m]))

# ISR Shrink Rate
rate(kafka_server_replicamanager_isrshrinks_persecond_total[1m])

# ISR Expand Rate
rate(kafka_server_replicamanager_isrexpands_persecond_total[1m])
```

### 5. Alertmanager 规则

```yaml
# rules/kafka-alerts.yml
groups:
  - name: kafka_alerts
    rules:
      - alert: KafkaBrokerDown
        expr: kafka_server_socket_servername_connectedclients == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Kafka broker is down (instance {{ $labels.instance }})"
          description: "Kafka broker {{ $labels.broker }} on port {{ $labels.port }} has no connected clients."

      - alert: KafkaUnderReplicatedPartitions
        expr: kafka_server_replicamanager_underreplicatedpartitions > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Kafka under-replicated partitions"
          description: "{{ $value }} partitions are under-replicated."

      - alert: KafkaOfflinePartitions
        expr: kafka_controller_kafkacontroller_offlinepartitionscount > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka offline partitions detected"
          description: "{{ $value }} partitions are offline."

      - alert: KafkaControllerLoss
        expr: kafka_controller_kafkacontroller_activecontrollercount == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active controller in Kafka cluster"
          description: "Kafka cluster has lost its controller leader."

      - alert: KafkaHighConsumerLag
        expr: sum(kafka_consumer_lag) by (topic, group) > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High consumer lag detected"
          description: "Consumer group {{ $labels.group }} on topic {{ $labels.topic }} has high lag."
```

## 🔐 安全配置

### SSL/TLS 加密

```properties
# server-sasl-ssl.properties
listeners=SSL://0.0.0.0:9093
advertised.listeners=SSL://kafka1.example.com:9093

ssl.keystore.location=/var/ssl/private/kafka.server.keystore.jks
ssl.keystore.password=<keystore-password>
ssl.key.password=<key-password>
ssl.truststore.location=/var/ssl/private/kafka.server.truststore.jks
ssl.truststore.password=<truststore-password>

security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
    username="admin" \
    password="admin-secret";
```

### SASL 用户创建

```bash
# 创建 scram-sha-512 用户
openssl kpasswd -sha512 admin

# 编辑 JAAS 配置
cat > /opt/kafka/config/scram-users.properties <<EOF
users += User[kafka-admin, KAFKA_SASL_SCRAM_SHA_512_SECRET=admin-secret]
users += User[user1, KAFKA_SASL_SCRAM_SHA_512_SECRET=user1-secret]
users += User[user2, KAFKA_SASL_SCRAM_SHA_512_SECRET=user2-secret]
EOF

# 应用配置
./bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter \
  --add-config file:///opt/kafka/config/scram-users.properties
```

## 🚨 故障排查

### 常见问题与解决方案

#### 1. 生产者消息发送失败

```python
# 检查 Producer 配置
print(f"Max block: {producer._configs['max_block']}")
print(f"Buffer memory: {producer._configs['buffer_memory']}")

# 增加超时时间
producer = KafkaProducer(
    bootstrap_servers=brokers,
    request_timeout_ms=30000,
    delivery_timeout_ms=120000,
    linger_ms=50
)
```

#### 2. 消费者无法消费消息

```python
# 检查 Consumer Group 状态
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# 重置 Consumer Offset
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic test-topic --reset-offsets --to-beginning --execute
```

#### 3. Topic 创建失败

```bash
# 检查 ZooKeeper 连接
telnet zookeeper1 2181

# 检查 Broker 配置
jstat -gc <kafka_pid> 1000

# 查看日志
tail -f /opt/kafka/logs/server.log
```

#### 4. Broker 启动失败

```bash
# 检查端口占用
netstat -tlnp | grep 9092

# 检查磁盘空间
df -h /var/lib/kafka/data

# 清理临时文件
rm -rf /tmp/kafka-*
```

## 📈 性能调优

### Producer 优化

```python
# 高性能 Producer 配置
producer = KafkaProducer(
    # 压缩
    compression_type='lz4',
    
    # 批量发送
    batch_size=32768,  # 32KB
    linger_ms=5,       # 等待 5ms 凑批
    buffer_memory=67108864,  # 64MB
    
    # 可靠性
    acks='all',        # 所有副本确认
    retries=3,
    max_in_flight_requests_per_connection=1,
    
    # 超时
    request_timeout_ms=30000,
    delivery_timeout_ms=120000
)
```

### Consumer 优化

```python
# 高性能 Consumer 配置
consumer = KafkaConsumer(
    group_id='group-id',
    auto_offset_reset='latest',
    enable_auto_commit=False,  # 手动提交
    
    # 拉取配置
    max_poll_records=5000,
    max_poll_interval_ms=300000,
    fetch_max_bytes=52428800,  # 50MB
    fetch_min_bytes=1,
    
    # 内存
    receive_buffer_bytes=32768,
    socket_receive_buffer_bytes=65536
)
```

## 📚 参考资料

- [Apache Kafka 官方文档](https://kafka.apache.org/documentation/)
- [Kafka 最佳实践](https://www.confluent.io/blog/kafka-best-practices/)
- [Kafka Performance Tuning Guide](https://docs.google.com/document/d/1Hd7nYBQ4bUjWjD0cRw7XhZ7vP3vQ6Q6y6g6y6g6y6g6/edit)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
