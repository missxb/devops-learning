# 大规模集群运维实战

## 📋 概述

当集群规模超过 1000 个节点、10000+ Pod 时，运维挑战完全不同。**规模带来复杂度，复杂度带来故障风险**。

---

## 🏗️ 大规模集群架构

### 集群规模分层

```
┌─────────────────────────────────────────────────────────────┐
│                    集群规模分层                              │
│                                                             │
│  Level 1: 小规模 (< 100节点)                                │
│  ├── 单集群                                                 │
│  ├── 手动运维可行                                           │
│  ├── 监控简单                                               │
│  └── 故障影响小                                             │
│                                                             │
│  Level 2: 中规模 (100-1000节点)                             │
│  ├── 单集群 + 多命名空间                                    │
│  ├── 半自动化运维                                           │
│  ├── 监控分层次                                             │
│  └── 故障影响中等                                           │
│                                                             │
│  Level 3: 大规模 (1000-10000节点)                           │
│  ├── 多集群管理                                             │
│  ├── 全自动化运维                                           │
│  ├── 监控联邦                                               │
│  └── 故障影响大                                             │
│                                                             │
│  Level 4: 超大规模 (> 10000节点)                            │
│  ├── 多地域多集群                                           │
│  ├── 平台化运维                                             │
│  ├── AI 辅助决策                                            │
│  └── 故障影响全球                                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 大规模集群挑战

### 挑战与解决方案

```
┌─────────────────────────────────────────────────────────────┐
│                  大规模集群挑战矩阵                          │
│                                                             │
│  挑战                │ 影响              │ 解决方案          │
│  ───────────────────┼──────────────────┼───────────────    │
│  API Server 压力     │ 调度延迟         │ 多副本+缓存       │
│  Etcd 性能瓶颈       │ 写入延迟         │ 分片+优化         │
│  网络路由表过大      │ 网络延迟         │ 网络分片          │
│  DNS 查询压力大      │ 解析延迟         │ CoreDNS 多副本    │
│  监控数据量大        │ 存储爆炸         │ 采样+聚合         │
│  日志量爆炸          │ 存储爆炸         │ Loki+采样         │
│  配置推送慢          │ 更新延迟         │ 增量推送          │
│  调度效率低          │ 启动慢           │ 调度器优化        │
└─────────────────────────────────────────────────────────────┘
```

---

## 📊 控制平面优化

### API Server 优化

```yaml
# kube-apiserver 配置优化

# 1. 多副本部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-apiserver
spec:
  replicas: 3  # 至少 3 副本
  template:
    spec:
      containers:
      - name: kube-apiserver
        command:
        - kube-apiserver
        - --max-requests-inflight=1000         # 最大并发请求
        - --max-mutating-requests-inflight=500 # 最大写请求
        - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        - --enable-aggregator-routing=true     # 聚合路由
        
        # 缓存配置
        - --watch-cache-size=100               # Watch 缓存大小
        
        # 限流配置
        - --apiserver-qps=5000                 # QPS 上限
        - --apiserver-burst=10000              # 突发上限
```

### Etcd 优化

```yaml
# etcd 性能优化

# 1. 硬件要求 (大规模)
# SSD: IOPS > 5000
# 内存: > 16GB
# 网络: 万兆网卡

# 2. 参数优化
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: etcd
spec:
  template:
    spec:
      containers:
      - name: etcd
        command:
        - etcd
        - --quota-backend-bytes=8589934592      # 数据大小上限 8GB
        - --snapshot-count=10000                # 快照间隔
        - --heartbeat-interval=100              # 心跳间隔
        - --election-timeout=1000               # 选举超时
        - --max-request-bytes=15728640          # 最大请求 15MB
        - --auto-compaction-retention=1         # 自动压缩
        
        # 性能调优
        env:
        - name: ETCD_MAX_WALS
          value: "50"                           # 最大 WAL 文件
        - name: GOGC
          value: "100"                          # GC 优化
```

### 控制平面架构

```
┌─────────────────────────────────────────────────────────────┐
│                   多副本控制平面                              │
│                                                             │
│           ┌─────────────────────────────────────┐          │
│           │         Load Balancer               │          │
│           │         (HAProxy/Nginx)             │          │
│           └─────────────────────────────────────┘          │
│                         │                                   │
│          ┌──────────────┼──────────────┐                   │
│          ▼              ▼              ▼                   │
│     ┌─────────┐   ┌─────────┐   ┌─────────┐              │
│     │ API-1   │   │ API-2   │   │ API-3   │              │
│     │ Leader  │   │ Follower│   │ Follower│              │
│     └─────────┘   └─────────┘   └─────────┘              │
│          │              │              │                   │
│          └──────────────┼──────────────┘                   │
│                         ▼                                   │
│           ┌─────────────────────────────────────┐          │
│           │          Etcd 集群                  │          │
│           │   etcd-1  etcd-2  etcd-3            │          │
│           └─────────────────────────────────────┘          │
│                                                             │
│           ┌─────────────────────────────────────┐          │
│           │         Scheduler 多副本            │          │
│           │   scheduler-1  scheduler-2          │          │
│           └─────────────────────────────────────┘          │
│                                                             │
│           ┌─────────────────────────────────────┐          │
│           │         Controller Manager          │          │
│           │   cm-1  cm-2                        │          │
│           └─────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

