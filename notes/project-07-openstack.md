# 项目七：OpenStack（Rocky 版）集群部署

## 📋 项目概述

OpenStack 是一个开源的云计算管理平台，提供 IaaS 服务。本项目基于 Rocky 版本部署完整的 OpenStack 云平台，包含控制节点、计算节点和网络节点。

## 🎯 学习目标

- 理解 OpenStack 核心组件架构
- 掌握数据库和消息队列配置
- 部署 Keystone 认证服务
- 部署 Glance、Nova、Neutron 等核心服务
- 配置 Horizon 仪表板
- 实施安全组和网络隔离

## 🏗️ 架构设计

```
┌──────────────────────────────────────────────────────┐
│              Load Balancer / VIP                     │
│               192.168.1.100                          │
└──────────────────┬───────────────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌───────┐   ┌───────┐   ┌───────┐
│Ctrl01 │   │Ctrl02 │   │Ctrl03 │
│(主)    │   │       │   │       │
└───┬───┘   └───┬───┘   └──────┘
    │           │           │
    │  MySQL    │  RabbitMQ │  Memcached
    │  (Galera) │  Cluster  │
    └───────────┼───────────┘
                │
    ┌───────────┼───────────┐
    ▼           ▼           ▼
┌───────┐  ┌───────┐  ┌───────┐
│Net01  │  │Comp01 │  │Comp02 │
│网络节点│  │计算节点│  │计算节点│
└───────┘  └───────┘  └───────┘
```

## 📦 组件清单

| 组件 | 服务 | 说明 |
|------|------|------|
| Keystone | identity | 统一认证 |
| Glance | image | 镜像管理 |
| Nova | compute | 计算资源管理 |
| Neutron | network | 网络管理 |
| Cinder | volume | 块存储 |
| Swift | object | 对象存储 |
| Horizon | dashboard | Web 仪表板 |
| Placement | placement | 资源调度 |

## 🔧 环境准备

### 1. 服务器规划

| 角色 | 主机名 | IP | CPU | RAM | Disk |
|------|--------|-----|-----|-----|------|
| 控制节点1 | ctrl01 | 10.0.0.11 | 4C | 8G | 100G |
| 控制节点2 | ctrl02 | 10.0.0.12 | 4C | 8G | 100G |
| 控制节点3 | ctrl03 | 10.0.0.13 | 4C | 8G | 100G |
| 网络节点 | net01 | 10.0.0.21 | 2C | 4G | 50G |
| 计算节点1 | comp01 | 10.0.0.31 | 8C | 16G | 200G |
| 计算节点2 | comp02 | 10.0.0.32 | 8C | 16G | 200G |

管理网络: 10.0.0.0/24
外部网络: 192.168.100.0/24
隧道网络: 172.16.0.0/16

### 2. 系统初始化

```bash
#!/bin/bash
# 所有节点执行

# 1. 关闭防火墙和 SELinux
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 2. 同步时间
yum install chrony -y
systemctl enable chronyd
systemctl start chronyd
chronyc tracking

# 3. 配置 Hosts
cat >> /etc/hosts <<EOF
10.0.0.11 ctrl01
10.0.0.12 ctrl02
10.0.0.13 ctrl03
10.0.0.21 net01
10.0.0.31 comp01
10.0.0.32 comp02
EOF

# 4. 配置 YUM 源
yum install -y centos-release-openstack-rocky
yum update -y
```

## 🗄️ 一、数据库部署（MySQL Galera）

### 1. 安装 MariaDB Galera Cluster

```bash
# 在所有控制节点执行
yum install -y mariadb-galera-server mariadb-galera-common rsync python2-PyMySQL

# 创建配置文件
mkdir -p /etc/mysql/conf.d
cat > /etc/my.cnf.d/galera.cnf <<EOF
[mariadb]
wsrep_on=ON
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_name=openstack-cluster
wsrep_cluster_address=gcomm://ctrl01,ctrl02,ctrl03
wsrep_node_name=ctrl01
wsrep_node_address=10.0.0.11
wsrep_sst_method=rsync
wsrep_sst_auth=sstuser:password
innodb_flush_log_at_trx_commit=1
innodb_buffer_pool_size=2G
bind-address=0.0.0.0

[embedded]
mixture=force

[galera]
wsrep_notify_cmd=/usr/bin/wsrep_notify
wsrep_slave_threads=8

[mysqld]
max_connections=1500
innodb_file_per_table=ON
innodb_autoinc_lock_mode=2
log_error=/var/log/mariadb/mariadb.log
EOF
```

