# 项目九：通过 Rook 部署 Ceph 分布式存储

## 📋 项目概述

Rook 是云原生存储编排工具，将 Ceph 等存储系统自动化部署到 Kubernetes 集群。本项目学习使用 Rook Operator 在 K8s 上部署生产级 Ceph 存储集群。

## 🎯 学习目标

- 理解 Rook Operator 架构
- 掌握 Ceph 核心组件原理
- 部署 Rook-Ceph 集群
- 创建存储池和 StorageClass
- 配置监控和告警

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌─────────────┐                                        │
│  │ Rook        │                                        │
│  │ Operator    │                                        │
│  └──────┬──────┘                                        │
│         │                                                │
│  ┌──────▼──────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Ceph        │  │ Ceph         │  │ Ceph         │   │
│  │ Monitor     │  │ Manager      │  │ OSD          │   │
│  │ (mon)       │  │ (mgr)        │  │ (osd)        │   │
│  └─────────────┘  └──────────────┘  └──────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Ceph Storage Pool                    │   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐         │   │
│  │  │ OSD1 │  │ OSD2 │  │ OSD3 │  │ OSD4 │         │   │
│  │  │100GB │  │100GB │  │100GB │  │100GB │         │   │
│  │  └──────┘  └──────┘  └──────┘  └──────┘         │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 📦 核心组件

| 组件 | 说明 | 副本数 |
|------|------|-------|
| Monitor (mon) | 集群元数据和状态 | 3 |
| Manager (mgr) | 监控和管理 | 1+ |
| OSD | 对象存储守护进程 | N |
| MDS | 元数据服务器 (CephFS) | 可选 |
| RGW | 对象存储网关 | 可选 |

## 🔧 部署步骤

### 1. 前置要求

```bash
# Kubernetes 1.19+
kubectl version

# 至少 3 个节点，每个节点有空白磁盘
lsblk

# 安装 Helm 3
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### 2. 克隆 Rook 仓库

```bash
git clone --single-branch --branch v1.8.0 https://github.com/rook/rook.git
cd rook/cluster/examples/kubernetes/ceph
```

### 3. 创建 Rook Operator

```bash
# 创建命名空间和 CRD
kubectl create -f crds.yaml -f common.yaml -f operator.yaml

# 验证 Operator 状态
kubectl get pods -n rook-ceph
```

### 4. 创建 Ceph 集群

```yaml
# cluster.yaml 关键配置
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v16.2.6
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 1
    allowMultiplePerNode: false
  storage:
    useAllNodes: true
    useAllDevices: true
    deviceFilter: ""
    config:
      osdsPerDevice: "1"
      encryptedDevice: "false"
  network:
    provider: host
  dashboard:
    enabled: true
    ssl: false
  monitoring:
    enabled: true
    rulesNamespace: rook-ceph
  crashCollector:
    disable: false
```

```bash
kubectl create -f cluster.yaml
```

### 5. 验证集群状态

```bash
# 查看 Ceph 集群状态
kubectl get cephcluster -n rook-ceph

# 查看 Ceph 状态
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status

# 查看 OSD 状态
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd status

# 查看 Monitor 状态
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph mon stat
```

### 6. 创建存储池

```bash
# 创建复制池 (3 副本)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool create replicapool 32 32 replicated
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool set replicapool size 3

# 创建纠删码池 (EC 4+2)
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd erasure-code-profile set ecprofile k=4 m=2
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- \
  ceph osd pool create ecpool 32 32 erasure ecprofile
```

### 7. 创建 StorageClass

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - discard
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: myfs
  pool: myfs-data0
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass.yaml
```

### 8. 测试动态存储

```yaml
# pvc-test.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
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
        volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: ceph-pvc
```

```bash
kubectl apply -f pvc-test.yaml
kubectl get pvc,pv
```

## 📊 监控配置

```bash
# 启用 Prometheus 监控
kubectl apply -f monitoring/prometheus.yaml
kubectl apply -f monitoring/grafana.yaml

# 访问 Grafana
kubectl port-forward svc/grafana 3000:3000 -n rook-ceph
```

## 📚 参考资料

- [Rook 官方文档](https://rook.io/docs/)
- [Ceph 官方文档](https://docs.ceph.com/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
