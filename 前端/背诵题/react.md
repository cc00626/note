## 为什么要使用全局状态管理，而不是挂到window上

### 1. 核心痛点：失去“响应式（Reactivity）”

这是最根本的区别。

- **`window` 方案：** `window` 只是一个普通的 JavaScript 对象。当你修改 `window.user = 'Gemini'` 时，React 根本不知道这个值变了。
  - **后果：** 你的 UI 不会自动更新。如果你想让界面变化，你得手动调用 `forceUpdate()` 或者手动触发 DOM 操作。这回到了 jQuery 时代的原始开发模式。
- **状态管理：** 无论是 Redux、Zustand 还是 Context，它们都深度集成了 React 的更新机制。一旦状态改变，关联的组件会自动重新渲染。

------

### 2. 内存安全性：闭包陷阱与数据孤岛

在 React 的并发模式（Concurrent Mode）下，数据的处理非常微妙。

- **`window` 是全局单例：** 它是脱离于 React 渲染周期之外的。如果你在异步操作中修改 `window` 上的数据，可能会产生难以调试的“竞态条件（Race Condition）”。
- **状态管理：** 它们通常使用**不可变数据（Immutable Data）**。每次更新都会生成一个新的快照。这保证了在 React 的双缓存 Fiber 树切换时，每一帧看到的都是一致、正确的数据。

------

### 3. 命名空间污染（Namespace Pollution）

`window` 是整个浏览器的公共空间，第三方脚本（如统计插件、广告插件、浏览器扩展）都在用它。

- **风险：** 如果你定义了 `window.config`，某个插件可能也定义了 `window.config`。
- **后果：** 你的业务逻辑可能会被莫名其妙地覆盖，这种 Bug 排查起来简直是噩梦。

------

### 4. 无法进行“时间旅行（Time Travel）”与调试

全局变量就像一个黑盒，你不知道它是**谁**在**什么时候**改的。

- **Redux/Zustand：** 它们强制通过 Action 或函数来修改状态。这意味着你可以追踪每一条修改记录：
  - *“在 10:05:01，Login 组件触发了 LOGIN_SUCCESS，导致状态里的 user 从 null 变成了 Gemini”。*
- **`window`：** 任何一行代码执行 `window.user = null` 都不留痕迹。

------

### 5. 什么时候**可以**挂载到 `window`？

虽然不推荐管理“业务状态”，但 `window` 在某些特定场景下是有用的：

- **静态配置：** 比如后端注入的全局 API 基础地址 `window.__API_CONFIG__`。
- **三方 SDK 挂载：** 比如 `window.wx`（微信 SDK）或地图 API。
- **极其临时的跨页面调试：** 仅限开发者自己在控制台手动改一下值。





## hooks的使用规范

### 1. 核心规范：两条铁律

#### 规范一：只在最顶层使用 Hooks (Only Call Hooks at the Top Level)

**不要在循环、条件判断或者嵌套函数中使用 Hook。**

- **原因：** React 依赖 Hook 的**调用顺序**来确定哪个 State 对应哪个变量。
- **原理：** 在 Fiber 节点内部，Hooks 是以**单向链表**的形式存储的。每次渲染时，React 都会按顺序“数着”往下走。如果你在 `if` 里写了 Hook，导致某次渲染少了一个，整条链表的索引就会错位，结果就是 `count` 拿到了 `name` 的值，造成数据崩溃。

#### 规范二：只在 React 函数中调用 Hooks (Only Call Hooks from React Functions)

**不要在普通的 JavaScript 函数中调用 Hook。**

- **允许的地方：**
  1. React 函数组件。
  2. 自定义 Hook（即以 `use` 开头的函数）。
- **原因：** Hook 的底层实现需要访问当前正在渲染的 Fiber 实例。普通 JS 函数没有这个上下文环境。

------

### 2. 命名规范：必须以 `use` 开头

不管是官方 Hook 还是自定义 Hook，必须遵循 **驼峰命名法（camelCase）** 且以 `use` 开头（如 `useAuth`）。

- **为什么？** 这是为了让静态代码检查工具（如 `eslint-plugin-react-hooks`）能够识别出这是一个 Hook，从而帮你检查它是否违反了第一条铁律。

## hoks的执行入口？都干了什么

在react执行的时候会有一个renderWithHooks ，是hooks的执行入口。

`renderWithHooks` 的核心任务有 **5件事**

1. 处理化hooks环境

   ```
   currentlyRenderingFiber = workInProgress
   workInProgressHook = null
   currentHook = null
   
   currentlyRenderingFiber  当前组件fiber
   workInProgressHook       当前正在处理的hook
   currentHook              上一次渲染的hook
   ```

   

2. 记录当前的fiber

3. 挂载或更新hook，判断有没有旧的fiber

4. 构建hooks链表

5. 执行组件函数，在执行的时候会调用hooks

## Hooks 在 Fiber 上是如何存储的？数据结构是什么？

```
Fiber
 └─ memoizedState
        ↓
      Hook1
        ↓
      Hook2
        ↓
      Hook3
```

React 会把 Hooks 存储在 Fiber 的 `memoizedState` 字段上。
 `memoizedState` 指向一个 Hook 对象的单向链表，每个 Hook 通过 `next` 指针连接。
 在组件渲染时，React 会按照 Hooks 的调用顺序依次创建或更新 Hook 节点，并挂到 Fiber 上。

```
{
 memoizedState,
 baseState,
 baseQueue,
 queue,
 next
}
```

## mount 和 update 阶段 Hooks 的执行流程有什么不同？

在 React 中，Hooks 的执行是在 `renderWithHooks` 中完成的。React 会根据 `current` 是否为 null 来判断当前是 mount 还是 update 阶段。

