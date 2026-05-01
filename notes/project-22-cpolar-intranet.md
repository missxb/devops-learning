# 内网穿透部署手册：基于 Docker + Cpolar

## 📋 项目概述

Cpolar 是一款安全的内网穿透工具，支持 HTTP/HTTPS/TCP 协议，可将内网服务暴露到公网。本项目学习使用 Docker 容器化部署 Cpolar，实现内网服务的公网访问。

## 🎯 学习目标

- 理解内网穿透原理
- 掌握 Cpolar 核心概念
- 容器化部署 Cpolar
- 配置 HTTP/TCP 隧道
- 实施安全加固

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                      公网用户                                │
│                    (Internet)                                │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  Cpolar 云服务器                             │
│              tunnel.cpolar.cn:80/443                         │
│         分配公网域名：xxx.cpolar.cn                          │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           │ 加密隧道
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   内网环境                                    │
│  ┌─────────────┐                                           │
│  │   Cpolar    │  ← Docker 容器                             │
│  │   Client    │     192.168.1.100:9200                    │
│  └──────┬──────┘                                           │
│         │                                                   │
│         ▼                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Web App   │  │   SSH       │  │   Database  │        │
│  │  :8080      │  │   :22       │  │   :3306     │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

## 📦 核心概念

| 术语 | 说明 | 示例 |
|------|------|------|
| **隧道 (Tunnel)** | 内网到公网的加密通道 | http-tunnel, ssh-tunnel |
| **域名 (Domain)** | Cpolar 分配的公网访问地址 | abc123.cpolar.cn |
| **区域 (Region)** | 隧道所在的服务器区域 | cn-vip, cn-free |
| **带宽 (Bandwidth)** | 隧道的传输速率限制 | 1Mbps, 10Mbps |

## 🔧 部署步骤

### 1. 注册 Cpolar 账号

```bash
# 访问官网注册
# https://www.cpolar.com/

# 获取 Authtoken
# 登录后在 Dashboard → 个人认证 → Authtoken
```

### 2. Docker 部署 Cpolar

```bash
# 创建数据目录
mkdir -p /opt/cpolar/data

# 运行 Cpolar 容器
docker run -d \
  --name cpolar \
  --restart=always \
  --net=host \
  -v /opt/cpolar/data:/root/.cpolar \
  -e AUTHTOKEN="your_autoken_here" \
  cpolar/cpolar
```

### 3. 配置隧道

```bash
# 进入容器配置
docker exec -it cpolar /bin/sh

# 配置 Authtoken
cpolar authtoken YOUR_AUTHTOKEN

# 创建 HTTP 隧道 (Web 服务)
cpolar http 8080

# 创建 TCP 隧道 (SSH)
cpolar tcp 22

# 创建 HTTPS 隧道
cpolar http 443
```

### 4. Docker Compose 部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  cpolar:
    image: cpolar/cpolar:latest
    container_name: cpolar
    restart: always
    network_mode: host
    volumes:
      - ./cpolar-data:/root/.cpolar
    environment:
      - AUTHTOKEN=your_authtoken_here
    command: ["http", "8080"]
    labels:
      - "com.cpolar.tunnel=http-web"
```

```bash
# 启动服务
docker-compose up -d

# 查看状态
docker-compose logs -f cpolar
```

### 5. 配置文件方式

```yaml
# config.yml
version: "2.0"
authtoken: "your_authtoken_here"
region: cn-vip

tunnels:
  web-service:
    proto:
      http: 8080
    domain_name:
      - "abc123.cpolar.cn"
    
  ssh-service:
    proto:
      tcp: 22
    remote_port: 6000
    
  database:
    proto:
      tcp: 3306
    remote_port: 6001
```

```bash
# 使用配置文件启动
docker run -d \
  --name cpolar \
  --restart=always \
  --net=host \
  -v /opt/cpolar/config.yml:/root/.cpolar/config.yml \
  cpolar/cpolar
```

## 🌐 隧道类型配置

### 1. HTTP 隧道 (Web 服务)

```bash
# 临时隧道
cpolar http 8080

# 固定域名隧道
cpolar http 8080 --subdomain=myapp

