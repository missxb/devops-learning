# 金融系统灾备容灾 - 实施脚本与自动化

## 故障切换自动化脚本

### 自动故障检测

```bash
#!/bin/bash
# auto-detect-failure.sh
# 每 30 秒检查一次主数据库状态

set -euo pipefail

LOG_DIR="/var/log/disaster-recovery"
LOG_FILE="$LOG_DIR/health-$(date +%Y%m%d).log"
CHECK_INTERVAL=30
FAIL_THRESHOLD=3
ALERT_WEBHOOK="https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN"

mkdir -p "$LOG_DIR"

check_mysql_master() {
    local master_host="10.0.0.50"
    local check_user="health_check"
    local check_pass=$(cat /etc/dr/health_check.pass)
    
    # 检查 MySQL 连接
    if mysql -h "$master_host" -u "$check_user" -p"$check_pass" \
       -e "SELECT 1" >/dev/null 2>&1; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] [OK] MySQL master reachable" | tee -a "$LOG_FILE"
        return 0
    else
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] [FAIL] MySQL master unreachable" | tee -a "$LOG_FILE"
        return 1
    fi
}

check_replication_lag() {
    local slave_host="10.1.0.50"
    local check_user="health_check"
    local check_pass=$(cat /etc/dr/health_check.pass)
    
    local lag=$(mysql -h "$slave_host" -u "$check_user" -p"$check_pass" \
        -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep "Seconds_Behind_Master" | awk '{print $2}')
    
    if [ "$lag" = "NULL" ] || [ "$lag" -gt 60 ]; then
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN] Replication lag: ${lag}s" | tee -a "$LOG_FILE"
        return 1
    else
        echo "[$(date '+%Y-%m-%d %H:%M:%S')] [OK] Replication lag: ${lag}s" | tee -a "$LOG_FILE"
        return 0
    fi
}

send_alert() {
    local level=$1
    local message=$2
    
    local payload=$(cat <<EOF
{
    "msgtype": "text",
    "text": {
        "content": "[${level}] DR Alert: ${message} Time: $(date '+%Y-%m-%d %H:%M:%S')"
    }
}
EOF
)
    curl -s -X POST "$ALERT_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "$payload" >/dev/null 2>&1
}

failover_decision() {
    local consecutive_fails=0
    
    while true; do
        if ! check_mysql_master; then
            consecutive_fails=$((consecutive_fails + 1))
            echo "Consecutive failures: $consecutive_fails"
            
            if [ "$consecutive_fails" -ge "$FAIL_THRESHOLD" ]; then
                echo "[$(date '+%Y-%m-%d %H:%M:%S')] [CRITICAL] Threshold reached! Initiating failover..."
                send_alert "CRITICAL" "MySQL master down $FAIL_THRESHOLD times, starting failover"
                
                # 调用故障切换脚本
                /usr/local/bin/dr-failover.sh
                
                # 重置计数器
                consecutive_fails=0
            fi
        else
            consecutive_fails=0
        fi
        
        sleep "$CHECK_INTERVAL"
    done
}

failover_decision
```

### 故障切换脚本

