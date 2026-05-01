# 项目四：Prometheus 监控系统

## 📋 项目概述

Prometheus 是一个开源的系统监控和告警系统，采用时序数据库存储数据。本项目实现完整的 Prometheus 监控体系，包括数据采集、存储、可视化和告警管理。

## 🎯 学习目标

- 理解 Prometheus 架构和组件
- 掌握 Exporter 部署和数据采集
- 配置告警规则 (Alertmanager)
- 实施 Grafana 可视化面板
- 实现多环境监控覆盖

## 🏗️ 架构设计

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Node      │     │ Application │     │   Service   │
│ Agent/Export│     │   SDK/Tags  │     │  Endpoints  │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                    Pushgateway                           │
│              (短期任务数据中转)                            │
└────────────────────────┬────────────────────────────────┘
                         │ Pull 方式
                         ▼
┌─────────────────────────────────────────────────────────┐
│                  Prometheus Server                        │
│              Storage: TSDB + Remote Storage                │
└────────────────┬────────────────────────────────────────┘
                 │
          ┌──────┼──────┐
          ▼             ▼
┌─────────────┐  ┌─────────────┐
│   Grafana   │  │ Alertmanager │
│ (可视化)    │  │ (告警管理)   │
└─────────────┘  └──────┬──────┘
                        │
                 ┌──────┼──────┐
                 ▼      ▼      ▼
               Email  Wechat  Webhook
```

## 📦 核心概念

| 概念 | 说明 |
|------|------|
| **Metrics** | 监控指标，唯一标识一个测量值 |
| **Label** | 键值对标签，描述数据维度 |
| **Metric Types** | Counter/Gauge/Histogram/Summary |
| **Target** | 被监控的端点 |
| **Job** | 逻辑分组，相同目标的集合 |
| **Scrape** | 拉取指标的周期操作 |
| **Rules** | 规则定义，用于生成告警和聚合 |
| **Alert** | 告警条件触发后的通知 |

## 🔄 指标类型详解

### 1. Counter（计数器）

```python
# 只增不减的指标
http_requests_total{method="GET", handler="/api/v1/query"} = 15234
node_cpu_seconds_total{cpu="0", mode="idle"} = 89234.56
kube_pod_container_status_restarts_total{pod="prometheus-0"} = 3
```

```python
# Prometheus 查询示例
rate(http_requests_total[5m])  # 每秒请求速率
sum(increase(kube_pod_container_status_restarts_total[24h]))  # 24 小时重启总数
```

### 2. Gauge（仪表盘）

```python
# 可增可减的指标
node_memory_MemAvailable_bytes = 2147483648
process_resident_memory_bytes = 536870912
prometheus_tsdb_storage_blocks_bytes = 1073741824
temperature_sensor_01 = 25.6
```

```python
# Prometheus 查询示例
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100  # 可用内存百分比
avg(process_resident_memory_bytes) by (job)  # 按 job 平均内存使用
```

### 3. Histogram（直方图）

```python
# 统计分布数据
http_request_duration_seconds_bucket{le="0.1"} = 1024
http_request_duration_seconds_bucket{le="0.5"} = 4096
http_request_duration_seconds_bucket{le="1.0"} = 8192
http_request_duration_seconds_sum = 2048.5
http_request_duration_seconds_count = 12288
```

```python
# Prometheus 查询示例
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))  # P95 延迟
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])  # 平均响应时间
```

### 4. Summary（摘要）

```python
# 预计算的百分位数
http_request_duration_seconds{quantile="0.5"} = 0.023
http_request_duration_seconds{quantile="0.95"} = 0.089
http_request_duration_seconds{quantile="0.99"} = 0.156
```

## 🔧 快速部署

### 1. Docker Compose 一键部署

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules/:/etc/prometheus/rules/
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--storage.tsdb.retention.size=10GB'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    restart: unless-stopped
  
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml
    ports:
      - "9093:9093"
    restart: unless-stopped
  
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
    restart: unless-stopped
  
  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped
  
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
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
  scrape_timeout: 10s
  external_labels:
    cluster: 'production'
    region: 'cn-north-1'

rule_files:
  - 'alert_rules/*.yml'
  - 'recording_rules/*.yml'

scrape_configs:
  # Prometheus 自身监控
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
        labels:
          environment: 'production'
  
  # Node Exporter - 主机监控
  - job_name: 'node_exporter'
    static_configs:
      - targets: 
          - 'node-exporter:9100'
          - 'server1.example.com:9100'
          - 'server2.example.com:9100'
        labels:
          datacenter: 'dc1'
  
  # Kube State Metrics - Kubernetes 监控
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
  
  # Cadvisor - Docker 容器监控
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
        labels:
          instance_type: 'docker-host'
  
  # Blackbox Exporter - 服务可用性探测
  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - http://example.com
          - https://api.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
  
  # MySQL Exporter - 数据库监控
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']
        labels:
          environment: 'production'
  
  # Redis Exporter - 缓存监控
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
        labels:
          redis_instance: 'main-redis'
  
  # Kafka Exporter - 消息队列监控
  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']
        labels:
          kafka_cluster: 'prod-cluster'

# 远程读写配置
remote_write:
  - url: 'http://thanos-receive:19291/api/v1/receive'
    remote_timeout: 30s
    write_relabel_configs:
      - source_labels: [__name__]
        regex: 'go_.*|process_.*'
        action: drop

remote_read:
  - url: 'http://thanos-query:19090/api/v1/read'
    remote_timeout: 30s
```

