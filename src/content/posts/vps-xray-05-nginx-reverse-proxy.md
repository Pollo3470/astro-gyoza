---
title: VPS 节点搭建系列教程（六）：搭建 Nginx 反向代理服务
date: 2026-01-05
summary: 配置 Nginx 反向代理，为 Plex、Homepage 等服务提供安全的 HTTPS 访问，包含安全加固、性能优化和通用配置片段。
category: 教程
tags: [VPS, Nginx, 反向代理, HTTPS, SSL, Web服务器]
sticky: 0
---

> **系列导航**
>
> 1. [VPS 初始化与基础环境配置](/posts/vps-xray-00-initialization)
> 2. [使用 Xray 搭建 VLESS+Vision+Reality 节点](/posts/vps-xray-01-vless-vision-reality)
> 3. [配置 SmartDNS 优化 DNS 解析](/posts/vps-xray-02-smartdns)
> 4. [购买域名并配置 Cloudflare DNS](/posts/vps-xray-03-domain-cloudflare)
> 5. [使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)
> 6. [搭建 Nginx 反向代理服务](/posts/vps-xray-05-nginx-reverse-proxy)（本文）
> 7. [Nginx Stream SNI 分流实现 443 端口复用](/posts/vps-xray-06-nginx-stream-sni)

## 前言

在前面的章节中，我们已经拥有了域名和 SSL 证书。本章将配置 Nginx 作为反向代理服务器，为家庭服务（如 Plex 媒体服务器）提供安全的 HTTPS 访问。

### 什么是反向代理？

反向代理是位于后端服务器前面的服务器，它接收客户端请求并转发给后端服务器，然后将响应返回给客户端。

```
客户端 ───HTTPS请求───> Nginx ───转发请求───> 后端服务（Plex/Homepage）
         (443端口)    (反向代理)              (内部地址)
```

反向代理的好处：

- **统一入口**：所有服务通过同一个域名访问
- **SSL 终端**：在 Nginx 处理 HTTPS，后端可使用 HTTP
- **安全隔离**：后端服务不直接暴露在公网
- **负载均衡**：可以分发请求到多个后端

> **重要**：本章配置的 Nginx 会监听在 `8443` 端口（而非标准的 443 端口），因为在下一章中我们会使用 Nginx Stream 模块在 443 端口进行 SNI 分流。

---

### 第一步：安装 Nginx 和 Stream 模块

首先安装 Nginx 及其 Stream 模块（用于下一章的 SNI 分流）：

```shell
# 安装 nginx 和 stream 模块
apt install -y nginx libnginx-mod-stream
```

安装完成后验证：

```shell
# 查看 nginx 版本
nginx -v

# 确认 stream 模块已安装
nginx -V 2>&1 | grep -o with-stream
```

如果输出包含 `with-stream`，说明 Stream 模块已正确安装。

> **说明**：`libnginx-mod-stream` 是 Nginx 的 Stream 模块，用于处理 TCP/UDP 层的流量转发。我们将在[下一章](/posts/vps-xray-06-nginx-stream-sni)使用它实现 443 端口的 SNI 分流。

---

### 第二步：优化 Nginx 主配置

