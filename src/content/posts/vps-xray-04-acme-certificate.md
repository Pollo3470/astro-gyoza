---
title: VPS 节点搭建系列教程（五）：使用 acme.sh 申请免费 SSL 证书
date: 2026-01-05
summary: 使用 acme.sh 通过 Cloudflare DNS 验证自动申请和续签 ZeroSSL 泛域名证书，支持 OCSP Stapling，实现 SSL 证书的自动化管理。
category: 教程
tags: [VPS, SSL, 证书, acme.sh, ZeroSSL, HTTPS]
sticky: 0
---

> **系列导航**
>
> 1. [VPS 初始化与基础环境配置](/posts/vps-xray-00-initialization)
> 2. [使用 Xray 搭建 VLESS+Vision+Reality 节点](/posts/vps-xray-01-vless-vision-reality)
> 3. [配置 SmartDNS 优化 DNS 解析](/posts/vps-xray-02-smartdns)
> 4. [购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)
> 5. [使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)（本文）
> 6. [搭建 Nginx 反向代理服务](/posts/vps-xray-05-nginx-reverse-proxy)
> 7. [Nginx Stream SNI 分流实现 443 端口复用](/posts/vps-xray-06-nginx-stream-sni)

## 前言

SSL/TLS 证书是 HTTPS 安全连接的基础。在本章中，我们将使用 `acme.sh` 脚本通过 Cloudflare DNS 验证的方式，申请免费的泛域名证书。

`acme.sh` 默认使用 **ZeroSSL** 作为证书颁发机构（CA），相比 Let's Encrypt，ZeroSSL 的优势包括：

- ✅ 支持 OCSP Stapling（提升 TLS 握手性能）
- ✅ 与 Let's Encrypt 一样完全免费
- ✅ 证书有效期 90 天，自动续签

### 什么是泛域名证书？

泛域名证书（Wildcard Certificate）可以保护一个域名及其所有子域名。例如：

- 申请 `*.example.com` 证书后
- `proxy.example.com`、`blog.example.com`、`www.example.com` 等都可以使用这张证书

这样我们只需要申请一次证书，就可以用于所有子域名，非常方便。

### 什么是 DNS 验证？

证书颁发机构需要验证你对域名的所有权。常见的验证方式有：

- **HTTP 验证**：在网站根目录放置特定文件
- **DNS 验证**：添加特定的 DNS TXT 记录

DNS 验证的优点：

- 不需要开放 80 端口
- 可以申请泛域名证书
- 适合在多台服务器上使用同一证书

---

### 第一步：安装依赖

`acme.sh` 可能需要 `socat` 作为依赖：

```shell
apt install -y socat
```

---

### 第二步：安装 acme.sh

`acme.sh` 是一个纯 Shell 编写的 ACME 协议客户端，支持多种 CA 和 DNS 提供商。

```shell
# 下载并安装 acme.sh
# 将 my@example.com 替换为你的邮箱（用于接收证书到期提醒和 ZeroSSL 账户注册）
curl https://get.acme.sh | sh -s email=my@example.com
```

安装完成后，重新加载 Shell 环境：

```shell
# 使 acme.sh 命令立即可用
source ~/.bashrc
```

验证安装：

```shell
# 查看 acme.sh 版本
acme.sh --version
```

> **说明**：`acme.sh` 会自动使用你提供的邮箱注册 ZeroSSL 账户，无需额外操作。

---

### 第三步：配置自动更新

开启 `acme.sh` 的自动更新功能，确保脚本保持最新：

```shell
acme.sh --upgrade --auto-upgrade
```

---

### 第四步：配置 Cloudflare API

`acme.sh` 需要通过 Cloudflare API 自动添加 DNS 记录来验证域名所有权。

在 VPS 上设置环境变量：

```shell
# 设置 Cloudflare API Token（上一章获取的）
export CF_Token="你的Cloudflare_API_Token"

# 设置 Cloudflare 账户邮箱
export CF_Email="你的Cloudflare账户邮箱"
```