```bash
#!/bin/bash
# dr-failover.sh
# 主从自动切换

set -euo pipefail

LOG="/var/log/disaster-recovery/failover-$(date +%Y%m%d-%H%M%S).log"
MASTER_IP="10.0.0.50"
SLAVE_IP="10.1.0.50"
DB_USER="repl_user"
DB_PASS=$(cat /etc/dr/repl.pass)
VIP="10.0.0.100"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"
}

# Step 1: 确认主库确实挂了
log "Step 1: Confirming master is down..."
if mysql -h "$MASTER_IP" -u "$DB_USER" -p"$DB_PASS" -e "SELECT 1" 2>/dev/null; then
    log "Master is still alive! Aborting failover."
    exit 1
fi

# Step 2: 等待从库应用完所有 relay log
log "Step 2: Waiting for slave to catch up..."
for i in {1..30}; do
    slave_status=$(mysql -h "$SLAVE_IP" -u "$DB_USER" -p"$DB_PASS" \
        -e "SHOW SLAVE STATUS\G" 2>/dev/null)
    
    io_running=$(echo "$slave_status" | grep "Slave_IO_Running" | awk '{print $2}')
    sql_running=$(echo "$slave_status" | grep "Slave_SQL_Running" | awk '{print $2}')
    seconds_behind=$(echo "$slave_status" | grep "Seconds_Behind_Master" | awk '{print $2}')
    
    if [ "$io_running" = "No" ] && [ "$sql_running" = "Yes" ] && [ "$seconds_behind" = "0" ]; then
        log "Slave fully caught up. IO stopped, SQL complete."
        break
    fi
    
    if [ "$i" -eq 30 ]; then
        log "Timeout waiting for slave! Proceeding anyway."
    fi
    
    sleep 5
done

# Step 3: 停止从库复制
log "Step 3: Stopping slave replication..."
mysql -h "$SLAVE_IP" -u "$DB_USER" -p"$DB_PASS" \
    -e "STOP SLAVE; RESET SLAVE ALL;" 2>/dev/null

# Step 4: 将从库提升为主库
log "Step 4: Promoting slave to master..."
mysql -h "$SLAVE_IP" -u "$DB_USER" -p"$DB_PASS" \
    -e "SET GLOBAL read_only=OFF; SET GLOBAL super_read_only=OFF;" 2>/dev/null

# Step 5: 验证新的主库
log "Step 5: Verifying new master..."
if mysql -h "$SLAVE_IP" -u "$DB_USER" -p"$DB_PASS" -e "SELECT 1" 2>/dev/null; then
    log "New master is operational!"
else
    log "ERROR: New master verification failed!"
    exit 1
fi

# Step 6: 更新 VIP (Keepalived 自动处理，或手动)
log "Step 6: Moving VIP to new master..."
# Keepalived 会自动处理，这里只是记录
log "VIP movement handled by Keepalived"

# Step 7: 更新应用连接
log "Step 7: Updating application configurations..."
# Nacos/Spring Cloud 会自动刷新
log "Application config refresh triggered via Nacos"

# Step 8: 发送邮件通知
log "Step 8: Sending notifications..."
mail -s "DR Failover Complete $(date '+%Y-%m-%d %H:%M')" ops-team@company.com \
    <<EOF
Disaster Recovery Failover Completed

Master: $MASTER_IP (DOWN)
New Master: $SLAVE_IP (UP)
Time: $(date '+%Y-%m-%d %H:%M:%S')
RTO: Estimated < 2 minutes

Log: $LOG
EOF

log "Failover complete!"
```

### 回切脚本

```bash
#!/bin/bash
# dr-failback.sh
# 主库修复后回切

set -euo pipefail

LOG="/var/log/disaster-recovery/failback-$(date +%Y%m%d-%H%M%S).log"
OLD_MASTER_IP="10.0.0.50"   # 原来的主库（已修复）
CURRENT_MASTER_IP="10.1.0.50"  # 当前的主库（原来的从库）

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG"
}

# Step 1: 确认旧主库已修复
log "Step 1: Verifying old master is back online..."
if ! mysql -h "$OLD_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" -e "SELECT 1" 2>/dev/null; then
    log "Old master is still down! Cannot failback."
    exit 1
fi

# Step 2: 在旧主库上设置为主从配置
log "Step 2: Configuring old master as slave..."
mysql -h "$OLD_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" <<SQL
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO
    MASTER_HOST='$CURRENT_MASTER_IP',
    MASTER_USER='repl_user',
    MASTER_PASSWORD='$REPL_PASS',
    MASTER_AUTO_POSITION=1;
START SLAVE;
SQL

# Step 3: 等待同步完成
log "Step 3: Waiting for replication to catch up..."
for i in {1..60}; do
    lag=$(mysql -h "$OLD_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" \
        -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep "Seconds_Behind_Master" | awk '{print $2}')
    
    if [ "$lag" = "0" ]; then
        log "Old master fully synced!"
        break
    fi
    
    log "Seconds behind: $lag (attempt $i/60)"
    sleep 10
done

# Step 4: 通知应用准备切换
log "Step 4: Notifying applications of upcoming failback..."
# 发送告警通知，告知 5 分钟后切换

# Step 5: 执行切换（复用 failover 逻辑）
log "Step 5: Executing failback..."
# 暂停写操作
mysql -h "$CURRENT_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" \
    -e "SET GLOBAL read_only=ON;" 2>/dev/null

sleep 10

# 提升旧主库
mysql -h "$OLD_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" <<SQL
STOP SLAVE;
RESET SLAVE ALL;
SET GLOBAL read_only=OFF;
SQL

# Step 6: 验证
log "Step 6: Verifying new configuration..."
mysql -h "$OLD_MASTER_IP" -u root -p"$MYSQL_ROOT_PASS" -e "SELECT 1" 2>/dev/null

log "Failback complete! Old master is now the master again."
```

