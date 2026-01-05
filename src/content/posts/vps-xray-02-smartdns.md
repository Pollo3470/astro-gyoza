---
title: VPS 节点搭建系列教程（三）：配置 SmartDNS 优化 DNS 解析
date: 2026-01-05
summary: 在 VPS 上部署 SmartDNS 作为 Xray 的 DNS 服务器，实现 DNS 解析优化、多上游 DNS 测速选优，提升代理访问速度。
category: 教程
tags: [VPS, SmartDNS, DNS, Xray, 网络优化]
sticky: 0
---

> **系列导航**
>
> 1. [VPS 初始化与基础环境配置](/posts/vps-xray-00-initialization)
> 2. [使用 Xray 搭建 VLESS+Vision+Reality 节点](/posts/vps-xray-01-vless-vision-reality)
> 3. [配置 SmartDNS 优化 DNS 解析](/posts/vps-xray-02-smartdns)（本文）
> 4. [购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)
> 5. [使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)
> 6. [搭建 Nginx 反向代理服务](/posts/vps-xray-05-nginx-reverse-proxy)
> 7. [Nginx Stream SNI 分流实现 443 端口复用](/posts/vps-xray-06-nginx-stream-sni)

## 前言

在上一章中，我们成功搭建了 VLESS+Vision+Reality 节点。默认情况下，Xray 会使用系统 DNS 或配置中指定的公共 DNS（如 8.8.8.8）进行域名解析。

但是，使用单一的 DNS 服务器可能会遇到以下问题：

- **解析速度不稳定**：单一 DNS 服务器可能偶尔响应缓慢
- **解析结果不是最优**：返回的 IP 地址可能不是距离最近、速度最快的
- **某些 DNS 可能被干扰**：特定的公共 DNS 在某些网络环境下可能不可用

**SmartDNS** 是一个优秀的 DNS 解决方案，它可以：

- ✅ 同时查询多个上游 DNS 服务器
- ✅ 对返回结果进行测速，选择最快的 IP
- ✅ 缓存 DNS 结果，加速后续查询
- ✅ 预取即将过期的域名，保持缓存新鲜

---

### 第一步：安装 SmartDNS

SmartDNS 可以通过多种方式安装。这里我们使用官方提供的安装脚本：

```shell
# 下载最新版 SmartDNS
# 访问 https://github.com/pymumu/smartdns/releases 获取最新版本号
wget https://github.com/pymumu/smartdns/releases/download/Release47.1/smartdns.1.2025.11.09-1443.x86_64-debian-all.deb

# 安装
dpkg -i smartdns.1.2025.11.09-1443.x86_64-debian-all.deb
```

安装完成后，验证安装：

```shell
# 查看 SmartDNS 版本
smartdns --version
```

---

### 第二步：配置 SmartDNS

编辑 SmartDNS 配置文件：

```shell
vim /etc/smartdns/smartdns.conf
```

将配置文件内容替换为以下内容：

```ini
# SmartDNS 配置文件

# 绑定地址和端口
# 使用 5353 端口，避免与系统 DNS（53端口）冲突
bind 127.0.0.1:5353
bind-tcp 127.0.0.1:5353

# 上游 DNS 服务器列表
# SmartDNS 会同时查询所有服务器，选择最快的结果返回
server 8.8.8.8        # Google DNS
server 8.8.4.4        # Google DNS 备用
server 1.1.1.1        # Cloudflare DNS
server 1.0.0.1        # Cloudflare DNS 备用
server 9.9.9.9        # Quad9 DNS（注重安全）
server 208.67.222.222 # OpenDNS

# 测速模式
# ping: 使用 ICMP ping 测速
# tcp:80: 使用 TCP 80 端口测速
# tcp:443: 使用 TCP 443 端口测速
speed-check-mode ping,tcp:80,tcp:443

# 启用域名预取
# 在缓存即将过期前自动刷新，保持缓存新鲜
prefetch-domain yes

# 禁用 HTTPS/SVCB 记录查询
# 某些客户端会查询 QTYPE 65 (HTTPS) 记录，可能导致问题
# 返回 SOA 表示不支持，让客户端回退到普通 A/AAAA 记录
force-qtype-SOA 65

# 日志级别
# 可选值：off, fatal, error, warn, notice, info, debug
log-level info
```

#### 配置项详解

| 配置项                | 说明                               |
| --------------------- | ---------------------------------- |
| `bind 127.0.0.1:5353` | 绑定到本地 5353 端口，仅本机可访问 |
| `server`              | 上游 DNS 服务器，可添加多个        |
| `speed-check-mode`    | 测速方式，多种方式组合使用         |
| `prefetch-domain`     | 预取功能，提前刷新即将过期的缓存   |
| `force-qtype-SOA 65`  | 阻止 HTTPS/SVCB 记录查询           |

> **为什么使用 5353 端口？**
>
> 系统的 DNS 服务默认使用 53 端口。SmartDNS 使用 5353 端口可以避免端口冲突，同时只让 Xray 使用 SmartDNS，不影响系统其他 DNS 解析。

