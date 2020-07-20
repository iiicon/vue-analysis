## render

`_render` 是 vue 实例的一个私有方法，用来把实例渲染成一个虚拟 Node, 定义在 `src/core/instance/render.js`

在\_render 函数中会执行 `vnode = render.call(vm._renderProxy, vm.$createElement)` 返回一个 VNode

- \_renderProxy 在 prod 环境是 vm，dev 环境会返回一个 proxy、
- \$createElement 是在 initRender 中定义的一个方法，会调用定义的 `createElement`

```flow js
// 对外接口
$createElement: (
  tag?: string | Component,
  data?: Object,
  children?: VNodeChildren
) => VNode;
```

## VNode

`Virtual DOM` 是用来替换真实 DOM 的，减少真实 DOM 昂贵的开销
Vue 定义在 `core/vdom/vnode.js`,

```js
class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void;
  parent: VNode | void;
  // ...
}
```

## update

vue 的 \_update 是一个私有方法，在首次渲染和数据更新的时候会调用他，把 virtual dom 转化成 real dom
定义在 `src/core/instance/lifecycle.js` 中，其中调用的核心就是 `vm.__patch__` 方法，我们在
`src/platforms/web/runtime/patch.js` 中定义了 `patch = createPatchFunction({ nodeOps, modules })`
通过 curry 返回一个函数 patch, 递归 insert 到 body 中，完成真实 dom 的渲染

```js
return function patch(oldVnode, vnode, hydrating, removeOnly) {
  createElm();
};
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // ...
  createChildren(vnode, children, insertedVnodeQueue);
  if (isDef(data)) {
    invokeCreateHooks(vnode, insertedVnodeQueue);
  }
  insert(parentElm, vnode.elm, refElm);
}
```

## 组件化

createComponent

我们在调用 `_render` 函数的时候，会调用 `$createElement` 方法，第一个参数可以接受一个对象`options`，
比如我们`export`的`vue`组件，vue 在内部会调用 `_createElement` 函数，`options`会作为`_createElement`的第二个参数`tag`传入，
如果是 `string` 的直接 new VNode，否则就调用 `createComponent`，在这个函数里 vue 会构造子类构造函数(调用 Vue.extend)，安装组件钩子函数，new VNode

patch
以 new Vue 的时候传入 App 组件为例，`_render`函数调用结束后会返回当前的渲染 VNode，接着就会执行 patch 过程，首先会判读当前 vnode 是不是一个组件 vnode，如果是的话就执行
`createComponent`方法，创建子 vm 实例，这个过程通过`createComponentInstanceForVnode` 创建，保存了占位符 vnode 和父 vm 实例（activeInstance），这个函数会通过
之前保存的 `Ctor` 实例化子 vm，实例化的过程会执行 `this._init`,合并 options 等一系列逻辑，
接着会`vm.$mount`方法，这个过程就是和之前一样先 `_render` 出渲染 vnode（中间用` vm.$vnode=_parentVnode``vnode.parent=_parentVnode `），再`_update`进行 patch，
在这个过程，`vm._vnode=vnode`,而且吧 vm 赋值给全局的 activeInstance,还要保存 parent children 的关系等，现在我们的 vnode（App）就是一个渲染 vnode，根是#app 的 div，
接下来在 patch 的过程中就不会再 `createComponent` 返回，会执行`nodeOps.createElement` 创建一个 dom 元素
往下执行会执行 `createChildren` 递归调用 `createElm` 去 patch，当执行完子组件的 children 子元素 后，就会执行子元素 insert 函数，由于没有 parentElm，所以什么都不会执行，
只是把 dom 保存在了 vnode.elm, `patch` 函数执行完毕后返回 vnode.elm, 到此子组件的挂载就结束了
回到了 `createComponent` 的 init 方法，执行 `initComponent` 方法 `vnode.elm = vnode.componentInstance.$el` 也就是占位符 vnode.elm 保存了
我们创建的 dom 元素，在这里是有 parentElm 的，就是 body，执行插入操作，组件渲染完毕

## 配置合并

