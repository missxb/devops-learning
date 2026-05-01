# CI/CD 流水线实战

## 📋 概述

CI/CD (持续集成/持续部署) 是 DevOps 的核心实践，通过自动化流程实现快速、可靠的软件交付。

## 🎯 学习目标

- 理解 CI/CD 核心概念
- 掌握主流 CI/CD 工具
- 设计高效流水线
- 实施部署策略
- 构建安全合规流程

## 🏗️ CI/CD 流程架构

```
┌─────────────────────────────────────────────────────────────┐
│                      CI/CD 流水线                            │
│                                                             │
│  代码提交 ──► 构建 ──► 测试 ──► 制品 ──► 部署 ──► 验证      │
│    │          │        │        │        │        │         │
│    ▼          ▼        ▼        ▼        ▼        ▼         │
│  ┌────┐    ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐     │
│  │Git │    │Docker│ │单元│   │镜像│   │K8s │   │健康│     │
│  │    │    │Build │ │测试│   │仓库│   │部署│   │检查│     │
│  └────┘    └────┘   └────┘   └────┘   └────┘   └────┘     │
│              │        │               │        │            │
│              ▼        ▼               ▼        ▼            │
│         ┌────────┐ ┌────┐        ┌────────┐ ┌────┐        │
│         │代码质量│ │集成│        │灰度发布│ │监控│        │
│         │扫描    │ │测试│        │蓝绿部署│ │告警│        │
│         └────────┘ └────┘        └────────┘ └────┘        │
└─────────────────────────────────────────────────────────────┘
```

## 📦 主流 CI/CD 工具对比

| 工具 | 类型 | 优势 | 适用场景 |
|------|------|------|----------|
| **Jenkins** | 自托管 | 插件丰富、灵活 | 复杂流水线 |
| **GitLab CI** | 自托管/SaaS | 与 GitLab 深度集成 | GitLab 用户 |
| **GitHub Actions** | SaaS | 与 GitHub 集成、免费额度 | 开源项目 |
| **ArgoCD** | GitOps | Kubernetes 原生、声明式 | K8s 部署 |
| **Tekton** | Kubernetes | 云原生、可扩展 | 企业级 K8s |

## 🔧 GitLab CI 实战

### 1. 基础配置

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_IMAGE: registry.example.com/myapp
  DOCKER_TLS_CERTDIR: "/certs"

# 构建阶段
build:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
  only:
    - branches

# 测试阶段
test:
  stage: test
  image: python:3.9
  script:
    - pip install -r requirements.txt
    - pytest tests/ --cov=src --cov-report=xml
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
  only:
    - branches

# 部署阶段
deploy:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE:$CI_COMMIT_SHA -n production
  environment:
    name: production
    url: https://myapp.example.com
  only:
    - main
  when: manual
```

### 2. 多环境部署

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy:dev
  - deploy:staging
  - deploy:prod

.deploy_template: &deploy_template
  image: bitnami/kubectl:latest
  before_script:
    - kubectl config use-context $K8S_CONTEXT
  script:
    - kubectl set image deployment/$APP_NAME $APP_NAME=$DOCKER_IMAGE:$CI_COMMIT_SHA -n $NAMESPACE
  environment:
    name: $ENV_NAME
    url: $ENV_URL

deploy_dev:
  <<: *deploy_template
  stage: deploy:dev
  variables:
    ENV_NAME: development
    NAMESPACE: dev
    APP_NAME: myapp
    ENV_URL: https://dev.myapp.example.com
  only:
    - develop

deploy_staging:
  <<: *deploy_template
  stage: deploy:staging
  variables:
    ENV_NAME: staging
    NAMESPACE: staging
    APP_NAME: myapp
    ENV_URL: https://staging.myapp.example.com
  only:
    - develop
  when: manual

deploy_prod:
  <<: *deploy_template
  stage: deploy:prod
  variables:
    ENV_NAME: production
    NAMESPACE: production
    APP_NAME: myapp
    ENV_URL: https://myapp.example.com
  only:
    - main
  when: manual
  allow_failure: false
```

### 3. 代码质量检查

