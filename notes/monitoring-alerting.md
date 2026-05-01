# 监控告警体系建设

## 📋 概述

监控告警是运维体系的核心，通过可观测性三支柱 (Metrics/Logs/Traces) 实现系统状态的全面感知。

## 🎯 学习目标

- 掌握监控指标设计
- 部署 Prometheus 监控栈
- 配置告警规则和管理
- 构建可视化仪表盘
- 实施告警降噪和路由

## 🏗️ 监控体系架构

```
┌─────────────────────────────────────────────────────────────┐
│                      监控告警体系                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              数据采集层 (Collection)                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │Exporter │  │  Agent  │  │  SDK    │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              存储层 (Storage)                        │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │           Prometheus / VictoriaMetrics      │   │   │
│  │  │           (时序数据库 TSDB)                  │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              告警层 (Alerting)                       │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              Alertmanager                    │   │   │
│  │  │  告警去重/分组/抑制/路由                     │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              展示层 (Visualization)                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │Grafana  │  │  Dashboard│  │ 报表    │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              通知层 (Notification)                   │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ 邮件    │  │ 钉钉    │  │ 企业微信 │            │   │
│  │  │ 短信    │  │ 电话    │  │ Webhook │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 📦 Prometheus 部署

### 1. Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.40.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  alertmanager:
    image: prom/alertmanager:v0.24.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'

  grafana:
    image: grafana/grafana:9.3.0
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false

  node-exporter:
    image: prom/node-exporter:v1.5.0
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
      - '--path.rootfs=/rootfs'

volumes:
  prometheus_data:
  alertmanager_data:
  grafana_data:
```

### 2. Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    monitor: 'prod-monitor'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alerts/*.yml"

scrape_configs:
  # Prometheus 自监控
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 节点监控
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # Kubernetes 服务发现
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}
```

## 🚨 告警规则设计

### 1. 基础设施告警

```yaml
# alerts/infrastructure.yml
groups:
  - name: infrastructure
    rules:
      # CPU 使用率过高
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高 CPU 使用率 ({{ $labels.instance }})"
          description: "CPU 使用率超过 80% (当前值：{{ $value }}%)"

      # 内存使用率过高
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高内存使用率 ({{ $labels.instance }})"
          description: "内存使用率超过 85% (当前值：{{ $value }}%)"

      # 磁盘空间不足
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 < 15
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "磁盘空间不足 ({{ $labels.instance }})"
          description: "磁盘可用空间低于 15% (当前值：{{ $value }}%)"

      # 节点宕机
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "节点宕机 ({{ $labels.instance }})"
          description: "节点 {{ $labels.instance }} 已宕机超过 1 分钟"
```

### 2. 应用告警

```yaml
# alerts/application.yml
groups:
  - name: application
    rules:
      # 服务不可用
      - alert: ServiceDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务不可用 ({{ $labels.job }})"
          description: "服务 {{ $labels.job }} 无法访问"

      # 高错误率
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "高错误率"
          description: "HTTP 5xx 错误率超过 5% (当前值：{{ $value | humanizePercentage }})"

      # 高延迟
      - alert: HighLatency
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高延迟"
          description: "P95 延迟超过 1 秒 (当前值：{{ $value }}s)"

      # Pod 重启
      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod 重启循环 ({{ $labels.pod }})"
          description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} 在 5 分钟内重启 {{ $value }} 次"
```

### 3. 业务告警

```yaml
# alerts/business.yml
groups:
  - name: business
    rules:
      # 订单量异常
      - alert: OrderVolumeDrop
        expr: sum(rate(orders_total[1h])) < 10
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "订单量异常下降"
          description: "当前小时订单量低于 10 单 (当前值：{{ $value }})"

      # 支付成功率下降
      - alert: PaymentSuccessRateDrop
        expr: sum(rate(payment_success_total[5m])) / sum(rate(payment_attempt_total[5m])) < 0.95
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "支付成功率下降"
          description: "支付成功率低于 95% (当前值：{{ $value | humanizePercentage }})"
