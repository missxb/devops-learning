# 企业实战项目一：电商平台 K8s 高可用架构

## 📋 项目背景

**客户：** 某电商平台，日均订单 10 万+，峰值 QPS 5000
**需求：** 构建高可用、可扩展的 K8s 生产环境
**预算：** ¥50 万 / 年
**周期：** 2 个月

---

## 🏗️ 项目架构

### 整体架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                   电商平台架构设计                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    用户入口                              ││
│  │           CDN (阿里云/腾讯云)                            ││
│  └─────────────────────────────────────────────────────────┘│
│                         │                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    接入层                                ││
│  │         ┌───────────┐        ┌───────────┐             ││
│  │         │   WAF     │        │  DDoS 防护 │             ││
│  │         │ 云盾/安恒 │        │ 云盾/DDoS │             ││
│  │         └───────────┘        └───────────┘             ││
│  │                       │                                 ││
│  │         ┌─────────────────────────────────┐            ││
│  │         │      Nginx Ingress Controller    │            ││
│  │         │      (多副本 + 高可用)            │            ││
│  │         └─────────────────────────────────┘            ││
│  └─────────────────────────────────────────────────────────┘│
│                         │                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    应用层                                ││
│  │                                                       ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ││
│  │  │  前端   │  │  商品   │  │  订单   │  │  用户   │  ││
│  │  │ Web    │  │ Service │  │ Service │  │ Service │  ││
│  │  │(Vue)   │  │ (Go)    │  │ (Java)  │  │ (Java)  │  ││
│  │  │ 3副本  │  │ 5副本   │  │ 10副本  │  │ 5副本   │  ││
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘  ││
│  │                                                       ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐               ││
│  │  │  支付   │  │  搜索   │  │  推荐   │               ││
│  │  │ Service │  │ Service │  │ Service │               ││
│  │  │ (Java)  │  │ (Go)    │  │ (Python)│               ││
│  │  │ 5副本   │  │ 3副本   │  │ 3副本   │               ││
│  │  └─────────┘  └─────────┘  └─────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
│                         │                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    中间件层                              ││
│  │                                                       ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐               ││
│  │  │  Redis  │  │  Kafka  │  │  Nacos  │               ││
│  │  │ 集群3节点│  │ 集群5节点│  │ 集群3节点│               ││
│  │  └─────────┘  └─────────┘  └─────────┘               ││
│  │                                                       ││
│  │  ┌─────────┐  ┌─────────┐                             ││
│  │  │ RabbitMQ│  │  ES     │                             ││
│  │  │ 集群3节点│  │ 集群3节点│                             ││
│  │  └─────────┘  └─────────┘                             ││
│  └─────────────────────────────────────────────────────────┘│
│                         │                                   │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    数据层                                ││
│  │                                                       ││
│  │  ┌─────────────────────┐  ┌─────────────────────┐    ││
│  │  │    MySQL 主从集群     │  │    MongoDB 集群     │    ││
│  │  │   Master (写)        │  │   Replica Set 3节点 │    ││
│  │  │   Slave-1 (读)       │  │                     │    ││
│  │  │   Slave-2 (读)       │  │                     │    ││
│  │  └─────────────────────┘  └─────────────────────┘    ││
│  │                                                       ││
│  │  ┌─────────────────────────────────────────────────┐  ││
│  │  │              OSS 对象存储                         │  ││
│  │  │         商品图片/用户头像/订单附件                 │  ││
│  │  └─────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 📐 集群规划

### 节点规划

