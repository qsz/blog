# ts实现一个简易的vue-router

本文将学习：

* 前端路由的实现
* vue 插件编写
* vue-router基本原理实现



## 前端路由

在vue-router路由中，常用到有`hash`和`history`两种模式。这两种模式，分别利用了`hash`和`window.history API`

### hash

> http:example.com/#/homepage

上面的url中，**#**及其后面的部分，我们称之为`hash`, 也可以叫做`URL片段标识符`。`hash`不会随着浏览器的请求上传给服务器，且改变`hash`值不会刷新页面。这一特性奠定了单页面的基础之一，即页面跳转无刷新。

#### hashchange事件

关于**hashchange**事件, 在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/hashchange_event)中是这样定义的

> 当URL的片段标识符更改时，将触发**hashchange**事件 (跟在＃符号后面的URL部分，包括＃符号)

因此只要改变`hash`值，就会触发**hashchange**事件。改变`hash`值的主要方法有：

* `<a>`标签
* 浏览器的前进后退
*  `window.location`的相关api(`window.location.hash`，` window.location.replace`)

接下来举个例子

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>hash</title>
</head>
<body>
    <button onclick="changeHash('/foo')">foo</button>
    <button onclick="changeHash('/bar')">bar</button>

    <div id='app'></div>
</body>
<script>
const app = document.getElementById('app')
window.addEventListener('hashchange', (event) => {
    console.log(event)
    app.innerText = event.newURL
})

function changeHash(hash) {
    window.location.hash = hash
}
</script>
</html>
```

运行上述代码，点击不同的button，利用`window.location.hash`改变页面url的`hash`部分，可以发现，`hashchange`事件被触发，`#app`展示的内容也随之改变。这也是`vue-router`的`hash`模式实现的基本原理。

### window.history API