# 带认证的隧道
cpolar http 8080 --basic-auth username:password

# 自定义请求头
cpolar http 8080 --host-header=rewrite
```

**输出示例：**
```
Forwarding  https://abc123.cpolar.cn -> http://localhost:8080
Forwarding  http://abc123.cpolar.cn -> http://localhost:8080
```

### 2. TCP 隧道 (SSH/数据库)

```bash
# SSH 隧道
cpolar tcp 22 --region=cn-vip

# MySQL 隧道
cpolar tcp 3306 --region=cn-vip

# 远程端口指定
cpolar tcp 22 --remote-port=6000
```

**连接示例：**
```bash
# SSH 连接
ssh -p 6000 user@tcp.cpolar.cn

# MySQL 连接
mysql -h tcp.cpolar.cn -P 6000 -u root -p
```

### 3. HTTPS 隧道

```bash
# 自动 HTTPS
cpolar http 443

# 强制 HTTPS 重定向
cpolar http 80 --scheme=https
```

## 🔒 安全加固

### 1. 访问控制

```bash
# IP 白名单
cpolar http 8080 --ip-restrict=192.168.1.0/24,10.0.0.0/8

# 基础认证
cpolar http 8080 --basic-auth=admin:SecurePass123

# 请求头验证
cpolar http 8080 --verify-header=X-Auth-Token:secret123
```

### 2. 日志审计

```bash
# 查看隧道日志
docker logs cpolar

# 查看连接统计
curl -s http://localhost:9200/api/tunnels

# 实时监控
watch -n 1 'docker logs cpolar --tail 20'
```

### 3. 防火墙配置

```bash
# 仅允许 Cpolar 容器访问内网服务
iptables -A INPUT -p tcp --dport 8080 -s 172.17.0.0/16 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -j DROP
```

## 📊 监控与告警

### 1. 健康检查

```yaml
# docker-compose.yml 添加健康检查
services:
  cpolar:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/api/status"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

### 2. Prometheus 监控

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'cpolar'
    static_configs:
      - targets: ['cpolar-host:9200']
    metrics_path: '/api/metrics'
```

### 3. 告警规则

```yaml
# alert-rules.yml
groups:
  - name: cpolar-alerts
    rules:
      - alert: CpolarTunnelDown
        expr: cpolar_tunnel_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Cpolar tunnel {{ $labels.tunnel }} is down"
      
      - alert: CpolarHighLatency
        expr: cpolar_tunnel_latency_ms > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on tunnel {{ $labels.tunnel }}"
```

## 🔧 故障排查

### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 隧道连接失败 | Authtoken 错误 | 重新配置 `cpolar authtoken` |
| 服务不可访问 | 防火墙阻挡 | 检查 iptables/安全组 |
| 速度慢 | 带宽限制 | 升级套餐或切换区域 |
| 域名无法解析 | DNS 缓存 | 清除本地 DNS 缓存 |

### 诊断命令

```bash
# 查看隧道状态
curl http://localhost:9200/api/tunnels

# 测试连通性
curl -I https://abc123.cpolar.cn

# 查看日志
docker logs cpolar --tail 100

# 网络诊断
ping abc123.cpolar.cn
traceroute abc123.cpolar.cn
```

## 📚 应用场景

### 1. 开发环境暴露

```bash
# 本地开发 Web 应用
cpolar http 3000 --subdomain=dev-myapp

# 预览地址：https://dev-myapp.cpolar.cn
```

### 2. 远程调试

```bash
# 暴露本地 API 服务
cpolar http 8080 --basic-auth=dev:debug123

# 微信开发回调
# 配置回调 URL: https://xxx.cpolar.cn/wechat/callback
```

### 3. 临时演示

```bash
# 快速分享本地服务
cpolar http 80 --subdomain=demo

# 分享给客户：https://demo.cpolar.cn
```

### 4. 远程运维

```bash
# SSH 隧道
cpolar tcp 22 --remote-port=6000

# 远程连接
ssh -p 6000 user@tcp.cpolar.cn
```

## 📚 参考资料

- [Cpolar 官方文档](https://www.cpolar.com/docs)
- [内网穿透技术原理](https://en.wikipedia.org/wiki/Port_forwarding)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