通过分析，我们知道 new Vue 的过程有两种场景，一种是我们外部代码主动调用 `new Vue(options)` 实例化一个 Vue 对象，另一种就是组件内部通过
`new Vue(options)` 实例化子组件，无论哪种场景，都会执行 `this._init(options)`，他首先就会执行一个 mergeOptions 的逻辑

```flow js
export function mergeOptions(
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== "production") {
    checkComponents(child);
  }

  if (typeof child === "function") {
    child = child.options;
  }

  normalizeProps(child, vm);
  normalizeInject(child, vm);
  normalizeDirectives(child);
  const extendsFrom = child.extends;
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm);
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm);
    }
  }
  const options = {};
  let key;
  for (key in parent) {
    mergeField(key);
  }
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key);
    }
  }
  function mergeField(key) {
    const strat = strats[key] || defaultStrat;
    options[key] = strat(parent[key], child[key], vm, key);
  }
  return options;
}
```

外部调用 `new Vue` 会把传入的 options 和 Vue 的 options 做一层合并
子组件在实例化的时候会用 `Vue.extend` 生成子类构造器 Sub, Sub 的 options（比如刚开始实例化 render 的 App 组件） 会和 Vue 的 options 做一层合并
`this._init` 会执行 `initInternalCompoennt` 方法，`Object.create(Sub.options)` 赋值给 vm.\$options, 保存`parent和_parentVnode`

关于合并策略函数 `mergeField` 会根据不同的 key 去合并
比如 `data` 会深度合并，`hooks` 会返回数组，`methods props computed inject` 会覆盖等等，具体 key 具体分析

值得一提的是 如果有 mixins 和 extends 属性，则会先和 Vue 进行合并，也就是合并到 Vue 构造器上

## 生命周期

### beforeCreate 和 created

```flow js
Vue.prototype._init = function(options?: Object) {
  // ...
  initLifecycle(vm);
  initEvents(vm);
  initRender(vm);
  callHook(vm, "beforeCreate");
  initInjections(vm); // resolve injections before data/props
  initState(vm); // 初始化 props、data、methods、watch、computed 等属性
  initProvide(vm); // resolve provide after data/props
  callHook(vm, "created");
  // ...
};
```

### beforeMount 和 mounted

beforeMount 是在 dom 挂载之前，他的调用时机是在 mountComponent 函数中，在渲染 vnode 之前
mounted 是分两种情况，`new Vue` 的时候是在 `mountComponent` 函数中

```flow js
if (vm.$vnode == null) {
  vm._isMounted = true;
  callHook(vm, "mounted");
}
```

子组件是在 patch 的时候调用 `invokeInsertHook` 函数

```flow js
function invokeInsertHook(vnode, queue, initial) {
  // delay insert hooks for component root nodes, invoke them after the
  // element is really inserted
  if (isTrue(initial) && isDef(vnode.parent)) {
    vnode.parent.data.pendingInsert = queue;
  } else {
    for (let i = 0; i < queue.length; ++i) {
      queue[i].data.hook.insert(queue[i]);
    }
  }
}
```

挨个执行子组件 `vnode.data.hook.insert()`，这个函数执行 `callHook('mounted')`

### beforeUpdate 和 updated

`beforeUpdate` 是作为 new Watcher 的参数传入的，在 `scheduler.js` 的 `flushSchedulerQueue`
函数中，有 before 就会调用 `watcher.before()`

```flow js
function flushSchedulerQueue() {
  // ...
  if (watcher.before) {
    watcher.before();
  }
  // ...
}
```

`updated` 钩子是在更新 `watchers` 的时候执行

```flow js
function callUpdatedHooks(queue) {
  let i = queue.length;
  while (i--) {
    const watcher = queue[i];
    const vm = watcher.vm;
    if (vm._watcher === watcher && vm._isMounted && !vm._isDestroyed) {
      callHook(vm, "updated");
    }
  }
}
```

### beforeDestroy 和 destroyed

