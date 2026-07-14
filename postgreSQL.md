上篇文章我们知道, Redis 解决的核心矛盾: 速度 vs 持久化

那PostgreSQL 解决的是另一个更深处的矛盾,数据的 权限 vs 准确 的矛盾

想象一个场景:你运营一家电商。你有用户数据、订单数据、商品数据。如果随便用文件（JSON/CSV）存，你可以想怎么存怎么存,数据权限完全放开,这会带来几个问题

- 用户 A 下了一个订单，引用了商品 ID=999，但商品表里根本没有 999 -> 数据指向了虚空
- 两个客服同时修改同一个订单的状态，一个改成"已发货"，一个改成"已退款" -> 最终状态取决于谁的手速快
- 扣款成功了，但写订单时程序崩溃 -> 钱扣了，货没记录

这三个问题本质指向一件事:当数据之间存在关系是,权限自由 = 数据混乱

关系型数据库的诞生就是为了用数学上的约束消灭这种腐败。它的武器叫做关系模型

关系模型:可以想象成一个极度严格的会计系统
- 表 (Table) = 账本。每本账本只记一类事（用户账本、订单账本、商品账本）
- 行 (Row) = 账本里的一条记录（一个用户、一笔订单）
- 列 (Column) = 记录的固定格式（姓名、金额、日期——每条记录必须按这个格式填）
- 主键 (Primary Key) = 每条记录的唯一编号（身份证号——绝不重复）
- 外键 (Foreign Key) = 跨账本的引用凭证（订单里写"用户ID=42"——这个 42 必须在用户账本里真实存在，否则会计拒绝入账）

![[1.png]]

ACID:关系模型的四条铁律

1. A:Atomicity(原子性)

要么全做完,要么全不做

2. C:Consistency(一致性)

所有的数据必须满足预设的规则,外键必须指向真实存在的记录

3. I:Isolation(隔离性)

多人同时操作时，每个人看到的数据状态如同只有自己在操作

4. D:Durability(持久性)

事务一旦提交成功,数据就永久写入磁盘,断电也不丢失


数据建模

上面所说,用约束保证数据正确,那么约束作用在什么之上? 表结构

一个表设计得好不好,决定了:
- 数据能不能被约束保护（设计烂了，约束都加不上去）
- 查询快不快（设计烂了，一个简单需求要扫全表）
- 数据会不会冗余腐败（同一个事实存了两处，改一处忘一处）

所以,数据建模是关系型数据库的地基


数据类型

每一列必须声明一个数据类型,这是最细粒度的约束,决定这一列能装什么

常见类型有:
1. 整数:INTEGER(int4)、BIGINT (int8)
2. 小数:NUMERIC(10,2)
3. 文本: VARCHAR(255)、TEXT
4. 布尔:BOOLEAN
5. 时间:TIMESTAMP WITH TIME ZONE
6. 自增主键:SERIAL / BIGSERIAL

约束

类型管的是 单个格子 里能填什么,约束管的就是 整行、整列、甚至跨表的规则

约束有:
1. PRIMARY KEY (PK) :唯一 + 非空
2. NOT NULL:不允许控制
3. UNIQUE: 全列不重复
4. FOREIGN KEY(FK):值必须在另一张表中存在
5. CHECK:自定义条件
6. DEFAULT:未填时自动赋值

举个例子:
```sql
CREATE TABLE courses (
      id         SERIAL       PRIMARY KEY,          -- 自增唯一ID
      title      VARCHAR(200) NOT NULL,             -- 课程名，不能为空
      price      NUMERIC(10,2) NOT NULL CHECK (price > 0),  -- 金额精确，必须为正
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()     -- 默认当前时间
);
```


范式

消灭冗余,先画个图理解一下什么是冗余
![[2.png]]

像这种,在同一表中,同一部门下,出现重复的部门电话,这部分就是冗余,这可能一个问题

把 研发部 的电话改成 010-9999,你只更新第一个,忘了第二个,这会出现 同一部门出现了两个不同的电话,这就叫做 更新异常

