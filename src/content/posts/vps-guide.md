---
title: VPS 从零到一：多功能服务器搭建指南 (Nginx, Docker, sing-box & 泛域名证书)
date: 2025-07-08
lastMod: 2025-12-24
summary: 详细记录从零开始搭建多功能VPS服务器的完整过程，包括自动化SSL证书管理、sing-box代理服务部署、Nginx反向代理配置等，打造集网络优化、媒体服务和API代理于一体的多功能服务器。
category: 教程
tags:
  [VPS, Linux, Nginx, Docker, sing-box, SSL证书, 反向代理, VLESS, Hysteria2, acme.sh, Cloudflare]
sticky: 2
---

## 前言

由于目前跨省跨运营商限速严重，导致我即使宽带有公网ip，在跨运营商的情况下网络质量惨不忍睹。

因此，我入坑了优化线路的VPS，利用其CN2GIA+CMIN2的三网优化，通过Nginx反向代理Plex作为我的plex中转服务器，有效改善了跨网环境下的plex播放体验。

同时，使用nginx作为Movie Pilot的企业微信通知代理。

顺便再使用sing-box自建了节点。

将此过程记录为下文内容。

我们将在这篇指南中完成以下核心任务：

1.  **系统初始化**：更新系统并安装必备工具。
2.  **获取泛域名证书**：使用 `acme.sh` 和 Cloudflare API 自动申请并续签免费的 Let's Encrypt 泛域名 SSL 证书。
3.  **部署核心服务**：安装并配置强大的 `sing-box`，同时启用 VLESS-Reality 和 Hysteria2 两种协议。
4.  **配置 Web 反向代理**：利用 Nginx 为其他服务（如 Plex、企业微信 API）提供安全的 HTTPS 访问。

**准备工作：**

在开始之前，请确保你拥有：

- 一台全新的 VPS（推荐使用 Debian 12 系统）。
- 一个属于你自己的域名，并已将其 DNS 解析托管到 Cloudflare。
- Cloudflare 账户的邮箱和 Global API Key 或专用的 API Token。

让我们开始吧！

---

### 第一步：基础环境准备

首先，登录你的 VPS，更新软件包列表并升级所有已安装的软件，确保系统处于最新状态。

```shell
# 更新软件包列表
apt update

# 升级已安装的软件包
apt upgrade -y

# 安装一些必备工具
# curl: 用于下载文件
# sudo: 用于权限管理
# socat: acme.sh 申请证书时可能需要的依赖
# nginx: 我们将用它作为反向代理服务器
apt install curl sudo socat nginx -y
```

### 第二步：安装 Docker

我们使用官方提供的一键安装脚本来安装 Docker。

```shell
# 下载 Docker 安装脚本
curl -fsSL https://get.docker.com -o get-docker.sh

# 执行脚本进行安装
sh get-docker.sh
```

安装完成后，Docker 服务会自动启动。

---

### 第三步：配置泛域名 SSL 证书 (acme.sh)

为了后续服务的安全，我们需要配置 HTTPS。使用 `acme.sh` 可以自动化申请和续签 Let's Encrypt 提供的免费 SSL 证书。我们将申请一张泛域名证书（`*.example.com`），这样你所有的子域名都可以使用它，一劳永逸。

1.  **安装 acme.sh**
    将 `my@example.com` 替换成你自己的邮箱，用于接收证书到期提醒。

    ```shell
    curl https://get.acme.sh | sh -s email=my@example.com
    ```

2.  **重载 Shell 环境并配置自动更新**
    安装脚本会自动将 `acme.sh` 的路径添加到环境变量中，我们需要重新加载 `~/.bashrc` 使其生效。

    ```shell
    # 使 acme.sh 命令立即生效
    source ~/.bashrc

    # 开启 acme.sh 的自动更新功能
    acme.sh --upgrade --auto-upgrade
    ```

3.  **配置 Cloudflare API 并申请证书**
    `acme.sh` 需要通过 Cloudflare 的 API 来自动添加 DNS 记录，以验证你对域名的所有权（DNS-01 质询）。

    > **如何获取 Cloudflare API Token？**
    > 登录 Cloudflare -> 我的个人资料 -> API 令牌 -> 创建令牌 -> 使用“编辑区域 DNS”模板 -> 在“区域资源”中选择你的域名 -> 创建令牌。创建后请立即复制保存好。

    在终端中执行以下命令，将 `你的CloudFlare Token` 和 `你的邮箱` 替换为真实信息。

    ```shell
    export CF_Token="你的CloudFlare_Token"
    export CF_Email="你的Cloudflare账户邮箱"
    ```

    现在，开始申请证书。请将 `example.com` 替换为你的主域名。`--keylength ec-384` 表示我们使用 ECC 384 位密钥，安全性更高。

    ```shell
    acme.sh --issue -d "example.com" -d "*.example.com" --dns dns_cf --keylength ec-384
    ```

    如果一切顺利，你将看到证书成功生成的提示。证书文件默认存放在 `~/.acme.sh/` 目录下。

