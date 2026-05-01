# 可观测性体系建设

## 📋 概述

可观测性是运维的核心能力，让系统"透明化"。我的理念：**看不见的管不了，管不了的会出事**。

---

## 🏗️ 可观测性三支柱

### 三支柱模型

```
┌─────────────────────────────────────────────────────────────┐
│                   可观测性三支柱                              │
│                                                             │
│         ┌─────────────────────────────────────┐            │
│         │           Tracing (链路)             │            │
│         │  请求完整路径/依赖关系               │            │
│         │  Jaeger/Zipkin/SkyWalking            │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │           Logging (日志)             │            │
│         │  事件记录/错误追踪                   │            │
│         │  Loki/ELK/ClickHouse                 │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │          Metrics (指标)              │            │
│         │  数值监控/趋势分析                   │            │
│         │  Prometheus/InfluxDB                 │            │
│         └─────────────────────────────────────┘            │
│                                                             │
│  三支柱协同：                                               │
│  ├── Metrics 发现异常                                       │
│  ├── Tracing 定位位置                                       │
│  └── Logging 查看详情                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 Metrics 指标体系

### 指标分层设计

```
┌─────────────────────────────────────────────────────────────┐
│                    指标分层模型                              │
│                                                             │
│  Layer 1: 基础设施指标                                      │
│  ├── CPU/内存/磁盘/网络                                     │
│  ├── 节点状态                                               │
│  └── 系统负载                                               │
│                                                             │
│  Layer 2: 容器平台指标                                      │
│  ├── Pod 状态                                               │
│  ├── 容器资源                                               │
│  ├── K8s 组件                                               │
│  └── 调度延迟                                               │
│                                                             │
│  Layer 3: 应用指标                                          │
│  ├── QPS/TPS                                                │
│  ├── 响应时间                                               │
│  ├── 错误率                                                 │
│  ├── 并发数                                                 │
│                                                             │
│  Layer 4: 业务指标                                          │
│  ├── 订单量                                                 │
│  ├── 支付成功率                                             │
│  ├── 用户活跃                                               │
│  ├── 转化率                                                 │
└─────────────────────────────────────────────────────────────┘
```

### RED 方法 (应用指标)

```
┌─────────────────────────────────────────────────────────────┐
│                    RED 方法                                  │
│                                                             │
│  Rate (速率)                                                │
│  ├── QPS: 每秒请求数                                        │
│  ├── TPS: 每秒事务数                                        │
│  └── 流量趋势                                               │
│                                                             │
│  Errors (错误)                                              │
│  ├── 错误率: 失败请求占比                                   │
│  ├── 错误类型                                               │
│  ├── 错误分布                                               │
│                                                             │
│  Duration (延迟)                                            │
│  ├── P50: 中位数延迟                                        │
│  ├── P95: 95 分位延迟                                       │
│  ├── P99: 99 分位延迟                                       │
│  ├── MAX: 最大延迟                                          │
└─────────────────────────────────────────────────────────────┘
```

### USE 方法 (资源指标)

```
┌─────────────────────────────────────────────────────────────┐
│                    USE 方法                                  │
│                                                             │
│  Utilization (使用率)                                       │
│  ├── CPU 使用率                                             │
│  ├── 内存使用率                                             │
│  ├── 磁盘使用率                                             │
│  ├── 网络带宽使用率                                         │
│                                                             │
│  Saturation (饱和度)                                        │
│  ├── CPU 负载                                               │
│  ├── 内存压力                                               │
│  ├── IO 压力                                                │
│  ├── 网络拥塞                                               │
│                                                             │
│  Errors (错误)                                              │
│  ├── 系统错误                                               │
│  ├── 硬件错误                                               │
│  ├── 资源限制错误                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Prometheus 部署实战

### Prometheus Stack 安装

```yaml
# kube-prometheus-stack.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: prometheus-stack
  namespace: monitoring
spec:
  chart: kube-prometheus-stack
  repo: https://prometheus-community.github.io/helm-charts
  targetNamespace: monitoring
  
  valuesContent: |
    # Prometheus 配置
    prometheus:
      prometheusSpec:
        retention: 7d
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: standard
              resources:
                requests:
                  storage: 50Gi
        
        # 资源限制
        resources:
          limits:
            cpu: 2
            memory: 4Gi
          requests:
            cpu: 1
            memory: 2Gi
        
        # 压测配置
        scrapeInterval: 15s
        evaluationInterval: 15s
    
    # Grafana 配置
    grafana:
      adminPassword: admin123
      plugins:
      - grafana-piechart-panel
      - grafana-clock-panel
      
      # 数据源
      additionalDataSources:
      - name: Loki
        type: loki
        url: http://loki:3100
      - name: Alertmanager
        type: alertmanager
        url: http://alertmanager:9093
    
    # Alertmanager 配置
    alertmanager:
      alertmanagerSpec:
        storage:
          volumeClaimTemplate:
            spec:
              storageClassName: standard
              resources:
                requests:
                  storage: 10Gi
```

