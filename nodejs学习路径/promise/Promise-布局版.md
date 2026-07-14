# Promise

依旧先看 **Promise** 为何诞生。

在早期，单线程（如：js）一次只能做一件事，打个比方：

![[nodejs学习路径/promise/1.png]]

如图所知，当采取 **同步餐厅** 模式，服务员去厨房等待出餐的时候，其他人都点不了单，这流程就卡住了。

![[nodejs学习路径/promise/2.png]]

这个 **取餐号**，其实就很像 Promise 的前身。

在 Promise 出现之前，js 更常见的做法是：**回调函数**。

来看回调函数：一段作为参数传递给另一段代码的可执行代码。

```js
function a(time){
	setTimeout(() =>{
		console.log('启动')
	},time)
}

function b(data){
 return data ? data : 1000
}

a(b)
```

回调函数嵌套较少时，这没什么问题，但一旦嵌套过多，就会出现 **回调地狱**：

```js
function step1(callback) {
    setTimeout(() => {
        console.log("步骤1完成");
        callback("数据1");
    }, 1000);
}

function step2(data, callback) {
    setTimeout(() => {
        console.log("步骤2完成，收到：", data);
        callback("数据2");
    }, 1000);
}

function step3(data, callback) {
    setTimeout(() => {
        console.log("步骤3完成，收到：", data);
        callback("数据3");
    }, 1000);
}

//难以理解和维护
step1((result1) => {
    step2(result1, (result2) => {
        step3(result2, (result3) => {
            console.log("所有步骤完成：", result3);
        });
    });
});
```

那 Promise 就诞生了。

它的英文意思是 **“承诺”**，这个名字其实很贴切：你现在先拿到一个“承诺”，真正的结果会在以后给你。

Promise 有三种状态：

1. `Pending`（等待中）
2. `Fulfilled`（成功）
3. `Rejected`（失败）

这里面，真正最终结果其实只有两种：**成功** 或 **失败**。

而 `Pending` 更像是“还没出结果”的中间状态。

为什么说它重要？

因为 Promise 的状态变化是固定的：

> 状态只能从 `Pending -> Fulfilled`，或者 `Pending -> Rejected`，这是不可逆的。

也就是说，一旦成功了，就不会突然又变失败；一旦失败了，也不会突然再变成功。

Promise 保证了三件事：

1. 结果最终会落到成功或失败其中一个方向
2. 成功和失败是分开的，不会混在一起
3. 结果一旦确定，就不会反复变化

接着看 Promise 最基本的写法。

```js
const p = new Promise((resolve, reject) => {
    const ok = true

    if (ok) {
        resolve("成功了")
    } else {
        reject("失败了")
    }
})
```

这里可以这样理解：

- `new Promise(...)`：创建一个 Promise
- `resolve(...)`：把它推向成功
- `reject(...)`：把它推向失败

那结果怎么接？

就靠 `then` 和 `catch`。

```js
p
  .then((result) => {
      console.log("成功:", result)
  })
  .catch((error) => {
      console.log("失败:", error)
  })
```

如果成功，就走 `then`。

如果失败，就走 `catch`。

这和回调函数最大的区别就在这里：**成功怎么处理、失败怎么处理，被拆开了，而且可以顺着往下写。**

比如把刚才那种回调地狱，改成 Promise 风格之后，会更像这样：

```js
step1()
  .then((result1) => step2(result1))
  .then((result2) => step3(result2))
  .then((result3) => {
      console.log("所有步骤完成：", result3)
  })
  .catch((error) => {
      console.log("中间出错了：", error)
  })
```

这样写的好处很明显：

- 代码不再一层套一层
- 顺序更清楚
- 出错时可以统一交给 `catch`

最后还有一个常见的方法：`finally`

```js
p
  .then((result) => {
      console.log(result)
  })
  .catch((error) => {
      console.log(error)
  })
  .finally(() => {
      console.log("不管成功还是失败，最后都会执行")
  })
```

`finally` 的意思就是：**不管成功还是失败，这段代码最后都要执行。**

## 总结

最后，如果只用一句话去理解 Promise，那就是：**它不是立刻给你结果，而是先给你一个“以后会告诉你结果”的承诺。**

它之所以重要，不只是因为它能表示异步结果，更是因为它把 **成功**、**失败**、**后续处理** 这几件事拆得更清楚了。也正因为这样，Promise 才把 js 从回调地狱里往外拉了一大步。
