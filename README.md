#### 一、简答题
**1、当我们点击按钮的时候动态给data增加的成员是否是响应式数据，如果不是的话，如果把新增成员设置成响应式数据，它的内部原理是什么。**

```javascript
let vm = new Vue({
 el: '#el'
 data: {
  o: 'object',
  dog: {}
 },
 method: {
  clickHandler () {
   // 该 name 属性是否是响应式的
   this.dog.name = 'Trump'
  }
 }
})
```
**解析：**  vue中的数据是在创建vue实例时被创建的响应式，在vue.js中new Observer()中将数据转成响应式，但是当实例创建完后给实例新增的成员只是给实例对象添加了一个js属性，所以他不是响应式的。如果把新增成员设置成响应式数据，可以通过Vue.set(object,propertyName,value)/this.$set(object,propertyName,value)方法想嵌套对象添加响应式属性，在vue中set的源码为：

```javascript
export function set (target: Array<any> | Object, key: any, val: any): any {
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    target.length = Math.max(target.length, key)
    target.splice(key, 1, val)
    return val
  }
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  const ob = (target: any).__ob__
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  if (!ob) {
    target[key] = val
    return val
  }
  defineReactive(ob.value, key, val)
  ob.dep.notify()
  return val
}
```
上述代码执行过程：   
1、如果是在开发环境，且 target 未定义（为 null、undefined ）或 target 为基础数据类型（string、boolean、number、symbol）时，抛出告警；     
2、如果 target 为数组且 key 为有效的数组 key 时，将数组的长度设置为 target.length 和 key 中的最大的那一个，然后调用数组的 splice 方法（ vue 中重写的 splice 方法）添加元素；     
3、如果属性 key 存在于 target 对象中且 key 不是 Object.prototype 上的属性时，表明这是在修改 target 对象属性 key 的值（不管 target 对象是否是响应式的，只要key存在于target对象中，就执行这一步逻辑），此时就直接将 value 直接赋值给 target[key]；     
4、判断target，当target为vue实例或根数据data对象时，在开发环境下抛错；  
5、当一个数据为响应式时，vue会给该数据添加一个ob属性，因此可以通过判断target对象是否存在ob属性来判断target是否是响应式数据，当target是非响应式数据时，我们就按照普通对象添加属性的方式来处理；当target对象是响应式数据时，我们将 target 的属性 key 也设置为响应式并手动触发通知其属性值的更新；           
-----> defineReactive(ob.value, key, val) 是将新增属性设置为响应式; ob.dep.notify() 手动触发通知该属性值的更新。    
-----> 所以将代码修改为：

```Javascript
let vm = new Vue({
        el: '#app',
        data: {
            msg: 'object',
            dog: {
                name: undefined
            }
        },
        method: {
            clickHandler() {
                this.$set(this.data.dog, name, 'Trump')
            }
        }
    })
```

 

**2、请简述 Diff 算法的执行过程**    
**解析：** Diff算法的执行就是执行patch 函数，在patch函数中传入新旧VNode，比较新旧节点的差异，把差异渲染到DOM，返回新的VNode，作为下一次patch()的oldVnode。  
**patch()的执行过程**：     
* 如果oldVnode和vnode相同（key和sel相同）调用patchVnode()，找节点的差异并更新DOM；
* 如果oldVnode 是DOM元素
    - 把DOM元素转换成oldVnode
    - 调用createElm()把vnode 转换为真实 DOM，记录到vnode.elm
    - 把刚创建的DOM元素插入到parent中
    - 移除老节点
    - 触发用户设置的create钩子函数     
    
**patchVnode()的执行过程**      
* patchVnode(oldVnode, vnode, insertedVnodeQueue)
* 对比oldVnode和vnode的差异，把差异渲染到 DOM。
    - 如果他们都有文本节点并且不相等，那么将 el 的文本节点设置为 vnode 的文本节点。
    - 如果 oldVnode 有子节点而 vnode 没有，则删除 el 的子节点。
    - 如果 oldVnode 没有子节点而 vnode 有，则将 vnode 的子节点真实化之后添加到 el
    - 如果两者都有子节点，则执行 updateChildren 函数比较子节点。
    
