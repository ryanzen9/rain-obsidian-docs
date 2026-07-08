---
title: "Cloudflare Tunnel 内网穿透完整配置指南 - 零公网IP实现域名直连"
source: "https://www.wangxubin.site/Blogs/posts/cloudflare-tunnel-blog/#%E5%85%AB%E9%85%8D%E7%BD%AE%E9%AA%8C%E8%AF%81%E4%B8%8E%E8%B0%83%E8%AF%95"
author:
  - "[[wangxb96]]"
published: 2025-10-12
created: 2026-07-08
description: "本文基于真实配置实践，详细介绍如何使用 Cloudflare Tunnel 实现零公网IP的内网穿透方案。通过自有域名配合 cloudflared 工具，可将内网服务（HTTP/SSH等）安全暴露到公网，无需路由器端口映射或固定IP。适用于家庭服务器、开发环境、远程访问等场景。本次更新基于多设备实际配置经验，新增快速配置指南和多用户管理方案。"
tags:
  - "clippings"
---
> 本文基于真实配置实践，详细介绍如何使用 Cloudflare Tunnel 实现零公网IP的内网穿透方案。通过自有域名配合 cloudflared 工具，可将内网服务（HTTP/SSH等）安全暴露到公网，无需路由器端口映射或固定IP。适用于家庭服务器、开发环境、远程访问等场景。本次更新基于多设备实际配置经验，新增快速配置指南和多用户管理方案。

## 更新日志

- **2025-10-12**: 新增第十四部分“快速配置指南”，包含单设备5步快速配置和多设备最佳实践；完善SSH配置详解，包括用户端软件安装和每项参数含义；添加服务器端多用户管理方案，包括账户管理、权限隔离和安全加固。
- **2025-10-11**: 初始发布，涵盖完整配置流程和实战案例。

## 摘要

传统内网穿透方案往往需要公网IP或第三方服务，存在稳定性、安全性、成本等问题。Cloudflare Tunnel 作为官方免费服务，通过加密隧道连接内网与 Cloudflare 边缘网络，配合自有域名即可实现稳定的内网穿透。本文记录了从零开始的完整配置过程，涵盖 cloudflared 安装、隧道创建、DNS配置、服务部署等关键步骤，并提供 HTTP 与 SSH 双协议实战案例。

## 一、技术背景

### 1.1 传统内网穿透痛点

- **公网IP依赖** ：需要运营商分配公网IP或购买云服务器
- **路由器配置** ：端口映射配置复杂，安全风险高
- **第三方服务** ：依赖 frp、ngrok 等工具，存在稳定性问题
- **成本考量** ：商业服务收费，免费服务限制多

### 1.2 Cloudflare Tunnel 优势

- **零成本** ：Cloudflare 提供免费 Tunnel 服务
- **高安全** ：端到端加密，无需开放防火墙端口
- **高可用** ：依托 Cloudflare 全球边缘网络
- **易管理** ：Web 面板 + 命令行双重管理

## 二、架构原理

### 2.1 整体架构图

```
graph TB
    subgraph "用户端"
        U1[浏览器用户]
        U2[SSH客户端]
        U3[移动设备]
    end
    
    subgraph "Cloudflare 全球网络"
        CF1[边缘节点 - 美国]
        CF2[边缘节点 - 欧洲]
        CF3[边缘节点 - 亚洲]
        DNS[Cloudflare DNS]
    end
    
    subgraph "家庭/企业内网"
        subgraph "服务器"
            CD[cloudflared 守护进程]
            WEB[Web服务 :8080]
            SSH[SSH服务 :22]
            DB[数据库 :5432]
        end
        RT[路由器/防火墙]
    end
    
    U1 --> CF2
    U2 --> CF1
    U3 --> CF3
    
    CF1 -.-> |加密隧道| CD
    CF2 -.-> |加密隧道| CD
    CF3 -.-> |加密隧道| CD
    
    CD --> WEB
    CD --> SSH
    CD --> DB
    
    DNS --> CF1
    DNS --> CF2
    DNS --> CF3
    
    RT -.-> |仅出站连接| CF1
```
```
#mermaid-1782527783113{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783113 .error-icon{fill:#a44141;}#mermaid-1782527783113 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783113 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783113 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783113 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783113 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783113 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783113 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783113 .marker.cross{stroke:lightgrey;}#mermaid-1782527783113 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783113 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783113 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783113 .cluster-label span,#mermaid-1782527783113 p{color:#F9FFFE;}#mermaid-1782527783113 .label text,#mermaid-1782527783113 span,#mermaid-1782527783113 p{fill:#ccc;color:#ccc;}#mermaid-1782527783113 .node rect,#mermaid-1782527783113 .node circle,#mermaid-1782527783113 .node ellipse,#mermaid-1782527783113 .node polygon,#mermaid-1782527783113 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783113 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783113 .node .label{text-align:center;}#mermaid-1782527783113 .node.clickable{cursor:pointer;}#mermaid-1782527783113 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783113 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783113 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783113 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783113 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783113 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783113 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783113 .cluster text{fill:#F9FFFE;}#mermaid-1782527783113 .cluster span,#mermaid-1782527783113 p{color:#F9FFFE;}#mermaid-1782527783113 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783113 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783113 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}家庭/企业内网Cloudflare 全球网络用户端服务器加密隧道加密隧道加密隧道仅出站连接路由器/防火墙cloudflared 守护进程Web服务 :8080SSH服务 :22数据库 :5432边缘节点 - 美国边缘节点 - 欧洲边缘节点 - 亚洲Cloudflare DNS浏览器用户SSH客户端移动设备
```

### 2.2 数据流向图

```
sequenceDiagram
    participant User as 用户浏览器
    participant CF as Cloudflare边缘
    participant Tunnel as cloudflared隧道
    participant Service as 内网服务
    
    Note over User,Service: 1. 初始连接建立
    Tunnel->>CF: 建立持久WebSocket连接
    CF->>Tunnel: 连接确认
    
    Note over User,Service: 2. 用户请求处理
    User->>CF: HTTPS请求 app.example.com
    CF->>CF: DNS解析到隧道
    CF->>Tunnel: 转发请求到隧道
    Tunnel->>Service: HTTP请求 localhost:8080
    Service->>Tunnel: HTTP响应
    Tunnel->>CF: 返回响应
    CF->>User: HTTPS响应
```
```
内网服务cloudflared隧道Cloudflare边缘用户浏览器内网服务cloudflared隧道Cloudflare边缘用户浏览器#mermaid-1782527783167{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783167 .error-icon{fill:#a44141;}#mermaid-1782527783167 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783167 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783167 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783167 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783167 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783167 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783167 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783167 .marker.cross{stroke:lightgrey;}#mermaid-1782527783167 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783167 .actor{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783167 text.actor>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783167 .actor-line{stroke:lightgrey;}#mermaid-1782527783167 .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:lightgrey;}#mermaid-1782527783167 .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:lightgrey;}#mermaid-1782527783167 #arrowhead path{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783167 .sequenceNumber{fill:black;}#mermaid-1782527783167 #sequencenumber{fill:lightgrey;}#mermaid-1782527783167 #crosshead path{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783167 .messageText{fill:lightgrey;stroke:none;}#mermaid-1782527783167 .labelBox{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783167 .labelText,#mermaid-1782527783167 .labelText>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783167 .loopText,#mermaid-1782527783167 .loopText>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783167 .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:#81B1DB;fill:#81B1DB;}#mermaid-1782527783167 .note{stroke:hsl(180, 0%, 18.3529411765%);fill:hsl(180, 1.5873015873%, 28.3529411765%);}#mermaid-1782527783167 .noteText,#mermaid-1782527783167 .noteText>tspan{fill:rgb(183.8476190475, 181.5523809523, 181.5523809523);stroke:none;}#mermaid-1782527783167 .activation0{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783167 .activation1{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783167 .activation2{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783167 .actorPopupMenu{position:absolute;}#mermaid-1782527783167 .actorPopupMenuPanel{position:absolute;fill:#1f2020;box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#mermaid-1782527783167 .actor-man line{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783167 .actor-man circle,#mermaid-1782527783167 line{stroke:#81B1DB;fill:#1f2020;stroke-width:2px;}#mermaid-1782527783167 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}1. 初始连接建立2. 用户请求处理建立持久WebSocket连接连接确认HTTPS请求 app.example.comDNS解析到隧道转发请求到隧道HTTP请求 localhost:8080HTTP响应返回响应HTTPS响应
```

### 2.3 协议支持矩阵

