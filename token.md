
token,一个查询权限的问题

那么,没有token之前是怎么样的呢

![[1.png]]

所有的请求都要查一次数据库验证身份,那用户数量少没什么问题,那要是10万个并发呢?数据库被 身份验证 请求淹没

这就是 token 要解决的第一个问题,把 身份验证 这个高频操作从数据库中剥离出来,让服务器自己判断

![[2.png]]

但这又有一个问题

服务器不查数据库,怎么就能判断这一段字符串是对的?

那就来到了 签名机制

首先,token的验证,是纯粹的数学运算,不依赖任何外部系统

接着我们来看,他到底是怎么做到的

先看 JWT 到底长什么样

![[3.png]]

这三段分别代表了

1. Header(头部):声明 我用什么算法签名的
```js
{
 "alg": "HS256",
 "typ": "JWT"
}
```

2. Payload(载荷):塞进去的业务数据
```js
{
 "id": 1,
 "exp": 1699999999
}
```

3. Signature(签名):整个 JWT 的核心

那 签名 到底是怎么算的?

用公式写出来
```js
Signature = HMAC-SHA256(
    Base64(Header) + "." + Base64(Payload),
    SECRET_KEY
)
```

在看看 签名过程 和 验证过程都是怎么样的

- 签名过程:
![[4.png]]

- 验证过程:
![[5.png]]

代码:
```js
const auth = (req,res,next) =>{
    try{
        const token = req.headers.authorization.split(' ')[1]
        const user = jwt.verify(token,process.env.JWT_SECRET)
        req.user = user
        next()
    }catch(error){
        res.status(401).json({message:"token无效"})
    }
}
```

那么为什么篡改必定被发现?

- 输入:"id:1" + SECRET -> 输出:4f2b8c1a3e9d...
- 输入:"id:2" + SECRET -> 输出:a81e3f07bc42...
- 输入:"id:1" + 猜的key -> 输出: 7721dd90ef15...

修改任意部分,都会导致输出的 Signature 不同,直接走 catch