# 项目六：基于二进制安装 Kubernetes 1.19.3

## 📋 项目概述

Kubernetes（简称 K8s）是容器编排领域的标准平台。本项目学习不依赖 kubeadm 或 minikube，而是通过手动编译下载二进制文件的方式部署一个完整的高可用 Kubernetes 集群，深入理解每个组件的作用和配置原理。

## 🎯 学习目标

- 掌握 K8s 架构和核心组件
- 理解 CA 证书签发原理
- 手动部署 etcd 高可用集群
- 部署 master 节点（apiserver/scheduler/controller-manager）
- 部署 worker 节点并加入集群
- 配置网络插件 Calico/CNI
- 搭建 DNS 服务 CoreDNS

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Load Balancer (Nginx)                      │
│                     VIP: 192.168.1.100:6443                         │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
     ┌────────▼────────┐  ┌───────▼────────┐  ┌────────▼────────┐
     │   Master Node 1 │  │ Master Node 2  │  │ Master Node 3   │
     │   k8s-master01  │  │ k8s-master02   │  │ k8s-master03    │
     │                 │  │                │  │                 │
     │ kube-apiserver  │  │ kube-apiserver │  │ kube-apiserver  │
     │ kube-scheduler  │  │ kube-scheduler │  │ kube-scheduler  │
     │ controller-mgr  │  │ controller-mgr │  │ controller-mgr  │
     │ kube-proxy      │  │ kube-proxy     │  │ kube-proxy      │
     │ etcd            │  │ etcd           │  │ etcd            │
     └────────────────┘  └───────┬────────┘  └────────┬────────┘
              │                   │                     │
              └───────────────────┼─────────────────────┘
                                  │
              ┌───────────────────┼─────────────────────┐
              │                   │                     │
     ┌────────▼────────┐  ┌───────▼────────┐  ┌────────▼────────┐
     │ Worker Node 1   │  │ Worker Node 2  │  │ Worker Node N   │
     │ k8s-node01      │  │ k8s-node02     │  │ k8s-nodeN       │
     │                 │  │                │  │                 │
     │ containerd      │  │ containerd     │  │ containerd      │
     │ kubelet         │  │ kubelet        │  │ kubelet         │
     │ kube-proxy      │  │ kube-proxy     │  │ kube-proxy      │
     │ calico          │  │ calico         │  │ calico          │
     └─────────────────┘  └────────────────┘  └─────────────────┘
```

## 📦 软件版本清单

| 组件 | 版本 | 说明 |
|------|------|------|
| Kubernetes | v1.19.3 | 当前稳定版 |
| etcd | v3.4.17 | 分布式键值存储 |
| CoreDNS | v1.7.0 | K8s DNS 服务 |
| CNI Plugins | v0.9.1 | 容器网络接口 |
| Calico | v3.18.1 | CNI 网络插件 |
| Containerd | v1.4.13 | 容器运行时 |
| Keepalived | - | VIP 管理 |
| HAProxy | - | 负载均衡 |

## 🔧 环境准备

### 1. 服务器规划

| 主机名 | IP 地址 | 角色 | 系统 |
|--------|---------|------|------|
| lb01 | 192.168.1.10 | Nginx + Keepalived | CentOS 7.9 |
| master01 | 192.168.1.11 | API Server + etcd | CentOS 7.9 |
| master02 | 192.168.1.12 | API Server + etcd | CentOS 7.9 |
| master03 | 192.168.1.13 | API Server + etcd | CentOS 7.9 |
| node01 | 192.168.1.14 | Worker Node | CentOS 7.9 |
| node02 | 192.168.1.15 | Worker Node | CentOS 7.9 |

VIP: 192.168.1.100

### 2. 系统初始化脚本

```bash
#!/bin/bash
# 在所有节点执行此脚本

# 1. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
firewall-cmd --state

# 2. 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 3. 关闭 Swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# 4. 设置主机名
hostnamectl set-hostname <hostname>

# 5. 添加 hosts 解析
cat >> /etc/hosts <<EOF
192.168.1.10   lb01
192.168.1.11   master01
192.168.1.12   master02
192.168.1.13   master03
192.168.1.14   node01
192.168.1.15   node02
EOF

# 6. 内核参数调优
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches = 524288
EOF

sysctl -p /etc/sysctl.d/k8s.conf

# 7. 加载内核模块
modprobe overlay
modprobe br_netfilter

# 8. 同步系统时间
yum install ntpdate -y
ntpdate time.windows.com
crontab -e
* */1 * * * /usr/sbin/ntpdate time.windows.com >/dev/null 2>&1
```

### 3. 安装基础工具

```bash
# 安装常用工具
yum install -y wget curl git vim net-tools nmap lrzsz tree

