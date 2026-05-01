# 企业实战项目二：金融系统灾备容灾架构

## 📋 项目背景

**客户:** 某互联网金融公司  
**业务规模:** 日均交易 50 万笔，峰值 QPS 2000  
**合规要求:** 等保三级、金融监管要求  
**容灾等级:** RTO < 5 分钟, RPO ≈ 0 (接近零丢失)  
**预算:** ¥150 万 / 年  

---

## 💡 灾备标准定义

### RTO/RPO 指标

```
┌─────────────────────────────────────────────────────────────┐
│                   灾备关键指标                               │
│                                                             │
│  RTO (Recovery Time Objective)                              │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                            │
│  恢复时间目标：允许的最大停机时间                            │
│  ───────                                                     │
│  ├── 金融系统：< 5 分钟                                     │
│  ├── 核心服务：< 15 分钟                                    │
│  └── 一般服务：< 1 小时                                      │
│                                                             │
│  RPO (Recovery Point Objective)                             │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                            │
│  数据恢复点目标：允许的最大数据丢失量                        │
│  ───────                                                     │
│  ├── 金融核心交易：≈ 0 (实时同步)                          │
│  ├── 用户数据：< 1 秒                                       │
│  └── 日志数据：< 5 分钟                                      │
└─────────────────────────────────────────────────────────────┘
```

### 金融系统容灾标准

```
一级容灾 (同城双活):
├── 两个数据中心距离 < 50km
├── RTO < 5 分钟
├── RPO ≈ 0
└── 成本：¥80-100 万/年

二级容灾 (异地备份):
├── 两个数据中心距离 > 300km
├── RTO < 30 分钟
├── RPO < 5 分钟
└── 成本：¥30-50 万/年

三级容灾 (数据备份):
├── 冷备模式
├── RTO < 2 小时
├── RPO < 24 小时
└── 成本：¥10-20 万/年
```

---

## 🏗️ 灾备架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      主数据中心 (A 区)                        │
│                     杭州西湖区数据中心                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Kubernetes 集群                      │   │
│  │  master: 3 nodes | worker: 15 nodes                │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│          ┌──────────────┼──────────────┐                   │
│          ▼              ▼              ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │ MySQL A     │ │ Redis A     │ │ Kafka A     │         │
│  │ Master      │ │ Cluster     │ │ Cluster     │         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│          │              │              │                   │
│          ▼              ▼              ▼                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              流量分发层 (Ingress)                      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
           │                    │
           │                   专线
           │                    │
┌──────────▼────────────────────▼─────────────────────────────┐
│                      从数据中心 (B 区)                        │
│                     杭州余杭数据中心                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  Kubernetes 集群                      │   │
│  │  master: 3 nodes | worker: 10 nodes                │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│          ┌──────────────┼──────────────┐                   │
│          ▼              ▼              ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐         │
│  │ MySQL B     │ │ Redis B     │ │ Kafka B     │         │
│  │ Slave       │ │ Cold Standby│ │ Cold Standby│         │
│  └─────────────┘ └─────────────┘ └─────────────┘         │
│                                                             │
│  备注：B 区平时处理部分非核心业务流量                         │
└─────────────────────────────────────────────────────────────┘
```

### 数据同步架构

```
┌─────────────────────────────────────────────────────────────┐
│                   数据同步方案                                │
│                                                             │
│  【MySQL】主从复制 + MHA + Orchestrator                     │
│  ├── 半同步复制 (semi-sync)                                 │
│  ├── Binlog 延迟 < 1 秒                                     │
│  └── 自动故障切换 < 30 秒                                   │
│                                                             │
│  【Redis】哨兵模式 + Sentinel Cluster                        │
│  ├── 读写分离                                              │
│  ├── AOF 实时持久化                                        │
│  └── 跨机房复制                                            │
│                                                             │
│  【Kafka】MirrorMaker 2                                    │
│  ├── 全量同步 + 增量同步                                    │
│  ├── MirrorMaker 2 实时流式复制                              │
│  └── 消息顺序保证                                          │
│                                                             │
│  【文件存储】Rsync+lsyncd                                  │
│  ├── 定时同步（每分钟）                                    │
│  └── 实时增量同步                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 灾备实施方案

### 第一阶段：基础搭建 (2 周)

#### 1.1 网络连通性

```bash
# 建立专线连接
# A 区：10.0.0.0/24
# B 区：10.1.0.0/24
# 专线带宽：1Gbps

# 配置 DNS 解析
cat >> /etc/dns.conf << EOF
zone "cluster.internal" {
    primary 10.0.0.10;
    secondary 10.1.0.10;
}
EOF

# 验证专线连通性
ping -c 4 10.1.0.10
traceroute 10.1.0.10
```

#### 1.2 数据库主从配置