```
graph LR
    subgraph "支持的协议类型"
        HTTP[HTTP/HTTPS<br/>Web服务]
        SSH_P[SSH<br/>远程终端]
        TCP[TCP<br/>数据库/自定义]
        WS[WebSocket<br/>实时通信]
    end
    
    subgraph "应用场景"
        WEB_APP[Web应用<br/>博客/管理后台]
        REMOTE[远程开发<br/>SSH/VS Code]
        DATABASE[数据库访问<br/>PostgreSQL/MySQL]
        GAMING[游戏服务器<br/>Minecraft等]
    end
    
    HTTP --> WEB_APP
    SSH_P --> REMOTE
    TCP --> DATABASE
    TCP --> GAMING
    WS --> WEB_APP
```
```
#mermaid-1782527783182{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783182 .error-icon{fill:#a44141;}#mermaid-1782527783182 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783182 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783182 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783182 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783182 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783182 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783182 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783182 .marker.cross{stroke:lightgrey;}#mermaid-1782527783182 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783182 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783182 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783182 .cluster-label span,#mermaid-1782527783182 p{color:#F9FFFE;}#mermaid-1782527783182 .label text,#mermaid-1782527783182 span,#mermaid-1782527783182 p{fill:#ccc;color:#ccc;}#mermaid-1782527783182 .node rect,#mermaid-1782527783182 .node circle,#mermaid-1782527783182 .node ellipse,#mermaid-1782527783182 .node polygon,#mermaid-1782527783182 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783182 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783182 .node .label{text-align:center;}#mermaid-1782527783182 .node.clickable{cursor:pointer;}#mermaid-1782527783182 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783182 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783182 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783182 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783182 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783182 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783182 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783182 .cluster text{fill:#F9FFFE;}#mermaid-1782527783182 .cluster span,#mermaid-1782527783182 p{color:#F9FFFE;}#mermaid-1782527783182 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783182 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783182 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}应用场景支持的协议类型Web应用
博客/管理后台远程开发
SSH/VS Code数据库访问
PostgreSQL/MySQL游戏服务器
Minecraft等HTTP/HTTPS
Web服务SSH
远程终端TCP
数据库/自定义WebSocket
实时通信
```

### 2.4 工作流程详解

1. **隧道建立** ：cloudflared 客户端与 Cloudflare 边缘建立持久连接
2. **DNS 绑定** ：子域名通过 CNAME 记录指向隧道
3. **流量代理** ：用户请求经 Cloudflare 转发至内网服务
4. **协议支持** ：支持 HTTP/HTTPS、SSH、TCP 等多种协议

## 三、环境准备

### 3.1 前置条件

- ✅ **Cloudflare 账号** ：免费注册即可
- ✅ **自有域名** ：已接入 Cloudflare DNS 托管
- ✅ **Linux 服务器** ：Ubuntu 24.04 LTS（本文环境）
- ✅ **出站网络** ：能够访问 Cloudflare 服务

### 3.2 系统信息确认

```bash
# 检查系统版本
cat /etc/os-release

# 检查CPU架构
uname -m

# 检查网络连通性
ping -c 4 1.1.1.1
```

## 四、cloudflared 安装

### 4.1 Ubuntu/Debian 官方源安装

```bash
# 更新软件源并安装依赖
sudo apt-get update
sudo apt-get install -y wget gnupg lsb-release

# 添加 Cloudflare 官方密钥和源
wget -q https://packages.cloudflare.com/cloudflare-main.gpg -O- | \
  sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] \
  https://packages.cloudflare.com/ $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/cloudflare.list

# 安装 cloudflared
sudo apt-get update
sudo apt-get install -y cloudflared

# 验证安装
cloudflared --version
```

### 4.2 通用二进制安装（备选方案）

```bash
# 下载最新版本（适用于网络受限环境）
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 \
  -o cloudflared

# 设置可执行权限并移动到系统目录
chmod +x cloudflared
sudo mv cloudflared /usr/local/bin/

# 验证安装
cloudflared --version
```

## 五、隧道配置流程

### 5.0 配置流程总览

```
flowchart TD
    Start([开始配置]) --> Install[安装 cloudflared]
    Install --> Auth[Cloudflare 认证]
    Auth --> Create[创建隧道]
    Create --> DNS[配置 DNS 路由]
    DNS --> Config[编写配置文件]
    Config --> Service[部署系统服务]
    Service --> Test[测试验证]
    Test --> Monitor[监控运维]
    
    Auth --> AuthSub[浏览器授权选择域名]
    Create --> CreateSub[生成隧道ID和凭证文件]
    DNS --> DNSSub[子域名CNAME指向隧道]
    Config --> ConfigSub[ingress规则配置]
    Service --> ServiceSub[systemd守护进程]
    Test --> TestSub[HTTP/SSH连通性验证]
    
    style Start fill:#e1f5fe
    style Monitor fill:#c8e6c9
    style Test fill:#fff3e0
```
```
#mermaid-1782527783205{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783205 .error-icon{fill:#a44141;}#mermaid-1782527783205 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783205 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783205 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783205 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783205 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783205 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783205 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783205 .marker.cross{stroke:lightgrey;}#mermaid-1782527783205 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783205 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783205 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783205 .cluster-label span,#mermaid-1782527783205 p{color:#F9FFFE;}#mermaid-1782527783205 .label text,#mermaid-1782527783205 span,#mermaid-1782527783205 p{fill:#ccc;color:#ccc;}#mermaid-1782527783205 .node rect,#mermaid-1782527783205 .node circle,#mermaid-1782527783205 .node ellipse,#mermaid-1782527783205 .node polygon,#mermaid-1782527783205 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783205 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783205 .node .label{text-align:center;}#mermaid-1782527783205 .node.clickable{cursor:pointer;}#mermaid-1782527783205 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783205 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783205 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783205 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783205 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783205 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783205 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783205 .cluster text{fill:#F9FFFE;}#mermaid-1782527783205 .cluster span,#mermaid-1782527783205 p{color:#F9FFFE;}#mermaid-1782527783205 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783205 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783205 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}开始配置安装 cloudflaredCloudflare 认证创建隧道配置 DNS 路由编写配置文件部署系统服务测试验证监控运维浏览器授权选择域名生成隧道ID和凭证文件子域名CNAME指向隧道ingress规则配置systemd守护进程HTTP/SSH连通性验证
```

### 5.1 Cloudflare 认证

```bash
# 登录 Cloudflare 账号
cloudflared tunnel login
```

> 💡 **提示** ：执行后会输出一个 URL，在浏览器中打开并选择要管理的域名进行授权。

### 5.2 创建隧道

```bash
# 创建命名隧道
cloudflared tunnel create my-tunnel

# 查看隧道信息
ls -la ~/.cloudflared/
```

**输出示例** ：

```
Created tunnel my-tunnel with id 167e64e3-17b5-4bf8-bd7b-f7691f658416
Tunnel credentials written to /home/user/.cloudflared/167e64e3-17b5-4bf8-bd7b-f7691f658416.json
```

### 5.3 DNS 路由配置

```bash
# 将子域名指向隧道（以 app.example.com 为例）
cloudflared tunnel route dns my-tunnel app.example.com

# SSH 访问子域名
cloudflared tunnel route dns my-tunnel ssh.example.com
```

## 六、服务配置

### 6.1 创建配置文件

```bash
# 创建系统级配置目录
sudo mkdir -p /etc/cloudflared

# 复制凭证文件到系统目录
sudo cp ~/.cloudflared/167e64e3-17b5-4bf8-bd7b-f7691f658416.json /etc/cloudflared/
sudo chown root:root /etc/cloudflared/167e64e3-17b5-4bf8-bd7b-f7691f658416.json
sudo chmod 400 /etc/cloudflared/167e64e3-17b5-4bf8-bd7b-f7691f658416.json
```

### 6.2 配置 ingress 规则

#### ingress 路由机制图

```
flowchart LR
    subgraph "外部请求"
        R1[app.example.com]
        R2[ssh.example.com]
        R3[api.example.com]
        R4[其他域名]
    end
    
    subgraph "cloudflared ingress 处理"
        I1[规则1: hostname匹配]
        I2[规则2: hostname匹配]
        I3[规则3: hostname匹配]
        I4[兜底规则: 404]
    end
    
    subgraph "内网服务"
        S1[localhost:8080<br/>Web服务]
        S2[localhost:22<br/>SSH服务]
        S3[localhost:3000<br/>API服务]
        S4[HTTP 404<br/>错误页面]
    end
    
    R1 --> I1 --> S1
    R2 --> I2 --> S2
    R3 --> I3 --> S3
    R4 --> I4 --> S4
```
```
#mermaid-1782527783231{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783231 .error-icon{fill:#a44141;}#mermaid-1782527783231 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783231 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783231 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783231 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783231 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783231 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783231 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783231 .marker.cross{stroke:lightgrey;}#mermaid-1782527783231 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783231 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783231 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783231 .cluster-label span,#mermaid-1782527783231 p{color:#F9FFFE;}#mermaid-1782527783231 .label text,#mermaid-1782527783231 span,#mermaid-1782527783231 p{fill:#ccc;color:#ccc;}#mermaid-1782527783231 .node rect,#mermaid-1782527783231 .node circle,#mermaid-1782527783231 .node ellipse,#mermaid-1782527783231 .node polygon,#mermaid-1782527783231 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783231 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783231 .node .label{text-align:center;}#mermaid-1782527783231 .node.clickable{cursor:pointer;}#mermaid-1782527783231 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783231 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783231 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783231 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783231 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783231 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783231 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783231 .cluster text{fill:#F9FFFE;}#mermaid-1782527783231 .cluster span,#mermaid-1782527783231 p{color:#F9FFFE;}#mermaid-1782527783231 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783231 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783231 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}内网服务cloudflared ingress 处理外部请求localhost:8080
Web服务localhost:22
SSH服务localhost:3000
API服务HTTP 404
错误页面规则1: hostname匹配规则2: hostname匹配规则3: hostname匹配兜底规则: 404app.example.comssh.example.comapi.example.com其他域名
```