他们都是在 `$destroy` 函数中执行
`beforeDestroy` 在函数刚开始调用
执行完 `vm.__patch__(vm._vnode, null)` 就会调用 `destroyed` 钩子函数，然后卸载事件等一系列逻辑

### activated 和 deactivated

与 keep-alive 相关

## 组件注册

### 全局注册

使用 `Vue.component('my-component', options)` 注册，
通过 `Vue.extend(options)` 返回一个组件的 `'my-component'` 构造器 X，
`X.options = mergeOptions(Super.options,extendOptions)`，
通过`this.options[type + 's'][id] = definition` 挂载到 `Vue.options.components` 上，

在 `new Vue` 执行 `this._init` 的时候会执行 `mergeOptions`, 把 `Vue.options options` 合并到 `vm.$options` 上,
在 `_creatElement` `resovleAssets` 的时候，就能在 `components.__proto__` 上找到 `my-component` 返回构造器，
接着在 `createComponent` 的时候返回对应 vnode

### 局部注册

在创建组件 Ctor 执行`Vue.extend` 的时候 `mergeOptions(Super.options, extendOptions)` 把 Vue.options 和组件 options 合并到 Sub.options
到子组件 `_init` 的时候，会执行 `initInternalComponent` 不会走`mergeOptions` 的逻辑，
通过 `vm.$options = Object.create(Sub.options)` 扩展到子 vm 原型上，
在 `_creatElement` `resovleAssets` 的时候，就能在 `options.components` 上获取到当前 component 和局部注册的 component，返回相应的构造器
接着在 `createComponent` 的时候返回对应 vnode

## 异步组件

### 工厂函数

```flow js
Vue.component("async-example", function(resolve, reject) {
  // 这个特殊的 require 语法告诉 webpack
  // 自动将编译后的代码分割成不同的块，
  // 这些块将通过 Ajax 请求自动下载。
  require(["./my-async-component"], resolve);
});
```

我们在调用 `_createElement` 执行 `createComponent` 的时候，如果 Ctor 没有 cid，就会执行
`resolveAsyncComponent(factory, baseCtor)` 生成对应的组件构造器
resolveAsyncComponent 函数刚开始的时候，执行`const res = factory(resolve, reject)`
并返回 undefined，会生成 `CommetVnode`，进而 render patch 成一个注释节点

当 factory 执行，获取到异步组件执行回调调用 `resolve` 时，会把生成的构造器赋值给`factory.resolved`,
同时执行 `forceRender(true)`,`$forceRender` 会执行 `_watcher` 的 get 方法，进而执行 `_render` 再次调用 `resolveAsyncComponent`
这次进来就有 `factory.resolve`, 他就是我们返回的组件构造器

### promise

```flow js
// 动态 import 返回 promise
Vue.component("async-example", () => import("./my-async-component"));
```

webpack 2+ 支持了异步加载的语法糖：`() => import('./my-async-component')`，会在打包的时候单独打包成一个文件
promise 的用法就和工厂函数基本类似，我们在调用 `resolveAsyncComponent(factory, baseCtor)` 的时候
如果 factory 返回的 res 是 promise，我们就会增加 then 方法 `res.then(resolve, reject)`，这样当文件加载完成就会
调用 `resolve` 方法，剩下的逻辑就和工厂函数一样

### 高级异步组件

通过配置，可以支持一些 laoding 和 error

```flow js
const AsyncComp = () => ({
  // 需要加载的组件。应当是一个 Promise
  component: import("./MyComp.vue"),
  // 加载中应当渲染的组件
  loading: LoadingComp,
  // 出错时渲染的组件
  error: ErrorComp,
  // 渲染加载中组件前的等待时间。默认：200ms。
  delay: 200,
  // 最长等待时间。超出此时间则渲染错误组件。默认：Infinity
  timeout: 3000
});
Vue.component("async-example", AsyncComp);
```

## 响应式对象

Vue 的响应式对象就是给对象添加一个 `enumbealbe = false` 的 `__ob__` 属性对象