**updateChildren()的执行过程**  

```javascript
// updateChildren()

updateChildren (parentElm, oldCh, newCh) {
    let oldStartIdx = 0  //oldVnode 的 startIdx, 初始值为 0
    let newStartIdx = 0 //vnode 的 startIdx, 初始值为 0
    let oldEndIdx = oldCh.length - 1 //oldVnode 的 endIdx, 初始值为 oldCh.length - 1
    let oldStartVnode = oldCh[0] //oldVnode 的初始开始节点
    let oldEndVnode = oldCh[oldEndIdx] //oldVnode 的初始结束节点
    let newEndIdx = newCh.length - 1  //vnode 的 endIdx, 初始值为 newCh.length - 1
    let newStartVnode = newCh[0]  //vnode 的初始开始节点
    let newEndVnode = newCh[newEndIdx] //vnode 的初始结束节点
    let oldKeyToIdx
    let idxInOld
    let elmToMove
    let before
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (oldStartVnode == null) {   // 对于vnode.key的比较，会把oldVnode = null
            oldStartVnode = oldCh[++oldStartIdx] 
        }else if (oldEndVnode == null) {
            oldEndVnode = oldCh[--oldEndIdx]
        }else if (newStartVnode == null) {
            newStartVnode = newCh[++newStartIdx]
        }else if (newEndVnode == null) {
            newEndVnode = newCh[--newEndIdx]
        }else if (sameVnode(oldStartVnode, newStartVnode)) {
            patchVnode(oldStartVnode, newStartVnode)
            oldStartVnode = oldCh[++oldStartIdx]
            newStartVnode = newCh[++newStartIdx]
        }else if (sameVnode(oldEndVnode, newEndVnode)) {
            patchVnode(oldEndVnode, newEndVnode)
            oldEndVnode = oldCh[--oldEndIdx]
            newEndVnode = newCh[--newEndIdx]
        }else if (sameVnode(oldStartVnode, newEndVnode)) {
            patchVnode(oldStartVnode, newEndVnode)
            api.insertBefore(parentElm, oldStartVnode.el, api.nextSibling(oldEndVnode.el))
            oldStartVnode = oldCh[++oldStartIdx]
            newEndVnode = newCh[--newEndIdx]
        }else if (sameVnode(oldEndVnode, newStartVnode)) {
            patchVnode(oldEndVnode, newStartVnode)
            api.insertBefore(parentElm, oldEndVnode.el, oldStartVnode.el)
            oldEndVnode = oldCh[--oldEndIdx]
            newStartVnode = newCh[++newStartIdx]
        }else {
          // 使用key时的比较
            if (oldKeyToIdx === undefined) {
                oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx) // 有key生成index表
            }
            idxInOld = oldKeyToIdx[newStartVnode.key]
            if (!idxInOld) {
                api.insertBefore(parentElm, createEle(newStartVnode).el, oldStartVnode.el)
                newStartVnode = newCh[++newStartIdx]
            }
            else {
                elmToMove = oldCh[idxInOld]
                if (elmToMove.sel !== newStartVnode.sel) {
                    api.insertBefore(parentElm, createEle(newStartVnode).el, oldStartVnode.el)
                }else {
                    patchVnode(elmToMove, newStartVnode)
                    oldCh[idxInOld] = null
                    api.insertBefore(parentElm, elmToMove.el, oldStartVnode.el)
                }
                newStartVnode = newCh[++newStartIdx]
            }
        }
    }
    if (oldStartIdx > oldEndIdx) {
        before = newCh[newEndIdx + 1] == null ? null : newCh[newEndIdx + 1].el
        addVnodes(parentElm, before, newCh, newStartIdx, newEndIdx)
    }else if (newStartIdx > newEndIdx) {
        removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)
    }
}
```

