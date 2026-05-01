# ClawsOps 自动化脚本库

## 📋 说明

这是我日常运维中积累的脚本，**能自动的绝不手动**。这些脚本都是实战检验过的，直接能用。

---

## 🔧 日常巡检脚本

### 系统健康检查

```bash
#!/bin/bash
# health-check.sh - 系统健康检查
# 用法：./health-check.sh

echo "=== ClawsOps 系统健康检查 ==="
echo "检查时间：$(date)"
echo ""

# CPU 检查
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
if [ $CPU_USAGE -gt 80 ]; then
    echo "❌ CPU 使用率过高：${CPU_USAGE}%"
else
    echo "✅ CPU 使用率正常：${CPU_USAGE}%"
fi

# 内存检查
MEM_USAGE=$(free | grep Mem | awk '{printf "%.1f", ($3/$2)*100}')
if [ ${MEM_USAGE%.*} -gt 85 ]; then
    echo "❌ 内存使用率过高：${MEM_USAGE}%"
else
    echo "✅ 内存使用率正常：${MEM_USAGE}%"
fi

# 磁盘检查
echo ""
echo "磁盘使用情况："
df -h | grep -E "^/dev" | while read line; do
    USAGE=$(echo $line | awk '{print $5}' | cut -d'%' -f1)
    MOUNT=$(echo $line | awk '{print $6}')
    if [ $USAGE -gt 80 ]; then
        echo "❌ $MOUNT 使用率过高：${USAGE}%"
    else
        echo "✅ $MOUNT 使用率正常：${USAGE}%"
    fi
done

# 服务检查
echo ""
echo "核心服务状态："
SERVICES="nginx mysql redis docker"
for svc in $SERVICES; do
    if systemctl is-active --quiet $svc; then
        echo "✅ $svc 运行正常"
    else
        echo "❌ $svc 未运行"
    fi
done

echo ""
echo "=== 检查完成 ==="
```

---

## 🔧 数据库运维脚本

### 自动备份脚本

```bash
#!/bin/bash
# mysql-backup.sh - MySQL 自动备份
# 用法：./mysql-backup.sh

BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
HOST="localhost"
USER="backup"
PASSWORD="BackupPassword123!"
DATABASES="mydb orders users"

# 创建备份目录
mkdir -p $BACKUP_DIR

echo "=== 开始备份 ==="

# 全库备份
mysqldump -h$HOST -u$USER -p$PASSWORD \
    --single-transaction \
    --routines \
    --triggers \
    --all-databases | gzip > $BACKUP_DIR/full_$DATE.sql.gz

# 单库备份
for db in $DATABASES; do
    mysqldump -h$HOST -u$USER -p$PASSWORD \
        --single-transaction $db | gzip > $BACKUP_DIR/${db}_$DATE.sql.gz
done

# 验证备份
echo "验证备份..."
gunzip -t $BACKUP_DIR/full_$DATE.sql.gz && echo "✅ 全库备份验证成功"

# 清理旧备份 (保留 7 天)
find $BACKUP_DIR -name "*.gz" -mtime +7 -delete
echo "清理 7 天前备份"

echo "=== 备份完成 ==="
echo "备份文件：$BACKUP_DIR/full_$DATE.sql.gz"
```

### 备份恢复脚本

```bash
#!/bin/bash
# mysql-restore.sh - MySQL 恢复脚本
# 用法：./mysql-restore.sh <backup_file>

BACKUP_FILE=$1
HOST="localhost"
USER="root"
PASSWORD="RootPassword123!"

if [ -z "$BACKUP_FILE" ]; then
    echo "请指定备份文件"
    exit 1
fi

echo "=== 开始恢复 ==="
echo "备份文件：$BACKUP_FILE"

# 确认恢复
read -p "确认恢复数据库？此操作将覆盖现有数据 (yes/no): " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
    echo "取消恢复"
    exit 0
fi

# 恢复数据库
gunzip -c $BACKUP_FILE | mysql -h$HOST -u$USER -p$PASSWORD

echo "=== 恢复完成 ==="
echo "请验证数据完整性"
```

---

## 🔧 服务管理脚本

### 批量重启服务

```bash
#!/bin/bash
# restart-services.sh - 批量重启服务
# 用法：./restart-services.sh <service_name>

SERVICE=$1

if [ -z "$SERVICE" ]; then
    echo "请指定服务名称"
    exit 1
fi

echo "=== 批量重启 $SERVICE ==="

# 获取所有节点
NODES=$(kubectl get nodes -o name)

for node in $NODES; do
    echo "重启 $node 上的 $SERVICE..."
    kubectl delete pod -l app=$SERVICE --field-selector spec.nodeName=$node
    sleep 5
done

echo "=== 重启完成 ==="
kubectl get pods -l app=$SERVICE
```

