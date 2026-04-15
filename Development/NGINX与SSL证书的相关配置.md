---
Title: NGINX与SSL证书的相关配置
Draft: false
tags:
  - 运维
Author: Ruby Ceng
---

## 引言：

配置网关、前端部署、域名和 DNS 设置以及 SSL 证书配置。

## 配置 NGINX

### 安装

#### 使用系统服务进行安装

对于 Ubuntu/Debian：

```bash
# 1. 更新软件包列表
sudo apt update

# 2. 安装 Nginx
sudo apt install nginx -y
```

Nginx 默认端口为 HTTP (80)、HTTPS (443)，配置开放防火墙：

```bash
# 查看可用的应用配置
sudo ufw app list

# 允许 'Nginx Full' (包含 HTTP 和 HTTPS)
sudo ufw allow 'Nginx Full'

# 启用防火墙 (如果尚未启用)
sudo ufw enable

# 检查状态
sudo ufw status
```

配置默认开机自启以及相关命令：

```bash
# 启动 Nginx 服务
sudo systemctl start nginx

# 设置 Nginx 开机自启
sudo systemctl enable nginx

# 查看 Nginx 服务状态
sudo systemctl status nginx
# 按 'q' 退出状态查看

# 停止 Nginx 服务
sudo systemctl stop nginx

# 重启 Nginx 服务 (先停后启，服务会中断)
sudo systemctl restart nginx

# 重新加载配置 (推荐！不中断服务，平滑加载新配置)
sudo systemctl reload nginx
```

#### 使用 Docker Compose 进行配置

**Docker Compose 文件：**

```yaml
version: "3.8"

services:
  nginx:
    image: nginx:latest # 使用最新的官方 Nginx 镜像
    container_name: my_nginx_server # 给容器起个名字
    ports:
      - "80:80" # 将宿主机的 80 端口映射到容器的 80 端口
      - "443:443" # 将宿主机的 443 端口映射到容器的 443 端口
    volumes:
      # 挂载你的网站文件到容器的默认网站目录
      - ./www:/usr/share/nginx/html
      # 挂载你的 Nginx 配置文件到容器的配置目录
      - ./nginx/conf.d:/etc/nginx/conf.d
    restart: always # 如果容器挂了，自动重启
```

**相关命令：**

```bash
# 在后台启动并构建服务
docker-compose up -d

# 查看正在运行的容器服务
docker-compose ps

# 查看 Nginx 服务的日志
docker-compose logs -f nginx

# 停止并移除容器、网络等资源
docker-compose down

# 如果你修改了 Nginx 配置文件 (e.g., default.conf)
# 你需要重启容器来让配置生效
docker-compose restart nginx
```

#### 直接使用 NPM

详情查看配置 NPM（Nginx Proxy Manager）章节。

### 相关配置

#### 目录结构

- `/etc/nginx/nginx.conf`：主配置文件
- `/etc/nginx/sites-available/`：存放所有可用的网站配置文件
- `/etc/nginx/sites-enabled/`：存放已启用的网站配置 (通常是 sites-available 中文件的符号链接)

#### 配置文件 (nginx.conf)

```nginx
server {
    listen 80;
    listen [::]:80;

    # 网站根目录
    root /var/www/your_domain;
    index index.html index.htm;

    # 你的域名
    server_name your_domain.com www.your_domain.com;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 测试并重载 Nginx

```bash
# 测试 Nginx 配置语法是否有误
sudo nginx -t

# 如果测试成功 (syntax is ok, test is successful)，则重载 Nginx
sudo systemctl reload nginx
```

## 配置 NPM（Nginx Proxy Manager）

### Docker Compose 部署

**Docker Compose 文件内容：**

```yaml
version: "3.8"
services:
  app:
    image: "jc21/nginx-proxy-manager:latest"
    container_name: "nginx-proxy-manager"
    restart: unless-stopped
    ports:
      # 这些是公共端口，用于处理代理流量
      - "80:80" # HTTP 端口
      - "443:443" # HTTPS 端口
      # 这是管理员后台访问端口
      - "81:81"
    volumes:
      # 将配置数据和 SSL 证书持久化到宿主机
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

**启动命令：**

```bash
docker-compose up -d
```

### 访问 Admin UI 界面

**首次登录：**