* 当 oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx 时，执行如下循环判断：
    - oldStartVnode 为 null，则 oldStartVnode 等于 oldCh 的下一个子节点，即 oldStartVnode 的下一个兄弟节点
    - oldEndVnode 为 null, 则 oldEndVnode 等于 oldCh 的相对于 oldEndVnode 上一个子节点，即 oldEndVnode 的上一个兄弟节点
    - newStartVnode 为 null，则 newStartVnode 等于 newCh 的下一个子节点，即 newStartVnode 的下一个兄弟节点
    - newEndVnode 为 null, 则 newEndVnode 等于 newCh 的相对于 newEndVnode 上一个子节点，即 newEndVnode 的上一个兄弟节点
    - oldEndVnode 和 newEndVnode 为相同节点则执行 patchVnode(oldStartVnode, newStartVnode)，执行完后 oldStartVnode 为此节点的下一个兄弟节点，newStartVnode 为此节点的下一个兄弟节点
    - oldEndVnode 和 newEndVnode 为相同节点则执行 patchVnode(oldEndVnode, newEndVnode)，执行完后 oldEndVnode 为此节点的上一个兄弟节点，newEndVnode 为此节点的上一个兄弟节点
    - oldStartVnode 和 newEndVnode 为相同节点则执行 patchVnode(oldStartVnode, newEndVnode)，执行完后 oldStartVnode 为此节点的下一个兄弟节点，newEndVnode 为此节点的上一个兄弟节点
    - oldEndVnode 和 newStartVnode 为相同节点则执行 patchVnode(oldEndVnode, newStartVnode)，执行完后 oldEndVnode 为此节点的上一个兄弟节点，newStartVnode 为此节点的下一个兄弟节点
    - 使用 key 时的比较：     
       oldKeyToIdx为未定义时，由 key 生成 index 表，具体实现为  createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)。

* createKeyToOldIdx
    ```javascript
    
    function createKeyToOldIdx(children: Array<VNode>, beginIdx: number, endIdx: number): KeyToIndexMap {
      let i: number, map: KeyToIndexMap = {}, key: Key | undefined, ch;
      for (i = beginIdx; i <= endIdx; ++i) {
        ch = children[i];
        if (ch != null) {
          key = ch.key;
          if (key !== undefined) map[key] = i;
        }
      }
      return map;
    }
    ```
    -----> 在createKeyToOldIdx方法中，用oldCh中的key属性作为键，而对应的节点的索引作为值。然后再判断在newStartVnode的属性中是否有key，且是否在oldKeyToIndx中找到对应的节点。
    
    - 如果不存在这个 key，那么就将这个 newStartVnode 作为新的节点创建且插入到原有的 root 的子节点中，然后将 newStartVnode 替换为此节点的下一个兄弟节点。
    - 如果存在这个key，那么就取出 oldCh 中的存在这个 key 的 vnode，然后再进行 diff 的过程，并将 newStartVnode 替换为此节点的下一个兄弟节点。
* 当上述 9 个判断执行完后，oldStartIdx 大于 oldEndIdx，则将 vnode 中多余的节点根据 newStartIdx 插入到 dom 中去；newStartIdx 大于 newEndIdx，则将 dom 中在区间 [oldStartIdx， oldEndIdx]的元素节点删除。

---
#### 二、编程题
**1、模拟 VueRouter 的 hash 模式的实现，实现思路和 History 模式类似，把 URL 中的 # 后面的内容作为路由的地址，可以通过 hashchange 事件监听路由地址的变化。**
 
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        #nav a {
            margin-right: 10px;
        }

        #nav a.act {
            color: #FF0000;
        }
    </style>
</head>
<body>
    <nav id="nav"></nav>
    <main id="app"></main>
    <script>
        class Router {
            constructor() {
                this.navs = [{
                    path: "#index",
                    title: "首页",
                    content: "首页-内容"
                }, {
                    path: "#news",
                    title: "新闻",
                    content: "新闻-内容"
                }, {
                    path: "#about",
                    title: "关于",
                    content: "关于-内容"
                }]

                this.navNode = document.getElementById("nav");
                this.el = document.getElementById("app");
            }
            init() {
                this.createNav();
                this.haddleHashChage();
                //监听hash值变动
                window.addEventListener("hashchange", this.haddleHashChage.bind(this));
            }
            createNav() {//创建导航
                let fragment = document.createDocumentFragment();
                this.navs.forEach(nav => {
                    let tagA = document.createElement("a");
                    tagA.href = nav.path;
                    tagA.innerText = nav.title;
                    fragment.appendChild(tagA);
                })
                this.navNode.appendChild(fragment);
            }
            haddleHashChage() {//根据hash值，变动内容
                const hashVal = window.location.hash || this.navs[0].path;
                this.navs.forEach((nav, index) => {
                    let curNodes = this.navNode.childNodes[index];
                    curNodes.className = "";
                    if (nav.path == hashVal) {
                        curNodes.className = "act";
                        this.el.innerHTML = nav.content;
                    }
                })
            }
        }

        const router = new Router();
        router.init();
    </script>
