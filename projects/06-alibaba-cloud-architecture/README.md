# 企业实战项目六：阿里云企业级全栈架构

## 📋 项目背景

**客户:** 某跨境电商公司  
**业务场景:** 
- 中国用户 + 东南亚用户（新加坡）
- 日活 500 万，峰值 QPS 10000
- 要求等保三级、数据安全合规

**云厂商选型:** 阿里云（主）+ AWS（备）  
**云支出:** ¥200 万 / 年  
**团队:** 3 人运维 + 5 人研发

---

## 🏗️ 整体架构

### 跨区域高可用架构

```
┌─────────────────────────────────────────────────────────────┐
│                    全球流量调度层                              │
│                                                             │
│                  阿里云 GSLB (全球负载均衡)                    │
│                  ↓                                          │
│         ┌─────────────────┬──────────────────┐             │
│         ▼                 ▼                  ▼             │
│    ┌──────────┐     ┌──────────┐      ┌──────────┐        │
│    │ 杭州 Region│   │ 上海 Region│    │ 新加坡   │        │
│    │ (主)      │   │ (热备)    │     │ 亚太     │        │
│    └──────────┘     └──────────┘      └──────────┘        │
│                                                             │
│  GSLB 策略: 延迟最低 + 权重分配 + 故障自动切换                │
└─────────────────────────────────────────────────────────────┘
```

---

## ☁️ 阿里云核心服务选型

### 计算资源

```
┌─────────────────────────────────────────────────────────────┐
│                   阿里云计算服务矩阵                         │
│                                                             │
│  场景            │ 推荐服务        │ 选型理由                 │
│  ────────────────┼─────────────────┼────────────────────      │
│  K8s 容器编排    │ ACK (托管版)     │ 免运维 Master，支持 ARM  │
│  函数计算        │ FC              │ 事件驱动，冷启动 < 1s    │
│  弹性伸缩        │ ESS             │ 基于监控指标自动伸缩       │
│  裸金属          │ ECS Bare Metal  │ 数据库/高 I/O 场景       │
│  竞价实例        │ ECS Spot        │ 批处理/测试，省 90%      │
│  Serverless      │ SAE             │ 微服务免 IaaS           │
└─────────────────────────────────────────────────────────────┘
```

### 存储与数据库

```
┌─────────────────────────────────────────────────────────────┐
│                   阿里云数据服务选型                         │
│                                                             │
│  业务需求           │ 阿里云产品    │ 规格配置                  │
│  ──────────────────┼─────────────┼────────────────────       │
│  关系型数据库       │ RDS MySQL   │ 8C32G 高可用版             │
│  缓存              │ Redis 企业版 │ 主从 16G                  │
│  消息队列          │ RocketMQ    │ 企业版，1000 TPS           │
│  对象存储          │ OSS         │ 标准 + 低频 + 归档         │
│  文件存储          │ NAS         │ 容量型                     │
│  时序数据库        │ TSDB        │ 监控数据                   │
│  图数据库          │ GraphDB     │ 推荐/风控                   │
│  数据仓库          │ MaxCompute  │ 离线分析                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### 第一阶段：VPC 网络架构 (2 天)

#### 1.1 跨区域 VPC 互联

```yaml
# alibaba-cloud-network.yaml - 网络架构

# 杭州 VPC (主)
- name: vpc-hangzhou
  cidr: 10.0.0.0/16
  vpc_name: ecommerce-cn-hangzhou
  
  # 子网规划
  subnets:
  - name: subnet-master
    cidr: 10.0.0.0/24
    zone: cn-hangzhou-a
    usage: k8s-master
    
  - name: subnet-worker-a
    cidr: 10.0.1.0/24
    zone: cn-hangzhou-a
    usage: k8s-worker
    
  - name: subnet-worker-b
    cidr: 10.0.2.0/24
    zone: cn-hangzhou-b
    usage: k8s-worker
    
  - name: subnet-rds
    cidr: 10.0.3.0/24
    zone: cn-hangzhou-a
    usage: rds-redis
    
  - name: subnet-nat
    cidr: 10.0.4.0/24
    zone: cn-hangzhou-a
    usage: nat-gateway

# 上海 VPC (热备)
- name: vpc-shanghai
  cidr: 10.1.0.0/16
  vpc_name: ecommerce-cn-shanghai
  
  subnets:
  - name: subnet-master-sh
    cidr: 10.1.0.0/24
    zone: cn-shanghai-b
    usage: k8s-master
    
  - name: subnet-worker-sh
    cidr: 10.1.1.0/24
    zone: cn-shanghai-b
    usage: k8s-worker

