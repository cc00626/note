

## useRef的原理

### 1. 本质：逃离渲染的“保险箱”

`useRef` 的本质是一个在组件整个生命周期内**保持引用不变**的普通 JavaScript 对象。

- **数据结构：** 永远是 `{ current: ... }`。
- **核心逻辑：** 无论组件重新渲染多少次，`useRef` 返回的永远是同一个内存地址的对象。
- **关键特性：** 修改 `ref.current` **不会**触发组件重新渲染（Re-render）。

------

### 2. 内部原理 (Fiber 级别)

在 React 内部，`useRef` 的工作机制如下：

1. **挂载期 (Mount)：** React 创建一个 `{ current: initialValue }` 对象，并将其存储在该组件对应的 **Fiber 节点**的 `memoizedState` 属性中。
2. **更新期 (Update)：** 每次函数组件重新执行时，React 只是简单地从 Fiber 中取出同一个对象并返回。
3. **对比机制：** * `useState` 修改值会调用 `dispatchAction` 触发调度更新。
   - `useRef` 修改值只是纯粹的 JavaScript 对象属性赋值，React 引擎完全感知不到变化。

------

### 3. 两大核心应用场景

#### 场景 A：访问 DOM 节点

这是最常用的场景，用于直接操作浏览器原生的 DOM 元素。

```
const inputRef = useRef(null);

const focusInput = () => {
  // 直接操作 DOM 属性
  inputRef.current.focus();
};

return <input ref={inputRef} />;
```

- **赋值时机：** React 会在 **Commit 阶段**（DOM 挂载后）将 DOM 元素赋给 `current`。
- **清理时机：** 组件卸载时，React 会自动将 `current` 设为 `null`。

#### 场景 B：跨渲染周期的“记忆存根”

用于存储那些不需要反映在 UI 上的变量，避免因为闭包陷阱拿到过时的值。

- **存储定时器：** `timerRef.current = setInterval(...)`
- **记录上一次的 Props：** 在 `useEffect` 中更新 Ref，可对比新旧值。
- **防止竞态请求：** 存储一个标记位，判断异步回调是否该执行。



## useCallback的原理

### 1. 核心矛盾：函数组件的“自杀式”更新

在 JavaScript 中，每次执行函数都会创建一个新的内存地址。

- **现象：** 每次父组件重新渲染（Re-render），内部定义的函数都会被“重新创建”。
- **后果：** 即使函数逻辑没变，它的**引用地址**变了。如果这个函数作为 Props 传给子组件，会导致被 `React.memo` 包裹的子组件误以为数据变了，从而引发无意义的重复渲染。

------

### 2. 内部原理：闭包与 Fiber 存储

`useCallback` 的本质是利用 **Memoization（记忆化）** 技术。

#### 挂载阶段 (Mount)

当组件首次渲染时，React 执行 `useCallback(fn, deps)`：

1. 它将你传入的函数 `fn` 和依赖项数组 `deps` 直接存储在当前 Fiber 节点的 `memoizedState` 中。
2. 返回你传入的这个 `fn`。

#### 更新阶段 (Update)

当组件重新渲染时，React 会对比新旧依赖项：

1. **对比依赖项：** 使用 `Object.is` 算法逐一比对 `deps` 数组。
2. **命中缓存：** 如果依赖项**完全一致**，React 会直接从 Fiber 节点取出**上次存好的旧函数**返回，丢弃本次新创建的函数。
3. **失效更新：** 如果依赖项**发生了变化**，React 才会存储并返回本次新创建的函数。

------

### 3. useCallback vs useMemo

理解这两者的关系能让你一眼看穿 React 的 Hooks 体系：

- **`useMemo`：** 缓存的是**函数的执行结果**（返回值）。
- **`useCallback`：** 缓存的是**函数本身**。

> **底层公式：** `useCallback(fn, deps)` 其实就等价于 `useMemo(() => fn, deps)`。

------

### 4. 关键误区：并不是所有函数都要加 useCallback

这是新手最容易犯的错误。**滥用 `useCallback` 反而会变慢。**