### 3. 告警规则配置

```yaml
# alert_rules/node_alerts.yml
groups:
  - name: node_alerts
    rules:
      # CPU 使用率过高
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
          category: resource
        annotations:
          summary: "High CPU usage detected on {{ $labels.instance }}"
          description: "CPU usage is above 85% for more than 10 minutes. Current value: {{ $value }}%"
          runbook_url: "https://wiki.company.com/runbooks/high-cpu"
      
      # 内存使用率过高
      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 5m
        labels:
          severity: critical
          category: resource
        annotations:
          summary: "High memory usage detected on {{ $labels.instance }}"
          description: "Memory usage is above 90%. Current value: {{ $value }}%"
      
      # 磁盘空间不足
      - alert: DiskSpaceRunningLow
        expr: (1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100 > 85
        for: 15m
        labels:
          severity: warning
          category: storage
        annotations:
          summary: "Disk space running low on {{ $labels.instance }} {{ $labels.mountpoint }}"
          description: "Disk usage is above 85%. Current value: {{ $value }}%"
      
      # 文件系统读取错误
      - alert: FilesystemErrors
        expr: rate(node_disk_io_time_weighted_seconds_total[5m]) > 100
        for: 5m
        labels:
          severity: warning
          category: storage
        annotations:
          summary: "Filesystem I/O errors on {{ $labels.instance }}"
          description: "High number of filesystem I/O errors detected."
      
      # 网络接收丢包
      - alert: NetworkRxPacketDrops
        expr: rate(node_network_receive_drop_total[5m]) > 0.01
        for: 10m
        labels:
          severity: warning
          category: network
        annotations:
          summary: "Network RX packet drops on {{ $labels.instance }} {{ $labels.device }}"
          description: "Network receive packet drops detected."
      
      # 系统负载过高
      - alert: SystemLoadHigh
        expr: node_load1 > node_cpu_core_count * 2
        for: 15m
        labels:
          severity: warning
          category: resource
        annotations:
          summary: "System load is high on {{ $labels.instance }}"
          description: "1-minute load average exceeds twice the number of CPU cores."

  - name: service_alerts
    rules:
      # HTTP 5xx 错误率过高
      - alert: HighHTTPErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          category: application
        annotations:
          summary: "High HTTP error rate detected"
          description: "{{ $value | humanizePercentage }} of HTTP requests are failing with 5xx status."
      
      # 应用服务不可用
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          category: availability
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "Service {{ $labels.instance }} has been down for more than 1 minute."
      
      # Prometheus 抓取失败
      - alert: PrometheusTargetDown
        expr: up == 0 and not (job =~ 'alertmanager|grafana')
        for: 5m
        labels:
          severity: warning
          category: monitoring
        annotations:
          summary: "Prometheus target down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."
```

### 4. Alertmanager 配置

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.company.com:587'
  smtp_from: 'alerts@company.com'
  smtp_auth_username: 'alerts@company.com'
  smtp_auth_password: 'password'
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
  wechat_api_key: 'corp-secret-key'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
    - match:
        severity: warning
      receiver: 'warning-receiver'
      repeat_interval: 8h

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'ops-team@company.com'
        send_resolved: true
  
  - name: 'critical-alerts'
    wechat_configs:
      - agent_id: '1000001'
        corp_id: 'your-corp-id'
        api_secret: 'your-api-secret'
        message: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        send_resolved: true
    
    email_configs:
      - to: 'oncall-team@company.com'
        send_resolved: true

  - name: 'warning-receiver'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'  # 使用环境变量
        channel: '#monitoring-warnings'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

## 📊 Grafana 仪表板

### 推荐仪表板 ID

