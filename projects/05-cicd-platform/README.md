# 企业实战项目五：CI/CD 自动化平台建设

## 📋 项目背景

**客户:** 某电商科技公司  
**现状问题:** 
- 发布流程手动，耗时 2-3 小时
- 部署错误频发
- 无法回滚
- 缺乏环境一致性  

**建设目标:**
- 实现一键发布
- 缩短发布时间到 15 分钟
- 支持蓝绿发布/金丝雀发布
- 自动回滚机制

---

## 🏗️ CI/CD 架构设计

### 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                   CI/CD 流水线架构                            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                  代码仓库 (GitLab)                     │  │
│  │              gitlab.example.com                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│                         ▼                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                持续集成 (CI)                           │  │
│  │                                                        │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │   Pipeline Stage: Build                        │  │  │
│  │  │   ├── Code Compile                             │  │  │
│  │  │   ├── Unit Test                                │  │  │
│  │  │   └── Code Quality (SonarQube)                 │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │                        │                              │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │   Pipeline Stage: Containerize                 │  │  │
│  │  │   ├── Docker Build                             │  │  │
│  │  │   ├── Vulnerability Scan                       │  │  │
│  │  │   └── Push to Registry                         │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│                         ▼                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                持续交付 (CD)                           │  │
│  │                                                        │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │   ArgoCD GitOps                                 │  │  │
│  │  │   - Config Git Repo                             │  │  │
│  │  │   - Auto Sync to K8s                            │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │                        │                              │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │   Deployment Strategy                          │  │  │
│  │  │   ├── Blue-Green                               │  │  │
│  │  │   ├── Canary                                   │  │  │
│  │  │   └── Rolling                                  │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                   │
│                         ▼                                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Kubernetes Cluster                       │  │
│  │              production / staging                    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 实施步骤

### 第一阶段：GitLab CI 配置 (3 天)

#### 1.1 GitLab Runner 安装

```bash
#!/bin/bash
# install-gitlab-runner.sh

# 安装 GitLab Runner
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
yum install gitlab-runner -y

# 注册 Runner
gitlab-runner register \
  --non-interactive \
  --url https://gitlab.example.com/ \
  --registration-token "your-registration-token" \
  --executor kubernetes \
  --kubernetes-images "node:alpine" \
  --kubernetes-namespace ci-cd \
  --kubernetes-service-account runner-sa \
  --description "k8s-runner-01"
```

#### 1.2 CI Pipeline 配置

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test
  - scan
  - deploy

variables:
  DOCKER_REGISTRY: registry.example.com
  APP_NAME: ecommerce-api
  K8S_NAMESPACE: production

# Build Stage
build:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  script:
    - echo "Building $APP_NAME..."
    - docker build -t $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA .
    - docker push $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
  artifacts:
    reports:
      dockerfile: Dockerfile
      sast: sonar-report.json

# Test Stage
test:
  stage: test
  image: node:18-alpine
  script:
    - npm install
    - npm run lint
    - npm run test:unit -- --coverage
    - npm run test:e2e
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# Security Scan Stage
security_scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image --severity HIGH,CRITICAL --format sarif --output trivy-results.sarif $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
    - cat trivy-results.sarif
  dependencies: []
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# Deploy to Staging
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/api api=$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA -n staging
    - kubectl rollout status deployment/api -n staging
  environment:
    name: staging
    url: https://api-staging.example.com
  only:
    - develop

# Deploy to Production
deploy-prod:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    # 蓝绿部署 - 先扩容新版本
    - kubectl scale deployment api --replicas=3 -n production --record
    # 等待健康检查
    - sleep 60
    # 流量切换
    - kubectl patch service api-pool -p '{"spec":{"selector":{"version":"v2"}}}' -n production
    # 监控验证
    - curl -f https://api.example.com/health
    # 缩容旧版本
    - kubectl scale deployment api-old --replicas=0 -n production
  environment:
    name: production
    url: https://api.example.com
  when: manual
  only:
    - main