```
┌─────────────────────────────────────────────────────────────┐
│                    K8s 集群节点规划                          │
│                                                             │
│  控制平面节点 (3 台)                                         │
│  ├── master-01: 192.168.1.10 (4C8G100G SSD)                │
│  ├── master-02: 192.168.1.11 (4C8G100G SSD)                │
│  └── master-03: 192.168.1.12 (4C8G100G SSD)                │
│                                                             │
│  工作节点 (10 台)                                            │
│  ├── worker-01~03: 192.168.1.20~22 (8C16G200G SSD)         │
│  │   用途：前端/商品/用户服务                                │
│  │                                                         │
│  ├── worker-04~06: 192.168.1.23~25 (16C32G500G SSD)        │
│  │   用途：订单服务 (高资源需求)                             │
│  │                                                         │
│  ├── worker-07~08: 192.168.1.26~27 (8C32G1T HDD)           │
│  │   用途：数据库/存储 (高磁盘需求)                          │
│  │                                                         │
│  └── worker-09~10: 192.168.1.28~29 (4C8G100G SSD)          │
│  │   用途：监控/日志/CI (基础服务)                           │
│                                                             │
│  总计：13 台服务器                                          │
│  ├── CPU：96 核                                             │
│  ├── 内存：176 GB                                           │
│  └── 存储：2.3 TB                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 部署步骤

### 第一阶段：基础设施准备 (1 周)

#### 1.1 服务器初始化

```bash
#!/bin/bash
# init-server.sh - 服务器初始化脚本

# 系统更新
yum update -y

# 安装基础工具
yum install -y wget curl git vim net-tools telnet

# 时间同步
yum install -y chrony
systemctl enable chronyd
systemctl start chronyd

# 关闭防火墙 (K8s 内部网络)
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config

# 关闭 Swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# 内核参数优化
cat >> /etc/sysctl.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
sysctl -p

# 配置 hosts
cat >> /etc/hosts << EOF
192.168.1.10 master-01
192.168.1.11 master-02
192.168.1.12 master-03
192.168.1.20 worker-01
192.168.1.21 worker-02
192.168.1.22 worker-03
EOF
```

#### 1.2 Docker 安装

```bash
#!/bin/bash
# install-docker.sh

# 安装 Docker
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io

# 配置 Docker
mkdir -p /etc/docker
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://registry.cn-hangzhou.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

# 启动 Docker
systemctl enable docker
systemctl start docker

# 验证
docker version
docker info
```

#### 1.3 Kubernetes 安装

```bash
#!/bin/bash
# install-k8s.sh

# 安装 kubeadm/kubelet/kubectl
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF

yum install -y kubeadm-1.24.0 kubelet-1.24.0 kubectl-1.24.0
systemctl enable kubelet
```

### 第二阶段：集群搭建 (1 周)

#### 2.1 初始化控制平面

```bash
# 在 master-01 执行

# 初始化集群
kubeadm init \
  --kubernetes-version v1.24.0 \
  --control-plane-endpoint "192.168.1.10:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

# 配置 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 其他 master 加入集群
kubeadm join 192.168.1.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

#### 2.2 安装网络插件

```bash
# 安装 Calico
kubectl apply -f https://docs.projectcalico.org/v3.23/manifests/calico.yaml

# 验证网络
kubectl get pods -n kube-system | grep calico
```

#### 2.3 Worker 节点加入

```bash
# 在 worker 节点执行
kubeadm join 192.168.1.10:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# 验证节点
kubectl get nodes -o wide
```

### 第三阶段：应用部署 (2 周)

#### 3.1 部署 Ingress Controller

```yaml
# nginx-ingress.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: nginx-ingress
  namespace: kube-system
spec:
  chart: ingress-nginx
  repo: https://kubernetes.github.io/ingress-nginx
  
  valuesContent: |
    controller:
      replicaCount: 3
      service:
        type: LoadBalancer
        loadBalancerIP: 192.168.1.100
      
      # 资源配置
      resources:
        limits:
          cpu: 2
          memory: 4Gi
        requests:
          cpu: 500m
          memory: 512Mi
      
      # 配置优化
      config:
        proxy-body-size: "10m"
        proxy-read-timeout: "60"
        proxy-send-timeout: "60"
        use-proxy-protocol: "false"
```

#### 3.2 部署监控系统

```yaml
# prometheus-stack.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: prometheus-stack
  namespace: monitoring
spec:
  chart: kube-prometheus-stack
  repo: https://prometheus-community.github.io/helm-charts
  
  valuesContent: |
    prometheus:
      prometheusSpec:
        retention: 7d
        storageSpec:
          volumeClaimTemplate:
            spec:
              storageClassName: standard
              resources:
                requests:
                  storage: 100Gi
    
    grafana:
      adminPassword: admin123
      additionalDataSources:
      - name: Loki
        type: loki
        url: http://loki:3100
    
    alertmanager:
      alertmanagerSpec:
        storage:
          volumeClaimTemplate:
            spec:
              resources:
                requests:
                  storage: 10Gi
```

