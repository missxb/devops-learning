# 项目十八：MongoDB 高级应用

## 📋 项目概述

深入学习 MongoDB 的高级特性，包括分片集群、聚合管道、性能优化和生产运维。

## 🎯 学习目标

- 掌握 MongoDB 分片架构
- 熟练使用聚合管道
- 实施性能优化策略
- 配置备份和恢复
- 监控和故障排查

## 🏗️ 分片集群架构

```
┌─────────────────────────────────────────────────────────┐
│                    Mongos (查询路由)                      │
│                   192.168.1.10:27017                     │
└────┬─────────────────────┬─────────────────────┬────────┘
     │                     │                     │
┌────▼────┐          ┌────▼────┐          ┌────▼────┐
│ Config  │          │ Config  │          │ Config  │
│ Server  │          │ Server  │          │ Server  │
│   1     │          │   2     │          │   3     │
└─────────┘          └─────────┘          └─────────┘

┌─────────────────┐              ┌─────────────────┐
│   Shard 1       │              │   Shard 2       │
│  ┌───────────┐  │              │  ┌───────────┐  │
│  │  Primary  │  │              │  │  Primary  │  │
│  │  rs1-1    │  │              │  │  rs2-1    │  │
│  └───────────┘  │              │  └───────────┘  │
│  ┌───────────┐  │              │  ┌───────────┐  │
│  │ Secondary │  │              │  │ Secondary │  │
│  │  rs1-2    │  │              │  │  rs2-2    │  │
│  └───────────┘  │              │  └───────────┘  │
└─────────────────┘              └─────────────────┘
```

## 🔧 分片集群部署

### 1. 启动 Config Server

```bash
# 所有配置节点
mongod --configsvr --replSet csReplSet \
  --dbpath /data/configdb \
  --port 27019 \
  --bind_ip 0.0.0.0

# 初始化配置副本集
mongo --port 27019
> rs.initiate({
    _id: "csReplSet",
    configsvr: true,
    members: [
      { _id: 0, host: "config1:27019" },
      { _id: 1, host: "config2:27019" },
      { _id: 2, host: "config3:27019" }
    ]
  })
```

### 2. 启动 Shard

```bash
# 每个分片节点
mongod --shardsvr --replSet rs1 \
  --dbpath /data/shard1 \
  --port 27018 \
  --bind_ip 0.0.0.0

# 初始化分片副本集
mongo --port 27018
> rs.initiate({
    _id: "rs1",
    members: [
      { _id: 0, host: "shard1-1:27018" },
      { _id: 1, host: "shard1-2:27018" }
    ]
  })
```

### 3. 启动 Mongos

```bash
mongos --configdb csReplSet/config1:27019,config2:27019,config3:27019 \
  --port 27017 \
  --bind_ip 0.0.0.0
```

### 4. 配置分片

```javascript
// 添加分片
use admin
db.runCommand({ addShard: "rs1/shard1-1:27018,shard1-2:27018" })

// 启用数据库分片
db.adminCommand({ enableSharding: "mydb" })

// 分片集合
db.adminCommand({
  shardCollection: "mydb.users",
  key: { user_id: "hashed" }
})
```

## 📊 聚合管道

```javascript
// 聚合示例
db.orders.aggregate([
  { $match: { status: "completed", date: { $gte: ISODate("2026-01-01") } } },
  { $group: { _id: "$customer_id", total: { $sum: "$amount" }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 },
  { $lookup: { from: "customers", localField: "_id", foreignField: "_id", as: "customer" } }
])
```

## 📚 参考资料

- [MongoDB 官方文档](https://docs.mongodb.com/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