#### 配置文件创建

```bash
# 创建 /etc/cloudflared/config.yml
sudo tee /etc/cloudflared/config.yml >/dev/null <<'EOF'
tunnel: my-tunnel
credentials-file: /etc/cloudflared/167e64e3-17b5-4bf8-bd7b-f7691f658416.json

ingress:
  # HTTP 服务转发
  - hostname: app.example.com
    service: http://localhost:8080
  
  # SSH 服务转发
  - hostname: ssh.example.com
    service: ssh://localhost:22
  
  # 兜底规则（必需）
  - service: http_status:404
EOF
```

### 6.3 创建 systemd 服务

```bash
# 创建服务文件
sudo tee /etc/systemd/system/cloudflared.service >/dev/null <<'EOF'
[Unit]
Description=cloudflared tunnel
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/local/bin/cloudflared --config /etc/cloudflared/config.yml tunnel run my-tunnel
Restart=on-failure
TimeoutStartSec=0
User=root

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动服务
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflared

# 检查服务状态
sudo systemctl status cloudflared --no-pager
```

## 七、实战案例

### 7.1 HTTP 服务暴露

#### 启动测试服务

```bash
# 创建测试页面
echo "Hello from Cloudflare Tunnel on $(hostname)" > /tmp/index.html

# 启动 Python HTTP 服务器
python3 -m http.server 8080 --bind 127.0.0.1 --directory /tmp &

# 验证本地访问
curl http://localhost:8080
```

#### 外网访问验证

```bash
# 测试外网访问
curl -I https://app.example.com

# 预期输出
# HTTP/2 200 
# server: cloudflare
# content-type: text/html
```

### 7.2 SSH 服务配置

#### SSH 连接流程图

```
sequenceDiagram
    participant Client as SSH客户端
    participant CF as Cloudflare边缘
    participant Tunnel as cloudflared
    participant SSH as SSH服务器
    
    Note over Client,SSH: SSH through Cloudflare Tunnel
    
    Client->>Client: 读取 ~/.ssh/config
    Client->>Client: 启动 ProxyCommand
    Client->>CF: cloudflared access ssh连接
    CF->>Tunnel: 建立SSH隧道
    Tunnel->>SSH: 转发到 localhost:22
    SSH->>Tunnel: SSH握手响应
    Tunnel->>CF: 返回握手
    CF->>Client: 建立SSH会话
    
    loop SSH会话
        Client->>CF: SSH命令/数据
        CF->>Tunnel: 转发
        Tunnel->>SSH: 执行
        SSH->>Tunnel: 结果
        Tunnel->>CF: 返回
        CF->>Client: 显示结果
    end
```
```
SSH服务器cloudflaredCloudflare边缘SSH客户端SSH服务器cloudflaredCloudflare边缘SSH客户端#mermaid-1782527783256{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783256 .error-icon{fill:#a44141;}#mermaid-1782527783256 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783256 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783256 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783256 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783256 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783256 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783256 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783256 .marker.cross{stroke:lightgrey;}#mermaid-1782527783256 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783256 .actor{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783256 text.actor>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783256 .actor-line{stroke:lightgrey;}#mermaid-1782527783256 .messageLine0{stroke-width:1.5;stroke-dasharray:none;stroke:lightgrey;}#mermaid-1782527783256 .messageLine1{stroke-width:1.5;stroke-dasharray:2,2;stroke:lightgrey;}#mermaid-1782527783256 #arrowhead path{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783256 .sequenceNumber{fill:black;}#mermaid-1782527783256 #sequencenumber{fill:lightgrey;}#mermaid-1782527783256 #crosshead path{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783256 .messageText{fill:lightgrey;stroke:none;}#mermaid-1782527783256 .labelBox{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783256 .labelText,#mermaid-1782527783256 .labelText>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783256 .loopText,#mermaid-1782527783256 .loopText>tspan{fill:lightgrey;stroke:none;}#mermaid-1782527783256 .loopLine{stroke-width:2px;stroke-dasharray:2,2;stroke:#81B1DB;fill:#81B1DB;}#mermaid-1782527783256 .note{stroke:hsl(180, 0%, 18.3529411765%);fill:hsl(180, 1.5873015873%, 28.3529411765%);}#mermaid-1782527783256 .noteText,#mermaid-1782527783256 .noteText>tspan{fill:rgb(183.8476190475, 181.5523809523, 181.5523809523);stroke:none;}#mermaid-1782527783256 .activation0{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783256 .activation1{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783256 .activation2{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:#81B1DB;}#mermaid-1782527783256 .actorPopupMenu{position:absolute;}#mermaid-1782527783256 .actorPopupMenuPanel{position:absolute;fill:#1f2020;box-shadow:0px 8px 16px 0px rgba(0,0,0,0.2);filter:drop-shadow(3px 5px 2px rgb(0 0 0 / 0.4));}#mermaid-1782527783256 .actor-man line{stroke:#81B1DB;fill:#1f2020;}#mermaid-1782527783256 .actor-man circle,#mermaid-1782527783256 line{stroke:#81B1DB;fill:#1f2020;stroke-width:2px;}#mermaid-1782527783256 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}SSH through Cloudflare Tunnelloop[SSH会话]读取 ~/.ssh/config启动 ProxyCommandcloudflared access ssh连接建立SSH隧道转发到 localhost:22SSH握手响应返回握手建立SSH会话SSH命令/数据转发执行结果返回显示结果
```

#### 服务器端配置

```bash
# 确认 SSH 服务运行
sudo systemctl status ssh

# 检查端口监听
ss -lntp | grep :22
```

#### 客户端 SSH 配置架构

```
graph TB
    subgraph "本地客户端"
        SSH_CMD[ssh命令]
        SSH_CONFIG[~/.ssh/config]
        CLOUDFLARED_CLIENT[cloudflared客户端]
    end
    
    subgraph "Cloudflare网络"
        EDGE[边缘节点]
    end
    
    subgraph "远程服务器"
        CLOUDFLARED_SERVER[cloudflared守护进程]
        SSH_SERVER[SSH服务 :22]
    end
    
    SSH_CMD --> SSH_CONFIG
    SSH_CONFIG --> CLOUDFLARED_CLIENT
    CLOUDFLARED_CLIENT --> EDGE
    EDGE --> CLOUDFLARED_SERVER
    CLOUDFLARED_SERVER --> SSH_SERVER
```
```
#mermaid-1782527783268{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783268 .error-icon{fill:#a44141;}#mermaid-1782527783268 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783268 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783268 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783268 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783268 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783268 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783268 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783268 .marker.cross{stroke:lightgrey;}#mermaid-1782527783268 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783268 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783268 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783268 .cluster-label span,#mermaid-1782527783268 p{color:#F9FFFE;}#mermaid-1782527783268 .label text,#mermaid-1782527783268 span,#mermaid-1782527783268 p{fill:#ccc;color:#ccc;}#mermaid-1782527783268 .node rect,#mermaid-1782527783268 .node circle,#mermaid-1782527783268 .node ellipse,#mermaid-1782527783268 .node polygon,#mermaid-1782527783268 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783268 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783268 .node .label{text-align:center;}#mermaid-1782527783268 .node.clickable{cursor:pointer;}#mermaid-1782527783268 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783268 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783268 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783268 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783268 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783268 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783268 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783268 .cluster text{fill:#F9FFFE;}#mermaid-1782527783268 .cluster span,#mermaid-1782527783268 p{color:#F9FFFE;}#mermaid-1782527783268 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783268 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783268 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}远程服务器Cloudflare网络本地客户端cloudflared守护进程SSH服务 :22边缘节点ssh命令~/.ssh/configcloudflared客户端
```

#### 客户端 SSH 配置

##### 用户端软件准备

在本地客户端（Mac/Windows/Linux）上，需要安装以下软件：

1. **SSH客户端** ：
	- Mac：系统自带 OpenSSH
		- Windows：安装 OpenSSH（Windows 10+ 自带）或 PuTTY
		- Linux：系统自带 OpenSSH
