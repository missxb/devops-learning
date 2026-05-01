# 灾备与容灾 (Disaster Recovery)

## 📋 概述

灾备是保障业务连续性的最后一道防线，通过数据备份、系统冗余、故障切换等手段，确保在灾难发生时能够快速恢复业务。

## 🎯 学习目标

- 理解灾备核心指标 (RTO/RPO)
- 设计灾备架构
- 实施备份策略
- 配置故障切换
- 执行灾备演练

## 🏗️ 灾备架构设计

### 灾备级别

```
┌─────────────────────────────────────────────────────────────┐
│                    灾备能力成熟度                            │
│                                                             │
│  Level 5: 业务连续性                                        │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  多活数据中心  │  自动切换  │  零数据丢失  │  RTO<1min│  │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  Level 4: 实时灾备                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  主从数据中心  │  自动切换  │  秒级延迟  │  RTO<5min │  │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  Level 3: 在线灾备                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  热备数据中心  │  手动切换  │  分钟级延迟│  RTO<30min│  │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  Level 2: 离线灾备                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  温备数据中心  │  手动切换  │  小时级延迟│  RTO<4h   │  │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  Level 1: 数据备份                                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  备份存储    │  手动恢复  │  天级恢复  │  RTO<24h   │  │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 核心指标

| 指标 | 说明 | 计算公式 | 示例 |
|------|------|----------|------|
| **RTO** | 恢复时间目标 | 业务中断到恢复的时间 | < 30 分钟 |
| **RPO** | 恢复点目标 | 可容忍的数据丢失量 | < 5 分钟 |
| **MTTR** | 平均修复时间 | 总修复时间/故障次数 | < 1 小时 |
| **MTBF** | 平均故障间隔 | 总运行时间/故障次数 | > 720 小时 |

### 典型架构

```
┌─────────────────────────────────────────────────────────────┐
│                    两地三中心架构                            │
│                                                             │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │   北京主中心     │────────►│   北京从中心     │           │
│  │   (生产)       │ 同步复制 │   (同城灾备)    │           │
│  │   192.168.1.x  │         │   192.168.2.x  │           │
│  └────────┬────────┘         └─────────────────┘           │
│           │                                                 │
│           │ 异步复制                                        │
│           ▼                                                 │
│  ┌─────────────────┐                                       │
│  │   上海灾备中心   │                                       │
│  │   (异地灾备)    │                                       │
│  │   10.0.1.x     │                                       │
│  └─────────────────┘                                       │
│                                                             │
│  距离：北京 - 上海 约 1200km                                │
│  延迟：约 20-30ms                                          │
└─────────────────────────────────────────────────────────────┘
```

## 💾 备份策略

### 3-2-1 备份原则

```
3 份数据副本
  ├── 1 份生产数据
  ├── 1 份本地备份
  └── 1 份异地备份

2 种存储介质
  ├── 磁盘 (快速恢复)
  └── 磁带/对象存储 (长期归档)

1 个异地存储
  └── 不同地理位置 (防止区域性灾难)
```

### 备份计划

```yaml
# 备份策略配置
backup_policy:
  # 数据库备份
  database:
    full_backup:
      schedule: "0 2 * * 0"  # 每周日 2:00
      retention: 30d
      storage: oss://backup-bucket/database/full
    
    incremental_backup:
      schedule: "0 2 * * 1-6"  # 每天 2:00
      retention: 7d
      storage: oss://backup-bucket/database/incr
    
    binlog_backup:
      schedule: "0 */1 * * *"  # 每小时
      retention: 7d
      storage: oss://backup-bucket/database/binlog

  # 文件备份
  files:
    schedule: "0 3 * * *"  # 每天 3:00
    retention: 90d
    storage: oss://backup-bucket/files

  # 配置备份
  config:
    schedule: "0 4 * * 0"  # 每周日 4:00
    retention: 365d
    storage: oss://backup-bucket/config
```

### 备份脚本

```bash
#!/bin/bash
# 完整备份脚本

set -e

BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/full"
LOG_FILE="/var/log/backup_$BACKUP_DATE.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "=== 开始备份 ==="

# 1. 数据库备份
log "备份数据库..."
mysqldump -h db-master -u backup -p'xxx' \
  --single-transaction \
  --all-databases | gzip > $BACKUP_DIR/mysql_$BACKUP_DATE.sql.gz

# 2. Redis 备份
log "备份 Redis..."
redis-cli -h redis-master BGSAVE
sleep 5
cp /var/lib/redis/dump.rdb $BACKUP_DIR/redis_$BACKUP_DATE.rdb

# 3. 应用文件备份
log "备份应用文件..."
tar -czf $BACKUP_DIR/app_$BACKUP_DATE.tar.gz \
  /var/www/html \
  /etc/nginx \
  /etc/mysql

# 4. 上传到对象存储
log "上传到 OSS..."
ossutil cp -r $BACKUP_DIR oss://backup-bucket/daily/$BACKUP_DATE/

# 5. 上传到异地
log "同步到异地..."
rsync -avz $BACKUP_DIR/ backup@shanghai:/backup/daily/$BACKUP_DATE/

# 6. 清理本地旧备份
log "清理 7 天前备份..."
find $BACKUP_DIR -mtime +7 -delete

# 7. 验证备份
log "验证备份完整性..."
ossutil ls oss://backup-bucket/daily/$BACKUP_DATE/

log "=== 备份完成 ==="

# 发送通知
curl -X POST "https://oapi.dingtalk.com/robot/send?access_token=xxx" \
  -H "Content-Type: application/json" \
  -d "{\"msgtype\":\"text\",\"text\":{\"content\":\"备份完成：$BACKUP_DATE\"}}"