# 新加坡 VPC (亚太)
- name: vpc-singapore
  cidr: 10.2.0.0/16
  vpc_name: ecommerce-ap-southeast-1
```

#### 1.2 CEN 云企业网互联

```bash
#!/bin/bash
# cen-setup.sh - 跨区域 VPC 互联

# 创建 CEN 实例
aliyun cen CreateCenInstance \
  --CenInstanceId "cen-ecommerce-global" \
  --CenInstanceName "跨境电商-全球互联" \
  --Description "杭州-上海-新加坡跨区域互联"

CEN_ID=$(aliyun cen DescribeCenInstances --CenInstanceName "跨境电商-全球互联" \
  | jq -r '.Cens.Cen[0].CenId')

# 加载杭州 VPC
aliyun cen AttachVPCToCen \
  --CenId "$CEN_ID" \
  --ChildInstanceId "vpc-hangzhou" \
  --ChildInstanceType VPC \
  --ChildInstanceRegionId cn-hangzhou

# 加载上海 VPC
aliyun cen AttachVPCToCen \
  --CenId "$CEN_ID" \
  --ChildInstanceId "vpc-shanghai" \
  --ChildInstanceType VPC \
  --ChildInstanceRegionId cn-shanghai

# 加载新加坡 VPC
aliyun cen AttachVPCToCen \
  --CenId "$CEN_ID" \
  --ChildInstanceId "vpc-singapore" \
  --ChildInstanceType VPC \
  --ChildInstanceRegionId ap-southeast-1

# 创建云连接网 (CCN) 用于带宽包
aliyun ccn CreateCloudConnectionNetwork \
  --CcnInstanceName "跨境电商-CCN" \
  --Description "跨区域带宽包"

CCN_ID=$(aliyun ccn DescribeCloudConnectionNetworks | jq -r '.CloudConnectionNetworks[0].CcnId')

# 购买跨区域带宽包
aliyun cen AttachCenChildInstanceToCenBandwidthPackage \
  --CenId "$CEN_ID" \
  --CcnId "$CCN_ID" \
  --BandwidthPackageId "bps-xxx"

echo "✅ CEN 互联配置完成"
```

---

### 第二阶段：ACK 集群部署 (3 天)

#### 2.1 杭州 ACK Pro 集群

```yaml
# ack-hangzhou.yaml

apiVersion: alibabacloud.com/v1
kind: ManagedKubernetesCluster
metadata:
  name: ack-ecommerce-hangzhou
  namespace: ack-system
spec:
  regionId: cn-hangzhou
  name: ecommerce-production
  
  # 集群类型
  clusterType: ManagedKubernetes
  kubernetesVersion: "1.28.3-aliyun.1"
  
  # 网络配置
  networking:
    mode: Terway  # 阿里云高性能 CNI
    podVswitchId: subnet-worker-a
    serviceVswitchId: subnet-master
    podCidr: 172.20.0.0/16
    serviceCidr: 192.168.0.0/16
    
  # Master 节点 (托管，免运维)
  masterCount: 3
  masterInstanceTypes:
  - ecs.c7.xlarge
  masterSystemDiskCategory: cloud_essd
  masterSystemDiskSize: 120
  
  # Worker 节点池
  workerNodePools:
  - name: app-pool
    vswitchIds: [subnet-worker-a, subnet-worker-b]
    instanceTypes: [ecs.c7.4xlarge]  # 16C64G
    count: 8
    systemDiskCategory: cloud_essd
    systemDiskSize: 200
    diskPerformanceLevel: PL1
    autoScaling:
      enabled: true
      min: 5
      max: 20
      scaleDownCooldown: 300
    labels:
      role: application
      environment: production
      
  - name: db-pool
    vswitchIds: [subnet-rds]
    instanceTypes: [ecs.r7.4xlarge]  # 16C128G
    count: 3
    systemDiskCategory: cloud_essd
    systemDiskSize: 500
    diskPerformanceLevel: PL1
    labels:
      role: database
      environment: production
    taints:
    - key: database
      value: "true"
      effect: NoSchedule
      
  - name: spot-pool
    vswitchIds: [subnet-worker-a]
    instanceTypes:
    - ecs.c7.2xlarge
    - ecs.c7.4xlarge
    count: 5
    spotStrategy: SpotAsPriceGo
    spotPriceLimit: 0.5
    labels:
      role: batch
      pricing: spot

  # 安全组
  securityGroupId: sg-hangzhou-ack
  extraSecurityGroups:
  - sg-rds-access
  
  # SSH 配置
  sshFlags: true
  keyPair: ack-ecommerce
