## vue响应式原理

Vue2 通过 Object.defineProperty 对数据进行劫持，在 getter 中进行依赖收集，在 setter 中派发更新。每个属性都会对应一个 Dep，用来管理依赖它的 Watcher。当组件渲染时会创建 Watcher 并收集依赖，当数据变化时 Dep 会通知所有 Watcher 进行更新。Vue 为了性能使用异步更新队列和 nextTick 合并多次更新。

Vue3 使用 Proxy 重写响应式系统，通过 track 收集依赖，通过 trigger 触发更新，依赖关系使用 WeakMap + Map + Set 存储，并通过 effect 重新执行副作用函数。

## vue2和vue3的区别

- **API 风格**
  Vue2 使用的是选项式 API（Options API），数据必须放在 `data` 中，方法必须放在 `methods` 中。
  Vue3 引入了组合式 API（Composition API），数据和方法可以放在一起，逻辑复用性更强，代码组织更加灵活。
- **响应式系统**
  Vue2 使用 `Object.defineProperty` 实现响应式，存在一些局限，比如无法监听对象新增属性或数组索引变化。
  Vue3 使用 ES6 的 `Proxy` 实现响应式，性能更优，并且可以监听更多类型的数据变化，但是只能监听复杂数据类型。
- **生命周期**
  Vue3 提供 `setup` 函数，可以代替 Vue2 中的 `created` 和 `beforeCreate` 生命周期函数，逻辑初始化更集中、更清晰。
- **TypeScript 支持**
  Vue3 对代码进行了重写，更好地支持 TypeScript，Props、事件、响应式数据都有类型推导，IDE 提示更完善。
- **代码体积和性能**
  Vue3 核心包体积更小，性能更好。初始化和渲染速度相比 Vue2 更快，内存占用更少。
- **模板根节点**
  Vue2 的模板只能有一个根节点，而 Vue3 的模板可以有多个根节点（Fragment），减少不必要的 DOM 包装。

## ref和reactive的区别

ref 和 reactive 都是 Vue3 用来创建响应式数据的 API。
 ref 通常用于基本类型数据，它会返回一个 RefImpl 对象，通过 `.value` 访问数据，本质是通过 getter 和 setter 实现依赖收集和派发更新。
 reactive 只能用于对象类型，它基于 Proxy 对整个对象做代理，在 get 时进行依赖收集，在 set 时触发更新。

在使用上 ref 需要 `.value`，而 reactive 可以直接访问属性。

另外 reactive 在解构时会丢失响应式，需要使用 toRefs 处理，而 ref 不会有这个问题。

因此在组合式 API 中，Vue 官方更推荐优先使用 ref，reactive 适用于复杂对象状态管理。



**为什么reactive解构会丢失响应式**

reactive 是基于 Proxy 实现的响应式，依赖收集发生在 Proxy 的 get 拦截中。当使用解构语法时，本质是执行 `const count = state.count`，这一步会直接把值赋给一个新的变量，而这个变量不再经过 Proxy 代理。因此后续 state.count 发生变化时，count 变量不会更新，从而丢失响应式。

Vue 提供了 `toRefs` 方法把 reactive 对象的每个属性转成 ref，从而保持响应式。

## v-if和v-show的区别

| 特性         | v-if                                   | v-show                                                 |
| ------------ | -------------------------------------- | ------------------------------------------------------ |
| **渲染方式** | 条件为 false 时，元素 **不渲染到 DOM** | 元素始终渲染到 DOM，通过 `display: none` 控制显示/隐藏 |
| **适用场景** | 条件切换 **不频繁**                    | 条件切换 **频繁**                                      |
| **开销**     | 条件切换时 **有销毁和重建 DOM** 的开销 | 条件切换只修改 **CSS 显示/隐藏**，开销小               |
| **优点**     | 节省 DOM 资源                          | 切换速度快，无需重复渲染                               |
| **缺点**     | 切换频繁会有性能损耗                   | DOM 一直存在，占用资源                                 |

## 虚拟dom实现的原理

**为什么**？

操作真实dom繁琐，不利于开发维护。

使用虚拟dom，将数据和视图分离，开发者可以更好的关注数据。



**概念**

虚拟 DOM 是用 JavaScript 对象来描述真实 DOM 结构的一种抽象层。框架在初次渲染时会根据 render 函数生成虚拟 DOM 树，再通过 mount 过程创建真实 DOM。当组件状态发生变化时，会重新生成新的虚拟 DOM，然后通过 diff 算法对比新旧虚拟 DOM 树，找出最小的差异，最后通过 patch 阶段只更新变化的真实 DOM。

为了提高 diff 性能，Vue 和 React 采用了几个优化策略，例如只比较同级节点、不同类型节点直接替换、使用 key 来优化列表 diff。Vue3 还使用了最长递增子序列算法减少 DOM 移动，并在编译阶段通过 PatchFlag 和 Block Tree 标记动态节点，从而减少运行时的 diff 开销。



## vue异步更新的过程

1. 触发响应式监听，收集依赖数据，所有要触发的effect函数放入更新队列中，等待本轮代码结束之后统一进行更新。
2. 更新队列会放入微任务，使用vue内部的nextTick进行回调的更新，优先使用Promise.then 放入微任务中。降级使用MutationObserver、setImmediate、setTimeout
3. 等待dom更新完毕，执行用户的nexttick回调。
4. 更新完成数据，触发rende函数，生成虚拟dm，diff算法比较，更新真实dom。



## vue组件缓存的方法

使用keep-alive标签包裹动态组件或者路由视图

## vue组件缓存的生命周期

当一个组件被 `<keep-alive>` 包裹且是**第一次**加载时，它的生命周期钩子触发顺序如下：

1. `beforeCreate`
2. `created`
3. `beforeMount`
4. `mounted`
5. **`activated`** (这是缓存组件特有的，在挂载后立即触发)

> **注意：** 此时组件已经进入了缓存。当你切走再回来时，前四个钩子（创建和挂载）都**不会**再触发，只会触发 `activated`。



缓存组件有两个专属的生命周期钩子：

- **`activated` (激活)**：
  - 触发时机：组件被看到时（初始化完成或从缓存中恢复）。
  - 用途：重新获取数据、启动定时器、初始化图表。
- **`deactivated` (停用)**：
  - 触发时机：组件被切走（隐藏）但未销毁时。
  - 用途：停止定时器、取消事件监听、记录滚动位置。

## 多层路由缓存方案

在多级嵌套路由中，每一层 `router-view` 都必须被 `<keep-alive>` 包裹，缓存链条才能打通

**缓存失效（最常见）**：如果中间层的 `router-view` 没有写 `<keep-alive>`，那么底层子组件即便设置了缓存也会失效。

针对深层嵌套，业界常用 **“路由扁平化”** 方案：在路由守卫或 Vuex 中动态维护一个 `cachedViews` 数组，将多级路由渲染到同一层级的 `router-view` 中。
