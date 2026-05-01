# 云原生安全最佳实践

## 📋 概述

云原生环境的安全挑战与传统环境完全不同。**容器、K8s、微服务带来新的安全风险**，需要从设计阶段就考虑安全。

---

## 🏗️ 云原生安全架构

### 安全分层模型

```
┌─────────────────────────────────────────────────────────────┐
│                   云原生安全四层模型                          │
│                                                             │
│         ┌─────────────────────────────────────┐            │
│         │    Layer 4: 工作负载安全             │            │
│         │  应用代码/依赖/运行时安全             │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │    Layer 3: 容器安全                 │            │
│         │  镜像安全/运行时隔离/资源限制         │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │    Layer 2: 集群安全                 │            │
│         │  K8s 配置/RBAC/网络策略              │            │
│         └─────────────────────────────────────┘            │
│                        │                                    │
│         ┌─────────────────────────────────────┐            │
│         │    Layer 1: 主机安全                 │            │
│         │  OS 加固/SSH/防火墙                  │            │
│         └─────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🖼️ 镜像安全

### 镜像安全检查

```yaml
# Trivy 镜像扫描配置
apiVersion: batch/v1
kind: CronJob
metadata:
  name: image-scan
spec:
  schedule: "0 2 * * *"  # 每天 2:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: trivy
            image: aquasec/trivy:latest
            command:
            - trivy
            - image
            - --severity
            - HIGH,CRITICAL
            - --ignore-unfixed
            - --format
            - json
            - --output
            - /tmp/report.json
            - myapp:latest
```

### 镜像安全最佳实践

```dockerfile
# 安全 Dockerfile 示例

# 1. 使用最小基础镜像
FROM alpine:3.18  # 比 Ubuntu 小 10 倍

# 2. 指定版本，不用 latest
FROM nginx:1.24-alpine

