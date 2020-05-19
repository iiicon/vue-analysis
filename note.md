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

## 问题

- vm 实例加载 render 方法的时机
- watch 和 method 中修改相同属性
- **vue 组件需要一个根**
- 组件 vnode 有 data
- 子组件的 render 挂载的时机
- vm 实例加载 render 方法的时机
- 递归 patch insert 执行 insertHook
- render 和 lifecycle 顺序
  先执行 initLifecycle 会执行\$mount,进而执行\_render
- 子组件 `quene` `insert` 和父组件
  顺序(只有他是 `componentvnode` 才会 push 到 `quene`) 渲染 node 和组件 vnode 就是
  同一个组件
