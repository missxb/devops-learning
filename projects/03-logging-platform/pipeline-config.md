# 企业级日志平台 - 日志解析与告警配置

## Fluentbit / Promtail 解析配置

### Kubernetes Pod 日志解析

```yaml
# promtail-pipeline-config.yaml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
- url: http://loki.logging:3100/loki/api/v1/push

scrape_configs:
# 1. 应用 JSON 日志
- job_name: kubernetes-app-json
  pipeline_stages:
  - docker: {}
  
  - json:
      expressions:
        timestamp: time
        level: level
        message: msg
        trace_id: traceId
        span_id: spanId
        service: service
        user_id: userId
        request_id: requestId
      
  - timestamp:
      source: timestamp
      format: "2006-01-02T15:04:05.000Z"
  
  - labels:
      level:
      service:
      trace_id:
  
  - labeldrop:
    - filename
    - __path__
    - stream
  
  - output:
      source: message

  kubernetes_sd_configs:
  - role: pod
  
  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app
  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace
  - source_labels: [__meta_kubernetes_pod_name]
    target_label: pod

# 2. Nginx 访问日志
- job_name: nginx-access
  pipeline_stages:
  - regex:
      expression: '^(?P<remote_addr>[^ ]+) - (?P<remote_user>[^ ]+) \[(?P<time_local>[^\]]+)\] "(?P<request>[^"]*)" (?P<status>[\d]+) (?P<body_bytes_sent>[\d]+) "(?P<http_referer>[^"]*)" "(?P<http_user_agent>[^"]*)" (?P<request_time>[\d.]+) (?P<upstream_response_time>[^ ]+)$'
  
  - timestamp:
      source: time_local
      format: "02/Jan/2006:15:04:05 -0700"
  
  - labels:
      status:
      remote_addr:
  
  - metrics:
      request_duration_seconds:
        type: Histogram
        source: request_time
        config:
          buckets: [0.01, 0.05, 0.1, 0.5, 1, 5, 10]
          help: Request duration in seconds
  
  - output:
      source: message

  kubernetes_sd_configs:
  - role: pod
    selectors:
    - role: pod
      label: app.kubernetes.io/name=nginx-ingress

# 3. 错误日志特殊处理
- job_name: error-logs
  pipeline_stages:
  - match:
      selector: '{level="ERROR"}'
      stages:
      - json:
          expressions:
            error_message: error
            stack_trace: stack
            error_code: errorCode
      - labels:
          error_code:
      - output:
          source: error_message

# 4. 审计日志
- job_name: audit-logs
  pipeline_stages:
  - json:
      expressions:
        action: action
        resource: resource
        user: user
        ip: sourceIP
        result: result
  
  - labels:
      action:
      resource:
      result:
  
  - output:
      source: message

  kubernetes_sd_configs:
  - role: pod
    selectors:
    - role: pod
      label: type=audit
```

---

## 日志告警规则

### Loki LogQL 告警

```yaml
# loki-alert-rules.yaml

groups:
- name: application-errors
  rules:
  
  # 5xx 错误率突增
  - alert: HighHTTP5xxErrorRate
    expr: |
      sum(rate({namespace="production"} |= "500" | json | line_format "{{.status}}" [5m])) by (app)
      /
      sum(rate({namespace="production"}[5m])) by (app) * 100 > 5
    for: 3m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "应用 {{ $labels.app }} HTTP 5xx 错误率超过 5%"
      description: "当前错误率: {{ $value }}%"
      
  # 数据库连接超时
  - alert: DatabaseConnectionTimeout
    expr: |
      count_over_time({namespace="production"} |= "connection timeout" | json [5m]) > 10
    for: 2m
    labels:
      severity: critical
      team: database
    annotations:
      summary: "数据库连接超时次数过多"
      description: "5 分钟内出现 {{ $value }} 次连接超时"
      
  # OutOfMemory 错误
  - alert: OOMDetected
    expr: |
      count_over_time({namespace="production"} |= "OutOfMemoryError" [10m]) > 0
    for: 1m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "检测到 JVM OutOfMemory 错误"
      
  # 磁盘空间不足（日志写入失败）
  - alert: LogDiskSpaceLow
    expr: |
      count_over_time({app="promtail"} |= "disk full" [5m]) > 0
    for: 1m
    labels:
      severity: warning
      team: sre
    annotations:
      summary: "日志写入磁盘空间不足"
      
  # 安全审计 - 登录失败
  - alert: MultipleLoginFailures
    expr: |
      sum(rate({app="auth-service"} |= "login failed" | json [5m])) by (user) > 5
    for: 5m
    labels:
      severity: warning
      team: security
    annotations:
      summary: "用户 {{ $labels.user }} 登录失败次数过多"
      description: "5 分钟内失败 {{ $value }} 次"

- name: infrastructure
  rules:
  
  # Loki 写入延迟
  - alert: LokiWriteLatencyHigh
    expr: histogram_quantile(0.99, sum(rate(loki_request_duration_seconds_bucket{route="/loki/api/v1/push"}[5m])) by (le)) > 2
    for: 5m
    labels:
      severity: warning
      team: sre
    annotations:
      summary: "Loki 写入延迟 P99 超过 2 秒"
      
  # Promtail 连接失败
  - alert: PromtailConnectionFailed
    expr: |
      count_over_time({app="promtail"} |= "failed to send" [5m]) > 5
    for: 3m
    labels:
      severity: warning
      team: sre
    annotations:
      summary: "Promtail 日志发送失败"
```

