# 电商平台 K8s 高可用 - 部署详细步骤

## 📋 环境准备

### 操作系统要求

```bash
# 所有节点执行

# CentOS 7/8
cat /etc/redhat-release

# 内核版本
uname -r

# 要求 >= 3.10
```

---

## 🔧 Kubernetes 集群安装

### Master 节点初始化

#### 1. 生成 Kubeadm Config

```yaml
# kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.24.0
controlPlaneEndpoint: "192.168.1.10:6443"

networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"

controllerManager:
  extraArgs:
    node-cidr-mask-size: "24"

scheduler: {}

# 镜像仓库配置
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
```

#### 2. 初始化管理平面

```bash
# master-01

# 加载内核模块
modprobe overlay
modprobe br_netfilter

# 配置系统参数
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

# 初始化集群
kubeadm init --config kubeadm-config.yaml \
  --upload-certs \
  --certificate-key abcdef1234567890abcdef1234567890

# 备份证书密钥
cat ~/pki/ca.key > ca.key
```

#### 3. 配置 kubectl

```bash
# 所有 master 节点执行

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 4. 其他 Master 加入

```bash
# master-02, master-03 执行

kubeadm join 192.168.1.10:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key abcdef1234567890abcdef1234567890
```

---

## 🖥️ Worker 节点加入

### 批量部署脚本

```bash
#!/bin/bash
# join-workers.sh

WORKERS="worker-01 worker-02 worker-03 worker-04 worker-05 worker-06 worker-07 worker-08 worker-09 worker-10"
IP_FILE="/root/workers.csv"

for worker in $WORKERS; do
  # 从文件读取 IP
  ip=$(grep "^$worker," $IP_FILE | cut -d',' -f2)
  
  echo "Joining worker: $worker ($ip)"
  
  ssh $user@$ip << 'EOF'
  modprobe overlay
  modprobe br_netfilter
  sysctl --system
  
  yum install -y docker-ce containerd.io kubeadm kubelet kubectl
  
  systemctl enable docker
  systemctl start docker
  systemctl enable kubelet
  systemctl start kubelet
  EOF

  # 在 master 上添加 taint 容忍 (业务 Pod 自动调度)
  kubectl label nodes $worker environment=production
  
  # 验证
  sleep 10
  kubectl get nodes | grep $worker
done
```

---

## 🌐 网络插件安装

### Calico 配置

```yaml
# calico-tigera-operator.yaml

apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  variant: Calico
  version: v3.23
  controlPlaneReplicas: 2
  
  # Calico 网络配置
  calicoNetwork:
    # 需要支持 VxLAN
    bgp: Disabled
    ipPools:
    - cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
      
    mtu: 1440
    
    # IPIP Mode (可选)
    # nodeToNodeEncryption: IPIP
```

```bash
# 安装 Calico
kubectl apply -f https://docs.projectcalico.org/v3.23/manifests/tigera-operator.yaml
kubectl apply -f calico-tigera-operator.yaml

# 验证
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get node -o wide
```

---

## 📊 监控系统部署

### Prometheus Stack 部署

#### 1. 存储类配置

```yaml
# local-storage.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

#### 2. 部署 Prometheus

```bash
# 使用 Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-ssd \
  --set grafana.adminPassword=admin123
```

#### 3. Grafana Dashboard

```bash
# 导入 Dashboard ID

# 总览面板
export PROMETHEUS_IP=<your-prometheus-ip>
curl http://$PROMETHEUS_IP/api/grafana/dashboards/import \
  -H "Authorization: Bearer admin123" \
  -F dashboard=@kubernetes-cluster.json \
  -F folderId=1
```

---

## 📝 日志系统部署

### Loki 部署

```bash
# Loki + Promtail 一键部署

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace logging \
  --create-namespace \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set promtail.enabled=true
```

### Fluentbit DaemonSet (替代方案)

```yaml
# fluent-bit-k8s-events.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-parser
  namespace: kube-system
data:
  parser.conf: |
    [PARSER]
        Name   json_log_parser
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```

---

## 🎯 应用部署配置

### Namespace 规划

```yaml
# namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: ecommerce
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    team: ecommerce
---
apiVersion: v1
kind: Namespace
metadata:
  name: middleware
  labels:
    component: middleware
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    component: monitoring
```

```bash
kubectl apply -f namespaces.yaml
```

---

## 🛠️ 部署脚本集

### Nginx Ingress Controller

```yaml
# ingress-controller.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/managed-by: kustomize
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
  replicas: 3
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                - ingress-nginx
            topologyKey: "kubernetes.io/hostname"
      
      containers:
      - name: controller
        image: registry.k8s.io/ingress-nginx/controller:v1.5.1
        args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx
        - --election-id=ingress-nginx-leader
        - --controller-class=k8s.io/ingress-nginx
        - --ingress-class=nginx
        ports:
        - containerPort: 80
          protocol: TCP
        - containerPort: 443
          protocol: TCP
        resources:
          limits:
            cpu: 2
            memory: 4Gi
          requests:
            cpu: 500m
            memory: 512Mi
```

---

## 🔍 健康检查

### 集群健康检查

```bash
#!/bin/bash
# cluster-health.sh

echo "=== K8s Cluster Health Check ==="

# 节点状态
kubectl get nodes -o wide

# 所有 Pod 状态
kubectl get pods -A | awk 'NR==1 || $3!="Running" || $4 !~ /^[0-9]+\/[0-9]+$/'

# 资源使用情况
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -10

# API Server 响应时间
time kubectl get pods --raw='/healthz' > /dev/null 2>&1

# Etcd 延迟
etcdctl endpoint status --cluster --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --write-out=table

# 关键服务
kubectl get deployment,pods -n kube-system --show-labels
```

---

## 📚 参考文档

- [Kubernetes 官方文档](https://kubernetes.io/docs/home/)
- [Calico 文档](https://docs.projectcalico.org/networking/calico)
- [Prometheus 文档](https://prometheus.io/docs/introduction/overview/)
- [Loki 文档](https://grafana.com/docs/loki/latest/)

---

*电商平台 K8s 高可用 - 部署步骤 | ClawsOps ⚙️*