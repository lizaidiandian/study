# PostgreSQL：为什么数据不能只靠“存下来”

## PostgreSQL 想解决什么问题

上篇文章我们知道, Redis 解决的核心矛盾: 速度 vs 持久化

那 PostgreSQL 解决的是另一个更深处的矛盾: **数据可以自由地存,但数据之间必须保持准确和一致。**

想象一个场景:你运营一家电商。你有用户数据、订单数据、商品数据。如果随便用文件（JSON/CSV）存，你可以想怎么存怎么存，灵活是灵活，但数据之间的关系没人帮你约束，这会带来几个问题。

- 用户 A 下了一个订单，引用了商品 ID=999，但商品表里根本没有 999 -> 数据指向了虚空
- 两个客服同时修改同一个订单的状态，一个改成"已发货"，一个改成"已退款" -> 最终状态取决于谁的手速快
- 扣款成功了，但写订单时程序崩溃 -> 钱扣了，货没记录

这三个问题本质指向一件事: 当数据之间存在关系时，**随意存放 = 数据很容易混乱。**

关系型数据库的出现，就是为了给这些关系加上约束。它的核心武器叫做 **关系模型**。

关系模型:可以想象成一个极度严格的会计系统
- 表 (Table) = 账本。每本账本只记一类事（用户账本、订单账本、商品账本）
- 行 (Row) = 账本里的一条记录（一个用户、一笔订单）
- 列 (Column) = 记录的固定格式（姓名、金额、日期——每条记录必须按这个格式填）
- 主键 (Primary Key) = 每条记录的唯一编号（身份证号——绝不重复）
- 外键 (Foreign Key) = 跨账本的引用凭证（订单里写"用户ID=42"——这个 42 必须在用户账本里真实存在，否则会计拒绝入账）

![[编程学习/PostgreSQL/1.png]]

## ACID：事务为什么能让数据更可靠

ACID 是数据库事务最常见的四个保证

1. A:Atomicity(原子性)

要么全做完,要么全不做

2. C:Consistency(一致性)

事务完成后，数据仍然要满足预设的规则，比如外键不能指向不存在的记录

3. I:Isolation(隔离性)

多人同时操作时，每个人看到的数据状态如同只有自己在操作

4. D:Durability(持久性)

事务一旦提交成功，数据库会通过日志和持久化机制保证它在故障后仍能恢复


## 数据建模：约束到底作用在什么上

上面所说,用约束保证数据正确,那么约束作用在什么之上? 表结构

一个表设计得好不好,决定了:
- 数据能不能被约束保护（设计烂了，约束都加不上去）
- 查询快不快（设计烂了，一个简单需求要扫全表）
- 数据会不会冗余腐败（同一个事实存了两处，改一处忘一处）

所以,数据建模是关系型数据库的地基


### 数据类型

每一列必须声明一个数据类型,这是最细粒度的约束,决定这一列能装什么

常见类型有:
1. 整数:INTEGER(int4)、BIGINT (int8)
2. 小数:NUMERIC(10,2)
3. 文本: VARCHAR(255)、TEXT
4. 布尔:BOOLEAN
5. 时间:TIMESTAMP WITH TIME ZONE
6. 自增主键:SERIAL / BIGSERIAL

### 约束

类型管的是 单个格子 里能填什么,约束管的就是 整行、整列、甚至跨表的规则

约束有:
1. PRIMARY KEY (PK) :唯一 + 非空
2. NOT NULL:不允许为空
3. UNIQUE: 全列不重复
4. FOREIGN KEY(FK):值必须在另一张表中存在
5. CHECK:自定义条件
6. DEFAULT:未填时自动赋值

举个例子:
```sql
-- 自增唯一ID
-- 课程名，不能为空   
-- 金额精确，必须为正
-- 默认当前时间
CREATE TABLE courses (
      id         SERIAL       PRIMARY KEY,         
      title      VARCHAR(200) NOT NULL,            
      price      NUMERIC(10,2) NOT NULL CHECK (price > 0), 
      created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()    
);
```


### 范式

消灭冗余,先画个图理解一下什么是冗余
![[编程学习/PostgreSQL/2.png]]

像这种,在同一表中,同一部门下,出现重复的部门电话,这部分就是冗余,这可能一个问题