- `initState` 的时候执行 `initProps initData initWatch initMethods initComputed`
- `initData` 做两件事，通过 `proxy` 方法，把 `vm._data` 上的属性代理到 `vm` 上, 另一个就是用
  `observe` 方法观察整个 data 的变化，把 data 变成响应式的，添加 `__ob__` 属性
- `initProps` 也是做两件事，通过 `defineReactive` 把 `_props` 上的属性值变成响应式， 通过 `vm._props.xxx`
  访问到定义 props 中对应的属性, 另一个通过 `proxy` 方法，把 `vm._props` 上的属性代理到 `vm` 上

> `proxy` 代理的作用是把 props 和 data 上的属性代理到 vm 实例上
> `observe` 函数的功能就是用来监测数据的变化
> `Observer` 是一个类，它的作用是给对象的属性添加 getter 和 setter，用于依赖收集和派发更新

## 依赖收集

依赖收集就是订阅数据变化 Watcher 的收集

响应式对象 getter 相关的逻辑就是做依赖收集，Dep 是整个依赖收集的核心，`Dep.target` 是全局唯一
的 Watcher，也就是同一时间只有一个 watcher 被计算
Dep 实际上是对 watcher 的一种管理，watcher 中有四个属性和当前的 dep 有关

```flow js
this.deps = [];
this.newDeps = [];
this.depIds = new Set();
this.newDepIds = new Set();
```

我们在实例化渲染 watcher 的时候，会执行 get 方法，首先会执行 `pushTarget(this)` ,接着执行 `updateComponent`
这个方法会执行 `vm._render` 过程中会访问 vm 上的数据，进而触发对应数据的 getter 函数
执行 getter 的过程中会在每一个数据对应的 dep 上，执行 `dep.depend()`,他会执行 `Dep.target.addDep(this)`
就会给 `newDeps` 和 `newDepIds` 添加当前的 dep，然后把当前的 watcher 订阅到这个数据持有的 dep.subs 中
到此为止当前 vm 的依赖收集已经结束了
接下来执行 `popTarget`，把当前的 Dep.target 恢复成上一个状态
接着会执行清空依赖的过程，deps 是保存的上一个相关的 dep，在第二次进行 cleanupDeps 的时候，会把 newDeps 中没有的从
dep.subs 取消订阅，这样即使数据发生了改变，也不会触发重新渲染，算是 vue 一个比较重要的优化

## 派发更新

派发更新就是当数据发生改变后，通知所有订阅了这个数据变化的 watcher 执行 update

我们收集依赖的目的就是可以在修改数据的时候，可以对相关的依赖派发更新
当我们在组件中对响应式的数据做了修改，就会 setter 的逻辑，最后会调用 `dep.notify()`,
notify 中会遍历 dep.subs 中的 watcher，并执行 `sub[i].update()`，
update 中会执行 `queueWatcher(this)` 把每次数据改变添加到一个队列里，
然后在`nextTick` 执行 `flushSchedulerQueue`，挨个执行 `watcher.run`
在这里会执行 `watcher` 的 `this.get()` 也就是 `updateComponent` 函数，并执行传入的回调函数 `this.cb`
所以这就是当我们去修改组件相关的响应式数据的时候，会触发组件重新渲染的原因，接着就会重新执行 patch 的过程，
但它和首次渲染有所不同

## nextTick

> Vue.nextTick Vue.prototype.\$nextTick

> 数据的变化到 DOM 的重新渲染是一个异步过程，发生在下一个 tick

next-tick 中定义了 `timerFunc`, 用来异步执行 `flushCallbacks`
首先判断支持 promise，用 promise.then
接着判断支持 `MutationObserver` , 用 `new MutationObserver(flushCallbacks)`
接着如果支持 `setImmediate` 用 `setImmediate`
最后都不支持就用 `setTimeout`

nextTick 函数接受 cb 函数 或者 ctx promise 对象两种参数
函数执行会把 cb 包在一个匿名函数中，push 到 callbacks 中，在下一个 tick 执行这些函数
如果是 promise 对象，就有一个局部的变量 \_resolve 保存 promise.resolve 函数，执行 \_resolve
就会跳到 then 的逻辑中，在下个 tick 执行

