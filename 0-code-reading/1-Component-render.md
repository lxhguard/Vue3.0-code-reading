# 1-Vue3.0组件如何渲染的？

## 1. 组件渲染的入口

```js
//  Vue.js 3.0 初始化一个应用
import { createApp } from 'vue'
import App from './app'
const app = createApp(App) // app: { mount:()=>{}等很多生命周期 }
app.mount('#app')
·
```

## 2. createApp()

函数代码最终位于：```packages/runtime-dom/src/index.ts```的**第53行**的```createApp```。

作用：

- 创建 app 对象
- 重写 app.mount

> 重写是为了支持跨平台渲染

```ts
export const createApp = ((...args) => {
  const app = ensureRenderer().createApp(...args)

  if (__DEV__) {
    injectNativeTagCheck(app)
  }

  const { mount } = app
  app.mount = (containerOrSelector: Element | string): any => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
        const component = app._component
    if (!isFunction(component) && !component.render && !component.template) {
      component.template = container.innerHTML
    }
    // clear content before mounting
    container.innerHTML = ''
    const proxy = mount(container)
    container.removeAttribute('v-cloak')
    container.setAttribute('data-v-app', '')
    return proxy
  }

  return app
}) as CreateAppFunction<Element>
```

## 3. ensureRenderer()创建渲染器对象

ensureRenderer()函数代码位于：```packages/runtime-dom/src/index.ts```的**第32行**的```ensureRenderer()```。

```ts
import { createRenderer } from '@vue/runtime-core'
let renderer: Renderer<Element> | HydrationRenderer
function ensureRenderer() {
    // 【闭包】延时创建渲染器
    // 优点：当用户仅依赖Vue3.0的某个特性时(不包含渲染器)，不创建渲染器，可通过tree-shaking移除渲染器。
  return renderer || (renderer = createRenderer<Node, Element>(rendererOptions))
}
export const render = ((...args) => {
  ensureRenderer().render(...args)
}) as RootRenderFunction<Element>
```

createRenderer()函数代码位于：```packages/runtime-core/index.ts```的**第32行**的```createRenderer()```。

```ts
export { createRenderer, createHydrationRenderer } from './renderer'
```

继续查阅```./renderer```, createRenderer()代码和baseCreateRenderer()代码如下：

> createRenderer():第395行
> baseCreateRenderer():第424行

```ts
// 【函数柯里化】
export function createRenderer<
  HostNode = RendererNode,
  HostElement = RendererElement
>(options: RendererOptions<HostNode, HostElement>) {
  return baseCreateRenderer<HostNode, HostElement>(options)
}
// 简化版baseCreateRenderer
function baseCreateRenderer(options) {
  function render(vnode, container) {
    // 组件渲染的核心逻辑
  }
  return {
    render,
    createApp: createAppAPI(render)
  }
}
```

createAppAPI()函数代码位于：```packages/runtime-core/src/apiCreateApp.ts```的**第123行**的```createAppAPI()```。

```ts
// 简化版createAppAPI
function createAppAPI(render) {
  // createApp createApp 方法接受的两个参数：根组件的对象和 prop
  return function createApp(rootComponent, rootProps = null) {
    const app = {
      _component: rootComponent,
      _props: rootProps,
      mount(rootContainer) {
        // 创建根组件的 vnode
        const vnode = createVNode(rootComponent, rootProps)
        // 利用渲染器渲染 vnode
        render(vnode, rootContainer)
        app._container = rootContainer
        return vnode.component.proxy
      }
    }
    return app
  }
}
```

## 4. app.mount()对比

app.mount 可支持 **跨平台渲染**，步骤为：**创建vnode, 渲染vnode**。

原有的：

```ts
mount(rootContainer: HostElement, isHydrate?: boolean): any {
    if (!isMounted) {
        const vnode = createVNode(
            rootComponent as ConcreteComponent,
            rootProps
        )
        // store app context on the root VNode.
        // this will be set on the root instance on initial mount.
        vnode.appContext = context

        // HMR root reload
        if (__DEV__) {
            context.reload = () => {
                render(cloneVNode(vnode), rootContainer)
            }
        }

        if (isHydrate && hydrate) {
            hydrate(vnode as VNode<Node, Element>, rootContainer as any)
        } else {
            render(vnode, rootContainer)
        }
        isMounted = true
        app._container = rootContainer
        // for devtools and telemetry
        ;(rootContainer as any).__vue_app__ = app

        if (__DEV__ || __FEATURE_PROD_DEVTOOLS__) {
            devtoolsInitApp(app, version)
        }

        return vnode.component!.proxy
    } else if (__DEV__) {
        warn(
            `App has already been mounted.\n` +
            `If you want to remount the same app, move your app creation logic ` +
            `into a factory function and create fresh app instances for each ` +
            `mount - e.g. \`const createMyApp = () => createApp(App)\``
        )
    }
},
```

重写的：

```ts
app.mount = (containerOrSelector: Element | string): any => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
        const component = app._component
    if (!isFunction(component) && !component.render && !component.template) {
      component.template = container.innerHTML
    }
    // clear content before mounting
    container.innerHTML = ''
    const proxy = mount(container)
    container.removeAttribute('v-cloak')
    container.setAttribute('data-v-app', '')
    return proxy
}
```

## 5. createVNode()创建vnode

