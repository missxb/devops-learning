# 企业实战项目四：监控系统 Prometheus Stack 部署实战

## 📋 项目背景

**客户:** 某金融科技公司  
**监控需求:** 
- 全面覆盖基础设施、应用、业务指标
- 支持告警自动响应
- Grafana 统一可视化  
- 满足等保三级要求

---

## 🏗️ 监控架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                   监控体系架构                               │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  数据源层                              │  │
│  │  K8s / EC2 / RDS / Redis / Kafka / App              │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│  ┌────────────────────────▼──────────────────────────────┐  │
│  │                 采集层                                  │  │
│  │  ├── Node Exporter (主机指标)                         │  │
│  │  ├── Blackbox Exporter (HTTP/TCP/ICMP)               │  │
│  │  ├── cAdvisor (容器指标)                              │  │
│  │  └── PushGateway (临时任务指标)                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│  ┌────────────────────────▼──────────────────────────────┐  │
│  │                 存储计算层                              │  │
│  │  ├── Prometheus TSDB (时序数据库)                     │  │
│  │  ├── Thanos (分布式存储 + 查询加速)                    │  │
│  │  └── Alertmanager (告警管理)                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│  ┌────────────────────────▼──────────────────────────────┐  │
│  │                 展示告警层                              │  │
│  │  ├── Grafana Dashboard                                │  │
│  │  ├── Alertmanager Webhook                             │  │
│  │  └── PagerDuty 集成                                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 部署实施

### 第一阶段：基础监控 (3 天)

#### 1.1 节点监控配置

```yaml
# node-exporter-daemonset.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      containers:
      - name: node-exporter
        image: quay.io/prometheus/node-exporter:v1.5.0
        args:
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
        volumeMounts:
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /host/root
          readOnly: true
        ports:
        - containerPort: 9100
          protocol: TCP
      hostPID: true
      volumes:
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
    protocol: TCP
  selector:
    app: node-exporter
```

#### 1.2 应用监控配置

```yaml
# application-metrics.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-app-metrics
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-metrics
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:latest
        env:
        # Micrometer 暴露 JVM 指标
        - name: MANAGEMENT_ENDPOINT_METRICS_ENABLED
          value: "true"
        - name: MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED
          value: "true"
```

---

### 第二阶段：Prometheus 部署 (5 天)

#### 2.1 Prometheus 配置优化

```yaml
# prometheus-values.yaml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

prometheus:
  prometheusSpec:
    replicaCount: 2
    retention: 15d
    
    # 资源限制
    resources:
      requests:
        memory: 4Gi
        cpu: 1
      limits:
        memory: 8Gi
        cpu: 2
    
    # 持久化存储
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 200Gi
    
    # 服务发现
    serviceDiscoveryFunctions:
    - kubernetes:
        namespaces:
          names:
          - production
          - staging
          - middleware
        
    # 外部标签
    externalLabels:
      cluster: production
      
    # 报警阈值
    ruleSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false
    
    # 额外参数
    extraArgs:
    - storage.tsdb.retention.time=15d
    - storage.tsdb.wal-compression=true
    - web.enable-lifecycle
```

#### 2.2 自定义抓取配置

```yaml
# custom-scrapes.yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: custom-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      monitor: app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    scrapeTimeout: 10s
    honorLabels: true
    
    # 指标过滤
    metricRelabelings:
    # Drop 高基数指标
    - action: drop
      sourceLabels: [__name__]
      regex: 'jvm_gc_*_count{.*}'
      
    # 重命名指标
    - action: replace
      replacement: $1
      sourceLabels: ['instance']
      targetLabel: pod
  
  namespaceSelector:
    matchNames:
    - production
    - staging
```

---

### 第三阶段：告警系统 (5 天)

#### 3.1 Alertmanager 配置

```yaml
# alertmanager-values.yaml

alertmanager:
  alertmanagerSpec:
    global:
      resolve_timeout: 5m
      
    route:
      group_by: ['alertname', 'severity']
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 4h
      
      receiver: default
      
      routes:
      # P0 级别告警（立即通知）
      - match:
          severity: critical
        receiver: critical-pager
        continue: true
        
      # P1 级别告警（钉钉通知）
      - match:
          severity: warning
        receiver: dingtalk-warning
        
      # P2 级别告警（邮件通知）
      - match:
          severity: info
        receiver: email-info
        
    receivers:
    # DingTalk 企业微信机器人
    - name: dingtalk-warning
      dingtalk_configs:
      - api_url: https://oapi.dingtalk.com/robot/send?access_token=xxx
        secret: xxx
        message: |
          {{ range .Alerts }}
          **告警**: {{ .Annotations.summary }}
          **级别**: {{ .Labels.severity }}
          **开始时间**: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
          **详情**: {{ .Annotations.description }}
          {{ end }}
          
    # 邮件通知
    - name: email-info
      email_configs:
      - to: ops-team@company.com
        html: |
          <h1>{{ len .Alerts }}</h1>
          <p>告警列表:</p>
          <ul>
          {{ range .Alerts }}
          <li>{{ .Annotations.summary }}</li>
          {{ end }}
          </ul>
          
    # PagerDuty 集成
    - name: critical-pager
      pagerduty_configs:
      - routing_key: your-routing-key
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
```