在 mount 阶段，React 会使用 `HooksDispatcherOnMount`，每个 Hook 调用都会执行 mount 版本函数，例如 `mountState`、`mountEffect`。这些函数会创建新的 Hook 对象，并通过 `mountWorkInProgressHook` 挂到 Fiber 的 Hook 链表上，同时初始化 state 和更新队列。

在 update 阶段，React 会使用 `HooksDispatcherOnUpdate`，调用 update 版本函数，例如 `updateState`、`updateEffect`。此时 React 会通过 `updateWorkInProgressHook` 读取旧 Fiber 上的 Hook，并创建新的 Hook 节点，同时复制旧 state，然后根据更新队列计算新的 state。

简单来说，mount 阶段负责创建 Hook 并初始化状态，而 update 阶段负责复用旧 Hook 并计算新的状态。

## useState 的更新队列是如何设计的？

更新队列

```
UpdateQueue = {
  pending,   // 环形链表
  lanes,     
  dispatch,
  lastRenderedReducer,
  lastRenderedState
}
```

为什么使用环形链表

> 环形链表的时间复杂度是O(1)
>
> update.next = pending.next
> pending.next = update
> queue.pending = update
>
> 普通链表要将更新对象插入到对尾，时间复杂度是O(n)



## useEffect 的实现原理是什么？

useEffect 在 render 阶段不会执行副作用函数，而是创建一个 Effect 对象，存入 Fiber 的 effect 链表中。React 在 commit 阶段通过 flushPassiveEffects 遍历 effect list，先执行上一次的 cleanup，再执行新的 effect。依赖数组通过 Object.is 比较，如果依赖没有变化则跳过执行。

## useEffect 的依赖数组是如何比较的？

React 在 updateEffect 阶段通过 areHookInputsEqual 比较依赖数组。比较方式是逐项使用 Object.is 进行浅比较。如果任意一项依赖变化，则标记 HookHasEffect，在 commit 阶段重新执行 effect。

## useEffect 的清理函数什么时候执行？

effect重新执行前

组件卸载的时候

严格开发模式，会故意执行两次

## useEffect 中调用 setState 会有什么问题？

在 useEffect 中调用 setState 是允许的，但需要注意几个问题。首先，setState 会触发新的 render，因此如果依赖数组处理不当，很容易导致无限更新循环。其次，由于 effect 在 commit 阶段执行，所以第一次 render 后再 setState 会导致额外的一次 render。另外 effect 内部存在闭包问题，如果直接使用旧 state 可能导致数据不一致，通常需要使用函数式更新。最后，如果 effect 每次 render 都执行，也会带来性能问题。



## useEffect闭包陷阱

React 每次 render 都会创建新的函数作用域，useEffect 在依赖数组为空时只会在第一次 render 执行，所以 setInterval 的回调函数捕获的是第一次 render 的 count 值。即使后续组件重新渲染，定时器中的闭包仍然引用旧的作用域，因此打印的始终是旧值，这就是 React 中典型的 Stale Closure 问题。



使用useRef解决，或者useEffect



## 如何用 useMemo 模拟 useCallback

useCallback本质就是useMemo的语法糖

```javascript
function useCallback(callback, deps) {
  return useMemo(() => callback, deps);
}
```

## 为什么react 16要引入fiber





## 双缓存机制是如何进行工作的

在fiber节点上有一个` fiber.alternate`字段用于连接两棵树

1. 当组件发生更新的时候，会调用setState。首先找到current tree

2. 创建workInProgress tree，基于current。
3. 在Reconciliation阶段，在workInProgress tree上进行diff比较，计算更新，这个过程可以被打断。
4. 在commit阶段，react会交换两颗树，然后进行更新。

## alternate 指针的作用是什么？

在 React Fiber 架构中，每个 Fiber 节点都有一个 alternate 指针，它用于连接 current Fiber 和 workInProgress Fiber。React 在更新 UI 时会维护两棵 Fiber Tree，一棵是当前已经渲染到页面上的 current Tree，另一棵是用于计算下一次 UI 的 workInProgress Tree。

alternate 指针让这两个树中的对应节点可以互相引用，这样 React 在更新时可以基于 current Fiber 复用已有的 Fiber 节点，而不是重新创建整棵树，从而提升性能。

同时，这种设计也是 React 双缓存机制的基础，使 React 可以在后台构建新的 UI，而不影响当前页面，并支持可中断渲染和并发更新。

## FiberRootNode 的作用是什么？

他是react应用的一个根管理器，当我们调用`ReactDOM.createRoot(container)` 时，React 内部创建的第一样东西就是 `FiberRootNode`。



他保存着运行期间的全局状态，比如并发更新的优先级，待提交的任务，当前的双缓存树指向



| **属性名**          | **作用**       | **面试解析**                                                 |
| ------------------- | -------------- | ------------------------------------------------------------ |
| **`current`**       | **核心指针**   | 指向当前页面正在显示的 **`HostRootFiber`**（即 Fiber 树的根）。 |
| **`containerInfo`** | **挂载点**     | 保存你传入 `createRoot` 的那个 DOM 容器（如 `#root`）。      |
| **`pendingLanes`**  | **任务优先级** | 一个位掩码，记录了当前应用中所有**待处理**的更新任务优先级。 |
| **`finishedWork`**  | **待提交作品** | Render 阶段完成后，构建好的 WIP 树会挂在这里，等待 Commit 阶段刷到屏幕上。 |
| **`callbackNode`**  | **调度任务**   | 如果当前有正在进行的调度任务（由 Scheduler 管理），会存在这里。 |

## React 的 render 和 commit 两个阶段分别做什么？

在render阶段，主要执行beginwork，completwork两个流程，react会深度遍历整棵树。