把 研发部 的电话改成 010-9999,你只更新第一个,忘了第二个,这会出现 同一部门出现了两个不同的电话,这就叫做 更新异常

那么范式是怎么做的:每个事实只存一次
![[编程学习/PostgreSQL/3.png]]

这样,再去改电话,就只需要改一处，就不容易漏改

范式等级:
1. 1NF:每个格子只存一个值(不能赛数组/逗号分隔列表)
2. 2NF:非主键列必须完全依赖整个主键（不能只依赖主键的一部分）
3. 3NF:非主键列之间不能互相依赖（如"部门→部门电话"）

但也不是必须硬追求范式,有时我们会需要 反范式
```text
你的报表需求是"查询订单时同时显示用户名"。范式化设计需要 JOIN 两张表。如果这个查询每秒跑 10000 次，JOIN 的性能代价很高，你可能会选择在订单表里冗余存一份用户名——用一致性风险换查询速度
```

所以,默认范式化,只在性能瓶颈被 EXPLAIN 证实后才反范式,并且必须有机制保证冗余数据的同步

更好记一点的方式:会随时间变化的属性(邮箱,姓名)一般不适合随意冗余,而成交时刻的快照(成交价)往往可以冗余。


## SQL：怎么操作关系数据

### CRUD

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

### WHERE：过滤
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

### JOIN：把表连接起来

join 是 SQL 最重要的能力之一,也是把表连接起来的关键

举例:你有用户账本和订单账本,现在 给我一张表，每一行显示：谁、买了什么课、花了多少钱, 那么你就需要把两个账本按照共同的线索(user_id) 拼接在一起
```sql
select u.name,c.title,e.paid_price,e.paid_at
from enrollments e
join users u on e.user_id = u.id
join courses c on e.course_id = c.id
```

代码中的 join,其实是 inner join的简写,它的意思是,返回两边都能匹配上的,还有 
- left join :左边全保留,右边匹配不上的填null
- right join: 右边全保留,左边匹配不上的填null

画个图就好理解了

![[编程学习/PostgreSQL/4.png]]

实战用法:找出所有没买过课的用户
```sql
SELECT u.name, u.email
FROM users u
LEFT JOIN enrollments e ON u.id = e.user_id
WHERE e.id IS NULL;
```

为什么是 e.id is null ?因为 left join 保留了左表的所有行,匹配不上的右列表全部填null


### 聚合

```sql
-- COUNT: 总共多少条购买记录
SELECT COUNT(*) FROM enrollments;

-- SUM: 平台总收入
SELECT SUM(paid_price) FROM enrollments;

-- GROUP BY: 每门课的购买人数和总收入
SELECT c.id, c.title,
    COUNT(*) AS buyer_count,
    SUM(e.paid_price) AS total_revenue
FROM enrollments e
JOIN courses c ON e.course_id = c.id
GROUP BY c.id, c.title
ORDER BY total_revenue DESC;
```

group by 的意思是: 先按某个维度把记录分组,再对每一组做统计。这里按课程 id 和课程名分组，就能得到每门课各自的购买人数和总收入。

having:对分组结果再过滤
```sql
-- 找出购买人数超过100的课程
SELECT c.id, c.title, COUNT(*) AS buyer_count
FROM enrollments e
JOIN courses c ON e.course_id = c.id
GROUP BY c.id, c.title
HAVING COUNT(*) > 100;
```

发现,where 和 having 很像,它两的区别是,前者在 分组前过滤单行, having 在分组后过滤整组

sql执行顺序

![[编程学习/PostgreSQL/5.png]]


## 索引：为什么查询不必每次扫全表

从物理矛盾出发

索引要解决的问题，本质上是一个物理矛盾——磁盘太慢。

你在前端做过搜索功能对吧？假设你有一个 10000 条数据的数组，要找 id === 7749 的那条。最笨的办法：

```js
// 从头到尾扫一遍
for (let i = 0; i < arr.length; i++) {
  if (arr[i].id === 7749) return arr[i];
}
```

在内存里，这个循环跑 10000 次也就微秒级别，你感觉不到。

但 PostgreSQL 的数据最终要落在磁盘上。虽然数据库和操作系统会尽量把常用页缓存到内存里，但缓存没命中时，读磁盘仍然比读内存慢很多。

