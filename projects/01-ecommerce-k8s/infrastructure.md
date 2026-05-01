# 电商平台 K8s 高可用 - 基础设施自动化

## Terraform 基础设施即代码

### 变量定义

```hcl
# variables.tf

variable "region" {
  description = "阿里云区域"
  type        = string
  default     = "cn-hangzhou"
}

variable "cluster_name" {
  description = "K8s 集群名称"
  type        = string
  default     = "ecommerce-production"
}

variable "master_count" {
  description = "Master 节点数量"
  type        = number
  default     = 3
}

variable "worker_count" {
  description = "Worker 节点数量"
  type        = number
  default     = 15
}

variable "worker_instance_type" {
  description = "Worker 节点规格"
  type        = string
  default     = "ecs.c7.4xlarge" # 16C64G
}

variable "worker_disk_size" {
  description = "Worker 系统盘大小 (GB)"
  type        = number
  default     = 100
}

variable "worker_disk_type" {
  description = "Worker 系统盘类型"
  type        = string
  default     = "cloud_essd"
}

variable "enable_monitoring" {
  description = "是否启用 Prometheus 监控"
  type        = bool
  default     = true
}

variable "vpc_cidr" {
  description = "VPC 网段"
  type        = string
  default     = "10.0.0.0/16"
}

variable "pod_cidr" {
  description = "Pod 网段 (Flannel/Calico)"
  type        = string
  default     = "172.20.0.0/16"
}

variable "service_cidr" {
  description = "Service 网段"
  type        = string
  default     = "192.168.0.0/16"
}
```

### 网络配置

```hcl
# vpc.tf

resource "alicloud_vpc" "main" {
  vpc_name   = var.cluster_name
  cidr_block = var.vpc_cidr
}

resource "alicloud_vswitch" "master" {
  vpc_id       = alicloud_vpc.main.id
  zone_id      = "${var.region}a"
  cidr_block   = "10.0.0.0/24"
  vswitch_name = "${var.cluster_name}-master"
}

resource "alicloud_vswitch" "worker" {
  vpc_id       = alicloud_vpc.main.id
  zone_id      = "${var.region}b"
  cidr_block   = "10.0.1.0/24"
  vswitch_name = "${var.cluster_name}-worker"
}

resource "alicloud_vswitch" "middleware" {
  vpc_id       = alicloud_vpc.main.id
  zone_id      = "${var.region}a"
  cidr_block   = "10.0.2.0/24"
  vswitch_name = "${var.cluster_name}-middleware"
}

resource "alicloud_nat_gateway" "main" {
  vpc_id = alicloud_vpc.main.id
  name   = "${var.cluster_name}-nat"
  specification = "Small"
}

resource "alicloud_eip_association" "nat" {
  allocation_id = alicloud_eip.nat.id
  instance_id   = alicloud_nat_gateway.main.id
}

resource "alicloud_eip" "nat" {
  name              = "${var.cluster_name}-nat-eip"
  bandwidth         = 100
  internet_charge_type = "PayByTraffic"
}

resource "alicloud_route_table_entry" "nat" {
  route_table_id    = alicloud_vpc.main.route_table_id
  destination_cidrblock = "0.0.0.0/0"
  nexthop_id        = alicloud_nat_gateway.main.id
}
```

### K8s 集群

