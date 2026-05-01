# 容量规划与成本优化

## 📋 概述

容量规划和成本优化是运维的进阶能力。**别等资源满了再扩容，也别让资源空着浪费钱**。

---

## 📐 容量规划方法论

### 容量规划流程

```
┌─────────────────────────────────────────────────────────────┐
│                   容量规划六步法                             │
│                                                             │
│  Step 1: 基线测量                                           │
│  ├── 当前资源使用统计                                       │
│  ├── 业务指标基线                                           │
│  └── 性能基准测试                                           │
│                                                             │
│  Step 2: 增长预测                                           │
│  ├── 业务增长趋势                                           │
│  ├── 用户增长预测                                           │
│  └── 峰值估算                                               │
│                                                             │
│  Step 3: 资源估算                                           │
│  ├── CPU 需求                                               │
│  ├── 内存需求                                               │
│  ├── 存储需求                                               │
│  └── 网络需求                                               │
│                                                             │
│  Step 4: 预留缓冲                                           │
│  ├── 安全余量 30%                                           │
│  ├── 峰值应对                                               │
│  └── 灾备预留                                               │
│                                                             │
│  Step 5: 成本评估                                           │
│  ├── 资源成本                                               │
│  ├── 运维成本                                               │
│  └── 机会成本                                               │
│                                                             │
│  Step 6: 执行监控                                           │
│  ├── 扩容时机                                               │
│  ├── 监控验证                                               │
│  └── 动态调整                                               │
└─────────────────────────────────────────────────────────────┘
```

### 容量指标体系

```
┌─────────────────────────────────────────────────────────────┐
│                    容量指标矩阵                              │
│                                                             │
│  资源类型 │ 当前使用 │ 峰值 │ 趋势 │ 预警线 │ 扩容阈值      │
│  ─────────┼──────────┼──────┼──────┼───────┼──────────      │
│  CPU      │ 60%      │ 85%  │ +5%/月│ 80%   │ 90%           │
│  内存     │ 70%      │ 90%  │ +8%/月│ 85%   │ 95%           │
│  磁盘     │ 50TB     │ 80TB │ +2TB/月│ 80%   │ 90%           │
│  网络     │ 5Gbps    │ 8Gbps│ +10%/月│ 80%   │ 90%           │
│  QPS      │ 5000     │ 8000 │ +15%/月│ 80%   │ 90%           │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 容量计算公式

### 基础公式

```
CPU 需求:
  核数 = 当前核数 × (目标QPS / 当前QPS) × (1 + 缓冲率)
  
示例:
  当前: 32核，QPS 5000
  目标: QPS 8000，缓冲 30%
  需要: 32 × (8000/5000) × 1.3 = 32 × 1.6 × 1.3 = 66 核

内存需求:
  GB = 当前内存 × (目标连接数 / 当前连接数) × (1 + 缓冲率)
  
示例:
  当前: 64GB，连接数 10000
  目标: 连接数 20000，缓冲 30%
  需要: 64 × (20000/10000) × 1.3 = 64 × 2 × 1.3 = 166 GB

存储需求:
  TB = 当前存储 + (月增量 × 月数) × (1 + 缓冲率)
  
示例:
  当前: 100TB，月增 5TB
  预测12个月，缓冲 30%
  需要: 100 + (5 × 12) × 1.3 = 100 + 60 × 1.3 = 178 TB
```

---

## 💰 成本优化策略

### 云资源成本优化

```
┌─────────────────────────────────────────────────────────────┐
│                    云成本优化矩阵                            │
│                                                             │
│  优化项           │ 方法                │ 预期节省           │
│  ─────────────────┼────────────────────┼───────────────     │
│  闲置实例         │ 定期扫描+自动关停   │ 20-30%            │
│  超配实例         │ 右配-sizing        │ 15-25%            │
│  长期资源         │ 预留实例/包年包月   │ 30-50%            │
│  临时资源         │ Spot/竞价实例      │ 50-70%            │
│  存储优化         │ 冷热分离+压缩      │ 20-40%            │
│  网络优化         │ 专线+共享带宽      │ 10-30%            │
└─────────────────────────────────────────────────────────────┘
```

### 成本监控脚本

```bash
#!/bin/bash
# cloud-cost-monitor.sh - 云成本监控

echo "=== 云成本监控报告 ==="
echo "日期：$(date)"
echo ""

# 阿里云成本查询 (使用阿里云 CLI)
# 查看本月消费
aliyun bss OpenAPI QueryBill --OwnerUid 123456 --BillingCycle 2026-05

# AWS 成本查询
aws ce get-cost-and-usage \
    --time-period Start=2026-05-01,End=2026-05-31 \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE

# 资源利用率分析
echo ""
echo "=== 低利用率资源 ==="

# CPU 利用率 < 10% 的实例
aliyun ecs DescribeInstances \
    --Status Running \
    | jq '.Instances.Instance[] | select(.CPU < 10)'

# 闲置弹性 IP
aliyun ecs DescribeEipAddresses \
    --Status Available

# 闲置云盘
aliyun ecs DescribeDisks \
    --Status Available \
    --Type data

echo ""
echo "=== 优化建议 ==="
echo "1. 低利用率实例建议降配或关停"
echo "2. 闲置资源建议释放"
echo "3. 长期资源建议购买预留实例"
```

---

## 📈 自动扩缩容

### Kubernetes HPA 配置

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # CPU 指标
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # 内存指标
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # 自定义指标 (QPS)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 1000
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### 预测性扩容

```yaml
# 使用 Prometheus 指标预测
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: predictive-hpa
  annotations:
    predictive-autoscaling: enabled
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
  # 基于 5 分钟前的 QPS 预测
  - type: External
    external:
      metric:
        name: http_requests_predicted
        selector:
          matchLabels:
            prediction_window: 5m
      target:
        type: AverageValue
        averageValue: 800
```

---

## 📋 容量规划模板

```markdown
# [系统名称] 容量规划报告

## 当前基线

| 资源 | 当前配置 | 使用率 | 峰值 |
|------|----------|--------|------|
| CPU | 32核 | 60% | 85% |
| 内存 | 64GB | 70% | 90% |
| 存储 | 100TB | 50% | 80TB |
| QPS | - | 5000 | 8000 |

## 业务预测

- 用户增长：+20%/季度
- 业务增长：+15%/季度
- 峰值预测：+30% (节假日)

## 容量估算

| 时间 | CPU | 内存 | 存储 | 预算 |
|------|-----|------|------|------|
| 3个月 | 48核 | 96GB | 130TB | +¥10万 |
| 6个月 | 64核 | 128GB | 160TB | +¥25万 |
| 12个月 | 96核 | 256GB | 200TB | +¥50万 |

## 执行计划

1. 立即：低利用率实例优化
2. 3个月：存储扩容
3. 6个月：计算资源扩容
4. 12个月：架构优化 (分库分表)
```

---

## 💡 成本优化实践

### 我踩过的坑

```
坑 1: 按量付费没有监控
后果: 月底账单爆炸，老板找谈话
改进: 每日成本监控 + 异常告警

坑 2: 测试环境忘记关
后果: 测试资源跑了一个月
改进: 测试环境自动定时关停

坑 3: 超配大实例
后果: CPU利用率 5%，浪费 95%
改进: 定期右配 sizing

坑 4: 没用预留实例
后果: 按量付费贵 50%
改进: 长期资源购买包年包月
```

---

*容量规划与成本优化 | 运维进阶能力*