---

## 🌐 网络优化

### 大规模网络挑战

```
问题：节点数 > 1000，Pod 数 > 10000
影响：
├── 路由表过大 (> 10000 条)
├── ARP 表爆炸
├── iptables 规则过多 (> 10000)
├── 网络延迟增加
└── IP 地址管理复杂

解决方案：
├── 网络插件选择 (Calico/Flannel/Cilium)
├── 路由分片
├── IP 地址池管理
└── 网络策略优化
```

### Calico 优化配置

```yaml
# calico-config.yaml

apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  # 大规模优化
  datastoreType: kubernetes
  
  # BGP 配置
  bgp:
    enabled: true
    asNumber: 64512
    
    # 路由反射器 (减少全连接)
    routeReflector:
      enabled: true
      clusterID: 1.0.0.1
      
  # IP 地址管理
  ipPool:
    cidr: 10.244.0.0/16
    blockSize: 26         # 每个 Node 分配 /26 (64 IP)
    natOutgoing: true
    
  # 性能优化
  felixConfiguration:
    iptablesRefreshInterval: 90s  # iptables 刷新间隔
    iptablesPostWriteCheckInterval: 1s
    iptablesLockTimeout: 0s
    ipsetsRefreshInterval: 90s
```

### Cilium 优化 (推荐超大规模)

```yaml
# cilium-config.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
data:
  # eBPF 替代 iptables (性能更好)
  enable-bpf-masquerade: "true"
  
  # 大规模优化
  cluster-id: "1"
  cluster-name: "cluster-prod"
  
  # IP 地址管理
  ipam: "cluster-pool"
  cluster-pool-ipv4-cidr: "10.0.0.0/8"
  cluster-pool-ipv4-mask-size: "24"
  
  # 性能优化
  enable-node-port: "true"
  enable-health-check-nodeport: "true"
  
  # 监控优化
  monitor-aggregation: "medium"
  monitor-interval: "5s"
```

---

## 📊 监控优化

### Prometheus 大规模部署

```yaml
# Prometheus 联邦架构

# 1. 分片 Prometheus (每个分片监控一部分)
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-shard-0
spec:
  shards: 4          # 4 个分片
  replicaExternalLabelName: prometheus_replica
  
  # 数据保留
  retention: 7d
  storage:
    volumeClaimTemplate:
      spec:
        resources:
          requests:
            storage: 100Gi

# 2. 联邦 Prometheus (聚合全局数据)
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-federation
spec:
  remoteWrite:
  - url: http://prometheus-shard-0:9090/api/v1/write
  - url: http://prometheus-shard-1:9090/api/v1/write
  - url: http://prometheus-shard-2:9090/api/v1/write
  - url: http://prometheus-shard-3:9090/api/v1/write
  
  # 只保留聚合数据
  retention: 30d
```

### 指标采样策略

```yaml
# 降低指标量

# 1. 降低采集频率
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-monitor
spec:
  endpoints:
  - port: metrics
    interval: 30s       # 从 15s 改为 30s
    scrapeTimeout: 10s

# 2. 指标过滤
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: metric-filter
spec:
  groups:
  - name: drop_rules
    rules:
    # Drop 高基数指标
    - record: node_cpu_seconds_total:filtered
      expr: |
        node_cpu_seconds_total 
        unless on(mode) 
        node_cpu_seconds_total{mode=~"idle|iowait"}
```

---

## 🔄 自动化运维平台

### 自动化巡检平台

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-inspector
spec:
  schedule: "0 8 * * *"  # 每天 8:00
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: inspector
            image: cluster-inspector:1.0
            command:
            - /bin/sh
            - -c
            - |
              # 节点健康检查
              kubectl get nodes --no-headers | awk '{print $1}' | while read node; do
                kubectl describe node $node | grep -E "Ready|Memory|Disk"
              done
              
              # 资源使用 Top10
              kubectl top pods --all-namespaces --sort-by=cpu | head -10
              kubectl top pods --all-namespaces --sort-by=memory | head -10
              
              # 异常 Pod
              kubectl get pods --all-namespaces --field-selector=status.phase!=Running
              
              # 镜像版本检查
              kubectl get pods --all-namespaces -o json | jq '.items[].spec.containers[].image' | sort | uniq -c
```

---

## 📋 大规模集群 Checklist

```markdown
# 大规模集群运维 Checklist

## 控制平面
- [ ] API Server 多副本 (≥3)
- [ ] Etcd SSD 存储
- [ ] Scheduler 多副本
- [ ] Controller Manager 多副本
- [ ] 限流参数配置

## 数据平面
- [ ] 网络插件选型
- [ ] DNS 多副本
- [ ] 日志采集优化
- [ ] 监控分片

## 自动化
- [ ] 自动扩缩容
- [ ] 自动巡检
- [ ] 自动告警
- [ ] 自动修复

## 文档
- [ ] 架构文档
- [ ] SOP 流程
- [ ] 故障预案
- [ ] 容量规划
```

---

*大规模集群运维实战 | 1000+ 节点经验*