# CI/CD 自动化平台 - 完整流水线与 GitOps 配置

## GitLab CI 完整流水线

### 多阶段 Pipeline

```yaml
# .gitlab-ci.yml - 电商订单服务

stages:
  - lint
  - build
  - test
  - security
  - deploy-staging
  - deploy-production

variables:
  DOCKER_REGISTRY: registry.ecommerce.com
  APP_NAME: order-service
  HELM_CHART: charts/order-service
  K8S_NAMESPACE_STAGING: staging
  K8S_NAMESPACE_PROD: production
  DOCKER_TLS_CERTDIR: "/certs"

# ==================== Lint ====================
lint:
  stage: lint
  image: node:18-alpine
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  script:
    - npm ci
    - npm run lint
    - npm run type-check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

# ==================== Build ====================
build:
  stage: build
  image: docker:24.0
  services:
    - docker:24.0-dind
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - docker-cache/
  before_script:
    - echo "$CI_REGISTRY_PASSWORD" | docker login -u "$CI_REGISTRY_USER" --password-stdin $CI_REGISTRY
  script:
    - |
      IMAGE_TAG="$CI_REGISTRY/$APP_NAME:$CI_COMMIT_SHA"
      LATEST_TAG="$CI_REGISTRY/$APP_NAME:latest"
      
      docker build \
        --cache-from "$IMAGE_TAG" \
        --build-arg BUILD_DATE="$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
        --build-arg VCS_REF="$CI_COMMIT_SHORT_SHA" \
        --build-arg VERSION="$CI_COMMIT_TAG" \
        -t "$IMAGE_TAG" \
        -t "$LATEST_TAG" \
        -f Dockerfile.production .
      
      docker push "$IMAGE_TAG"
      docker push "$LATEST_TAG"
      
      echo "IMAGE=$IMAGE_TAG" >> build.env
  artifacts:
    reports:
      dotenv: build.env
    paths:
      - docker-cache/
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ==================== Test ====================
unit-test:
  stage: test
  image: node:18-alpine
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  script:
    - npm ci
    - npm run test:unit -- --coverage --coverageReporters=json
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      junit: reports/junit.xml
    paths:
      - coverage/
    expire_in: 30 days
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main"

e2e-test:
  stage: test
  image: mcr.microsoft.com/playwright:v1.35.0
  needs:
    - deploy-staging
  script:
    - npm ci
    - npx playwright install --with-deps chromium
    - E2E_BASE_URL="https://order-staging.ecommerce.com" \
      npx playwright test --reporter=html,junit
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

# ==================== Security ====================
sast:
  stage: security
  image: docker:24.0
  variables:
    SECURE_ANALYZERS_PREFIX: "$CI_SERVER_FQDN"
  script:
    - echo "Running SAST analysis..."
  artifacts:
    reports:
      sast: gl-sast-report.json
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

dependency-check:
  stage: security
  image: owasp/dependency-check:latest
  script:
    - dependency-check --project "$APP_NAME" \
      --scan . \
      --format JSON \
      --format HTML \
      --out reports/
  artifacts:
    paths:
      - reports/
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

container-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy image --exit-code 0 --severity LOW,MEDIUM --format table -o trivy-low-med.report $IMAGE
    - trivy image --exit-code 1 --severity HIGH,CRITICAL --format table -o trivy-high-crit.report $IMAGE || {
        echo "❌ High/Critical vulnerabilities found!";
        cat trivy-high-crit.report;
        exit 1;
      }
  artifacts:
    paths:
      - trivy-*.report
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ==================== Deploy Staging ====================
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  environment:
    name: staging
    url: https://order-staging.ecommerce.com
  script:
    - |
      # 更新 Helm values
      cat > staging-values.yaml <<EOF
      replicaCount: 2
      image:
        repository: $APP_NAME
        tag: "$CI_COMMIT_SHA"
      resources:
        requests:
          cpu: 500m
          memory: 512Mi
        limits:
          cpu: 1000m
          memory: 1Gi
      EOF
      
      # 使用 kubectl apply (GitOps 模式: ArgoCD 会同步)
      kubectl apply -f - <<YAML
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: order-service
        namespace: $K8S_NAMESPACE_STAGING
        labels:
          app: order-service
          env: staging
      spec:
        replicas: 2
        selector:
          matchLabels:
            app: order-service
        template:
          metadata:
            labels:
              app: order-service
              env: staging
          spec:
            containers:
            - name: order-service
              image: $DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA
              ports:
              - containerPort: 8080
              env:
              - name: NODE_ENV
                value: staging
              - name: DB_HOST
                valueFrom:
                  secretKeyRef:
                    name: order-db-credentials
                    key: host
              readinessProbe:
                httpGet:
                  path: /health
                  port: 8080
                initialDelaySeconds: 15
                periodSeconds: 5
              livenessProbe:
                httpGet:
                  path: /health
                  port: 8080
                initialDelaySeconds: 30
                periodSeconds: 10
      YAML
      
      # 等待滚动更新完成
      kubectl rollout status deployment/order-service -n $K8S_NAMESPACE_STAGING --timeout=300s
      
      echo "✅ Staging deployment complete: $CI_COMMIT_SHA"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ==================== Deploy Production ====================
deploy-production:
  stage: deploy-production
  image: bitnami/kubectl:latest
  environment:
    name: production
    url: https://order.ecommerce.com
  script:
    - |
      PROD_IMAGE="$DOCKER_REGISTRY/$APP_NAME:$CI_COMMIT_SHA"
      
      echo "🚀 Starting production deployment: $PROD_IMAGE"
      
      # ====== 蓝绿部署流程 ======
      
      # Step 1: 部署新版本（绿色）到生产环境，初始副本为 0
      cat > green-deployment.yaml <<YAML
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: order-service-green
        namespace: $K8S_NAMESPACE_PROD
        labels:
          app: order-service
          version: green
          deploy-id: "$CI_PIPELINE_IID"
      spec:
        replicas: 0
        selector:
          matchLabels:
            app: order-service
            version: green
        template:
          metadata:
            labels:
              app: order-service
              version: green
          spec:
            containers:
            - name: order-service
              image: $PROD_IMAGE
              ports:
              - containerPort: 8080
              env:
              - name: NODE_ENV
                value: production
              - name: DB_HOST
                valueFrom:
                  secretKeyRef:
                    name: order-db-credentials
                    key: host
              readinessProbe:
                httpGet:
                  path: /health
                  port: 8080
                initialDelaySeconds: 15
                periodSeconds: 5
              resources:
                requests:
                  cpu: "1"
                  memory: "1Gi"
                limits:
                  cpu: "2"
                  memory: "2Gi"
      YAML
      
      kubectl apply -f green-deployment.yaml
      
      # Step 2: 扩容绿色部署
      echo "Scaling green deployment..."
      kubectl scale deployment order-service-green \
        --replicas=3 -n $K8S_NAMESPACE_PROD
      
      kubectl rollout status deployment/order-service-green \
        -n $K8S_NAMESPACE_PROD --timeout=300s
      
      # Step 3: 健康检查 - 等待就绪
      echo "Running health checks..."
      sleep 30
      
      # Step 4: 切换流量（修改 Service selector）
      echo "Switching traffic to green..."
      kubectl patch service order-service -n $K8S_NAMESPACE_PROD -p \
        '{"spec":{"selector":{"version":"green"}}}'
      
      # Step 5: 监控验证 - 检查错误率
      echo "Monitoring for errors..."
      for i in {1..6}; do
        ERROR_RATE=$(curl -s "http://grafana.monitoring:3000/api/datasources/proxy/1/api/v1/query" \
          --data-urlencode 'query=sum(rate(http_requests_total{service="order-service",status=~"5.."}[5m])) / sum(rate(http_requests_total{service="order-service"}[5m])) * 100' \
          -H "Authorization: Bearer $GRAFANA_TOKEN" | jq '.data.result[0].value[1]')
        
        if [ "${ERROR_RATE:-0}" != "null" ]; then
          echo "  Error rate check $i: ${ERROR_RATE}%"
        fi
        sleep 10
      done
      
      # Step 6: 缩容旧版本（蓝色）
      echo "Scaling down blue deployment..."
      kubectl scale deployment order-service-blue \
        --replicas=0 -n $K8S_NAMESPACE_PROD
      
      # Step 7: 清理旧版本
      echo "Cleaning up..."
      kubectl delete deployment order-service-blue \
        -n $K8S_NAMESPACE_PROD --ignore-not-found
      
      # 重命名绿色为蓝色，下次部署使用新的绿色
      kubectl patch deployment order-service-green -n $K8S_NAMESPACE_PROD -p \
        '{"metadata":{"name":"order-service-blue"}}'
      
      echo "✅ Production deployment complete!"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

# ==================== Post-Deploy ====================
notify:
  stage: deploy-production
  image: curlimages/curl:latest
  script:
    - |
      curl -s -X POST "$SLACK_WEBHOOK" \
        -H "Content-Type: application/json" \
        -d "{
          \"text\": \"🚀 *Order Service Deployed to Production*\\nCommit: $CI_COMMIT_SHORT_SHA\\nAuthor: $CI_COMMIT_AUTHOR\\nPipeline: $CI_PIPELINE_URL\"
        }"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success
```