```

#### 2.2 ACK 插件配置

```bash
#!/bin/bash
# ack-addons.sh

# 安装核心插件
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alibaba-cloud-controller-manager
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: cloud-controller-manager
        image: registry.cn-hangzhou.aliyuncs.com/acs/ccm:v1.9.3
        args:
        - --cloud-provider=alibabacloud
        - --configure-cloud-routes=true
        - --cluster-cidr=172.20.0.0/16
EOF

# 配置 CSI 插件 (云盘/ NAS /OSS)
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-essd
provisioner: diskplugin.csi.alibabacloud.com
parameters:
  type: cloud_essd
  performanceLevel: PL1
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
- discard
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-nas-standard
provisioner: nasplugin.csi.alibabacloud.com
parameters:
  volumeType: standard
  server: "nas-xxx.cn-hangzhou.nas.aliyuncs.com"
  path: "/ecommerce"
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-oss
provisioner: ossplugin.csi.alibabacloud.com
parameters:
  bucket: "ecommerce-data"
  url: "oss-cn-hangzhou-internal.aliyuncs.com"
  akId: "${OSS_AK_ID}"
  akSecret: "${OSS_AK_SECRET}"
EOF

# 配置 Ingress (ALB)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alb-ingress-controller
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: alb-ingress-controller
        image: registry.cn-hangzhou.aliyuncs.com/acs/alb-ingress-controller:v1.1.2
        args:
        - --ingress-class=alb
        - --cluster-name=ecommerce-production
        - --config-map=kube-system/alb-config
EOF
```

---

### 第三阶段：云原生中间件 (5 天)

#### 3.1 RDS MySQL 高可用

```bash
#!/bin/bash
# rds-setup.sh

# 创建 RDS MySQL 高可用版
aliyun rds CreateInstance \
  --Engine MySQL \
  --EngineVersion 8.0 \
  --InstanceType rds.mysql.c4.xlarge \
  --InstanceStorage 500 \
  --ZoneId cn-hangzhou-a \
  --VSwitchId subnet-rds \
  --VPCId vpc-hangzhou \
  --PayType Postpaid \
  --InstanceDescription ecommerce-mysql-prod \
  --Category HighAvailability  # 高可用版（主备）

RDS_ID=$(aliyun rds DescribeDBInstances | jq -r '.Items.DBInstance[0].DBInstanceId')

# 创建数据库和账号
aliyun rds CreateDatabase \
  --DBInstanceId "$RDS_ID" \
  --DBName ecommerce_orders \
  --CharacterSet utf8mb4

aliyun rds CreateAccount \
  --DBInstanceId "$RDS_ID" \
  --AccountName ecommerce_app \
  --AccountPassword "$(openssl rand -base64 16)" \
  --AccountType Normal

# 授权
aliyun rds GrantAccountPrivilege \
  --DBInstanceId "$RDS_ID" \
  --AccountName ecommerce_app \
  --DBName ecommerce_orders \
  --AccountPrivilege ReadWrite

# 配置白名单
aliyun rds ModifySecurityIps \
  --DBInstanceId "$RDS_ID" \
  --SecurityIps "10.0.0.0/16,10.1.0.0/16"

# 开启 SQL 审计
aliyun rds ModifySqlCollectorPolicy \
  --DBInstanceId "$RDS_ID" \
  --SqlCollectorStatus Enable \
  --RetentionDays 30

# 创建只读实例
aliyun rds CreateReadOnlyInstance \
  --MasterInstanceId "$RDS_ID" \
  --InstanceType rds.mysql.c4.xlarge \
  --InstanceStorage 500 \
  --PayType Postpaid

echo "✅ RDS MySQL 高可用配置完成"
```

#### 3.2 Redis 企业版

```bash
#!/bin/bash
# redis-setup.sh

# 创建 Redis 企业版（主从）
aliyun r-kvstore CreateInstance \
  --EngineVersion Redis5.0 \
  --ArchitectureStandard  # 标准版
  --InstanceClass redis.master.mid.default \
  --Capacity 16 \
  --ZoneId cn-hangzhou-a \
  --VSwitchId subnet-rds \
  --VPCId vpc-hangzhou \
  --PayType Postpaid \
  --InstanceName ecommerce-redis-prod

REDIS_ID=$(aliyun r-kvstore DescribeInstances | jq -r '.Instances.Instance[0].InstanceId')