### 自定义指标采集

```yaml
# 自定义 ServiceMonitor

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
    
    # 指标重标签
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_name]
      targetLabel: pod
    - sourceLabels: [__meta_kubernetes_namespace]
      targetLabel: namespace
```

---

## 📝 Logging 日志体系

### Loki 部署

```yaml
# loki-stack.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: loki-stack
  namespace: logging
spec:
  chart: loki-stack
  repo: https://grafana.github.io/helm-charts
  targetNamespace: logging
  
  valuesContent: |
    loki:
      config:
        auth_enabled: false
        ingester:
          chunk_idle_period: 5m
          chunk_block_size: 262144
        schema_config:
          configs:
          - from: 2020-10-24
            store: boltdb-shipper
            object_store: filesystem
            schema: v11
            index:
              prefix: index_
              period: 24h
        
        limits_config:
          reject_old_samples: true
          reject_old_samples_max_age: 168h
    
    promtail:
      config:
        clients:
        - url: http://loki:3100/loki/api/v1/push
        
        scrape_configs:
        - job_name: kubernetes-pods
          kubernetes_sd_configs:
          - role: pod
          pipeline_stages:
          - docker: {}
          - labels:
              - stream
          - match:
              selector: '{app="myapp"}'
              stages:
              - regex:
                  expression: '^(?P<time>[^ ]+) (?P<level>[^ ]+) (?P<message>.+)$'
              - labels:
                  - level
```

---

## 🔍 Tracing 链路追踪

### Jaeger 部署

```yaml
# jaeger-operator.yaml

apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger
  namespace: tracing
spec:
  strategy: production
  
  # 存储配置
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
  
  # UI 配置
  ui:
    options:
      dependencies:
        menuEnabled: true
  
  # 采集配置
  ingress:
    enabled: true
    hosts:
    - jaeger.example.com
```

### 应用集成 Jaeger

```yaml
# Jaeger Agent Sidecar

apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  # 应用容器
  - name: app
    image: myapp:1.0
    env:
    - name: JAEGER_AGENT_HOST
      value: localhost
    - name: JAEGER_AGENT_PORT
      value: "6831"
    - name: JAEGER_SAMPLER_TYPE
      value: probabilistic
    - name: JAEGER_SAMPLER_PARAM
      value: "0.1"  # 10% 采样
    
  # Jaeger Agent Sidecar
  - name: jaeger-agent
    image: jaegertracing/jaeger-agent:1.35
    args:
    - --reporter.type=grpc
    - --reporter.grpc.host-port=jaeger-collector:14250
    ports:
    - containerPort: 6831
      protocol: UDP
    - containerPort: 6832
      protocol: UDP
    - containerPort: 5778
      protocol: HTTP
```

---

## 🚨 告警系统

### Alertmanager 配置

```yaml
# alertmanager-config.yaml

global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:25'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'password'

# 路由配置
route:
  group_by: ['alertname', 'cluster']
  group_wait: 30s        # 等待 30s 聚合告警
  group_interval: 5m     # 每 5m 发送一组
  repeat_interval: 4h    # 重复告警间隔 4h
  
  receiver: 'default'
  routes:
  # P0 告警
  - match:
      severity: critical
    receiver: 'critical'
    continue: true
  
  # P1 告警
  - match:
      severity: warning
    receiver: 'warning'

# 接收器配置
receivers:
- name: 'default'
  webhook_configs:
  - url: http://localhost:5001/

- name: 'critical'
  # 钉钉通知
  webhook_configs:
  - url: http://dingtalk-webhook:8060/dingtalk/critical/send
  # 邮件通知
  email_configs:
  - to: 'ops-team@example.com'
  # 电话通知 (集成第三方)
  webhook_configs:
  - url: http://sms-service:8080/call

- name: 'warning'
  # 钉钉通知
  webhook_configs:
  - url: http://dingtalk-webhook:8060/dingtalk/warning/send
  # 邮件通知
  email_configs:
  - to: 'ops-team@example.com'

# 抑制规则
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  equal: ['alertname', 'cluster']
```

---

## 📋 可观测性 Checklist

```markdown
# 可观测性建设 Checklist

## Metrics
- [ ] 基础设施指标采集
- [ ] 容器指标采集
- [ ] 应用指标暴露
- [ ] 业务指标定义
- [ ] 告警规则配置
- [ ] Grafana Dashboard

## Logging
- [ ] 日志采集配置
- [ ] 日志存储配置
- [ ] 日志检索配置
- [ ] 日志告警规则
- [ ] 日志 Dashboard

## Tracing
- [ ] Jaeger 部署
- [ ] 应用集成
- [ ] 采样配置
- [ ] 存储配置
- [ ] Dashboard

## Alerting
- [ ] 告警分级
- [ ] 告警路由
- [ ] 告警通知
- [ ] 告警抑制
- [ ] 告警静默
```

---

*可观测性体系建设 | 看不见的管不了*