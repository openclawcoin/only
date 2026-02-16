# AI Service Offline Server Configuration Guide

## 概述

AI Service 是一个离线签名服务器，用于为 AI Agent 授权签名。

## 架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     离线服务器 (Air-gapped)                        │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  操作系统: Ubuntu 22.04 LTS (最小化安装)                  │   │
│   │  网络: 物理隔离，无互联网连接                              │   │
│   │  存储: 加密硬盘                                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  AI Service 应用                                          │   │
│   │  - Node.js + Express                                    │   │
│   │  - 私钥存储在环境变量                                     │   │
│   │  - 只接收签名请求，返回签名                                │   │
│   │  - 不发送任何区块链交易                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  私钥管理:                                               │   │
│   │  - 环境变量: AI_SERVICE_KEY                              │   │
│   │  - 从不写入磁盘                                           │   │
│   │  - 从不输出到日志                                         │   │
│   │  - 从不显示在终端                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 只能通过 API 返回签名
                              ▼ (物理隔离，不能反向访问)
┌─────────────────────────────────────────────────────────────────┐
│                        公网服务器                                   │
│                                                                 │
│   - Nginx 反向代理                                             │
│   - HTTPS (Let's Encrypt)                                      │
│   - Rate limiting                                              │
│   - 只转发请求到内部网络                                         │
└─────────────────────────────────────────────────────────────────┘
```

## 硬件配置推荐

### 入门配置 (个人/小团队)
```
CPU: 1 core
RAM: 1 GB
SSD: 20 GB
成本: ~$50/月 (租用) 或 ~$200 (一次性设备)
```

### 生产配置 (企业级)
```
CPU: 2 cores
RAM: 4 GB
SSD: 50 GB
成本: ~$100/月 (租用) 或 ~$500 (一次性设备)
```

### 高安全配置 (银行级)
```
CPU: 4 cores
RAM: 8 GB
SSD: 100 GB (加密)
额外: HSM 硬件安全模块
成本: ~$500+/月 或 ~$2000+ (一次性设备)
```

## 安装步骤

### 1. 准备离线服务器

```bash
# 使用最小化 Ubuntu 22.04
sudo apt update && sudo apt upgrade -y

# 安装必要工具
sudo apt install -y nodejs npm git

# 检查版本
node --version  # 应 >= 18
npm --version
```

### 2. 部署 AI Service 应用

```bash
# 创建应用目录
mkdir -p /opt/ai-service
cd /opt/ai-service

# 复制应用代码
# (通过 USB 或其他离线媒介)
cp -r /path/to/ai-service-offline.js ./

# 安装依赖 (如果需要)
npm init -y
npm install express ethers
```

### 3. 配置私钥

**方式 A: 环境变量 (推荐)**

```bash
# 创建 systemd 服务文件
sudo nano /etc/systemd/system/ai-service.service
```

```ini
[Unit]
Description=AI Service Offline Signer
After=network.target

[Service]
Type=simple
User=ai-service
WorkingDirectory=/opt/ai-service
ExecStart=/usr/bin/node ai-service-offline.js
Environment=AI_SERVICE_KEY=your_private_key_here
Restart=always
RestartSec=5

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/opt/ai-service

[Install]
WantedBy=multi-user.target
```

```bash
# 启动服务
sudo systemctl daemon-reload
sudo systemctl enable ai-service
sudo systemctl start ai-service

# 检查状态
sudo systemctl status ai-service
```

**方式 B: 硬件钱包 (更安全)**

如果你有 Ledger 或 Trezor：

```javascript
// 使用硬件钱包签名
const { ethers } = require("ethers");
const HID = require("node-hid");

// 通过硬件钱包签名 (私钥从不接触服务器)
const device = new HID.HID(..., ...);
```

### 4. 配置防火墙

```bash
# 只允许内部网络访问
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.0.0.0/8 to any port 3000
sudo ufw enable
```

### 5. 配置日志

```bash
# 创建日志目录
sudo mkdir -p /var/log/ai-service
sudo chown ai-service:ai-service /var/log/ai-service

# 配置 logrotate
sudo nano /etc/logrotate.d/ai-service
```

```
/var/log/ai-service/*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    create 640 ai-service ai-service
}
```

## 安全检查清单

### 部署前检查

- [ ] 服务器物理隔离
- [ ] SSH 已禁用密码登录
- [ ] SSH 密钥已配置
- [ ] 防火墙已配置
- [ ] 自动更新已禁用
- [ ] 没有不必要的服务运行
- [ ] 私钥只存储在环境变量中
- [ ] 日志不包含敏感信息

### 日常检查

```bash
# 检查服务状态
sudo systemctl status ai-service

# 检查最近登录
last

# 检查异常进程
ps aux

# 检查网络连接
ss -tunpl

# 检查资源使用
htop
```

## 监控告警

### 创建监控脚本

```bash
#!/bin/bash
# /opt/ai-service/monitor.sh

# 检查服务状态
if ! systemctl is-active --quiet ai-service; then
    echo "AI Service is down!"
    # 发送告警 (邮件/Slack等)
    curl -X POST https://alert.example.com \
        -H "Content-Type: application/json" \
        -d '{"message": "AI Service is down!"}'
fi

# 检查磁盘空间
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "Disk usage is ${DISK_USAGE}%"
fi

# 检查内存使用
MEM_USAGE=$(free | grep Mem | awk '{print $3/$2 * 100}')
if [ $(echo "$MEM_USAGE > 80" | bc) -eq 1 ]; then
    echo "Memory usage is ${MEM_USAGE}%"
fi
```

```bash
# 添加 crontab
crontab -e

# 每5分钟检查一次
*/5 * * * * /opt/ai-service/monitor.sh
```

## 应急响应

### 如果怀疑私钥泄露

1. **立即停止服务**
```bash
sudo systemctl stop ai-service
```

2. **更换 AI Service Signer**
```javascript
// 调用合约
await token.setAIServiceSigner(newSignerAddress);
```

3. **调查泄露原因**
- 检查日志
- 检查服务器访问记录
- 检查网络流量

4. **部署新服务器**
- 使用新私钥
- 配置新服务器
- 更新 DNS/反向代理

## 备份与恢复

### 私钥备份

**重要**: 私钥必须备份，但必须安全！

```bash
# 打印私钥 (只在紧急情况使用)
echo $AI_SERVICE_KEY

# 导出到加密文件 (如果需要存储)
gpg -c ai-service-key.txt
```

### 恢复流程

1. 在新服务器上设置环境变量
2. 启动服务
3. 测试签名功能
4. 更新 DNS/反向代理

## FAQ

### Q: 私钥可以修改吗？

A: 可以。Owner 可以随时调用 `setAIServiceSigner()` 修改签名地址。

### Q: 如果服务器宕机怎么办？

A: 
- 部署备用服务器
- 使用负载均衡
- 快速切换到备用服务器

### Q: 如何验证服务器安全？

A:
- 定期安全审计
- 渗透测试
- 日志分析

## 联系

如有问题，请联系 protocol maintainer。

## License

MIT
