# 服务网格 (Istio)

## 📋 概述

Istio 是开源的服务网格实现，提供服务发现、负载均衡、TLS、监控、链路追踪等能力，无需修改应用代码。

## 🎯 学习目标

- 理解服务网格架构
- 掌握 Istio 核心组件
- 配置流量管理策略
- 实施安全策略
- 可观测性集成

## 🏗️ Istio 架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Istio 服务网格                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              数据平面 (Data Plane)                   │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Sidecar│  │  Sidecar│  │  Sidecar│            │   │
│  │  │ Envoy   │  │  Envoy  │  │  Envoy  │            │   │
│  │  │ ┌─────┐ │  │ ┌─────┐ │  │ ┌─────┐ │            │   │
│  │  │ │ App │ │  │ │ App │ │  │ │ App │ │            │   │
│  │  │ └─────┘ │  │ └─────┘ │  │ └─────┘ │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              控制平面 (Control Plane)                │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  Pilot  │  │  Mixer  │  │ Citadel │            │   │
│  │  │ 配置管理│  │ 策略遥测│  │ 证书管理│            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │              Istiod (1.8+)                   │   │   │
│  │  │    Pilot + Mixer + Citadel 整合              │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 🔧 Istio 部署

### 1. 安装 Istio

```bash
# 下载 istioctl
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/bin:$PATH

# 验证安装
istioctl version

# 安装 Istio (生产配置)
istioctl install --set profile=demo \
  --set values.pilot.replicaCount=3 \
  --set values.global.proxy.resources.requests.cpu=100m \
  --set values.global.proxy.resources.requests.memory=128Mi \
  --set values.global.proxy.resources.limits.cpu=500m \
  --set values.global.proxy.resources.limits.memory=256Mi \
  -y

# 验证安装
kubectl get pods -n istio-system
```

### 2. 注入 Sidecar

```bash
# 命名空间级别注入
kubectl label namespace default istio-injection=enabled

# 应用级别注入
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  annotations:
    sidecar.istio.io/inject: "true"
spec:
  containers:
  - name: myapp
    image: myapp:1.0
```

### 3. Gateway 配置

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "myapp.example.com"
    tls:
      httpsRedirect: true
  
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - "myapp.example.com"
    tls:
      mode: SIMPLE
      credentialName: myapp-tls-cert
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
  - "myapp.example.com"
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /api
    route:
    - destination:
        host: api-service
        port:
          number: 8080
  
  - route:
    - destination:
        host: web-service
        port:
          number: 80
```

## 🚦 流量管理

### 1. 路由规则

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
spec:
  hosts:
  - reviews
  http:
  # 基于权重的路由 (金丝雀发布)
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
  
  # 基于 Header 的路由
  - match:
    - headers:
        end-user:
          exact: "test-user"
    route:
    - destination:
        host: reviews
        subset: v2
  
  # 基于 URI 的路由
  - match:
    - uri:
        prefix: /api/v2
    route:
    - destination:
        host: reviews
        subset: v2
```

### 2. 故障注入

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
spec:
  hosts:
  - reviews
  http:
  - fault:
      # 延迟注入
      delay:
        percentage:
          value: 10
        fixedDelay: 5s
      
      # 中止注入
      abort:
        percentage:
          value: 5
        httpStatus: 500
    
    route:
    - destination:
        host: reviews
        subset: v1
```

### 3. 超时与重试

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-vs
spec:
  hosts:
  - reviews
  http:
  - timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,reset,connect-failure,retriable-4xx
    
    route:
    - destination:
        host: reviews
        subset: v1
```

### 4. 熔断器

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-dr
spec:
  host: reviews
  trafficPolicy:
    # 连接池配置
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    
    # 熔断器配置
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
```

## 🔒 安全策略

### 1. mTLS 配置

```yaml
# 网格级别 mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT

# 命名空间级别 mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT

# 服务级别 mTLS (允许明文)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: api-service
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  mtls:
    mode: PERMISSIVE
```

### 2. 授权策略

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  action: ALLOW
  rules:
  # 允许特定服务访问
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/frontend-service"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/*"]
  
  # 允许特定 Header
  - from:
    - source:
        namespaces: ["production"]
    when:
    - key: request.auth.claims[iss]
      values: ["https://auth.example.com"]
```

### 3. JWT 认证

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  jwtRules:
  - issuer: "https://auth.example.com"
    jwksUri: "https://auth.example.com/.well-known/jwks.json"
    audiences:
    - "api-service"
    forwardOriginalToken: true
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-service
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]
```

## 📊 可观测性

### 1. 指标收集

```yaml
# Prometheus 配置
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-monitor
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: pilot
  endpoints:
  - port: http-monitoring
    interval: 30s
```

### 2. 链路追踪

```yaml
# Jaeger 集成
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0
        zipkin:
          address: jaeger-collector.istio-system:9411
  components:
    tracing:
      enabled: true
```

### 3. 访问日志

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogFormat: |
      [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" 
      %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% 
      "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%" 
      "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%"
```

## 🔧 故障排查

### 常用命令

```bash
# 检查配置
istioctl analyze

# 查看代理配置
istioctl proxy-config all <pod-name>

# 查看路由
istioctl proxy-config route <pod-name>

# 查看监听器
istioctl proxy-config listener <pod-name>

# 查看集群
istioctl proxy-config cluster <pod-name>

# 查看端点
istioctl proxy-config endpoint <pod-name>

# 日志查看
istioctl proxy-status
kubectl logs -n istio-system -l app=istiod

# Sidecar 注入检查
istioctl kube-inject -f deployment.yaml
```

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Sidecar 未注入 | 命名空间未标记 | `kubectl label namespace default istio-injection=enabled` |
| 503 UF 错误 | 上游服务不可用 | 检查 DestinationRule 和 Service |
| mTLS 失败 | 证书问题 | 检查 PeerAuthentication 配置 |
| 路由不生效 | VirtualService 未绑定 | 检查 hosts 和 gateway 配置 |

## 📚 参考资料

- [Istio 官方文档](https://istio.io/docs/)
- [Envoy 代理文档](https://www.envoyproxy.io/docs)
- [Service Mesh 实践](https://servicemesh.patterns/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