# 创建账号
aliyun r-kvstore CreateAccount \
  --InstanceId "$REDIS_ID" \
  --AccountName ecommerce \
  --AccountPassword "$(openssl rand -base64 16)" \
  --AccountType Standard

# 设置白名单
aliyun r-kvstore ModifyAllowList \
  --InstanceId "$REDIS_ID" \
  --Whitelist "10.0.0.0/16,10.1.0.0/16"

# 开启持久化 (AOF)
aliyun r-kvstore ModifyInstanceAttribute \
  --InstanceId "$REDIS_ID" \
  --EnableAOF true \
  --AofFsync everysec

echo "✅ Redis 企业版配置完成"
```

#### 3.3 RocketMQ 企业版

```bash
#!/bin/bash
# rocketmq-setup.sh

aliyun rocketmq CreateInstance \
  --InstanceName ecommerce-mq-prod \
  --SpecType standard \
  --Message-type Topic \
  --TopicCount 50 \
  --TPS 1000 \
  --ZoneId cn-hangzhou-a \
  --VSwitchId subnet-rds \
  --VPCId vpc-hangzhou

MQ_ID=$(aliyun rocketmq DescribeInstances | jq -r '.Instances[0].InstanceId')

# 创建 Topic
aliyun rocketmq CreateTopic \
  --InstanceId "$MQ_ID" \
  --Topic "ORDER_CREATED" \
  --Remark "订单创建事件"

aliyun rocketmq CreateTopic \
  --InstanceId "$MQ_ID" \
  --Topic "PAYMENT_COMPLETED" \
  --Remark "支付完成事件"

aliyun rocketmq CreateTopic \
  --InstanceId "$MQ_ID" \
  --Topic "INVENTORY_UPDATED" \
  --Remark "库存更新事件"

echo "✅ RocketMQ 企业版配置完成"
```

---

### 第四阶段：Serverless 与弹性 (3 天)

#### 4.1 FC 函数计算

```yaml
# fc-functions.yaml

# 图片处理函数
apiVersion: fc.fc-deploy
metadata:
  name: image-processor
  runtime: python3.9
  
spec:
  handler: index.handler
  memorySize: 512
  timeout: 30
  cpu: 0.5
  
  environment:
    OSS_BUCKET: ecommerce-images
    OSS_ENDPOINT: oss-cn-hangzhou-internal.aliyuncs.com
    
  triggers:
  - name: oss-trigger
    type: oss
    config:
      bucket: ecommerce-raw
      events: oss:ObjectCreated:PutObject
      filter:
        key:
          prefix: uploads/
          suffix: .jpg
  
  code:
    src: ./functions/image-processor/

# 订单超时取消函数
apiVersion: fc.fc-deploy
metadata:
  name: order-timeout-check
  runtime: python3.9
  
spec:
  handler: index.handler
  memorySize: 256
  timeout: 60
  
  triggers:
  - name: timer-trigger
    type: timer
    config:
      payload: '{"check_interval": 300}'
      cron: "0 */5 * * * *"  # 每 5 分钟
  
  code:
    src: ./functions/order-timeout/
```

#### 4.2 ESS 弹性伸缩

```yaml
# ess-config.yaml

# 应用层自动伸缩
- name: app-scaling-group
  scaling_group_name: ecommerce-app-scaling
  min_size: 5
  max_size: 20
  default_cooldown: 300
  health_check_type: ECS
  health_check_grace_period: 300
  
  # 伸缩配置
  launch_template:
    image_id: ubuntu_22_04_x64_20G_alibase_20240101.vhd
    instance_type: ecs.c7.4xlarge
    security_group_id: sg-hangzhou-ack
    system_disk_category: cloud_essd
    system_disk_size: 200
    data_disks:
    - category: cloud_essd
      size: 100
      performance_level: PL1
    
  # 伸缩规则
  scaling_rules:
  - name: cpu-scale-up
    scaling_rule_type: TargetTrackingScalingRule
    target_value: 70  # CPU 使用率超过 70%
    metric_type: CPU
    cooldown: 300
    
  - name: qps-scale-up
    scaling_rule_type: TargetTrackingScalingRule
    target_value: 1000  # 每实例 QPS 超过 1000
    metric_type: QPS
    cooldown: 300
    
  - name: night-scale-down
    scaling_rule_type: TargetTrackingScalingRule
    target_value: 30
    metric_type: CPU
    cooldown: 600
    scale_in_cooldown: 600

  # 定时伸缩
  scheduled_tasks:
  - name: morning-peak
    task_type: ScheduledTask
    launch_time: "2024-01-01T08:00:00Z"
    recurrence_type: Daily
    recurrence_value: "08:00"
    desired_capacity: 15
    
  - name: midnight-low
    task_type: ScheduledTask
    launch_time: "2024-01-01T00:00:00Z"
    recurrence_type: Daily
    recurrence_value: "00:00"
    desired_capacity: 5
