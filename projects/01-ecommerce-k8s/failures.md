# 电商平台 K8s 高可用 - 故障案例与改进

## 📋 故障回顾

这个项目在实施过程中经历了**12 次故障**，每一次都成为宝贵的经验积累。下面是主要故障的完整记录和分析。

---

## 🔥 故障一：首次部署 Pod 调度失败

### 现象
```bash
$ kubectl get pods -n production
NAME                                READY   STATUS    RESTARTS   AGE
frontend-7d59f6b84c-xk9j2           0/1     Pending   0          15m
order-service-67d5c4f7b8-zxvbn      0/1     Pending   0          15m
```

### 原因分析

```yaml
kubectl describe pod frontend-xxx
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5m    default-scheduler  0/13 nodes are available:
                                                           1 node(s) had untolerated taint {node-role.kubernetes.io/master: }
                                                           12 node(s) did not match Pod node selector
```

问题：Pod 有节点选择器，但 worker 节点没有对应标签。

### 解决方案

```bash
# 给 worker 节点打标签
kubectl label nodes worker-01~worker-10 tier=application environment=production

# 修改 Deployment 节点选择器
kubectl patch deployment frontend \
  --patch '{"spec":{"template":{"spec":{"nodeSelector":{"tier":"application"}}}}}'
```

### 改进措施

1. **文档记录**：运维手册增加"资源准备"章节
2. **自动化**：使用 Terraform 自动配置节点标签
3. **测试**：部署前运行 `kubectl drain` 模拟真实环境

---

## 🔥 故障二：存储卷挂载失败

### 现象
```bash
$ kubectl describe pod order-db-0 | grep Events
FailedMount Unable to attach or mount volumes: 
unmounted volumes=[data], unattached volumes=[data kube-api-access]: timed out waiting for the condition
```

### 原因分析

StorageClass 使用了动态 PV，但底层存储未就绪。

### 解决方案

```yaml
# 临时方案：使用空目录 (仅测试环境)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: order-db
spec:
  template:
    spec:
      containers:
      - name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        emptyDir: {}  # 仅测试环境使用
```

### 生产方案

```yaml
# 生产环境：使用本地 SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: "local-ssd"
  resources:
    requests:
      storage: 50Gi
```

### 改进措施

1. **明确存储类型**：区分测试/生产环境存储配置
2. **提前验证**：部署前检查 PV/PVC 状态
3. **监控告警**：PVC 绑定失败立即告警

---

## 🔥 故障三：数据库连接池耗尽

### 现象
```log
[ERROR] Too many connections -> Max execution time exceeded
```

```sql
-- 检查连接数
SHOW PROCESSLIST;
-- Total: 100 (超过 max_connections)

SHOW VARIABLES LIKE 'max_connections';
-- +-------------+-------+
-- | Variable_name | Value |
-- +-------------+-------+
-- | max_connections | 100 |
-- +-------------+-------+
```

### 原因分析

1. 订单高峰期 QPS 达到 5000
2. 数据库配置 max_connections=100
3. 应用层连接池默认值也是 100
4. 并发用户同时访问导致连接耗尽

### 解决方案

```yaml
# MySQL 优化配置
max_connections = 500
wait_timeout = 28800
interactive_timeout = 28800
thread_cache_size = 50
```

```yaml
# Spring Boot 连接池优化
spring:
  datasource:
    hikari:
      maximum-pool-size: 200
      minimum-idle: 50
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### 监控告警

```yaml
# 连接数告警规则
- alert: HighDatabaseConnections
  expr: mysql_global_status_threads_connected / 100 * 100 > 80
  for: 5m
  annotations:
    summary: "数据库连接数超过 80%"
```

---

## 🔥 故障四：Redis 内存溢出 OOM

### 现象
```bash
$ kubectl logs redis-master-0 -n middleware
...
OOM command not allowed when used memory > 'maxmemory'
Killed: Killed process 12345 (redis-server) total-vm:xxx
```

### 原因分析

1. Redis 配置 maxmemory=4GB
2. 缓存数据量持续增长
3. 缺少过期策略导致内存堆积
4. 大 Key 占用过多内存

### 解决方案

```yaml
# Redis 配置优化
maxmemory-policy allkeys-lru
maxmemory 4gb

# 清理大 Key
redis-cli --bigkeys

# 定期清理过期键
15 0 * * * redis-cli DEBUG SLEEP 10
```

### 最佳实践

```bash
#!/bin/bash
# weekly-memory-cleanup.sh

# 每周日凌晨 3 点执行
redis-cli INFO memory | grep used_memory_human
redis-cli MEMORY DOCTOR
redis-cli MEMORY PURGE
```

---

## 🔥 故障五：Ingress SSL 证书配置错误

### 现象
```bash
curl -k https://api.example.com
curl: (56) OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 104
```

### 原因分析

Cert-manager 生成的证书未正确应用到 Ingress。

### 解决方案

```yaml
# Certificate 资源
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-example-tls
spec:
  secretName: api-example-tls-secret
  dnsNames:
  - api.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
---
# Ingress TLS 配置
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-example-tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 80
```

---

## 💡 总体改进措施

### 流程改进

1. **变更管理**: 所有变更必须经过评审和测试
2. **发布窗口**: 非紧急修复只在维护窗口进行
3. **灰度发布**: 新功能先小流量灰度

### 监控增强

1. **应用层监控**: 全链路追踪 Jaeger
2. **业务指标**: 订单成功率实时看板
3. **用户视角**: Synthetic monitoring (定时请求)

### 架构优化

1. **服务降级**: 支付接口降级方案
2. **限流熔断**: Sentinel/Ratelimit 配置
3. **多活容灾**: 两地三中心规划

---

*电商平台 K8s 高可用 - 故障案例 | ClawsOps ⚙️*