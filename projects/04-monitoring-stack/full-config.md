# 监控系统 Prometheus Stack - 完整采集配置

## 完整 Node Exporter 配置

### 主机指标采集

```yaml
# node-exporter-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: node-exporter-config
  namespace: monitoring
data:
  scrape-config.yaml: |
    - job_name: 'node-exporter'
      static_configs:
      - targets:
        - 'node-exporter.monitoring:9100'
      
      relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):(.*)'
        target_label: instance
        replacement: '${1}'
      
      metric_relabel_configs:
      # 只保留关键指标
      - source_labels: [__name__]
        regex: '(node_cpu_seconds_total|node_memory_.*|node_disk_.*|node_network_.*|node_filesystem_.*|node_load1|node_load5|node_load15)'
        action: keep
```

### 完整 Prometheus 配置

```yaml
# prometheus-full-config.yaml

global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s
  
  external_labels:
    cluster: production
    datacenter: hangzhou-a
    environment: production

alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager.monitoring:9093

rule_files:
- /etc/prometheus/rules/*.yml

scrape_configs:
# 1. Prometheus 自身监控
- job_name: 'prometheus'
  static_configs:
  - targets: ['localhost:9090']
    labels:
      instance: prometheus-master

# 2. Node Exporter - 主机指标
- job_name: 'node-exporter'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [monitoring]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: keep
    regex: node-exporter

# 3. Kubelet 指标
- job_name: 'kubelet'
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  authorization:
    credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)

# 4. API Server 指标
- job_name: 'kubernetes-apiservers'
  kubernetes_sd_configs:
  - role: endpoints
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  authorization:
    credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: default;kubernetes;https

# 5. cAdvisor - 容器指标
- job_name: 'cadvisor'
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  authorization:
    credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'container_(network_tcp_usage_total|tasks_state|memory_failures_total)'
    action: keep

# 6. MySQL Exporter
- job_name: 'mysql'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [middleware]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: keep
    regex: mysql
  - source_labels: [__address__]
    replacement: '$1:9104'
    target_label: __address__

# 7. Redis Exporter
- job_name: 'redis'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [middleware]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: keep
    regex: redis
  - source_labels: [__address__]
    replacement: '$1:9121'
    target_label: __address__

# 8. Kafka Exporter
- job_name: 'kafka'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [middleware]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    action: keep
    regex: kafka
  - source_labels: [__address__]
    replacement: '$1:9308'
    target_label: __address__

# 9. 应用自定义指标 (Micrometer/JMX)
- job_name: 'application-metrics'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: [production, staging]
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_ip]
    action: replace
    regex: (\d+);(.+)
    replacement: '$2:$1'
    target_label: __address__
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace

# 10. Blackbox Exporter - 外部探测
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
  - targets:
    - https://api.ecommerce.com/health
    - https://www.ecommerce.com
  relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  - source_labels: [__param_target]
    target_label: instance
  - target_label: __address__
    replacement: blackbox-exporter.monitoring:9115

- job_name: 'blackbox-tcp'
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
  - targets:
    - 10.0.0.50:3306  # MySQL
    - 10.0.0.60:6379  # Redis
    - 10.0.0.70:9092  # Kafka
  relabel_configs:
  - source_labels: [__address__]
    target_label: __param_target
  - source_labels: [__param_target]
    target_label: instance
  - target_label: __address__
    replacement: blackbox-exporter.monitoring:9115

# 11. Thanos Sidecar (如果启用)
- job_name: 'thanos-sidecar'
  kubernetes_sd_configs:
  - role: pod
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_component]
    action: keep
    regex: thanos-sidecar
```

---

## 完整告警规则集

### 基础设施告警

