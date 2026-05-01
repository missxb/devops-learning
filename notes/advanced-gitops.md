# GitOps 实战指南

## 📋 概述

GitOps 是现代化的运维方式：**Git 是唯一真理源**。配置、部署、变更都通过 Git 管理，自动化同步到生产环境。

---

## 🏗️ GitOps 核心原则

### 四大原则

```
┌─────────────────────────────────────────────────────────────┐
│                    GitOps 四原则                             │
│                                                             │
│  1. 声明式                                                  │
│     系统描述声明式 (YAML)                                   │
│     Git 存储完整状态                                        │
│                                                             │
│  2. 版本化                                                  │
│     所有变更通过 Git                                        │
│     有完整历史记录                                          │
│     可审计可回滚                                            │
│                                                             │
│  3. 自动化                                                  │
│     Git 变更自动同步                                        │
│     无需手动操作                                            │
│     自动验证                                                │
│                                                             │
│  4. 持续同步                                                │
│     持续监控状态差异                                        │
│     自动拉取对齐                                            │
│     状态自愈合                                              │
└─────────────────────────────────────────────────────────────┘
```

### GitOps vs 传统 CI/CD

```
┌─────────────────────────────────────────────────────────────┐
│                GitOps vs 传统 CI/CD                          │
│                                                             │
│  传统 CI/CD:                                                │
│  Git → CI → Push 到生产                                    │
│  ├── CI 主动推送                                            │
│  ├── 生产环境被动接收                                       │
│  ├── 需要生产环境凭证                                       │
│  └── 配置分散在 CI 系统                                     │
│                                                             │
│  GitOps:                                                    │
│  Git → Controller → 拉取并同步                             │
│  ├── Controller 在集群内                                    │
│  ├── 集群主动拉取                                           │
│  ├── 只需 Git 凭证                                          │
│  └── 配置集中在 Git                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 ArgoCD 部署实战

### ArgoCD 安装

```bash
# 1. 安装 ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. 获取初始密码
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 3. 访问 UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 4. 登录 CLI
argocd login localhost:8080
```

### ArgoCD 配置

```yaml
# argocd-install.yaml

apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec:
  # Git 仓库配置
  repository:
    url: https://github.com/missxb/devops-learning
    type: git
  
  # 同步策略
  syncPolicy:
    automated:
      prune: true           # 自动删除不在 Git 的资源
      selfHeal: true        # 自动修复漂移
    syncOptions:
    - CreateNamespace=true
  
  # 监控配置
  server:
    extraArgs:
    - --insecure  # 开发环境禁用 TLS
  
  # HA 配置
  ha:
    enabled: true
