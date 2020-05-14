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
往下执行会执行 `createChildren` 递归调用 `createElm` 去 patch，当执行完子组件的 children子元素 后，就会执行子元素 insert 函数，由于没有parentElm，所以什么都不会执行，
只是把 dom 保存在了 vnode.elm, `patch` 函数执行完毕后返回 vnode.elm, 到此子组件的挂载就结束了
回到了 `createComponent` 的init方法，执行 `initComponent` 方法 `vnode.elm = vnode.componentInstance.$el` 也就是占位符vnode.elm 保存了
我们创建的dom元素，在这里是有parentElm的，就是body，执行插入操作，组件渲染完毕

## 问题

- 组件 vnode 有 data
- 子组件的 render 挂载的时机