### 服务健康检查

```bash
#!/bin/bash
# service-health.sh - 服务健康检查
# 用法：./service-health.sh <service_name>

SERVICE=$1

if [ -z "$SERVICE" ]; then
    echo "请指定服务名称"
    exit 1
fi

echo "=== $SERVICE 健康检查 ==="

# 检查 Pod 状态
echo "Pod 状态："
kubectl get pods -l app=$SERVICE -o wide

# 检查 Pod 日志
echo ""
echo "最近日志："
kubectl logs -l app=$SERVICE --tail=20

# 检查 Pod 资源
echo ""
echo "资源使用："
kubectl top pods -l app=$SERVICE

# 检查 Service
echo ""
echo "Service 状态："
kubectl get svc -l app=$SERVICE

# 测试服务连通性
SERVICE_IP=$(kubectl get svc -l app=$SERVICE -o jsonpath='{.items[0].spec.clusterIP}')
echo ""
echo "测试连通性："
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://$SERVICE_IP/health
```

---

## 🔧 监控告警脚本

### 告警通知脚本

```bash
#!/bin/bash
# alert-notify.sh - 告警通知脚本
# 用法：./alert-notify.sh <severity> <message>

SEVERITY=$1
MESSAGE=$2
WEBHOOK_URL="https://oapi.dingtalk.com/robot/send?access_token=xxx"

# 构建消息
TITLE="[$SEVERITY 告警] ClawsOps"
CONTENT="$MESSAGE\n时间：$(date)\n节点：$(hostname)"

# 发送钉钉通知
curl -X POST $WEBHOOK_URL \
    -H "Content-Type: application/json" \
    -d "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"$TITLE\",\"text\":\"$CONTENT\"}}"

# 发送邮件 (P0/P1)
if [ "$SEVERITY" == "P0" ] || [ "$SEVERITY" == "P1" ]; then
    echo "$MESSAGE" | mail -s "$TITLE" ops-team@example.com
fi

# 发送短信 (P0)
if [ "$SEVERITY" == "P0" ]; then
    curl -X POST "https://sms-api.example.com/send" \
        -d "phone=138xxxx&message=$MESSAGE"
fi
```

---

## 🔧 清理脚本

### 日志清理脚本

```bash
#!/bin/bash
# log-cleanup.sh - 日志清理脚本
# 用法：./log-cleanup.sh

echo "=== 日志清理 ==="

# 清理系统日志
find /var/log -type f -name "*.log" -mtime +30 -delete
echo "清理 /var/log 30 天前日志"

# 清理应用日志
find /var/www/logs -type f -name "*.log" -mtime +7 -delete
echo "清理应用日志 7 天前"

# 清理 Docker 日志
find /var/lib/docker/containers -type f -name "*-json.log" -exec truncate -s 100M {} \;
echo "限制 Docker 日志大小 100M"

# 清理临时文件
find /tmp -type f -mtime +1 -delete
echo "清理临时文件"

echo "=== 清理完成 ==="
df -h
```

---

## 🔧 安全脚本

### 安全检查脚本

```bash
#!/bin/bash
# security-check.sh - 安全检查脚本
# 用法：./security-check.sh

echo "=== ClawsOps 安全检查 ==="

# SSH 检查
echo "SSH 配置检查："
grep "PermitRootLogin" /etc/ssh/sshd_config
grep "PasswordAuthentication" /etc/ssh/sshd_config

# 端口检查
echo ""
echo "开放端口："
netstat -tlnp

# 用户检查
echo ""
echo "具有 sudo 权限的用户："
grep -E "^sudo|^wheel" /etc/group

# 失败登录检查
echo ""
echo "最近失败登录："
lastb | head -10

# 密码过期检查
echo ""
echo "密码即将过期用户："
awk -F: '{if($5<7) print $1}' /etc/shadow

# 防火墙检查
echo ""
echo "防火墙状态："
ufw status || iptables -L | head -20

echo "=== 检查完成 ==="
```

---

## 📋 脚本使用规范

```markdown
# ClawsOps 脚本规范

## 基本要求
1. 所有脚本必须有注释说明用法
2. 关键操作必须有确认提示
3. 生产环境执行前必须 dry-run
4. 脚本必须有执行日志

## 命名规范
- 功能_对象.sh 如：backup_mysql.sh
- 动作_服务.sh 如：restart_nginx.sh

## 目录结构
/scripts/
├── daily/         # 日常巡检脚本
├── backup/        # 备份恢复脚本
├── deploy/        # 部署发布脚本
├── monitor/       # 监控告警脚本
├── cleanup/       # 清理脚本
└── security/      # 安全检查脚本
```

---

*ClawsOps 脚本库 v1.0 | 能自动的绝不手动*