# 项目一：Harbor 多实例高可用共享存储

## 📋 项目概述

Harbor 是一个开源的容器镜像仓库，用于存储、管理和分发 Docker 镜像。本项目实现 Harbor 多实例高可用部署，使用共享存储确保数据一致性。

## 🎯 学习目标

- 理解 Harbor 架构和组件
- 掌握 Harbor 高可用部署方案
- 配置共享存储（NFS/Ceph）
- 实现负载均衡和故障转移

## 🏗️ 架构设计

```
                    ┌─────────────────┐
                    │   Load Balancer │
                    │   (Nginx/HAProxy)│
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼────────┐    │    ┌────────▼────────┐
     │   Harbor Node 1  │    │    │   Harbor Node 2  │
     │   - Core         │    │    │   - Core         │
     │   - Registry     │    │    │   - Registry     │
     │   - JobService   │    │    │   - JobService   │
     └────────┬─────────┘    │    └─────────┬─────────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │  Shared Storage │
                    │  (NFS/Ceph RBD) │
                    └─────────────────┘
```

## 📦 核心组件

| 组件 | 功能 |
|------|------|
| Core | 核心服务，处理 API 请求、认证授权 |
| Registry | 镜像存储和分发 |
| JobService | 异步任务处理（复制、扫描等） |
| Database | 存储元数据（PostgreSQL） |
| Redis | 缓存和任务队列 |
| ChartMuseum | Helm Chart 仓库 |

## 🔧 部署步骤

### 1. 前置要求

- Docker 19.03+
- Docker Compose 1.18+
- 共享存储（NFS/Ceph）
- 至少 2 台服务器

### 2. 配置共享存储

```bash
# NFS 服务器配置
mkdir -p /data/harbor/data
mkdir -p /data/harbor/registry

# 导出共享目录
echo "/data/harbor *(rw,sync,no_root_squash)" >> /etc/exports
exportfs -r
```

### 3. 挂载共享存储

```bash
# 在 Harbor 节点上挂载
mount -t nfs <nfs-server>:/data/harbor/data /data/harbor/data
mount -t nfs <nfs-server>:/data/harbor/registry /data/harbor/registry
```

### 4. Harbor 配置文件

```yaml
# harbor.yml
hostname: harbor.example.com
http:
  port: 80
https:
  port: 443
  certificate: /your/cert/path
  private_key: /your/private/key/path

database:
  password: root123
  max_idle_conns: 50
  max_open_conns: 1000

data_volume: /data/harbor

clair:
  updaters_interval: 12

jobservice:
  max_job_workers: 10

notification:
  webhook_job_max_retry: 10

log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor
```

### 5. 部署 Harbor

```bash
# 下载 Harbor 安装包
wget https://github.com/goharbor/harbor/releases/download/v2.8.0/harbor-offline-installer-v2.8.0.tgz
tar -zxvf harbor-offline-installer-v2.8.0.tgz
cd harbor

# 安装
./install.sh --with-clair --with-chartmuseum
```

## ✅ 验证部署

```bash
# 检查服务状态
docker-compose ps

# 测试登录
curl -u admin:Harbor12345 https://harbor.example.com/api/v2.0/users/me

# 推送测试镜像
docker pull hello-world
docker tag hello-world harbor.example.com/library/hello-world
docker push harbor.example.com/library/hello-world
```

## 🔐 安全配置

- 启用 HTTPS
- 配置防火墙规则
- 定期更新证书
- 启用镜像漏洞扫描
- 配置访问控制策略

## 📊 监控指标

- CPU/Memory 使用率
- 磁盘空间
- 镜像推送/拉取速率
- API 响应时间
- 数据库连接数

## 🚨 故障排查

### 常见问题

1. **共享存储挂载失败**
   - 检查 NFS 服务状态
   - 验证网络连通性
   - 检查权限配置

2. **镜像推送失败**
   - 检查 Registry 日志
   - 验证存储空间
   - 确认认证配置

3. **数据库连接超时**
   - 检查 PostgreSQL 状态
   - 验证连接池配置
   - 检查网络延迟

## 📚 参考资料

- [Harbor 官方文档](https://goharbor.io/docs/)
- [Harbor GitHub](https://github.com/goharbor/harbor)
- [Docker Registry 最佳实践](https://docs.docker.com/registry/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
