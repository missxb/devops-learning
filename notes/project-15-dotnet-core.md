# 项目十五：.NET Core 容器化

## 📋 项目概述

将.NET Core 应用容器化部署到 Kubernetes 集群，实现微服务架构的现代化部署方案。

## 🎯 学习目标

- 掌握 Docker 多阶段构建
- 理解.NET Core 运行时优化
- 配置健康检查和日志
- 部署到 Kubernetes
- 实施 CI/CD 流程

## 📦 Dockerfile 最佳实践

### 1. 多阶段构建

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src

# Copy csproj and restore dependencies
COPY *.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:5.0 AS runtime
WORKDIR /app

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# Copy published app
COPY --from=build /app/publish .

# Expose port
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# Entry point
ENTRYPOINT ["dotnet", "YourApp.dll"]
```

### 2. Kubernetes 部署配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dotnet-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: dotnet-app
  template:
    metadata:
      labels:
        app: dotnet-app
    spec:
      containers:
      - name: dotnet-app
        image: your-registry/dotnet-app:latest
        ports:
        - containerPort: 80
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__DefaultConnection
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: connection-string
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: dotnet-app-service
spec:
  selector:
    app: dotnet-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

## 📚 参考资料

- [.NET Core 官方文档](https://docs.microsoft.com/dotnet/)
- [Docker 官方文档](https://docs.docker.com/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
