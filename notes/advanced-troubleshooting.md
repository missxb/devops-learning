# 故障排查实战手册

## 📋 概述

故障排查是运维的核心能力。我的方法论：**先恢复再定位，信息比技术重要，复盘比追责有用**。

---

## 🧭 故障排查方法论

### 故障排查流程

```
┌─────────────────────────────────────────────────────────────┐
│                   故障排查六步法                             │
│                                                             │
│  Step 1: 快速恢复                                           │
│  ├── 优先恢复业务                                           │
│  ├── 回滚是最快方案                                         │
│  └── 不要纠结根因                                           │
│                                                             │
│  Step 2: 信息收集                                           │
│  ├── 监控数据                                               │
│  ├── 日志分析                                               │
│  ├── 用户反馈                                               │
│  └── 时间线梳理                                             │
│                                                             │
│  Step 3: 症状分析                                           │
│  ├── 服务状态                                               │
│  ├── 资源状态                                               │
│  ├── 网络状态                                               │
│  └── 配置状态                                               │
│                                                             │
│  Step 4: 定位根因                                           │
│  ├── 5 Why 分析                                            │
│  ├── 二分法定位                                             │
│  ├── 对比法验证                                             │
│  └── 排除法缩小                                             │
│                                                             │
│  Step 5: 修复实施                                           │
│  ├── 制定方案                                               │
│  ├── 评估风险                                               │
│  ├── 实施修复                                               │
│  └── 验证效果                                               │
│                                                             │
│  Step 6: 复盘改进                                           │
│  ├── 时间线记录                                             │
│  ├── 根因分析                                               │
│  ├── 改进措施                                               │
│  └── 文档沉淀                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 常用故障排查工具

### 系统层故障排查

```bash
# CPU 故障排查

# 1. 查看整体 CPU
top -c

# 2. 查看线程 CPU
top -H -p <pid>

# 3. 查看历史 CPU
sar -u 1 10

# 4. 函数热点分析
perf top -g -p <pid>

# 5. 火焰图生成
perf record -g -p <pid> -- sleep 60
perf script | stackcollapse-perf.pl | flamegraph.pl > cpu.svg
```

```bash
# 内存故障排查

# 1. 查看整体内存
free -h

# 2. 查看进程内存
ps aux --sort=-%mem | head -10

# 3. 查看进程内存详情
pmap -x <pid>

# 4. 内存泄漏检测
valgrind --tool=memcheck --leak-check=full ./app

# 5. 查看 Slab 内存
slabtop
```

```bash
# 磁盘 I/O 故障排查

# 1. 查看磁盘 IO
iostat -x -k 1 10

# 2. 查看进程 IO
iotop -oP

# 3. 查看特定进程 IO
pidstat -d -p <pid> 1 10

# 4. 查看磁盘使用
df -h

