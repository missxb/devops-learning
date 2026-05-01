# 电商平台 K8s 高可用 - 运维手册

## 📋 日常巡检

### 每日巡检脚本

```bash
#!/bin/bash
# daily-ops.sh - 每日运维巡检

LOG_FILE="/root/logs/daily-ops-$(date +%Y%m%d).log"
echo "=== 每日运维巡检 ($(date '+%Y-%m-%d %H:%M:%S')) ===" > $LOG_FILE

echo "" >> $LOG_FILE
echo "【1. 集群节点状态】" >> $LOG_FILE
kubectl get nodes >> $LOG_FILE 2>&1

echo "" >> $LOG_FILE
echo "【2. Pod 运行状态】" >> $LOG_FILE
kubectl get pods -A --field-selector=status.phase!=Running | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【3. 资源使用情况】" >> $LOG_FILE
kubectl top nodes >> $LOG_FILE 2>&1

echo "" >> $LOG_FILE
echo "【4. 关键服务部署】" >> $LOG_FILE
kubectl get deployment,svc,pod -n production | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【5. 存储状态】" >> $LOG_FILE
kubectl get pvc,pv -n production | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【6. 数据库连接数】" >> $LOG_FILE
kubectl exec -it order-service-db-0 -n production -- mysql -e "SHOW PROCESSLIST;" | wc -l | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【7. Redis 内存使用】" >> $LOG_FILE
kubectl exec -it redis-master-0 -n middleware -- redis-cli INFO memory | grep used_memory_human: | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【8. Kafka Topic 状态】" >> $LOG_FILE
kubectl exec -it kafka-0 -n middleware -- kafka-topics.sh --describe --topic orders | tee -a $LOG_FILE

echo "" >> $LOG_FILE
echo "【9. 监控告警状态】" >> $LOG_FILE
kubectl port-forward svc/alertmanager-main 9093:9093 &>/dev/null &
FIRING=$(curl -s http://localhost:9093/api/v1/alerts | jq '.data[].status.state')
kill %1
echo "$FIRING" >> $LOG_FILE

echo "" >> $LOG_FILE
echo "【10. 日志错误数量 (近 1 小时)】" >> $LOG_FILE
kubectl logs -n production --since=1h | grep -ci error | tee -a $LOG_FILE

# 发送报告
mail -s "每日运维巡检报告 $(date +%Y-%m-%d)" ops@company.com < $LOG_FILE

echo "✅ 巡检完成，报告：$LOG_FILE"
```

---

## 🔧 故障应急处理

### P0 故障响应流程

```markdown
# P0 故障应急响应 SOP

## Step 1: 确认 (5 分钟内)
├── 收到告警通知
├── 检查告警详情
├── 确认影响范围
└── 拉起应急群聊

## Step 2: 止血 (15 分钟内)
├── 优先恢复业务
├── 回滚最近变更
├── 流量切至备用链路
└── 扩容关键服务

## Step 3: 定位 (30 分钟+)
├── 收集证据
│   ├── 监控数据截图
│   ├── 错误日志
│   └── 时间线记录
├── 根因分析
│   └── 5 Why 分析法
└── 临时修复方案

## Step 4: 验证
├── 功能测试
├── 性能回归
└── 监控系统观察

## Step 5: 复盘
├── 故障报告
├── 改进措施
└── 文档更新
```

### 常见故障处理

#### 案例 1: API 响应超时

```bash
# 症状：QPS 下降，延迟飙升

# 步骤 1: 快速诊断
kubectl top pods -n production --sort-by=cpu
kubectl top pods -n production --sort-by=memory

# 步骤 2: 查看错误日志
kubectl logs -n production -l app=order-service --tail=50 | grep ERROR

# 步骤 3: 检查数据库连接
kubectl exec -it order-service-db-0 -n production -- mysql -e "SHOW PROCESSLIST;"

# 发现原因：慢查询导致连接池满

# 步骤 4: 临时修复
# 找到慢 SQL 并 Kill
kubectl exec -it order-service-db-0 -n production -- mysql -e "KILL QUERY 123;"

# 步骤 5: 长期方案
# 添加索引优化 SQL
```

#### 案例 2: Pod 频繁重启

```bash
# 症状：Pod CrashLoopBackOff

# 诊断步骤
kubectl describe pod <pod-name> -n production

# 查看事件
kubectl describe pod <pod-name> -n production | grep Events

# 查看容器日志
kubectl logs <pod-name> -n production --previous

# 可能原因及处理
# 1. OOMKill -> 增加内存限制
kubectl patch deployment frontend -p '{"spec":{"template":{"spec":{"containers":[{"name":"frontend","resources":{"limits":{"memory":"1Gi"}}}]}}}}'

# 2. 启动命令错误 -> 修复镜像或配置
kubectl edit deployment <deployment-name>

# 3. 配置问题 -> 检查 ConfigMap/Secret
kubectl get configmap -n production
kubectl get secret -n production
```

#### 案例 3: 磁盘空间不足

```bash
# 检查磁盘使用
df -h

# 查看 Pod 日志大小
du -sh /var/log/pods/*/*/logs

# 清理旧日志
kubectl delete po --all -n logging --grace-period=0
kubectl gc pods --force-deletion=true -n kube-system

# 调整日志轮转
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-rotate-config
  namespace: logging
data:
  rotate.conf: |
    [service]
        input.file.rotate = true
        output.logrotate.rotate = true
        output.logrotate.compress = true
        output.logrotate.maxsize = 100MB
        output.logrotate.maxcopies = 5
        output.logrotate.interval = daily
EOF
```

---

## 🚀 扩缩容操作

### 手动扩容

```bash
# 扩容订单服务
kubectl scale deployment order-service --replicas=20 -n production

# 验证
kubectl get pods -n production -w

# 预期结果：新 Pod 自动调度，Load Balancer 自动添加后端
```

### HPA 自动扩缩容

```yaml
# hpa.yaml

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 10
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---

## 🔄 升级策略

### Kubernetes 版本升级

```bash
# 滚动升级流程

# Step 1: 备份 Etcd
etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Step 2: 升级 master 节点
# 依次在 master-01, master-02, master-03 执行

yum update kubeadm-1.25.0 kubeadm-1.25.0 kubelet-1.25.0 kubectl-1.25.0
kubeadm upgrade plan
kubeadm upgrade apply v1.25.0
systemctl restart kubelet

# Step 3: 升级 worker 节点
yum update kubeadm kubelet kubectl
kubeadm upgrade node
systemctl restart kubelet

# Step 4: 验证升级
kubectl version
kubectl get nodes
```

---

## 🛡️ 安全加固

### RBAC 权限控制

```yaml
# restricted-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch"]  # 不允许 create/delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: production
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### NetworkPolicy 网络隔离

```yaml
# default-deny-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# allow-traffic.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

---

## 📞 联系信息

### 值班联系方式

```
On-Call 值班表:
├── 主值班：张三 (zhangsan@company.com)
├── 副值班：李四 (lisi@company.com)
└── 紧急联系人：王五 (wangwu@company.com)

Slack 应急群: #ecommerce-emergency
钉钉群：https://dingtalk.group/example
电话会议号：123-456-7890
```

---

*电商平台 K8s 高可用 - 运维手册 | ClawsOps ⚙️*