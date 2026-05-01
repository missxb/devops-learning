# 项目十：Kubernetes 云架构平台

## 📋 项目概述

基于 Kubernetes 构建企业级云架构平台，整合微服务、服务网格、CI/CD、监控告警等完整生态体系，实现云原生应用的全生命周期管理。

## 🎯 学习目标

- 掌握云原生架构设计原则
- 部署服务网格 Istio
- 配置 GitOps 工作流 (ArgoCD)
- 实施可观测性 (Logging/Tracing/Metrics)
- 构建多租户平台

## 🏗️ 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         用户层                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ 开发者   │  │ 运维人员 │  │ 产品经理 │  │ 最终用户 │       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
└───────┼─────────────┼─────────────┼─────────────┼──────────────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      接入层 (Ingress)                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   Nginx         │  │   Traefik       │  │   Istio         │ │
│  │   Ingress       │  │   Ingress       │  │   Gateway       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   应用服务层     │  │   中间件层       │  │   数据层         │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ 微服务    │  │  │  │ 消息队列  │  │  │  │  MySQL    │  │
│  │ (Deploy)  │  │  │  │ (Kafka)   │  │  │  │  (RDS)    │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ 前端应用  │  │  │  │ 缓存      │  │  │  │  MongoDB  │  │
│  │ (React)   │  │  │  │ (Redis)   │  │  │  │  集群     │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   DevOps 平台    │  │   可观测性       │  │   安全合规       │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ GitLab    │  │  │  │ Prometheus│  │  │  │ RBAC      │  │
│  │ Jenkins   │  │  │  │ Grafana   │  │  │  │ Network   │  │
│  │ ArgoCD    │  │  │  │ ELK Stack │  │  │  │ Policy    │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

## 📦 核心组件清单

| 分类 | 组件 | 版本 | 用途 |
|------|------|------|------|
| 服务网格 | Istio | 1.12 | 流量管理、安全 |
| API 网关 | Kong/APISIX | 2.x | API 管理 |
| GitOps | ArgoCD | 2.3 | 持续部署 |
| CI/CD | Jenkins/GitLab CI | - | 持续集成 |
| 监控 | Prometheus | 2.30 | 指标采集 |
| 可视化 | Grafana | 8.x | 数据展示 |
| 日志 | EFK/ELK | 7.x | 日志收集 |
| 追踪 | Jaeger | 1.26 | 链路追踪 |
| 安全 | OPA/Gatekeeper | 3.x | 策略管理 |

## 🔧 部署流程

### 1. 部署 Istio 服务网格

```bash
# 下载 istioctl
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/bin:$PATH

# 安装 Istio (demo 配置)
istioctl install --set profile=demo -y

# 验证安装
kubectl get pods -n istio-system

# 启用 sidecar 自动注入
kubectl label namespace default istio-injection=enabled
```

### 2. 部署 ArgoCD GitOps

```bash
# 创建命名空间
kubectl create namespace argocd

# 安装 ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 查看初始密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 访问 UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 配置 Git 仓库
argocd repo add https://github.com/your-org/gitops-repo.git \
  --username ${GIT_USER} --password ${GIT_TOKEN}
```

### 3. 部署 Prometheus Stack

```bash
# 添加 Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 安装 kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set prometheus.service.type=LoadBalancer \
  --set grafana.service.type=LoadBalancer \
  --set alertmanager.service.type=LoadBalancer

# 访问 Grafana
kubectl get svc -n monitoring | grep grafana
```

### 4. 部署 EFK 日志系统

```yaml
# elasticsearch-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
        resources:
          limits:
            cpu: 1000m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 512Mi
        ports:
        - containerPort: 9200
          name: http
        - containerPort: 9300
          name: transport
        env:
        - name: cluster.name
          value: k8s-logs
        - name: node.name
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: discovery.seed_hosts
          value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
        - name: cluster.initial_master_nodes
          value: "es-cluster-0,es-cluster-1,es-cluster-2"
        - name: ES_JAVA_OPTS
          value: "-Xms512m -Xmx512m"
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 50Gi
```

### 5. 配置多租户

```yaml
# namespace-template.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-a
  labels:
    tenant: tenant-a
  annotations:
    net.beta.kubernetes.io/network-policy: |
      {
        "ingress": {
          "isolation": "DefaultDeny"
        }
      }
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
    pods: "50"
    services: "20"
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### 6. 配置 OPA 策略

```yaml
# gatekeeper-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["app", "owner"]
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDenyPrivileged
metadata:
  name: deny-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

## 📊 监控告警配置

```yaml
# alert-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
spec:
  groups:
  - name: k8s-alerts
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 5 > 0
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has restarted {{ $value }} times in 5 minutes"
    
    - alert: NodeMemoryPressure
      expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Node {{ $labels.instance }} is under memory pressure"
    
    - alert: ServiceDown
      expr: probe_success == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Service {{ $labels.job }} is down"
```

## 📚 参考资料

- [Cloud Native Computing Foundation](https://www.cncf.io/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Istio 官方文档](https://istio.io/docs/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
