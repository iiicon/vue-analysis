## render

`_render` 是 vue实例的一个私有方法，用来把实例渲染成一个虚拟Node, 定义在 `src/core/instance/render.js`

在_render函数中会执行 `vnode = render.call(vm._renderProxy, vm.$createElement)` 返回一个 VNode

- _renderProxy 在 prod 环境是 vm，dev环境会返回一个 proxy、
- $createElement 是在 initRender 中定义的一个方法，会调用定义的 `createElement`

```flow js
// 对外接口
$createElement: (tag?: string | Component, data?: Object, children?: VNodeChildren) => VNode;
```

## VNode

`Virtual DOM` 是用来替换真实DOM的，减少真实DOM昂贵的开销
Vue 定义在 `core/vdom/vnode.js`, 

```js
class VNode {
  tag: string | void
  data: VNodeData | void
  children: ?Array<VNode>
  text: string| void
  elm: Node | void
  ns: string | void
  context: Component | void
  parent: VNode | void
  // ...                    
}
```
