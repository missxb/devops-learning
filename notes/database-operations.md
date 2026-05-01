# 数据库运维 (Database Ops)

## 📋 概述

数据库是企业的核心资产，数据库运维涵盖部署、监控、备份、优化、安全等全生命周期管理。

## 🎯 学习目标

- 掌握数据库高可用架构
- 实施备份恢复策略
- 性能监控与优化
- 容量规划与扩展
- 安全加固与审计

## 🏗️ 数据库高可用架构

### MySQL 主从复制

```
┌─────────────────────────────────────────────────────────────┐
│                    MySQL 高可用架构                          │
│                                                             │
│  ┌─────────────┐                                           │
│  │   Master    │  192.168.1.10:3306                       │
│  │  (读写)     │  binlog: mysql-bin.000001                │
│  └──────┬──────┘                                           │
│         │                                                   │
│         │ 复制                                             │
│         ▼                                                   │
│  ┌─────────────┐     ┌─────────────┐                      │
│  │  Slave 1    │     │  Slave 2    │                      │
│  │  (只读)     │     │  (只读)     │                      │
│  │192.168.1.11 │     │192.168.1.12 │                      │
│  └─────────────┘     └─────────────┘                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              MHA / Orchestrator                      │   │
│  │              (自动故障转移)                          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### MySQL 配置

```ini
# Master 配置 (my.cnf)
[mysqld]
server-id = 1
log-bin = mysql-bin
binlog_format = ROW
binlog_expire_logs_seconds = 604800
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
relay_log = relay-bin
read_only = OFF

# Slave 配置 (my.cnf)
[mysqld]
server-id = 2
log-bin = mysql-bin
binlog_format = ROW
gtid_mode = ON
enforce_gtid_consistency = ON
log_slave_updates = ON
relay_log = relay-bin
read_only = ON
super_read_only = ON
```

### 复制配置

```sql
-- Master 创建复制用户
CREATE USER 'repl'@'%' IDENTIFIED BY 'ReplPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 查看 Master 状态
SHOW MASTER STATUS;

-- Slave 配置复制
CHANGE MASTER TO
  MASTER_HOST='192.168.1.10',
  MASTER_USER='repl',
  MASTER_PASSWORD='ReplPassword123!',
  MASTER_AUTO_POSITION=1;

-- 启动复制
START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

## 💾 备份恢复策略

### 备份类型对比

| 类型 | 工具 | 优点 | 缺点 |
|------|------|------|------|
| **逻辑备份** | mysqldump | 灵活、可跨版本 | 速度慢、恢复慢 |
| **物理备份** | XtraBackup | 速度快、支持增量 | 占用空间大 |
| **快照备份** | 云盘快照 | 秒级备份恢复 | 依赖云平台 |

### mysqldump 备份

```bash
#!/bin/bash
# 全量备份脚本

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
HOST="192.168.1.10"
USER="backup"
PASSWORD="BackupPassword123!"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 全量备份
mysqldump -h $HOST -u $USER -p$PASSWORD \
  --single-transaction \
  --master-data=2 \
  --routines \
  --triggers \
  --events \
  --all-databases | gzip > $BACKUP_DIR/full_$DATE.sql.gz

# 备份 binlog
mysqlbinlog --read-from-remote-server --host=$HOST \
  --user=$USER --password=$PASSWORD \
  --raw --stop-never mysql-bin.* > $BACKUP_DIR/binlog_$DATE

# 删除 7 天前备份
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete

# 验证备份
gunzip -t $BACKUP_DIR/full_$DATE.sql.gz
```

### XtraBackup 备份

```bash
#!/bin/bash
# 全量备份

BACKUP_DIR="/backup/xtrabackup"
DATE=$(date +%Y%m%d_%H%M%S)

# 全量备份
xtrabackup --backup \
  --target-dir=$BACKUP_DIR/full_$DATE \
  --user=backup \
  --password=BackupPassword123!

# 准备备份
xtrabackup --prepare \
  --target-dir=$BACKUP_DIR/full_$DATE

# 增量备份
xtrabackup --backup \
  --target-dir=$BACKUP_DIR/incr_$DATE \
  --incremental-basedir=$BACKUP_DIR/full_$DATE \
  --user=backup \
  --password=BackupPassword123!
```

### 恢复流程

```bash
# mysqldump 恢复
gunzip -c full_20260501.sql.gz | mysql -u root -p

# XtraBackup 恢复
# 1. 停止 MySQL
systemctl stop mysqld

# 2. 清空数据目录
rm -rf /var/lib/mysql/*

# 3. 恢复数据
xtrabackup --copy-back --target-dir=/backup/xtrabackup/full_20260501

# 4. 修改权限
chown -R mysql:mysql /var/lib/mysql

# 5. 启动 MySQL
systemctl start mysqld
```

## 📊 性能监控

### 关键指标

| 指标 | 说明 | 阈值 |
|------|------|------|
| **QPS** | 每秒查询数 | 视业务而定 |
| **TPS** | 每秒事务数 | 视业务而定 |
| **连接数** | 当前连接数 | < 80% max_connections |
| **慢查询** | 执行慢的 SQL | < 100/分钟 |
| **缓冲池命中率** | InnoDB 缓冲池 | > 99% |
| **复制延迟** | Slave 延迟秒数 | < 10s |