## 检测变化的注意事项

对于使用 `Object.defineProperty` 实现响应式的对象，当我们去给这个对象添加一个新的属性的时候，
是不能触发它的 setter 的

Vue.set 定义在 `observer/index` 中，接受三个参数，target 是对象或者数组，key 是对象 key 或者数组 index
val 是添加的值，如果是数组，就直接 splice 插入，接下来判断对象，如果已经存在于 target 说明已经是响应式了，返回值就可以
否则就拿到 target 的 ob 对象，如果 ob 不存在说明不是响应式对象，普通对象直接设置值就可以了，否则就调用 `defineReactive`
把 target 上面对应 key 变为响应式的，接着会执行 `ob.dep.notify` 派发更新，在 `defineReactive` 中因为是 childOb 会触发，
所以收集依赖就会到对应 Observer 所持有的 dep 中，这个时候执行 `dep.notify` 就会触发对应更新
而数组则是重新实现的部分 api, 会先调用原生 api，然后把插入的数组的新值 inserted 拿出来，通过 `ob.observeArray` 变成响应式对象，
然后手动 `ob.dep.notify()` 派发更新

## computed watcher

计算属性的初始化是发生在在 Vue 实例初始化阶段的 `initState` 中，在 `initComputed` 中，
用 `vm._computedWatchers` 来保存自定义的 computed 对象值，在对 computed 对象遍历的过程中，
每一个 `userDef` 都赋值为一个 `getter`，并实例化一个 `computed watcher`,
接着用 `defineComputed` 把每一个 key 都设置 getter 和 setter，getter 就是 `createComputedGetter`
执行返回的 `computedGetter` 函数,
这个函数在返回对应属性会执行 `watcher.evaluate` 进而执行 `watcher.get`
执行完 `watcher.get` 后，当前的 `Dep.target` 就是这个 `computed watcher`,也就是访问 data 的上数据的时候把依赖收集到这个 watcher 上
然后执行`watcher.depend` 遍历当前的 deps（上一个 deps，依赖的属性），然后执行`dep.depend` 收集依赖，一旦有了变化就走 setter 逻辑

## user watcher

侦听属性的本质是 user watcher, 有 user deep sync 等属性
在 `initState` 的时候实例化 `Watcher`，实例化的时候会执行设置 `user = true`,执行 get `pushTarget`,
执行 `getter` 获取 `value`， 并收集依赖，如果 `deep` 为 `true`，则调用 `traverse` 方法，递归访问属性收集依赖，
实例化结束后悔判断 `immediate` 如果 true，直接调用回调函数，也就是我们传过来的侦听函数
最后返回一个 `unwatchFn` 函数
在 set 的时候，触发对应值的 `dep.notify`, 执行对应 `watcher.update` push 到 `queneWathcer` 在 nextTick 执行 `run`,
如果 sync 为 true 的话，则不会 push，会在 `update` 的时候立即执行 `watcher.run`,

## 组件更新

当我们数据更新的时候，会通知 render watcher 更新，进而触发 updateComponent，他会调用 patch 方法
更新的逻辑和首次渲染的逻辑是不一样的，因为两次 vnode 都存在，要根据 sameVnode(oldVnode, vnode)
判断他们是否是相同的 vNode 来决定走不同的更新逻辑 （首先判断 key 是否相同，然后对于同步组件判断 isComment data input,
对于异步组件判断 asyncFactory 是否相同）
**如果新旧节点不同**，那么更新的逻辑非常简单，他本质上是要替换已经存在的节点，分为三步

1. 创建新节点，通过 createElm 方法，创建新的节点并插入到 DOM 中
2. 更新父的占位符节点，把 elm 更新为当前新的 elm，并执行一系列钩子
3. 删除旧节点，把 oldVnode 从当前 dom 树中删除，如果父节点存在，执行 removeVnodes 方法

**新旧节点相同**，如果 sameVnode 返回 true，就会调用 patchVnode 方法，把新的 vnode patch 到旧的 vnode 上，主要分为四步

