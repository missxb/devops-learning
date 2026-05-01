# 项目十一：Mongo 4.4.2+ 副本集 + 认证部署

## 📋 项目概述

MongoDB 副本集提供数据冗余和自动故障转移能力。本项目部署 MongoDB 4.4.2+ 版本的 3 节点副本集集群，并配置基于角色的访问控制 (RBAC)。

## 🎯 学习目标

- 理解 MongoDB 副本集架构
- 掌握主从选举机制
- 配置认证和授权
- 实施备份和恢复策略
- 监控副本集状态

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    MongoDB Replica Set                   │
│                     rs0 (3 节点)                          │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│  │   Primary   │───►│  Secondary  │───►│  Secondary  │ │
│  │   (主节点)   │    │   (从节点 1)  │    │   (从节点 2)  │ │
│  │  192.168.1.11│    │ 192.168.1.12│    │ 192.168.1.13│ │
│  │  Port 27017 │    │ Port 27017  │    │ Port 27017  │ │
│  │  读写操作   │    │  只读操作   │    │  只读操作   │ │
│  └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                   │                   │       │
│         └───────────────────┼───────────────────┘       │
│                             ▼                           │
│                  ┌─────────────────┐                    │
│                  │    Oplog 同步    │                    │
│                  │  (异步复制)      │                    │
│                  └─────────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

## 📦 副本集角色

| 角色 | 说明 | 数量 |
|------|------|-----|
| Primary | 主节点，处理所有写操作 | 1 |
| Secondary | 从节点，复制数据，可读 | 1-N |
| Arbiter | 仲裁节点，只参与选举 | 可选 |

## 🔧 部署步骤

### 1. 环境准备

```bash
# 所有节点执行
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 配置 hosts
cat >> /etc/hosts <<EOF
192.168.1.11 mongo01
192.168.1.12 mongo02
192.168.1.13 mongo03
EOF

# 创建数据目录
mkdir -p /data/db
mkdir -p /data/log
chown -R mongod:mongod /data
```

### 2. 安装 MongoDB

```bash
# 创建 YUM 源
cat > /etc/yum.repos.d/mongodb-org-4.4.repo <<EOF
[mongodb-org-4.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
EOF

# 安装 MongoDB
yum install -y mongodb-org

# 启动服务
systemctl enable mongod
systemctl start mongod
systemctl status mongod
```

### 3. 配置副本集

```yaml
# /etc/mongod.conf
storage:
  dbPath: /data/db
  journal:
    enabled: true

systemLog:
  destination: file
  logAppend: true
  path: /data/log/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0

replication:
  replSetName: "rs0"

security:
  authorization: disabled  # 先关闭认证，配置副本集后再开启
```

### 4. 初始化副本集

```bash
# 在 mongo01 上执行
mongo --host mongo01 --port 27017

> rs.initiate({
    _id: "rs0",
    members: [
      { _id: 0, host: "mongo01:27017", priority: 2 },
      { _id: 1, host: "mongo02:27017", priority: 1 },
      { _id: 2, host: "mongo03:27017", priority: 1 }
    ]
  })

# 查看副本集状态
rs.status()

# 查看配置
rs.conf()

# 检查主节点
rs.isMaster()
```

### 5. 创建管理员用户

```javascript
// 在 Primary 节点执行
use admin

// 创建 root 用户
db.createUser({
  user: "root",
  pwd: "SecurePassword123!",
  roles: [
    { role: "root", db: "admin" },
    { role: "clusterAdmin", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})

// 创建应用用户
db.createUser({
  user: "appuser",
  pwd: "AppPassword123!",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// 创建只读用户
db.createUser({
  user: "readonly",
  pwd: "ReadPassword123!",
  roles: [
    { role: "read", db: "myapp" }
  ]
})
```

### 6. 启用认证

```bash
# 修改配置文件，启用认证
# /etc/mongod.conf
security:
  authorization: enabled

# 重启 MongoDB
systemctl restart mongod

# 验证登录
mongo --host mongo01 -u root -p SecurePassword123! --authenticationDatabase admin
```

## ✅ 客户端连接

### Python 连接示例