如果一次查询需要在磁盘上逐行扫描 1000 万行，成本就会很高——这叫 **全表扫描(Sequential Scan)**。

 
关键推论

索引的代价不是免费的——那个卡片柜本身也占空间，而且每次新书入馆（INSERT）、书改名（UPDATE）、书下架（DELETE），你都得同
  步更新卡片柜。

所以索引的本质是一笔交易：

>用额外的存储空间 + 写入时的维护成本，换取读取时的速度。

这 Redis 里的东西直接呼应——Redis 的 Sorted Set 用跳表维护有序性，也是"花额外内存维护结构，换 O(log N)查找"。PostgreSQL 的默认索引 B-tree 干的是同一件事，只不过是在磁盘上。

B-tree

那他设计目的非常明确:用尽可能少的磁盘 IO 次数完成查找

那它是怎么做的?把 每个节点 塞尽可能多的数据

举个例子:在100万封信中,找到寄给 王大锤 的那封
1. 大桌子上放两百个标签,写着范围: A-C 去左边"、"D-F去中间" ... ->1 次磁盘 IO
2. 放更细的标签:你找到了 W区,桌子上有更细的标签"Wa-Wc"、"Wd-Wf",找到 "Wang" 所在 ->1 次磁盘 IO
3. 直接打开抽屉，里面就是那几封"王小明"的信，每封信上贴着原始地址(Heap 中的行位置) ->1 次磁盘 IO

共三次磁盘 IO,解决 100万行数据

![[编程学习/PostgreSQL/6.png]]

查找 user_id = 133 的过程
1. 读根节点 → 133 在 100~200 之间 → 走第2个指针
2. 读中间节点 → 133 在 101~178 之间 → 走第4个指针
3. 读叶子节点 → 找到 133 对应的行位置
4. 根据这个行位置回 Heap 取出完整行数据

为什么是 O(log N) 但极低常数

一个 8KB 页通常能塞下很多键值，所以 B-tree 的层数不会很深。
![[编程学习/PostgreSQL/7.png]]

对比全表扫描 100万行可能需要几千次 IO,但B-tree 只要 3次,这就是索引的威力

那为什么B-tree不像Redis一样,每个节点只存一个值,如果是这样的话,假设有800万个数据
```text
二叉树:  log₂(800万) ≈ 23 层 -> 23次 IO
B-tree: log₂₀₀(800万) = 3 层 -> 3次IO
```

由于操作的对象是磁盘,这里假设每次 IO 10ms
```text
二叉树: 10ms x 23 = 230ms
B-tree: 10ms x 3 = 30ms
```

差了近 8 倍,这看着 230ms 也还能接受,但这是单次查询,如果是在高并发下,每秒几千次查询,那这个差距会很大

而Redis 操作的对象是内存,内存的读取时间大约为 100ns
```text
100ns x 23 = 2.3μs 这根本感觉不出来
```

所以这就是 B-tree 把节点撑胖的原因,磁盘 IO 太贵,一次 IO 读取 8kb,那就把 8kb 塞满有用数据

索引类型

前面所讲的 B-tree 是默认索引,其核心能力为:有序性,因为 B-tree 内部是排好序的，所以等值、范围、排序它全能干

Hash 索引

用 JavaScript 的 Map 举例
```js
const map = new Map()
map.set('test@example.com', userData)
map.get('test@example.com')
```

理论来说 Hash 只用一次 IO 就行,但是他限制极大,只能用 = 来进行等值查找

GIN 倒排索引
```sql
-- 你想查"标签包含 'javascript' 的文章"
WHERE tags @> ARRAY['javascript']

-- 或者全文搜索"内容包含'数据库'的文章"
WHERE to_tsvector(content) @@ to_tsquery('数据库')
```

这里用B-tree 是做不到的,B-tree 索引一个值对应一行,但这里是一个值对应多行,一行也包含多个值,也叫多对多关系

GiST 通用搜索树

解决的问题:空间和集合数据
```sql
-- 查找距离某个坐标5公里内的所有餐厅
WHERE ST_DWithin(location, ST_MakePoint(116.4, 39.9), 5000)

-- 查找所有与某个时间段重叠的预订
WHERE reservation_period && '[2026-07-01, 2026-07-15]'
```
这里B-tree 只能做一维的大小比较,但地理坐标是二维的,时间段重叠是区域的