```

---

## 📁 GitOps 仓库结构

### 单仓库模式

```
┌─────────────────────────────────────────────────────────────┐
│                   单仓库结构                                 │
│                                                             │
│  gitops-repo/                                               │
│  ├── apps/                   # 应用配置                     │
│  │   ├── frontend/                                          │
│  │   │   ├── base/          # 基础配置                     │
│  │   │   │   ├── deployment.yaml                            │
│  │   │   │   ├── service.yaml                               │
│  │   │   │   └── configmap.yaml                             │
│  │   │   └── overlays/      # 环境差异                     │
│  │   │       ├── dev/                                       │
│  │   │       │   └── kustomization.yaml                     │
│  │   │       └── prod/                                      │
│  │   │           └── kustomization.yaml                     │
│  │   └── backend/                                           │
│  │       ├── base/                                          │
│  │       └── overlays/                                      │
│  │                                                          │
│  ├── infrastructure/        # 基础设施                      │
│  │   ├── monitoring/                                        │
│  │   ├── networking/                                        │
│  │   └── security/                                          │
│  │                                                          │
│  └── clusters/               # 集群配置                      │
│  │   ├── dev/                                               │
│  │   │   └── argocd-app.yaml                                │
│  │   └── prod/                                              │
│  │       └── argocd-app.yaml                                │
│  │                                                          │
│  └── argocd/                 # ArgoCD 自身配置               │
│      ├── applications/                                      │
│      ├── projects/                                          │
│      └── configs/                                           │
└─────────────────────────────────────────────────────────────┘
```

### ArgoCD Application 配置

```yaml
# applications/frontend.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/missxb/devops-learning
    targetRevision: HEAD
    path: apps/frontend/overlays/prod
    
    # Kustomize 配置
    kustomize:
      namePrefix: prod-
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  # 同步策略
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    
    # 每分钟检查一次
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # 健康检查
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas  # HPA 控制副本数，忽略差异
```

---

## 🔄 GitOps 工作流

### 应用发布流程

```
┌─────────────────────────────────────────────────────────────┐
│                   GitOps 发布流程                            │
│                                                             │
│  开发者                                                     │
│  ├── 编写代码                                               │
│  ├── 推送到代码仓库                                         │
│  └── CI 构建镜像                                           │
│                                                             │
│  CI 系统                                                    │
│  ├── 构建镜像                                               │
│  ├── 推送镜像仓库                                           │
│  └── 更新 GitOps 仓库镜像版本                               │
│                                                             │
│  GitOps 仓库                                                │
│  ├── 接收更新                                               │
│  ├── 自动触发 ArgoCD                                       │
│  └── 开始同步                                               │
│                                                             │
│  ArgoCD                                                     │
│  ├── 检测 Git 变更                                         │
│  ├── 拉取新配置                                             │
│  ├── 生成 K8s 资源                                         │
│  └── 应用到集群                                             │
│                                                             │
│  K8s 集群                                                   │
│  ├── 创建/更新资源                                          │
│  ├── 启动新 Pod                                            │
│  └── ArgoCD 检查健康状态                                   │
│                                                             │
│  监控系统                                                   │
│  ├── 监控部署状态                                           │
│  ├── 验证功能正常                                           │
│  └── 发送通知                                               │
└─────────────────────────────────────────────────────────────┘
```

### CI 集成脚本

```yaml
# GitLab CI 示例

stages:
- build
- update-gitops

build-image:
  stage: build
  script:
  - docker build -t registry.example.com/myapp:$CI_COMMIT_SHA .
  - docker push registry.example.com/myapp:$CI_COMMIT_SHA

update-gitops:
  stage: update-gitops
  script:
  # 更新 GitOps 仓库的镜像版本
  - git clone https://gitlab.example.com/gitops/config.git
  - cd config
  - yq e ".spec.template.spec.containers[0].image = \"registry.example.com/myapp:$CI_COMMIT_SHA\"" -i apps/frontend/base/deployment.yaml
  - git config user.email "ci@example.com"
  - git config user.name "CI Pipeline"
  - git commit -am "Update frontend to $CI_COMMIT_SHA"
  - git push origin main
  only:
  - main
```

---

## 🛡️ GitOps 安全

### Git 凭证管理

```yaml
# ArgoCD Git 凭证

apiVersion: v1
kind: Secret
metadata:
  name: repo-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  url: https://github.com/missxb/devops-learning
  username: missxb
  password: ${GITHUB_TOKEN}
  type: git
```

### RBAC 配置

```yaml
# argocd-rbac.yaml

apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
spec:
  rbac:
    policy.default: role:readonly  # 默认只读
    
    # 自定义角色
    policy.csv: |
      p, role:admin, applications, *, */*, allow
      p, role:developer, applications, get, */*, allow
      p, role:developer, applications, sync, dev/*, allow
      
    # 用户映射
    scopes: [groups]
```

---

## 📋 GitOps Checklist

```markdown
# GitOps 实施 Checklist

## 准备阶段
- [ ] 选择工具 (ArgoCD/Flux)
- [ ] 设计仓库结构
- [ ] 制定分支策略
- [ ] 准备 Git 凭证

## 配置阶段
- [ ] 安装 ArgoCD
- [ ] 配置 Git 仓库
- [ ] 创建 Application
- [ ] 配置同步策略

## 安全阶段
- [ ] RBAC 配置
- [ ] Git 凭证加密
- [ ] 审计日志
- [ ] 回滚策略

## 监控阶段
- [ ] 同步状态监控
- [ ] 健康检查
- [ ] 异常告警
- [ ] 日志记录

## 文档阶段
- [ ] 编写操作手册
- [ ] 记录发布流程
- [ ] 故障处理预案
- [ ] 团队培训
```

---

*GitOps 实战指南 | Git 是唯一真理源*