```

## 📧 Alertmanager 配置

### 1. 告警路由

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

templates:
  - 'templates/*.tmpl'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default-receiver'
  routes:
    # 紧急告警 - 电话通知
    - match:
        severity: critical
      receiver: 'phone-alert'
      continue: true
    
    # 警告告警 - 钉钉通知
    - match:
        severity: warning
      receiver: 'dingtalk-alert'
    
    # 数据库告警 - DBA 团队
    - match:
        team: database
      receiver: 'dba-team'
    
    # 夜间免打扰 (23:00-08:00)
    - match_re:
        severity: warning
      receiver: 'night-silent'
      active_time_intervals:
        - weekdays: [monday:friday]
          times:
            - start_time: "23:00"
              end_time: "08:00"

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'ops-team@example.com'
        send_resolved: true

  - name: 'phone-alert'
    webhook_configs:
      - url: 'http://phone-gateway:8080/alert'
        send_resolved: true

  - name: 'dingtalk-alert'
    webhook_configs:
      - url: 'http://dingtalk-webhook:8060/dingtalk/webhook1/send'
        send_resolved: true

  - name: 'dba-team'
    email_configs:
      - to: 'dba-team@example.com'
        send_resolved: true

  - name: 'night-silent'
    # 静默接收器，不发送任何通知

inhibit_rules:
  # 当节点宕机时，抑制该节点上的所有告警
  - source_match:
      alertname: NodeDown
    target_match_re:
      alertname: '.+'
    equal: ['instance']
```

### 2. 钉钉通知模板

```yaml
# templates/dingtalk.tmpl
{{ define "dingtalk.title" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "dingtalk.content" }}
{{ range .Alerts }}
**告警:** {{ .Annotations.summary }}
**级别:** {{ .Labels.severity }}
**时间:** {{ .StartsAt.Format "2006-01-02 15:04:05" }}
**详情:** {{ .Annotations.description }}
{{ end }}
{{ end }}
```

## 📊 Grafana 仪表盘

### 1. 节点监控仪表盘

```json
{
  "dashboard": {
    "title": "Node Exporter",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (irate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{ instance }}"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100",
            "legendFormat": "{{ instance }}"
          }
        ]
      },
      {
        "title": "Disk Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100",
            "legendFormat": "{{ instance }} - {{ mountpoint }}"
          }
        ]
      }
    ]
  }
}
```

### 2. 应用监控仪表盘

```json
{
  "dashboard": {
    "title": "Application Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{ service }}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ]
      },
      {
        "title": "Latency Percentiles",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P95"
          },
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "P99"
          }
        ]
      }
    ]
  }
}
```

## 🔧 告警优化

### 1. 告警降噪

```yaml
# 抑制规则
inhibit_rules:
  # 数据中心级别故障时，抑制该数据中心的所有告警
  - source_match:
      alertname: DatacenterDown
    target_match_re:
      alertname: '.+'
    equal: ['datacenter']

  # 当集群不可达时，抑制集群内所有 Pod 告警
  - source_match:
      alertname: ClusterUnreachable
    target_match:
      alertname: PodNotReady
    equal: ['cluster']
```

### 2. 告警分级

| 级别 | 响应时间 | 通知方式 | 示例 |
|------|----------|----------|------|
| **P0-Critical** | 5 分钟 | 电话 + 短信 + IM | 全站宕机 |
| **P1-High** | 15 分钟 | 短信 + IM | 核心功能不可用 |
| **P2-Medium** | 1 小时 | IM | 非核心功能异常 |
| **P3-Low** | 4 小时 | 邮件 | 性能下降 |

### 3. 值班轮岗

```yaml
# 值班表配置
mute_time_intervals:
  - name: night-time
    time_intervals:
      - weekdays: ['monday:friday']
        times:
          - start_time: '23:00'
            end_time: '08:00'
  
  - name: weekend
    time_intervals:
      - weekdays: ['saturday', 'sunday']

# 在路由中引用
routes:
  - match:
      severity: warning
    receiver: 'oncall-team'
    mute_time_intervals:
      - night-time
      - weekend
```

## 📚 参考资料

- [Prometheus 官方文档](https://prometheus.io/docs/)
- [Grafana 官方文档](https://grafana.com/docs/)
- [SRE 运维解密](https://sre.google/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
