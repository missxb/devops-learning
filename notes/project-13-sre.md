# 项目十三：SRE 运维体系

## 📋 项目概述

SRE（Site Reliability Engineering）是 Google 提出的可靠性工程实践，将软件工程方法应用于运维问题。本项目学习 SRE 核心理念、实施方法和工具链。

## 🎯 学习目标

- 理解 SRE 核心原则
- 掌握 SLI/SLO/SLA 设计
- 实施错误预算管理
- 建立应急响应机制
- 构建可观测性体系

## 🏗️ SRE 核心框架

```
┌─────────────────────────────────────────────────────────────┐
│                      SRE 运维体系                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  SLI/SLO    │  │  错误预算    │  │  应急响应    │        │
│  │  指标定义   │  │  管理策略   │  │  流程机制   │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │
│         └────────────────┼────────────────┘                │
│                          ▼                                 │
│  ┌───────────────────────────────────────────────────┐    │
│  │              可观测性平台                          │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │    │
│  │  │ Metrics │  │  Logs   │  │ Tracing │          │    │
│  │  │(Prometheus)│ │ (ELK)  │  │(Jaeger) │          │    │
│  │  └─────────┘  └─────────┘  └─────────┘          │    │
│  └───────────────────────────────────────────────────┘    │
│                          │                                 │
│         ┌────────────────┼────────────────┐               │
│         ▼                ▼                ▼               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  变更管理    │  │  容量规划    │  │  自动化运维   │       │
│  │  (Change)   │  │ (Capacity)  │  │  (Automation)│       │
│  └─────────────┘  └─────────────┘  └─────────────┘       │
└───────────────────────────────────────────────────────────┘
```

## 📊 SLI/SLO/SLA 设计

### 1. 指标定义

| 术语 | 说明 | 示例 |
|------|------|------|
| **SLI** | 服务等级指标 | 请求延迟、成功率、吞吐量 |
| **SLO** | 服务等级目标 | 99.9% 请求在 200ms 内完成 |
| **SLA** | 服务等级协议 | 未达 SLO 的赔偿条款 |

### 2. SLI 指标设计

```yaml
# Web 服务 SLI 示例
availability:
  name: "请求可用性"
  measurement: "成功请求数 / 总请求数"
  target: "99.9%"

latency:
  name: "请求延迟"
  measurement: "P99 延迟"
  target: "< 200ms"

throughput:
  name: "系统吞吐量"
  measurement: "每秒处理请求数"
  target: "> 10000 QPS"

correctness:
  name: "数据正确性"
  measurement: "数据一致性检查通过率"
  target: "100%"
```

### 3. SLO 配置示例

```yaml
# SLO 配置文件
apiVersion: slo.example.com/v1
kind: ServiceLevelObjective
metadata:
  name: api-availability
  namespace: production
spec:
  service: api-gateway
  target: 99.9
  window: 30d
  indicator:
    name: availability
    query: |
      sum(rate(http_requests_total{status=~"2.."}[5m])) 
      / sum(rate(http_requests_total[5m]))
  
  alerting:
    burn_rate:
      - threshold: 14.4  # 严重
        window: 5m
      - threshold: 6.0   # 高
        window: 30m
      - threshold: 2.0   # 中
        window: 2h
      - threshold: 1.0   # 低
        window: 6h
```

## 💰 错误预算管理

### 1. 错误预算计算

```
月度可用性 SLO: 99.9%
月度错误预算 = 100% - 99.9% = 0.1%

月度总分钟数 = 30 天 × 24 小时 × 60 分钟 = 43200 分钟
允许宕机时间 = 43200 × 0.1% = 43.2 分钟

如果当月已宕机 30 分钟
剩余错误预算 = 43.2 - 30 = 13.2 分钟
```

### 2. 错误预算策略

```yaml
# 错误预算消耗策略
error_budget_policy:
  # 预算消耗 > 50%
  - threshold: 50%
    actions:
      - "发送警告通知"
      - "冻结非紧急变更"
  
  # 预算消耗 > 80%
  - threshold: 80%
    actions:
      - "升级通知管理层"
      - "冻结所有变更"
      - "启动可靠性改进计划"
  
  # 预算耗尽
  - threshold: 100%
    actions:
      - "暂停所有新功能发布"
      - "全员投入可靠性改进"
      - "事后复盘会议"
```

### 3. 燃烧率监控

```promql
# 计算燃烧率
# 实际消耗速率 / 预期消耗速率

# 7 天燃烧率
(1 - (sum(rate(http_requests_total{status=~"2.."}[7d])) 
      / sum(rate(http_requests_total[7d])))) 
/ (1 - 0.999)

# 燃烧率告警
# 燃烧率 > 14.4 (严重)
burn_rate_critical = 14.4
# 燃烧率 > 6 (高)
burn_rate_high = 6.0
```

## 🚨 应急响应机制

### 1. 事件分级

| 级别 | 说明 | 响应时间 | 示例 |
|------|------|----------|------|
| **P0** | 核心功能完全不可用 | 5 分钟 | 全站宕机 |
| **P1** | 核心功能严重受损 | 15 分钟 | 支付失败 |
| **P2** | 非核心功能不可用 | 1 小时 | 部分 API 超时 |
| **P3** | 轻微问题 | 4 小时 | 偶发错误 |

### 2. On-Call 轮值