</body>
</html>
```


**2、在模拟 Vue.js 响应式源码的基础上实现 v-html 指令，以及 v-on 指令。**
 

```javascript
// minivue.js

class Vue {
  constructor(options) {
    // 1. 通过属性保存选项的数据
    this.$options = options || {}
    this.$data = options.data || {}
    this.$methods = options.methods || {}
    this.$el = typeof options.el === 'string' ? document.querySelector(options.el) : options.el  // '#App' App
    // 2. 把data中的成员转成getter和setter，注入到vue的实例中
    this._proxyData(this.$data)
    // 把 methods 中的成员注入到 vue 实例中 
    this._proxyMethods(this.$methods)
    // 3. 调用observer对象，监听数据的变化
    new Observer(this.$data)
    // 4. 调用compiler对象，解析指定和插值表达式
    new Compiler(this) // 传入vue实例
  }
  _proxyData(data) {
    // 遍历data中的所有属性
    Object.keys(data).forEach(key => {
      // 把data的属性注入到vue实例中
      Object.defineProperty(this, key, {
        enumerable: true,
        configurable: true,
        get() {
          return data[key]
        },
        set(newVal) {
          if (newVal === data[key]) {
            return
          }
          data[key] = newVal
        }
      })
    })
  }

  _proxyMethods(methods) {
    Object.keys(methods).forEach(key => {
      // 把 methods 的成员注入到 vue 实例中
      this[key] = methods[key]
    })
  }
}
```

```javascript
// compiler.js

class Compiler {
  constructor (vm) {
    this.el = vm.$el
    this.vm = vm
    this.compile(this.el) // 编译模板
  }
  // 编译模板，处理文本节点和元素节点
  compile (el) {
    let childNodes = el.childNodes
    Array.from(childNodes).forEach(node => {
      // 处理文本节点
      if (this.isTextNode(node)) {
        this.compileText(node)
      } else if (this.isElementNode(node)) {
        // 处理元素节点
        this.compileElement(node)
      }

      // 判断node节点，是否有子节点，如果有子节点，要递归调用compile
      if (node.childNodes && node.childNodes.length) {
        this.compile(node)
      }
    })
  }
  // 编译元素节点，处理指令
  compileElement (node) {
    // console.log(node.attributes)
    // 遍历所有的属性节点
    Array.from(node.attributes).forEach(attr => {
      // 判断属性是否是指令
      let attrName = attr.name
      if (this.isDirective(attrName)) { 
        // v-text --> text
        attrName = attrName.substr(2)
        let key = attr.value
        if (attrName.startsWith('on')) {
          const event = attrName.replace('on:', '') // 获取事件名
              // 事件更新
          return this.eventUpdate(node, key, event)
        }
        this.update(node, key, attrName)
      }
    })
  }
  
  update (node, key, attrName) {
    let updateFn = this[attrName + 'Updater']
    updateFn && updateFn.call(this, node, this.vm[key], key)
  }
  eventUpdate(node, key, event) {
    this.onUpdater(node, key, event)
  }
  // 处理 v-text 指令
  textUpdater (node, value, key) {
    node.textContent = value
    new Watcher(this.vm, key, (newValue) => {
      node.textContent = newValue
    })
  }
  // 处理 v-model 指令
  modelUpdater (node, value, key) {
    node.value = value
    new Watcher(this.vm, key, (newValue) => {
      node.value = newValue
    })
    // 双向绑定
    node.addEventListener('input', () => {
      this.vm[key] = node.value
    })
  }