4.  **安装证书**
    将上一步申请的证书安装到 `/etc/acme-certs/example.com` 目录下，后续应用（如Nginx、sing-box）中统一使用该路径下的证书文件。

    > **坑点**：acme.sh 的安装命令 (`--install-cert`) 只能使用一次。如果你后续为另一个服务（如 sing-box）再次单独执行安装命令，会覆盖掉之前的配置，导致前一个服务（如 Nginx）在证书续签后无法自动获得新证书。
    > **因此，必须在这里一次性为所有服务同时指定安装路径和 reloadcmd，不能分别安装**。

    配置的reloadcmd命令会在证书自动续签后自动重启Nginx和sing-box服务。

    ```shell
    # 创建统一的证书存放目录
    mkdir -p /etc/acme-certs/example.com

    # 安装证书
    acme.sh --install-cert -d example.com --ecc \
    --cert-file      /etc/acme-certs/example.com/cert.pem \
    --key-file       /etc/acme-certs/example.com/key.pem \
    --fullchain-file /etc/acme-certs/example.com/fullchain.pem \
    --reloadcmd      "systemctl reload nginx && systemctl restart sing-box"
    ```

    **注意**：这里我们为 `example.com` 这个主域安装了证书。由于申请的是泛域名证书，它对所有子域名（如 `plex.example.com`）都有效。

---

### 第四步：配置 Nginx 反向代理

Nginx 是一个高性能的 Web 服务器和反向代理。我们将用它来代理两个服务：Plex 媒体服务器和一个企业微信通知 API。

1.  **配置 Plex 反向代理**
    创建一个新的 Nginx 配置文件。

    ```shell
    vim /etc/nginx/sites-available/plex.conf
    ```

    将以下内容粘贴进去，并替换 `plex.example.com` 和 `https://yourplex.com` 为你自己的 Plex 域名和实际的 Plex 服务器地址。

    ```nginx
    # HTTP (80) 请求自动跳转到 HTTPS (8443)
    server {
        listen 80;
        server_name plex.example.com;
        return 301 https://$host:8443$request_uri;
    }

    # Plex HTTPS 服务主配置
    server {
        listen 8443 ssl http2;
        server_name plex.example.com;

        client_max_body_size 100M; // 允许上传的最大文件大小，如字幕

        # --- SSL 证书配置 ---
        ssl_certificate /etc/acme-certs/example.com/fullchain.pem;
        ssl_certificate_key /etc/acme-certs/example.com/key.pem;

        # --- SSL 优化与安全配置 ---
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;
        ssl_stapling on;
        ssl_stapling_verify on;
        resolver 8.8.8.8 1.1.1.1 valid=300s;
        resolver_timeout 5s;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # --- Plex 特定的反向代理配置 ---
        location / {
            # 代理到你的 Plex 服务器地址
            proxy_pass https://yourplex.com;

            # --- 重要的代理头部信息 ---
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Plex-Client-Identifier $http_x_plex_client_identifier;
            proxy_set_header Cookie $http_cookie;

            # --- WebSocket 支持 ---
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';

            # --- 针对流媒体的优化 ---
            proxy_buffering off; # 关闭代理缓冲，对流媒体至关重要
            proxy_read_timeout 36000s; # 防止长时间观看导致连接中断
            proxy_redirect off;
        }
    }
    ```

2.  **配置企业微信 API 代理**

    ```shell
    vim /etc/nginx/sites-available/qyapi.conf
    ```

    粘贴以下配置，并将 `qyapi.example.com` 替换为你的域名。

    ```nginx
    # HTTPS 服务器块：处理 SSL 和反向代理
    server {
        listen 8443 ssl http2; # 与 Plex 共用 8443 端口
        server_name qyapi.example.com; # 替换为你的域名

        # --- SSL 证书配置 ---
        ssl_certificate /etc/acme-certs/example.com/fullchain.pem;
        ssl_certificate_key /etc/acme-certs/example.com/key.pem;

        # --- SSL 性能和安全优化 ---
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers 'TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256';
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;

        # --- 反向代理配置 ---
        # 只代理特定的 API 路径
        location ~ ^/cgi-bin/(gettoken|message/send|menu/create)$ {
            proxy_pass https://qyapi.weixin.qq.com;

            # --- 代理头部设置 ---
            proxy_set_header Host qyapi.weixin.qq.com;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 增加代理缓冲区大小
            proxy_buffers 4 256k;
            proxy_buffer_size 128k;
            proxy_busy_buffers_size 256k;
        }

        # --- 其他所有路径返回 404 ---
        location / {
            return 404;
        }
    }
    ```