这里我觉得还是要对B-tree 和 GIN 在做个了解
![[编程学习/PostgreSQL/9.png]]

小总结一下:
![[编程学习/PostgreSQL/8.png]]


EXPLAIN

为什么需要 EXPLAIN ?

你给表加了索引,但 PostgreSQL 不一定会用它,为什么?

PostgreSQL 内部有一个查询规划器(Query Planner),每次你执行 sql,他自动评估,走索引快,还是全表扫描快?然后选代价最低的方案

EXPLAIN 就是让你看选择了哪个方案
```sql
EXPLAIN SELECT * FROM users WHERE email = 'wang@test.com'
```

PostgreSQL 不会真正执行这条查询，而是返回它的执行计划:

A:索引命中
```sql
Index Scan using idx_users_email on users
      Index Cond: (email = 'wang@test.com'::text)
      cost=0.42..8.44  rows=1  width=72
```

b:全表扫描
```sql
Seq Scan on users
      Filter: (email = 'wang@test.com'::text)
      cost=0.00..18523.00  rows=1  width=72
```

关键字段:
Index Scan:走了索引                   
Seq Scan:全表扫描（Sequential Scan）
Index Cond:索引过滤条件
Filter:在扫描结果上再过滤 
cost=0.42..8.44:预估代价（起始..总计）
rows=1:预估返回行数                  


EXPLAIN ANALYZE：不光看计划，还要看实际执行
```sql
Index Scan using idx_users_email on users
      Index Cond: (email = 'wang@test.com'::text)
      cost=0.42..8.44  rows=1  width=72
      actual time=0.028..0.030  rows=1  loops=1
      Planning Time: 0.085 ms
      Execution Time: 0.052 ms
```

只要记得五个常见执行节点就好

Seq Scan:全表扫描，逐行读
Index Scan:走B-tree，找到行后回Heap取数据
Index Only Scan:只读索引，不回Heap
Bitmap Index Scan:先用索引收集行位置，再批量回Heap
Filter:拿到数据后再逐行过滤

有些时候 PostgreSQL 会故意不用索引?

1. 返回行数太多——导航仪算出来全表扫描更快
2. 没给查询列建索引，或条件写法导致索引失效
3. 表太小——规划器判定不值得走索引

复合索引


复合索引 = 把多列塞进同一个索引里：
```text
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
```

  单列索引 (user_id)：
```
user_id=12345 → [行1, 行2, 行3, ..., 行50]   ← 50行全捞出来 再蛮力过滤status
```


复合索引 (user_id, status)：
```
(12345, 'cancelled')  → [行x, 行x, ...]
(12345, 'completed')  → [行x, 行x, ...]
(12345, 'pending')    → [行1, 行2, 行3]   ← 直接命中3行，零浪费
(12345, 'shipped')    → [行x, 行x, ...]
```

加了复合索引后的 EXPLAIN：

之前（单列索引）：
```
Index Scan using idx_orders_user_id
    Index Cond: (user_id = 12345)
    Filter: (status = 'pending')           ← 蛮力
    Rows Removed by Filter: 47             ← 47行白读
```


之后（复合索引）：
```text
Index Scan using idx_orders_user_status
    Index Cond: (user_id = 12345 AND status = 'pending')   ← 两个都走索引
    Rows Removed by Filter: 0                               ← 零浪费
```

复合索引的关键陷阱：列顺序

复合索引的列顺序不能随便写，有一个严格规则：

>最左前缀原则：查询必须从索引的最左列开始，才能命中。

  ```sql
 -- 索引：(user_id, status)
WHERE user_id = 12345 AND status = 'pending'   --  两列都命中
WHERE user_id = 12345                          --  只用最左列，能命中
WHERE status = 'pending'                       --  跳过了最左列，索引失效
  ```

复合索引 (user_id, status) 的内部排列:
```
(10001, 'cancelled')
(10001, 'completed')     ← user_id相同时，按status排序
(10001, 'pending')
(12345, 'cancelled')
(12345, 'completed')     ← 先按user_id大排序
(12345, 'pending')
(12345, 'shipped')
(20000, 'completed')

 知道user_id → 直接跳到12345区域 → 再找status 
 只知道status → 'pending'散落在每个user_id下面 → 没法跳 
```






![[Pasted image 20260716104050.png]]