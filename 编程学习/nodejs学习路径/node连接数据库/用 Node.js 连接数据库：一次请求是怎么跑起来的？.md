# 用 Node.js 连接数据库：一次请求是怎么跑起来的？

这篇主要是把一个简单后端的流程串起来：Node.js 接收请求，PostgreSQL 存数据，Redis 存缓存，最后把结果返回给前端。

## 先引入要用的东西

```js
//Web服务器框架  处理http请求,定义路由
const express = require('express')
//PostgreSQL驱动  连接数据库,执行SQL
const { Client } = require('pg')
//环境变量加载器  把.env文件里的密钥读进代码
const dotenv = require('dotenv')
//密码哈希工具  注册是加密密码 登录时比对密码
const bcrypt = require('bcrypt')
//Redis驱动  连接 Redis 缓存、读写缓存数据
const redis = require('redis')
```

关于 **Client**，这里用的是解构赋值，意思是只需要 pg 包中的 Client 这一个功能。

引入之后就是初始化配置。

```js
dotenv.config()
```

它会读取项目根目录的 .env 文件，把里面的键值对放进 **process.env** 对象中。

```text
.env文件
DATABASE_URL='xxxxxxxxxxxxxxxxxxxxxx'

REDIS_URL='xxxxxxxxxxxxxxxxxxxxxxxxx'
```

接着创建 PostgreSQL 和 Redis 客户端实例。这里只是创建，还没有真正连接。

```js
const client = new Client(process.env.DATABASE_URL)

const redisClient = redis.createClient({
	url:process.env.REDIS_URL
})
```

最后创建 Express 应用实例，后面的 **app.get**、**app.post** 等都会挂在它身上。

```js
const app = express()
```

```js
app.use(express.json())
```

这是注册一个中间件。如果前端发来的请求体是 JSON 格式，它会帮我们解析成 JS 对象，放进 **req.body**。

## 路由：别人访问什么，就做什么

```js
app.get('/',(req.res)=>{
	res.send('hello world')
})
```

- **app**：Express 实例。
- **get**：请求方法。
- **/**：请求路径。
- **req**：请求对象，也就是客户端传过来的值。
- **res**：响应对象，也就是返回给客户端的内容。

简单来说，有人用 GET 方法访问根路径时，就回复 hello world。

注册接口则是把用户信息写进数据库。

```js
app.post('/register', async (req, res) => {
    try {
        await client.query(`insert into users (name,password,email) values ($1,$2,$3)`, [req.body.name, await bcrypt.hash(req.body.password, 10), req.body.email])
        res.json({ message: '注册成功' })
    } catch (err) {
        console.log(err)
        res.status(400).json({ message: '注册失败' })
    }

})
```

- **async / await**：因为密码哈希和数据库插入都是异步操作，所以回调函数要使用 async。
- **try / catch**：保护可能出错的代码。例如邮箱重复，违反 UNIQUE 约束时，插入就会失败。
- **client.query**：执行 SQL。值不直接写进 SQL，而是用 **$1、$2、$3** 传入，这样可以防止 SQL 注入,因为这样数据库会把传进来的值当纯文本处理。
- **bcrypt.hash**：把密码变成哈希值，10 可以简单理解成哈希时的计算强度。
- **res.json**：给客户端返回 JSON 内容。
- **res.status(400)**：给客户端返回状态码。

登录时，不是把密码再 hash 一次后直接比较，而是用 **bcrypt.compare** 比对“用户输入的密码”和“数据库里存的哈希值”。

```js
app.post('/login', async (req, res) => {
    try {
        const data = await client.query(`select * from users where email = $1`, [req.body.email])
        if(data.rows.length === 0){
            res.status(400).json({message:'用户不存在'})
            return
        }
        if (await bcrypt.compare(req.body.password, data.rows[0].password)) {
            res.json({ message: '登录成功',data:data.rows[0].id })
        }else{
            res.status(400).json({message:'密码错误'})
            return
        }
    } catch (err) {
        console.log(err)
        res.status(400).json({ message: '登录失败' })
    }
})
```

## GET、POST 和 Redis 缓存

GET 请求和 POST 请求里，**req** 后面跟的内容通常不一样：

- **app.get**：常用 **req.query**，数据在 URL 参数上。
- **app.post**：常用 **req.body**，数据在 JSON 里。

什么是 URL 参数？

```text
https://xxx.com/details?teamid=0  ?后面的teamid=0 就是参数
```

下面这个接口是查用户资料。它会先查 Redis，有缓存就直接返回；没有缓存才去查 PostgreSQL，然后把结果写回 Redis。

```js
app.get('/profile',async (req,res) =>{
    const id = req.query.id
    const cached = await redisClient.get(id)
    try{
        if(cached){
            res.json({message:"用户信息已缓存",data:JSON.parse(cached)})
         }else{
            const userId = await client.query('select id,name,email,created_at from users where id = $1',[id])
            await redisClient.set(id,JSON.stringify(userId.rows[0]))
            res.json({message:"用户信息已缓存成功",data:userId.rows[0]})
         }
    }catch(error){
        res.status(400).json({message:error.message})
    }
})
```

- **redisClient.get**：读取缓存。没有就是 null，有就返回对应的值。
- **redisClient.set**：写入缓存，前者是值的名称，后者是具体的值。
- **JSON.stringify**：JS 对象变成 JSON 字符串。
- **JSON.parse**：JSON 字符串变回 JS 对象。

Redis 只能存字符串，所以对象写进去之前，要先用 JSON.stringify 转成字符串。

这里用户的 id 写在 URL 里，别人修改 id 后就可能查看别人的账号。更关键的还是要判断：当前用户有没有权限查看这份资料。

## 最后启动服务器

```js
app.listen(3000, async () => {
    await client.connect()
    await redisClient.connect()
})
```

- **client.connect**：连接 PostgreSQL。
- **redisClient.connect**：连接 Redis。

把连接写在这里，想表达的就是：服务启动时，也把 PostgreSQL 和 Redis 连上。这样整个流程就串起来了。

## 最后收一下

如果只用一句话理解这段代码，那就是：**Express 负责接请求，PostgreSQL 负责存用户数据，bcrypt 负责处理密码，Redis 负责缓存用户信息。**

整个流程就是：先引入工具和配置，再定义路由，路由里查数据库或缓存，最后启动服务器。