2. **cloudflared客户端** ：
	```bash
	# Mac (使用 Homebrew)
	brew install cloudflare/cloudflare/cloudflared
	   
	# Windows (使用 Chocolatey)
	choco install cloudflared
	   
	# Linux
	wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
	sudo dpkg -i cloudflared-linux-amd64.deb
	```
3. **验证安装** ：
	```bash
	ssh -V  # 检查SSH版本
	cloudflared version  # 检查cloudflared版本
	```

##### SSH配置详解

在本地机器的 `~/.ssh/config` 中添加配置，每项含义如下：

```
# Cloudflare Tunnel SSH 配置示例
Host ssh.example.com
    # 目标主机名（通过Cloudflare Tunnel访问）
    HostName ssh.example.com
    # 远程服务器用户名
    User your-username
    # SSH端口（通常22）
    Port 22
    # 代理命令：通过cloudflared建立隧道连接
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
    # SSH私钥文件路径
    IdentityFile ~/.ssh/id_ed25519
    # 仅使用指定密钥文件，不使用默认密钥
    IdentitiesOnly yes
    # 每15秒发送心跳包，保持连接
    ServerAliveInterval 15
    # 心跳失败3次后断开连接
    ServerAliveCountMax 3
```
```
# 简化别名配置
Host my-server
    HostName ssh.example.com
    User your-username
    Port 22
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 15
    ServerAliveCountMax 3
```

**配置项详细说明** ：

- `Host` ：SSH连接的别名，可自定义
- `HostName` ：实际连接的域名/IP
- `User` ：远程服务器的用户名
- `Port` ：SSH服务端口（默认22）
- `ProxyCommand` ：使用cloudflared作为代理，建立安全隧道
- `IdentityFile` ：指定SSH私钥文件路径
- `IdentitiesOnly` ：仅使用指定的密钥文件
- `ServerAliveInterval` ：心跳间隔，防止连接超时
- `ServerAliveCountMax` ：最大心跳失败次数

##### SSH密钥对生成

```bash
# 生成ED25519密钥对（推荐）
ssh-keygen -t ed25519 -C "your-email@example.com"

# 或RSA（兼容性更好）
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# 查看公钥
cat ~/.ssh/id_ed25519.pub
```

#### SSH 连接测试

```bash
# 通过域名连接
ssh ssh.example.com

# 通过别名连接
ssh my-server

# 详细调试模式
ssh -vvv my-server
```

##### 服务器端多用户管理方案

对于多用户访问同一服务器的场景，需要合理管理用户权限和SSH访问：

###### 用户账户管理

```bash
# 创建新用户
sudo useradd -m -s /bin/bash newuser
sudo passwd newuser

# 设置用户组（可选）
sudo usermod -aG sudo newuser  # 添加到sudo组
sudo usermod -aG docker newuser  # 添加到docker组
```

###### SSH密钥管理

```bash
# 为每个用户创建独立的authorized_keys
sudo mkdir -p /home/newuser/.ssh
sudo touch /home/newuser/.ssh/authorized_keys
sudo chmod 700 /home/newuser/.ssh
sudo chmod 600 /home/newuser/.ssh/authorized_keys
sudo chown -R newuser:newuser /home/newuser/.ssh

# 添加用户公钥
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..." | sudo tee -a /home/newuser/.ssh/authorized_keys
```

###### 权限隔离配置

```bash
# 限制用户访问目录（使用chroot或权限设置）
# 编辑 /etc/ssh/sshd_config 添加：
# Match User newuser
#     ChrootDirectory /home/newuser
#     AllowTcpForwarding no
#     X11Forwarding no

# 重启SSH服务
sudo systemctl restart ssh
```

###### 多用户Tunnel配置

为不同用户配置独立的子域名和tunnel：

```yaml
# /etc/cloudflared/config.yml - 多用户配置
tunnel: main-tunnel
credentials-file: /etc/cloudflared/credentials.json

ingress:
  # 管理员SSH
  - hostname: admin.example.com
    service: ssh://localhost:22
  
  # 普通用户SSH
  - hostname: user1.example.com
    service: ssh://localhost:22
  
  # 开发用户SSH
  - hostname: dev.example.com
    service: ssh://localhost:22
  
  - service: http_status:404
```
```bash
# 为每个用户创建DNS路由
cloudflared tunnel route dns main-tunnel admin.example.com
cloudflared tunnel route dns main-tunnel user1.example.com
cloudflared tunnel route dns main-tunnel dev.example.com
```

###### 客户端多用户配置

```
# 多用户SSH配置
Host admin-server
    HostName admin.example.com
    User admin
    ProxyCommand cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519_admin

Host user1-server
    HostName user1.example.com
    User user1
    ProxyCommand cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519_user1

Host dev-server
    HostName dev.example.com
    User developer
    ProxyCommand cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519_dev
```

###### 安全加固建议

```bash
# 禁用密码认证，只允许公钥
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# 禁用root登录
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# 限制用户登录
echo "AllowUsers admin user1 developer" | sudo tee -a /etc/ssh/sshd_config

# 重启服务
sudo systemctl restart ssh
```

## 八、配置验证与调试

### 8.1 完整验证流程

```
flowchart TD
    Start([开始验证]) --> ServiceCheck{cloudflared服务运行?}
    ServiceCheck -- 否 --> StartService[启动服务]
    ServiceCheck -- 是 --> TunnelCheck{隧道连接正常?}
    
    StartService --> TunnelCheck
    TunnelCheck -- 否 --> CheckLogs[检查日志]
    TunnelCheck -- 是 --> DNSCheck{DNS解析正确?}
    
    CheckLogs --> FixConfig[修复配置]
    FixConfig --> ServiceCheck
    
    DNSCheck -- 否 --> FixDNS[修复DNS设置]
    DNSCheck -- 是 --> HTTPTest{HTTP服务可达?}
    
    FixDNS --> DNSCheck
    HTTPTest -- 否 --> CheckIngress[检查ingress规则]
    HTTPTest -- 是 --> SSHTest{SSH服务可达?}
    
    CheckIngress --> FixConfig
    SSHTest -- 否 --> CheckSSHConfig[检查SSH配置]
    SSHTest -- 是 --> Success([验证成功])
    
    CheckSSHConfig --> FixSSHConfig[修复SSH配置]
    FixSSHConfig --> SSHTest
    
    style Success fill:#c8e6c9
    style Start fill:#e1f5fe
```
```
#mermaid-1782527783286{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783286 .error-icon{fill:#a44141;}#mermaid-1782527783286 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783286 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783286 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783286 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783286 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783286 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783286 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783286 .marker.cross{stroke:lightgrey;}#mermaid-1782527783286 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783286 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783286 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783286 .cluster-label span,#mermaid-1782527783286 p{color:#F9FFFE;}#mermaid-1782527783286 .label text,#mermaid-1782527783286 span,#mermaid-1782527783286 p{fill:#ccc;color:#ccc;}#mermaid-1782527783286 .node rect,#mermaid-1782527783286 .node circle,#mermaid-1782527783286 .node ellipse,#mermaid-1782527783286 .node polygon,#mermaid-1782527783286 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783286 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783286 .node .label{text-align:center;}#mermaid-1782527783286 .node.clickable{cursor:pointer;}#mermaid-1782527783286 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783286 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783286 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783286 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783286 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783286 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783286 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783286 .cluster text{fill:#F9FFFE;}#mermaid-1782527783286 .cluster span,#mermaid-1782527783286 p{color:#F9FFFE;}#mermaid-1782527783286 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783286 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783286 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}否是否是否是否是否是开始验证cloudflared服务运行?启动服务隧道连接正常?检查日志DNS解析正确?修复配置修复DNS设置HTTP服务可达?检查ingress规则SSH服务可达?检查SSH配置验证成功修复SSH配置
```

### 8.2 服务状态检查

```bash
# 查看隧道连接状态
sudo journalctl -u cloudflared -f --no-pager

# 检查端口监听
netstat -tulpn | grep cloudflared

# 查看隧道指标
curl http://127.0.0.1:20241/metrics
```

### 8.3 问题诊断矩阵