```yaml
# .gitlab-ci.yml
code_quality:
  stage: test
  image: sonarsource/sonar-scanner-cli:latest
  script:
    - sonar-scanner
      -Dsonar.projectKey=$CI_PROJECT_NAME
      -Dsonar.sources=.
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  only:
    - branches

security_scan:
  stage: test
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 0 --format template --template "@contrib/html.tpl" -o trivy-report.html $DOCKER_IMAGE:$CI_COMMIT_SHA
    - trivy image --exit-code 2 --severity HIGH,CRITICAL $DOCKER_IMAGE:$CI_COMMIT_SHA
  artifacts:
    reports:
      container_scanning: trivy-report.json
  only:
    - branches
```

## 🐙 GitHub Actions 实战

### 1. 基础工作流

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ -v --cov=src

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
          fail_ci_if_error: true

  deploy:
    runs-on: ubuntu-latest
    needs: [build, test]
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v3

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG }}

      - name: Deploy to K8s
        run: |
          kubectl set image deployment/myapp myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl rollout status deployment/myapp
```

### 2. 矩阵构建

```yaml
# .github/workflows/matrix-build.yml
name: Matrix Build

on: push

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.8', '3.9', '3.10']
        os: [ubuntu-latest, macos-latest]
      fail-fast: false

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}

      - name: Test
        run: pytest tests/ -v
```

### 3. 定时任务

```yaml
# .github/workflows/scheduled.yml
name: Scheduled Tasks

on:
  schedule:
    - cron: '0 2 * * *'  # 每天 2:00 UTC
  workflow_dispatch:

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
      - name: Cleanup old artifacts
        uses: kolpav/purge-artifacts-action@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          expire-in: 7d

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

## 🚀 ArgoCD GitOps

### 1. ArgoCD 部署

```yaml
# argocd-install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  template:
    spec:
      containers:
      - name: argocd-server
        image: quay.io/argoproj/argocd:v2.5.0
        ports:
        - containerPort: 8080
        - containerPort: 8083
```

### 2. Application 定义

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo.git
    targetRevision: HEAD
    path: applications/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 3. 应用集

```yaml
# argocd-appset.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-set
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: dev
            url: https://dev-k8s.example.com
          - cluster: staging
            url: https://staging-k8s.example.com
          - cluster: prod
            url: https://prod-k8s.example.com
  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-repo.git
        targetRevision: HEAD
        path: 'applications/myapp/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## 📊 部署策略

### 1. 滚动更新

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

### 2. 蓝绿部署

```yaml
# 蓝色环境
apiVersion: v1
kind: Service
metadata:
  name: myapp-blue
spec:
  selector:
    app: myapp
    version: blue
  ports:
  - port: 80
    targetPort: 8080

# 绿色环境
apiVersion: v1
kind: Service
metadata:
  name: myapp-green
spec:
  selector:
    app: myapp
    version: green
  ports:
  - port: 80
    targetPort: 8080

# 流量切换 (修改 Ingress)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-blue  # 切换为 myapp-green
            port:
              number: 80
```

### 3. 金丝雀发布

```yaml
# Istio VirtualService
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 90
    - destination:
        host: myapp
        subset: canary
      weight: 10
```

## 🔒 安全合规

### 1. 密钥管理

```yaml
# SealedSecret
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secret
  namespace: production
spec:
  encryptedData:
    db-password: AgBy8i...
  template:
    metadata:
      name: app-secret
    type: Opaque
```

### 2. 镜像签名

```yaml
# Cosign 验证
apiVersion: v1
kind: Pod
metadata:
  name: verified-pod
  annotations:
    cosign.sigstore.dev/signed: "true"
    cosign.sigstore.dev/signature: MEUCIQ...
spec:
  containers:
  - name: app
    image: ghcr.io/org/app@sha256:abc123...
```

### 3. 策略执行

```yaml
# OPA Gatekeeper
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedImages
metadata:
  name: allowed-images
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    sources:
      - "gcr.io/my-project/*"
      - "ghcr.io/my-org/*"
```

## 📚 参考资料

- [GitLab CI 文档](https://docs.gitlab.com/ee/ci/)
- [GitHub Actions 文档](https://docs.github.com/actions)
- [ArgoCD 文档](https://argo-cd.readthedocs.io/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
