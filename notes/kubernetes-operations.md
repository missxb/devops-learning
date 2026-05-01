# Kubernetes 运维实战

## 📋 概述

Kubernetes 是容器编排的事实标准，掌握 K8s 运维是云原生工程师的核心能力。

## 🎯 学习目标

- 理解 K8s 架构设计
- 掌握核心资源管理
- 配置高可用集群
- 实施监控和日志
- 故障诊断和调优

## 🏗️ K8s 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Control Plane (控制平面)                  │
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │   etcd    │  │  API      │  │ Scheduler │              │
│  │ (键值存储) │  │  Server   │  │ (调度器)  │              │
│  └───────────┘  └───────────┘  └───────────┘              │
│                                                             │
│  ┌───────────────────────────────────────────────────┐    │
│  │          Controller Manager (控制器管理器)          │    │
│  │  - Node Controller    - Replication Controller    │    │
│  │  - Endpoint Controller - Service Account Controller│   │
│  └───────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ API
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Worker Nodes (工作节点)                   │
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │  kubelet  │  │  kube-    │  │ Container │              │
│  │ (节点代理) │  │  proxy    │  │  Runtime  │              │
│  └───────────┘  └───────────┘  └───────────┘              │
│                                                             │
│  ┌───────────────────────────────────────────────────┐    │
│  │                    Pods                            │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐           │    │
│  │  │Container│  │Container│  │Container│           │    │
│  │  └─────────┘  └─────────┘  └─────────┘           │    │
│  └───────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## 📦 核心资源

### 1. Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "200m"
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
```

### 2. Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 80
```

### 3. Service

```yaml
# ClusterIP (默认)
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

# LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer

# NodePort
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
```

### 4. ConfigMap 和 Secret

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgres://localhost:5432/mydb"
  LOG_LEVEL: "info"

# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: "secretpassword"
  API_KEY: "apikey123"
```

## 🔧 常用运维命令

### 1. 集群管理

```bash
# 查看节点
kubectl get nodes -o wide

# 查看组件状态
kubectl get componentstatuses

# 查看集群信息
kubectl cluster-info

# 查看 API 资源
kubectl api-resources
```

### 2. 应用管理

```bash
# 部署应用
kubectl apply -f deployment.yaml

# 查看部署
kubectl get deployments

# 扩缩容
kubectl scale deployment myapp --replicas=5

# 滚动更新
kubectl set image deployment/myapp myapp=myapp:2.0

# 回滚
kubectl rollout undo deployment/myapp

# 查看历史
kubectl rollout history deployment/myapp
```

### 3. 故障排查

```bash
# 查看 Pod 状态
kubectl get pods -o wide

# 查看 Pod 详情
kubectl describe pod myapp-pod

# 查看日志
kubectl logs myapp-pod
kubectl logs -f myapp-pod  # 跟随
kubectl logs myapp-pod -c container-name  # 指定容器

# 进入容器
kubectl exec -it myapp-pod -- /bin/sh

# 端口转发
kubectl port-forward myapp-pod 8080:80

# 查看事件
kubectl get events --sort-by='.lastTimestamp'
```

### 4. 资源管理

```bash
# 查看资源使用
kubectl top nodes
kubectl top pods

# 资源配额
kubectl describe quota -n namespace

# 限制范围
kubectl describe limitrange -n namespace
```

## 📊 监控告警

### 1. Metrics Server

```bash
# 安装 Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 查看资源使用
kubectl top nodes
kubectl top pods
```

### 2. Prometheus 监控

```yaml
# ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
```

### 3. 告警规则

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
spec:
  groups:
  - name: myapp
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
```

## 🔒 安全加固

### 1. RBAC 配置

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 2. Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
```

## 🔧 故障排查指南

### Pod 无法启动

```bash
# 查看 Pod 状态
kubectl describe pod <pod-name>

# 常见问题：
# - ImagePullBackOff: 镜像拉取失败
# - CrashLoopBackOff: 容器启动失败
# - Pending: 资源不足或调度失败
# - ContainerCreating: 存储或网络问题
```

### 节点异常

```bash
# 查看节点状态
kubectl describe node <node-name>

# 查看 kubelet 日志
journalctl -u kubelet -f

# 驱逐节点
kubectl drain <node-name> --ignore-daemonsets --delete-local-data

# 恢复节点
kubectl uncordon <node-name>
```

## 📚 参考资料

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 权威指南](https://github.com/kubernetes/kubernetes)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