```sql
-- A 区 MySQL Master 配置
my.cnf 中配置：
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

replica-source-host = 10.1.0.50
replica-source-user = 'repl_user'
replica-source-password = 'xxx'
```

```sql
-- B 区 MySQL Slave 配置
show slave status\G

-- 关键状态检查
----------------
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
Master_Relay_Log_Pos: 234567890
Relay_Log_Space: 234567890
```

#### 1.3 Kafka 镜像复制

```yaml
# MirrorMaker 2 配置
config.json

{
  "connector.class": "io.confluent.connect.mirror.MirrorHeartbeatConnector",
  "clusters": "prod_a,prod_b",
  "admin.bootstrap.servers": "kafka-a.example.com:9092,kafka-b.example.com:9092",
  "topic.rename.regex": "(.*)",
  "topic.rename.replacement": "$1",
  "refresh.topics.interval.seconds": 60,
  "sync.topic.acls.enabled": true,
  "consumer.max.poll.records": 10000,
  "producer.acks": "all"
}

# 部署 MirrorMaker
kubectl apply -f mirrormaker2-deployment.yaml
```

---

### 第二阶段：高可用配置 (1 周)

#### 2.1 MySQL 高可用 (MHA)

```yaml
# MHA 配置文件 mha_manager.cnf

manager_workdir=/var/log/mha
manager_log=/var/log/mha/mha_manager.log
remote_workdir=/var/log/mha

master_ssh_user=root
ssh_port=22

user=monitor
password=monitor_pass
ping_interval=1

master_ip_failover_script=/usr/local/bin/master_ip_failover.sh
node_failure_detection_script=/usr/local/bin/node_failure_check.sh

failover_checkpoint_file=/var/log/mha/checkpoint.file
```

#### 2.2 Kubernetes HA 配置

```yaml
# etcd 高可用配置

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: etcd
            topologyKey: kubernetes.io/hostname
      
      containers:
      - name: etcd
        image: quay.io/coreos/etcd:v3.5.0
        
        resources:
          limits:
            cpu: "2"
            memory: "4Gi"
          requests:
            cpu: "1"
            memory: "2Gi"
        
        env:
        - name: ETCD_INITIAL_CLUSTER_STATE
          value: existing
        - name: ETCD_HEARTBEAT_INTERVAL
          value: "100"
        - name: ETCD_ELECTION_TIMEOUT
          value: "1000"
```

#### 2.3 Ingress HA 配置

```yaml
# Nginx Ingress Controller HA

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 确保始终有可用的 Ingress
  
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```

---

### 第三阶段：测试演练 (1 周)

#### 3.1 故障注入测试

```bash
#!/bin/bash
# chaos-test.sh - 混沌工程测试

echo "=== 开始混沌工程测试 ==="

# Test 1: 节点故障
echo "【测试 1】模拟节点故障..."
kubectl cordon worker-01
kubectl drain worker-01 --ignore-daemonsets --force
sleep 30
kubectl get pods -A | grep Pending
# 预期：Pod 重新调度到其他节点

# Test 2: Pod 故障
echo "【测试 2】模拟 Pod 故障..."
kubectl delete pod frontend-xxxxxx -n production
sleep 30
kubectl get pods -n production
# 预期：Deployment 自动重启新 Pod

# Test 3: 数据库主库故障
echo "【测试 3】模拟数据库主库故障..."
kubectl exec -it mysql-master-0 -n production -- systemctl stop mysqld
sleep 60
kubectl exec -it mysql-slave-0 -n production -- mysql -e "SHOW MASTER STATUS;"
# 预期：Slave 提升为 Master，RTO < 1 分钟

# Test 4: 网络分区
echo "【测试 4】模拟网络分区..."
iptables -I OUTPUT -d 10.1.0.0/24 -j DROP
sleep 30
kubectl exec -it order-service-0 -n production -- curl http://10.1.0.10:3306
# 预期：切换到备用链路，RTO < 5 分钟

# Test 5: 资源耗尽
echo "【测试 5】模拟内存不足..."
kubectl run stress-test --rm -ti --image=alpine \
  --command -- sh -c "dd if=/dev/zero of=/dev/null bs=1M count=10000"
sleep 30
kubectl top pods -n production
# 预期：OOM Killer 触发，容器重启

echo "=== 混沌测试完成 ==="
```

#### 3.2 故障切换演练

