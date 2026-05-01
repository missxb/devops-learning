# 项目十九：rsync/lsyncd/sersync 数据同步

## 📋 项目概述

学习 Linux 下常用的数据同步工具，实现文件实时同步和备份。

## 🎯 学习目标

- 掌握 rsync 增量同步
- 配置 lsyncd 实时同步
- 使用 sersync 监控同步
- 实施数据备份策略

## 📦 工具对比

| 工具 | 原理 | 实时性 | 适用场景 |
|------|------|--------|----------|
| rsync | 增量算法 | 定时 | 备份、迁移 |
| lsyncd | inotify+rsync | 近实时 | 文件同步 |
| sersync | inotify+rsync | 实时 | 单向同步 |

## 🔧 rsync 配置

### 1. 基础同步

```bash
# 本地同步
rsync -avz /source/ /backup/

# 远程同步 (push)
rsync -avz /source/ user@remote:/backup/

# 远程同步 (pull)
rsync -avz user@remote:/source/ /backup/

# 删除目标多余文件
rsync -avz --delete /source/ /backup/
```

### 2. 守护进程模式

```ini
# /etc/rsyncd.conf
uid = root
gid = root
use chroot = no
max connections = 10
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsync.lock
log file = /var/log/rsyncd.log

[backup]
path = /backup
comment = Backup Directory
read only = no
auth users = backup_user
secrets file = /etc/rsync.passwd
```

```bash
# 创建密码文件
echo "backup_user:password" > /etc/rsync.passwd
chmod 600 /etc/rsync.passwd

# 启动服务
rsync --daemon --config=/etc/rsyncd.conf

# 同步
rsync -avz /source/ backup_user@remote::backup/
```

## lsyncd 配置

```lua
-- /etc/lsyncd.conf
settings {
  logfile = "/var/log/lsyncd.log",
  statusFile = "/var/log/lsyncd-status.log",
  statusInterval = 20
}

sync {
  default.rsync,
  source = "/data/source",
  target = "user@remote:/data/backup",
  delay = 5,
  rsync = {
    archive = true,
    compress = true,
    verbose = true
  }
}
```

## sersync 配置

```xml
<!-- /usr/local/sersync/conf/conf.xml -->
<sersync>
  <localpath watch="/data/source">
    <remote ip="192.168.1.100" name="backup"/>
  </localpath>
  <rsync>
    <commonParams params="-artuz"/>
    <auth start="true" users="backup_user" passwordfile="/etc/sersync.passwd"/>
  </rsync>
</sersync>
```

## 📚 参考资料

- [rsync 官方文档](https://rsync.samba.org/)
- [lsyncd GitHub](https://github.com/axkibe/lsyncd)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
