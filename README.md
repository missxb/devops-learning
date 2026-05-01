# DevOps 学习笔记与实践

> 📚 基于 xhaihua 的 DevOps 项目知识库学习整理

## 🎯 学习目标

在 3 天内掌握企业级 DevOps 运维知识，完成教学实践项目。

**学习周期:** 2026-05-01 ~ 2026-05-04

## 📖 学习目录

### 容器与编排
- [ ] 项目一：Harbor 多实例高可用共享存储
- [ ] 项目六：基于二进制安装 Kubernetes 1.19.3
- [ ] 项目八：通过阿里云 ECS 部署高可用 k8s 集群
- [ ] 项目九：通过 rook 部署 Ceph 分布式存储
- [ ] 项目十：Kubernetes 云架构平台
- [ ] 项目十五:.NET Core 容器化
- [ ] 基于 kubeadm 安装 Kubernetes 1.21

### 消息队列
- [ ] 项目二：RabbitMQ 3.8 镜像队列集群
- [ ] 项目三：Kafka 集群部署及监控

### 数据库
- [ ] 项目五：Redis 5.0 集群部署
- [ ] 项目十一：mongo 4.4.2+ 副本集 + 认证部署
- [ ] 项目十八：mongodb

### 监控告警
- [ ] 项目四：prometheus 监控系统

### 负载均衡
- [ ] 项目十九：LB+keepalived

### 分布式存储
- [ ] 项目十七：glusterfs 存储

### SRE 运维体系
- [ ] 项目十三：SRE
- [ ] 项目十三：SRE 运维体系
- [ ] 项目十六：运维体系化建设

### 其他
- [ ] 项目七:openstack（Rocky 版）集群部署
- [ ] 项目十二：系统架构设计
- [ ] 项目十九:rsync/lsyncd/sersync

## 🗂️ 目录结构

```
devops-learning/
├── README.md                 # 本文件
├── notes/                    # 学习笔记
│   ├── project-01-harbor.md
│   ├── project-02-rabbitmq.md
│   └── ...
├── practice/                 # 实践代码和脚本
│   ├── docker-compose/
│   ├── kubernetes/
│   ├── ansible/
│   └── scripts/
└── resources/                # 参考资料
    └── yuque-docs/          # Yuque 原始文档
```

## 🛠️ 技术栈

- **容器:** Docker, Harbor
- **编排:** Kubernetes, OpenStack
- **消息队列:** RabbitMQ, Kafka
- **数据库:** MongoDB, Redis
- **监控:** Prometheus, Grafana, Alertmanager
- **存储:** Ceph, GlusterFS
- **CI/CD:** Jenkins, GitLab CI
- **配置管理:** Ansible

## 📝 学习环境

- **主机:** CentOS 8.x
- **虚拟化:** KVM/QEMU
- **网络:** Bridge networking

---

*创建时间：2026-05-01 | ClawsOps ⚙️*