3.  **应用 Nginx 配置**
    将写好的配置文件链接到 `sites-enabled` 目录，使其生效。

    ```shell
    ln -s /etc/nginx/sites-available/plex.conf /etc/nginx/sites-enabled/
    ln -s /etc/nginx/sites-available/qyapi.conf /etc/nginx/sites-enabled/
    ```

    最后，检查配置语法是否正确，然后重启 Nginx。

    ```shell
    # 测试 Nginx 配置
    nginx -t
    # 如果显示 "syntax is ok" 和 "test is successful"，则可以重启

    # 重启 Nginx 服务
    systemctl restart nginx
    ```

---

### 第五步：安装与配置 sing-box

`sing-box` 是一个功能强大且高度可配置的代理平台。我们将配置 VLESS-Reality 和 Hysteria2 两种协议。

1. **安装 sing-box**
   使用官方一键脚本进行安装。

   ```shell
   curl -fsSL https://sing-box.app/install.sh | sh
   ```

   安装后，将其设置为开机自启。

   ```shell
   systemctl enable sing-box
   ```

2. **配置证书路径**
   我们已经在“第三步”中配置了泛域名证书及其自动安装路径（`/etc/acme-certs/example.com`），sing-box 将直接使用这些证书。

   **注意：** Hysteria2 协议需要使用真实的 SSL 证书。

3. **配置 sing-box**
   编辑 `sing-box` 的配置文件：

   ```shell
   vim /etc/sing-box/config.json
   ```

   在编辑前，我们需要生成一些必要的密钥：

   ```shell
   # 生成 Reality 公私钥对
   sing-box generate reality-keypair
   # 输出示例：
   # Private key: OGF8xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx_xxxxxxxx
   # Public key:  df4dxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx_xxxxxxxx

   # 生成一个随机的 short_id (可选，但推荐)
   sing-box generate rand 8 --hex
   # 输出示例：
   # a1b2c3d4e5f6a7b8

   # 生成一个随机密码用于 Hysteria2
   sing-box generate rand 16
   # 输出示例：
   # YourStrongPassword123
   ```

   现在，将以下 JSON 内容粘贴到 `config.json` 文件中，并根据注释和刚刚生成的密钥替换相应字段：

   ```json
   {
     "log": {
       "level": "info",
       "timestamp": true
     },
     "dns": {
       "servers": [
         {
           "type": "udp",
           "server": "8.8.8.8"
         }
       ]
     },
     "inbounds": [
       {
         "type": "vless",
         "listen": "::",
         "listen_port": 443,
         "users": [
           {
             "uuid": "换成你自己的UUID", // 可使用 `sing-box generate uuid` 生成
             "flow": "xtls-rprx-vision"
           }
         ],
         "tls": {
           "enabled": true,
           "server_name": "sub.example.com", // 替换为你的子域名，不能是通配符
           "reality": {
             "enabled": true,
             "handshake": {
               "server": "www.microsoft.com", // 伪装的目标网站，要求支持TLS 1.3, X25519, H2
               "server_port": 443
             },
             "private_key": "粘贴你生成的Reality私钥", // 刚刚生成的 Private key
             "short_id": [
               "粘贴你生成的short_id" // 刚刚生成的 16 位 hex 字符串
             ]
           }
         }
       },
       {
         "type": "hysteria2",
         "listen": "::",
         "listen_port": 8444, // 自定义一个未被占用的端口
         "users": [
           {
             "password": "设置一个强大的密码" // 粘贴你刚刚生成的 Hysteria2 密码
           }
         ],
         "masquerade": "https://bing.com", // 可选，用于伪装
         "tls": {
           "enabled": true,
           "alpn": ["h3"],
           "certificate_path": "/etc/acme-certs/example.com/fullchain.pem",
           "key_path": "/etc/acme-certs/example.com/key.pem"
         }
       }
     ],
     "outbounds": [
       {
         "type": "direct",
         "tag": "direct"
       }
     ],
     "route": {}
   }
   ```

4. **启动 sing-box**
   保存配置文件后，启动并检查 `sing-box` 的状态。

   ```shell
   systemctl restart sing-box
   systemctl status sing-box
   ```

   如果状态显示 `active (running)`，则表示服务已成功启动。

5. **追踪 sing-box 日志**

   ```
   journalctl -u sing-box -o cat -f
   ```

---

## 总结

恭喜你！你已经成功地将一台全新的 VPS 打造成了一个功能强大的多用途服务器。

我们完成了：

- 系统的基础更新与软件安装。
- 通过 `acme.sh` 实现了泛域名证书的自动化管理。
- 部署了 `sing-box` 并提供了 VLESS-Reality 和 Hysteria2 两种高性能服务。
- 利用 Nginx 成功反向代理了 Plex 和企业微信 API，并为它们提供了安全的 HTTPS 访问。

现在，你的服务器已经准备就绪。你可以开始享受安全的网络访问，或者继续探索，在 Docker 的帮助下部署更多有趣的应用。祝你玩得开心！