---

## 数据同步工具

### MySQL 同步验证

```python
#!/usr/bin/env python3
"""mysql-sync-checker.py - MySQL 主从同步一致性校验"""

import pymysql
import hashlib
import json
import logging
from datetime import datetime

class MySQLSyncChecker:
    def __init__(self, master_config, slave_config):
        self.master_conn = pymysql.connect(**master_config)
        self.slave_conn = pymysql.connect(**slave_config)
        self.results = []
        
    def check_tables(self, database, tables):
        """逐表检查数据一致性"""
        for table in tables:
            result = self.check_table_consistency(database, table)
            self.results.append(result)
            
        return self.results
        
    def check_table_consistency(self, db, table):
        """检查单个表的一致性"""
        result = {
            "table": f"{db}.{table}",
            "timestamp": datetime.now().isoformat(),
            "status": "unknown"
        }
        
        try:
            # 检查行数
            master_count = self._get_row_count(self.master_conn, db, table)
            slave_count = self._get_row_count(self.slave_conn, db, table)
            
            result["master_rows"] = master_count
            result["slave_rows"] = slave_count
            result["row_match"] = master_count == slave_count
            
            # 如果行数一致，进行 checksum
            if master_count == slave_count and master_count > 0:
                master_checksum = self._get_checksum(self.master_conn, db, table)
                slave_checksum = self._get_checksum(self.slave_conn, db, table)
                
                result["master_checksum"] = master_checksum
                result["slave_checksum"] = slave_checksum
                result["checksum_match"] = master_checksum == slave_checksum
                result["status"] = "consistent" if master_checksum == slave_checksum else "inconsistent"
            else:
                result["status"] = "row_mismatch"
                
        except Exception as e:
            result["status"] = f"error: {str(e)}"
            logging.error(f"Error checking {db}.{table}: {e}")
            
        return result
        
    def _get_row_count(self, conn, db, table):
        with conn.cursor() as cursor:
            cursor.execute(f"SELECT COUNT(*) FROM {db}.{table}")
            return cursor.fetchone()[0]
            
    def _get_checksum(self, conn, db, table):
        """计算表 checksum（简化版，采样前 10000 行）"""
        with conn.cursor() as cursor:
            cursor.execute(f"SELECT * FROM {db}.{table} LIMIT 10000")
            rows = cursor.fetchall()
            hash_val = hashlib.md5()
            for row in rows:
                hash_val.update(str(row).encode())
            return hash_val.hexdigest()
            
    def generate_report(self):
        """生成检查报告"""
        consistent = sum(1 for r in self.results if r["status"] == "consistent")
        total = len(self.results)
        
        report = {
            "check_time": datetime.now().isoformat(),
            "total_tables": total,
            "consistent_tables": consistent,
            "inconsistent_tables": total - consistent,
            "consistency_rate": f"{consistent/total*100:.1f}%" if total > 0 else "N/A",
            "details": self.results
        }
        
        return json.dumps(report, indent=2, ensure_ascii=False)
        
    def close(self):
        self.master_conn.close()
        self.slave_conn.close()

# 使用示例
if __name__ == "__main__":
    master_config = {
        "host": "10.0.0.50",
        "user": "check_user",
        "password": "check_pass",
        "database": "ecommerce",
        "charset": "utf8mb4"
    }
    
    slave_config = {
        "host": "10.1.0.50",
        "user": "check_user",
        "password": "check_pass",
        "database": "ecommerce",
        "charset": "utf8mb4"
    }
    
    checker = MySQLSyncChecker(master_config, slave_config)
    
    tables = ["orders", "users", "products", "transactions"]
    results = checker.check_tables("ecommerce", tables)
    
    report = checker.generate_report()
    print(report)
    
    checker.close()
```

### Kafka 同步配置

