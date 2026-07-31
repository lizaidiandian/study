# 从 createApp 到虚拟 DOM：Vue3 背后的思路变了什么

## Vue2 时代的问题

**createApp**,一个vue3新加的内容,他为了解决什么麻烦而诞生?

在 vue2 的时代,创建一个应用是这样的:

```vue
Vue.component('My-button',{...})
Vue.mixin({...})
```
最后
```vue
new Vue({render: h => h(App)}).$mount('#app')
```

所有东西都挂在一个共享的 **Vue 构造函数** 上面,看起来挺好,但问题也出现在共享上

打个比方: a公司 和 b公司 共用一个厨房

![[编程学习/nodejs学习路径/createApp到虚拟DOM/1.png]]

小项目中还好,一旦你的页面上需要运行两个独立的vue应用,那么它们的全局配置会互相污染,比如
1. app a注册的组件, app b 也能看到
2. app b加的全局混入,app a 也被影响了

## createApp 做了什么改变

那么, **createApp** 来了,核心思想为
```
不再让所有应用共享同一个全局 Vue 对象，而是让每个应用拥有自己独立的**应用实例**。
```

![[编程学习/nodejs学习路径/createApp到虚拟DOM/2.png]]


那么,还能带来一个好处,**最终包体积更小**,

来看代码
```vue
//vue2
import Vue from 'vue'

//vue3 
import {createApp,nextTick} from 'vue'
```

vue2里,这种写法并不利于打包工具去精确判断你到底用了哪些能力

但vue3里,如果你只用了nextTick,打包工具就更容易只把你真正用到的部分留下来

这有个专门的术语 **Tree-Shaking**,直译是 "摇树"

它是现代打包工具(如:vite)的一项核心能力:
```
打包时,自动检测并剔除你代码中从未使用过的部分
```

vue3的这种写法,打包工具更容易分析你用了什么,没用什么,而vue2那种默认导入整个 Vue 的方式,就没那么容易做精确裁剪


但是有个前提,它只对 **ES Module 的具名导入** 更容易生效:

什么是ES Module 的具名导入?

那我们先知道 模块 是什么

在 js 中,一个文件就是一个模块,模块之间通过 export 和 import 来交换东西

第一种:默认导出

```js
//文件名 car.js
let car = {
	name:'雅迪',
	price: 3000
}

export default car 

//导入时
import car from './car.js'

console.log(car.name) // '雅迪'
console.log(car) // {name:'雅迪',price:3000}
```

在这里,即使你不用 price,它也更难被精确裁掉

第二种:具名导出

```js
export let name = '雅迪'
export let price = 3000

import {name} from './car.js'

console.log(name) //雅迪
```
在这里,就像我们点名了只要 name ,所以 price 就更容易在打包时被剔除掉



接着来看 **createApp** 最基本的用法
```js
import {createApp} from 'vue'
import App from './App.vue'

const app = createApp(App)
app.mount('#app')
//也可以缩写成 createApp(App).mount('#app')
```

来看他们背后做了些什么事

1. **创建应用实例**
![[编程学习/nodejs学习路径/createApp到虚拟DOM/3.png]]
此时,你获取了图中的能力,但并没有开始使用

2. **中间配置**(这一步按需来做)
```js
const app = createApp(App)
//注册一个全局按钮组件
app.component('MyButton', ...)
//安装 路由 插件
app.use(router)
//错误处理
app.config.errorHandler = (err) => {...}
//挂载
app.mount('#app')
```

3. **挂载** `app.mount('#app')` 
    **#app**  是一个 css id选择器
  ```html
  //index.html
  <div id="app"></div>
  ```
 它做了三件事
 ![[编程学习/nodejs学习路径/createApp到虚拟DOM/4.png]]
 图中第三点举例
 
 ```html
 //之前
 <div id="app"></div>
 
 //之后
 <div id="app">
	<header>...</header>
	<main>...</main>
	<footer>...</footer>
 </div>
 ```


再讲一个 **虚拟DOM**

**DOM**(Document Object Model，文档对象模型),浏览器里的是 **真实 DOM**，而 **虚拟 DOM**
是框架在 JS 里维护的一层表示，不是浏览器原生 DOM 的一种

先知道 DOM 本身是什么

浏览器把你的 HTML 解析之后,在内存中构建出一棵树,页面上每个元素就是树上的一个节点

![[编程学习/nodejs学习路径/createApp到虚拟DOM/5.png]]
这棵像倒过来的树一样的就是 真实DOM ,换成代码就是
```html
//只展示转换内容
<html>
<head>
  <title>Myfriend</title>
</head>
<body>
  <div id="app">
    <h1>hello</h1>
    <ul>
      <li>A</li>
      <li>B</li>
    </ul>
  </div>
</body>
</html>
```

那我们直接去修改真实DOM有什么问题?

假设我们有一个列表,列表中有1000条数据,现在,修改第 12 条数据的内容
1. 全部重新加载:把 1000 条数据全部删掉,重新创建 1000 条数据
2. 手动修改: 精准找到 第12条,但是,当页面结构变复杂,几十个组件,上百个状态,如果继续手动追踪,那 心力耗费 极大

那么,**虚拟 DOM** 出现了

他就是一个普通的 js 对象

**真实 DOM 节点**:一个真正的 `<div>`,浏览器要为他分配内存,计算样式,确定布局位置,绑定事件监听器

**虚拟 DOM 节点(VNode)**
```js
{
	tag:'div',
	props:{id:'app'},
	children:[
		{tag:'h1',children:['hello']}
	]
}
```

**Diff 算法**:找出不同的地方

把 "hello" 改成 "world"

![[编程学习/nodejs学习路径/createApp到虚拟DOM/6.png]]

diff 两棵树节点分别对比
![[编程学习/nodejs学习路径/createApp到虚拟DOM/7.png]]

**patch** 只更新那一个真实 DOM
![[编程学习/nodejs学习路径/createApp到虚拟DOM/8.png]]

再看总流程

![[编程学习/nodejs学习路径/createApp到虚拟DOM/9.png]]

再说下 **patch** 的操作类型
```
1.替换文本
node.textContent = "world"

2.修改属性
class="old" -> class="new"
el.setAttribute('class','new')

3.新增节点
parent.appendChild(新节点)

4.删除节点
parent.removeChild(旧节点)

5.移动节点
parent.insertBefore(节点,新位置)
```

## 总结

最后,如果只用一句话去理解这篇,那就是: **Vue3 不只是把 `new Vue()` 改成了 `createApp()`，而是顺手把“应用怎么创建”“能力怎么挂载”“代码怎么被裁剪”“页面怎么被更新”这一整套思路都重新整理了一遍。**

前半篇讲的 `createApp`，本质上是在解决 Vue2 时代全局共享、多个应用互相污染的问题；中间讲的 Tree Shaking 和模块导出，是在说明 Vue3 为什么更适合现代打包工具；后半篇讲虚拟 DOM、diff 和 patch，则是在回答：应用创建完之后，Vue 最后到底是怎么把变化落到页面上的。

所以这篇真正想串起来的不是几个零散知识点，而是一条完整的线：**从应用实例的创建，到页面更新的发生，Vue3 背后的设计思路和 Vue2 已经不太一样了。**