  // 处理 v-html 指令
  htmlUpdater(node, value, key) {
    node.innerHTML = value
    new Watcher(this.vm, key, (newValue) => {
        node.innerHTML = newValue
    })
  }

  // 处理 v-on 指令
  onUpdater(node, key, event) {
    node.addEventListener(event, (e) => this.vm[key](e))
  }
  
  // 编译文本节点，处理差值表达式
  compileText (node) {
    // console.dir(node)
    // {{  msg }}
    let reg = /\{\{(.+?)\}\}/
    let value = node.textContent
    if (reg.test(value)) {
      let key = RegExp.$1.trim()
      node.textContent = value.replace(reg, this.vm[key])

      // 创建watcher对象，当数据改变更新视图
      new Watcher(this.vm, key, (newValue) => {
        node.textContent = newValue
      })
    }
  }
  // 判断元素属性是否是指令
  isDirective (attrName) {
    return attrName.startsWith('v-')
  }
  // 判断节点是否是文本节点
  isTextNode (node) {
    return node.nodeType === 3
  }
  // 判断节点是否是元素节点
  isElementNode (node) {
    return node.nodeType === 1
  }
}
```


**3、参考 Snabbdom 提供的电影列表的示例，利用Snabbdom 实现类似的效果，如图：**

```
import { h, init } from 'snabbdom'
import style from 'snabbdom/modules/style'
import eventlisteners from 'snabbdom/modules/eventlisteners'
import { originalData } from './originData'

let patch = init([style,eventlisteners])

let data = [...originalData]
const container = document.querySelector('#container')

var sortBy = 'rank';
let vnode = view(data);

// 初次渲染
let oldVnode = patch(container, vnode)


// 渲染
function render() {
    oldVnode = patch(oldVnode, view(data));
}
// 生成新的VDOM
function view(data) {
    return h('div#container',
        [
            h('h1', 'Top 10 movies'),
            h('div',
                [
                    h('a.btn.add',
                        { on: { click: add } }, 'Add'),
                    'Sort by: ',
                    h('span.btn-group',
                        [
                            h('a.btn.rank',
                                {
                                    'class': { active: sortBy === 'rank' },
                                    on: { click: [changeSort, 'rank'] }
                                }, 'Rank'),
                            h('a.btn.title',
                                {
                                    'class': { active: sortBy === 'title' },
                                    on: { click: [changeSort, 'title'] }
                                }, 'Title'),
                            h('a.btn.desc',
                                {
                                    'class': { active: sortBy === 'desc' },
                                    on: { click: [changeSort, 'desc'] }
                                }, 'Description')
                        ])
                ]),
            h('div.list', data.map(movieView))
        ]);
}

// 添加一条数据 放在最上面
function add() {
    const n = originalData[Math.floor(Math.random() * 10)];
    data = [{ rank: data.length+1, title: n.title, desc: n.desc, elmHeight: 0 }].concat(data);
    render();
}
// 排序
function changeSort(prop) {
    sortBy = prop;
    data.sort(function (a, b) {
        if (a[prop] > b[prop]) {
            return 1;
        }
        if (a[prop] < b[prop]) {
            return -1;
        }
        return 0;
    });
    render();
}

// 单条数据
function movieView(movie) {
    return h('div.row', {
        key: movie.rank,
        style: {
            display: 'none', 
            delayed: { transform: 'translateY(' + movie.offset + 'px)', display: 'block' },
            remove: { display: 'none', transform: 'translateY(' + movie.offset + 'px) translateX(200px)' }
        },
        hook: {
            insert: function insert(vnode) {
                movie.elmHeight = vnode.elm.offsetHeight;
            }
        }
    }, [
        h('div', { style: { fontWeight: 'bold' } }, movie.rank),
        h('div', movie.title), h('div', movie.desc),
        h('div.btn.rm-btn', {on: { click: [remove, movie]}}, 'x')]);
}
// 删除数据
function remove(movie) {
    data = data.filter(function (m) {
        return m !== movie;
    });
    render()
}
```
