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
如果是 `string` 的直接new VNode，否则就调用 `createComponent`，在这个函数里 vue会构造子类构造函数(调用Vue.extend)，安装组件钩子函数，new VNode


##

-  vm 实例加载 render 方法的时机