在[`vue-router`](https://router.vuejs.org/zh/guide/essentials/navigation.html)文档里写着

> 你也许注意到 `router.push`、 `router.replace` 和 `router.go` 跟 [`window.history.pushState`、 `window.history.replaceState` 和 `window.history.go`](https://developer.mozilla.org/en-US/docs/Web/API/History)好像， 实际上它们确实是效仿 `window.history` API 的。

#### 在history中跳转

`history`提供用 `back()`, `forward()`和 `go()` 方法来完成在用户历史记录中向后和向前的跳转

##### **window.go**

 `go(number)` 方法载入到会话历史中的某一特定页面。

`window.history.go(-1)`向后移动一个页面，等同于`window.back()`

`window.history.go(1)`向前移动一个页面, 等同于调用了 `window.forward()`

#### 添加和修改历史记录中的条目

`history.pushState()`和 `history.replaceState()`是`HTML5`中引入的方法。它们分别可以**无刷新**的添加和修改历史记录条目。通常与`window.onpopstate`事件配合使用

##### history.pushState()

`history`对象提供了对浏览器的会话历史记录的访问。我们可以把历史记录看成`栈`，`history.pushState()`就是在`栈`末尾再增加一条历史记录。他有三个参数： 一个状态对象, 一个标题, 和 (可选的) 一个URL

* **状态对象** — 状态对象state是一个JavaScript对象，表示创建新的历史记录条目。在`popstate`事件中event对象的`state`属性可以拿到该值
* **标题** — Firefox 目前忽略这个参数，但未来可能会用到
* **URL** — 该参数定义了新的历史URL记录

##### history.replaceState()

`history.replaceState()` 的使用与 `history.pushState()` 非常相似，区别在于  `replaceState()`  是修改了当前的历史记录项而不是新建一个

#### window.onpopstate事件

在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/onpopstate)中是这样定义的

> `每当处于激活状态的历史记录条目发生变化时,popstate`事件就会在`对应window`对象上触发. 如果当前`处于激活状态的历史记录条目是由``history.pushState()`方法创建,或者由`history.replaceState()方法修改过`的, 则`popstate事件对象的``state`属性包含了这个历史记录条目的state对象的一个拷贝

在这里需要注意的是`pushState`和`replaceState`只会添加或修改一个历史记录条目，并不会触发`onpopstate`事件。只有点击浏览器后退和前进按钮，或者调用`histtory`提供的 `back()`, `forward()`和 `go()` 方法才会触发该事件

#### 举个例子

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>history</title>
</head>
<body>
<div>
  <button onclick="pushState('foo1', '/foo1')">pushState_foo</button>
  <button onclick="pushState('bar1', '/bar1')">pushState_bar</button>
</div>

<div>
  <button onclick="replaceState('foo2', '/foo2')">replaceState_foo</button>
  <button onclick="replaceState('bar2', '/bar2')">replaceState_bar</button>
</div>

<div>
  <button onclick="back()">history.back()</button>
  <button onclick="go()">history.go(1)</button>
</div>


<div id='app'></div>
</body>
<script>
  const app = document.getElementById('app')

  window.addEventListener('popstate', (event) => {
    app.innerText = event.state.name
  })

  function pushState(name, hash) {
    window.history.pushState({name}, name, hash)
  }

  function replaceState(name, hash) {
    window.history.replaceState({name}, name, hash)
  }

  function back() {
    window.history.back()
  }

  function go() {
    history.go(1)
  }
</script>
</html>

```

`history`的`length`属性，返回浏览器历史列表中的 URL 数量。

1.当页面第一次加载完成:  `history.length`  === 1

2.再点击`pushState_foo`按钮：url变为`/foo1`，  `history.length`  === 2。由此可见历史记录中新增了一个条目

3.再点击`replaceState_bar`按钮：url变为`/bar2`，  `history.length`  === 2。点击浏览器回退按钮，`url`变为`/`而不是`/foo1`。由此可见历史记录中的`/foo1`条目被替换为`/bar2`

4.再点击`history.go`按钮，`url`变为`bar2`，且`popstate`触发。`event.state`中拿到了` window.history.replaceState`时存储的信息，页面显示`bar2`

#### 对比

与`hash`相比，`history`模式不但`url`显示简洁，而且新的历史记录项可以关联任意数据，而基于哈希值的方式，则必须将所有相关数据编码到一个短字符串里。

#### 参考

* [History_API](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)



## 实现TsVRouter

[完整代码地址](https://github.com/qsz/TsVRouter)

### Vue插件开发

`Vue.use(plugin)`用于安装Vue插件。需要`plugin`提供`install`方法。这个方法的第一个参数是 `Vue` 构造器，第二个参数是一个可选的选项对象，如下所示

```js
MyPlugin.install = function (Vue, options) {
  // 1. 添加全局方法或属性
  Vue.myGlobalMethod = function () {
    // 逻辑...
  }

  // 2. 添加全局资源
  Vue.directive('my-directive', {
    bind (el, binding, vnode, oldVnode) {
      // 逻辑...
    }
    ...
  })

  // 3. 注入组件选项
  Vue.mixin({
    created: function () {
      // 逻辑...
    }
    ...
  })

  // 4. 添加实例方法
  Vue.prototype.$myMethod = function (methodOptions) {
    // 逻辑...
  }
}
```

### 功能实现

利用[start-tslib](https://www.npmjs.com/package/start-tslib)快速搭建一个typescript开发环境

#### 定义类型

为了类型约束和代码提示，我们首先定义`TsRouter`所需的类型和`TsVue`类型。

**RouterMode**：路由模式

```typescript
type RouterMode = 'html5' | 'hash'
```
**TsRouterOptions**：路由参数类型
```typescript
// 单个Route配置
interface RouteConfig {
    path : string
    name ?: string
    component : ComponentOptions<Vue>
}

// 参数类型接口
interface TsRouterOptions {
    routes : RouteConfig[]
    mode ?: RouterMode
}
```
**Route**：单个Route接口。包含了当前路由信息，即`event.state`

```typescript
// 单个Route接口
export interface Route {
    path ?: string
    name ?: string
    hash ?: string
}
```

**TsRouter**：抽象类。包含了`install`，`mode`和路由跳转方法等信息

```typescript
export abstract class TsRouter {
    static install: Function;
    abstract mode: RouterMode;
    abstract options: TsRouterOptions;
    abstract push(state: Route, url: any): void;
    abstract replace(state: Route, url: any): void;
    abstract go(n: number): void;
    abstract maper: Maper;
    abstract history: History;
}

```

**TsVue**：用于调用`vue`相关方法时候的代码提示

```typescript
import Vue, { CreateElement, VueConstructor, ComponentOptions } from "vue";
namespace TsVue {
    export interface TsConstructor extends VueConstructor {
        util: {
            defineReactive: Function
        } 
    } 
    export interface TsCreateElement extends CreateElement {}
    export interface TsComponentOptions extends ComponentOptions<Vue> {}
    export interface $options extends ComponentOptions<Vue> {}
    export interface TVue extends Vue {
        $options: ComponentOptions<Vue> & { tsRouter: TsRouter};
        $tsRouter: TsRouter;
        $tsRoute: Route;
        $parent: TVue;
        _tsRouterRoot: TVue;
        _tsRouter: TsRouter;
        _tsRoute: Route;
    }
}
```

#### 实现install方法

首先看代码

```typescript
// 插件应该暴露一个 install 方法。这个方法的第一个参数是 Vue 构造器
function install(this: TsVue.TVue, Vue: TsVue.TsConstructor) {
    Vue.mixin({
        beforeCreate (this: TsVue.TVue) {
          // 这里的 this 是 Vue根实例,为每个Vue根实例缓存一个Vue根实例,方便Vue组件实例中读取到Vue根实例
          // 根实例中的 $options 与 组件实例中 的$options 不同
          // 其中之一是根实例中的 $options 可以获取到配置项
          // 配置项中有我们的router实例
          if(this.$options.tsRouter) { // 说明是 根实例
            this._tsRouterRoot = this;
            this._tsRouter = this.$options.tsRouter;
            // 关键的地方，将_tsRoute变为响应式, 既将this._tsRouter.history和其子属性变为响应式
            // history 为路由实例的history对象
            Vue.util.defineReactive(this, '_tsRoute', this._tsRouter.history) 
          } else {
            // 把 _tsRouterRoot 传递给 组件实例
            this._tsRouterRoot = (this.$parent && this.$parent._tsRouterRoot) || this
          }
        }
    })

    // 为每个Vue组件实例注入$tsRouter，用于获取当前的router实例
    Object.defineProperty(Vue.prototype, '$tsRouter', {
      get () { 
        return this._tsRouterRoot._tsRouter 
      }
    })

    // $tsRoute： 当前路由信息
    Object.defineProperty(Vue.prototype, '$tsRoute', {
      get () { 
        return this._tsRouterRoot._tsRoute.current   // current为当前路由信息
        // return this._tsRouterRoot._tsRouter.history   // current为当前路由信息
      }
    })
    // 全局注册组件
    Vue.component('tsrouter-view', RouterView)
    Vue.component('tsrouter-link', RouterLink)
} 
```

`install`的第一个参数是 Vue 构造器，可以调用Vue提供的全局方法。在`install`里，用`Vue.mixin`全局混入，这将影响注册之后所有创建的每个 Vue 实例。

##### **vm.$options**

在这里为实例提供`beforeCreate`钩子：在这个钩子中，通过`this.$options`可以获取当前 Vue 实例的初始化选项。举个例子：

```js
new Vue({
  customOption: 'foo',
  created: function () {
    console.log(this.$options.customOption) // => 'foo'
  }
})
```

在这里，**需要注意的是**：这里的`this`指的是根实例，与我们组件实例不同。在组件实例中无法通过`this.$options`获取到初始化选项。因此在组件实例中，用`this.$parent._tsRouterRoot`获取当前组件的父实例即根实例的初始化选项

##### **Vue.utils.defineReactive**

`defineReactive`是vue内部提供的方法，定义在`/src/core/observer`。功能是将一个对象的子对象递归调用`Object.defineProperty`，使该对象的所有子对象都具有响应式

```typescript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  const property = Object.getOwnPropertyDescriptor(obj, key)
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  const getter = property && property.get
  const setter = property && property.set
  if ((!getter || setter) && arguments.length === 2) {
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      // #7981: for accessor properties without setter
      if (getter && !setter) return
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      dep.notify()
    }
  })
}
```

我们利用该方法：`Vue.util.defineReactive(this, '_tsRoute', this._tsRouter.history) `，将`tsRouter的history`变为一个响应式对象。`history`中设置了`current`用于表示当前路由信息。这样，每当我们改变`history.current`，用到`current`值的vue实例（如`router-view`）就会触发`render`重新渲染，形成页面的改变。关于这部分的实现，在后面`view`实现会具体提到

```typescript
interface History {
    current: Route;
    rootRouter: TsRouter;
    push(state: Route, url: any): void;
    replace(state: Route,url: any): void;
    go(n: number): void;
}
```

##### **Object.defineProperty**

在`install`中，用`Object.defineProperty`扩展`Vue.prototype`，添加`$tsRouter`和`$tsRoute`，分别表示路由实例和路由信息，这样在每个页面中可以通过`this.$tsRouter`来调用路由提供的`push`等方法，就像`vue-router`中调用`this.$router.push`那样。

##### **Vue.component**

用`Vue.component`全局注册`tsrouter-view`和`tsrouter-link`组件。方便全局引用组件。

#### 实现TsVRouter构造函数

```typescript
export default class TsVRouter extends TsRouter{
  static install = install;
  mode: RouterMode;
  options: TsRouterOptions;
  maper: Maper;
  history: Html5History | HashHistory;