> **关于 nginxconfig.io**
>
> [nginxconfig.io](https://www.digitalocean.com/community/tools/nginx) 是 DigitalOcean 提供的一个优秀的 Nginx 配置生成工具。它可以根据你的需求生成安全、高性能的 Nginx 配置，包括 SSL 设置、安全头、反向代理等。
>
> 本教程中的配置片段参考了该工具的最佳实践，基于 nginxconfig 模版进行了少量微调。

首先，我们需要优化 Nginx 的主配置文件。

```shell
# 备份原配置
cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak

# 编辑主配置
vim /etc/nginx/nginx.conf
```

将内容替换为：

```nginx
user                 www-data;
pid                  /run/nginx.pid;
worker_processes     auto;
worker_rlimit_nofile 65535;

# 加载模块（stream 模块将在下一章使用）
include              /etc/nginx/modules-enabled/*.conf;

events {
    multi_accept       on;
    worker_connections 65535;
    use                epoll;
}

http {
    # 基础设置
    charset                utf-8;
    sendfile               on;
    tcp_nopush             on;
    tcp_nodelay            on;
    server_tokens          off;
    log_not_found          off;
    types_hash_max_size    2048;
    types_hash_bucket_size 64;
    client_max_body_size   16M;

    # MIME 类型
    include                mime.types;
    default_type           application/octet-stream;

    # 日志格式（支持 Proxy Protocol）
    log_format main '[$time_local] $proxy_protocol_addr "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent"';

    # 日志文件
    access_log /var/log/nginx/access.log main;
    error_log  /var/log/nginx/error.log notice;

    # SSL 会话缓存
    ssl_session_timeout    1d;
    ssl_session_cache      shared:SSL:10m;
    ssl_session_tickets    off;

    # SSL 协议和加密套件（Mozilla Intermediate 配置）
    ssl_protocols          TLSv1.2 TLSv1.3;
    ssl_ciphers            ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # OCSP Stapling
    ssl_stapling           on;
    ssl_stapling_verify    on;
    resolver               1.1.1.1 8.8.8.8 valid=60s;
    resolver_timeout       2s;

    # WebSocket 支持
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }

    # 加载配置文件
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

#### 配置说明

| 配置项                     | 说明                                |
| -------------------------- | ----------------------------------- |
| `worker_processes auto`    | 自动根据 CPU 核心数设置工作进程     |
| `worker_connections 65535` | 每个工作进程最大连接数              |
| `use epoll`                | 使用 epoll 事件模型（Linux 高性能） |
| `server_tokens off`        | 隐藏 Nginx 版本号                   |
| `ssl_session_cache`        | SSL 会话缓存，提升性能              |
| `ssl_protocols`            | 只启用 TLS 1.2 和 1.3               |

---

### 第三步：创建通用配置片段

为了避免重复配置，我们创建一些可复用的配置片段。

```shell
# 创建配置目录
mkdir -p /etc/nginx/nginxconfig.io
```

#### 2.1 安全头配置

```shell
vim /etc/nginx/nginxconfig.io/security.conf
```

```nginx
# 安全响应头
add_header X-XSS-Protection          "1; mode=block" always;
add_header X-Content-Type-Options    "nosniff" always;
add_header Referrer-Policy           "no-referrer-when-downgrade" always;
add_header Content-Security-Policy   "default-src 'self' http: https: ws: wss: data: blob: 'unsafe-inline'; frame-ancestors 'self';" always;
add_header Permissions-Policy        "interest-cohort=()" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# 禁止访问隐藏文件（.git, .env 等）
location ~ /\.(?!well-known) {
    deny all;
}
```

#### 2.2 反向代理配置

```shell
vim /etc/nginx/nginxconfig.io/proxy.conf
```

```nginx
# HTTP 版本和缓存
proxy_http_version                 1.1;
proxy_cache_bypass                 $http_upgrade;

# 代理 SSL
proxy_ssl_server_name              on;

# 代理头
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        $connection_upgrade;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;

# 超时设置
proxy_connect_timeout              60s;
proxy_send_timeout                 60s;
proxy_read_timeout                 60s;
```

#### 2.3 通用配置

```shell
vim /etc/nginx/nginxconfig.io/general.conf
```

```nginx
# favicon.ico
location = /favicon.ico {
    log_not_found off;
}

# robots.txt
location = /robots.txt {
    log_not_found off;
}

# gzip 压缩
gzip            on;
gzip_vary       on;
gzip_proxied    any;
gzip_comp_level 6;
gzip_types      text/plain text/css text/xml application/json application/javascript application/rss+xml application/atom+xml image/svg+xml;
```

---

### 第四步：配置默认拒绝站点

为了安全起见，我们需要配置一个默认服务器，拒绝未知域名的访问。

```shell
vim /etc/nginx/conf.d/restrict.conf
```

```nginx
# HTTP 默认服务器 - 拒绝所有未匹配的请求
server {
    listen      80 default_server;
    listen      [::]:80 default_server;
    server_name _;
    return      444;
}

# HTTPS 默认服务器 - 拒绝未匹配的 SNI
# 注意：监听 8443 端口，后续由 stream 模块转发
server {
    listen               127.0.0.1:8443 ssl http2 proxy_protocol default_server;
    set_real_ip_from     127.0.0.1;
    real_ip_header       proxy_protocol;
    ssl_reject_handshake on;
    return               444;
}
```

> **说明**：
>
> - `return 444` 是 Nginx 特有的状态码，表示直接关闭连接，不返回任何响应
> - `ssl_reject_handshake on` 拒绝 TLS 握手，比返回错误页面更安全
> - `proxy_protocol` 表示这个端口接收 Proxy Protocol 头（用于获取真实客户端 IP）
>
> 关于 Proxy Protocol 的详细原理，请参考[下一章：Nginx Stream SNI 分流](/posts/vps-xray-06-nginx-stream-sni#第一步理解-proxy-protocol)。

---

### 第五步：配置反向代理站点

现在我们来配置具体的反向代理站点。以博客服务为例：

```shell
vim /etc/nginx/sites-available/blog.conf
```

```nginx
# Blog 后端服务器
upstream blog_backend {
    # 替换为你的后端服务地址（如家庭服务器上的博客服务）
    server home.example.com:3000;
    keepalive 8;
}

# HTTPS 服务器
server {
    # 监听 8443 端口，启用 SSL、HTTP/2 和 Proxy Protocol
    listen 127.0.0.1:8443 ssl http2 proxy_protocol;
    server_name blog.example.com;

    # 从 Proxy Protocol 获取真实 IP
    set_real_ip_from           127.0.0.1;
    real_ip_header             proxy_protocol;

    # SSL 证书配置
    ssl_certificate     /etc/acme-certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/acme-certs/example.com/key.pem;

    # 安全配置
    include             nginxconfig.io/security.conf;

    # 反向代理配置
    location / {
        proxy_pass http://blog_backend;
        proxy_set_header Host $host;
        include nginxconfig.io/proxy.conf;
    }

    # 通用配置
    include nginxconfig.io/general.conf;
}

# HTTP 重定向到 HTTPS
server {
    listen      80;
    server_name blog.example.com;
    return      301 https://$host$request_uri;
}
```

#### 配置说明

| 配置项                  | 说明                                 |
| ----------------------- | ------------------------------------ |
| `listen 127.0.0.1:8443` | 只监听本地，由 stream 模块转发       |
| `proxy_protocol`        | 接收 Proxy Protocol 以获取真实 IP    |
| `set_real_ip_from`      | 信任来自 127.0.0.1 的 Proxy Protocol |
| `keepalive 8`           | 与后端保持 8 个长连接                |

---

### 第六步：启用站点配置

创建符号链接启用站点：

```shell
# 创建 sites-enabled 目录（如果不存在）
mkdir -p /etc/nginx/sites-enabled

# 删除默认站点（如果存在）
rm -f /etc/nginx/sites-enabled/default

# 启用 Blog 站点
ln -sf /etc/nginx/sites-available/blog.conf /etc/nginx/sites-enabled/
```

---

### 第七步：测试并重启 Nginx

```shell
# 测试配置语法
nginx -t
```

如果显示 `syntax is ok` 和 `test is successful`，说明配置正确。

```shell
# 重启 Nginx
systemctl restart nginx

# 查看状态
systemctl status nginx
```

---

### 第八步：验证配置

由于我们的 Nginx 监听在 8443 端口，且需要 Proxy Protocol，目前还无法直接访问测试。

可以检查端口监听情况：

```shell
# 查看 Nginx 监听的端口
ss -tlnp | grep nginx
```

你应该能看到：

```
LISTEN  0  511  127.0.0.1:8443  ...  nginx
LISTEN  0  511  0.0.0.0:80      ...  nginx
```

在下一章配置完 Nginx Stream SNI 分流后，就可以通过 443 端口正常访问了。

---

## 添加更多站点

如果你需要添加更多反向代理站点，可以按照类似的模式：

```shell
vim /etc/nginx/sites-available/dashboard.conf
```

```nginx
upstream dashboard_backend {
    server 192.168.1.100:8080;
    keepalive 8;
}

server {
    listen 127.0.0.1:8443 ssl http2 proxy_protocol;
    server_name dashboard.example.com;

    set_real_ip_from           127.0.0.1;
    real_ip_header             proxy_protocol;

    ssl_certificate     /etc/acme-certs/example.com/fullchain.pem;
    ssl_certificate_key /etc/acme-certs/example.com/key.pem;

    include             nginxconfig.io/security.conf;

    location / {
        proxy_pass http://dashboard_backend;
        proxy_set_header Host $host;
        include nginxconfig.io/proxy.conf;
    }

    include nginxconfig.io/general.conf;
}

server {
    listen      80;
    server_name dashboard.example.com;
    return      301 https://$host$request_uri;
}
```

启用站点：

```shell
ln -sf /etc/nginx/sites-available/dashboard.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## Nginx 目录结构

配置完成后，你的 Nginx 目录结构应该如下：

```
/etc/nginx/
├── nginx.conf                 # 主配置文件
├── conf.d/
│   └── restrict.conf         # 默认拒绝配置
├── sites-available/
│   ├── blog.conf             # Blog 反向代理
│   └── dashboard.conf        # Dashboard 反向代理
├── sites-enabled/
│   ├── blog.conf -> ../sites-available/blog.conf
│   └── dashboard.conf -> ../sites-available/dashboard.conf
├── nginxconfig.io/
│   ├── security.conf         # 安全头配置
│   ├── proxy.conf            # 代理配置
│   └── general.conf          # 通用配置
└── modules-enabled/
    └── (下一章配置 stream 模块)
```

---

## 常见问题

**问题 1：nginx -t 报错 "unknown directive"**

确保已安装相关模块：

```shell
apt install libnginx-mod-http-headers-more-filter
```

**问题 2：SSL 证书找不到**

确认证书路径正确，且权限允许 Nginx 读取：

```shell
ls -la /etc/acme-certs/example.com/
```

**问题 3：后端服务无法连接**

- 确认后端服务正在运行
- 确认防火墙允许访问
- 使用 `curl` 测试后端连接

---

## 总结

在本章中，我们完成了：

- ✅ 优化 Nginx 主配置
- ✅ 创建可复用的配置片段（安全、代理、通用）
- ✅ 配置默认拒绝未知域名访问
- ✅ 配置 Blog 等服务的反向代理
- ✅ 理解 Proxy Protocol 的作用

目前 Nginx 监听在 8443 端口，等待 Stream 模块的 SNI 分流。在下一章中，我们将配置 Nginx Stream 模块，实现：

- 443 端口的 SNI 分流
- Xray 流量和反向代理流量共存
- Xray 使用自己的证书（"偷自己证书"）

---

**上一篇**：[VPS 节点搭建系列教程（五）：使用 acme.sh 申请免费 SSL 证书](/posts/vps-xray-04-acme-certificate)

**下一篇**：[VPS 节点搭建系列教程（七）：Nginx Stream SNI 分流实现 443 端口复用](/posts/vps-xray-06-nginx-stream-sni)