### Prometheus 监控

```yaml
# MySQL Exporter 配置
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: mysql-monitor
spec:
  selector:
    matchLabels:
      app: mysql
  endpoints:
  - port: metrics
    interval: 15s
---
# 告警规则
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: mysql-alerts
spec:
  groups:
  - name: mysql
    rules:
    - alert: MySQLDown
      expr: mysql_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "MySQL 实例宕机"
    
    - alert: MySQLReplicationLag
      expr: mysql_slave_status_seconds_behind_master > 60
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "MySQL 复制延迟超过 60 秒"
    
    - alert: MySQLHighConnections
      expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections > 0.8
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "MySQL 连接数超过 80%"
    
    - alert: MySQLSlowQueries
      expr: rate(mysql_global_status_slow_queries[5m]) > 10
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "MySQL 慢查询过多"
```

### 慢查询分析

```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SET GLOBAL log_queries_not_using_indexes = 'ON';

-- 查看慢查询
SHOW VARIABLES LIKE 'slow_query_log%';

-- 分析慢查询日志
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

-- 查看当前执行
SHOW PROCESSLIST;

-- 查看执行计划
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- 查看表统计
SELECT table_name, table_rows, data_length, index_length
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length DESC;
```

## 🔧 性能优化

### 索引优化

```sql
-- 创建索引
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_status_created ON orders(status, created_at);

-- 删除未使用索引
SELECT * FROM sys.schema_unused_indexes;

-- 索引维护
OPTIMIZE TABLE users;
ANALYZE TABLE orders;
```

### 配置优化

```ini
# my.cnf 优化配置
[mysqld]
# 内存配置
innodb_buffer_pool_size = 4G
innodb_buffer_pool_instances = 8
innodb_log_file_size = 512M
innodb_log_buffer_size = 64M

# 连接配置
max_connections = 500
thread_cache_size = 50
table_open_cache = 2000

# IO 配置
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000
innodb_flush_method = O_DIRECT

# 日志配置
slow_query_log = 1
long_query_time = 1
log_queries_not_using_indexes = 1
```

### SQL 优化

```sql
-- ❌ 避免
SELECT * FROM orders;  -- 全表扫描
SELECT * FROM users WHERE YEAR(created_at) = 2026;  -- 索引失效
SELECT * FROM orders WHERE status = 'pending' OR status = 'processing';  -- OR 导致索引失效

-- ✅ 推荐
SELECT id, user_id, amount FROM orders WHERE status = 'pending';  -- 只选需要的列
SELECT * FROM users WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01';  -- 范围查询
SELECT * FROM orders WHERE status IN ('pending', 'processing');  -- 使用 IN
```

## 🔒 安全加固

### 用户权限

```sql
-- 创建应用用户
CREATE USER 'app'@'192.168.1.%' IDENTIFIED BY 'AppPassword123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'app'@'192.168.1.%';

-- 创建只读用户
CREATE USER 'readonly'@'192.168.1.%' IDENTIFIED BY 'ReadPassword123!';
GRANT SELECT ON mydb.* TO 'readonly'@'192.168.1.%';

-- 创建备份用户
CREATE USER 'backup'@'192.168.1.%' IDENTIFIED BY 'BackupPassword123!';
GRANT RELOAD, LOCK TABLES, REPLICATION CLIENT, SELECT ON *.* TO 'backup'@'192.168.1.%';

-- 回收权限
REVOKE DELETE ON mydb.* FROM 'app'@'192.168.1.%';

-- 查看权限
SHOW GRANTS FOR 'app'@'192.168.1.%';
```

### 审计配置

```sql
-- 安装审计插件
INSTALL PLUGIN audit_log SONAME 'audit_log.so';

-- 配置审计
SET GLOBAL audit_log_policy = ALL;
SET GLOBAL audit_log_format = JSON;
SET GLOBAL audit_log_file = /var/log/mysql/audit.log;

-- 查看审计日志
cat /var/log/mysql/audit.log | jq .
```

## 📈 容量规划

### 存储增长预测

```
当前数据量：100GB
日增数据：500MB
月增数据：15GB
年增数据：180GB

存储规划：
- 当前：200GB (预留 50%)
- 1 年后：400GB
- 3 年后：1TB

分库分表阈值：
- 单表数据量：> 1000 万行
- 单库大小：> 100GB
- QPS: > 5000
```

### 分库分表策略

```sql
-- 水平分表 (按用户 ID 取模)
-- users_0: user_id % 10 = 0
-- users_1: user_id % 10 = 1
-- ...
-- users_9: user_id % 10 = 9

-- 垂直分库
-- 用户库：users, profiles
-- 订单库：orders, order_items
-- 商品库：products, categories

-- 使用中间件
-- ShardingSphere
-- MyCAT
-- Vitess
```

## 📚 参考资料

- [MySQL 官方文档](https://dev.mysql.com/doc/)
- [Percona Toolkit](https://www.percona.com/software/percona-toolkit)
- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