- **成本：** 每次渲染，React 依然要创建新函数，还要额外创建一个数组（deps），并进行一次循环遍历对比。
- **结论：** 只有在以下两种场景下，`useCallback` 才有意义：
  1. **配合 `React.memo`：** 当函数作为 Props 传递给被 `memo` 的子组件时。
  2. **作为其他 Hook 的依赖：** 比如该函数被用在了 `useEffect` 的依赖项中，为了防止 Effect 频繁触发

## useMemo原理

### 1. 核心原理：记忆化存储（Memoization）

`useMemo` 的原理可以概括为：**“空间换时间”**。React 在内部开辟了一块空间，用来记录上一次计算的结果和当时的依赖项。

#### 挂载阶段 (Mount)

当组件首次渲染时，React 执行 `useMemo(calculateValue, deps)`：

1. **执行函数：** 调用 `calculateValue()` 得到初始计算结果。
2. **存入 Fiber：** 将该**结果**和**依赖项数组**一起存入当前 Fiber 节点的 `memoizedState` 中。
3. **返回结果：** 将计算出的值返回给组件。

#### 更新阶段 (Update)

当组件重新渲染时，React 会执行以下逻辑：

1. **取出旧数据：** 从 Fiber 节点取出上次存的 `{ value, deps }`。
2. **对比依赖项：** 使用 `Object.is` 逐一对比新的 `deps` 和旧的 `deps`。
3. **判断逻辑：**
   - **如果依赖项没变：** 直接返回上次存好的 `value`，**完全不执行**计算函数。
   - **如果依赖项变了：** 重新执行 `calculateValue()`，存入新的结果和依赖项，并返回新值。

------

### 2. useMemo 与 useCallback 的底层联系

在 React 源码中，两者的逻辑高度统一。你可以把 `useCallback` 看作是 `useMemo` 的一个**特例**：

- `useMemo(() => value, deps)` -> 缓存值。
- `useCallback(fn, deps)` -> 缓存函数。
- **本质等价：** `useCallback(fn, deps)` 内部其实就是在做 `useMemo(() => fn, deps)`。

------

### 3. 为什么需要 useMemo？（两个关键场景）

#### 场景 A：跳过耗时计算

假设你有一个包含数千条数据的列表，需要进行复杂的过滤或排序：

JavaScript

```
const sortedList = useMemo(() => {
  // 假设这个排序需要消耗 100ms
  return list.sort((a, b) => b.value - a.value);
}, [list]); // 只有 list 变了才重排，点击其他按钮（修改其他 State）不会触发重排
```

#### 场景 B：保持引用稳定性（防止子组件误触发渲染）

这是 `useMemo` 最容易被忽视但极其重要的用途。在 JavaScript 中，`[] !== []`，`{} !== {}`。 如果你在父组件里定义了一个对象并传给子组件：

JavaScript

```
// ❌ 错误做法：每次父组件渲染，options 都是新对象引用
const options = { color: 'blue' }; 

// ✅ 正确做法：options 的引用在依赖不变时保持稳定
const options = useMemo(() => ({ color: 'blue' }), []);

return <Child config={options} />; // 配合 React.memo，子组件就不会因为引用变化而重绘
```

------

### 4. 误区：为什么不能到处都用 useMemo？

很多开发者认为“缓存总比不缓存好”，但这其实是错误的：

1. **内存开销：** 每一个 `useMemo` 都会在 Fiber 节点上增加存储负担。
2. **比较开销：** 每次渲染时，React 遍历依赖项数组并进行 `Object.is` 对比也是需要时间的。
3. **代码复杂度：** 过多的 `useMemo` 会让代码变得难以维护，容易引入闭包陷阱。

**准则：** 只有当**计算确实昂贵**（如处理大数据集）或者为了**维持引用一致性**（避免子组件无效重绘）时才使用。



## useEffect 和useLayoutEffect的区别

### 1. 一句话总结区别

- **`useEffect` (异步)：** 浏览器把内容**画到屏幕上之后**才执行。不会阻塞页面渲染，体验更流畅。
- **`useLayoutEffect` (同步)：** 浏览器**计算完布局但还没画到屏幕上**时执行。会阻塞页面渲染，直到它执行完。

------

### 2. 执行时机对比图

1. **React 渲染 (Render)：** 生成虚拟 DOM，计算 Diff。
2. **更新真实 DOM (Commit)：** React 修改了实际的 DOM 节点。
3. **`useLayoutEffect` 执行：** 此时 DOM 已经变了，但浏览器**还没开始重绘**（屏幕上还是旧内容）。
4. **浏览器重绘 (Paint)：** 将最新的 DOM 渲染到屏幕上。
5. **`useEffect` 执行：** 屏幕上已经看到新内容了，它才在后台默默执行。