#### 3.3 部署日志系统

```yaml
# loki-stack.yaml

apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: loki-stack
  namespace: logging
spec:
  chart: loki-stack
  repo: https://grafana.github.io/helm-charts
  
  valuesContent: |
    loki:
      config:
        auth_enabled: false
        limits_config:
          reject_old_samples: true
          reject_old_samples_max_age: 168h
    
    promtail:
      config:
        clients:
        - url: http://loki:3100/loki/api/v1/push
```

---

## 📊 运维手册

### 日常巡检

```bash
#!/bin/bash
# daily-check.sh

echo "=== 电商平台每日巡检 ==="

# 集群状态
kubectl get nodes
kubectl get pods -A | grep -v Running | head -20

# 资源使用
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -10

# 服务健康
curl -s http://frontend.example.com/health
curl -s http://api.example.com/health

# 数据库状态
mysql -h db-master -e "SHOW PROCESSLIST;" | wc -l
redis-cli INFO memory | grep used_memory_human

# 监控告警
curl -s http://alertmanager:9093/api/v1/alerts | jq '.data[] | select(.status.state=="firing")'
```

### 故障响应

```markdown
# 故障响应 SOP

## P0 故障 (5 分钟响应)
1. 接到告警 → 1 分钟内确认
2. 拉应急群 → 通知相关人员
3. 检查服务状态 → kubectl get pods
4. 检查日志 → kubectl logs
5. 临时恢复 → 回滚/重启
6. 根因分析 → 故障排查

## 回滚流程
kubectl rollout undo deployment/frontend -n production
kubectl rollout undo deployment/order-service -n production

## 扩容流程
kubectl scale deployment order-service --replicas=20 -n production
```

---

## 📋 项目交付清单

```markdown
# 项目交付清单

## 基础设施
- [x] 13 台服务器初始化完成
- [x] Docker 安装配置完成
- [x] K8s 集群搭建完成 (3 master + 10 worker)
- [x] Calico 网络插件安装完成
- [x] Ingress Controller 部署完成

## 应用服务
- [x] 前端服务部署完成 (3副本)
- [x] 商品服务部署完成 (5副本)
- [x] 订单服务部署完成 (10副本)
- [x] 用户服务部署完成 (5副本)
- [x] 支付服务部署完成 (5副本)

## 中间件
- [x] Redis 集群部署完成 (3节点)
- [x] Kafka 集群部署完成 (5节点)
- [x] MySQL 主从部署完成 (1主2从)
- [x] MongoDB 集群部署完成 (3节点)

## 监控运维
- [x] Prometheus Stack 部署完成
- [x] Loki 日志系统部署完成
- [x] Grafana Dashboard 配置完成
- [x] 告警规则配置完成

## 文档交付
- [x] 架构设计文档
- [x] 部署手册
- [x] 运维手册
- [x] 故障应急预案
```

---

## 💰 项目成本

```
┌─────────────────────────────────────────────────────────────┐
│                    项目成本明细                              │
│                                                             │
│  硬件成本                                                   │
│  ├── 13 台服务器租赁：¥30 万/年                             │
│  ├── 网络带宽 (100M)：¥5 万/年                              │
│  └── OSS 存储 (500G)：¥2 万/年                              │
│  │                                                         │
│  软件成本                                                   │
│  ├── 阿里云服务：¥5 万/年                                   │
│  ├── 商业镜像仓库：¥2 万/年                                 │
│  │                                                         │
│  运维成本                                                   │
│  ├── 运维工程师 1 人：¥10 万/年                             │
│  │                                                         │
│  总计：¥54 万/年                                           │
│  │                                                         │
│  优化后成本：¥40 万/年                                     │
│  ├── 使用竞价实例：节省 30%                                 │
│  ├── 资源优化 sizing：节省 20%                              │
│  └── 预留实例包年：节省 50%                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 📚 相关文档

- [项目架构设计](./architecture.md)
- [部署详细步骤](./deployment.md)
- [运维手册](./operations.md)
- [故障案例](./failures.md)

---

*企业实战项目一：电商平台 K8s 高可用架构 | ClawsOps ⚙️*