```

## 🔄 故障切换

### 数据库切换

```bash
#!/bin/bash
# MySQL 主从切换脚本

MASTER_HOST="192.168.1.10"
SLAVE_HOST="192.168.1.11"

# 1. 停止主库写入
log "停止主库写入..."
ssh root@$MASTER_HOST "mysql -e 'SET GLOBAL read_only=ON;'"

# 2. 等待从库同步
log "等待从库同步..."
while true; do
    LAG=$(ssh root@$SLAVE_HOST "mysql -e 'SHOW SLAVE STATUS\G' | grep Seconds_Behind_Master | awk '{print $2}'")
    if [ "$LAG" == "0" ]; then
        break
    fi
    log "复制延迟：$LAG 秒"
    sleep 1
done

# 3. 停止从库复制
log "停止从库复制..."
ssh root@$SLAVE_HOST "mysql -e 'STOP SLAVE;'"

# 4. 提升从库为主库
log "提升从库为主库..."
ssh root@$SLAVE_HOST "mysql -e 'RESET SLAVE ALL; RESET MASTER;'"
ssh root@$SLAVE_HOST "mysql -e 'SET GLOBAL read_only=OFF;'"

# 5. 更新应用配置
log "更新应用连接配置..."
# 更新配置中心/负载均衡器

# 6. 验证新主库
log "验证新主库..."
ssh root@$SLAVE_HOST "mysql -e 'SHOW MASTER STATUS\G'"

log "切换完成！新主库：$SLAVE_HOST"
```

### Kubernetes 故障切换

```yaml
# 多集群故障切换
apiVersion: v1
kind: ConfigMap
metadata:
  name: failover-config
data:
  primary-cluster: "cluster-a"
  secondary-cluster: "cluster-b"
  failover-threshold: "3"
---
# 故障切换控制器
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failover-controller
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: controller
        image: failover-controller:1.0
        env:
        - name: PRIMARY_KUBECONFIG
          value: /etc/kubeconfig/primary
        - name: SECONDARY_KUBECONFIG
          value: /etc/kubeconfig/secondary
```

## 🧪 灾备演练

### 演练计划

```yaml
# 灾备演练计划
drill_plan:
  # 桌面演练 (每月)
  tabletop:
    frequency: monthly
    participants:
      - 运维团队
      - 开发团队
      - 业务团队
    scenarios:
      - 数据库主库宕机
      - 机房断电
      - 网络中断

  # 技术演练 (每季度)
  technical:
    frequency: quarterly
    scenarios:
      - 数据库主从切换
      - 应用故障转移
      - DNS 切换

  # 全链路演练 (每年)
  full_drill:
    frequency: yearly
    scope: 完整业务链路
    duration: 4-8 小时
    notification: 提前 2 周通知
```

### 演练检查清单

```markdown
# 灾备演练检查清单

## 演练前
- [ ] 确定演练时间和范围
- [ ] 通知相关团队
- [ ] 备份当前生产状态
- [ ] 准备回滚方案
- [ ] 准备监控仪表盘

## 演练中
- [ ] 记录每个步骤时间
- [ ] 监控系统指标
- [ ] 记录遇到的问题
- [ ] 验证业务功能

## 演练后
- [ ] 恢复生产环境
- [ ] 验证数据完整性
- [ ] 编写演练报告
- [ ] 更新灾备文档
- [ ] 制定改进计划
```

### 演练报告模板

```markdown
# 灾备演练报告

## 基本信息
- 演练日期：2026-05-01
- 演练类型：数据库主从切换
- 参与人员：张三、李四、王五
- 持续时间：45 分钟

## 演练场景
模拟主库 (192.168.1.10) 宕机，切换到从库 (192.168.1.11)

## 时间线
- 14:00 - 开始演练
- 14:02 - 检测到主库异常
- 14:05 - 确认需要切换
- 14:10 - 执行切换脚本
- 14:15 - 从库提升为主库
- 14:20 - 应用连接切换完成
- 14:30 - 业务验证通过
- 14:45 - 演练结束

## 指标达成
| 指标 | 目标 | 实际 | 状态 |
|------|------|------|------|
| RTO | 30min | 20min | ✅ |
| RPO | 5min | 0min | ✅ |
| 数据完整性 | 100% | 100% | ✅ |

## 问题与改进
1. 问题：切换脚本需要手动确认
   改进：实现自动化切换 (负责人：张三，截止：2026-05-15)

2. 问题：应用连接池刷新慢
   改进：优化连接池配置 (负责人：李四，截止：2026-05-10)

## 结论
演练成功，RTO/RPO 均达到目标，建议每季度执行一次
```

## 📊 监控告警

### 灾备健康检查

```yaml
# 灾备监控配置
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-alerts
spec:
  groups:
  - name: disaster-recovery
    rules:
    - alert: ReplicationLag
      expr: mysql_slave_status_seconds_behind_master > 60
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "数据库复制延迟超过 60 秒"
    
    - alert: BackupFailed
      expr: backup_success == 0
      for: 1h
      labels:
        severity: critical
      annotations:
        summary: "备份失败"
    
    - alert: DRSiteUnreachable
      expr: probe_dr_site == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "灾备站点不可达"
    
    - alert: BackupStorageLow
      expr: backup_storage_available / backup_storage_total < 0.2
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "备份存储空间不足 20%"
```

## 📚 参考资料

- 《灾难恢复规划与设计》
- [AWS 灾备最佳实践](https://aws.amazon.com/architecture/disaster-recovery/)
- [阿里云灾备方案](https://help.aliyun.com/product/53240.html)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