### 2. 启动首个节点并初始化

```bash
# 在 ctrl01 上首次启动（带 --wsrep-new-cluster）
galera_new_cluster

# 登录并创建 OpenStack 用户和数据库
mysql -u root -e "
CREATE DATABASE keystone;
CREATE DATABASE glance;
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE neutron;
CREATE DATABASE cinder;
CREATE DATABASE placement;

GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY 'db_pass';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'db_pass';

CREATE USER 'sstuser'@'localhost' IDENTIFIED BY 'password';
GRANT RELOAD, LOCK, PROCESS, REPLICATION CLIENT ON *.* TO 'sstuser'@'localhost';
FLUSH PRIVILEGES;
"
```

### 3. 启动其他节点

```bash
# 在 ctrl02 和 ctrl03 上
systemctl enable mariadb
systemctl start mariadb

# 验证集群状态
mysql -u root -e "SHOW STATUS LIKE 'wsrep_%';"
```

## 📨 二、消息队列部署（RabbitMQ）

### 1. 安装 RabbitMQ

```bash
# 所有控制节点
yum install -y rabbitmq-server
systemctl enable rabbitmq-server
systemctl start rabbitmq-server

# 创建 openstack 用户
rabbitmqctl add_user openstack password
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
rabbitmqctl set_topic_permissions openstack ".*" ".*" ".*"

# 启用 management 插件
rabbitmq-plugins enable rabbitmq_management

# 设置复制策略
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}' --apply-to queues
```

## 👤 三、Keystone 身份认证

### 1. 创建数据库和 API 端点

```bash
# 创建数据库
mysql -u root -pdb_pass -e "CREATE DATABASE keystone;"
mysql -u root -pdb_pass -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone_dbpass';"
mysql -u root -pdb_pass -e "GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone_dbpass';"

# 安装 Keystone
yum install -y openstack-keystone python2-keystoneclient httpd mod_wsgi

# 编辑 /etc/keystone/keystone.conf
[database]
connection = mysql+pymysql://keystone:keystone_dbpass@ctrl01/keystone

[token]
provider = fernet
driver = memcache

[memcache]
servers = localhost:11211

[token]
driver = cache.memcache_token_cache
token_format = uuid

[assignment]
driver = sql
```

### 2. 初始化 Keystone 数据库

```bash
su -s /bin/sh -c "keystone-manage db_sync" keystone

# 初始化 Fernet keys
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

# 引导 admin
keystone-manage bootstrap --bootstrap-password admin \
    --bootstrap-admin-url http://ctrl01:5000/v3/ \
    --bootstrap-internal-url http://ctrl01:5000/v3/ \
    --bootstrap-public-url http://ctrl01:5000/v3/ \
    --bootstrap-region-id RegionOne
```

### 3. 配置 Apache HTTP Server

```bash
cat > /etc/httpd/conf.d/wsgi-keystone.conf <<EOF
Listen 5000
<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{PROJECT_NAME}
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined
    Require all granted
</VirtualHost>

Listen 35357
<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{PROJECT_NAME}
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-admin-error.log
    CustomLog /var/log/httpd/keystone-admin-access.log combined
    Require all granted
</VirtualHost>
EOF

systemctl enable httpd
systemctl restart httpd
```

## 🖼️ 四、Glance 镜像服务

### 1. 创建数据库和用户

```bash
mysql -u root -pdb_pass -e "CREATE DATABASE glance;"
mysql -u root -pdb_pass -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance_dbpass';"
mysql -u root -pdb_pass -e "GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance_dbpass';"

# 创建 glance 用户和赋予 admin 权限
openstack user create --domain default --password glance_password glance
openstack role add --project service --user glance admin

# 创建 glance 服务实体和 API 端点
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne image public http://ctrl01:9292
openstack endpoint create --region RegionOne image internal http://ctrl01:9292
openstack endpoint create --region RegionOne image admin http://ctrl01:9292
```

### 2. 配置 Glance

