# Token 和 JWT：登录凭据是怎么工作的？

读了上篇文章我们知道：HTTP 本身不会记住登录状态，所以浏览器可以用 Cookie 自动携带 Session ID，服务端再根据 Session ID 查找登录信息。

那么问题来了：客户端不止浏览器，还有手机 App、命令行工具，或者多个后端服务。它们当然也可以手动维护和发送 Cookie，但 **Cookie + Session** 这套组合更依赖服务端状态和浏览器行为，在跨服务、非浏览器场景下不一定方便。

我们需要一种更通用的方式，让请求携带可验证的凭据，这就是 **Token** 要解决的问题。

![[编程学习/nodejs学习路径/token/1.png]]

## 先分清几个容易混淆的概念

1. **Token**：后续请求可以出示的身份凭据。
2. **Session ID**：一种不透明 Token。它本身没有业务含义，服务端需要查 Session 状态才能知道它对应谁。
3. **JWT**：Token 可以采用的一种具体数据格式。
4. **Cookie**：浏览器或客户端携带数据的一种方式。
5. **Bearer**：一种“持有即使用”的凭据使用方式。谁拿到了 Bearer Token，谁通常就能拿它访问资源。

所以，Token 和 JWT 不是两个平级的认证方案，而是“类别”和“一种具体实现”的关系。

## Token 的基本请求流程

```text
1. 用户提交账号密码
Client --> Auth Server

2. 认证通过，服务器签发访问凭据
Client <-- Access Token -- Auth Server

3. 后续访问资源时携带凭据
Client -- Authorization: Bearer <Token> --> Resource Server

4. 资源服务器验证凭据后，才执行操作
Resource Server -- 允许 / 拒绝 --> Client
```

Token 本身通常不是“身份”这个人，而是让服务端能够确认身份的一把钥匙：

- **Token 是随机 Session ID**：服务端必须查表或查缓存，才能知道它对应哪个用户。
- **Token 自带已签名的声明**：服务端可以先验证签名，再读取其中受信任的身份信息。

无论是哪一种，Token 被盗后都可能被冒用。因此 Token 的保管、传输和过期策略都很重要。

## JWT 到底长什么样？

接下来看看 JWT 的结构和验签过程。

![[编程学习/nodejs学习路径/token/2.png]]

JWT 通常由三段组成，中间用 `.` 分隔：

```text
header.payload.signature
```

![[编程学习/nodejs学习路径/token/3.png]]

这三段分别代表：

### 1. Header（头部）

声明签名算法和数据类型：

```js
{
  "alg": "HS256",
  "typ": "JWT"
}
```

但要注意，Header 是客户端携带的内容，服务端不能盲目信任其中的 `alg`。服务端应该在代码中明确限制允许使用的算法。

### 2. Payload（载荷）

Payload 里放的是 **Claims（声明）**，例如用户主体、过期时间和权限范围：

```js
{
  "sub": "hzh",
  "exp": 1699999999,
  "scope": "orders:read"
}
```

Payload 不是加密区域，拿到 JWT 的人通常都能解码查看，所以不要把密码、银行卡号等敏感信息放进去。

### 3. Signature（签名）

签名用于证明 JWT 没有被篡改，并且是持有对应密钥的一方签发的。

如果使用 `HS256`，可以简化表示为：

```js
Signature = HMAC-SHA256(
  Base64URL(Header) + "." + Base64URL(Payload),
  SECRET_KEY
)
```

这里的 `HS256` 只是 JWT 支持的多种算法之一。实际也可以使用 RSA、ECDSA 等非对称签名算法；JWT 标准使用的是 **Base64URL 编码**，不是普通 Base64 的简单替代说法。

## 签名和验证分别做了什么？

### 签名过程

![[编程学习/nodejs学习路径/token/4.png]]

认证服务使用自己的密钥，对 Header 和 Payload 计算签名，再把三段内容拼成 JWT 发给客户端。

### 验证过程

![[编程学习/nodejs学习路径/token/5.png]]

资源服务器收到 JWT 后，需要完成几件事：

1. 解析 JWT 的三段内容。
2. 使用服务端信任的密钥验证签名。
3. 限制允许的算法，避免接受不安全或未预期的算法。
4. 检查 `exp`、`iss`、`aud` 等声明。
5. 验证通过后，才根据 `sub` 和 `scope` 等信息执行授权判断。

一个简化的 Node.js 中间件示例：

```js
const auth = (req, res, next) => {
  try {
    const header = req.headers.authorization || ''
    const [scheme, token] = header.split(' ')

    if (scheme !== 'Bearer' || !token) {
      return res.status(401).json({ message: '缺少 Bearer Token' })
    }

    const user = jwt.verify(token, process.env.JWT_SECRET, {
      algorithms: ['HS256'],
      issuer: 'auth.example.com',
      audience: 'orders-api'
    })

    req.user = user
    next()
  } catch (error) {
    return res.status(401).json({ message: 'Token 无效或已过期' })
  }
}
```

这只是示例，真实项目还需要根据业务检查权限、密钥管理、错误处理和日志策略。

## JWT 能防篡改，但防不了盗用

JWT 的签名可以防止攻击者修改 Payload 后还通过验证，但它解决不了另一个问题：

> 假如一个有效 Token 被偷走了，攻击者可以直接拿它使用吗？

通常可以。因为 Bearer Token 的特点就是“谁持有，谁使用”。所以我们还需要给 Token 设置合理的过期时间，降低泄露后的影响范围。

## Access Token 与 Refresh Token

![[编程学习/nodejs学习路径/token/6.png]]

- **Access Token**：访问业务 API，通常有效期较短。
- **Refresh Token**：向认证服务申请新的 Access Token，通常有效期更长，因此需要更谨慎地保存。

一次典型的生命周期大致如下：

![[编程学习/nodejs学习路径/token/7.png]]

最关键的是：

- Resource Server 通常只信任 Access Token。
- Auth Server 才负责处理 Refresh Token。

Refresh Token 的长期续期能力更强，泄露后的影响也更大。如果任意业务 API 都接受它，就相当于把“续期权限”暴露给更多服务和更多攻击面。

## Refresh Token 轮换（Rotation）

可以用轮换机制，进一步降低长期凭据被复制后的风险：

1. 客户端使用 `R1` 刷新。
2. 认证服务签发 `A2 + R2`，并立即废弃 `R1`。
3. 如果之后又有人使用 `R1`，说明它可能被复制或窃取。
4. 服务端可以撤销整个登录会话链，要求重新登录。

但轮换也不是绝对完美的。

假如攻击者先用被偷的 `R1` 成功刷新，合法客户端随后因为网络重试也使用 `R1`，就可能触发复用检测。为了避免继续信任可能已经泄露的长期凭据，服务端可能会终止整个会话链，让用户重新登录。

这就是安全性和用户体验之间的取舍：**宁愿要求重新登录，也不要继续信任可能已经泄露的 Refresh Token。**

## 总结

可以把整件事记成这样：

> **Token 是后续请求携带的凭据；Session ID 是需要服务端查状态的不透明 Token；JWT 是一种带签名声明的 Token 格式；Cookie 和 Authorization Header 是携带凭据的方式；Bearer 表示持有凭据通常就能使用它。**

最终，服务端要做的事情始终是：验证凭据没有被伪造或篡改，确认它没有过期，再根据其中的身份和权限决定是否允许这次操作。
