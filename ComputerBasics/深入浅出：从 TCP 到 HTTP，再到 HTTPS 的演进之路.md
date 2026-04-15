---
Title: 深入浅出：从 TCP 到 HTTP，再到 HTTPS 的演进之路
Draft: false
Tags:
  - 计算机网络
Author: Ruby Ceng
---

## 参考资料

- [Let's Encrypt 官方文档](https://letsencrypt.org/docs/)
- [OpenSSL 官方文档](https://www.openssl.org/docs/)
- [NGINX HTTPS 配置指南](https://nginx.org/en/docs/http/configuring_https_servers.html)

## 引言

在互联网世界中，我们每天都在进行着各种各样的网络活动：浏览网页、发送消息、在线购物等等。这些看似简单的操作背后，都离不开一系列复杂的网络协议在默默地工作。今天，我们就来深入浅出地聊聊互联网通信中最重要的几个协议：TCP、HTTP 和 HTTPS，并探索它们之间的关系与演进。

## 互联网通信的基石：TCP 协议

首先，我们不得不提到 **TCP (Transmission Control Protocol)**，即 **传输控制协议**。TCP 协议是互联网协议族 (TCP/IP) 中的核心协议之一，它位于 **传输层**，为应用程序提供 **可靠的、面向连接的、字节流传输** 服务。

- **可靠性 (Reliability):** TCP 协议通过 **序号 (Sequence Number)**、**确认应答 (Acknowledgement)**、**超时重传 (Retransmission)** 等机制，确保数据包能够 **可靠地、按顺序地** 送达目的地，不会丢失或乱序。
- **面向连接 (Connection-oriented):** 在使用 TCP 进行数据传输之前，通信双方需要先建立 **TCP 连接** (三次握手)，数据传输结束后再 **断开连接** (四次挥手)。这种面向连接的特性保证了通信的可靠性和有序性。
- **字节流传输 (Byte-stream):** TCP 将数据视为 **无结构的字节流** 进行传输，应用程序无需关心数据包的分割和重组，TCP 会自动处理。

**简单来说，TCP 协议就像是互联网上的“可靠快递公司”，它负责确保你的包裹 (数据) 安全、完整地送达目的地，并提供连接和跟踪服务。**

## 构建 Web 世界的语言：HTTP 协议

有了可靠的 TCP 协议作为基础，我们就可以在其之上构建 **应用层协议**，用于实现更高级的网络应用。其中，最重要、最广泛应用的就是 **HTTP (Hypertext Transfer Protocol)**，即 **超文本传输协议**。

HTTP 协议是用于在 **Web 浏览器** 和 **Web 服务器** 之间传输数据的 **应用层协议**，是构建万维网 (WWW) 的核心协议。

- **请求-响应模式 (Request-Response):** HTTP 采用 **客户端-服务器架构**，客户端 (通常是浏览器) 发起 **HTTP 请求 (Request)**，服务器接收并处理请求后，返回 **HTTP 响应 (Response)**。
- **无状态协议 (Stateless):** HTTP 是 **无状态的**，服务器 **不保存** 任何关于客户端的 **历史会话信息**。 每次 HTTP 请求都被视为一个独立的事务。如果需要保持会话状态 (例如用户登录状态)，需要通过 Cookie、Session、Token 等技术在应用层实现。
- **基于文本的协议 (Text-based):** HTTP 请求和响应消息都是 **基于文本** 的，人类可读，易于理解和调试。

**当我们使用浏览器访问网页时，背后就发生了一系列 HTTP 请求和响应：**

1.  **浏览器输入 URL 并按下回车:** 例如 `http://example.com/index.html`
2.  **浏览器解析 URL:** 识别协议 (HTTP)、域名 (example.com)、路径 (/index.html)。
3.  **DNS 查询:** 将域名解析为服务器 IP 地址。
4.  **建立 TCP 连接:** 浏览器与服务器在 80 端口建立 TCP 连接 (三次握手)。
5.  **构建 HTTP 请求消息:**

    ```
    GET /index.html HTTP/1.1
    Host: example.com
    User-Agent: ...
    Accept: text/html, ...

    ```

6.  **发送 HTTP 请求:** 浏览器通过 TCP 连接将请求消息发送给服务器。
7.  **服务器处理请求:** Web 服务器 (例如 Nginx, Apache) 接收并处理请求，找到对应的资源 (例如 `index.html` 文件)。
8.  **构建 HTTP 响应消息:**

    ```
    HTTP/1.1 200 OK
    Server: nginx
    Content-Type: text/html; charset=UTF-8
    Content-Length: ...

    <!DOCTYPE html>
    <html>
    <head>
    ...
    </html>
    ```

9.  **发送 HTTP 响应:** 服务器通过 TCP 连接将响应消息发送给浏览器。
10. **浏览器解析响应:** 浏览器解析 HTTP 响应，读取响应体中的 HTML 内容，并渲染网页。
11. **TCP 连接保持 (持久连接):** HTTP/1.1 默认使用 **持久连接 (Keep-Alive)**，TCP 连接在一段时间内保持打开状态，用于后续请求，提高效率。

**HTTP 协议定义了丰富的请求方法 (例如 GET, POST, PUT, DELETE) 和状态码 (例如 200 OK, 404 Not Found, 500 Internal Server Error)，用于实现各种 Web 应用功能。**

### HTTP 的无状态性与 TCP 持久连接

值得注意的是，虽然 HTTP 协议是 **无状态的**，但这 **并不意味着** 每次 HTTP 请求都要重新建立 TCP 连接。 **HTTP/1.1 引入了持久连接 (Keep-Alive) 机制，允许在同一个 TCP 连接上发送多个 HTTP 请求和响应。**

**即使多个 HTTP 请求可能复用同一个 TCP 连接，在 HTTP 协议层面，每个 HTTP 请求仍然是 _无状态的_。** 服务器不会记住前一个请求的信息，每个请求都是独立的。

**持久连接只是在 _传输层_ 提高了效率，它 _没有改变_ HTTP 协议在 _应用层_ 的无状态特性。**

## 从明文到加密：HTTPS 的诞生

HTTP 协议虽然功能强大，但存在一个致命的缺陷： **数据传输是明文的，不安全！** 这意味着，如果 HTTP 通信被中间人截获，数据内容 (包括敏感信息，例如用户名、密码、信用卡号等) 将会完全暴露，存在被窃听、篡改的风险。

为了解决 HTTP 的安全问题，就诞生了 **HTTPS (HTTP Secure)**，即 **安全的超文本传输协议**。 **HTTPS 实际上就是 HTTP over SSL/TLS，即在 HTTP 协议的基础上，使用 SSL/TLS 协议对通信进行加密。**

**HTTPS 相比 HTTP 主要完善了以下安全特性：**

- **机密性 (Confidentiality):** 通过 SSL/TLS 协议加密，防止数据被窃听。
- **完整性 (Integrity):** 通过 SHA-256 哈希，数据校验，防止数据被篡改。
- **身份验证 (Authentication):** 通过 SSL/TLS 证书，验证服务器身份，防止中间人攻击。

**当我们访问 `https://` 开头的网站时，浏览器和服务器之间就会建立 HTTPS 连接，数据传输将被加密保护，极大地提升了网络通信的安全性。**

## 深入探索 SSL-TLS

### SSL/TLS 工作原理

当客户端（如浏览器）与服务器建立 TCP 连接后，**SSL/TLS 协议**会启动握手过程。在这个过程中，双方协商加密算法、交换密钥，并通过证书验证服务器的身份（有时也会验证客户端）。握手完成后，数据传输通过加密通道进行，防止被窃听或篡改。

### 加密位置

TCP 负责可靠的数据传输，但本身不提供加密。SSL/TLS 在 TCP 之上添加了安全层，因此 HTTPS（HTTP over SSL/TLS）能够实现安全通信。

### 加密过程详解

服务器公钥用于密钥交换，而非直接加密所有数据。加密使用的是每个连接独一无二的**会话密钥**（Session Key）。过程如下：

1.  客户端向服务端请求证书以及公钥，经过第三方 CA 机构进行域名和 IP 的匹配，验证证书有效性。
2.  匹配通过后进行密钥协商。
3.  客户端生成一个随机的会话密钥（Session Key）。
4.  客户端使用服务器的公钥加密这个会话密钥，并发送给服务器（这里用到了**非对称加密**）。
5.  服务器使用自己的私钥解密收到的信息，得到会话密钥（只有服务器拥有私钥才能解密）。
6.  现在客户端和服务器都拥有了同一个会话密钥，但这个密钥只有它们双方知道，所有数据都通过这个密钥进行**对称加密**。

> **备注：** 传统 HTTP 协议没有加密机制，它直接在 TCP 层之上以明文形式传输数据。传统 HTTP 本身无法解决这些问题。为了弥补这些缺陷，人们引入了 SSL/TLS，推出了 HTTPS 协议。通过网络抓包可以捕获 HTTP 的明文数据，如 ARP 欺骗（ARP Spoofing）或网关上进行端口镜像等。

## 证书类型

证书主要分为两种类型：

### 自签证书（内网，测试）

自签证书是由自己生成的证书，不依赖任何第三方机构（证书颁发机构，简称 CA）。这种证书通常用于测试或内网环境，但因为它不被浏览器或操作系统信任，在公网上使用时，浏览器会显示"不安全"或"证书不受信任"的警告。

### 第三方证书（Let's Encrypt、DigiCert）

第三方证书是由受信任的证书颁发机构签发的，基于信任链机制。**Let's Encrypt** 是一个免费、自动化、开放的证书颁发机构（CA）。它签发的证书是受信任的第三方证书，而不是自签证书。Let's Encrypt 的根证书已被主流浏览器和操作系统信任，因此它签发的证书在公网上是受信任的，不会显示警告。

## 证书配置实践

### 1\. 使用 Let's Encrypt 进行证书签发

```bash
# 下载 Let's Encrypt 客户端
# CentOS
sudo yum install epel-release
sudo yum install certbot

# 生成证书  默认情况下，证书存放在 /etc/letsencrypt/live/
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

### 2\. 使用 OpenSSL 生成自签证书

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
> **SSL** (Secure Sockets Layer) 是早期的加密协议。它由 Netscape 公司开发，最初用于保护 Web 浏览器的通信安全。SSL 协议有多个版本，例如 SSL 2.0 和 SSL 3.0。
>
> **TLS** (Transport Layer Security) 是 SSL 的继任者和标准化版本。它由互联网工程任务组 (IETF) 在 SSL 3.0 的基础上进行标准化和改进，并更名为 TLS。TLS 协议也有多个版本，例如 TLS 1.0, TLS 1.1, TLS 1.2, TLS 1.3（目前最新最安全的版本是 TLS 1.3）。

## 多项目共用证书

证书是与域名相绑定的，也就意味着只要是相同的域名或者子域名，一个证书可以服务于多个项目，可以同时开启 HTTPS 连接。NGINX 前端静态文件直接服务，后端 API 通过 `proxy_pass` 转发。

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    location / {
        root /var/www/html;  # 前端静态文件目录
        index index.html;
    }

    location /api/ {
        proxy_pass http://localhost:3000;  # 后端 API 服务器
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

server {
    listen 80;
    server_name example.com;
    # 重定向 HTTP 到 HTTPS
    return 301 https://$host$request_uri;
}
```

## 总结

SSL/TLS 是现代网络安全的基石，通过在 TCP 层之上添加加密层，有效保护了数据传输的安全性。无论是使用自签证书还是第三方证书，正确配置 SSL/TLS 都能显著提升网站的安全性，防止数据被窃听和篡改。