```
graph TB
    subgraph "常见问题类型"
        P1[网络连接问题]
        P2[配置文件问题]  
        P3[DNS解析问题]
        P4[权限问题]
        P5[端口占用问题]
    end
    
    subgraph "诊断方法"
        D1[ping/curl测试]
        D2[配置文件验证]
        D3[dig/nslookup]
        D4[权限检查]
        D5[端口扫描]
    end
    
    subgraph "解决方案"
        S1[检查防火墙/代理]
        S2[修复配置语法]
        S3[重新配置DNS]
        S4[修复文件权限]
        S5[停止冲突进程]
    end
    
    P1 --> D1 --> S1
    P2 --> D2 --> S2
    P3 --> D3 --> S3
    P4 --> D4 --> S4
    P5 --> D5 --> S5
```
```
#mermaid-1782527783313{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783313 .error-icon{fill:#a44141;}#mermaid-1782527783313 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783313 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783313 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783313 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783313 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783313 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783313 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783313 .marker.cross{stroke:lightgrey;}#mermaid-1782527783313 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783313 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783313 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783313 .cluster-label span,#mermaid-1782527783313 p{color:#F9FFFE;}#mermaid-1782527783313 .label text,#mermaid-1782527783313 span,#mermaid-1782527783313 p{fill:#ccc;color:#ccc;}#mermaid-1782527783313 .node rect,#mermaid-1782527783313 .node circle,#mermaid-1782527783313 .node ellipse,#mermaid-1782527783313 .node polygon,#mermaid-1782527783313 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783313 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783313 .node .label{text-align:center;}#mermaid-1782527783313 .node.clickable{cursor:pointer;}#mermaid-1782527783313 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783313 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783313 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783313 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783313 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783313 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783313 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783313 .cluster text{fill:#F9FFFE;}#mermaid-1782527783313 .cluster span,#mermaid-1782527783313 p{color:#F9FFFE;}#mermaid-1782527783313 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783313 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783313 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}解决方案诊断方法常见问题类型检查防火墙/代理修复配置语法重新配置DNS修复文件权限停止冲突进程ping/curl测试配置文件验证dig/nslookup权限检查端口扫描网络连接问题配置文件问题DNS解析问题权限问题端口占用问题
```

### 8.4 常见问题排查

#### 隧道无法连接

```bash
# 检查网络连通性
curl -I https://api.cloudflare.com/client/v4/zones

# 检查配置文件语法
cloudflared tunnel ingress validate

# 测试隧道配置
cloudflared tunnel ingress rule https://app.example.com
```

#### DNS 解析问题

```bash
# 检查 DNS 记录
dig app.example.com
nslookup ssh.example.com

# 检查 Cloudflare DNS 设置
curl -X GET "https://api.cloudflare.com/client/v4/zones/{zone_id}/dns_records" \
  -H "Authorization: Bearer {api_token}"
```

## 九、高级配置

### 9.1 多服务架构设计

```
graph TB
    subgraph "外部访问"
        U1[Web用户]
        U2[API客户端]
        U3[数据库客户端]
        U4[SSH用户]
        U5[管理员]
    end
    
    subgraph "Cloudflare 边缘"
        CF[全球边缘节点]
    end
    
    subgraph "域名路由"
        D1[app.example.com]
        D2[api.example.com]
        D3[db.example.com]
        D4[ssh.example.com]
        D5[admin.example.com]
    end
    
    subgraph "内网服务集群"
        subgraph "Web层"
            WEB[React App :3000]
            ADMIN[Admin Panel :8081]
        end
        subgraph "API层"
            API[REST API :8080]
        end
        subgraph "数据层"
            DB[PostgreSQL :5432]
        end
        subgraph "系统层"
            SSH_S[SSH Server :22]
        end
        CD[cloudflared]
    end
    
    U1 --> D1 --> CF
    U2 --> D2 --> CF  
    U3 --> D3 --> CF
    U4 --> D4 --> CF
    U5 --> D5 --> CF
    
    CF --> CD
    CD --> WEB
    CD --> API
    CD --> DB
    CD --> SSH_S
    CD --> ADMIN
```
```
#mermaid-1782527783341{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783341 .error-icon{fill:#a44141;}#mermaid-1782527783341 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783341 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783341 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783341 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783341 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783341 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783341 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783341 .marker.cross{stroke:lightgrey;}#mermaid-1782527783341 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783341 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783341 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783341 .cluster-label span,#mermaid-1782527783341 p{color:#F9FFFE;}#mermaid-1782527783341 .label text,#mermaid-1782527783341 span,#mermaid-1782527783341 p{fill:#ccc;color:#ccc;}#mermaid-1782527783341 .node rect,#mermaid-1782527783341 .node circle,#mermaid-1782527783341 .node ellipse,#mermaid-1782527783341 .node polygon,#mermaid-1782527783341 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783341 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783341 .node .label{text-align:center;}#mermaid-1782527783341 .node.clickable{cursor:pointer;}#mermaid-1782527783341 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783341 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783341 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783341 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783341 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783341 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783341 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783341 .cluster text{fill:#F9FFFE;}#mermaid-1782527783341 .cluster span,#mermaid-1782527783341 p{color:#F9FFFE;}#mermaid-1782527783341 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783341 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783341 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}内网服务集群域名路由Cloudflare 边缘外部访问Web层API层数据层系统层cloudflaredSSH Server :22PostgreSQL :5432REST API :8080React App :3000Admin Panel :8081app.example.comapi.example.comdb.example.comssh.example.comadmin.example.com全球边缘节点Web用户API客户端数据库客户端SSH用户管理员
```

### 9.2 多服务配置文件

```yaml
# /etc/cloudflared/config.yml - 扩展版本
tunnel: my-tunnel
credentials-file: /etc/cloudflared/credentials.json

ingress:
  # Web 应用
  - hostname: app.example.com
    service: http://localhost:3000
  
  # API 服务
  - hostname: api.example.com
    service: http://localhost:8080
  
  # 数据库管理
  - hostname: admin.example.com
    service: http://localhost:8081
    originRequest:
      httpHostHeader: admin.example.com
  
  # SSH 访问
  - hostname: ssh.example.com
    service: ssh://localhost:22
  
  # TCP 代理（如数据库）
  - hostname: db.example.com
    service: tcp://localhost:5432
  
  - service: http_status:404
```

### 9.3 负载均衡配置

```
graph TB
    subgraph "用户请求"
        REQ[高并发请求]
    end
    
    subgraph "Cloudflare 负载均衡"
        LB[Load Balancer]
        CACHE[Edge Cache]
    end
    
    subgraph "多实例部署"
        T1[Tunnel 1<br/>Primary]
        T2[Tunnel 2<br/>Backup]
        T3[Tunnel 3<br/>Backup]
    end
    
    subgraph "后端服务池"
        S1[Service Instance 1]
        S2[Service Instance 2] 
        S3[Service Instance 3]
    end
    
    REQ --> CACHE
    CACHE --> LB
    LB --> T1
    LB --> T2
    LB --> T3
    
    T1 --> S1
    T2 --> S2
    T3 --> S3
    
    style T1 fill:#c8e6c9
    style T2 fill:#ffecb3
    style T3 fill:#ffecb3
```
```
#mermaid-1782527783380{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783380 .error-icon{fill:#a44141;}#mermaid-1782527783380 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783380 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783380 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783380 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783380 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783380 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783380 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783380 .marker.cross{stroke:lightgrey;}#mermaid-1782527783380 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783380 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783380 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783380 .cluster-label span,#mermaid-1782527783380 p{color:#F9FFFE;}#mermaid-1782527783380 .label text,#mermaid-1782527783380 span,#mermaid-1782527783380 p{fill:#ccc;color:#ccc;}#mermaid-1782527783380 .node rect,#mermaid-1782527783380 .node circle,#mermaid-1782527783380 .node ellipse,#mermaid-1782527783380 .node polygon,#mermaid-1782527783380 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783380 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783380 .node .label{text-align:center;}#mermaid-1782527783380 .node.clickable{cursor:pointer;}#mermaid-1782527783380 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783380 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783380 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783380 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783380 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783380 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783380 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783380 .cluster text{fill:#F9FFFE;}#mermaid-1782527783380 .cluster span,#mermaid-1782527783380 p{color:#F9FFFE;}#mermaid-1782527783380 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783380 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783380 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}后端服务池多实例部署Cloudflare 负载均衡用户请求Service Instance 1Service Instance 2Service Instance 3Tunnel 1
PrimaryTunnel 2
BackupTunnel 3
BackupLoad BalancerEdge Cache高并发请求
```

### 9.4 Cloudflare Access 安全集成

