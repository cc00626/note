### Redux 的三大原则是什么

**单一数据源**

整个应用的 `state`（状态）被储存在一棵 object tree（对象树）中，并且这个 object tree 只存在于唯一一个 `store` 中。

**状态是只读的**

唯一改变 `state` 的方法就是触发 `action`，`action` 是一个用于描述已发生事件的普通对象。

**使用纯函数进行修改**

为了描述 `action` 如何改变 `state` tree，你需要编写 `reducers`。Reducer 必须是纯函数。相同的输入总会得到相同的输出

### Redux 的数据流是怎样的

**第一步：UI 触发事件 (The Trigger)** 用户在页面上点击了一个“提交”按钮，或者网络请求返回了数据。View 层捕获到这个事件。

**第二步：派发 Action (Dispatching)** View 层不能直接去改数据，它必须调用 `store.dispatch(action)`。



**第三步：Store 调用 Reducer (Processing)** Store 收到 Action 后，会自动把**当前的 State** 和这个 **Action** 一起传给 Reducer 函数。

- *话术加分：* “因为 Reducer 是纯函数，它绝对不会在原有的 state 上直接修改，而是通过深拷贝或浅拷贝（通常推荐结合解构赋值 `...state`），生成并返回一个新的 state 对象。”

**第四步：根 Reducer 合并状态 (Combining)** 在实际的大型项目中，我们会用 `combineReducers` 把多个拆分的子 Reducer 合并成一个根 Reducer。当 Action 派发时，所有的子 Reducer 都会被执行一遍，它们各自判断是否需要处理这个 Action，最后返回自己负责的那部分新状态，组合成一棵新的全局 State 树。

**第五步：Store 更新并通知 View (Updating & Subscribing)** Store 将 Reducer 返回的新 State 保存下来，替换掉旧的 State。接着，Store 会触发所有通过 `store.subscribe()` 注册过的监听函数。React-Redux 这样的绑定库会感知到数据的变化（通过浅比较对象引用），并触发对应组件的重新渲染 (Re-render)。



### 为什么使用redux，不使用window？

**可预测性与数据的“可追踪性”**

**`window` 是“无法无天”的：** 任何地方、任何组件、任何一段第三方脚本都可以随时修改 `window.userData = ...`。当 Bug 出现（比如用户头像突然变了）时，你根本无从查起是谁在什么时候动了它。

**Redux 是“法治社会”：** 它规定了**单向数据流**。

- 你想改数据？必须派发 `Action`。
- 数据怎么改？必须经过 `Reducer`。
- 你可以通过 **Redux DevTools** 看到每一条 Action 的时间线，甚至可以“时光倒流”回到上一个状态。

**响应式更新**

Redux 与 UI 框架（React/Vue）有深度的集成，而 `window` 只是个死板的对象。

- **`window` 不会通知你：** 当你执行 `window.count = 1` 时，React 组件并不知道数据变了，页面不会自动刷新。你需要手动写一套监听机制来强制重新渲染，这非常痛苦。
- **Redux 自动触发渲染：** 通过 `react-redux` 的 `useSelector` 或 `connect`，组件会自动订阅（Subscribe）它关心的那部分状态。一旦 Store 里的数据更新，组件会精准地、自动地进行 **Re-render**

**内存管理与闭包保护**

- **污染全局作用域：** 把所有东西挂在 `window` 上会导致全局变量污染，容易产生命名冲突（尤其是引入第三方库时）。
- **闭包安全：** Redux 的 `store` 被封装在闭包中，外部无法直接修改 `state` 的值（因为它是只读的），只能通过定义的接口（Dispatch）进行交互。这极大增强了数据的安全性