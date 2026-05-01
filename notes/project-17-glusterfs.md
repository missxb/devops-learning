# 项目十七：GlusterFS 分布式存储

## 📋 项目概述

GlusterFS 是一个可扩展的网络附加存储 (NAS) 文件系统，支持 PB 级存储容量。本项目学习 GlusterFS 集群部署和管理。

## 🎯 学习目标

- 理解 GlusterFS 架构原理
- 掌握卷类型和配置
- 部署高可用集群
- 实施数据保护和备份
- 性能调优和监控

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    Client (FUSE/NFS)                     │
│                                                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │              GlusterFS Volume                    │    │
│  │                                                  │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐      │    │
│  │  │  Brick1  │  │  Brick2  │  │  Brick3  │      │    │
│  │  │ Server1  │  │ Server2  │  │ Server3  │      │    │
│  │  │ 100GB    │  │ 100GB    │  │ 100GB    │      │    │
│  │  └──────────┘  └──────────┘  └──────────┘      │    │
│  │                                                  │    │
│  │  Volume Type: Distributed-Replicated             │    │
│  │  Replica Count: 3                                │    │
│  │  Total Capacity: 300GB (100GB usable)            │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## 📦 卷类型

| 类型 | 说明 | 副本数 | 利用率 |
|------|------|--------|--------|
| Distribute | 分布式 | 1 | 100% |
| Replica | 复制式 | 2-3 | 50%/33% |
| Disperse | 纠删码 | 4+2 | 67% |

## 🔧 部署步骤

### 1. 环境准备

```bash
# 所有节点执行
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 安装 GlusterFS
yum install -y centos-release-gluster
yum install -y glusterfs-server

# 启动服务
systemctl enable glusterd
systemctl start glusterd
systemctl status glusterd
```

### 2. 创建集群

```bash
# 在 node1 上执行，将其他节点加入集群
gluster peer probe node2
gluster peer probe node3
gluster peer probe node4

# 查看集群状态
gluster peer status
```

### 3. 创建存储卷

```bash
# 创建复制卷 (3 副本)
gluster volume create gv0 replica 3 \
  node1:/data/brick1 \
  node2:/data/brick2 \
  node3:/data/brick3

# 创建分布式复制卷
gluster volume create gv0 replica 2 \
  node1:/data/brick1 \
  node2:/data/brick2 \
  node3:/data/brick3 \
  node4:/data/brick4

# 启动卷
gluster volume start gv0

# 查看卷信息
gluster volume info
gluster volume status
```

### 4. 挂载使用

```bash
# FUSE 挂载
mount -t glusterfs node1:/gv0 /mnt/gluster

# NFS 挂载
mount -t nfs node1:/gv0 /mnt/gluster

# 验证
df -h
```

## 📚 参考资料

- [GlusterFS 官方文档](https://docs.gluster.org/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