```
flowchart TB
    subgraph "身份提供商"
        IDP1[Google Workspace]
        IDP2[Microsoft Azure AD]
        IDP3[GitHub]
        IDP4[SAML IdP]
    end
    
    subgraph "Cloudflare Access"
        AUTH[Access 认证网关]
        POLICY[访问策略引擎]
        JWT[JWT 令牌验证]
    end
    
    subgraph "受保护资源"
        APP1[admin.example.com]
        APP2[api.example.com]
        SSH_APP[ssh.example.com]
    end
    
    subgraph "用户访问流程"
        USER[用户] --> AUTH
        AUTH --> IDP1
        IDP1 --> AUTH
        AUTH --> POLICY
        POLICY --> JWT
        JWT --> APP1
        JWT --> APP2
        JWT --> SSH_APP
    end
    
    style AUTH fill:#fff3e0
    style POLICY fill:#e8f5e8
    style JWT fill:#f3e5f5
```
```
#mermaid-1782527783404{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783404 .error-icon{fill:#a44141;}#mermaid-1782527783404 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783404 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783404 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783404 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783404 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783404 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783404 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783404 .marker.cross{stroke:lightgrey;}#mermaid-1782527783404 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783404 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783404 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783404 .cluster-label span,#mermaid-1782527783404 p{color:#F9FFFE;}#mermaid-1782527783404 .label text,#mermaid-1782527783404 span,#mermaid-1782527783404 p{fill:#ccc;color:#ccc;}#mermaid-1782527783404 .node rect,#mermaid-1782527783404 .node circle,#mermaid-1782527783404 .node ellipse,#mermaid-1782527783404 .node polygon,#mermaid-1782527783404 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783404 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783404 .node .label{text-align:center;}#mermaid-1782527783404 .node.clickable{cursor:pointer;}#mermaid-1782527783404 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783404 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783404 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783404 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783404 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783404 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783404 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783404 .cluster text{fill:#F9FFFE;}#mermaid-1782527783404 .cluster span,#mermaid-1782527783404 p{color:#F9FFFE;}#mermaid-1782527783404 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783404 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783404 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}用户访问流程受保护资源Cloudflare Access身份提供商用户admin.example.comapi.example.comssh.example.comAccess 认证网关访问策略引擎JWT 令牌验证Google WorkspaceMicrosoft Azure ADGitHubSAML IdP
```

#### Access 策略配置示例

##### 方法一：Web 面板配置

1. **登录 Cloudflare Zero Trust 控制台**
	```bash
	# 访问 https://one.dash.cloudflare.com/
	# 选择你的域名
	```
2. **创建 Access 应用程序**
	- 导航到 `Access > Applications`
		- 点击 `Add an application`
		- 选择 `Self-hosted` 或 `SaaS`
3. **配置应用程序详情**
	```yaml
	# 应用程序配置示例
	Name: SSH Access to Server
	Session Duration: 24 hours
	Application domain: ssh.example.com
	Application type: SSH
	```
4. **设置身份策略**
	```yaml
	# 策略配置
	Policy Name: Admin SSH Access
	Action: Allow
	Include Rules:
	  - Emails: admin@example.com
	  - Groups: admins@company.com
	Require Rules:
	  - IP ranges: 192.168.1.0/24 (可选)
	```
5. **配置身份提供商**
	- 导航到 `Settings > Authentication`
		- 添加提供商：Google、GitHub、Azure AD 等

##### 方法二：API 配置示例

```bash
# 1. 获取账户和区域信息
curl -X GET "https://api.cloudflare.com/client/v4/accounts" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json"

# 2. 创建 Access 应用程序
curl -X POST "https://api.cloudflare.com/client/v4/accounts/ACCOUNT_ID/access/apps" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "SSH Access",
    "domain": "ssh.example.com",
    "type": "ssh",
    "session_duration": "24h",
    "policies": ["POLICY_ID"]
  }'

# 3. 创建访问策略
curl -X POST "https://api.cloudflare.com/client/v4/accounts/ACCOUNT_ID/access/policies" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "Admin Access Policy",
    "include": [
      {"email": {"email": "admin@example.com"}},
      {"ip": {"ip": "192.168.1.0/24"}}
    ],
    "require": [],
    "exclude": []
  }'
```

##### 方法三：Terraform 配置示例

```
# main.tf
terraform {
  required_providers {
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

# 配置 Cloudflare 提供商
provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

# 创建 Access 应用程序
resource "cloudflare_access_application" "ssh_app" {
  zone_id          = var.zone_id
  name             = "SSH Access to Server"
  domain           = "ssh.example.com"
  type             = "ssh"
  session_duration = "24h"
}

# 创建访问策略
resource "cloudflare_access_policy" "admin_policy" {
  application_id = cloudflare_access_application.ssh_app.id
  zone_id        = var.zone_id
  name           = "Admin SSH Access"
  precedence     = 1
  decision       = "allow"

  include {
    email = ["admin@example.com"]
  }

  include {
    ip = ["192.168.1.0/24"]
  }
}

# 配置身份提供商 (GitHub 示例)
resource "cloudflare_access_identity_provider" "github" {
  account_id = var.account_id
  name       = "GitHub OAuth"
  type       = "github"

  config {
    client_id     = var.github_client_id
    client_secret = var.github_client_secret
  }
}

# variables.tf
variable "cloudflare_api_token" {
  description = "Cloudflare API Token"
  type        = string
  sensitive   = true
}

variable "zone_id" {
  description = "Cloudflare Zone ID"
  type        = string
}

variable "account_id" {
  description = "Cloudflare Account ID"
  type        = string
}

variable "github_client_id" {
  description = "GitHub OAuth App Client ID"
  type        = string
}

variable "github_client_secret" {
  description = "GitHub OAuth App Client Secret"
  type        = string
  sensitive   = true
}
```

##### 方法四：多租户配置示例

```yaml
# 多租户 Access 配置
# 为不同用户组配置不同的访问权限

# 管理员策略 - 完全访问
admin_policy:
  name: "Administrator Access"
  include:
    - email: ["admin@company.com"]
    - groups: ["admins"]
  allow:
    - ssh: "ssh.example.com"
    - http: "admin.example.com"

# 开发者策略 - 开发环境访问
dev_policy:
  name: "Developer Access"
  include:
    - email_domain: ["@company.com"]
    - groups: ["developers"]
  allow:
    - ssh: "dev.example.com"
    - http: "staging.example.com"
  deny:
    - http: "admin.example.com"

# 访客策略 - 只读访问
guest_policy:
  name: "Guest Access"
  include:
    - email: ["guest@partner.com"]
  allow:
    - http: "portal.example.com"
  require:
    - mfa: true
```

##### 验证配置

```bash
# 测试 Access 策略
curl -I https://ssh.example.com \
  -H "CF-Access-Client-Id: YOUR_CLIENT_ID" \
  -H "CF-Access-Client-Secret: YOUR_CLIENT_SECRET"

# 检查策略应用
cloudflared access login
cloudflared access ssh --hostname ssh.example.com
```

### 9.3 自动更新配置

```bash
# 创建更新脚本
sudo tee /etc/cron.daily/cloudflared-update >/dev/null <<'EOF'
#!/bin/bash
/usr/local/bin/cloudflared update
systemctl restart cloudflared
EOF

sudo chmod +x /etc/cron.daily/cloudflared-update
```

## 十、性能与安全

### 10.1 性能优化

```yaml
# 配置文件优化选项
tunnel: my-tunnel
credentials-file: /etc/cloudflared/credentials.json

# 连接池配置
connection-pool-size: 4
heartbeat-interval: 30s
max-heartbeats: 3

ingress:
  - hostname: app.example.com
    service: http://localhost:8080
    originRequest:
      # 启用压缩
      enableCompression: true
      # 连接超时
      connectTimeout: 30s
      # 保持连接
      keepAliveTimeout: 90s
```

### 10.2 安全配置

- **最小权限原则** ：仅暴露必要服务端口
- **访问控制** ：配合 Cloudflare Access 实现身份认证
- **证书管理** ：Cloudflare 自动处理 SSL/TLS 证书
- **日志监控** ：定期检查访问日志和异常连接

## 十一、故障排查流程

### 11.1 系统化排查流程

```
flowchart TD
    Start(🔍 开始排查) --> NetCheck{🌐 网络连通正常?}
    NetCheck -- ❌ 否 --> FixNetwork[🔧 检查出站网络]
    NetCheck -- ✅ 是 --> ServiceCheck{⚙️ cloudflared服务正常?}
    ServiceCheck -- ❌ 否 --> RestartService[🔄 重启服务]
    ServiceCheck -- ✅ 是 --> DNSCheck{🔗 DNS解析正确?}
    DNSCheck -- ❌ 否 --> FixDNS[📝 修复DNS配置]
    DNSCheck -- ✅ 是 --> ConfigCheck{📋 配置文件正确?}
    ConfigCheck -- ❌ 否 --> FixConfig[✏️ 修复配置]
    ConfigCheck -- ✅ 是 --> LogCheck[📊 查看详细日志]
    LogCheck --> ContactSupport[📞 联系技术支持]
    
    FixNetwork --> NetCheck
    RestartService --> ServiceCheck
    FixDNS --> DNSCheck
    FixConfig --> ConfigCheck
    
    style Start fill:#e1f5fe
    style ContactSupport fill:#ffebee
```
```
#mermaid-1782527783429{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783429 .error-icon{fill:#a44141;}#mermaid-1782527783429 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783429 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783429 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783429 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783429 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783429 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783429 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783429 .marker.cross{stroke:lightgrey;}#mermaid-1782527783429 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783429 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783429 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783429 .cluster-label span,#mermaid-1782527783429 p{color:#F9FFFE;}#mermaid-1782527783429 .label text,#mermaid-1782527783429 span,#mermaid-1782527783429 p{fill:#ccc;color:#ccc;}#mermaid-1782527783429 .node rect,#mermaid-1782527783429 .node circle,#mermaid-1782527783429 .node ellipse,#mermaid-1782527783429 .node polygon,#mermaid-1782527783429 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783429 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783429 .node .label{text-align:center;}#mermaid-1782527783429 .node.clickable{cursor:pointer;}#mermaid-1782527783429 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783429 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783429 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783429 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783429 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783429 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783429 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783429 .cluster text{fill:#F9FFFE;}#mermaid-1782527783429 .cluster span,#mermaid-1782527783429 p{color:#F9FFFE;}#mermaid-1782527783429 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783429 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783429 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}❌ 否✅ 是❌ 否✅ 是❌ 否✅ 是❌ 否✅ 是🔍 开始排查🌐 网络连通正常?🔧 检查出站网络⚙️ cloudflared服务正常?🔄 重启服务🔗 DNS解析正确?📝 修复DNS配置📋 配置文件正确?✏️ 修复配置📊 查看详细日志📞 联系技术支持
```

