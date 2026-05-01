# 企业实战项目三：企业级日志平台建设 (ELK + Loki)

## 📋 项目背景

**客户:** 某电商平台  
**现状问题:** 
- 日志分散在各服务器，排查困难
- Elasticsearch 资源消耗大，维护成本高
- 搜索查询慢，无法实时分析
- 告警响应不及时  

**建设目标:**
- 统一日志采集和管理
- 支持实时搜索和分析
- 降低运维成本 50%
- 故障定位时间从小时级降至分钟级

---

## 🏗️ 技术方案选型

### ELK vs Loki 对比

```
┌─────────────────────────────────────────────────────────────┐
│                   日志系统方案对比                            │
│                                                             │
│                    ELK Stack          │    Loki             │
│  ───────────────────┼─────────────────┼────────────────     │
│  架构设计           │                 │                     │
│  ├── Elasticsearch │                 │  基于对象存储         │
│  ├── Logstash      │   成熟稳定      │  轻量级              │
│  └── Kibana        │                 │  Grafana UI          │
│                     │                 │                     │
│  索引策略           │                 │                     │
│  ├── 全文索引       │   索引全部字段   │  只索引标签         │
│  ├── 占用空间大     │   P99: ¥50万/年 │  只存原始数据       │
│  └── 查询灵活但慢   │                 │  快速聚合           │
│                     │                 │                     │
│  成本               │                 │                     │
│  ──────────────────┼─────────────────┼────────────────     │
│  存储成本           │   高 (SSD x8)    │   低 (HDD/x8)        │
│  CPU 成本           │   高 (索引开销)   │   低                │
│  维护成本           │   中等            │   低                │
│                     │                 │                     │
│  适用场景           │                 │                     │
│  ├── 完整搜索需求   │                 │  海量日志            │
│  ├── 复杂关联查询   │   适合大型企    │  监控告警            │
│  └── 深度日志分析   │   业场景         │  实时监控            │
│                     │                 │  成本敏感场景        │
│                     │                 │                     │
│  推荐决策           │                 │                     │
│  ──────────────────┼─────────────────┴────────────────     │
│  预算充足 + 需深度分析 → ELK                                   │
│  成本敏感 + 海量日志     → Loki                             │
│  混合方案                → ELK+Loki                           │
└─────────────────────────────────────────────────────────────┘
```

### 最终方案：Hybrid Stack (ELK + Loki)

```
┌─────────────────────────────────────────────────────────────┐
│                   Hybrid 日志架构图                          │
│                                                             │
│  实时日志采集 (Promtail)                                    │
│       ↓                                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │               日志分类路由                          │   │
│  └────────────────────────────────────────────────────┘   │
│          ↓                         ↓                       │
│  ┌──────────────┐         ┌──────────────┐                │
│  │   Loki       │         │    ELK       │                │
│  │ (Grafana)    │         │ (Kibana)     │                │
│  │              │         │              │                │
│  │ 热日志 <7天  │         │ 历史日志 >30天│               │
│  │ 用于监控告警 │         │ 用于深度分析  │                │
│  │              │         │              │                │
│  │ Minio/S3    │         │ Elasticsearch│               │
│  └──────────────┘         └──────────────┘                │
│                                                             │
│  使用比例建议：                                             │
│  ├── 60% 流量走 Loki (监控告警)                              │
│  ├── 40% 流量走 ELK (深度分析)                               │
│  └── 成本优化 50%+                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### 第一阶段：环境准备 (3 天)

#### 1.1 集群规划

```
┌─────────────────────────────────────────────────────────────┐
│                   日志系统资源分配                           │
│                                                             │
│  Loki 节点 (4 台)                                              │
│  ├── loki-0,0:1,0:2:  16C32G500G SSD                        │
│  ├── loki-0:3:         8C16G200G HDD                        │
│  └── MinIO S3 兼容存储：2TB                                 │
│                                                             │
│  ELK 节点 (6 台)                                               │
│  ├── elasticsearch-0~2: 16C64G1T SSD (主节点)                │
│  ├── logstash-0~1:     8C32G200G SSD                      │
│  └── kibana-0:          8C16G200G SSD                      │
│                                                             │
│  采集器：所有工作节点 + 宿主机                                  │
└─────────────────────────────────────────────────────────────┘
```

#### 1.2 Loki 配置

```yaml
# loki-values.yaml