# 5. 查看目录大小
du -sh /var/*
```

```bash
# 网络故障排查

# 1. 查看连接状态
netstat -antp
ss -s

# 2. 查看流量
iftop -i eth0

# 3. 抓包分析
tcpdump -i eth0 -nn port 80

# 4. 网络延迟
ping target_ip
traceroute target_ip

# 5. DNS 解析
nslookup domain
dig domain
```

---

## 🐳 容器故障排查

### Pod 状态排查

```bash
# 1. 查看 Pod 状态
kubectl get pods -n <namespace> -o wide

# 2. 查看 Pod 详情
kubectl describe pod <pod-name> -n <namespace>

# 3. 查看 Pod 日志
kubectl logs <pod-name> -n <namespace> --tail=100

# 4. 查看前一个容器日志
kubectl logs <pod-name> -n <namespace> --previous

# 5. 进入容器
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# 6. 查看容器资源
kubectl top pod <pod-name> -n <namespace>
```

### 常见 Pod 故障状态

```
┌─────────────────────────────────────────────────────────────┐
│                   Pod 故障状态解析                           │
│                                                             │
│  状态          │ 原因                  │ 解决方案           │
│  ─────────────┼──────────────────────┼───────────────     │
│  Pending       │ 资源不足              │ 检查节点资源       │
│               │ PVC 未绑定            │ 检查 PVC/PV       │
│               │ 节点选择器不匹配      │ 检查 nodeSelector │
│               │                                                       │
│  CrashLoopBackOff│ 镜像不存在          │ 检查镜像名         │
│               │ 启动命令错误          │ 检查 entrypoint    │
│               │ 配置错误              │ 检查 ConfigMap    │
│               │ 权限不足              │ 检查 RBAC         │
│               │                                                       │
│  ImagePullBackOff│ 镜像不存在          │ 检查镜像仓库       │
│               │ 认证失败              │ 检查 secret        │
│               │ 网络不通              │ 检查网络           │
│               │                                                       │
│  FailedScheduling│ CPU/内存不足        │ 检查资源请求       │
│               │ 节点不健康            │ 检查节点状态       │
│               │ 污点容忍              │ 检查 tolerations   │
│               │                                                       │
│  Evicted      │ 节点资源不足          │ 检查节点资源       │
│               │ 镜像垃圾回收          │ 调整 GC 参数       │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ 数据库故障排查

### MySQL 故障排查

```sql
# 1. 查看连接数
SHOW PROCESSLIST;
SHOW STATUS LIKE 'Threads_connected';

# 2. 查看锁等待
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

# 3. 查看慢查询
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 10;

# 4. 查看死锁
SHOW ENGINE INNODB STATUS;

# 5. 查看复制状态
SHOW SLAVE STATUS;
```

### Redis 故障排查

```bash
# 1. 查看基本信息
redis-cli INFO

# 2. 查看内存使用
redis-cli INFO memory

# 3. 查看慢日志
redis-cli SLOWLOG GET 10

# 4. 查看大 Key
redis-cli --bigkeys

# 5. 查看连接数
redis-cli CLIENT LIST
```

---

## 🌐 Web 服务故障排查

### Nginx 故障排查

```bash
# 1. 查看进程状态
ps aux | grep nginx
systemctl status nginx

# 2. 查看连接数
netstat -antp | grep nginx

# 3. 查看日志
tail -f /var/log/nginx/error.log
tail -f /var/log/nginx/access.log

# 4. 测试配置
nginx -t

# 5. 查看状态
nginx_status=$(curl -s http://localhost/nginx_status)
echo $nginx_status
```

---

## 🔍 故障排查案例

### 案例 01：服务启动失败

```bash
# 症状：Pod 状态 CrashLoopBackOff

# 排查步骤：
kubectl logs myapp-pod --previous

# 发现错误：
Error: Config file not found: /app/config.yaml

# 定位根因：
kubectl describe pod myapp-pod | grep -A 10 Events

# 发现 ConfigMap 挂载失败

# 修复：
kubectl get configmap app-config
kubectl apply -f configmap.yaml
kubectl delete pod myapp-pod  # 重新创建
```

### 案例 02：CPU 使用率异常

```bash
# 症状：CPU 使用率 100%

# 排查步骤：
top -H -p <pid>

# 发现线程 CPU 高
perf top -g -p <pid>

# 发现热点函数：json_decode()

# 定位根因：
# 日志中有大量 JSON 解析调用
# 某个接口返回超大 JSON

# 修复：
# 优化接口，减少返回数据量
```

### 案例 03：网络延迟高

```bash
# 症状：API 响应慢，延迟 > 5s

# 排查步骤：
curl -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTransfer: %{time_starttransfer}s\nTotal: %{time_total}s\n" http://api.example.com

# 发现 DNS 解析慢 (> 2s)

# 定位根因：
dig api.example.com
cat /etc/resolv.conf

# 发现 DNS Server 配置错误

# 修复：
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

---

## 📋 故障排查 Checklist

```markdown
# 故障排查 Checklist

## 服务故障
- [ ] 检查服务状态 (systemctl)
- [ ] 检查日志 (journalctl/logs)
- [ ] 检查进程
- [ ] 检查端口
- [ ] 检查资源 (CPU/内存)

## 网络故障
- [ ] 检查连接 (netstat/ss)
- [ ] 检查防火墙 (iptables/ufw)
- [ ] 检查 DNS (nslookup/dig)
- [ ] 检查延迟
- [ ] 抓包分析 (tcpdump)

## 数据库故障
- [ ] 检查进程 (systemctl)
- [ ] 检查连接数
- [ ] 检查锁等待 (INNODB_LOCKS)
- [ ] 检查慢查询
- [ ] 检查复制状态

## 容器故障
- [ ] 检查 Pod 状态
- [ ] 检查容器日志
- [ ] 检查资源配置
- [ ] 检查事件 (Events)
- [ ] 检查健康检查
```

---

## 💡 故障排查心得

1. **先恢复再定位** - 用户能用比什么都重要
2. **回滚不丢人** - 硬撑才丢人
3. **信息比技术重要** - 每 15 分钟同步进展
4. **二分法定位** - 缩小范围比盲目尝试快
5. **对比法验证** - 和正常环境对比找差异
6. **文档要记录** - 每次故障都是教材

---

*故障排查实战手册 | 先恢复再定位*