那么范式是怎么做的:每个事实只存一次
![[3.png]]

这样,再去改电话,就只需要改一处,决定不会漏改

范式等级:
1. 1NF:每个格子只存一个值(不能赛数组/逗号分隔列表)
2. 2NF:非主键列必须完全依赖整个主键（不能只依赖主键的一部分）
3. 3NF:非主键列之间不能互相依赖（如"部门→部门电话"）

但也不是必须硬追求范式,有时我们会需要 反范式
```text
你的报表需求是"查询订单时同时显示用户名"。范式化设计需要 JOIN 两张表。如果这个查询每秒跑 10000 次，JOIN 的性能代价很高，你可能会选择在订单表里冗余存一份用户名——用一致性风险换查询速度
```

所以,默认范式化,只在性能瓶颈被 EXPLAIN 证实后才反范式,并且必须有机制保证冗余数据的同步

更好记一点的方式:会随时间变化的属性(邮箱,姓名)绝不冗余,而成交时刻的快照(成交价)可以冗余


CRUD,相信都不陌生,这里简单说一下
- Create (INSERT) = 新增
- Read (SELECT) = 查看
- Update (UPDATE) = 修改
- Delete (DELETE) = 删除

对应代码
```sql
INSERT INTO users (name, email) VALUES ('张三', 'zhang@example.com');

SELECT title, price FROM courses;

UPDATE courses SET price = 399.00 WHERE id = 1;

DELETE FROM users WHERE id = 5;
```

where:过滤
```sql
-- 精确匹配
SELECT * FROM users WHERE email = 'zhang@example.com';

-- 范围
SELECT * FROM courses WHERE price BETWEEN 100 AND 500;

-- 模糊匹配（% = 任意字符）
SELECT * FROM courses WHERE title LIKE '%后端%';

-- 多值
SELECT * FROM users WHERE id IN (1, 3, 7);

-- 组合条件
SELECT * FROM courses WHERE price > 200 AND title LIKE '%入门%';

-- 空值判断（注意：不是 = NULL，是 IS NULL）
SELECT * FROM users WHERE email IS NULL;
```

join:SQL最重要的能力,把表连接起来的关键

举例:你有用户账本和订单账本,现在 给我一张表，每一行显示：谁、买了什么课、花了多少钱, 那么你就需要把两个账本按照共同的线索(user_id) 拼接在一起
```sql
select u.name,c.title,e.paid_price,e.paid_at
form enrollments e
join users u on e.user_id = u.id
join courses c on e.course_id = c.id
```

代码中的 join,其实是 inner join的简写,它的意思是,返回两边都能匹配上的,还有 
- left join :左边全保留,右边匹配不上的填null
- right join: 右边全保留,左边匹配不上的填null

画个图就好理解了

![[4.png]]

实战用法:找出所有没买过课的用户
```sql
SELECT u.name, u.email
FROM users u
LEFT JOIN enrollments e ON u.id = e.user_id
WHERE e.id IS NULL;
```

为什么是 e.id is null ?因为 left join 保留了左表的所有行,匹配不上的右列表全部填null


聚合

```sql
-- COUNT: 总共多少条购买记录
SELECT COUNT(*) FROM enrollments;

-- SUM: 平台总收入
SELECT SUM(paid_price) FROM enrollments;

-- GROUP BY: 每门课的购买人数和总收入
SELECT c.title,
    COUNT(*) AS buyer_count,
    SUM(e.paid_price) AS total_revenue
FROM enrollments e
JOIN courses c ON e.course_id = c.id
GROUP BY c.title
ORDER BY total_revenue DESC;
```

group by,意思就是你只能对一张表下的 c.title 相同的做统计

having:对分组结果再过滤
```sql
-- 找出购买人数超过100的课程
SELECT c.title, COUNT(*) AS buyer_count
FROM enrollments e
JOIN courses c ON e.course_id = c.id
GROUP BY c.title
HAVING COUNT(*) > 100;
```

发现,where 和 having 很像,它两的区别是,前者在 分组前过滤单行, having 在分组后过滤整组

