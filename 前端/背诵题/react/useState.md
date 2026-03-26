### useState原理

在执行函数组件之前，React 会进入 `renderWithHooks` 函数。此时，每一个 `useState` 对应一个内部的 `Hook` 对象，这些对象按顺序挂载在 Fiber 节点的 `memoizedState` 属性上，形成一个**单向链表**。

**Hook 对象的数据结构如下：**

- **`memoizedState`**：对于 `useState` 来说，这就是它保存的真实状态值。
- **`baseState`**：基础状态，用于在多个高低优先级任务并存时进行状态合并计算。
- **`queue`**：更新队列。它是一个**环形链表**，存储了所有待执行的 `setCount` 动作。
- **`next`**：指针，指向组件中的下一个 Hook 节点。



React 会根据 `current === null` 来判断是首次渲染还是更新渲染，并为其分配不同的 **Dispatcher**。

**第一阶段：Mount (首次调用)**

1. **创建 Hook 对象**：按代码书写顺序创建一个新的 Hook 对象。
2. **初始化**：将 `initialValue` 存入 `memoizedState`。
3. **返回**：返回当前状态和修改状态的函数 `dispatchAction`。
4. **连接**：通过 `next` 指针将这个 Hook 挂在 Fiber 的链表末尾。

**第二阶段：调用 Setter (触发更新)**

当你调用 `setCount(prev => prev + 1)` 时，React 内部执行的是 `dispatchAction`：

- **创建 Update 对象**：包含你要更新的值或函数。
- **插入环形链表**：将更新对象插入到 `Hook.queue.pending` 中。
- **优化**：由于是**环形链表**，`pending` 始终指向最后一个更新，而 `pending.next` 就是第一个更新。这使得插入操作的时间复杂度为 $O(1)$。
- **调度**：触发 Fiber 的调度任务，准备进行下一次 Render 阶段。

**第三阶段：Update (重新渲染)**

当组件再次执行时：

1. **复用 Hook**：React 按照顺序从 Fiber 的 `memoizedState` 链表中取出对应的 Hook 对象。
2. **计算新值**：遍历 `queue.pending` 环形链表，依次执行所有的更新动作，得出最新的 `memoizedState`。
3. **返回新状态**：返回计算后的状态值和同一个 `dispatchAction` 函数。



**批量处理 (Batching)**：在 React 18+ 中，无论在原生事件、`setTimeout` 还是 `Promise` 中，多次 `setState` 都会被自动合并，仅触发一次 Render 阶段。

**函数式更新**：

- **场景**：新状态依赖旧状态（如循环累加）。
- **做法**：`setCount(prev => prev + 1)`。
- **目的**：避免“闭包陷阱”导致引用到旧渲染周期中的状态值。

**Bailout 机制**：React 使用 `Object.is` 比较新旧状态。若引用一致，React 会跳过该组件及其子树的 Render 阶段，优化性能。



### setState是同步还是异步

setState 既不是完全同步也不是完全异步，它的行为取决于 React 的调度机制。在 React 的合成事件中，setState 会被批量更新，因此看起来是异步的；

而在某些非 React 控制的环境中，在 React 18 之前可能是同步执行的。

但在 React 18 之后，引入了自动批处理机制，大多数情况下都会进行批量更新。setState 的本质是将更新加入队列，由调度器统一处理，而不是立即修改 state。

