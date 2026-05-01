# 云安全与合规

## 📋 概述

云安全是云计算时代的基石，涵盖身份认证、访问控制、数据加密、网络安全、合规审计等多个维度。

## 🎯 学习目标

- 掌握云安全核心原则
- 实施零信任架构
- 配置合规审计策略
- 数据加密与密钥管理
- 安全事件响应

## 🏗️ 云安全架构

```
┌─────────────────────────────────────────────────────────────┐
│                    云安全纵深防御体系                        │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              边界安全层 (Perimeter)                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │   WAF   │  │   DDoS  │  │  防火墙  │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              网络安全层 (Network)                    │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │   VPC   │  │  安全组  │  │  NACL   │            │   │
│  │  │  隔离   │  │  规则   │  │  访问控制│            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              身份安全层 (Identity)                   │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │   IAM   │  │   MFA   │  │   SSO   │            │   │
│  │  │  权限   │  │  多因素  │  │  单点登录│            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              数据安全层 (Data)                       │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  加密   │  │  脱敏   │  │  备份   │            │   │
│  │  │  KMS    │  │  分类分级│  │  恢复   │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              审计合规层 (Compliance)                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │  日志   │  │  审计   │  │  合规   │            │   │
│  │  │  CloudTrail│ │  追踪   │  │  检查   │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 🔐 身份与访问管理 (IAM)

### 1. 最小权限原则

```json
// 阿里云 RAM 策略示例
{
  "Version": "1",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecs:Describe*",
        "ecs:List*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:StartInstance",
        "ecs:StopInstance"
      ],
      "Resource": [
        "acs:ecs:*:*:instance/i-xxx01",
        "acs:ecs:*:*:instance/i-xxx02"
      ],
      "Condition": {
        "IpAddress": {
          "acs:SourceIp": "192.168.1.0/24"
        },
        "DateLessThan": {
          "acs:CurrentTime": "2026-12-31T23:59:59Z"
        }
      }
    }
  ]
}
```

### 2. 角色分离

| 角色 | 权限范围 | 适用人员 |
|------|----------|----------|
| **Admin** | 全权限 | 运维负责人 |
| **Developer** | 开发环境 + 只读生产 | 开发人员 |
| **Operator** | 生产环境操作 | 运维人员 |
| **Auditor** | 只读 + 审计日志 | 审计人员 |
| **Service** | 服务间调用 | 应用服务账号 |

### 3. MFA 强制启用

```bash
# AWS CLI 配置 MFA
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name admin-mfa \
  --outfile QRCode.png

# 启用 MFA
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin-mfa \
  --authentication-code1 123456 \
  --authentication-code2 654321
```

## 🔒 数据加密

### 1. 密钥管理 (KMS)

```yaml
# 密钥策略
apiVersion: kms.cloud/v1
kind: Key
metadata:
  name: app-encryption-key
spec:
  keySpec: AES_256
  keyUsage: ENCRYPT_DECRYPT
  rotationPeriod: 90d
  policy:
    principals:
      - service: rds
      - service: oss
    actions:
      - kms:Encrypt
      - kms:Decrypt
      - kms:GenerateDataKey
```

### 2. 存储加密

```bash
# 阿里云 OSS 服务端加密
ossutil mb oss://my-bucket \
  --server-side-encryption KMS \
  --sse-kms-keyid acs:kms:cn-hangzhou:123456:key/xxx

# RDS 透明数据加密 (TDE)
ALTER DATABASE mydb SET ENCRYPTION ON;

# 云盘加密
alicloud_disk encrypted_disk {
  name        = "encrypted-disk"
  size        = 100
  encrypted   = true
  kms_key_id  = "key-xxx"
}
```

### 3. 传输加密

```nginx
# Nginx TLS 配置
server {
    listen 443 ssl http2;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    
    # 仅允许 TLS 1.2+
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # 强加密套件
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}
```

## 🛡️ 网络安全

### 1. VPC 隔离设计

```
┌─────────────────────────────────────────────────────────────┐
│                         VPC: 10.0.0.0/16                     │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  公有子网        │  │  私有子网        │                  │
│  │  10.0.1.0/24    │  │  10.0.2.0/24    │                  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │                  │
│  │  │   SLB     │  │  │  │   K8s     │  │                  │
│  │  │   NAT     │  │  │  │   Nodes   │  │                  │
│  │  └───────────┘  │  │  └───────────┘  │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                              │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  数据库子网      │  │  管理子网        │                  │
│  │  10.0.3.0/24    │  │  10.0.4.0/24    │                  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │                  │
│  │  │   RDS     │  │  │  │  Bastion  │  │                  │
│  │  │  Redis    │  │  │  │  Monitor  │  │                  │
│  │  └───────────┘  │  │  └───────────┘  │                  │
│  └─────────────────┘  └─────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### 2. 安全组规则

```yaml
# 应用服务器安全组
security_group_rules:
  - direction: ingress
    protocol: tcp
    port_range: 80/443
    source: 0.0.0.0/0
    description: "HTTP/HTTPS 访问"
  
  - direction: ingress
    protocol: tcp
    port_range: 22
    source: 10.0.4.0/24  # 仅管理子网
    description: "SSH 管理"
  
  - direction: egress
    protocol: -1
    port_range: -1/-1
    destination: 0.0.0.0/0
    description: "允许所有出站"

# 数据库安全组
security_group_rules:
  - direction: ingress
    protocol: tcp
    port_range: 3306
    source: 10.0.2.0/24  # 仅应用子网
    description: "MySQL 访问"
  
  - direction: ingress
    protocol: tcp
    port_range: 22
    source: 10.0.4.0/24  # 仅管理子网
    description: "SSH 管理"
```

