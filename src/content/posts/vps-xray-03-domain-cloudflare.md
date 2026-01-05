---
title: VPS 节点搭建系列教程（四）：购买域名并配置 Cloudflare DNS
date: 2026-01-05
summary: 手把手教你购买域名并将 DNS 解析托管到 Cloudflare，为后续申请 SSL 证书和配置 Nginx 反向代理做准备。
category: 教程
tags: [VPS, 域名, Cloudflare, DNS, 网站搭建]
sticky: 0
---

> **系列导航**
>
> 1. [VPS 初始化与基础环境配置](/posts/vps-xray-00-initialization)
> 2. [使用 Xray 搭建 VLESS+Vision+Reality 节点](/posts/vps-xray-01-vless-vision-reality)
> 3. [配置 SmartDNS 优化 DNS 解析](/posts/vps-xray-02-smartdns)
> 4. [购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)（本文）
> 5. [使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)
> 6. [搭建 Nginx 反向代理服务](/posts/vps-xray-05-nginx-reverse-proxy)
> 7. [Nginx Stream SNI 分流实现 443 端口复用](/posts/vps-xray-06-nginx-stream-sni)

## 前言

在前面的章节中，我们使用"偷别人证书"的方式搭建了 Reality 节点。这种方式的优点是简单快速，但也有局限性：

- 无法搭建真正的网站服务
- 无法实现 443 端口复用
- 伪装程度不如"偷自己证书"完美

从本章开始，我们将准备自己的域名和证书，最终实现：

- 拥有自己的域名和 SSL 证书
- 使用 Nginx 反向代理搭建真实网站
- 使用 SNI 分流实现 443 端口复用
- Reality 节点使用自己的证书（"偷自己证书"），fallback 到真实网站

---

### 第一步：选择并购买域名

#### 域名比价网站

1. **TLD-LIST**: https://tld-list.com
2. **哪煮米**: https://www.nazhumi.com

#### 域名注册商推荐

| 注册商                   | 优点                                  | 缺点             |
| ------------------------ | ------------------------------------- | ---------------- |
| **Cloudflare Registrar** | 成本价销售，无隐藏费用                | 可选域名后缀较少 |
| **Namecheap**            | 价格实惠，首年优惠多                  | 续费可能较贵     |
| **Spaceship**            | 价格实惠，首年优惠多，免费 WHOIS 隐私 | 续费可能较贵     |
| **阿里云/腾讯云**        | 中文界面，支付方便                    | 需要实名认证     |

#### 域名选择建议

1. **后缀选择**：
   - `.com`：最常见，信任度高
   - `.net`、`.org`：次优选择
   - `.me`、`.io`：适合个人项目
   - `.top`、`.xyz`纯数字域名：便宜
   - 避免免费或过于冷门的后缀
2. **长度**：尽量简短，容易记忆

3. **内容**：避免包含敏感词汇

#### 购买流程（以 Cloudflare Registrar 为例）