------

### 3. 详细对比

### ⚡ useEffect (推荐的首选)

- **特点：** 异步、非阻塞。
- **适用场景：** 绝大多数情况。
  - 发送 API 请求。
  - 设置订阅、定时器。
  - 修改不会直接导致视觉闪烁的状态。
- **优点：** 性能好，不会拖慢页面显示的响应速度。

### 🛠️ useLayoutEffect (特殊工具)

- **特点：** 同步、阻塞。
- **适用场景：** 处理 **“视觉闪烁”** 问题。
  - **测量 DOM：** 比如你需要根据一个元素的宽度，去动态计算另一个元素的位置。
  - **防止抖动：** 如果在 `useEffect` 里修改 DOM，用户会先看到位置 A，然后瞬间跳到位置 B（闪烁）。在 `useLayoutEffect` 里改，用户直接看到最终位置 B。
- **缺点：** 如果逻辑很重，会导致页面卡顿（白屏时间变长）。

------

### 4. 经典的“闪烁”案例对比

假设我们要做一个功能：点击按钮，让一个数字从 0 变成 100，并立即随机改变位置。

- **使用 `useEffect` 的过程：**

  1. 用户点击。
  2. 数字变成 100，出现在默认位置（用户看到了 100）。
  3. `useEffect` 执行，修改位置。
  4. 浏览器再次重绘，数字跳到了新位置。

  - **结果：** 用户看到数字**闪**了一下。

- **使用 `useLayoutEffect` 的过程：**

  1. 用户点击。
  2. 数字变成 100，React 更新 DOM。
  3. `useLayoutEffect` 立即执行，测量并修改位置。
  4. 浏览器一次性把“100”和“新位置”画出来。

  - **结果：** 用户直接看到在新位置的 100，**无闪烁**。

------

### 5. 开发建议 (Rule of Thumb)

1. **默认全用 `useEffect`。**
2. 只有当你发现你的代码导致了 **UI 闪烁**（元素跳动、错位）时，才尝试将其替换为 `useLayoutEffect`。
3. **注意：** 如果你在使用 Next.js 等服务端渲染（SSR）框架，`useLayoutEffect` 会在控制台报警告，因为它在服务器端没法测量 DOM。



## Memo、useMemo、useCallback区别

| **工具**          | **类型**       | **作用对象**         | **核心目的**                          |
| ----------------- | -------------- | -------------------- | ------------------------------------- |
| **`React.memo`**  | 高阶组件 (HOC) | **组件 (Component)** | 如果 Props 没变，跳过组件的重新渲染。 |
| **`useMemo`**     | Hook           | **返回值 (Value)**   | 缓存复杂的计算结果，避免重复计算。    |
| **`useCallback`** | Hook           | **函数 (Function)**  | 缓存函数引用，保持内存地址不变。      |



## useReducer能代替redux

1. 创建state和reducer

```javascript
// store/reducer.js
export const initialState = { count: 0, user: null };

export function authReducer(state, action) {
  switch (action.type) {
    case 'LOGIN':
      return { ...state, user: action.payload };
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    default:
      return state;
  }
}
```

2. 简历全局context

```javascript
// store/StoreContext.js
import React, { createContext, useReducer, useContext } from 'react';
import { authReducer, initialState } from './reducer';

const GlobalContext = createContext();

export const StoreProvider = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, initialState);

  return (
    // 将 state 和 dispatch 同时放入 Context
    <GlobalContext.Provider value={{ state, dispatch }}>
      {children}
    </GlobalContext.Provider>
  );
};

// 自定义 Hook，方便组件调用
export const useStore = () => useContext(GlobalContext);
```

3. 在根组件中使用

```javascript
// index.js 或 App.js
import { StoreProvider } from './store/StoreContext';

function App() {
  return (
    <StoreProvider>
      <Navbar />
      <MainContent />
    </StoreProvider>
  );
}
```

4. 使用

```javascript
function Counter() {
  const { state, dispatch } = useStore();

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>加 1</button>
    </div>
  );
}
```

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