```

---

### 第二阶段：ArgoCD GitOps (5 天)

#### 2.1 ArgoCD 安装配置

```yaml
# argocd-install.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-applicationset
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/missxb/devops-learning.git
      revision: HEAD
      directories:
      - path: projects/*/applications/*
        exclude: false
  template:
    metadata:
      name: '{{path.basename}}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/missxb/devops-learning.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename | replace "-" "_"} }'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

#### 2.2 应用清单配置

```yaml
# applications/api-production.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-production
  namespace: argocd
  labels:
    team: backend
    env: production
spec:
  project: default
  
  source:
    repoURL: https://github.com/missxb/devops-learning.git
    targetRevision: HEAD
    path: deployments/api
      
  destination:
    server: https://kubernetes.default.svc
    namespace: production
    
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - Validate=true
    - PruneLast=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
    
    # 滚动更新策略
    rollingSync:
      maxParallelUpdates: 2
      updatePodsByDeleting: false
      
  # 健康检查
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
```

---

### 第三阶段：蓝绿/金丝雀发布 (7 天)

#### 3.1 蓝绿部署方案

```yaml
# blue-green-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-blue  # 当前生产环境
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: api
      version: blue
  template:
    metadata:
      labels:
        app: api
        version: blue
    spec:
      containers:
      - name: api
        image: registry.example.com/api:v1.0.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-green  # 新版本待部署
  namespace: production
spec:
  replicas: 0  # 初始为 0
  selector:
    matchLabels:
      app: api
      version: green
  template:
    metadata:
      labels:
        app: api
        version: green
    spec:
      containers:
      - name: api
        image: registry.example.com/api:v1.1.0  # 新版本
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
# Service 路由
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: production
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: api
    version: blue  # 指向蓝色版本
```

#### 3.2 金丝雀发布脚本

```python
#!/usr/bin/env python3
"""CanaryDeployment.py - 金丝雀发布工具"""

import requests
import time
from kubernetes import client, config
import logging

class CanaryDeployer:
    def __init__(self, namespace='production', canary_percentage=10):
        self.namespace = namespace
        self.canary_percentage = canary_percentage
        
    def scale_canary(self, percentage):
        """按比例缩放金丝雀 Pod 数量"""
        v1 = client.AppsV1Api()
        
        # 获取当前副本数
        deployment = v1.read_namespaced_deployment(
            'api-canary',
            self.namespace
        )
        
        total_replicas = deployment.spec.replicas
        canary_count = int(total_replicas * percentage / 100)
        
        # 调整金丝雀副本数
        deployment.spec.replicas = canary_count
        v1.patch_namespaced_deployment(
            'api-canary',
            self.namespace,
            deployment
        )
        print(f"✅ Canary scaled to {canary_count} pods ({percentage}%)")
        
    def health_check(self):
        """金丝雀健康检查"""
        try:
            response = requests.get(
                'http://api-canary.healthcheck.local/health',
                timeout=5
            )
            return response.status_code == 200
        except:
            return False
            
    def rollback(self):
        """金丝雀回滚"""
        logging.warning("🚨 Rollback triggered!")
        self.scale_canary(0)
        # TODO: 恢复主版本
        
    def promote(self):
        """金丝雀提升为主版本"""
        logging.info("🎉 Promoting canary...")
        # 切换到金丝雀版本
        # ...
        
# 使用示例
deployer = CanaryDeployer()

# 阶段 1: 10% 流量到金丝雀
print("Phase 1: 10% traffic to canary")
deployer.scale_canary(10)
time.sleep(300)  # 等待 5 分钟

# 检查健康
if deployer.health_check():
    # 阶段 2: 50% 流量
    print("Phase 2: 50% traffic to canary")
    deployer.scale_canary(50)
else:
    print("❌ Health check failed, triggering rollback")
    deployer.rollback()
```

---

## 📊 发布流程对比

### 手动 vs 自动化发布

```
┌─────────────────────────────────────────────────────────────┐
│               发布效率对比                                     │
│                                                             │
│  指标              │ 手动发布    │ 自动化发布         │
│  ──────────────────┼───────────┼────────────────     │
│  发布准备时间      │ 30 分钟     │ 2 分钟             │
│  实际部署时间      │ 90 分钟     │ 5 分钟             │
│  验证时间          │ 30 分钟     │ 2 分钟             │
│  总耗时           │ 150 分钟    │ 9 分钟             │
│  故障率           │ 15%         │ 2%                 │
│  回滚时间         │ 45 分钟     │ 30 秒               │
│                                                             │
│  【效率提升】                                                │
│  ├── 速度提升：94%                                          │
│  ├── 错误减少：87%                                          │
│  └── 回滚速度：180 倍                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 💰 ROI 分析

```
┌─────────────────────────────────────────────────────────────┐
│                   CI/CD 项目投资回报                           │
│                                                             │
│  投入成本                                                  │
│  ├── 人力成本 (2 周开发 +1 周培训): ¥10 万                      │
│  ├── 工具采购 (SonarQube/GitHub Actions): ¥5 万               │
│  └── 总计：¥15 万                                           │
│                                                             │
│  年度收益                                                   │
│  ├── 节省开发时间 (每天少等 1.5h × 10 人 × 250 天): ¥75 万        │
│  ├── 减少生产事故 (每月减少 2 次 × ¥50 万损失): ¥120 万           │
│  ├── 提高发布频率 (从 2 次/周到 10 次/天): 价值无法估量           │
│  └── 总计：¥195 万/年                                      │
│                                                             │
│  ROI                                                       │
│  ├── 年度收益：¥195 万                                      │
│  ├── 投入成本：¥15 万                                       │
│  └── ROI: 13 倍                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 📋 发布 Checklist

```markdown
# 发布前检查清单

## 代码质量
- [ ] 单元测试覆盖率 > 80%
- [ ] SonarQube 无严重问题
- [ ] 代码审查通过
- [ ] 性能测试通过

## 安全扫描
- [ ] 镜像漏洞扫描通过
- [ ] 依赖库 CVE 检查通过
- [ ] 敏感信息无泄露

## 基础设施
- [ ] 数据库变更已执行
- [ ] 配置文件已同步
- [ ] 负载均衡已配置
- [ ] 告警规则已启用

## 应急预案
- [ ] 回滚方案已验证
- [ ] 紧急联系人已通知
- [ ] 监控 Dashboard 已就绪
```

---

*CI/CD 自动化平台建设 | ClawsOps ⚙️*