  constructor(options: TsRouterOptions = { routes: []}) {
    super()
    this.options = options;
    this.mode = options.mode || RouterModeValue.hash;
    this.maper = createMap(options.routes);
    switch(options.mode) {
      case RouterModeValue.hash:
        this.history = new HashHistory(this);
        break;
      case RouterModeValue.html5:
        if(window.onpopstate) { 
          this.history = new Html5History(this);
        } else { // 降级处理
          this.history = new HashHistory(this);
        }
        break;
      default:
        this.history = new HashHistory(this);
        break;    
    }
  }
  push(state: Route, url: any): void {
    this.history.push(state, url)
  }
  replace(state: Route, url: any): void {
    this.history.replace(state, url)
  }
  go(n: number): void {
     this.history.go(n)
  }
}
```

在这里，我们根据`mode`类型，分别将`TsVRouter`的`history`赋值为`Html5History`或`HashHistory`。`TsVRouter`还有一个`maper`属性，用于存储`path`、`name`和`component`之间的映射关系。这样就能根据当前路由的`path`或者`name`来获取相应的`component`

#### 路由对象实现

路由对象分为`Html5History`或`HashHistory`。实现很简单，就是根据前文提到的`history api`和`hash`的实现方法。直接看代码

##### Html5History

```typescript
class Html5History extends BaseHistory {
    constructor(rootRouter: TsRouter) {
        super(rootRouter);
        window.addEventListener('popstate', (event) => { // history.go 才会触发popstate事件
            this.current = event.state                   // 改变当前路由信息
        })
    }
    push(state: Route, url?: any): void {
        history.pushState(state, '', this.getUrl(state, url))
        this.current = state
    }
    replace(state: Route, url?: any): void {
        history.replaceState(state, '', this.getUrl(state, url))
        this.current = state
    }
}
```

##### HashHistory

```typescript
class HashtHistory extends BaseHistory {
    constructor(rootRouter: TsRouter) {
        super(rootRouter);
        window.addEventListener('hashchange', () => {
            const {pathToName} = rootRouter.maper
            const path = window.location.hash.split('#')[1]
            const name = pathToName[path]
            this.current = {
                path,
                name,
                hash: window.location.hash
            }
        })
    }
    push(state: Route, url?: any): void {
        const hash = this.getUrl(state, url)
        window.location.hash = hash || ''
    }
    replace(state: Route, url?: any): void {
        const hash = this.getUrl(state, url)
        const href = window.location.href
        window.location.replace(`${href.split('#')[0]}#${hash}`)
    }
}
```

#### RouterView和RouterLink实现

##### RouterView

在`RouterView`组件中根据映射关系`maper`拿到当前`url`对应的`component`，调用`render`函数进行渲染

```typescript
const RouterView: TsVue.TsComponentOptions =  {
    name: 'RouterView',
    render(this: TsVue.TVue, h: TsVue.TsCreateElement) {
        const maper = this.$tsRouter.maper;
        const {path, name} = this.$tsRoute;
        const {nameMap, pathMap} = maper;
        const component = name && nameMap[name] ? nameMap[name]: (path && pathMap[path] ? pathMap[path]: '')
        return h(component , {}, this.$slots.default)
    }
}
```

##### render

`render`函数是字符串模板的代替方案，可以用`JavaScript `来渲染页面，就像`react`的`jsx`那样。该渲染函数接收一个 `createElement` 方法作为第一个参数用来创建 `VNode`

`CreateElement`在vue里是这么定义的

```typescript
interface CreateElement {
  (tag?: string | Component<any, any, any, any> | AsyncComponent<any, any, any, any> | (() => Component), children?: VNodeChildren): VNode;
  (tag?: string | Component<any, any, any, any> | AsyncComponent<any, any, any, any> | (() => Component), data?: VNodeData, children?: VNodeChildren): VNode;
}
```

可见该方法可以接收 组件或者html标签名，数据对象(如class，attrs等等)，children子属性等。

##### RouterLink

`RouterLink`接收参数`to`，用来表示跳转的页面路径。可以用`name`也可以用`path`。如：`  <tsrouter-link :to="{path: '/foo'}">`或者`  <tsrouter-link :to="{name: 'foo'}">`。当为`name`时，则根据`maper`中的映射关系，拿到对应的`path`。在这里将`RouterLink`渲染为`<a>`标签，同时用`event.preventDefault()`阻止其默认行为，防止页面跳转导致页面刷新，用`history.push`来代替`<a>`标签的跳转。

```typescript
const RouterLink: TsVue.TsComponentOptions =  {
    props: {
        to: {
            type: Object,
            required: true
        }
    },
    name: 'RouterLink',
    render(this: TsVue.TVue, h: TsVue.TsCreateElement) {
        let data = Object.create(null)
        const tsRouter = this.$tsRouter
        const {nameToPath} = tsRouter.maper;
        const {path, name} = this.$props.to;
        const url = name && nameToPath[name] ? nameToPath[name]: path;
        data['attrs'] = {
            href: url
        }
        data['on'] = {
            click: (event: Event) => {
                event.preventDefault();
                tsRouter.push({ path: url, name }, url)
            }
        }
        return h('a', data, this.$slots.default )
    }
}
```

#### 使用TsVRouter

`TsVRouter`功能开发完成，接下来使用它

```js
// main.js
import Vue from 'vue'
import App from './App'
import foo from './foo'
import bar from './bar'
import TsVRouter from 'ts-vrouter'