# 3. 非 root 用户运行
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# 4. 最小权限
RUN chmod 755 /app && chmod 644 /app/config/*

# 5. 健康检查
HEALTHCHECK --interval=30s CMD curl -f http://localhost/health || exit 1

# 6. 限制资源
# 在 K8s 中配置，不在 Dockerfile

# 7. 多阶段构建 (减少攻击面)
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
CMD ["node", "server.js"]
```

---

## 🛡️ 容器运行时安全

### Pod 安全策略

```yaml
# PodSecurityPolicy (K8s 1.25+ 用 Pod Security Standards)

apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false              # 禁止特权模式
  runAsUser:
    rule: MustRunAsNonRoot       # 必须非 root
  seLinux:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'persistentVolumeClaim'
  hostNetwork: false             # 禁止 host 网络
  hostIPC: false                 # 禁止 host IPC
  hostPID: false                 # 禁止 host PID
  readOnlyRootFilesystem: true   # 只读根文件系统
  requiredDropCapabilities:
  - ALL                          # 禁止所有 Linux capabilities
```

### 安全 Pod 配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  # 安全上下文
  securityContext:
    runAsNonRoot: true            # 非 root 运行
    runAsUser: 1000
    readOnlyRootFilesystem: true  # 只读文件系统
    seccompProfile:
      type: RuntimeDefault        # seccomp 默认
    capabilities:
      drop:
      - ALL                       # 禁止所有 capabilities
  
  containers:
  - name: app
    image: myapp:1.0
    
    # 容器级安全
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
    
    # 资源限制 (防止 DoS)
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi
    
    # 只读挂载
    volumeMounts:
    - name: config
      mountPath: /app/config
      readOnly: true
  
  volumes:
  - name: config
    configMap:
      name: app-config
```

---

## 🔐 RBAC 配置

### RBAC 最佳实践

```yaml
# 1. 服务账号最小权限

apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
automountServiceAccountToken: false  # 不自动挂载 token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
# 只允许必要的权限
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  resourceNames: ["app-config", "app-secret"]  # 只允许特定资源
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: app-role
```

### RBAC 权限矩阵

```
┌─────────────────────────────────────────────────────────────┐
│                    RBAC 权限矩阵                             │
│                                                             │
│  角色        │ Namespace │ 资源       │ 动作               │
│  ───────────┼──────────┼────────────┼─────────────────    │
│  开发者      │ dev       │ pods       │ get,list           │
│              │           │ logs       │ get                │
│              │           │ deploy     │ create,update      │
│  运维        │ prod      │ pods       │ get,list,delete    │
│              │           │ nodes      │ get,list           │
│              │           │ secrets    │ get (受限)         │
│  审计        │ all       │ all        │ get,list           │
│  管理员      │ all       │ all        │ *                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🌐 网络安全策略

### NetworkPolicy 配置

```yaml
# 网络隔离策略

# 1. 默认拒绝所有入站
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # 所有 Pod
  policyTypes:
  - Ingress                # 没有 ingress 规则 = 拒绝所有

---
# 2. 允许特定入站
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    # 只允许 Ingress Controller
    - namespaceSelector:
        matchLabels:
          name: ingress-system
    # 只允许前端调用后端
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 3. 限制出站
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  # 只允许访问数据库
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
  
  # 只允许 DNS
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## 🔑 密钥管理

### Vault 集成

```yaml
# Vault Agent 注入

apiVersion: v1
kind: Pod
metadata:
  name: app-with-vault
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-role"
    vault.hashicorp.com/agent-inject-secret-db: "secret/data/database"
    vault.hashicorp.com/agent-inject-template-db: |
      {{- with secret "secret/data/database" -}}
      username={{ .Data.data.username }}
      password={{ .Data.data.password }}
      {{- end }}
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: vault-secrets
      mountPath: /vault/secrets
      readOnly: true
```

### Secret 加密

```yaml
# K8s Secret 加密配置

apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <BASE64-ENCODED-KEY>
  - identity: {}  # 兜底
```

---

## 📋 安全审计

### 审计日志配置

```yaml
# kube-apiserver 审计配置

apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 记录所有 Secret 访问
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  
# 记录所有写操作
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods", "services", "deployments"]
  verbs: ["create", "update", "delete"]
  
# 不记录读操作 (减少日志量)
- level: None
  resources:
  - group: ""
    resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
```

---

## 🚨 安全漏洞处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                    漏洞处理流程                              │
│                                                             │
│  发现漏洞                                                   │
│  ├── 自动扫描 (Trivy/Clair)                                 │
│  ├── CVE 通知                                               │
│  └── 安全审计                                               │
│                                                             │
│  评估风险                                                   │
│  ├── CVSS 分级 (Critical/High/Medium/Low)                  │
│  ├── 影响范围                                               │
│  └── 利用难度                                               │
│                                                             │
│  制定方案                                                   │
│  ├── 升级依赖                                               │
│  ├── 替换组件                                               │
│  ├── 临时缓解                                               │
│  └── 风险接受                                               │
│                                                             │
│  实施修复                                                   │
│  ├── 紧急修复 (< 24 小时)                                   │
│  ├── 快速修复 (< 7 天)                                      │
│  ├── 常规修复 (< 30 天)                                     │
│  └── 接受风险                                               │
│                                                             │
│  验证效果                                                   │
│  ├── 重新扫描                                               │
│  ├── 功能验证                                               │
│  └── 文档记录                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 安全 Checklist

```markdown
# 云原生安全 Checklist

## 镜像安全
- [ ] 基础镜像最小化
- [ ] 定期扫描漏洞
- [ ] 不使用 latest 标签
- [ ] 非 root 用户运行

## 容器安全
- [ ] 禁止特权模式
- [ ] 只读文件系统
- [ ] 资源限制配置
- [ ] seccomp 配置

## 集群安全
- [ ] RBAC 最小权限
- [ ] NetworkPolicy 配置
- [ ] Secret 加密存储
- [ ] 审计日志开启

## 网络安全
- [ ] 默认拒绝入站
- [ ] 限制出站流量
- [ ] Service Mesh mTLS
- [ ] Ingress TLS

## 运维安全
- [ ] SSH 禁止 root
- [ ] 定期更新系统
- [ ] 安全审计定期
- [ ] 应急预案准备
```

---

*云原生安全最佳实践 | 安全左移，设计即考虑安全*