---
Title: Soybean Admin NestJS 学习(2)：使用Passport实现登录认证
Draft: false
tags:
  - ERP
  - 开源项目
  - TypeScript
  - NestJS
Author: Ruby Ceng
---

# Soybean Admin NestJS 学习(2)：使用 Passport 实现登录认证

## 1. 流程介绍

### 1.1 认证流程 (Authentication)

1. 用户登录 → AuthenticationController.login()
2. 凭据验证 → 密码 bcrypt 哈希比较
3. JWT 生成 → 生成访问令牌和刷新令牌
4. 角色缓存 → 用户角色存储到 Redis

### 1.2 功能特性

1. **双密钥机制**: 访问令牌和刷新令牌使用不同密钥
2. **令牌追踪**: 所有令牌都记录在数据库中
3. **角色缓存**: Redis 缓存提高权限检查性能
4. **审计日志**: 完整记录登录和操作日志
5. **多租户**: Domain 隔离确保租户间数据安全
6. **地理定位**: IP 地址解析和用户代理记录

## 2. 用户登录

### 2.1 登录接口

```typescript
// authentication.controller.ts
async login(
  @Body() dto: PasswordLoginDto,
  @Request() request: FastifyRequest,
): Promise<ApiRes<any>> {
  const { ip, port } = getClientIpAndPort(request);
  let region = 'Unknown';

  try {
    // 获取用户ip地址等相关信息
    const ip2regionResult = await Ip2regionService.getSearcher().search(ip);
    region = ip2regionResult.region || region;
  } catch (_) {}

  const token = await this.authenticationService.execPasswordLogin(
    new PasswordIdentifierDTO(
      dto.identifier,
      dto.password,
      ip,
      region,
      request.headers[USER_AGENT] ?? '',
      'TODO',
      'PC',
      port,
    ),
  );

  return ApiRes.success(token);
}
```