```bash
#!/bin/bash
# failover-drill.sh - 故障切换演练脚本

LOG_FILE="/root/logs/failover-$(date +%Y%m%d).log"

echo "=== 故障切换演练 ($(date '+%Y-%m-%d %H:%M:%S')) ===" > $LOG_FILE

# Step 1: 记录当前状态
kubectl get pods -A >> $LOG_FILE
kubectl get pvc,pv -A >> $LOG_FILE
mysql -h mysql-master-0 -e "SHOW GLOBAL STATUS LIKE 'Threads_connected';" >> $LOG_FILE

# Step 2: 通知应用停止写数据
curl -X POST http://order-service-api/admin/pause-writing

# Step 3: 等待队列清空
echo "等待 30 秒让队列清空..."
sleep 30

# Step 4: 启动故障切换
kubectl exec -it mha-manager -n database -- masterha_admin --command="start_master_to_slave_failover"

# Step 5: 监控切换过程
watch -n 5 "kubectl get pods -n production" >> $LOG_FILE

# Step 6: 切换完成后恢复服务
curl -X POST http://order-service-api/admin/resume-writing

# Step 7: 验证数据一致性
diff_db_data main_site backup_site

# Step 8: 回切到主站
kubectl exec -it mha-manager -n database -- masterha_admin --command="switch_to_slave"

echo "" >> $LOG_FILE
echo "=== 故障切换完成 ===" 
echo "切换时间：$(date)" >> $LOG_FILE
mysql -h mysql-new-master -e "SHOW GLOBAL STATUS LIKE 'Threads_connected';" >> $LOG_FILE

# 发送报告
mail -s "故障切换演练报告 $(date +%Y-%m-%d)" ops@company.com < $LOG_FILE
```

---

## 📊 容灾监控指标

### 监控 Dashboard 设计

```yaml
# Prometheus 监控规则

groups:
- name: disaster-recovery
  rules:
  # MySQL 主从延迟监控
  - alert: MySQLReplicationDelayHigh
    expr: mysql_global_status_sec_behind_master > 10
    for: 5m
    annotations:
      summary: "MySQL 主从延迟超过 10 秒"
      
  # 数据一致性检查
  - alert: DataConsistencyMismatch
    expr: data_consistency_diff_count > 0
    for: 1m
    annotations:
      summary: "检测到数据不一致"
      
  # 专线带宽使用率
  - alert: DedicatedLineBandwidthHigh
    expr: dedicated_line_bytes_in / 125000000 * 100 > 80
    for: 10m
    annotations:
      summary: "专线带宽使用率超过 80%"
      
  # 备用站健康检查
  - alert: BackupSiteUnhealthy
    expr: probe_success == 0 and cluster == "backup-site"
    annotations:
      summary: "备用站点不可达"
```

---

## 📋 容灾 Checklist

```markdown
# 容灾建设 Checklist

## 基础设施
- [ ] 双数据中心专线连通
- [ ] K8s 集群双活部署
- [ ] DNS 多活配置
- [ ] 负载均衡器 HA 配置

## 数据库
- [ ] MySQL 主从复制配置
- [ ] MHA 高可用部署
- [ ] 自动故障切换测试
- [ ] 数据一致性校验

## 中间件
- [ ] Redis 哨兵模式
- [ ] Kafka MirrorMaker 部署
- [ ] RabbitMQ 镜像队列
- [ ] Nacos 双活注册中心

## 存储
- [ ] NFS 双活挂载
- [ ] 对象存储跨区域复制
- [ ] 本地数据定时备份
- [ ] 备份完整性验证

## 监控告警
- [ ] 主从延迟监控
- [ ] 数据一致性监控
- [ ] 专线带宽监控
- [ ] 自动切换告警

## 应急预案
- [ ] 故障切换 SOP 文档
- [ ] 演练脚本准备
- [ ] 团队应急培训
- [ ] 联系方式通讯录
```

---

## 💰 项目成本分析

```
┌─────────────────────────────────────────────────────────────┐
│                   灾备项目成本明细                           │
│                                                             │
│  一、基础设施成本                                             │
│  ├── A 区数据中心 (15 台服务器): ¥60 万/年                    │
│  ├── B 区数据中心 (10 台服务器): ¥40 万/年                    │
│  ├── 专线带宽 (1Gbps): ¥15 万/年                             │
│  ├── DNS 服务：¥3 万/年                                       │
│  └── 小计：¥118 万/年                                      │
│                                                             │
│  二、软件成本                                                 │
│  ├── Oracle MySQL Enterprise: ¥5 万/年                       │
│  ├── MHA Manager: ¥3 万/年                                   │
│  ├── Monitoring Stack: ¥2 万/年                              │
│  └── 小计：¥9 万/年                                        │
│                                                             │
│  三、运维成本                                                 │
│  ├── 运维工程师：¥15 万/年                                   │
│  └── 定期演练费用：¥3 万/年                                   │
│  └── 小计：¥18 万/年                                       │
│                                                             │
│  总计：¥145 万/年                                           │
│                                                             │
│  ROI 分析                                                   │
│  ├── 避免单次故障损失：¥50-100 万                            │
│  ├── 满足合规要求：避免罚款 ¥50 万                           │
│  └── 品牌信誉保障：价值无法估量                              │
└─────────────────────────────────────────────────────────────┘
```

---

*金融系统灾备容灾架构 | ClawsOps ⚙️*