```yaml
# On-Call 轮值表
oncall_schedule:
  primary:
    - week: "2026-W18"
      engineer: "张三"
      backup: "李四"
    - week: "2026-W19"
      engineer: "李四"
      backup: "王五"
  
  escalation_policy:
    - level: 1
      contact: "oncall-primary"
      timeout: 15m
    - level: 2
      contact: "oncall-backup"
      timeout: 10m
    - level: 3
      contact: "team-lead"
      timeout: 5m
    - level: 4
      contact: "engineering-director"
```

### 3. 事件响应流程

```
事件检测 ──► 告警触发 ──► On-Call 响应
                              │
                              ▼
                        初步评估 (P0-P3)
                              │
                              ▼
                        启动应急流程
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        问题诊断          临时修复        用户沟通
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                        验证修复
                              │
                              ▼
                        事件关闭
                              │
                              ▼
                        事后复盘
```

### 4. 事件报告模板

```markdown
# 事件报告

## 基本信息
- 事件 ID: INC-2026-0501-001
- 发生时间：2026-05-01 14:30 UTC
- 恢复时间：2026-05-01 15:45 UTC
- 持续时间：75 分钟
- 级别：P1

## 影响范围
- 受影响服务：API Gateway, User Service
- 影响用户数：约 50,000
- 错误率峰值：85%

## 时间线
- 14:30 - 监控系统检测到错误率上升
- 14:35 - On-Call 工程师响应
- 14:45 - 确认为数据库连接池耗尽
- 15:00 - 扩容数据库连接池
- 15:30 - 错误率恢复正常
- 15:45 - 事件关闭

## 根本原因
数据库连接池配置过小，无法应对突发流量

## 改进措施
- [ ] 增加连接池大小 (负责人：张三，截止：2026-05-05)
- [ ] 添加连接池监控告警 (负责人：李四，截止：2026-05-03)
- [ ] 优化慢查询 (负责人：王五，截止：2026-05-10)
```

## 🔍 可观测性体系

### 1. 三大支柱

```
┌─────────────────────────────────────────────────┐
│              可观测性 (Observability)            │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────┐ │
│  │   Metrics   │  │    Logs     │  │ Tracing │ │
│  │   指标      │  │    日志     │  │  追踪   │ │
│  │             │  │             │  │         │ │
│  │ - 聚合数据  │  │ - 详细记录  │  │ - 链路  │ │
│  │ - 时间序列  │  │ - 结构化    │  │ - 延迟  │ │
│  │ - 告警      │  │ - 搜索      │  │ - 依赖  │ │
│  └─────────────┘  └─────────────┘  └─────────┘ │
│        │                │               │       │
│        └────────────────┼───────────────┘       │
│                         ▼                        │
│              ┌──────────────────┐               │
│              │   Grafana Dash   │               │
│              │   统一可视化      │               │
│              └──────────────────┘               │
└─────────────────────────────────────────────────┘
```

### 2. Prometheus 监控配置

```yaml
# prometheus.yml 关键配置
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "slo-alerts.yml"
  - "burn-rate-alerts.yml"

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 3. 日志收集 (ELK Stack)

```yaml
# Filebeat 配置
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/*.log
  processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
          - logs_path:
              logs_path: "/var/log/containers/"

output.elasticsearch:
  hosts: ['http://elasticsearch:9200']
  indices:
    - index: "logs-%{[agent.version]}-%{+yyyy.MM.dd}"
```

### 4. 链路追踪 (Jaeger)

```yaml
# Jaeger 部署配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.26
        ports:
        - containerPort: 5775
          protocol: UDP
        - containerPort: 6831
          protocol: UDP
        - containerPort: 6832
          protocol: UDP
        - containerPort: 5778
          protocol: TCP
        - containerPort: 16686
          protocol: TCP
        - containerPort: 14268
          protocol: TCP
        environment:
        - name: COLLECTOR_ZIPKIN_HOST_PORT
          value: ":9411"
```

## 🤖 自动化运维

### 1. 自动化原则

```
1. 消除重复性工作 (Toil Reduction)
   - 目标：运维工作 < 50%
   
2. 自动化优先
   - 能自动化的绝不手动
   
3. 渐进式自动化
   - 从简单到复杂
   - 从安全到关键
```

### 2. 自动化场景

```yaml
# 自动化运维场景
automation_scenarios:
  # 自动扩缩容
  auto_scaling:
    trigger: "CPU > 80% 持续 5 分钟"
    action: "增加 2 个实例"
    cooldown: "10 分钟"
  
  # 自动故障恢复
  auto_recovery:
    trigger: "健康检查失败"
    action: "重启容器"
    max_retries: 3
  
  # 自动证书续期
  cert_renewal:
    trigger: "证书过期前 30 天"
    action: "自动续期"
    notification: "邮件通知"
  
  # 自动备份
  auto_backup:
    schedule: "每天 02:00"
    action: "全量备份"
    retention: "30 天"
```

## 📈 SRE 成熟度模型

```
Level 1: 被动响应
  - 事件驱动
  - 手动操作
  - 无明确指标

Level 2: 基础实践
  - 有监控告警
  - 基础自动化
  - 开始定义 SLI

Level 3: 规范运营
  - SLO 管理
  - 错误预算
  - On-Call 轮值

Level 4: 持续优化
  - 数据驱动决策
  - 高度自动化
  - 主动预防

Level 5: 引领创新
  - 业界最佳实践
  - 工具输出
  - 文化建设
```

## 📚 参考资料

- 《Google SRE 运维解密》
- 《SRE Workbook》
- [Google SRE 官方文档](https://sre.google/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