> **ip2region** ([https://github.com/lionsoul2014/ip2region](https://github.com/lionsoul2014/ip2region)) - 是一个离线 IP 地址定位库和 IP 定位数据管理框架，10 微秒级别的查询效率，提供了众多主流编程语言的 xdb 数据生成和查询客户端实现。

ip2region 的 xdb 数据生成和查询客户端实现示例：

```typescript
// a-simple-test.ts
import { Searcher } from "ip2region-ts";
import path from "path";

// --- 准备工作 ---
// 1. 确定你的 xdb 文件的存放路径
// path.join() 能帮你正确地拼接路径
// '__dirname' 指的是当前文件所在的目录
const dbPath = path.join(__dirname, "../resources/ip2region.xdb");

// 2. 使用文件路径创建一个 searcher 实例
// newWithFileOnly 表示 searcher 只会使用这个文件，不会缓存到内存
const searcher = Searcher.newWithFileOnly(dbPath);

// --- 开始查询 ---
async function findIpLocation(ip: string) {
  try {
    // 3. 调用 search 方法进行查询，这是一个异步操作
    const result = await searcher.search(ip);

    console.log(`查询 IP: ${ip}`);
    console.log("查询结果: ", result);

    if (result && result.region) {
      // 4. 打印出最关键的 region 信息
      console.log(`地理位置: ${result.region}`);
    } else {
      console.log("未找到该IP的信息");
    }
  } catch (error) {
    console.error("查询出错了:", error);
  }
}

// --- 测试一下 ---
findIpLocation("114.114.114.114"); // 一个公共DNS的IP
findIpLocation("180.169.11.234"); // 一个上海的IP

// 输出示例：
// 查询 IP: 114.114.114.114
// 查询结果:  { region: '中国|0|江苏省|南京市|信风网络', io_count: 1, took: 0.123 }
// 地理位置: 中国|0|江苏省|南京市|信风网络

// 查询 IP: 180.169.11.234
// 查询结果:  { region: '中国|华东|上海市|上海市|电信', io_count: 1, took: 0.088 }
// 地理位置: 中国|华东|上海市|上海市|电信
```

### 2.2 凭据验证

密码 BCrypt 哈希比较，密码使用加密存储：

```typescript
// domain/user.ts
async loginUser(
  password: string,
): Promise<{ success: boolean; message: string }> {
  if (this.status !== Status.ENABLED) {
    return {
      success: false,
      message: `User is ${this.status.toLowerCase()}.`,
    };
  }

  const isPasswordValid = await this.verifyPassword(password);
  if (!isPasswordValid) {
    return { success: false, message: 'Invalid credentials.' };
  }

  return { success: true, message: 'Login successful' };
}
```

## 3. JWT 生成

### 3.1 生成 JWT 令牌

```typescript
// authentication.service.ts
private async generateAccessToken(
  userId: string,
  username: string,
  domain: string,
): Promise<{ token: string; refreshToken: string }> {
  // 1. 构建JWT载荷
  const payload: IAuthentication = {
    uid: userId,        // 用户唯一标识
    username: username, // 用户名
    domain: domain,     // 租户域
  };

  // 2. 生成访问令牌 (短期有效)
  const accessToken = await this.jwtService.signAsync(payload);

  // 3. 生成刷新令牌 (长期有效)
  const refreshToken = await this.jwtService.signAsync(payload, {
    secret: this.securityConfig.refreshJwtSecret,      // 独立密钥
    expiresIn: this.securityConfig.refreshJwtExpiresIn, // 12小时
  });

  return { token: accessToken, refreshToken };
}
```

### 3.2 双令牌机制详解

**访问令牌 (Access Token):**

- 使用默认 JWT 密钥: `JWT_SECRET-soybean-admin-nest!@#123`
- 默认过期时间: 2 小时 (60 × 60 × 2)
- 用于 API 访问认证

**刷新令牌 (Refresh Token):**

- 使用独立密钥: `REFRESH_TOKEN_SECRET-soybean-admin-nest!@#123`
- 默认过期时间: 12 小时 (60 × 60 × 12)
- 用于刷新访问令牌

**接口说明:**

- **登录接口** (`/auth/login`):

  - 接收用户名和密码
  - 调用 `this.authenticationService.execPasswordLogin()` 执行核心登录逻辑
  - 成功后返回一个 token 对象

- **刷新令牌接口** (`/auth/refreshToken`):
  - 接收一个 refreshToken
  - 调用 `this.authenticationService.refreshToken()` 执行刷新逻辑
  - 成功后返回一个新的 token 对象

```typescript
// authentication.service.ts
async refreshToken(dto: RefreshTokenDTO) {
  const tokenDetails = await this.queryBus.execute<
    TokensByRefreshTokenQuery,
    TokensReadModel | null
  >(new TokensByRefreshTokenQuery(dto.refreshToken));

  if (!tokenDetails) {
    throw new NotFoundException('Refresh token not found.');
  }

  await this.jwtService.verifyAsync(tokenDetails.refreshToken, {
    secret: this.securityConfig.refreshJwtSecret,
  });

  const tokensAggregate = new TokensEntity(tokenDetails);
  await tokensAggregate.refreshTokenCheck();

  const tokens = await this.generateAccessToken(
    tokensAggregate.userId,
    tokenDetails.username,
    tokenDetails.domain,
  );

  tokensAggregate.apply(
    new TokenGeneratedEvent(
      tokens.token,
      tokens.refreshToken,
      tokensAggregate.userId,
      tokensAggregate.username,
      tokensAggregate.domain,
      dto.ip,
      dto.region,
      dto.userAgent,
      dto.requestId,
      dto.type,
      dto.port,
    ),
  );

  this.publisher.mergeObjectContext(tokensAggregate);
  tokensAggregate.commit();

  return tokens;
}
```

### 3.3 双密钥认证流程

现在，我们可以将整个双密钥认证流程串联起来：

**1. 配置:**

- 在 `libs/config/src/security.config.ts` 中为 Access Token 和 Refresh Token 分别定义了不同的 secret 和 expiresIn

**2. 用户登录:**

- 用户访问 `/auth/login` 接口
- AuthenticationController 调用 AuthenticationService
- AuthenticationService 验证用户凭据后，调用 generateAccessToken 方法
- 该方法使用不同的配置生成一个短生命周期的 Access Token 和一个长生命周期的 Refresh Token
- 两个 Token 被返回给客户端

**3. 访问受保护资源:**

- 客户端在请求头的 Authorization 字段中携带 Access Token 访问需要权限的接口
- 服务端的 JwtAuthGuard 会拦截请求，并使用 JwtStrategy 来验证 Access Token 的有效性（使用 jwtSecret）
- 验证成功则允许访问，失败则拒绝

**4. 令牌刷新:**

- 当 Access Token 过期时，客户端会收到 401 Unauthorized 错误
- 客户端检测到此错误后，调用 `/auth/refreshToken` 接口，并在请求体中附上 Refresh Token
- AuthenticationService 的 refreshToken 方法被触发
- 服务使用 refreshJwtSecret 验证 Refresh Token 的有效性
- 验证通过后，生成一套新的 Access Token 和 Refresh Token 返回给客户端
- 客户端更新本地存储的令牌，并使用新的 Access Token 重新发起之前失败的请求

### 3.4 令牌持久化

后续可以对令牌有效性进行操作，比如：

- 单点登录
- 强制下线等

```typescript
// token-generated.event.handler.ts
async handle(event: TokenGeneratedEvent) {
  const tokensProperties: TokensProperties = {
    accessToken: event.accessToken,
    refreshToken: event.refreshToken,
    status: TokenStatus.UNUSED,
    userId: event.userId,
    username: event.username,
    domain: event.domain,
    ip: event.ip,
    port: event.port,
    address: event.address,
    userAgent: event.userAgent,
    requestId: event.requestId,
    type: event.type,
    createdBy: event.userId,
  };

  const tokens = new TokensEntity(tokensProperties);
  await this.tokensWriteRepository.save(tokens);
}
```

### 3.5 双令牌机制的优势

#### A. 降低暴露风险 (Reduced Exposure)

Refresh Token 传输频率极低：它只在 Access Token 过期时（例如每 2 小时或 12 小时）才被传输一次，用于换取新令牌。相比于 Access Token 的每次请求都传输，它的暴露机会呈数量级下降。暴露得越少，被截获的概率就越低。

#### B. 更安全的存储策略 (Safer Storage)

由于两种令牌的用途不同，我们可以对它们采取不同的存储策略，尤其是 Refresh Token。

- **Access Token**: 通常存储在客户端的内存（如 JavaScript 变量）中。这很方便，但容易受到 XSS (跨站脚本) 攻击。
- **Refresh Token**: 可以（也应该）存储在 HttpOnly Cookie 中。
  - **什么是 `HttpOnly` Cookie？** 它是一种特殊的 Cookie，无法通过客户端的 JavaScript (document.cookie) 读取。只有浏览器在发送 HTTP 请求时会自动携带它。
  - **这为什么安全？** 这意味着即使你的网站存在 XSS 漏洞，攻击者也无法通过脚本窃取到 Refresh Token。这几乎完全免疫了 XSS 攻击对 Refresh Token 的威胁。

#### C. 泄露检测与主动防御 (Leakage Detection & Proactive Defense)

这是双令牌机制最强大的高级特性，被称为 "刷新令牌旋转 (Refresh Token Rotation)"。

这个项目虽然基础实现中未明确包含，但这是双令牌机制设计的标准最佳实践：

**一次性使用：** 当客户端使用 Refresh Token (我们称之为 RT_1) 来刷新时，服务器不仅返回一个新的 Access Token，还会返回一个全新的 Refresh Token (`RT_2`)。

**立即失效：** 服务器会立即将刚刚被使用过的 `RT_1` 标记为"已失效"。

现在，考虑攻击者窃取了 RT_1 的情况：

- **场景一：攻击者先于用户使用 `RT_1`**

  - 攻击者使用 RT_1 成功换取了新的令牌。服务器将 RT_1 标记为失效。
  - 当合法用户稍后也用 RT_1（因为他不知道已被盗用）来刷新时，服务器会发现这个 Refresh Token 已经被使用过或已失效。
  - **这是一个强烈的泄露信号！** 服务器可以立即采取行动：强制该用户的所有会话下线，并通知用户其账户可能已泄露。

- **场景二：用户先于攻击者使用 `RT_1`**
  - 用户正常刷新，获得了新的 RT_2，同时服务器将 RT_1 标记为失效。
  - 攻击者稍后使用他窃取到的 RT_1 尝试刷新，请求会立即失败，因为 RT_1 已经作废。

通过这种"旋转"机制，Refresh Token 的盗用行为可以被主动检测到，从而让系统有机会做出反应。而单令牌系统则完全无法做到这一点，一旦令牌泄露，在它过期之前你对此一无所知。

### 3.6 单令牌 vs 双令牌对比

| 特性       | 单令牌（长生命周期）           | 双令牌机制                                                          |
| ---------- | ------------------------------ | ------------------------------------------------------------------- |
| 生命周期   | 长（例如 7 天）                | Access Token: 短 (分钟级)<br>Refresh Token: 长 (天级)               |
| 传输频率   | 每次请求                       | Access Token: 每次请求<br>Refresh Token: 极少次 (仅刷新时)          |
| 泄露风险   | 高，因为频繁传输               | Access Token 风险高但危害小<br>Refresh Token 风险低                 |
| 存储安全   | 通常在 Local Storage，易受 XSS | Access Token: 内存<br>Refresh Token: 可存于 HttpOnly Cookie，防 XSS |
| 泄露后检测 | 无法检测，只能等令牌过期       | 可检测 (通过刷新令牌旋转机制)                                       |

## 4. 角色缓存

### 4.1 缓存用户角色信息

加快用户鉴权速度，提高性能：

```typescript
// authentication.service.ts
await RedisUtility.instance.del(key); // 删除旧的角色缓存
await RedisUtility.instance.sadd(key, ...result); // 添加角色集合
await RedisUtility.instance.expire(key, this.securityConfig.jwtExpiresIn); // 设置过期时间
```