1. 访问 [Cloudflare Registrar](https://dash.cloudflare.com/domains)
2. 搜索你想要的域名
3. 选择域名后缀，加入购物车
4. 完成支付

> **提示**：如果你已经在其他注册商购买了域名，可以跳过购买步骤，直接将 DNS 托管到 Cloudflare。

---

### 第二步：注册 Cloudflare 账号

如果你还没有 Cloudflare 账号，需要先注册一个。

1. 访问 [cloudflare.com](https://www.cloudflare.com)
2. 点击 **Sign Up**
3. 输入邮箱和密码，创建账号
4. 验证邮箱

---

### 第三步：将域名添加到 Cloudflare

#### 3.1 添加站点

1. 登录 Cloudflare Dashboard
2. 点击 **Add a Site**（添加站点）
3. 输入你的域名（例如 `example.com`）
4. 选择 **Free** 计划（免费版足够使用）
5. 点击 **Continue**

#### 3.2 获取 Cloudflare DNS 服务器

Cloudflare 会扫描你域名现有的 DNS 记录，并显示需要设置的 DNS 服务器（Nameservers）：

```
ns1.cloudflare.com
ns2.cloudflare.com
```

> **注意**：每个账号分配的 nameserver 可能不同，请以实际显示的为准。

#### 3.3 修改域名 DNS 服务器

登录你的域名注册商控制台，找到 DNS/Nameserver 设置，将 DNS 服务器修改为 Cloudflare 提供的地址。

**各注册商修改方式**：

- **Cloudflare Registrar**：如果域名在 Cloudflare 购买，DNS 已自动配置
- **Namecheap**：Domain List → Manage → Nameservers → Custom DNS
- **阿里云**：域名控制台 → DNS 修改
- **腾讯云**：域名管理 → 修改 DNS 服务器

修改后，点击 Cloudflare 页面的 **Done, check nameservers**。

> **坑点**：DNS 服务器修改需要时间生效，通常需要几分钟到 48 小时不等。Cloudflare 会在生效后发送邮件通知。

---

### 第四步：配置 DNS 记录

DNS 服务器生效后，我们需要添加 DNS 记录，将域名指向 VPS。

#### 4.1 添加 A 记录

在 Cloudflare Dashboard 中：

1. 选择你的域名
2. 点击左侧 **DNS** → **Records**
3. 点击 **Add record**

添加以下记录：

| Type | Name  | Content    | Proxy status      |
| ---- | ----- | ---------- | ----------------- |
| A    | @     | 你的VPS IP | DNS only (灰色云) |
| A    | proxy | 你的VPS IP | DNS only (灰色云) |
| A    | blog  | 你的VPS IP | DNS only (灰色云) |

#### 4.2 关于代理状态（Proxy status）

Cloudflare 的代理状态有两种：

- **Proxied（橙色云）**：流量经过 Cloudflare CDN

  - 优点：隐藏真实 IP，提供 DDoS 防护
  - 缺点：无法使用非 HTTP/HTTPS 服务，可能增加延迟

- **DNS only（灰色云）**：仅 DNS 解析，不经过 CDN
  - 优点：直连 VPS，延迟低
  - 缺点：暴露真实 IP

> **重要**：对于 Xray 节点使用的域名，**必须选择 DNS only（灰色云）**！因为 Xray 需要直接连接到你的 VPS，不能经过 Cloudflare 代理。

---

### 第五步：验证 DNS 配置

等待几分钟后，验证 DNS 是否已生效：

```shell
# 在本地电脑或 VPS 上执行
# Windows 用户使用 nslookup
nslookup example.com

# Linux/macOS 用户使用 dig
dig example.com
```

如果返回的 IP 地址是你的 VPS IP，说明 DNS 配置成功。

你也可以使用在线工具验证：

- [DNS Checker](https://dnschecker.org/)
- [WhatsMyDNS](https://www.whatsmydns.net/)

---

### 第六步：获取 Cloudflare API Token

为了在下一章使用 `acme.sh` 自动申请证书，我们需要获取 Cloudflare API Token。

#### 6.1 创建 API Token

1. 登录 Cloudflare Dashboard
2. 点击右上角头像 → **My Profile**
3. 点击左侧 **API Tokens**
4. 点击 **Create Token**

#### 6.2 使用模板创建

1. 找到 **Edit zone DNS** 模板，点击 **Use template**
2. 配置 Token 权限：
   - **Permissions**：Zone - DNS - Edit（默认已选）
   - **Zone Resources**：Include - Specific zone - 选择你的域名
   - 或者选择 **Include - All zones** 允许管理所有域名
3. 点击 **Continue to summary**
4. 点击 **Create Token**

#### 6.3 保存 Token

创建成功后，页面会显示 Token

> **坑点**：这个 Token **只会显示一次**！请立即复制并保存到安全的地方。如果丢失，只能重新创建。

#### 6.4 验证 Token

Cloudflare 会提供一个验证命令：

```shell
curl -X GET "https://api.cloudflare.com/client/v4/user/tokens/verify" \
     -H "Authorization: Bearer 你的Token" \
     -H "Content-Type:application/json"
```

如果返回 `"status": "active"`，说明 Token 有效。

---

### 第七步：配置 SSL/TLS 设置（可选）

在 Cloudflare Dashboard 中，建议进行以下 SSL/TLS 设置：

1. 点击左侧 **SSL/TLS** → **Overview**
2. 将加密模式设置为 **Full (strict)**

这样可以确保 Cloudflare 到源服务器的连接也是加密的（当你启用代理时）。

---

## 常见问题

**问题 1：DNS 修改后长时间不生效**

- DNS 传播最长需要 48 小时
- 可以尝试清除本地 DNS 缓存
- 使用 DNS Checker 检查全球 DNS 状态

**问题 2：提示 "Nameservers not changed"**

- 确认已在域名注册商处正确修改 NS 记录
- 确认没有多余的 NS 记录
- 耐心等待，可能需要几小时

**问题 3：域名无法解析**

- 检查是否添加了正确的 A 记录
- 确认 VPS IP 地址正确
- 使用 `dig` 命令诊断

---

## 总结

在本章中，我们完成了：

- ✅ 了解域名注册商选择和域名购买
- ✅ 将域名 DNS 托管到 Cloudflare
- ✅ 配置 A 记录指向 VPS
- ✅ 获取 Cloudflare API Token（用于申请证书）

现在域名已经准备就绪，在下一章中，我们将使用 `acme.sh` 申请免费的 SSL 证书。

---

**上一篇**：[VPS 节点搭建系列教程（三）：配置 SmartDNS 优化 DNS 解析](/posts/vps-xray-02-smartdns)

**下一篇**：[VPS 节点搭建系列教程（五）：使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)
