# 日志系统 (ELK/Loki)

## 📋 概述

日志是运维可观测性的三大支柱之一，ELK (Elasticsearch/Logstash/Kibana) 和 Loki 是主流的日志收集分析方案。

## 🎯 学习目标

- 掌握日志收集架构设计
- 部署 ELK Stack
- 配置 Loki+Grafana
- 日志解析与分析
- 日志告警与审计

## 🏗️ 日志系统架构对比

### ELK Stack 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      ELK 日志系统                            │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ Filebeat│    │Logstash │    │Elasticsearch│            │
│  │ (采集)  │───►│ (处理)  │───►│  (存储)   │            │
│  └─────────┘    └─────────┘    └─────┬─────┘            │
│                                      │                    │
│                                      ▼                    │
│                                ┌─────────┐               │
│                                │ Kibana  │               │
│                                │ (展示)  │               │
│                                └─────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Loki 架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Loki 日志系统                           │
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │Promtail │    │  Loki   │    │Grafana  │                │
│  │ (采集)  │───►│ (聚合)  │───►│ (展示)  │                │
│  └─────────┘    └─────────┘    └─────────┘                │
│       │              │                                     │
│       │              ▼                                     │
│       │        ┌─────────┐                                │
│       └───────►│  labels │  (索引，非全文)                 │
│                └─────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

### 方案对比

| 特性 | ELK | Loki |
|------|-----|------|
| **存储方式** | 全文索引 | 仅索引 labels |
| **资源消耗** | 高 (内存/CPU) | 低 |
| **查询能力** | 强大 (Lucene) | 有限 (LogQL) |
| **成本** | 高 | 低 |
| **适用场景** | 复杂分析/审计 | 应用调试/监控 |

## 🔧 ELK Stack 部署

### 1. Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
    volumes:
      - es_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    ulimits:
      memlock:
        soft: -1
        hard: -1

  logstash:
    image: docker.elastic.co/logstash/logstash:7.17.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      - LS_JAVA_OPTS=-Xmx1g -Xms1g
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash

volumes:
  es_data:
```

### 2. Filebeat 配置

```yaml
# filebeat.yml
filebeat.inputs:
  # Docker 容器日志
  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/lib/docker/containers/"
  
  # 系统日志
  - type: log
    enabled: true
    paths:
      - /var/log/syslog
      - /var/log/auth.log
    fields:
      type: syslog
  
  # Nginx 日志
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
      - /var/log/nginx/error.log
    fields:
      type: nginx

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat.log
```

### 3. Logstash 管道配置

```ruby
# logstash/pipeline/logstash.conf
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # 解析 Nginx 日志
  if [fields][type] == "nginx" {
    grok {
      match => { 
        "message" => "%{COMBINEDAPACHELOG}"
      }
    }
    date {
      match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
    geoip {
      source => "clientip"
      target => "geoip"
    }
  }
  
  # 解析系统日志
  if [fields][type] == "syslog" {
    grok {
      match => { 
        "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}"
      }
    }
    date {
      match => [ "syslog_timestamp" , "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
  
  # 添加通用字段
  mutate {
    add_field => {
      "environment" => "production"
      "datacenter" => "cn-hangzhou"
    }
  }
  
  # 删除不需要的字段
  mutate {
    remove_field => ["host", "agent", "ecs"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[fields][type]}-%{+YYYY.MM.dd}"
  }
  
  # 调试输出
  stdout {
    codec => rubydebug
  }
}
```

### 4. Kibana 索引模式

```bash
# 创建索引模板
curl -X PUT "localhost:9200/_template/logs_template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.lifecycle.name": "logs-policy",
    "index.lifecycle.rollover_alias": "logs"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message": { "type": "text" },
      "clientip": { "type": "ip" },
      "response": { "type": "integer" },
      "bytes": { "type": "long" },
      "geoip": {
        "properties": {
          "location": { "type": "geo_point" }
        }
      }
    }
  }
}'
```

## 🌿 Loki 部署

### 1. Loki 配置

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_block_size: 262144
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/cache
    shared_store: filesystem
  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/compactor
  shared_store: filesystem

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  max_cache_freshness_per_query: 10m

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h
```