```python
from pymongo import MongoClient, ASCENDING, DESCENDING
from pymongo.errors import ConnectionFailure, AutoReconnect
import time

class MongoDBClient:
    def __init__(self, hosts, username, password, db_name):
        self.hosts = hosts
        self.username = username
        self.password = password
        self.db_name = db_name
        self.client = None
        self.db = None
        self.connect()
    
    def connect(self):
        """连接 MongoDB 副本集"""
        try:
            uri = f"mongodb://{self.username}:{self.password}@{','.join(self.hosts)}/?replicaSet=rs0&authSource=admin"
            self.client = MongoClient(
                uri,
                serverSelectionTimeoutMS=5000,
                socketTimeoutMS=45000,
                connectTimeoutMS=20000,
                retryWrites=True,
                retryReads=True
            )
            self.db = self.client[self.db_name]
            
            # 测试连接
            self.client.admin.command('ping')
            print("Connected to MongoDB successfully!")
            
        except ConnectionFailure as e:
            print(f"Connection failed: {e}")
            raise
    
    def insert_document(self, collection_name, document):
        """插入文档"""
        try:
            collection = self.db[collection_name]
            result = collection.insert_one(document)
            return result.inserted_id
        except Exception as e:
            print(f"Insert failed: {e}")
            return None
    
    def insert_many(self, collection_name, documents):
        """批量插入"""
        try:
            collection = self.db[collection_name]
            result = collection.insert_many(documents)
            return result.inserted_ids
        except Exception as e:
            print(f"Batch insert failed: {e}")
            return []
    
    def find_documents(self, collection_name, query=None, sort=None, limit=0):
        """查询文档"""
        try:
            collection = self.db[collection_name]
            cursor = collection.find(query or {})
            
            if sort:
                cursor = cursor.sort(sort)
            
            if limit > 0:
                cursor = cursor.limit(limit)
            
            return list(cursor)
        except Exception as e:
            print(f"Query failed: {e}")
            return []
    
    def update_document(self, collection_name, query, update):
        """更新文档"""
        try:
            collection = self.db[collection_name]
            result = collection.update_one(query, {'$set': update})
            return result.modified_count
        except Exception as e:
            print(f"Update failed: {e}")
            return 0
    
    def delete_documents(self, collection_name, query):
        """删除文档"""
        try:
            collection = self.db[collection_name]
            result = collection.delete_many(query)
            return result.deleted_count
        except Exception as e:
            print(f"Delete failed: {e}")
            return 0
    
    def create_index(self, collection_name, keys, unique=False):
        """创建索引"""
        try:
            collection = self.db[collection_name]
            collection.create_index(keys, unique=unique)
            print(f"Index created on {collection_name}")
        except Exception as e:
            print(f"Create index failed: {e}")
    
    def get_replica_set_status(self):
        """获取副本集状态"""
        try:
            status = self.client.admin.command('replSetGetStatus')
            return status
        except Exception as e:
            print(f"Get status failed: {e}")
            return None
    
    def close(self):
        """关闭连接"""
        if self.client:
            self.client.close()

# 使用示例
if __name__ == '__main__':
    client = MongoDBClient(
        hosts=['mongo01:27017', 'mongo02:27017', 'mongo03:27017'],
        username='appuser',
        password='AppPassword123!',
        db_name='myapp'
    )
    
    # 插入数据
    doc = {'name': '张三', 'age': 30, 'city': '上海'}
    client.insert_document('users', doc)
    
    # 批量插入
    docs = [
        {'name': f'用户{i}', 'age': 20+i, 'city': '北京'}
        for i in range(10)
    ]
    client.insert_many('users', docs)
    
    # 查询数据
    users = client.find_documents('users', {'age': {'$gt': 25}})
    for user in users:
        print(user)
    
    # 创建索引
    client.create_index('users', [('name', ASCENDING)])
    client.create_index('users', [('age', DESCENDING)])
    
    # 查看副本集状态
    status = client.get_replica_set_status()
    print(f"Primary: {status['members'][0]['name']}")
    
    client.close()
```

## 🔄 副本集管理

### 1. 添加新节点

```javascript
// 在 Primary 上执行
rs.add("mongo04:27017")

// 添加仲裁节点
rs.addArb("mongo05:27017")
```

### 2. 移除节点

```javascript
// 先将被移除节点设置为从节点
rs.stepDown()

// 在 Primary 上执行
rs.remove("mongo03:27017")
```

### 3. 手动故障转移

```javascript
// 在 Primary 上执行，强制选举
rs.stepDown(60)

// 查看新的 Primary
rs.status()
```

### 4. 备份与恢复

```bash
# 备份整个数据库
mongodump --host mongo01 -u root -p SecurePassword123! \
  --authenticationDatabase admin --out /backup/mongodb/$(date +%Y%m%d)

# 备份单个数据库
mongodump --host mongo01 -u root -p SecurePassword123! \
  --authenticationDatabase admin --db myapp \
  --out /backup/mongodb/myapp

# 恢复数据库
mongorestore --host mongo01 -u root -p SecurePassword123! \
  --authenticationDatabase admin /backup/mongodb/20260501
```

## 📊 监控指标

```javascript
// 查看副本集状态
rs.status()

// 查看复制延迟
rs.printSecondaryReplicationInfo()

// 查看操作日志
rs.printReplicationInfo()

// 查看当前操作
db.currentOp()

// 查看数据库统计
db.stats()
```

## 🚨 故障排查

### 常见问题

1. **副本集无法选举**
```javascript
// 检查网络连通性
ping mongo01
ping mongo02

// 查看日志
tail -f /var/log/mongodb/mongod.log

// 强制重新配置
cfg = rs.conf()
cfg.version = cfg.version + 1
rs.reconfig(cfg, {force: true})
```

2. **复制延迟过高**
```javascript
// 检查从节点状态
rs.printSecondaryReplicationInfo()

// 优化网络或增加资源
```

## 📚 参考资料

- [MongoDB 官方文档](https://docs.mongodb.com/)
- [MongoDB 副本集管理](https://docs.mongodb.com/manual/replication/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