1. 执行 prePatch 钩子函数，拿到新的组件配置和组件实例，去执行 updateChildComponent 方法，
   updateChildComponent 的逻辑也非常简单，由于更新了 vnode，那么 vnode 对应的实例 vm 的一系列属性也会
   发生变化，包括占位符 vm.\$vnode 的更新，slot 的更新，listeners 的更新，props 更新等等
2. 执行 update 钩子函数，执行 module 的 update 钩子函数和用户自定义的钩子函数
3. 完成 patch 过程，如果 vnode 是文本节点且新旧文本不同，则直接替换文本内容，如果不是文本节点，则判断他们的子
   节点，并分为了几种情况：oldCh 和 ch 都存在且不相同时，使用 updateChildren 函数来更新子节点，如果只有 ch 存在
   表示旧节点不要了。如果旧的是文本节点先将旧的移除，然后通过 addVnodes 将 ch 批量插入到新节点 elm 下面
   如果只有 oldch 存在，表示更新的是空节点，用 removeVnodes 全部移除节点，只有当旧节点是文本节点的时候，则清楚文本内容
4. 执行 postPatch 钩子函数

组件更新的过程核心就是新旧 vnode diff，对新旧节点相同以及不同的情况分别做不同的处理。新旧节点不同的更新流程是
创建新节点->更新父占位符节点->删除旧节点；而新旧节点相同的更新流程是去获取他们的 children，根据不同情况做不同的
更新逻辑，最复杂的情况是新旧节点相同且他们都存在子节点，那么会执行 updateChildren 逻辑

## props

### props 初始化与规范化

在初始话 props 之前，首先会对 props 做一次 normalize，它发生在 mergeOptions 的时候
normalizeProps 就是把数组格式化为 `name: { type: null }` 对象格式化为 `name: { type: String }`
props 的初始化主要发生在 `_init` 阶段的 initState，其中 `initProps` 函数主要做三件事情：校验、响应式和代理

1. 校验就是检查一下父组件传过来的 `prop` 数据是否满足我们 `prop` 的定义规范,主要就是执行 `validateProp`, 主要做 3 件事情，处理 Boolean 类型数据
   处理默认数据，prop 断言，并返回最终的 `prop` 值
2. 响应式，回到 `initProps` 方法，当我们通过 `const value = validateProp(key, propsOptions, propsData, vm)`对 prop 做
   验证并且获取到 `prop` 值后，接下来通过 `defineReactive` 把 `prop` 变成响应式, 需要注意的是在开发环境我们会
   传一个 customSetter，当我们对 prop 赋值的时候会触发输出警告
3. 在经过响应式处理后，我们会把 prop 的值添加到 vm.\_props 中，比如 key 为 name 的 prop，它的值保存在 `vm._props.name` 中
   但是我们在组件中可以通过 this.name 访问到这个 prop,这就是代理做的事情。
   其实对于非根实例的子组件而言， prop 的代理发生在 Vue.extend 阶段，执行 `proxy(Comp.prototype,`\_props`, key)`,
   把 props 代理到原型上，这样不用为每个组件实例都做一层 proxy，是一种优化手段。

### props 更新

当父组件中 `props` 变化，触发父组件的重新渲染，会执行 `patch`，进而执行 `patchVnode` 函数，`patchVnode` 通常是一个
递归过程，当他遇到组件 `vnode` 的时候会执行，会执行组件更新过程的 `prepatch` 钩子函数，内部会调用 `updateChildComponents`
来更新 `props`，第二个参数就是父组件的 `propsData`，因为在组件的 `render` 过程中，对于组件节点会通过 `createComponent`
方法来创建组件 `vnode`，在创建组件 `vnode` 的过程中，首先从 `data` 中提取出 `propData`, 然后在 `new VNode` 的时候作为第七个参数
`VNodeComponentOptions` 中的属性传入，所以我们可以通过`vnode.componentOptions.propsData` 拿到 prop 数据,
然后通过 `updateChildComponent` 更新 `props`，而且保存 `propsData` `vm.$options.propsData = propsData`

### toggleObserving