#### 3.2 告警规则定义

```yaml
# alert-rules.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: infrastructure-alerts
  namespace: monitoring
spec:
  groups:
  # 基础设施告警
  - name: infrastructure.rules
    rules:
    # CPU 使用率过高
    - alert: HighCPUUsage
      expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "CPU 使用率超过 80% (实例：{{ $labels.instance }})"
        description: "CPU 使用率：{{ $value }}%, 持续时间：{{ $for }}"
        
    # 内存使用率过高
    - alert: HighMemoryUsage
      expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "内存使用率超过 85%"
        
    # 磁盘空间不足
    - alert: DiskSpaceLow
      expr: (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "磁盘空间不足 (实例：{{ $labels.instance }}, 挂载点：{{ $labels.mountpoint }})"
        
    # 网络带宽饱和
    - alert: NetworkBandwidthHigh
      expr: rate(node_network_receive_bytes_total[5m]) * 8 / 1000000000 > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "网络带宽使用率高"
        
    # 节点宕机
    - alert: NodeNotReady
      expr: kube_node_status_condition{condition="Ready",status="true"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Kubernetes 节点不健康"
        
  # 应用监控告警
  - name: application.rules
    rules:
    # 错误率过高
    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "错误率超过 5%"
        
    # 响应时间慢
    - alert: HighLatency
      expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 2
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "P95 响应时间超过 2 秒"
        
    # Pod 频繁重启
    - alert: PodRestartLoop
      expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod 在 1 小时内重启超过 5 次"
```

---

## 📊 Grafana Dashboard 模板

### 必备 Dashboard

```markdown
# 核心 Dashboard

## 1. 集群总览 Dashboard (ID: 3119)
├── 节点状态图
├── CPU/内存使用情况
├── Pod 数量趋势
└── 关键指标概览

## 2. 应用性能 Dashboard
├── QPS/TPS 趋势
├── 响应时间分布
├── 错误率统计
└── 资源消耗

## 3. 数据库监控 Dashboard
├── 连接数统计
├── Query 性能分析
├── 慢查询 TOP10
└── Buffer Pool 状态

## 4. 中间件监控 Dashboard
├── Kafka Topic 延迟
├── RabbitMQ Queue 积压
└── Redis 内存使用

## 5. SLA 仪表盘
├── 可用率 (99.9%)
├── 平均恢复时间 (MTTR)
├── 变更成功率
└── 告警统计
```

### Dashboard JSON 导入示例

```bash
#!/bin/bash
# import-dashboards.sh

# 集群总览
curl -s http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin123" \
  -d @dashboards/kubernetes-overview.json

# 应用性能
curl -s http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin123" \
  -d @dashboards/app-performance.json

# 数据库监控
curl -s http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin123" \
  -d @dashboards/database-monitoring.json
```

---

## 💰 成本优化

```
┌─────────────────────────────────────────────────────────────┐
│                   监控成本优化方案                           │
│                                                             │
│  【原始方案】                                                │
│  ├── Prometheus 全量采集 → 高额存储成本                      │
│  ├── 保留 30 天             → ¥50 万/年                       │
│  └── 总计：¥50 万/年                                        │
│                                                             │
│  【优化后方案】                                              │
│  ├── 缩短保留期至 7 天       → ¥10 万/年                      │
│  ├── PrometheusThanos     → 冷数据存储到对象存储 ¥5 万/年   │
│  ├── 指标采样策略           → 降低 100MB/s 写入               │
│  └── 总计：¥15 万/年                                       │
│                                                             │
│  【节省效果】                                                │
│  ├── 存储成本减少 70%                                      │
│  ├── CPU 资源减少 50%                                      │
│  └── 总体节省：¥35 万/年 (70%)                              │
└─────────────────────────────────────────────────────────────┘
```

---

*监控系统 Prometheus Stack 部署实战 | ClawsOps ⚙️*