```hcl
# kubernetes.tf

resource "alicloud_cs_managed_kubernetes" "main" {
  name                   = var.cluster_name
  cluster_spec           = "ack.pro.small"
  worker_vswitch_id      = alicloud_vswitch.worker.id
  worker_instance_types  = [var.worker_instance_type]
  worker_instances       = [var.worker_count]
  worker_disk_size       = var.worker_disk_size
  worker_disk_category   = var.worker_disk_type
  worker_disk_performance_level = "PL1"
  password               = var.k8s_password
  enable_ssh             = true
  proxy_mode             = "ipvs"
  runtime {
    name       = "containerd"
    version    = "1.6"
  }

  node_cidr_mask_size = 24

  service_cidr = var.service_cidr

  tags = {
    Environment = "production"
    Project     = "ecommerce"
    ManagedBy   = "terraform"
  }
}

resource "alicloud_cs_kubernetes_node_pool" "high-performance" {
  cluster_id             = alicloud_cs_managed_kubernetes.main.id
  name                   = "high-perf-pool"
  vswitch_id             = alicloud_vswitch.worker.id
  instance_types         = ["ecs.g7.4xlarge"] # 16C64G for DB nodes
  instance_charge_type   = "PostPaid"
  system_disk_category   = "cloud_essd"
  system_disk_size       = 200
  system_disk_performance_level = "PL1"
  min_size               = 3
  max_size               = 10
  scaling_group_id       = alicloud_cs_managed_kubernetes.main.scaling_group_id
}

resource "alicloud_cs_kubernetes_node_pool" "application" {
  cluster_id             = alicloud_cs_managed_kubernetes.main.id
  name                   = "app-pool"
  vswitch_id             = alicloud_vswitch.worker.id
  instance_types         = ["ecs.c7.4xlarge"]
  instance_charge_type   = "PostPaid"
  system_disk_category   = "cloud_essd"
  system_disk_size       = 100
  min_size               = 5
  max_size               = 20
  scaling_group_id       = alicloud_cs_managed_kubernetes.main.scaling_group_id
}

resource "alicloud_cs_kubernetes_node_pool" "middleware" {
  cluster_id             = alicloud_cs_managed_kubernetes.main.id
  name                   = "middleware-pool"
  vswitch_id             = alicloud_vswitch.middleware.id
  instance_types         = ["ecs.r7.4xlarge"] # 16C128G for memory-intensive
  instance_charge_type   = "PostPaid"
  system_disk_category   = "cloud_essd"
  system_disk_size       = 200
  min_size               = 3
  max_size               = 8
  scaling_group_id       = alicloud_cs_managed_kubernetes.main.scaling_group_id
}
```

### 存储配置

```hcl
# storage.tf

resource "alicloud_disk" "mysql_data" {
  count              = 3
  name               = "mysql-data-${count.index}"
  size               = 500
  category           = "cloud_essd"
  performance_level  = "PL1"
  zone_id            = "${var.region}a"
  encrypted          = true
  kms_key_id         = alicloud_kms_key.main.id
  tags = {
    Application = "mysql"
    Environment = "production"
  }
}

resource "alicloud_nas_file_system" "shared" {
  file_system_name = "${var.cluster_name}-nas"
  storage_type     = "standard"
  protocol_type    = "NFS"
  zone_id          = "${var.region}a"
}

resource "alicloud_nas_mount_target" "shared" {
  file_system_id = alicloud_nas_file_system.shared.id
  vswitch_id     = alicloud_vswitch.worker.id
  mount_point    = "/shared"
  access_group_name = "DEFAULT_GROUP"
}
```

### 输出

```hcl
# outputs.tf

output "cluster_id" {
  value = alicloud_cs_managed_kubernetes.main.id
}

output "cluster_endpoint" {
  value = alicloud_cs_managed_kubernetes.main.server
}

output "vpc_id" {
  value = alicloud_vpc.main.id
}

output "nat_eip" {
  value = alicloud_eip.nat.ip_address
}
```

## Kubernetes 部署清单

### Namespace 规划

```yaml
# namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    env: production
    team: platform
---
apiVersion: v1
kind: Namespace
metadata:
  name: middleware
  labels:
    env: production
    team: infrastructure
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    env: production
    team: sre
```

### 核心服务配置

```yaml
# frontend-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: ecommerce
  labels:
    app: frontend
    tier: web
spec:
  replicas: 6
  selector:
    matchLabels:
      app: frontend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: frontend
    spec:
      nodeSelector:
        role: application
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: frontend
              topologyKey: kubernetes.io/hostname
      containers:
      - name: frontend
        image: registry.ecommerce.com/frontend:2.3.1
        ports:
        - containerPort: 3000
        env:
        - name: API_GATEWAY_URL
          value: "http://api-gateway.ecommerce.svc.cluster.local:8080"
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: ecommerce
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
  type: ClusterIP
```

### 数据库 StatefulSet

```yaml
# mysql-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: middleware
  labels:
    app: mysql
    tier: database
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      nodeSelector:
        role: high-performance
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-root-secret
              key: password
        - name: MYSQL_DATABASE
          value: "ecommerce"
        - name: MYSQL_USER
          value: "ecommerce_app"
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-app-secret
              key: password
        resources:
          requests:
            cpu: "2000m"
            memory: "8Gi"
          limits:
            cpu: "4000m"
            memory: "16Gi"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        readinessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - 127.0.0.1
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - 127.0.0.1
          initialDelaySeconds: 60
          periodSeconds: 15
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "alicloud-disk-essd"
      resources:
        requests:
          storage: 500Gi
```

### Ingress 配置

```yaml
# ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.ecommerce.com
    - www.ecommerce.com
    secretName: ecommerce-tls
  rules:
  - host: api.ecommerce.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
  - host: www.ecommerce.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

---

*电商平台 K8s 高可用 - 基础设施自动化 | ClawsOps ⚙️*