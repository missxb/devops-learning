# 项目十九：LB+Keepalived 高可用负载均衡

## 📋 项目概述

使用 Nginx/HAProxy 配合 Keepalived 实现高可用负载均衡架构，确保服务持续可用。

## 🎯 学习目标

- 理解 VRRP 协议原理
- 配置 Keepalived 主备切换
- 部署 Nginx 负载均衡
- 实施健康检查
- 故障转移测试

## 🏗️ 架构设计

```
                    ┌─────────────────┐
                    │     Clients     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Virtual IP    │
                    │  192.168.1.100  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────▼────────┐    │    ┌────────▼────────┐
     │    Master       │    │    │    Backup       │
     │   Keepalived    │    │    │   Keepalived    │
     │   Nginx/LB      │    │    │   Nginx/LB      │
     │  192.168.1.11   │    │    │  192.168.1.12   │
     │   Priority 100  │    │    │   Priority 90   │
     └────────┬────────┘    │    └────────┬────────┘
              │             │             │
              └─────────────┼─────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────▼────┐ ┌──────▼─────┐ ┌─────▼────────┐
     │  Backend 1  │ │  Backend 2 │ │  Backend N   │
     │  192.168.2.1│ │192.168.2.2 │ │ 192.168.2.N │
     └─────────────┘ └────────────┘ └──────────────┘
```

## 🔧 Keepalived 配置

### 1. Master 节点配置

```bash
# 安装 Keepalived
yum install -y keepalived

# 配置文件 /etc/keepalived/keepalived.conf
vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight -20
    fall 3
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    virtual_ipaddress {
        192.168.1.100/24 dev eth0 label eth0:1
    }
    
    track_script {
        chk_nginx
    }
    
    notify_master "/etc/keepalived/notify_master.sh"
    notify_backup "/etc/keepalived/notify_backup.sh"
    notify_fault "/etc/keepalived/notify_fault.sh"
}
```

### 2. Backup 节点配置

```bash
# /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    
    virtual_ipaddress {
        192.168.1.100/24 dev eth0 label eth0:1
    }
    
    track_script {
        chk_nginx
    }
}
```

### 3. 健康检查脚本

```bash
#!/bin/bash
# /etc/keepalived/check_nginx.sh

counter=$(ps -C nginx --no-heading | wc -l)
if [ "$counter" -eq 0 ]; then
    systemctl start nginx
    sleep 2
    counter=$(ps -C nginx --no-heading | wc -l)
    if [ "$counter" -eq 0 ]; then
        exit 1
    fi
fi

exit 0
```

### 4. 通知脚本

```bash
#!/bin/bash
# /etc/keepalived/notify_master.sh

echo "Become MASTER" >> /var/log/keepalived.log
# 发送邮件通知
echo "Server $(hostname) is now MASTER" | mail -s "Keepalived Master" admin@example.com
```

## Nginx 负载均衡配置

```nginx
# /etc/nginx/nginx.conf
upstream backend {
    least_conn;
    server 192.168.2.1:80 weight=1 max_fails=3 fail_timeout=30s;
    server 192.168.2.2:80 weight=1 max_fails=3 fail_timeout=30s;
    server 192.168.2.3:80 weight=1 max_fails=3 fail_timeout=30s;
    
    keepalive 32;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Connection "";
        
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

## 🧪 故障转移测试

```bash
# 查看 VIP 状态
ip addr show eth0

# 查看 Keepalived 状态
systemctl status keepalived

# 查看 VRRP 广告
tcpdump -n -i eth0 vrrp

# 模拟 Master 故障
systemctl stop keepalived

# 在 Backup 上查看 VIP 是否漂移
ip addr show eth0 | grep 192.168.1.100
```

## 📚 参考资料

- [Keepalived 官方文档](https://www.keepalived.org/)
- [Nginx 官方文档](https://nginx.org/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