### 11.2 分层诊断架构

```
graph TB
    subgraph "Layer 1: 网络层"
        L1_1[网络连通性]
        L1_2[DNS 解析]
        L1_3[防火墙规则]
    end
    
    subgraph "Layer 2: 服务层"  
        L2_1[cloudflared 进程]
        L2_2[系统服务状态]
        L2_3[端口监听]
    end
    
    subgraph "Layer 3: 配置层"
        L3_1[配置文件语法]
        L3_2[ingress 规则]
        L3_3[凭证文件]
    end
    
    subgraph "Layer 4: 应用层"
        L4_1[后端服务状态]
        L4_2[应用程序日志]
        L4_3[性能指标]
    end
    
    L1_1 --> L2_1
    L1_2 --> L2_2  
    L1_3 --> L2_3
    L2_1 --> L3_1
    L2_2 --> L3_2
    L2_3 --> L3_3
    L3_1 --> L4_1
    L3_2 --> L4_2
    L3_3 --> L4_3
```
```
#mermaid-1782527783451{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783451 .error-icon{fill:#a44141;}#mermaid-1782527783451 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783451 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783451 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783451 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783451 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783451 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783451 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783451 .marker.cross{stroke:lightgrey;}#mermaid-1782527783451 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783451 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783451 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783451 .cluster-label span,#mermaid-1782527783451 p{color:#F9FFFE;}#mermaid-1782527783451 .label text,#mermaid-1782527783451 span,#mermaid-1782527783451 p{fill:#ccc;color:#ccc;}#mermaid-1782527783451 .node rect,#mermaid-1782527783451 .node circle,#mermaid-1782527783451 .node ellipse,#mermaid-1782527783451 .node polygon,#mermaid-1782527783451 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783451 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783451 .node .label{text-align:center;}#mermaid-1782527783451 .node.clickable{cursor:pointer;}#mermaid-1782527783451 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783451 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783451 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783451 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783451 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783451 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783451 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783451 .cluster text{fill:#F9FFFE;}#mermaid-1782527783451 .cluster span,#mermaid-1782527783451 p{color:#F9FFFE;}#mermaid-1782527783451 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783451 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783451 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}Layer 4: 应用层Layer 3: 配置层Layer 2: 服务层Layer 1: 网络层后端服务状态应用程序日志性能指标配置文件语法ingress 规则凭证文件cloudflared 进程系统服务状态端口监听网络连通性DNS 解析防火墙规则
```

### 常用调试命令速查

| 检查项目 | 命令 |
| --- | --- |
| 服务状态 | `sudo systemctl status cloudflared` |
| 实时日志 | `sudo journalctl -u cloudflared -f` |
| 配置验证 | `cloudflared tunnel ingress validate` |
| 规则测试 | `cloudflared tunnel ingress rule https://app.example.com` |
| 网络连通 | `curl -I https://api.cloudflare.com` |
| DNS 解析 | `dig app.example.com` |
| 端口检查 | `netstat -tulpn \\| grep cloudflared` |

## 十二、总结

Cloudflare Tunnel 为内网穿透提供了企业级的解决方案，具备以下显著优势：

### 12.1 技术优势

- **零配置公网** ：无需公网IP或路由器设置
- **高度安全** ：端到端加密，无需开放防火墙端口
- **协议丰富** ：支持 HTTP/SSH/TCP 等多种协议
- **全球加速** ：依托 Cloudflare CDN 网络

### 12.2 应用场景矩阵

```
graph TB
    subgraph "个人用户"
        P1[🏠 家庭实验室<br/>NAS/媒体中心]
        P2[💻 个人开发<br/>本地服务调试] 
        P3[🎮 游戏服务器<br/>朋友联机]
    end
    
    subgraph "企业用户"
        E1[🏢 内部系统<br/>OA/ERP访问]
        E2[🔧 运维监控<br/>Grafana/Prometheus]
        E3[📊 数据分析<br/>Jupyter/BI工具]
    end
    
    subgraph "开发团队"
        D1[🚀 CI/CD流水线<br/>Jenkins/GitLab]
        D2[🐛 测试环境<br/>Staging部署]
        D3[📝 文档系统<br/>Wiki/API文档]
    end
    
    subgraph "IoT场景"
        I1[🌡️ 传感器监控<br/>温度/湿度数据]
        I2[📹 视频监控<br/>摄像头直播]
        I3[🤖 智能家居<br/>Home Assistant]
    end
    
    style P1 fill:#e3f2fd
    style E1 fill:#f3e5f5
    style D1 fill:#e8f5e8
    style I1 fill:#fff8e1
```
```
#mermaid-1782527783484{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783484 .error-icon{fill:#a44141;}#mermaid-1782527783484 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783484 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783484 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783484 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783484 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783484 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783484 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783484 .marker.cross{stroke:lightgrey;}#mermaid-1782527783484 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783484 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783484 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783484 .cluster-label span,#mermaid-1782527783484 p{color:#F9FFFE;}#mermaid-1782527783484 .label text,#mermaid-1782527783484 span,#mermaid-1782527783484 p{fill:#ccc;color:#ccc;}#mermaid-1782527783484 .node rect,#mermaid-1782527783484 .node circle,#mermaid-1782527783484 .node ellipse,#mermaid-1782527783484 .node polygon,#mermaid-1782527783484 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783484 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783484 .node .label{text-align:center;}#mermaid-1782527783484 .node.clickable{cursor:pointer;}#mermaid-1782527783484 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783484 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783484 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783484 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783484 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783484 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783484 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783484 .cluster text{fill:#F9FFFE;}#mermaid-1782527783484 .cluster span,#mermaid-1782527783484 p{color:#F9FFFE;}#mermaid-1782527783484 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783484 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783484 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}IoT场景🌡️ 传感器监控
温度/湿度数据📹 视频监控
摄像头直播🤖 智能家居
Home Assistant开发团队🚀 CI/CD流水线
Jenkins/GitLab🐛 测试环境
Staging部署📝 文档系统
Wiki/API文档企业用户🏢 内部系统
OA/ERP访问🔧 运维监控
Grafana/Prometheus📊 数据分析
Jupyter/BI工具个人用户🏠 家庭实验室
NAS/媒体中心💻 个人开发
本地服务调试🎮 游戏服务器
朋友联机
```

### 12.3 部署模式对比

```
graph TB
    subgraph "传统方案"
        T1[🏠 家庭路由器<br/>端口映射]
        T2[☁️ 云服务器<br/>反向代理]
        T3[🔧 第三方工具<br/>frp/ngrok]
    end
    
    subgraph "Cloudflare Tunnel"
        CF1[🛡️ 零配置安全<br/>无需端口开放]
        CF2[🌍 全球加速<br/>CDN边缘网络]
        CF3[💰 零成本<br/>免费额度充足]
    end
    
    subgraph "对比维度"
        D1[安全性]
        D2[易用性] 
        D3[成本]
        D4[性能]
        D5[稳定性]
    end
    
    T1 --> D1
    T2 --> D2
    T3 --> D3
    CF1 --> D4
    CF2 --> D5
    CF3 --> D1
    
    style CF1 fill:#c8e6c9
    style CF2 fill:#c8e6c9
    style CF3 fill:#c8e6c9
```
```
#mermaid-1782527783505{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783505 .error-icon{fill:#a44141;}#mermaid-1782527783505 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783505 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783505 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783505 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783505 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783505 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783505 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783505 .marker.cross{stroke:lightgrey;}#mermaid-1782527783505 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783505 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783505 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783505 .cluster-label span,#mermaid-1782527783505 p{color:#F9FFFE;}#mermaid-1782527783505 .label text,#mermaid-1782527783505 span,#mermaid-1782527783505 p{fill:#ccc;color:#ccc;}#mermaid-1782527783505 .node rect,#mermaid-1782527783505 .node circle,#mermaid-1782527783505 .node ellipse,#mermaid-1782527783505 .node polygon,#mermaid-1782527783505 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783505 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783505 .node .label{text-align:center;}#mermaid-1782527783505 .node.clickable{cursor:pointer;}#mermaid-1782527783505 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783505 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783505 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783505 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783505 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783505 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783505 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783505 .cluster text{fill:#F9FFFE;}#mermaid-1782527783505 .cluster span,#mermaid-1782527783505 p{color:#F9FFFE;}#mermaid-1782527783505 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783505 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783505 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}对比维度Cloudflare Tunnel传统方案安全性易用性成本性能稳定性🛡️ 零配置安全
无需端口开放🌍 全球加速
CDN边缘网络💰 零成本
免费额度充足🏠 家庭路由器
端口映射☁️ 云服务器
反向代理🔧 第三方工具
frp/ngrok
```