// 注册路由
Vue.use(TsVRouter)
const tsRouter = new TsVRouter({
  mode: 'html5',  // history api 模式
  routes: [
    {path: '/foo', component: foo, name: 'foo'},
    {path: '/bar', component: bar, name: 'bar'}
  ]
})

new Vue({
  el: '#app',
  render: h => h(App),
  tsRouter
})


<!-- App.vue --> 
<template>
  <div>
    <h2>{{msg}}</h2>
    <tsrouter-link :to="{path: '/foo'}">跳转到 foo</tsrouter-link>
    <tsrouter-link :to="{path: '/bar'}">跳转到 bar</tsrouter-link>
    <tsrouter-view></tsrouter-view>
  </div>
</template>

<script>
export default {
  created() {
  },
  data() {
    return {
      msg: 'i am ts-vrouter demo'
    }
  }
}
</script>

<!-- bar.vue -->
<template>
  <div>
    {{msg}}
    <div @click='go'>turn to foo</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      msg: 'bar page'
    }
  },
  methods: {
    go() {
      this.$tsRouter.push({path:'/foo'});
    }
  }
}
</script>

<!-- foo.vue -->
<template>
  <div>foo pag</div>
</template>

<script>
export default {
  data() {
    return {}
  }
}
</script>

```

在`/bar`路径，点击`go`，成功跳转到`/foo`！



## 小结

`TsVRouter`基本功能已经完成。代码还是有很多需要优化的地方。这次学习是为了了解前端路由的实现原理。后续有时间尝试完善代码，并扩展`TsVRouter`