---

### 第三步：启动 SmartDNS

```shell
# 启动 SmartDNS 服务
systemctl start smartdns

# 设置开机自启
systemctl enable smartdns

# 查看服务状态
systemctl status smartdns
```

如果状态显示 `active (running)`，说明 SmartDNS 已成功启动。

---

### 第四步：测试 DNS 解析

使用 `dig` 命令测试 SmartDNS 是否正常工作：

```shell
# 安装 dig 工具（如果没有）
apt install -y dnsutils

# 测试 DNS 解析
dig @127.0.0.1 -p 5353 google.com
```

你应该能看到类似以下的输出：

```
;; ANSWER SECTION:
google.com.             300     IN      A       142.250.185.14

;; Query time: 50 msec
;; SERVER: 127.0.0.1#5353(127.0.0.1) (UDP)
```

再次查询同一域名，应该会更快（从缓存返回）：

```shell
dig @127.0.0.1 -p 5353 google.com
```

```
;; Query time: 0 msec
```

---

### 第五步：配置 Xray 使用 SmartDNS

现在我们需要修改 Xray 配置，让它使用 SmartDNS 进行 DNS 解析。

编辑 Xray 配置文件：

```shell
vim /usr/local/etc/xray/config.json
```

在配置文件中添加 `dns` 部分：

```json
{
  "log": {
    "loglevel": "info"
  },
  "dns": {
    "servers": ["udp://127.0.0.1:5353"]
  },
  "inbounds": [
    // ... 你的 inbound 配置保持不变
  ],
  "outbounds": [
    // ... 你的 outbound 配置保持不变
  ]
}
```

完整的配置示例：

```json
{
  "log": {
    "loglevel": "info"
  },
  "dns": {
    "servers": ["udp://127.0.0.1:5353"]
  },
  "inbounds": [
    {
      "tag": "VLESS-Vision-REALITY",
      "listen": "::",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "你的UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.microsoft.com:443",
          "serverNames": ["www.microsoft.com"],
          "privateKey": "你的私钥",
          "shortIds": ["你的shortId"]
        }
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom"
    },
    {
      "tag": "block",
      "protocol": "blackhole"
    }
  ]
}
```

保存配置后，重启 Xray：

```shell
# 检查配置语法
xray run -test -config /usr/local/etc/xray/config.json

# 重启 Xray
systemctl restart xray
```

---

### 第六步：验证 Xray DNS 配置

查看 Xray 日志，确认 DNS 解析正常：

```shell
# 实时查看日志
journalctl -u xray -o cat -f
```

当客户端通过代理访问网站时，你应该能在日志中看到 DNS 解析相关的信息。

---

## 进阶配置（可选）

### 使用 DoH/DoT 加密 DNS

如果你希望 DNS 查询也被加密，可以使用 DoH（DNS over HTTPS）或 DoT（DNS over TLS）：

```ini
# DoT (DNS over TLS)
server-tls 8.8.8.8:853
server-tls 1.1.1.1:853

# DoH (DNS over HTTPS)
server-https https://dns.google/dns-query
server-https https://cloudflare-dns.com/dns-query
```

### 配置 DNS 分流

如果你需要对特定域名使用特定的 DNS 服务器：

```ini
# 创建一个服务器组
server 8.8.8.8 -group google -exclude-default-group
server 8.8.4.4 -group google -exclude-default-group

# 指定域名使用该组
nameserver /google.com/google
nameserver /youtube.com/google
```

### 查看 SmartDNS 缓存统计

```shell
# 查看 SmartDNS 运行统计
smartdns --cache-print
```

---

## 常见问题

**问题 1：SmartDNS 无法启动**

```shell
# 查看详细日志
journalctl -u smartdns -n 50
```

常见原因：

- 5353 端口被占用
- 配置文件语法错误

**问题 2：DNS 解析失败**

```shell
# 检查 SmartDNS 是否在监听
ss -tulnp | grep 5353

# 手动测试解析
dig @127.0.0.1 -p 5353 google.com
```

**问题 3：解析速度没有提升**

- 确认 `speed-check-mode` 配置正确
- 检查上游 DNS 服务器是否可访问
- 等待缓存预热后测试

---

## 总结

在本章中，我们完成了：

- ✅ 安装 SmartDNS
- ✅ 配置多个上游 DNS 服务器
- ✅ 启用测速选优和缓存预取
- ✅ 配置 Xray 使用 SmartDNS

现在你的 Xray 节点拥有了更智能、更快速的 DNS 解析能力。

在接下来的章节中，我们将开始准备自己的域名和证书，为最终的 **"偷自己证书"** 方案做准备。

---

**上一篇**：[VPS 节点搭建系列教程（二）：使用 Xray 搭建 VLESS+Vision+Reality 节点](/posts/vps-xray-01-vless-vision-reality)

**下一篇**：[VPS 节点搭建系列教程（四）：购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)