### 3. WAF 配置

```yaml
# WAF 防护规则
waf_rules:
  # SQL 注入防护
  - name: sql-injection
    action: block
    patterns:
      - "(?i)(union|select|insert|update|delete|drop)"
  
  # XSS 防护
  - name: xss-protection
    action: block
    patterns:
      - "<script[^>]*>"
      - "javascript:"
  
  # CC 攻击防护
  - name: rate-limiting
    action: block
    threshold: 100  # 每秒请求数
    window: 1s
  
  # IP 黑名单
  - name: ip-blocklist
    action: block
    ips:
      - 192.168.100.0/24
```

## 📋 合规审计

### 1. 云审计日志

```yaml
# CloudTrail 配置
aws cloudtrail create-trail \
  --name security-audit \
  --s3-bucket-name audit-logs-bucket \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --kms-key-id arn:aws:kms:region:account:key/key-id

# 日志加密
aws s3api put-bucket-encryption \
  --bucket audit-logs-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:region:account:key/key-id"
      }
    }]
  }'
```

### 2. 合规检查规则

```yaml
# AWS Config 规则
config_rules:
  - name: required-mfa
    source: IAM_MFA_ENABLED
    parameters:
      excludeUsers: "break-glass-*"
  
  - name: encrypted-volumes
    source: ENCRYPTED_VOLUMES
  
  - name: restricted-ssh
    source: INCOMING_SSH_DISABLED
  
  - name: cloudtrail-enabled
    source: CLOUD_TRAIL_ENABLED
  
  - name: rds-encryption
    source: RDS_STORAGE_ENCRYPTED
```

### 3. 安全基线检查

```bash
#!/bin/bash
# 安全基线检查脚本

echo "=== 安全基线检查 ==="

# 检查密码策略
aws iam get-account-password-policy | jq '.PasswordPolicy'

# 检查根账号 MFA
aws iam get-account-summary | jq '.AccountUsageSummary'

# 检查访问密钥
aws iam list-access-keys --user-name root

# 检查安全组
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[?IpPermissions[?FromPort==22]]'

# 检查未加密卷
aws ec2 describe-volumes \
  --filters "Name=encrypted,Values=false" \
  --query 'Volumes[*].[VolumeId,Size]'
```

## 🚨 安全事件响应

### 1. 事件分级

| 级别 | 说明 | 响应时间 | 示例 |
|------|------|----------|------|
| **P0** | 数据泄露/系统被入侵 | 15 分钟 | 数据库被拖库 |
| **P1** | 高危漏洞/未授权访问 | 1 小时 | 管理员账号异常 |
| **P2** | 中危漏洞/异常行为 | 4 小时 | 异常登录尝试 |
| **P3** | 低危漏洞/安全告警 | 24 小时 | 端口扫描 |

### 2. 响应流程

```
事件检测 ──► 初步评估 ──► 遏制隔离 ──► 根因分析 ──► 恢复清理 ──► 复盘改进
    │            │            │            │            │            │
    ▼            ▼            ▼            ▼            ▼            ▼
  SIEM 告警    P0-P3 分级   隔离受影响   日志分析     恢复服务     更新规则
  用户报告     确定范围     系统/账号    取证分析     修补漏洞     流程改进
```

### 3. 取证脚本

```bash
#!/bin/bash
# 安全事件取证脚本

INSTANCE_ID=$1
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== 开始取证：$INSTANCE_ID ==="

# 保存进程列表
aws ssm send-command \
  --instance-ids $INSTANCE_ID \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["ps auxf > /tmp/ps_$TIMESTAMP.txt"]'

# 保存网络连接
aws ssm send-command \
  --instance-ids $INSTANCE_ID \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["netstat -tunapl > /tmp/netstat_$TIMESTAMP.txt"]'

# 保存登录历史
aws ssm send-command \
  --instance-ids $INSTANCE_ID \
  --document-name "AWS-RunShellScript" \
  --parameters 'commands=["last -a > /tmp/last_$TIMESTAMP.txt"]'

# 创建磁盘快照
aws ec2 create-snapshot \
  --volume-id vol-xxx \
  --description "Forensic snapshot $TIMESTAMP" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Purpose,Value=Forensic}]'

echo "=== 取证完成 ==="
```

## 📚 合规框架

| 框架 | 适用场景 | 核心要求 |
|------|----------|----------|
| **等保 2.0** | 中国境内 | 安全物理环境/通信网络/区域边界 |
| **GDPR** | 欧盟用户数据 | 数据主体权利/跨境传输 |
| **SOC 2** | 云服务审计 | 安全/可用性/处理完整性 |
| **ISO 27001** | 信息安全管理体系 | 风险评估/控制措施 |
| **PCI DSS** | 支付卡行业 | 持卡人数据保护 |

## 📚 参考资料

- [云安全联盟 (CSA)](https://cloudsecurityalliance.org/)
- [CIS 云安全基准](https://www.cisecurity.org/benchmark/cloud_computers)
- [阿里云安全最佳实践](https://help.aliyun.com/product/28625.html)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