---

## ArgoCD 应用管理

### 应用清单

```yaml
# argocd-application-order-service.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
  labels:
    app: order-service
    team: commerce
spec:
  project: ecommerce
  
  source:
    repoURL: https://github.com/missxb/devops-learning.git
    targetRevision: HEAD
    path: deployments/order-service/k8s
    
  destination:
    server: https://kubernetes.default.svc
    namespace: production
    
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
    - Validate=true
    
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
    
    # 资源排除
    managedNamespaceMetas:
      limitRange:
        limits:
        - default:
            cpu: 500m
            memory: 512Mi
          defaultRequest:
            cpu: 250m
            memory: 256Mi
          type: Container

---
# Staging 环境应用

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-staging
  namespace: argocd
  labels:
    app: order-service
    env: staging
spec:
  project: ecommerce
  
  source:
    repoURL: https://github.com/missxb/devops-learning.git
    targetRevision: develop
    path: deployments/order-service/k8s
    
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
    
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    
  # Staging 使用不同的 values
  helm:
    parameters:
    - name: replicaCount
      value: "1"
    - name: image.tag
      value: "latest"
```

### Helm Chart 模板

```yaml
# templates/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "order-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "order-service.selectorLabels" . | nindent 6 }}
  strategy:
    {{- toYaml .Values.deploymentStrategy | nindent 4 }}
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
      labels:
        {{- include "order-service.selectorLabels" . | nindent 8 }}
    spec:
      serviceAccountName: {{ include "order-service.serviceAccountName" . }}
      
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 8081
          protocol: TCP
        
        env:
        {{- range $key, $value := .Values.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        
        {{- with .Values.envFrom }}
        envFrom:
        {{- toYaml . | nindent 8 }}
        {{- end }}
        
        readinessProbe:
          httpGet:
            path: /health/readiness
            port: http
          initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          timeoutSeconds: {{ .Values.probes.readiness.timeoutSeconds }}
          failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
        
        livenessProbe:
          httpGet:
            path: /health/liveness
            port: http
          initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
        
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: {{ include "order-service.fullname" . }}-config

---
# templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: {{ include "order-service.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "order-service.labels" . | nindent 4 }}
  annotations:
    {{- toYaml .Values.service.annotations | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.port }}
    targetPort: http
    protocol: TCP
    name: http
  - port: 8081
    targetPort: metrics
    protocol: TCP
    name: metrics
  selector:
    {{- include "order-service.selectorLabels" . | nindent 4 }}

---
# templates/hpa.yaml

{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "order-service.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "order-service.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
{{- end }}
```

---

*CI/CD 自动化平台 - 完整流水线与 GitOps 配置 | ClawsOps ⚙️*