sql执行顺序

![[5.png]]


关卡 4：索引原理 | [费曼模式]

  从物理矛盾出发

  索引要解决的问题，本质上是一个物理矛盾——磁盘太慢。

  你在前端做过搜索功能对吧？假设你有一个 10000 条数据的数组，要找 id === 7749 的那条。最笨的办法：

  // 从头到尾扫一遍
  for (let i = 0; i < arr.length; i++) {
    if (arr[i].id === 7749) return arr[i];
  }

  在内存里，这个循环跑 10000 次也就微秒级别，你感觉不到。

  但 PostgreSQL 的数据在磁盘上。

  磁盘读一次数据的代价是内存的 10万倍 量级。你那个 for 循环如果变成"磁盘上逐行扫描 1000 万行"，就是灾难——这叫 全表扫描
  (Sequential Scan)。

  ---
  核心隐喻：图书馆

  ▎ 想象一个 100 万本书的图书馆，书架上的书按入馆时间排列（不是按主题、不是按作者）。
  ▎
  ▎ 现在有人问你："把《数据库系统概论》找出来。"
  ▎
  ▎ 没有索引 = 你只能从第一个书架开始，一本一本翻书名，直到找到为止。100 万本书，最坏情况全翻一遍。
  ▎
  ▎ 有索引 = 图书馆前台有一个 目录卡片柜。卡片按书名首字母排序。你先查卡片柜："数"→ 第 3 层抽屉 → "数据库系统概论 → 书架
  ▎ #4721，第 3 格"。三次查找，精准定位。

  映射关系：

  图书馆隐喻          PostgreSQL
  ─────────────────────────────────────
  100万本书           → 表中的 100万行数据
  书按入馆时间乱放     → 数据按插入顺序存在磁盘页(Heap)中
  从头翻到尾找书       → Sequential Scan (全表扫描)
  目录卡片柜          → Index (索引)
  卡片按书名排序       → 索引按列值排序
  卡片上写着书架位置   → 索引存储行的物理地址(ctid)
  查卡片柜3次定位      → B-tree 树高3层，3次磁盘IO

  ---
  关键推论

  索引的代价不是免费的——那个卡片柜本身也占空间，而且每次新书入馆（INSERT）、书改名（UPDATE）、书下架（DELETE），你都得同
  步更新卡片柜。

  所以索引的本质是一笔交易：

  ▎ 用额外的存储空间 + 写入时的维护成本，换取读取时的速度。

  这跟你在 Redis 里学过的东西有直接呼应——Redis 的 Sorted Set 用跳表维护有序性，也是"花额外内存维护结构，换 O(log N)
  查找"。PostgreSQL 的默认索引 B-tree 干的是同一件事，只不过是在磁盘上。

  ---
  现在进入苏格拉底模式，检验你的理解：

  一张表有 500 万行数据。你给 email 列加了索引，之后所有按 email 查找的 SELECT 都快了。但团队发现 INSERT 操作变慢了
  15%。用你刚学到的"交易"模型，解释为什么 INSERT 会变慢，并推演：如果这张表上有 5 个索引而不是 1 个，INSERT
  的代价会怎样变化？

  // [Agent Memory Anchor]
  {
    "current_node": "PostgreSQL 关卡4：索引原理 — 索引本质与代价模型",
    "mastered_nodes": ["Redis全7关", "PG关卡1:关系模型本质", "PG关卡2:数据建模", "PG关卡3:SQL核心"],
    "user_background_context": "前端转后端，用图书馆隐喻+JS数组扫描建立索引直觉",
    "current_mode": "2 (苏格拉底模式)",
    "clearance_status": "未达标 — 等待用户回答苏格拉底问题",
    "next_action": "评估用户对索引读写代价交易模型的理解"
  }

✻ Worked for 39s

※ recap: PostgreSQL学习进到关卡4索引原理，刚讲完索引的本质是用空间和写入成本换读取速度。你需要回答一个问题：为什么加索引
  后INSERT变慢，5个索引时代价怎样变化。