```ini
# /etc/glance/glance-api.conf
[database]
connection = mysql+pymysql://glance:glance_dbpass@ctrl01/glance

[keystone_authtoken]
www_authenticate_uri = http://ctrl01:5000
auth_url = http://ctrl01:5000
memcached_servers = ctrl01:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance_password

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

### 3. 下载测试镜像

```bash
yum install -y qemu-img wget
wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

# 导入镜像到 Glance
openstack image create "cirros-x86_64" \
    --file cirros-0.4.0-x86_64-disk.img \
    --disk-format qcow2 \
    --container-format bare \
    --public
```

## 💻 五、Nova 计算服务

### 1. 配置 Nova

```ini
# /etc/nova/nova.conf
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:password@ctrl01:5672/
my_ip = 10.0.0.11
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:nova_dbpass@ctrl01/nova_api

[database]
connection = mysql+pymysql://nova:nova_dbpass@ctrl01/nova

[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://ctrl01:5000
auth_url = http://ctrl01:5000
memcached_servers = ctrl01:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova_password

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
novncproxy_base_url = http://ctrl01:6080/vnc_auto.html

[glance]
api_servers = http://ctrl01:9292

[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
user_domain_name = Default
auth_url = http://ctrl01:5000/v3
username = placement
password = placement_password
```

### 2. 启动 Nova 服务

```bash
# 同步数据库
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova

# 验证细胞单元
nova-manage cell_v2 discover_hosts --verbose

# 启动服务
for service in api scheduler conductor consoleauth novncproxy; do
    systemctl enable openstack-nova-${service}.service
    systemctl start openstack-nova-${service}.service
done
```

## 🌐 六、Neutron 网络服务

### 1. 网络节点配置

```ini
# /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:password@ctrl01:5672
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[database]
connection = mysql+pymysql://neutron:neutron_dbpass@ctrl01/neutron

[keystone_authtoken]
www_authenticate_uri = http://ctrl01:5000
auth_url = http://ctrl01:5000
memcached_servers = ctrl01:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = neutron
password = neutron_password

[nova]
auth_url = http://ctrl01:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = nova
password = nova_password

[agent]
root_helper = sudo /usr/sbin/neutron-rootwrap /etc/neutron/rootwrap.conf
```

### 2. ML2 插件配置

```ini
# /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider,broadcast

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_ipset = true
```

### 3. 创建云网络

```bash
# 管理员身份执行
source admin-openrc.sh

# 创建外部网络
openstack network create --share --external \
    --provider-physical-network broadcast \
    --provider-network-type flat provider

openstack subnet create --network provider \
    --allocation-pool start=192.168.100.100,end=192.168.100.200 \
    --dns-nameserver 8.8.8.8 --gateway 192.168.100.1 \
    --subnet-range 192.168.100.0/24 provider_subnet

# 创建租户网络
openstack network create selfnet
openstack subnet create --network selfnet \
    --subnet-range 10.0.10.0/24 selfnet-subnet

# 创建路由
openstack router create myrouter
openstack router set myrouter --external-gateway provider
openstack router add subnet myrouter selfnet-subnet
```

##  七、Horizon 仪表板

```bash
# 安装 Horizon
yum install -y openstack-dashboard

# 配置 /etc/openstack-dashboard/local_settings
OPENSTACK_HOST = "ctrl01"
ALLOWED_HOSTS = ['*']
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'ctrl01:11211',
    }
}
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

# 重启服务
systemctl restart httpd memcached
```

## ✅ 验证与日常管理

```bash
# 查看可用实例类型
openstack flavor list

# 查看可用镜像
openstack image list

# 创建密钥对
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# 设置安全组规则
openstack security group rule create --proto icmp default
openstack security group rule create --proto tcp --dst-port 22 default
openstack security group rule create --proto tcp --dst-port 80 default

# 创建实例
openstack server create --image cirros-x86_64 \
    --flavor m1.tiny --network selfnet \
    --key-name mykey test-instance

# 分配浮动 IP
openstack floating ip create provider
openstack server add floating ip test-instance <floating_ip>
```

## 📚 参考资料

- [OpenStack Rocky 安装指南](https://docs.openstack.org/rocky/install-guide-rdo/)
- [OpenStack 官方文档](https://docs.openstack.org/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