```yaml
# infrastructure-alerts.yaml

groups:
- name: infrastructure.rules
  rules:
  
  # === 主机级别 ===
  
  - alert: HostHighCpuUsage
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 10m
    labels:
      severity: warning
      category: infrastructure
    annotations:
      summary: "主机 CPU 使用率过高 ({{ $labels.instance }})"
      description: "CPU 使用率 {{ $value }}%, 持续 10 分钟"
      
  - alert: HostHighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
    for: 10m
    labels:
      severity: warning
      category: infrastructure
    annotations:
      summary: "主机内存使用率过高 ({{ $labels.instance }})"
      
  - alert: HostDiskSpaceLow
    expr: (1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})) * 100 > 80
    for: 5m
    labels:
      severity: warning
      category: infrastructure
    annotations:
      summary: "主机磁盘空间不足 ({{ $labels.instance }}:{{ $labels.mountpoint }})"
      description: "磁盘使用率 {{ $value }}%"
      
  - alert: HostDiskWillFillIn24Hours
    expr: predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h], 24*3600) < 0
    for: 30m
    labels:
      severity: critical
      category: infrastructure
    annotations:
      summary: "磁盘将在 24 小时内写满 ({{ $labels.instance }})"
      
  - alert: HostNetworkReceiveSaturation
    expr: rate(node_network_receive_bytes_total[5m]) * 8 > 0.8 * node_network_speed_bytes{device!~"lo|docker.*|veth.*"}
    for: 5m
    labels:
      severity: warning
      category: infrastructure
    annotations:
      summary: "网络接收带宽饱和 ({{ $labels.instance }}:{{ $labels.device }})"
      
  - alert: HostHighLoad
    expr: node_load5 / count without(cpu, mode) (node_cpu_seconds_total{mode="idle"}) > 0.8
    for: 15m
    labels:
      severity: warning
      category: infrastructure
    annotations:
      summary: "主机负载过高 ({{ $labels.instance }})"

  # === K8s 级别 ===
  
  - alert: K8sNodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: critical
      category: kubernetes
    annotations:
      summary: "K8s 节点不可用 ({{ $labels.node }})"
      
  - alert: K8sPodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
    for: 15m
    labels:
      severity: warning
      category: kubernetes
    annotations:
      summary: "Pod 频繁重启 ({{ $labels.namespace }}/{{ $labels.pod }})"
      
  - alert: K8sPodNotReady
    expr: kube_pod_status_phase{phase=~"Pending|Failed"} > 0
    for: 15m
    labels:
      severity: warning
      category: kubernetes
    annotations:
      summary: "Pod 未就绪 ({{ $labels.namespace }}/{{ $labels.pod }})"
      
  - alert: K8sDeploymentReplicasMismatch
    expr: kube_deployment_spec_replicas != kube_deployment_status_replicas_available > 0
    for: 15m
    labels:
      severity: warning
      category: kubernetes
    annotations:
      summary: "Deployment 可用副本数不匹配 ({{ $labels.namespace }}/{{ $labels.deployment }})"
      
  - alert: K8sStatefulsetReplicasMismatch
    expr: kube_statefulset_status_replicas_ready != kube_statefulset_status_replicas
    for: 15m
    labels:
      severity: warning
      category: kubernetes
    annotations:
      summary: "StatefulSet 可用副本数不匹配"
      
  - alert: K8sPVCAlmostFull
    expr: kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100 > 80
    for: 5m
    labels:
      severity: warning
      category: kubernetes
    annotations:
      summary: "PVC 使用率过高 ({{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }})"

  # === 数据库级别 ===
  
  - alert: MysqlDown
    expr: mysql_up == 0
    for: 2m
    labels:
      severity: critical
      category: database
    annotations:
      summary: "MySQL 实例不可用 ({{ $labels.instance }})"
      
  - alert: MysqlReplicationLag
    expr: mysql_slave_seconds_behind_master > 30
    for: 5m
    labels:
      severity: critical
      category: database
    annotations:
      summary: "MySQL 主从延迟过高 ({{ $labels.instance }})"
      description: "延迟 {{ $value }} 秒"
      
  - alert: MysqlTooManyConnections
    expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100 > 80
    for: 5m
    labels:
      severity: warning
      category: database
    annotations:
      summary: "MySQL 连接数过多 ({{ $labels.instance }})"
      
  - alert: RedisDown
    expr: redis_up == 0
    for: 2m
    labels:
      severity: critical
      category: middleware
    annotations:
      summary: "Redis 实例不可用 ({{ $labels.instance }})"
      
  - alert: RedisOutOfMemory
    expr: redis_memory_used_bytes / redis_memory_max_bytes * 100 > 90
    for: 5m
    labels:
      severity: warning
      category: middleware
    annotations:
      summary: "Redis 内存使用率过高 ({{ $labels.instance }})"
      
  - alert: RedisReplicationLag
    expr: redis_connected_slaves == 0
    for: 5m
    labels:
      severity: warning
      category: middleware
    annotations:
      summary: "Redis 从节点断开 ({{ $labels.instance }})"
```

### 应用级别告警

```yaml
# application-alerts.yaml

groups:
- name: application.rules
  rules:
  
  - alert: HighHTTPRequestLatency
    expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m])) by (le, service)) > 2
    for: 5m
    labels:
      severity: warning
      category: application
    annotations:
      summary: "HTTP 请求 P95 延迟过高 ({{ $labels.service }})"
      
  - alert: HighHTTP5xxErrorRate
    expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100 > 5
    for: 5m
    labels:
      severity: critical
      category: application
    annotations:
      summary: "HTTP 5xx 错误率过高 ({{ $labels.service }})"
      
  - alert: HighJVMHeapUsage
    expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100 > 85
    for: 10m
    labels:
      severity: warning
      category: application
    annotations:
      summary: "JVM Heap 使用率过高 ({{ $labels.instance }})"
      
  - alert: HighGCTime
    expr: rate(jvm_gc_pause_seconds_sum[5m]) > 0.2
    for: 5m
    labels:
      severity: warning
      category: application
    annotations:
      summary: "GC 时间过长 ({{ $labels.instance }})"
      
  - alert: OrderCreationFailure
    expr: sum(rate(order_create_total{status="failed"}[5m])) > 10
    for: 5m
    labels:
      severity: critical
      category: business
    annotations:
      summary: "订单创建失败率过高"
      
  - alert: PaymentGatewayTimeout
    expr: sum(rate(payment_gateway_request_total{status="timeout"}[5m])) > 5
    for: 3m
    labels:
      severity: critical
      category: business
    annotations:
      summary: "支付网关超时次数过多"
```

---

*监控系统 Prometheus Stack - 完整配置 | ClawsOps ⚙️*