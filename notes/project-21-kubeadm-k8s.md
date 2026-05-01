# 项目二十一：基于 kubeadm 安装 Kubernetes 1.21

## 📋 项目概述

使用 kubeadm 工具快速部署 Kubernetes 1.21 集群，适合生产环境的标准安装方法。

## 🎯 学习目标

- 掌握 kubeadm 工作原理
- 部署高可用控制平面
- 配置网络插件
- 加入工作节点
- 集群管理和升级

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────┐
│              Load Balancer (HAProxy/Cloud LB)            │
│                   192.168.1.100:6443                     │
└────┬─────────────────────┬─────────────────────┬────────┘
     │                     │                     │
┌────▼────┐          ┌────▼────┐          ┌────▼────┐
│ Master  │          │ Master  │          │ Master  │
│   1     │          │   2     │          │   3     │
│ 2C4G    │          │ 2C4G    │          │ 2C4G    │
└─────────┘          └─────────┘          └─────────┘
     │                     │                     │
     └─────────────────────┼─────────────────────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
     ┌────────▼────┐ ┌─────▼────┐ ┌─────▼────┐
     │  Worker 1   │ │  Worker 2│ │  Worker 3│
     │   4C8G      │ │   4C8G   │ │   4C8G   │
     └─────────────┘ └──────────┘ └──────────┘
```

## 🔧 部署步骤

### 1. 系统初始化

```bash
# 所有节点执行
# 关闭 Swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# 配置内核参数
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

# 加载模块
modprobe br_netfilter
echo "br_netfilter" >> /etc/modules-load.d/k8s.conf
```

### 2. 安装容器运行时

```bash
# 安装 containerd
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y containerd.io

# 配置 containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
```

### 3. 安装 kubeadm

```bash
# 配置 Kubernetes YUM 源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
EOF

# 安装 kubeadm kubelet kubectl
yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0
systemctl enable kubelet
```

### 4. 初始化控制平面

```bash
# 在第一个 Master 节点执行
kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --kubernetes-version v1.21.0

# 配置 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 保存 join 命令
kubeadm token create --print-join-command
kubeadm init phase upload-certs --upload-certs
```

### 5. 加入其他节点

```bash
# 其他 Master 节点
kubeadm join 192.168.1.100:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx \
  --control-plane \
  --certificate-key xxxxx

# Worker 节点
kubeadm join 192.168.1.100:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx
```

### 6. 部署网络插件

```bash
# Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 或 Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 验证 Pod 状态
kubectl get pods -n kube-system -o wide
```

## 📊 集群管理

### 1. 查看集群信息

```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

### 2. 证书续期

```bash
# 检查证书过期时间
kubeadm certs check-expiration

# 续期所有证书
kubeadm certs renew all

# 重启控制平面组件
systemctl restart kubelet
```

### 3. 集群升级

```bash
# 升级 kubeadm
yum update -y kubeadm-1.22.0

# 升级计划
kubeadm upgrade plan

# 应用升级
kubeadm upgrade apply v1.22.0

# 升级 kubelet 和 kubectl
yum update -y kubelet-1.22.0 kubectl-1.22.0
systemctl restart kubelet
```

## 📚 参考资料

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [kubeadm 文档](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