> **提示**：这些环境变量会被 `acme.sh` 自动保存，后续续签时无需再次设置。

---

### 第五步：申请泛域名证书

现在开始申请证书。请将 `example.com` 替换为你的域名：

```shell
acme.sh --issue -d "example.com" -d "*.example.com" --dns dns_cf --keylength ec-384
```

#### 参数说明

| 参数                 | 说明                                |
| -------------------- | ----------------------------------- |
| `--issue`            | 申请证书                            |
| `-d "example.com"`   | 主域名                              |
| `-d "*.example.com"` | 泛域名（覆盖所有子域名）            |
| `--dns dns_cf`       | 使用 Cloudflare DNS 验证            |
| `--keylength ec-384` | 使用 ECC 384 位密钥（更安全、更快） |

#### 申请过程

执行命令后，`acme.sh` 会：

1. 通过 Cloudflare API 添加 DNS TXT 记录
2. 等待 DNS 记录生效
3. 请求 ZeroSSL 验证
4. 验证成功后获取证书
5. 自动删除临时的 DNS 记录

成功输出示例：

```
[Mon Jan  6 10:00:00 UTC 2026] Your cert is in: /root/.acme.sh/example.com_ecc/example.com.cer
[Mon Jan  6 10:00:00 UTC 2026] Your cert key is in: /root/.acme.sh/example.com_ecc/example.com.key
[Mon Jan  6 10:00:00 UTC 2026] The intermediate CA cert is in: /root/.acme.sh/example.com_ecc/ca.cer
[Mon Jan  6 10:00:00 UTC 2026] And the full chain certs is in: /root/.acme.sh/example.com_ecc/fullchain.cer
```

---

### 第六步：安装证书

证书申请成功后，需要将它安装到指定目录，以便 Nginx 和 Xray 使用。