### 2. Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  loki:
    image: grafana/loki:2.7.0
    container_name: loki
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  promtail:
    image: grafana/promtail:2.7.0
    container_name: promtail
    volumes:
      - ./promtail-config.yaml:/etc/promtail/config.yml
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    command: -config.file=/etc/promtail/config.yml
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:9.3.0
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    ports:
      - "3000:3000"
    depends_on:
      - loki

volumes:
  loki_data:
  grafana_data:
```

### 3. Promtail 配置

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # 系统日志
  - job_name: syslog
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          __path__: /var/log/syslog

  # Docker 容器日志
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*.log
    
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs:
      - json:
          expressions:
            tag:
          source: attrs
      - regex:
          expression: (?P<container_name>(?:[^|]+))
          source: tag
      - labels:
          - container_name
          - stream

  # Nginx 日志
  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          __path__: /var/log/nginx/*.log
    
    pipeline_stages:
      - regex:
          expression: '^(?P<remote_addr>\S+) - (?P<remote_user>\S+) \[(?P<time_local>[^\]]+)\] "(?P<request>[^"]+)" (?P<status>\d+) (?P<body_bytes_sent>\d+) "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)"'
      - labels:
          - status
          - remote_addr
```

## 📊 日志查询与分析

### LogQL 查询语法

```
# 基础查询
{job="nginx"}

# 多标签匹配
{job="nginx", status="500"}

# 正则匹配
{job=~"app-.*", environment="production"}

# 日志行数统计
count_over_time({job="nginx"}[1h])

# 错误日志统计
sum(count_over_time({job="nginx", status=~"5.."}[5m])) by (status)

# 日志内容过滤
{job="app"} |= "ERROR"

# 正则过滤
{job="app"} |~ "timeout|connection refused"

# 排除匹配
{job="app"} !~ "DEBUG|INFO"

# 提取字段
{job="nginx"} | pattern `<ip> - - [<time>] "<method> <path> <protocol>" <status>`

# 聚合统计
sum by (status) (rate({job="nginx"}[5m]))
```

### Kibana 查询语法

```
# 精确匹配
status: 200

# 范围查询
bytes: [1000 TO 10000]

# 通配符
user_agent: *Chrome*

# 布尔查询
status: 200 AND method: GET

# 存在性查询
_exists_: geoip

# 嵌套查询
geoip.location: [40.7128 TO 40.7138]
```

## 🚨 日志告警

### 1. Watcher 告警 (Elasticsearch)

```json
PUT _watcher/watch/nginx-error-alert
{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-nginx-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                { "range": { "@timestamp": { "gte": "now-5m" } } },
                { "term": { "response": 500 } }
              ]
            }
          },
          "size": 0,
          "aggs": {
            "error_count": { "value_count": { "field": "_id" } }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.error_count.value": {
        "gt": 10
      }
    }
  },
  "actions": {
    "send_dingtalk": {
      "webhook": {
        "url": "https://oapi.dingtalk.com/robot/send?access_token=xxx",
        "body": "{\"msgtype\":\"text\",\"text\":{\"content\":\"Nginx 500 错误告警：5 分钟内{{ctx.payload.aggregations.error_count.value}}次\"}}"
      }
    }
  }
}
```

### 2. Loki 告警 (Grafana)

```yaml
# alert-rules.yml
groups:
  - name: loki-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="nginx", status=~"5.."}[5m])) 
          / sum(rate({job="nginx"}[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Nginx 高错误率"
          description: "5xx 错误率超过 5%"
      
      - alert: LogVolumeSpike
        expr: |
          sum(rate({job="app"}[5m])) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "日志量异常"
          description: "日志量突增，可能存在异常"
```

## 📈 日志保留策略

### ILM (Index Lifecycle Management)

```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

## 📚 参考资料

- [Elastic 官方文档](https://www.elastic.co/guide/)
- [Loki 官方文档](https://grafana.com/docs/loki/)
- [Logstash 配置指南](https://www.elastic.co/guide/en/logstash/current/index.html)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