### 12.3 最佳实践

1. **域名规划** ：使用子域名区分不同服务
2. **安全加固** ：配置 Cloudflare Access 身份认证
3. **监控告警** ：设置服务状态监控和日志告警
4. **备份恢复** ：定期备份配置文件和凭证
5. **版本管理** ：保持 cloudflared 客户端最新版本

通过本文的完整配置流程，您可以快速搭建稳定可靠的内网穿透服务，享受 Cloudflare 全球网络带来的高性能体验。

## 十三、参考资料

- [Cloudflare Tunnel 官方文档](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [cloudflared GitHub 仓库](https://github.com/cloudflare/cloudflared)
- [Cloudflare Zero Trust 平台](https://one.dash.cloudflare.com/)
- [SSH 配置最佳实践](https://wiki.archlinux.org/title/OpenSSH)

## 十四、快速配置指南

### 14.1 单设备快速配置

```bash
# 1. 安装 cloudflared
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# 2. 登录并创建隧道
cloudflared tunnel login
cloudflared tunnel create my-tunnel

# 3. 配置 DNS
cloudflared tunnel route dns my-tunnel app.example.com

# 4. 创建配置文件
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/*.json /etc/cloudflared/
sudo tee /etc/cloudflared/config.yml > /dev/null <<EOF
tunnel: my-tunnel
credentials-file: /etc/cloudflared/$(ls /etc/cloudflared/*.json | xargs basename)

ingress:
  - hostname: app.example.com
    service: http://localhost:8080
  - service: http_status:404
EOF

# 5. 启动服务
sudo cloudflared service install
sudo systemctl start cloudflared
```

### 14.2 多设备配置最佳实践

#### 架构设计

每个设备使用独立隧道，避免路由冲突：

```
graph TB
    subgraph "设备A"
        T1[tunnel-device-a]
        S1[SSH :22]
    end
    
    subgraph "设备B"  
        T2[tunnel-device-b]
        S2[SSH :22]
    end
    
    subgraph "Cloudflare"
        DNS1[a.example.com → tunnel-device-a]
        DNS2[b.example.com → tunnel-device-b]
    end
    
    DNS1 --> T1 --> S1
    DNS2 --> T2 --> S2
```
```
#mermaid-1782527783528{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;fill:#ccc;}#mermaid-1782527783528 .error-icon{fill:#a44141;}#mermaid-1782527783528 .error-text{fill:#ddd;stroke:#ddd;}#mermaid-1782527783528 .edge-thickness-normal{stroke-width:2px;}#mermaid-1782527783528 .edge-thickness-thick{stroke-width:3.5px;}#mermaid-1782527783528 .edge-pattern-solid{stroke-dasharray:0;}#mermaid-1782527783528 .edge-pattern-dashed{stroke-dasharray:3;}#mermaid-1782527783528 .edge-pattern-dotted{stroke-dasharray:2;}#mermaid-1782527783528 .marker{fill:lightgrey;stroke:lightgrey;}#mermaid-1782527783528 .marker.cross{stroke:lightgrey;}#mermaid-1782527783528 svg{font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:16px;}#mermaid-1782527783528 .label{font-family:"trebuchet ms",verdana,arial,sans-serif;color:#ccc;}#mermaid-1782527783528 .cluster-label text{fill:#F9FFFE;}#mermaid-1782527783528 .cluster-label span,#mermaid-1782527783528 p{color:#F9FFFE;}#mermaid-1782527783528 .label text,#mermaid-1782527783528 span,#mermaid-1782527783528 p{fill:#ccc;color:#ccc;}#mermaid-1782527783528 .node rect,#mermaid-1782527783528 .node circle,#mermaid-1782527783528 .node ellipse,#mermaid-1782527783528 .node polygon,#mermaid-1782527783528 .node path{fill:#1f2020;stroke:#81B1DB;stroke-width:1px;}#mermaid-1782527783528 .flowchart-label text{text-anchor:middle;}#mermaid-1782527783528 .node .label{text-align:center;}#mermaid-1782527783528 .node.clickable{cursor:pointer;}#mermaid-1782527783528 .arrowheadPath{fill:lightgrey;}#mermaid-1782527783528 .edgePath .path{stroke:lightgrey;stroke-width:2.0px;}#mermaid-1782527783528 .flowchart-link{stroke:lightgrey;fill:none;}#mermaid-1782527783528 .edgeLabel{background-color:hsl(0, 0%, 34.4117647059%);text-align:center;}#mermaid-1782527783528 .edgeLabel rect{opacity:0.5;background-color:hsl(0, 0%, 34.4117647059%);fill:hsl(0, 0%, 34.4117647059%);}#mermaid-1782527783528 .labelBkg{background-color:rgba(87.75, 87.75, 87.75, 0.5);}#mermaid-1782527783528 .cluster rect{fill:hsl(180, 1.5873015873%, 28.3529411765%);stroke:rgba(255, 255, 255, 0.25);stroke-width:1px;}#mermaid-1782527783528 .cluster text{fill:#F9FFFE;}#mermaid-1782527783528 .cluster span,#mermaid-1782527783528 p{color:#F9FFFE;}#mermaid-1782527783528 div.mermaidTooltip{position:absolute;text-align:center;max-width:200px;padding:2px;font-family:"trebuchet ms",verdana,arial,sans-serif;font-size:12px;background:hsl(20, 1.5873015873%, 12.3529411765%);border:1px solid rgba(255, 255, 255, 0.25);border-radius:2px;pointer-events:none;z-index:100;}#mermaid-1782527783528 .flowchartTitleText{text-anchor:middle;font-size:18px;fill:#ccc;}#mermaid-1782527783528 :root{--mermaid-font-family:"trebuchet ms",verdana,arial,sans-serif;}Cloudflare设备B设备Aa.example.com → tunnel-device-ab.example.com → tunnel-device-btunnel-device-bSSH :22tunnel-device-aSSH :22
```

#### 服务器端配置（每台设备重复）

```bash
# 为每台设备创建独立隧道
cloudflared tunnel create device-a-tunnel
cloudflared tunnel route dns device-a-tunnel a.example.com

# 配置文件模板
sudo tee /etc/cloudflared/config.yml > /dev/null <<EOF
tunnel: device-a-tunnel
credentials-file: /etc/cloudflared/$(ls ~/.cloudflared/device-a-tunnel.json)

ingress:
  - hostname: a.example.com
    service: ssh://localhost:22
  - service: http_status:404
EOF

# 启动服务
sudo cloudflared service install
sudo systemctl start cloudflared
```

#### 客户端配置（用户本地机器）

在 `~/.ssh/config` 中添加多设备条目：

```
# 多设备SSH配置模板
Host device-a
    HostName a.example.com
    User your-username
    ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 15
    ServerAliveCountMax 3

Host device-b
    HostName b.example.com
    User your-username
    ProxyCommand /opt/homebrew/bin/cloudflared access ssh --hostname %h
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 15
    ServerAliveCountMax 3
```

#### 公钥认证配置

```bash
# 生成密钥对（如果没有）
ssh-keygen -t ed25519 -C "your-email@example.com"

# 复制公钥到每台设备
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@device-a
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@device-b

# 或手动添加
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI..." >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 14.3 连接测试

```bash
# 测试连接
ssh device-a
ssh device-b

# 验证公钥认证（无需密码）
ssh -o PasswordAuthentication=no device-a 'echo "Connected successfully"'
```

### 14.4 故障排查快速版

```bash
# 检查服务
sudo systemctl status cloudflared

# 查看日志
journalctl -u cloudflared -n 10

# DNS测试
nslookup a.example.com

# 隧道状态
cloudflared tunnel list
```

---

\*本文基于 Ubuntu 24.04 LTS 环境实测编写，配置过程已验证可用。如有问题或建议，欢迎反馈！