```flow js
export let shouldObserve: boolean = true;

export function toggleObserving(value: boolean) {
  shouldObserve = value;
}
```

在当前模块中定义 shouldObserve 变量，用来控制 observe 过程中是否需要把当前值变成一个 observe 对象，
那么为什么在 props 的初始化和更新过程中，多次执行 toggleObserving(false) 呢？我们分几种情况来分析

- 在 initProps 的时候：
  对于非实例的情况，我们会执行 `toggleObserving(false)`, 然后对于每一个 `prop` 值，去执行 `defineReactive(props, key, value)`
  去把它编程响应式，由于 `shouldObserve` 的值变成了 false，这个递归过程就被省略了，为什么会这样呢？
  因为对于对象的 `porp` 值，子组件的 `prop` 始终指向父组件的 `prop` 值，只要父组件的 `prop` 值变化，
  就会触发子组件的重新渲染，所以这个 `observe` 是可以省略的。最后恢复为 `true` 就好
- 在 validateProp 的时候：
  因为这里的逻辑是处理默认值的，而默认值是一个拷贝，所以需要把他变成响应式的，设为 `true`
- 在 updateChildComponent 的时候：
  不需要递归处理引用类型的 `props`，所以也需要设为 `false`
  
 ## 编译入口
 
 在执行 $mount 的时候，会有一系列的判断逻辑，
 如果没有 render 就会执行 `compileToFunction(template, options, vm) `去创建 render 函数
`compileToFunction` 是由 `createCompiler(baseOptions)` 生成的, 
而 `createCompiler` 则是由 `createCompilerCreator(function baseCompile(template, options){})` 执行后的返回值，
`createCompilerCreator(baseCompile: Function): Function {}` 定义在 `create-compiler` 中，
这个函数返回 createCompiler 函数，这里就是 createCompiler 函数的定义，它接受一个 baseOptions 的参数，
返回一个对象，包括 `compile` 方法和 `compileToFunctions` 属性，`compileToFunctions` 是由 `createCompileToFunctionsFn` 执行返回，
这里返回的函数就是 `compileToFunctions` 的定义，核心代码就是 `const compiled = compile(template, options)` 执行编译过程, 
`compile` 函数定义在 `createCompilerCreator` 函数中，就是先处理配置参数，再执行编译 `const compiled = baseCompile(template, finalOptions)`
`baseCompile` 在执行 `createCompilerCreator(baseCompile)` 中作为参数传入，这里是就是编译的入口，主要执行如下的逻辑
- 解析模板字符串生成AST `const ast = parse(template.trim(), options)`
- 优化语法树 `optimize(ast, options)`
- 生成代码 `const code = generate(ast, options)`

 

## 问题

- vm 实例加载 render 方法的时机
- watch 和 method 中修改相同属性
- **vue 组件需要一个根**
- 组件 vnode 有 data
- 子组件的 render 挂载的时机
- vm 实例加载 render 方法的时机
- 递归 patch insert 执行 insertHook
- render 和 lifecycle 顺序``
  先执行 initLifecycle 会执行\$mount,进而执行\_render
- 子组件 `quene` `insert` 和父组件
  顺序(只有他是 `componentvnode` 才会 push 到 `quene`) 渲染 node 和组件 vnode 就是
  同一个组件
- 各组件 vnode 之间的关系
- 有了 `resolveAsset` 子组件的 `Vue.extend` 就走不到了？
- defineReactive 中 dep 的层级问题
- 所以每次数据变化都会重新 render ？
- 每次都去实例化 watcher
- 响应式的数据有哪些
- 全局下的 dep 收集是过程最后的 Dep.target 指向
- userWatcher 的创建时机
- watcher 的时候渲染父组件会不会影响子组件
- set 执行时初始化的 watcher
- watcher 触发 render 的时机
- createElm 的时候 ancestor.elm = vnode.elm // ??????????????????????????? 多个 vnode 对应一个 element?
- invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch) // \*\*？？？？？
- 什么样的 vnode 有 key
- **看一下 updateChildren**