# 配置阿里云 YUM 源
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all && yum makecache
```

## 🔐 一、CA 证书制作

### 1. 安装 CFSSL

```bash
# 下载 cfssl
wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64
mv cfssl_1.6.4_linux_amd64 /usr/local/bin/cfssl
chmod +x /usr/local/bin/cfssl

wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64
mv cfssljson_1.6.4_linux_amd64 /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssljson

wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl-certinfo_1.6.4_linux_amd64
mv cfssl-certinfo_1.6.4_linux_amd64 /usr/local/bin/cfssl-certinfo
chmod +x /usr/local/bin/cfssl-certinfo
```

### 2. 创建 CA 配置文件

```bash
# 创建目录
mkdir -p /opt/k8s/work
cd /opt/k8s/work

# 创建 ca-config.json
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF

# 创建 ca-csr.json
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成 CA 证书和私钥
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
```

### 3. 分发证书

```bash
# 复制到所有节点
for host in master01 master02 master03 node01 node02; do
    scp ca*.pem root@${host}:/etc/kubernetes/cert/
done
```

## 💾 二、部署 etcd 集群

### 1. 下载 etcd 二进制文件

```bash
cd /opt/k8s/work
ETCD_VER=v3.4.17
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
  -o etcd-${ETCD_VER}-linux-amd64.tar.gz