> **前置条件**：在执行下面的命令之前，请确保已经完成 [Nginx 的安装](/posts/vps-xray-05-nginx-reverse-proxy#第一步安装-nginx-和-stream-模块)。

> **坑点**：`acme.sh` 的 `--install-cert` 命令设计为**只执行一次**！它会记录证书的安装路径和 reloadcmd。如果后续再次执行，会覆盖之前的配置。因此，必须**一次性配置好所有服务的重载命令**。

```shell
# 创建证书存放目录
mkdir -p /etc/acme-certs/example.com

# 安装证书到指定目录
# reloadcmd 会在证书续签后自动执行，重启相关服务
acme.sh --install-cert -d example.com --ecc \
  --cert-file      /etc/acme-certs/example.com/cert.pem \
  --key-file       /etc/acme-certs/example.com/key.pem \
  --fullchain-file /etc/acme-certs/example.com/fullchain.pem \
  --reloadcmd      "systemctl reload nginx && systemctl restart xray"
```

#### 参数说明

| 参数               | 说明                                                |
| ------------------ | --------------------------------------------------- |
| `-d example.com`   | 指定域名                                            |
| `--ecc`            | 使用 ECC 证书（与申请时的 --keylength ec-384 对应） |
| `--cert-file`      | 证书文件安装路径                                    |
| `--key-file`       | 私钥文件安装路径                                    |
| `--fullchain-file` | 完整证书链安装路径（推荐使用）                      |
| `--reloadcmd`      | 证书续签后执行的命令                                |

---

### 第七步：验证证书

检查证书文件是否已正确安装：

```shell
# 查看证书文件
ls -la /etc/acme-certs/example.com/
```

输出应该包含：

```
-rw-r--r-- 1 root root  xxx cert.pem
-rw-r--r-- 1 root root  xxx fullchain.pem
-rw------- 1 root root  xxx key.pem
```

查看证书信息：

```shell
# 查看证书详情
openssl x509 -in /etc/acme-certs/example.com/fullchain.pem -text -noout | head -20
```

你可以看到证书的：

- 颁发者（Issuer）：ZeroSSL
- 有效期（Validity）：90 天
- 主题（Subject）：你的域名
- 备用名称（Subject Alternative Name）：包含泛域名

---

### 第八步：理解自动续签

ZeroSSL 证书有效期为 90 天。`acme.sh` 会自动设置定时任务来续签证书。

查看自动续签任务：

```shell
# 查看 crontab
crontab -l | grep acme
```

你应该能看到类似：

```
0 0 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

这表示每天凌晨会检查证书是否需要续签。当证书剩余有效期少于 30 天时，会自动续签并执行 `reloadcmd`。

#### 手动续签（测试用）

```shell
# 强制续签证书
acme.sh --renew -d example.com --ecc --force
```

#### 查看证书状态

```shell
# 列出所有已申请的证书
acme.sh --list
```

---

## 证书文件说明

| 文件            | 用途                       | 配置时使用                |
| --------------- | -------------------------- | ------------------------- |
| `cert.pem`      | 服务器证书                 | 较少单独使用              |
| `key.pem`       | 私钥                       | `ssl_certificate_key`     |
| `fullchain.pem` | 完整证书链（包含中间证书） | `ssl_certificate`（推荐） |

> **推荐**：在 Nginx 和 Xray 配置中，证书使用 `fullchain.pem`，私钥使用 `key.pem`。

---

## 切换证书颁发机构（可选）

如果你更倾向于使用 Let's Encrypt，可以切换默认 CA：

```shell
# 切换到 Let's Encrypt
acme.sh --set-default-ca --server letsencrypt

# 切换回 ZeroSSL（默认）
acme.sh --set-default-ca --server zerossl
```

---

## 常见问题

**问题 1：申请失败，提示 DNS 验证错误**

```shell
# 查看详细日志
acme.sh --issue -d "example.com" -d "*.example.com" --dns dns_cf --debug 2
```

常见原因：

- API Token 权限不足，确认有 "Zone - DNS - Edit" 权限
- 环境变量设置错误
- DNS 服务器未正确指向 Cloudflare

**问题 2：提示 "Rate limit"**

ZeroSSL 和 Let's Encrypt 都有速率限制。解决方案：等待一段时间后重试，或使用测试环境：

```shell
# 使用 Let's Encrypt 测试环境（不计入速率限制，但证书不被信任）
acme.sh --issue -d "example.com" -d "*.example.com" --dns dns_cf --staging --server letsencrypt
```

**问题 3：证书续签后服务未重启**

检查 `--reloadcmd` 是否配置正确：

```shell
# 查看证书配置
cat ~/.acme.sh/example.com_ecc/example.com.conf | grep Le_ReloadCmd
```

如需修改，可以重新执行 `--install-cert` 命令。

**问题 4：提示 "nginx: command not found"**

确保 Nginx 已安装并在 PATH 中：

```shell
which nginx
```

如果 Nginx 尚未安装，可以先跳过 `--reloadcmd` 中的 nginx 部分，后续再更新。

---

## 总结

在本章中，我们完成了：

- ✅ 安装 acme.sh 证书管理工具
- ✅ 配置 Cloudflare API 验证
- ✅ 申请 ZeroSSL 泛域名 SSL 证书（支持 OCSP）
- ✅ 安装证书到指定目录
- ✅ 配置自动续签和服务重载

现在我们拥有了自己的 SSL 证书，可以用于：

- Nginx 反向代理的 HTTPS 服务
- Xray Reality 的"偷自己证书"配置

在下一章中，我们将配置 Nginx 反向代理服务。

---

**上一篇**：[VPS 节点搭建系列教程（四）：购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)

**下一篇**：[VPS 节点搭建系列教程（六）：搭建 Nginx 反向代理服务](/posts/vps-xray-05-nginx-reverse-proxy)