- 打开浏览器，访问 http://<你服务器的 IP>:81
- 使用默认的管理员账户登录：
  - **Email:** admin@example.com
  - **Password:** changeme
- 登录后，系统会立即要求你修改默认的用户名、邮箱和密码

## 配置 SSL/TLS 证书

SSL 证书是保障网站安全的重要工具，让网站从不安全的 HTTP 升级为安全的 HTTPS。核心作用是在 TCP 层之上建立一个加密通道，确保数据在传输过程中的安全性。

**工作原理：** 当客户端（如浏览器）与服务器建立 TCP 连接后，SSL/TLS 协议会启动握手过程。在这个过程中，双方协商加密算法（公私钥进行身份验证以及加密）、交换密钥，并通过证书验证服务器的身份（有时也会验证客户端）。握手完成后，数据传输通过加密通道进行，防止被窃听或篡改。

**加密位置：** TCP 负责可靠的数据传输，但本身不提供加密。SSL/TLS 在 TCP 之上添加了安全层，因此 HTTPS（HTTP over SSL/TLS）能够实现安全通信。

> **备注：** 传统 HTTP 协议没有加密机制，它直接在 TCP 层之上以明文形式传输数据。传统 HTTP 本身无法解决这些问题。为了弥补这些缺陷，人们引入了 SSL/TLS，推出了 HTTPS 协议。通过网络抓包可以捕获 HTTP 的明文数据。ARP 欺骗 (ARP Spoofing)，网关上进行端口镜像等。

**证书分类：**

- **自签证书 (内网，测试)**：自签证书是由自己生成的证书，不依赖任何第三方机构（证书颁发机构，简称 CA）。这种证书通常用于测试或内网环境，但因为它不被浏览器或操作系统信任，在公网上使用时，浏览器会显示"不安全"或"证书不受信任"的警告。