| 仪表板 | ID | 用途 |
|--------|-----|------|
| Node Exporter Full | 1860 | Linux 主机监控 |
| Prometheus Stats | 3662 | Prometheus 状态监控 |
| Dashboard for Container Monitoring | 893 | Docker 容器监控 |
| Kubernetes Cluster | 16210 | 集群总览 |
| Pod Status | 6417 | 容器状态监控 |

### 自定义 Dashboards JSON

```json
{
  "dashboard": {
    "id": null,
    "uid": "app-overview",
    "title": "Application Overview",
    "tags": ["application", "overview"],
    "timezone": "browser",
    "schemaVersion": 38,
    "version": 1,
    "refresh": "30s",
    "panels": [
      {
        "title": "QPS",
        "type": "timeseries",
        "datasource": {
          "type": "prometheus",
          "uid": "prometheus"
        },
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (handler)",
            "legendFormat": "{{ handler }}",
            "refId": "A"
          }
        ]
      },
      {
        "title": "P95 Latency",
        "type": "gauge",
        "datasource": {
          "type": "prometheus",
          "uid": "prometheus"
        },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, handler))",
            "legendFormat": "{{ handler }}",
            "refId": "A"
          }
        ]
      }
    ]
  }
}
```

## ✅ 最佳实践

### 1. 监控分层设计

```
第一层：基础设施监控 (Infrastructure Layer)
  ├── 物理服务器 / VM
  ├── 网络设备
  └── 虚拟化平台

第二层：中间件监控 (Middleware Layer)
  ├── 数据库 (MySQL, PostgreSQL, MongoDB)
  ├── 缓存 (Redis, Memcached)
  ├── 消息队列 (Kafka, RabbitMQ)
  └── 负载均衡 (Nginx, HAProxy)

第三层：应用层监控 (Application Layer)
  ├── 业务 API
  ├── 微服务调用链
  └── 前端性能

第四层：业务层监控 (Business Layer)
  ├── 转化率
  ├── 营收指标
  └── 用户行为
```

### 2. 命名规范

```yaml
# Metric 命名规则
# <子系统>_<指标名称>_{单位}_{可选后缀}

# 示例
# 正确命名
http_requests_total          # HTTP 请求总数
http_response_duration_seconds  # HTTP 响应时间
database_connections_active  # 活跃数据库连接数
cpu_usage_percent            # CPU 使用率百分比

# 避免的命名
RequestCount         # 不符合前缀规则
request_total        # 缺少具体上下文
duration             # 没有单位和前缀
```

### 3. 采集频率设计

| 层级 | 采集间隔 | 保留时长 | 说明 |
|------|----------|----------|------|
| 基础设施 | 15s | 15 天 | 资源监控需要高频 |
| 中间件 | 30s | 7 天 | 服务间通信波动小 |
| 应用 | 30s | 7 天 | 业务指标 |
| 业务 | 5min | 90 天 | 宏观趋势分析 |

### 4. 高可用部署

```yaml
# 主从 Prometheus 部署
prometheus:
  replicas: 3
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 500Gi
  config:
    remote_storage:
      - url: 'http://thanos-store-gateway:10901/api/v1/receive'
```

## 🚨 故障排查

### 常见问题

#### 1. 数据抓取失败

```bash
# 检查 Target 状态
curl -s localhost:9090/api/v1/targets

# 查看日志
tail -f /var/log/prometheus/prometheus.log

# 手动测试端点
curl http://target:9100/metrics
```

#### 2. 存储空间不足

```bash
# 清理旧数据
kubectl delete pods prometheus-prometheus-0

# 调整保留策略
curl -X POST http://localhost:9090/api/v1/admin/tsdb/delete_series \
  -H 'Content-Type: application/json' \
  -d '{"match":["up[5m]"]}""}'

# 启用远端写入
curl -X POST http://localhost:9090/-/reload
```

#### 3. 告警风暴

```python
# 在 Alertmanager 中设置抑制规则
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

## 📈 优化建议

### 1. PromQL 优化

```promql
# ❌ 避免的写法
sum(rate(http_requests_total[5m])) by (job) / sum(up)

# ✅ 推荐的写法
sum(rate(http_requests_total{job=~".*api.*"}[5m])) by (handler) / on() count(up)
```

### 2. 记录规则

```yaml
# recording_rules/rate_rules.yml
groups:
  - name: rate_recording_rules
    interval: 15s
    rules:
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
      
      - record: job:http_errors:ratio5m
        expr: >
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)
      
      - record: instance:node_cpu_utilization:rate5m
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

## 📚 参考资料

- [Prometheus 官方文档](https://prometheus.io/docs/)
- [Grafana 文档](https://grafana.com/docs/)
- [Alertmanager 文档](https://prometheus.io/docs/alerting/latest/configuration/)
- [Node Exporter 指标参考](https://github.com/prometheus/node_exporter)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
