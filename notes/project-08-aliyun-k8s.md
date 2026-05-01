# 项目八：通过阿里云 ECS 部署高可用 K8s 集群

## 📋 项目概述

在阿里云上使用 ECS 云服务器部署高可用 Kubernetes 集群，利用阿里云的 SLB、VPC、云盘等云产品实现生产级别的 K8s 环境。

## 🎯 学习目标

- 掌握云上 K8s 架构设计
- 使用 SLB 实现 API Server 高可用
- 配置 VPC 网络和交换机
- 使用云盘实现数据持久化
- 集成阿里云监控和日志服务

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    阿里云 SLB (负载均衡)                  │
│              公网 IP + 内网 IP (6443)                     │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │  ECS1  │  │  ECS2  │  │  ECS3  │
    │ Master │  │ Master │  │ Master │
    │  2C4G  │  │  2C4G  │  │  2C4G  │
    │ 40G 云盘│  │ 40G 云盘│  │ 40G 云盘│
    └────────┘  └────────┘  └────────┘
         │           │           │
         └───────────┼───────────┘
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │  ECS4  │  │  ECS5  │  │  ECS6  │
    │ Worker │  │ Worker │  │ Worker │
    │  4C8G  │  │  4C8G  │  │  4C8G  │
    │100G 云盘│  │100G 云盘│  │100G 云盘│
    └────────┘  └────────┘  └────────┘
```

## 📦 资源清单

| 资源 | 规格 | 数量 | 用途 |
|------|------|-----|------|
| ECS (Master) | ecs.c2.large (2C4G) | 3 | 控制节点 |
| ECS (Worker) | ecs.c2.xlarge (4C8G) | 3 | 工作节点 |
| SLB | slb.s1.small | 1 | API Server 负载均衡 |
| 云盘 (Master) | 40G 高效云盘 | 3 | 系统盘 |
| 云盘 (Worker) | 100G 高效云盘 | 3 | 系统 + 数据盘 |
| VPC | /24 网段 | 1 | 私有网络 |
| 交换机 | /24 子网 | 3 | 可用区分布 |

## 🔧 部署步骤

### 1. 创建 VPC 和交换机

```bash
# 使用阿里云 CLI
aliyun vpc CreateVpc \
  --RegionId cn-hangzhou \
  --VpcName k8s-vpc \
  --CidrBlock 172.16.0.0/16

aliyun vpc CreateVSwitch \
  --RegionId cn-hangzhou \
  --ZoneId cn-hangzhou-a \
  --VpcId vpc-xxx \
  --VSwitchId vsw-xxx \
  --CidrBlock 172.16.1.0/24
```

### 2. 创建 ECS 实例

```bash
# 创建 Master 节点
for i in 1 2 3; do
  aliyun ecs CreateInstance \
    --RegionId cn-hangzhou \
    --ImageId centos_7_9_x64_20G_alibase_20210420.vhd \
    --InstanceType ecs.c2.large \
    --VSwitchId vsw-xxx \
    --SecurityGroupId sg-xxx \
    --InstanceName k8s-master-0${i} \
    --InternetChargeType PayByTraffic \
    --InternetMaxBandwidthOut 5
done

# 创建 Worker 节点
for i in 1 2 3; do
  aliyun ecs CreateInstance \
    --RegionId cn-hangzhou \
    --ImageId centos_7_9_x64_20G_alibase_20210420.vhd \
    --InstanceType ecs.c2.xlarge \
    --VSwitchId vsw-xxx \
    --SecurityGroupId sg-xxx \
    --InstanceName k8s-worker-0${i} \
    --InternetChargeType PayByTraffic \
    --InternetMaxBandwidthOut 10
done
```

### 3. 创建 SLB 负载均衡

```bash
# 创建 SLB 实例
aliyun slb CreateLoadBalancer \
  --RegionId cn-hangzhou \
  --LoadBalancerName k8s-api-slb \
  --AddressType intranet \
  --VpcId vpc-xxx \
  --VSwitchId vsw-xxx

# 添加 TCP 监听 (6443)
aliyun slb CreateLoadBalancerTCPListener \
  --LoadBalancerId lb-xxx \
  --ListenerPort 6443 \
  --BackendServerPort 6443 \
  --HealthCheck connect \
  --HealthyThreshold 3 \
  --UnhealthyThreshold 3 \
  --ConnectTimeout 5 \
  --IdleTimeout 900

# 添加后端服务器
for ip in 172.16.1.11 172.16.1.12 172.16.1.13; do
  aliyun slb AddBackendServers \
    --LoadBalancerId lb-xxx \
    --BackendServers "[{\"ServerId\":\"${instance_id}\",\"Weight\":100}]"
done
```

### 4. 系统初始化

```bash
# 所有节点执行
#!/bin/bash

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 关闭 Swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# 配置内核参数
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF

sysctl -p /etc/sysctl.d/k8s.conf

# 加载模块
modprobe overlay
modprobe br_netfilter
echo "br_netfilter" >> /etc/modules-load.d/k8s.conf
echo "overlay" >> /etc/modules-load.d/k8s.conf

# 配置 hosts
cat >> /etc/hosts <<EOF
172.16.1.11 k8s-master-01
172.16.1.12 k8s-master-02
172.16.1.13 k8s-master-03
172.16.1.21 k8s-worker-01
172.16.1.22 k8s-worker-02
172.16.1.23 k8s-worker-03
EOF

# 同步时间
yum install -y chrony
systemctl enable chronyd
systemctl start chronyd
```

### 5. 安装 Docker 和 Kubelet

```bash
# 安装 Docker
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io

# 配置 Docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

systemctl enable docker
systemctl start docker

# 配置 Kubernetes YUM 源
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装 kubelet kubeadm kubectl
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet
```

### 6. 初始化集群

```bash
# 在第一个 Master 节点执行
kubeadm init \
  --control-plane-endpoint "172.16.1.100:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --kubernetes-version v1.21.0

# 保存 join 命令
kubeadm token create --print-join-command

# 获取证书密钥
kubeadm init phase upload-certs --upload-certs
```

### 7. 加入其他节点

```bash
# 其他 Master 节点加入
kubeadm join 172.16.1.100:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx \
  --control-plane \
  --certificate-key xxxxx

# Worker 节点加入
kubeadm join 172.16.1.100:6443 \
  --token xxxxx \
  --discovery-token-ca-cert-hash sha256:xxxxx
```

### 8. 配置网络插件 (Calico)

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 验证 Pod 状态
kubectl get pods -n kube-system -o wide
```

### 9. 配置阿里云云盘存储类

```yaml
# storage-class-aliyun.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-ssd
provisioner: disk.csi.alibabacloud.com
parameters:
  type: cloud_ssd
  regionId: cn-hangzhou
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-disk-efficiency
provisioner: disk.csi.alibabacloud.com
parameters:
  type: cloud_efficiency
  regionId: cn-hangzhou
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storage-class-aliyun.yaml
kubectl patch storageclass alicloud-disk-ssd -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## ✅ 验证集群

```bash
# 查看节点状态
kubectl get nodes -o wide

# 查看组件状态
kubectl get componentstatuses

# 测试部署应用
kubectl run nginx --image=nginx:1.19 --port=80
kubectl expose pod nginx --type=LoadBalancer --port=80

# 查看 SLB 状态
aliyun slb DescribeLoadBalancers --LoadBalancerId lb-xxx
```

## 📚 参考资料

- [阿里云 ACK 文档](https://help.aliyun.com/product/44879.html)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