loki:
  # 存储后端
  storage:
    type: filesystem
    
  # 分片配置
  replicas: 3
  
  # 查询引擎
  query_range:
    parallelise_shardable_queries: true
    
  # 压缩配置
  chunk_encoding: snappy
  chunk_store_config:
    index:
      prefix: loki_index_
      period: 2h
      
  # 保留策略
  retention_period: 7d
  
  # 流级别限流
  limits_config:
    enforce_metric_name: true
    reject_old_samples: true
    reject_old_samples_max_age: 168h
    max_cache_freshness_per_query: 10m
    
    # 租户配额
    per_stream_rate_limit: 10MB/s
    per_stream_rate_limit_burst: 15MB

promtail:
  # 数据采集配置
  config:
    clients:
    - url: http://loki:3100/loki/api/v1/push
    
    # 多源采集
    scrape_configs:
    # Kubernetes Pod 日志
    - job_name: kubernetes-pods
      pipeline_stages:
      - docker: {}
      - labels:
          stream:
          container:
      kubernetes_sd_configs:
      - role: pod
        
        # 按命名空间过滤
        selectors:
        - role: pod
          filters:
          - name: namespace
            action: equal
            value: production
        
    # 应用日志文件
    - job_name: application
      static_configs:
      - targets:
        - localhost
        labels:
          job: applogs
          __path__: /var/log/app/*.log
          
      # 日志解析
      pipeline_stages:
      - json:
          expressions:
            timestamp: time
            level: level
            message: msg
            trace_id: traceId
          source: message
            
      - regex:
          expression: '^(?P<timestamp>[^ ]+) (?P<level>[^ ]+) (?P<message>.+)$'
          
      # 重标签
      relabel_configs:
      - source_labels: [__path__]
        target_label: source
      - source_labels: [level]
        target_label: severity
```

---

### 第二阶段：部署实施 (5 天)

#### 2.1 安装 Loki

```bash
# 使用 Helm 安装

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --values loki-values.yaml \
  --set promtail.enabled=true \
  --set alertmanager.enabled=false \
  --set loki.resources.limits.memory=4Gi \
  --set loki.resources.requests.memory=2Gi \
  --set loki.persistence.size=500Gi
```

#### 2.2 安装 Prometheus + Alertmanager

```bash
# 监控告警栈
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.replicaExternalLabelName=prometheus_replica \
  --set alertmanager.alertmanagerSpec.externalUrl=http://alertmanager.example.com
```

#### 2.3 安装 Grafana Dashboard

```bash
# 导入 Dashboard

# 1. Loki 日志总览 (Dashboard ID: 13575)
curl http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin" \
  -H "Content-Type: application/json" \
  -d '{"dashboard":{"id":13575}}'

# 2. 日志统计 (Dashboard ID: 13576)
curl http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin" \
  -H "Content-Type: application/json" \
  -d '{"dashboard":{"id":13576}}'

# 3. 应用性能 (Dashboard ID: 13577)
curl http://grafana:3000/api/dashboards/import \
  -H "Authorization: Bearer admin" \
  -H "Content-Type: application/json" \
  -d '{"dashboard":{"id":13577}}'
```

---

### 第三阶段：ELK 集成 (5 天)

#### 3.1 Elasticsearch 配置

```yaml
# elasticsearch-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: elasticsearch
  replicas: 3
  
  template:
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.17.3
        
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
          limits:
            memory: "8Gi"
            cpu: "4"
        
        env:
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: cluster.name
          value: log-cluster
        - name: discovery.seed_hosts
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: cluster.initial_master_nodes
          value: "elasticsearch-0,elasticsearch-1,elasticsearch-2"
        - name: ES_JAVA_OPTS
          value: "-Xms4g -Xmx4g"
        
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: elasticsearch-data
```

#### 3.2 Logstash Pipeline 配置

```yaml
# logstash.conf

input {
  beats {
    port => 5044
  }
}

filter {
  # JSON 解析
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
      target => ""
      remove_field => ["message"]
    }
  }
  
  # 时间格式化
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
  
  # 添加元数据
  mutate {
    add_field => {
      "collector_host" => "%{host}"
      "collector_version" => "2.4.2"
    }
  }
  
  # 错误日志特殊处理
  if [level] == "ERROR" {
    mutate {
      add_tag => ["error_level"]
    }
  }
  
  # 错误码提取
  if [code] =~ /^[0-9]{3,}$/ {
    grok {
      match => { "code" => "%{NUMBER:error_code}" }
    }
  }
}

output {
  # 生产环境写入 Elasticsearch
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    document_type => "_doc"
    
    # 批量写入优化
    flush_size => 1000
    idle_flush_time => 10
  }
  
  # 控制台调试 (仅开发环境)
  stdout { codec => rubydebug }
}
```

---

## 📊 监控与告警

### 关键指标监控

```yaml
# 日志系统监控规则

groups:
- name: logging-system
  rules:
  # 采集器健康状态
  - alert: PromtailDown
    expr: up{job="promtail"} == 0
    for: 2m
    annotations:
      summary: "Logshipper 实例下线"
      
  # 日志发送延迟
  - alert: LokiWriteLatencyHigh
    expr: sum(rate(loki_write_requests_total[5m])) by (cluster) / sum(rate(loki_request_duration_seconds_count[5m])) by (cluster) > 100
    for: 5m
    annotations:
      summary: "日志写入延迟过高"
      
  # 存储空间不足
  - alert: LokiStorageLow
    expr: loki_tsdb_block_bytes / loki_tsdb_retention_period_bytes * 100 > 80
    for: 5m
    annotations:
      summary: "存储空间使用超过 80%"
      
  # ELK 集群健康度
  - alert: ElasticsearchClusterHealthRed
    expr: elasticsearch_cluster_health_status{status="red"} == 1
    for: 1m
    annotations:
      summary: "Elasticsearch 集群处于红色状态"
```

### Grafana Dashboard

```markdown
# 必备 Dashboard

1. **日志总览 Dashboard**
   ├── 今日日志量趋势
   ├── 各应用日志分布
   ├── 日志级别统计
   └── 错误日志 TOP10

2. **应用性能 Dashboard**
   ├── 应用日志 QPS
   ├── 错误率趋势
   ├── 响应时间分析
   └── 异常模式识别

3. **系统资源 Dashboard**
   ├── Loki 磁盘使用
   ├── Elasticsearch 节点状态
   ├── Logstash 队列积压
   └── 网络流量监控

4. **安全审计 Dashboard**
   ├── 登录失败次数
   ├── 异常访问模式
   ├── 敏感操作日志
   └── SQL 注入尝试
```

---

## 💰 成本对比

### 改造前后成本分析

```
┌─────────────────────────────────────────────────────────────┐
│                   日志系统成本对比                           │
│                                                             │
│  【改造前】单点 ELK 方案                                      │
│  ├── Elasticsearch 6 节点 × 16C64G: ¥80 万/年               │
│  ├── Logstash 4 节点 × 8C32G: ¥30 万/年                      │
│  ├── Kibana License: ¥5 万/年                               │
│  ├── 运维人力：1 人 × ¥15 万 = ¥15 万/年                       │
│  └── 总计：¥130 万/年                                       │
│                                                             │
│  【改造后】Hybrid 方案                                        │
│  ├── Loki 3 节点 × 8C32G: ¥30 万/年                           │
│  ├── MinIO S3 存储：¥5 万/年                                 │
│  ├── ELK 保留 3 节点做深度分析：¥40 万/年                      │
│  ├── Logstash 降级为 2 节点：¥15 万/年                        │
│  ├── 运维人力：0.5 人 × ¥15 万 = ¥7.5 万/年                     │
│  └── 总计：¥97.5 万/年                                     │
│                                                             │
│  【节省效果】                                                │
│  ├── 硬件成本：减少 25%                                    │
│  ├── 软件授权：减少 100% (改用开源版)                         │
│  ├── 运维人力：减少 50%                                    │
│  └── 总体节省：¥32.5 万/年 (25%)                            │
│                                                             │
│  【ROI 分析】                                                │
│  ├── 投入：¥32.5 万                                         │
│  ├── 每年节省：¥32.5 万                                    │
│  └── ROI: 100% (约 12 个月回本)                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 运维手册

### 日常巡检清单

```markdown
# 每日巡检 Checklist

- [ ] Loki 各组件健康状态
- [ ] Elasticsearch 集群健康度
- [ ] 存储空间使用情况 (< 80%)
- [ ] 日志发送延迟 (< 5 秒)
- [ ] 错误日志数量趋势
- [ ] 查询性能指标 (P99 < 3 秒)
- [ ] 告警规则触发情况
```

### 常见问题处理

```bash
#!/bin/bash
# troubleshooting.sh

# 问题 1: 日志丢失
echo "检查采集器状态..."
kubectl get pods -n logging | grep promtail
kubectl logs -n logging promtail-xxx -c main

echo "检查 Loki 接收队列..."
curl -s http://loki:3100/ready

echo "检查磁盘空间..."
df -h /var/lib/loki

# 问题 2: 查询缓慢
echo "检查 Loki 查询性能..."
curl -s http://loki:3100/query-stats

echo "检查索引碎片..."
kubectl exec -it loki-0 -- ls -lh /var/lib/loki/index

# 问题 3: Elasticsearch 慢
echo "查看节点状态..."
curl http://elasticsearch:9200/_cluster/health

echo "查看分片分布..."
curl http://elasticsearch:9200/_cat/shards?v
```

---

*企业级日志平台建设 | ClawsOps ⚙️*