---

## ELK 深度分析 Pipeline

### Logstash 高级解析

```ruby
# logstash-pipeline.conf

input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/pki/tls/certs/logstash.crt"
    ssl_key => "/etc/pki/tls/private/logstash.key"
  }
  
  tcp {
    port => 5045
    codec => json
  }
}

filter {
  # 1. JSON 解析
  if [message] =~ /^\{.*\}$/ {
    json {
      source => "message"
      target => ""
    }
  }
  
  # 2. 时间格式化
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
  }
  
  # 3. 应用日志解析
  if [app] {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:log_time} \[%{DATA:thread}\] %{LOGLEVEL:level} %{JAVACLASS:class} - %{GREEDYDATA:log_message}" }
    }
  }
  
  # 4. Nginx 日志解析
  if [nginx] {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    geoip {
      source => "remote_addr"
      target => "geoip"
      database => "/usr/share/logstash/geoip/GeoLite2-City.mmdb"
    }
    useragent {
      source => "http_user_agent"
      target => "user_agent"
    }
  }
  
  # 5. 错误级别分类
  if [level] == "ERROR" or [level] == "FATAL" {
    mutate {
      add_tag => ["error", "needs_attention"]
    }
  }
  
  # 6. 敏感信息脱敏
  ruby {
    code => '
      event.set("message", event.get("message").gsub(/(password|token|secret)[=:]\s*\S+/i, "\\1=***"))
    '
  }
  
  # 7. 添加环境信息
  mutate {
    add_field => {
      "collector" => "logstash"
      "environment" => "production"
      "datacenter" => "hangzhou-a"
    }
  }
  
  # 8. 删除冗余字段
  mutate {
    remove_field => ["ecs", "host.hostname", "agent.type", "agent.version", "input.type"]
  }
}

output {
  # 生产环境写入 Elasticsearch
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logs-%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ELASTIC_PASSWORD}"
    
    # 批量优化
    flush_size => 2000
    idle_flush_time => 5
    retry_on_initial_error => 3
    
    # 模板管理
    manage_template => true
    template_name => "logs"
    template_overwrite => true
  }
  
  # 错误日志单独输出
  if "error" in [tags] {
    elasticsearch {
      hosts => ["http://elasticsearch:9200"]
      index => "error-logs-%{+YYYY.MM.dd}"
      user => "elastic"
      password => "${ELASTIC_PASSWORD}"
    }
    
    # 同时发送到告警队列
    http {
      url => "http://alertmanager:9093/api/v1/alerts"
      http_method => "post"
      format => "json"
    }
  }
  
  # 调试输出（开发环境）
  if [environment] == "staging" {
    stdout {
      codec => rubydebug {
        metadata => true
      }
    }
  }
}
```

---

## Grafana 日志 Dashboard

### 关键查询语句

```logql
# 错误率趋势 (5 分钟窗口)
sum(rate({namespace="production"} |= "error" [5m])) by (app)
/
sum(rate({namespace="production"}[5m])) by (app) * 100

# P95 响应时间
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace="production"}[5m])) by (le))

# 日志量 TOP10 应用
topk(10, sum by (app) (rate({namespace="production"}[5m])))

# 异常请求模式检测
count_over_time({namespace="production", level="ERROR"} |~ "timeout|connection refused|out of memory" [1h])

# 用户行为追踪（按 trace_id）
{namespace="production", trace_id="$trace_id"} | line_format "{{.timestamp}} {{.level}} {{.message}}"

# 安全事件检测
sum by (user) (rate({app="auth-service"} |= "login failed" [5m])) > 5

# 业务指标提取
sum(rate({namespace="production"} |= "order_created" | json | unwrap amount [5m]))

# 容量趋势
sum by (pod) (rate({namespace="production"}[1h])) 
```

---

## 日志归档与生命周期

```yaml
# log-retention-policy.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: log-lifecycle
  namespace: logging
data:
  retention-policy.json: |
    {
      "indices": [
        {
          "pattern": "logs-*",
          "policy": {
            "phases": {
              "hot": {
                "actions": {
                  "rollover": {
                    "max_size": "50gb",
                    "max_age": "1d",
                    "max_docs": 10000000
                  },
                  "set_priority": { "priority": 100 }
                }
              },
              "warm": {
                "min_age": "7d",
                "actions": {
                  "shrink": { "number_of_shards": 1 },
                  "forcemerge": { "max_num_segments": 1 },
                  "set_priority": { "priority": 50 }
                }
              },
              "cold": {
                "min_age": "30d",
                "actions": {
                  "set_priority": { "priority": 0 },
                  "searchable_snapshot": {
                    "snapshot_repository": "log-archive"
                  }
                }
              },
              "delete": {
                "min_age": "90d",
                "actions": {
                  "delete": {}
                }
              }
            }
          }
        },
        {
          "pattern": "error-logs-*",
          "policy": {
            "phases": {
              "hot": {
                "actions": {
                  "rollover": { "max_size": "10gb", "max_age": "1d" }
                }
              },
              "delete": {
                "min_age": "180d",
                "actions": { "delete": {} }
              }
            }
          }
        }
      ]
    }
```

---

*企业级日志平台 - 解析与告警配置 | ClawsOps ⚙️*