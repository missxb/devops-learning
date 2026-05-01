# Docker 容器化运维实战

## 📋 概述

Docker 是容器化技术的核心，掌握 Docker 运维是云原生时代的基础技能。

## 🎯 学习目标

- 深入理解 Docker 架构
- 掌握镜像优化技巧
- 配置容器网络和存储
- 实施容器安全策略
- 生产环境最佳实践

## 🏗️ Docker 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Client                         │
│                    (CLI / API)                           │
└──────────────────────────┬──────────────────────────────┘
                           │ REST API
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    Docker Daemon                         │
│                    (dockerd)                             │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Images    │  │ Containers  │  │  Networks   │     │
│  │   Manager   │  │   Manager   │  │   Manager   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│                                                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Container Runtime                   │   │
│  │            (containerd + runc)                   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 🔧 镜像优化

### 1. 多阶段构建

```dockerfile
# ❌ 低效方式
FROM node:16
WORKDIR /app
COPY . .
RUN npm install && npm run build
CMD ["node", "dist/index.js"]

# ✅ 多阶段构建
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/index.js"]
```

### 2. 层缓存优化

```dockerfile
# ❌ 缓存效率低
COPY . .
RUN npm install

# ✅ 缓存效率高
COPY package*.json ./
RUN npm install
COPY . .
```

### 3. 镜像大小对比

| 基础镜像 | 大小 | 适用场景 |
|----------|------|----------|
| ubuntu:20.04 | 72MB | 通用场景 |
| alpine:3.14 | 5MB | 轻量应用 |
| distroless | 2MB | 生产环境 |
| scratch | 0B | 静态编译 |

## 🌐 容器网络

### 1. 网络类型

```bash
# 查看网络类型
docker network ls

# Bridge 网络 (默认)
docker network create --driver bridge mynet

# Host 网络
docker run --net=host nginx

# None 网络
docker run --net=none alpine

# Overlay 网络 (Swarm)
docker network create --driver overlay myoverlay
```

### 2. 容器互联

```bash
# 创建网络
docker network create app-network

# 启动容器加入网络
docker run -d --name web --network app-network nginx
docker run -d --name api --network app-network myapi

# 容器间通信
docker exec web curl http://api:8080
```

### 3. 端口映射

```bash
# 映射到宿主机
docker run -p 8080:80 nginx

# 映射到指定 IP
docker run -p 127.0.0.1:8080:80 nginx

# 映射端口范围
docker run -p 8080-8090:80-90 nginx

# 动态端口
docker run -P nginx
```

## 💾 容器存储

### 1. 存储类型

```bash
# Volume (推荐)
docker volume create myvol
docker run -v myvol:/data nginx

# Bind Mount
docker run -v /host/path:/container/path nginx

# tmpfs
docker run --tmpfs /app nginx
```

### 2. 数据持久化

```yaml
# docker-compose.yml
version: '3.8'
services:
  database:
    image: postgres:13
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    
  app:
    image: myapp
    volumes:
      - ./config:/app/config:ro
      - logs:/app/logs

volumes:
  pgdata:
  logs:
```

## 🔒 容器安全

### 1. 最小权限原则

```bash
# 非 root 用户运行
docker run --user 1000:1000 nginx

# 只读文件系统
docker run --read-only nginx

# 限制能力
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx
```

### 2. 安全扫描

```bash
# Docker Scout
docker scout cves myimage

# Trivy 扫描
trivy image myimage

# Clair 扫描
docker run -p 6060:6060 quay.io/coreos/clair
```

### 3. 镜像签名

```bash
# 启用内容信任
export DOCKER_CONTENT_TRUST=1

# 签名镜像
docker trust sign myimage:latest

# 验证镜像
docker trust inspect myimage:latest
```

## 📊 生产最佳实践

### 1. 资源限制

```bash
docker run -d \
  --name myapp \
  --memory="512m" \
  --memory-swap="1g" \
  --cpus="1.0" \
  --pids-limit=100 \
  myapp
```

### 2. 健康检查

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  --start-period=40s --retries=3 \
  CMD curl -f http://localhost/health || exit 1
```

### 3. 日志管理

```bash
# 配置日志驱动
docker run --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx

# 查看日志
docker logs --tail 100 --follow myapp
```

### 4. 重启策略

```bash
# 不重启
docker run --restart=no nginx

# 失败时重启
docker run --restart=on-failure:3 nginx

# 总是重启
docker run --restart=always nginx

# 除非手动停止
docker run --restart=unless-stopped nginx
```

## 🔧 故障排查

```bash
# 查看容器状态
docker ps -a

# 查看资源使用
docker stats

# 检查容器详情
docker inspect myapp

# 进入容器
docker exec -it myapp /bin/sh

# 查看进程
docker top myapp

# 查看变更
docker diff myapp
```

## 📚 参考资料

- [Docker 官方文档](https://docs.docker.com/)
- [Docker 最佳实践](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