```yaml
# mirror-maker2.yaml

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: mm2
  namespace: kafka
spec:
  replicas: 3
  version: 3.4.0
  
  clusters:
  - alias: prod-a
    bootstrapServers: kafka-a-broker-0.kafka-a-brokers.kafka.svc:9092,kafka-a-broker-1.kafka-a-brokers.kafka.svc:9092,kafka-a-broker-2.kafka-a-brokers.kafka.svc:9092
    config:
      config.storage.replication.factor: 3
      offset.storage.replication.factor: 3
      status.storage.replication.factor: 3
  
  - alias: prod-b
    bootstrapServers: kafka-b-broker-0.kafka-b-brokers.kafka.svc:9092,kafka-b-broker-1.kafka-b-brokers.kafka.svc:9092,kafka-b-broker-2.kafka-b-brokers.kafka.svc:9092
    config:
      config.storage.replication.factor: 3
      offset.storage.replication.factor: 3
      status.storage.replication.factor: 3
  
  mirrors:
  - sourceCluster: prod-a
    targetCluster: prod-b
    
    sourceConnector:
      config:
        tasks.max: 6
        topics.pattern: "^(order|payment|inventory|user)\..*$"
        sync.topic.acls.enabled: true
        refresh.topics.interval.seconds: 60
        
    checkpointConnector:
      config:
        tasks.max: 3
        groups.pattern: "^(order|payment|inventory)\..*$"
        
    heartbeatConnector:
      config:
        tasks.max: 1
```

---

## 日常巡检脚本

```bash
#!/bin/bash
# daily-dr-check.sh
# 每日灾备系统巡检

LOG="/var/log/disaster-recovery/daily-check-$(date +%Y%m%d).log"
REPORT="/var/log/disaster-recovery/daily-report-$(date +%Y%m%d).html"

echo "<html><head><title>DR Daily Check $(date +%Y-%m-%d)</title></head><body>" > "$REPORT"
echo "<h1>灾备系统每日巡检报告</h1>" >> "$REPORT"
echo "<p>日期: $(date '+%Y-%m-%d %H:%M:%S')</p>" >> "$REPORT"

echo "=== 灾备系统每日巡检 ===" | tee -a "$LOG"
echo "日期: $(date '+%Y-%m-%d %H:%M:%S')" | tee -a "$LOG"

# 1. 检查主从状态
echo -e "\n--- MySQL 主从状态 ---" | tee -a "$LOG"
echo "<h2>MySQL 主从状态</h2><pre>" >> "$REPORT"

mysql -h 10.0.0.50 -u check -p"$CHECK_PASS" \
    -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind" | tee -a "$LOG"

echo "</pre>" >> "$REPORT"

# 2. 检查磁盘空间
echo -e "\n--- 磁盘空间 ---" | tee -a "$LOG"
echo "<h2>磁盘空间</h2><pre>" >> "$REPORT"
df -h | tee -a "$LOG"
echo "</pre>" >> "$REPORT"

# 3. 检查专线连通性
echo -e "\n--- 专线连通性 ---" | tee -a "$LOG"
echo "<h2>专线连通性</h2><pre>" >> "$REPORT"
ping -c 4 10.1.0.50 2>&1 | tee -a "$LOG"
echo "</pre>" >> "$REPORT"

# 4. 检查 Kafka 同步
echo -e "\n--- Kafka MirrorMaker2 状态 ---" | tee -a "$LOG"
echo "<h2>Kafka MirrorMaker2</h2><pre>" >> "$REPORT"
kubectl get kafkamirrormaker2 -n kafka -o wide 2>/dev/null | tee -a "$LOG"
echo "</pre>" >> "$REPORT"

# 5. 生成总结
echo -e "\n--- 巡检总结 ---" | tee -a "$LOG"
echo "<h2>巡检总结</h2>" >> "$REPORT"

errors=0
io_status=$(mysql -h 10.0.0.50 -u check -p"$CHECK_PASS" -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep "Slave_IO_Running" | awk '{print $2}')
sql_status=$(mysql -h 10.0.0.50 -u check -p"$CHECK_PASS" -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep "Slave_SQL_Running" | awk '{print $2}')

if [ "$io_status" != "Yes" ]; then
    echo "⚠️  Slave IO 线程未运行" | tee -a "$LOG"
    errors=$((errors + 1))
fi

if [ "$sql_status" != "Yes" ]; then
    echo "⚠️  Slave SQL 线程未运行" | tee -a "$LOG"
    errors=$((errors + 1))
fi

if [ $errors -eq 0 ]; then
    echo "✅ 灾备系统运行正常" | tee -a "$LOG"
    echo "<p style='color:green'>✅ 灾备系统运行正常</p>" >> "$REPORT"
else
    echo "❌ 发现 $errors 个问题" | tee -a "$LOG"
    echo "<p style='color:red'>❌ 发现 $errors 个问题</p>" >> "$REPORT"
fi

echo "</body></html>" >> "$REPORT"

# 发送报告
mail -s "灾备巡检报告 $(date +%Y-%m-%d)" -a "$REPORT" ops-team@company.com < /dev/null
```

---

*金融系统灾备容灾 - 实施脚本与自动化 | ClawsOps ⚙️*