```

---

### 第五阶段：云原生监控与日志 (2 天)

#### 5.1 SLS 日志服务

```yaml
# sls-config.yaml

# 创建日志项目
- name: sls-ecommerce
  project: ecommerce-logs
  region: cn-hangzhou
  
  # Logstore 规划
  logstores:
  - name: application
    retention: 30
    shard_count: 10
    index:
      full_text_index: true
      keys:
      - name: level
        type: text
      - name: service
        type: text
      - name: trace_id
        type: text
      - name: error_message
        type: text
    
  - name: access
    retention: 15
    shard_count: 5
    index:
      keys:
      - name: status
        type: long
      - name: request_time
        type: double
      - name: remote_addr
        type: text
    
  - name: audit
    retention: 90
    shard_count: 3
    index:
      keys:
      - name: user
        type: text
      - name: action
        type: text
      - name: result
        type: text
  
  # 告警规则
  alerts:
  - name: high-error-rate
    query: "* | select count(*) as error_count where level='ERROR' group by service"
    threshold: 100
    period: 300
    notify:
      type: dingtalk
      url: "https://oapi.dingtalk.com/robot/send?access_token=xxx"
  
  - name: disk-space-warning
    query: "* | select host, disk_usage_percent from sls_metric where disk_usage_percent > 80"
    threshold: 1
    period: 600

  # 数据投递 (OSS 归档)
  deliveries:
  - name: oss-archive
    target_type: oss
    config:
      bucket: ecommerce-log-archive
      prefix: logs/{year}/{month}/{day}/
      format: json
      schedule_interval: 300  # 5 分钟
```

---

## 💰 成本分析

### 月度费用明细

```
┌─────────────────────────────────────────────────────────────┐
│                   阿里云月度费用明细                         │
│                                                             │
│  【计算资源】¥48,000/月                                       │
│  ├── ACK Pro 托管版：¥3,000/月                               │
│  ├── Worker 节点 (8 台 ecs.c7.4xlarge): ¥35,000/月           │
│  ├── 竞价实例 (5 台): ¥10,000/月                             │
│  └── FC 函数计算: ¥0 (按量，约 ¥1,000)                       │
│                                                             │
│  【数据库与中间件】¥28,000/月                                  │
│  ├── RDS MySQL 高可用 (8C32G): ¥15,000/月                    │
│  ├── Redis 企业版 (16G): ¥5,000/月                           │
│  ├── RocketMQ 企业版: ¥5,000/月                              │
│  └── DTS 数据传输: ¥3,000/月                                 │
│                                                             │
│  【存储】¥12,000/月                                          │
│  ├── OSS 标准存储 (5TB): ¥5,000/月                           │
│  ├── NAS 容量型 (2TB): ¥4,000/月                             │
│  ├── 云盘 ESSD (500G×15): ¥3,000/月                          │
│  └── 备份存储: ¥0 (按需)                                      │
│                                                             │
│  【网络】¥8,000/月                                           │
│  ├── CEN 跨区域带宽: ¥3,000/月                               │
│  ├── SLB (ALB): ¥2,000/月                                   │
│  ├── NAT 网关: ¥1,500/月                                     │
│  ├── EIP: ¥1,000/月                                         │
│  └── CDN 流量: ¥0 (按量，约 ¥500)                            │
│                                                             │
│  【安全与监控】¥4,000/月                                      │
│  ├── WAF: ¥2,000/月                                         │
│  ├── SLS 日志服务: ¥1,500/月                                 │
│  ├── 云安全中心: ¥500/月                                     │
│  └── CMS 监控: ¥0 (免费额度内)                                │
│                                                             │
│  月度总计：¥100,000                                          │
│  年度总计：¥1,200,000                                        │
│                                                             │
│  【成本优化建议】                                              │
│  ├── 预留实例 (RI) 购买：节省 40% (¥480,000/年)                │
│  ├── 竞价实例用于批处理：节省 70%                              │
│  ├── OSS 生命周期管理：归档低频，节省 30%                      │
│  └── 夜间缩容：节省 20%                                      │
│                                                             │
│  优化后年度：¥720,000 (节省 40%)                             │
└─────────────────────────────────────────────────────────────┘
```

---

*阿里云企业级全栈架构 | ClawsOps ⚙️*