tar -xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
mv etcd-${ETCD_VER}-linux-amd64/etcd* /opt/k8s/bin/
chmod +x /opt/k8s/bin/*
```

### 2. 创建 etcd 证书

```bash
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.1.11",
    "192.168.1.12",
    "192.168.1.13"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json \
  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

for host in master01 master02 master03; do
    scp etcd*.pem ca.pem root@${host}:/etc/kubernetes/cert/
done
```

### 3. 创建 etcd systemd 单元文件

```bash
cat > etcd.service <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=/var/lib/etcd \\
  --name=master01 \\
  --initial-advertise-peer-urls=http://192.168.1.11:2380 \\
  --listen-peer-urls=http://192.168.1.11:2380 \\
  --listen-client-urls=http://192.168.1.11:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls=https://192.168.1.11:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=master01=http://192.168.1.11:2380,master02=http://192.168.1.12:2380,master03=http://192.168.1.13:2380 \\
  --initial-cluster-state=new \\
  --cert-file=/etc/kubernetes/cert/etcd.pem \\
  --key-file=/etc/kubernetes/cert/etcd-key.pem \\
  --client-cert-auth=true \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/kubernetes/cert/etcd.pem \\
  --peer-key-file=/etc/kubernetes/cert/etcd-key.pem \\
  --peer-client-cert-auth=true \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

# 注意：需要在每个节点修改 --name 和对应 IP
```

### 4. 启动 etcd

```bash
# 在所有 master 节点执行
mkdir -p /var/lib/etcd
cp etcd.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

# 验证集群状态
/opt/k8s/bin/etcdctl --cacert=/etc/kubernetes/cert/ca.pem \
  --cert=/etc/kubernetes/cert/etcd.pem \
  --key=/etc/kubernetes/cert/etcd-key.pem \
  --endpoints="https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.13:2379" \
  endpoint health

/opt/k8s/bin/etcdctl --cacert=/etc/kubernetes/cert/ca.pem \
  --cert=/etc/kubernetes/cert/etcd.pem \
  --key=/etc/kubernetes/cert/etcd-key.pem \
  --endpoints="https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.13:2379" \
  cluster-health
```

## 🎛️ 三、部署 Master 组件

### 1. 下载 K8s 二进制

```bash
cd /opt/k8s/work
wget https://dl.k8s.io/v1.19.3/kubernetes-server-linux-amd64.tar.gz
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/k8s/bin/
cp kubectl /opt/k8s/bin/
chmod +x /opt/k8s/bin/*
```

### 2. 生成 kube-apiserver 证书

```bash
cat > apiserver-csr.json <<EOF
{
  "CN": "kube-apiserver",
  "hosts": [
    "127.0.0.1",
    "192.168.1.100",
    "10.0.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json \
  -profile=kubernetes apiserver-csr.json | cfssljson -bare apiserver
```

### 3. 创建 token 认证文件

```bash
BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

### 4. 配置 kube-apiserver

```bash
cat > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
After=network.target
After=etcd.service

[Service]
Type=notify
ExecStart=/opt/k8s/bin/kube-apiserver \\
  --anonymous-auth=false \\
  --basic-auth-file=/etc/kubernetes/conf/token.csv \\
  --secure-port=6443 \\
  --bind-address=192.168.1.11 \\
  --advertise-address=192.168.1.11 \\
  --allow-privileged=true \\
  --authorization-mode=Node,RBAC \\
  --client-ca-file=/etc/kubernetes/cert/ca.pem \\
  --tls-cert-file=/etc/kubernetes/cert/apiserver.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/apiserver-key.pem \\
  --etcd-servers=https://192.168.1.11:2379,https://192.168.1.12:2379,https://192.168.1.13:2379 \\
  --etcd-cafile=/etc/kubernetes/cert/ca.pem \\
  --etcd-certfile=/etc/kubernetes/cert/apiserver.pem \\
  --etcd-keyfile=/etc/kubernetes/cert/apiserver-key.pem \\
  --service-cluster-ip-range=10.0.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --kubelet-client-certificate=/etc/kubernetes/cert/apiserver.pem \\
  --kubelet-client-key=/etc/kubernetes/cert/apiserver-key.pem \\
  --log-dir=/var/log/kubernetes \\
  --logtostderr=false
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 5. 配置 kube-scheduler

```bash
cat > scheduler-kubeconfig.yml <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/cert/ca.pem
    server: https://192.168.1.11:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
preferences: {}
users: []
EOF

cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler

[Service]
ExecStart=/opt/k8s/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --leader-elect=true \\
  --kubeconfig=/etc/kubernetes/conf/scheduler-kubeconfig.yml \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 6. 配置 kube-controller-manager

```bash
cat > controller-manager-kubeconfig.yml <<EOF
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/cert/ca.pem
    server: https://192.168.1.11:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
preferences: {}
users: []
EOF

cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager

[Service]
ExecStart=/opt/k8s/bin/kube-controller-manager \\
  --address=127.0.0.1 \\
  --pod-eviction-timeout=5m0s \\
  --terminated-pod-gc-threshold=1000 \\
  --root-ca-file=/etc/kubernetes/cert/ca.pem \\
  --service-account-private-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --leader-elect=true \\
  --kubeconfig=/etc/kubernetes/conf/controller-manager-kubeconfig.yml \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 7. 启动 Master 组件

```bash
# 在所有 master 节点执行
mkdir -p /var/log/kubernetes
mkdir -p /etc/kubernetes/conf

# 复制配置文件到对应节点后执行
cp kube-apiserver.service kube-scheduler.service kube-controller-manager.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable kube-apiserver kube-scheduler kube-controller-manager
systemctl start kube-apiserver kube-scheduler kube-controller-manager

# 查看服务状态
systemctl status kube-apiserver kube-scheduler kube-controller-manager
```

## 🖥️ 四、部署 Worker 节点

### 1. 安装 containerd

```bash
# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加 Docker 仓库
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 containerd
yum install -y containerd.io
containerd --version

# 配置 containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd
```

### 2. 安装 kubelet 和 kube-proxy

```bash
cd /opt/k8s/work
cp ../kubernetes/server/bin/kubelet kube-proxy /opt/k8s/bin/
chmod +x /opt/k8s/bin/*
```

### 3. 为每个节点生成 kubeconfig

```bash
# 为 node01 生成证书和 kubeconfig
NODES=("node01=192.168.1.14" "node02=192.168.1.15")

for NODE in "${NODES[@]}"; do
    NAME=${NODE%%=*}
    IP=${NODE##*=}
    
    # 生成证书
    cat > ${NAME}-csr.json <<EOF
{
  "CN": "system:node:${NAME}",
  "hosts": ["${IP}"],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "ST": "Beijing",
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
EOF
    
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json \
      -profile=kubernetes ${NAME}-csr.json | cfssljson -bare ${NAME}
    
    # 设置集群参数
    /opt/k8s/bin/kubectl config set-cluster kubernetes \\
      --certificate-authority=/etc/kubernetes/cert/ca.pem \\
      --embed-certs=true \\
      --server=https://192.168.1.100:6443 \\
      --kubeconfig=${NAME}.kubeconfig
    
    # 设置客户端认证参数
    /opt/k8s/bin/kubectl config set-credentials system:node:${NAME} \\
      --client-certificate=/etc/kubernetes/cert/${NAME}.pem \\
      --client-key=/etc/kubernetes/cert/${NAME}-key.pem \\
      --embed-certs=true \\
      --kubeconfig=${NAME}.kubeconfig
    
    # 设置上下文参数
    /opt/k8s/bin/kubectl config set-context default \\
      --cluster=kubernetes \\
      --user=system:node:${NAME} \\
      --kubeconfig=${NAME}.kubeconfig
    
    # 选择默认上下文
    /opt/k8s/bin/kubectl config use-context default \\
      --kubeconfig=${NAME}.kubeconfig
done
```

### 4. 创建 kubelet 配置

```bash
cat > kubelet-config.yml <<EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m0s
  x509:
    clientCAFile: /etc/kubernetes/cert/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
clusterDomain: cluster.local
clusterDNS:
- 10.0.0.2
failSwapOn: false
maxPods: 110
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 15m
EOF
```

### 5. 创建 kubelet systemd 单元文件

```bash
cat > kubelet.service <<EOF
[Unit]
Description=Kubernetes Kubelet
After=containerd.service
Requires=containerd.service

[Service]
Type=notify
ExecStart=/opt/k8s/bin/kubelet \\
  --address=192.168.1.14 \\
  --port=10250 \\
  --rotate-certificates=true \\
  --cert-dir=/var/lib/kubelet/pki \\
  --hostname-override=node01 \\
  --kubeconfig=/etc/kubernetes/conf/node01.kubeconfig \\
  --config=/etc/kubernetes/conf/kubelet-config.yml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \\
  --cgroup-driver=systemd \\
  --fail-swap-on=false \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 6. 启动 Worker 节点

```bash
# 在 node01 上执行
scp node01.*.kubeconfig root@node01:/etc/kubernetes/conf/
cp kubelet.service kubelet-config.yml /etc/systemd/system/
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet

# 验证节点状态
kubectl get nodes
```

## 🌐 五、部署网络插件

### 1. 部署 Calico

```bash
# 下载 Calico YAML
wget https://raw.githubusercontent.com/projectcalico/calico/v3.18.1/manifests/calico.yaml

# 修改 Pod 网段
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' calico.yaml

# 应用配置
kubectl apply -f calico.yaml

# 等待 Calico Pod 就绪
kubectl get pods -n kube-system -w
```

### 2. 验证网络

```bash
# 创建测试 Pod
cat > nginx-deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF

kubectl apply -f nginx-deployment.yaml

# 测试网络连通性
kubectl run -it --rm test-pod --image=busybox --restart=Never -- sh
wget --spider --timeout=5 http://nginx-service
kubectl get svc,pods -o wide
```

## 🔍 六、CoreDNS 部署

### 1. 确认 CoreDNS 已部署

```bash
# 检查 CoreDNS 状态
kubectl get pods -n kube-system | grep coredns
kubectl get svc -n kube-system | grep dns
```

### 2. 测试 DNS 解析

```bash
# 进入测试 Pod
kubectl run -it --rm dns-test --image=busybox --restart=Never -- sh

# 测试内部服务解析
nslookup nginx-service
nslookup kubernetes.default.svc.cluster.local

# 测试外部域名解析
nslookup www.baidu.com
```

## ✅ 集群健康检查

```bash
# 查看所有节点状态
kubectl get nodes -o wide

# 查看所有 Pod 状态
kubectl get pods -A

# 检查关键组件
kubectl get componentstatuses  # 注意：v1.19+ 已废弃

# 检查 API Server 端点
kubectl get endpoints kube-controller-manager -n kube-system

# 测试 RBAC 权限
kubectl auth can-i create deployments --as=system:admin
kubectl auth can-i list pods -n kube-system

# 检查证书过期时间
openssl x509 -in /etc/kubernetes/cert/apiserver.pem -noout -text | grep Not
```

## 🚨 故障排查

### 常见问题

#### 1. NodeNotReady

```bash
# 检查 kubelet 日志
journalctl -u kubelet -f

# 检查网络连接
ping 10.0.0.1

# 检查 containerd 状态
crictl info
crictl ps -a
```

#### 2. Pod 无法启动

```bash
# 查看 Pod 事件
kubectl describe pod <pod-name>

# 检查网络插件
kubectl logs calico-node-xxx -n kube-system

# 清理残留网络接口
network-policy remove --interface docker0
```

#### 3. 证书问题

```bash
# 检查证书有效期
openssl x509 -in /etc/kubernetes/cert/kubelet-client-current.pem -noout -dates

# 重启组件以重新签发证书
systemctl restart kubelet
```

## 📈 性能调优建议

### 1. kubelet 参数优化

```yaml
# kubelet-config.yml
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "500Mi"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"
maxOpenFiles: 1000000
maxPods: 250
```

### 2. API Server 资源限制

```bash
# 添加资源限制到 apiserver manifest
--allocate-node-cidrs=true
--cluster-cidr=10.244.0.0/16
--requestheader-client-ca-file=/etc/kubernetes/cert/ca.pem
--proxy-client-cert-file=/etc/kubernetes/cert/front-proxy-client.pem
--proxy-client-key-file=/etc/kubernetes/cert/front-proxy-client-key.pem
```

## 📚 参考资料

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 源码分析](https://github.com/kubernetes/kubernetes)
- [Calico 文档](https://docs.projectcalico.org/)
- [Containerd 文档](https://containerd.io/docs/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