- **第三方证书 (Let's Encrypt、DigiCert)**：Let's Encrypt 是一个免费、自动化、开放的证书颁发机构（CA）。它签发的证书是受信任的第三方证书，而不是自签证书。Let's Encrypt 的根证书已被主流浏览器和操作系统信任，因此它签发的证书在公网上是受信任的，不会显示警告。

### 使用 Let's Encrypt 进行证书签发

```bash
# 下载 Let's Encrypt 客户端
# CentOS
sudo yum install epel-release
sudo yum install certbot

# 生成证书 默认情况下，证书存放在 /etc/letsencrypt/live/
sudo certbot certonly -d example.com -d www.example.com

# 配置 NGINX
sudo vi /etc/nginx/sites-available/default

# server {
#     listen 80;
#     server_name example.com;
#     # 重定向 HTTP 到 HTTPS
#     return 301 https://$host$request_uri;
# }

# server {
#     listen 443 ssl;
#     server_name example.com;
#     ssl_certificate /etc/nginx/ssl/cert.pem;      # 证书路径
#     ssl_certificate_key /etc/nginx/ssl/key.pem;   # 私钥路径
#     location / {
#         root /var/www/html;  # 网站根目录
#         index index.html;    # 默认页面
#     }
# }

# 重启 NGINX
sudo systemctl restart nginx
```

### 使用 OpenSSL 生成自签证书

```bash
# 创建存储证书的目录
sudo mkdir /etc/nginx/ssl

# 生成证书
sudo openssl req -x509 -newkey rsa:4096 -keyout /etc/nginx/ssl/key.pem -out /etc/nginx/ssl/cert.pem -days 365 -nodes

# 配置 NGINX 同上

# 重启 NGINX
sudo systemctl restart nginx
```

> **备注：** 我们所说的"SSL 证书"几乎都是指"TLS 证书"：
>
> **SSL (Secure Sockets Layer)** 是早期的加密协议。它由 Netscape 公司开发，最初用于保护 Web 浏览器的通信安全。SSL 协议有多个版本，例如 SSL 2.0 和 SSL 3.0。
>
> **TLS (Transport Layer Security)** 是 SSL 的继任者和标准化版本。它由互联网工程任务组 (IETF) 在 SSL 3.0 的基础上进行标准化和改进，并更名为 TLS。TLS 协议也有多个版本，例如 TLS 1.0, TLS 1.1, TLS 1.2, TLS 1.3 (目前最新最安全的版本是 TLS 1.3)。

### 会话密钥的协商过程

服务器的公钥主要用于**密钥交换**阶段，而非直接加密所有数据。服务器的公钥是公开的，没错。网站的 SSL/TLS 证书中就包含了公钥，任何人都可以获取到。在 HTTPS 连接建立的握手过程中，客户端（你的浏览器或 App）会使用服务器的公钥来加密一些信息，例如生成一个**会话密钥 (Session Key)**，并安全地发送给服务器。

**重点在于会话密钥 (Session Key)：** 每个连接独一无二的对称密钥，HTTPS 真正用来加密用户和服务器之间传输的所有数据的，是会话密钥。会话密钥是对称加密密钥。对称加密速度快，适合加密大量数据。会话密钥是临时生成的，并且每个 HTTPS 连接都使用不同的会话密钥。这才是区分用户和保证连接独立性的关键。

**会话密钥的协商过程 (简化版)：**

1. 客户端连接 HTTPS 服务器
2. 服务器发送 SSL/TLS 证书，包含公钥
3. 客户端验证证书有效性
4. 客户端生成一个随机的会话密钥 (Session Key)
5. 客户端使用服务器的公钥加密这个会话密钥，并发送给服务器（这里用到了非对称加密）
6. 服务器使用自己的私钥解密收到的信息，得到会话密钥（只有服务器拥有私钥才能解密）
7. 现在客户端和服务器都拥有了同一个会话密钥，但这个密钥只有它们双方知道
8. 后续所有的数据传输，客户端和服务器都使用这个会话密钥进行对称加密和解密

## 配置 Cloudflare 解析 CDN 与 SSL/TLS 证书

Cloudflare 是全球领先的 CDN 和网络安全服务提供商，为网站提供免费的 CDN 加速、DDoS 防护和 SSL/TLS 证书服务。通过 Cloudflare，可以实现全站 HTTPS 加密、性能优化和安全防护。

### 注册和基础配置

#### 注册 Cloudflare 账户

1. **注册账户**

   - 访问 [https://www.cloudflare.com](https://www.cloudflare.com)
   - 使用邮箱注册账户并验证

2. **添加域名**

   ```bash
   # 在 Cloudflare 控制台添加您的域名（不要添加 www 前缀）
   # 例如：example.com（正确）
   # 避免：www.example.com（错误）
   ```

3. **选择套餐**
   - **Free（免费）**：适合个人博客和小型网站
   - **Pro**：适合中小企业网站
   - **Business/Enterprise**：适合大型企业

#### 配置 DNS 解析

1. **DNS 记录扫描**

   Cloudflare 会自动扫描现有的 DNS 记录，检查并确认记录是否正确。

2. **更改 Nameservers**

   ```bash
   # Cloudflare 会提供两个 Nameserver，例如：
   # mia.ns.cloudflare.com
   # noah.ns.cloudflare.com

   # 在域名注册商处修改 DNS 服务器为 Cloudflare 提供的 Nameserver
   # 修改后需要等待 24-48 小时完全生效
   ```

3. **验证 DNS 生效**

   ```bash
   # 使用 dig 命令检查 DNS 解析
   dig example.com

   # 或使用 nslookup
   nslookup example.com

   # 检查 Nameserver 是否已更新
   dig NS example.com
   ```

### SSL/TLS 证书配置

#### Universal SSL 证书

Cloudflare 免费提供 Universal SSL 证书，自动为所有通过 Cloudflare 的域名启用 HTTPS。

1. **自动配置**

   - 域名添加到 Cloudflare 后会自动获得 Universal SSL 证书
   - 支持根域名和第一级通配符子域名
   - 证书自动续期，无需手动干预

2. **验证证书状态**
   ```bash
   # 在 Cloudflare 控制台查看：SSL/TLS > Edge Certificates
   # 确认 Universal SSL 证书状态为 "Active"
   ```

#### SSL/TLS 加密模式配置

Cloudflare 提供四种加密模式，需要根据源服务器配置选择合适的模式：

1. **Off（关闭）**

   ```nginx
   # 不推荐使用，所有连接都使用 HTTP
   # 源服务器配置：无需 SSL 证书
   ```

2. **Flexible（灵活）**

   ```nginx
   # 访客到 Cloudflare：HTTPS 加密
   # Cloudflare 到源服务器：HTTP 明文
   # 适用场景：源服务器无 SSL 证书的情况

   # Nginx 配置示例
   server {
       listen 80;
       server_name example.com www.example.com;

       location / {
           root /var/www/html;
           index index.html index.htm;
       }
   }
   ```

3. **Full（完全）**

   ```nginx
   # 访客到 Cloudflare：HTTPS 加密
   # Cloudflare 到源服务器：HTTPS 加密（可使用自签证书）

   # Nginx 配置示例
   server {
       listen 80;
       server_name example.com;
       return 301 https://$server_name$request_uri;
   }

   server {
       listen 443 ssl;
       server_name example.com www.example.com;

       # 使用自签证书或 Cloudflare Origin CA 证书
       ssl_certificate /etc/nginx/ssl/cert.pem;
       ssl_certificate_key /etc/nginx/ssl/key.pem;

       location / {
           root /var/www/html;
           index index.html index.htm;
       }
   }
   ```

4. **Full (Strict)（完全严格）**
   ```nginx
   # 访客到 Cloudflare：HTTPS 加密
   # Cloudflare 到源服务器：HTTPS 加密（必须使用受信任证书）
   # 推荐使用 Cloudflare Origin CA 证书
   ```

#### Cloudflare Origin CA 证书配置

Cloudflare Origin CA 提供免费的源服务器证书，专门用于 Cloudflare 到源服务器之间的加密。

1. **生成 Origin CA 证书**

   ```bash
   # 在 Cloudflare 控制台操作：
   # SSL/TLS > Origin Server > Create Certificate

   # 选择证书选项：
   # - 让 Cloudflare 生成私钥和 CSR（推荐）
   # - 或使用自己的私钥和 CSR

   # 配置域名覆盖范围：
   # - *.example.com（通配符）
   # - example.com（根域名）
   # - subdomain.example.com（特定子域名）

   # 选择证书有效期：
   # - 15 年（推荐）
   # - 10 年
   # - 5 年
   # - 1 年
   ```

2. **安装 Origin CA 证书**

   ```bash
   # 创建证书目录
   sudo mkdir -p /etc/nginx/ssl/cloudflare

   # 保存证书文件
   sudo nano /etc/nginx/ssl/cloudflare/cert.pem
   # 粘贴 Origin Certificate 内容

   # 保存私钥文件
   sudo nano /etc/nginx/ssl/cloudflare/key.pem
   # 粘贴 Private Key 内容

   # 下载并保存 Cloudflare Origin CA 根证书
   sudo wget -O /etc/nginx/ssl/cloudflare/origin_ca_ecc_root.pem \
   https://developers.cloudflare.com/ssl/static/origin_ca_ecc_root.pem

   # 设置文件权限
   sudo chmod 600 /etc/nginx/ssl/cloudflare/key.pem
   sudo chmod 644 /etc/nginx/ssl/cloudflare/cert.pem
   sudo chmod 644 /etc/nginx/ssl/cloudflare/origin_ca_ecc_root.pem
   ```

3. **配置 Nginx 使用 Origin CA 证书**

   ```nginx
   # /etc/nginx/sites-available/example.com
   server {
       listen 80;
       server_name example.com www.example.com;

       # HTTP 重定向到 HTTPS
       return 301 https://$server_name$request_uri;
   }

   server {
       listen 443 ssl http2;
       server_name example.com www.example.com;

       # Cloudflare Origin CA 证书配置
       ssl_certificate /etc/nginx/ssl/cloudflare/cert.pem;
       ssl_certificate_key /etc/nginx/ssl/cloudflare/key.pem;

       # SSL 优化配置
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_prefer_server_ciphers off;
       ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

       # 安全头部
       add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
       add_header X-Frame-Options DENY always;
       add_header X-Content-Type-Options nosniff always;

       # 网站配置
       root /var/www/example.com;
       index index.html index.htm index.php;

       location / {
           try_files $uri $uri/ =404;
       }

       # PHP 支持（如需要）
       location ~ \.php$ {
           fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
           fastcgi_index index.php;
           include fastcgi_params;
           fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
       }
   }
   ```

4. **验证配置并重启 Nginx**

   ```bash
   # 测试 Nginx 配置
   sudo nginx -t

   # 重新加载 Nginx 配置
   sudo systemctl reload nginx

   # 验证 SSL 证书
   openssl s_client -connect example.com:443 -servername example.com
   ```

### 高级配置和优化

#### Authenticated Origin Pulls 配置

为确保只有 Cloudflare 能够访问源服务器，可以配置 mTLS（双向 TLS 认证）。

1. **启用 Authenticated Origin Pulls**

   ```bash
   # 在 Cloudflare 控制台：SSL/TLS > Origin Server
   # 启用 "Authenticated Origin Pulls"
   ```

2. **下载 Cloudflare CA 证书**

   ```bash
   # 下载 Cloudflare 客户端证书
   sudo wget -O /etc/nginx/ssl/cloudflare/authenticated_origin_pull_ca.pem \
   https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem
   ```

3. **配置 Nginx mTLS**

   ```nginx
   server {
       listen 443 ssl http2;
       server_name example.com;

       # 原有的 SSL 证书配置
       ssl_certificate /etc/nginx/ssl/cloudflare/cert.pem;
       ssl_certificate_key /etc/nginx/ssl/cloudflare/key.pem;

       # 客户端证书验证配置
       ssl_client_certificate /etc/nginx/ssl/cloudflare/authenticated_origin_pull_ca.pem;
       ssl_verify_client on;

       # 其他配置...
   }
   ```

#### 页面规则和重定向配置

1. **强制 HTTPS**

   ```bash
   # 在 Cloudflare 控制台设置：
   # SSL/TLS > Edge Certificates > Always Use HTTPS：开启
   ```

2. **HSTS 配置**

   ```bash
   # SSL/TLS > Edge Certificates > HTTP Strict Transport Security (HSTS)
   # 启用 HSTS，设置 max-age 为 12 months
   # 可选择包含子域名
   ```

3. **自动 HTTPS 重写**
   ```bash
   # SSL/TLS > Edge Certificates > Automatic HTTPS Rewrites：开启
   # 自动将页面中的 HTTP 链接重写为 HTTPS
   ```

#### CDN 缓存优化

1. **缓存级别设置**

   ```bash
   # Caching > Configuration
   # Cache Level: Standard（标准缓存）
   # Browser Cache TTL: 4 hours（浏览器缓存时间）
   ```

2. **页面规则配置**

   ```bash
   # 静态资源缓存规则
   # URL: *.example.com/*.css
   # 设置：Cache Level = Cache Everything, Edge Cache TTL = 1 month

   # 动态内容规则
   # URL: *.example.com/api/*
   # 设置：Cache Level = Bypass
   ```

#### 安全配置

1. **防火墙规则**

   ```bash
   # Security > WAF
   # 启用 Web Application Firewall
   # 设置安全级别：Medium 或 High
   ```

2. **速率限制**

   ```bash
   # Security > WAF > Rate limiting rules
   # 创建规则限制每分钟请求数量
   ```

3. **Bot 管理**
   ```bash
   # Security > Bots
   # 配置 Bot Fight Mode 或 Super Bot Fight Mode
   ```

### 监控和维护

#### 分析和监控

1. **Analytics 数据查看**

   ```bash
   # Analytics & Logs > Web Analytics
   # 查看流量统计、缓存命中率、威胁阻止数量等
   ```

2. **Real-time Logs**
   ```bash
   # Analytics & Logs > Logs
   # 配置实时日志推送到外部服务
   ```

#### 证书续期和管理

1. **Universal SSL 自动续期**

   ```bash
   # Universal SSL 证书会自动续期，无需手动操作
   # 在 SSL/TLS > Edge Certificates 中监控状态
   ```

2. **Origin CA 证书管理**
   ```bash
   # Origin CA 证书有效期较长（最长 15 年）
   # 可在控制台撤销或重新生成证书
   # 建议定期轮换证书以提高安全性
   ```

#### 故障排除

1. **SSL 错误排查**

   ```bash
   # 检查证书链
   openssl s_client -connect example.com:443 -showcerts

   # 验证 SSL 配置
   curl -I https://example.com

   # 检查 Cloudflare SSL 状态
   dig example.com +short
   ```

2. **常见问题解决**

   ```bash
   # 重定向循环问题
   # 检查 SSL/TLS 加密模式是否正确配置

   # 502 错误
   # 检查源服务器是否正常运行
   # 验证 Origin CA 证书配置

   # DNS 解析问题
   # 确认 Nameserver 已正确更改
   # 检查 DNS 记录是否正确配置
   ```

---
